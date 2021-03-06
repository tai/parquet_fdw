[![Build Status](https://travis-ci.org/zilder/parquet_fdw.svg?branch=master)](https://travis-ci.org/zilder/parquet_fdw)

[WIP]

# parquet_fdw

Parquet foreign data wrapper for PostgreSQL.

## Installation

`parquet_fdw` requires `libarrow` and `libparquet` installed in your system (tested with version 0.13.0). Please refer to [libarrow installation page](https://arrow.apache.org/install/) or [building guide](https://github.com/apache/arrow/blob/master/cpp/README.md).
To build `parquet_fdw` run:
```sh
make install
```
or in case when PostgreSQL is installed in a custom location:
```sh
make install PG_CONFIG=/path/to/pg_config
```
Also additional compilation flags can be passed through `CCFLAGS` variable.

After extension was successfully installed run in `psql`:
```sql
create extension parquet_fdw;
```

## Using
To start using `parquet_fdw` you should first create server and user mapping. For example:
```sql
create server parquet_srv foreign data wrapper parquet_fdw;
create user mapping for postgres server parquet_srv options (user 'postgres');
```
Now you should be able to create foreign table from Parquet files. Currently `parquet_fdw` supports the following column [types](https://github.com/apache/arrow/blob/master/cpp/src/arrow/type.h) (to be extended shortly):

| Parquet type |  SQL type |
|--------------|-----------|
|        INT32 |      INT4 |
|        INT64 |      INT8 |
|    TIMESTAMP | TIMESTAMP |
|       DATE32 |      DATE |
|       STRING |      TEXT |
|       BINARY |     BYTEA |
|         LIST |     ARRAY |

Currently `parquet_fdw` doesn't support structs and nested lists.

Following options are supported:
* **filename** - path to Parquet file to read;
* **sorted** - space separated list of columns that Parquet file is already sorted by; that would help postgres to avoid redundant sorting when running query with `ORDER BY` clause;
* **use_mmap** - whether memory map operations will be used instead of file read operations (default `false`);
* **use_threads** - enables `arrow`'s parallel columns decoding/decompression (default `false`).

GUC variables:
* **parquet_fdw.use_threads** - global switch that allow user to enable or disable threads (default `true`).

Example:
```sql
create foreign table userdata (
    id           int,
    first_name   text,
    last_name    text
)
server parquet_srv
options (
    filename '/mnt/userdata1.parquet',
    sorted 'id'
);
```

### Parallel queries
`parquet_fdw` also supports [parallel query execution](https://www.postgresql.org/docs/current/parallel-query.html) (not to confuse with multi-threaded decoding feature of `arrow`). It is disabled by default; to enable it run `ANALYZE` command on the table. The reason behind this is that without statistics postgres may end up choosing a terrible parallel plan for certain queries which would be much worse than a serial one (e.g. grouping by a column with large number of distinct values).
