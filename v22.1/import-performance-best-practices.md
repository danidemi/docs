---
title: Import Performance Best Practices
summary: Best practices for optimizing import performance in CockroachDB.
toc: true
docs_area: migrate
---

This page provides best practices for optimizing [import](import.html) performance in CockroachDB.

Import speed primarily depends on the amount of data that you want to import. However, there are two main factors that have can have a large impact on the amount of time it will take to run an import:

- [Splitting data](#split-your-data-into-multiple-files)
- [Import format](#choose-a-performant-import-format)

{{site.data.alerts.callout_info}}
If the import size is small, then you do not need to do anything to optimize performance. In this case, the import should run quickly, regardless of the settings.
{{site.data.alerts.end}}

## Split your data into multiple files

Splitting the import data into multiple files can have a large impact on the import performance. The following formats support multi-file import using `IMPORT INTO`:

- `CSV`
- `DELIMITED DATA`
- `AVRO`

For these formats, we recommend splitting your data into as many files as there are nodes.

For example, if you have a 3-node cluster, split your data into 3 files, create your table, and [import into that table](import-into.html):

{% include_cached copy-clipboard.html %}
~~~ sql
CREATE TABLE customers (id UUID PRIMARY KEY, name TEXT, INDEX name_idx(name));
~~~

{% include_cached copy-clipboard.html %}
~~~ sql
IMPORT INTO customers (id, name)
    CSV DATA (
      's3://{BUCKET NAME}/{customers.csv}?AWS_ACCESS_KEY_ID={KEY ID}&AWS_SECRET_ACCESS_KEY={SECRET ACCESS KEY}',
      's3://{BUCKET NAME}/{customers2.csv}?AWS_ACCESS_KEY_ID={KEY ID}&AWS_SECRET_ACCESS_KEY={SECRET ACCESS KEY}',
      's3://{BUCKET NAME}/{customers3.csv}?AWS_ACCESS_KEY_ID={KEY ID}&AWS_SECRET_ACCESS_KEY={SECRET ACCESS KEY}',
      's3://{BUCKET NAME}/{customers4.csv}?AWS_ACCESS_KEY_ID={KEY ID}&AWS_SECRET_ACCESS_KEY={SECRET ACCESS KEY}',
    );
~~~

CockroachDB imports the files that you give it, and does not further split them. For example, if you import one large file for all of your data, CockroachDB will process that file on one node—even if you have more nodes available. However, if you import two files (and your cluster has at least two nodes), each node will process a file in parallel. This is why splitting your data into as many files as you have nodes will dramatically decrease the time it takes to import data.

{{site.data.alerts.callout_info}}
If you split the data into **more** files than you have nodes, it will not have a large impact on performance.
{{site.data.alerts.end}}

## Choose a performant import format

Import formats do not have the same performance because of the way they are processed. Below, import formats are listed from fastest to slowest:

1. [`CSV`](migrate-from-csv.html) or [`DELIMITED DATA`](import-into.html) (both have about the same import performance)
1. [`AVRO`](migrate-from-avro.html)
1. [`MYSQLDUMP`](migrate-from-mysql.html)
1. [`PGDUMP`](migrate-from-postgres.html)

We recommend formatting your import files as `CSV`, `DELIMITED DATA`, or `AVRO`. These formats can be processed in parallel by multiple threads, which increases performance. To import in these formats, use [`IMPORT INTO`](import-into.html).

`MYSQLDUMP` and `PGDUMP` run a single thread to parse their data, and therefore have substantially slower performance.

`MYSQLDUMP` and `PGDUMP` are two examples of "bundled" data. This means that the dump file contains both the table schema and the data to import. These formats are the slowest to import, with `PGDUMP` being the slower of the two. This is because CockroachDB has to first load the whole file, read the whole file to get the schema, create the table with that schema, and then import the data. While these formats are slow, see [Import the schema separately from the data](#import-the-schema-separately-from-the-data) for guidance on speeding up bundled-data imports.

{% include {{ page.version.version }}/import-table-deprecate.md %}

### Import the schema separately from the data

For single-table `MYSQLDUMP` or `PGDUMP` imports, split your dump data into two files:

1. A SQL file containing the table schema
1. A CSV file containing the table data

Then, import the schema-only file:

~~~ sql
> IMPORT TABLE customers
FROM PGDUMP
    'https://s3-us-west-1.amazonaws.com/cockroachdb-movr/datasets/employees-db/pg_dump/customers.sql' WITH ignore_unsupported_statements
;
~~~

And use the [`IMPORT INTO`](import-into.html) statement to import the CSV data into the newly created table:

~~~ sql
> IMPORT INTO customers (id, name)
CSV DATA (
    'https://s3-us-west-1.amazonaws.com/cockroachdb-movr/datasets/employees-db/pg_dump/customers.csv'
);
~~~

This method has the added benefit of alerting on potential issues with the import sooner; that is, you will not have to wait for the file to load both the schema and data just to find an error in the schema.

### Import into a schema with secondary indexes

When importing data into a table with secondary indexes, the import job will ingest the table data and required secondary index data concurrently. This may result in a longer import time compared to a table without secondary indexes. However, this typically adds less time to the initial import than following it with a separate pass to add the indexes. As a result, importing tables with their secondary indexes is the default workflow, suitable for most import jobs.

However, in **large** imports, it may be preferable to remove the secondary indexes from the schema, perform the import, and then re-create the indexes separately. This provides increased visibility into its progress and ability to retry each step independently.

- [Remove the secondary indexes](drop-index.html)
- [Perform the import](import-into.html)
- [Create a secondary index](schema-design-indexes.html#create-a-secondary-index)

## See also

- [`IMPORT`](import.html)
- [Migration Overview](migration-overview.html)
- [Migrate from Oracle](migrate-from-oracle.html)
- [Migrate from Postgres](migrate-from-postgres.html)
- [Migrate from MySQL](migrate-from-mysql.html)
- [Migrate from CSV](migrate-from-csv.html)
- [Migrate from Avro](migrate-from-avro.html)
