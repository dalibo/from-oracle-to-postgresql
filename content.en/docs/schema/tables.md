---
weight: 2
bookFlatSection: false
title: "Table migration"
previouspage: "types"
nextpage: "views"
---

## Table migration

Table definition are quite identical between both RDBMS, except that PostgreSQL 
does not have global temporary tables. Each temporary table is private to its 
own session. Data inserted into a temporary table can be automatically destroyed 
at the transaction end (using `ON COMMIT DELETE ROWS` clause) or only at the end 
of the session (with default implicit clause `ON COMMIT PRESERVE ROWS`). In addition,
PostgreSQL can also automatically drop the table when the transaction is finished 
(`ON COMMIT DROP`). A workaround exists and will be proposed later in this guide.

There's no equivalent in PostgreSQL of the storage clause like `INITTRANS` and
`MAXEXTENTS`. PostgreSQL allocates storage on a dynamic basis. Only `PCTFREE`,
which indicates a percentage of size left free in a block, has an equivalent
in fillfactor under PostgreSQL. However `PCTUSED` has no equivalent and has
no sense regarding how PostgreSQL manages its storage. 

References:

* [CREATE TABLE](http://www.postgresql.org/docs/current/static/sql-createtable.html)

### Virtual Columns

To replace virtual columns, PostgreSQL supports _generated columns_, introduced
in version 12. A generated column is a special column which is always computed 
from other columns. Only `STORED` capability is implemented; a `VIRTUAL` generated
column may be added at a future major release.

Basic example of a table with a virtal column with Oracle:

```sql
CREATE TABLE employees (
  id          NUMBER,
  first_name  VARCHAR2(10),
  salary      NUMBER(9,2),
  commission  NUMBER(3),
  salary2     NUMBER GENERATED ALWAYS 
    AS (ROUND(salary*(1+commission/100),2)) VIRTUAL,
);
```

This could be ported in PostgreSQL with the following syntax:

```sql
CREATE TABLE employees (
  id          BIGINT,
  first_name  VARCHAR(10),
  salary      DOUBLE PRECISION,
  commission  INTEGER,
  salary2     DOUBLE PRECISION GENERATED ALWAYS 
    AS (ROUND((salary*(1+commission/100))::numeric,2)) STORED
);
```

To emulate a non-stored data like `VIRTUAL` works with Oracle, views are usually 
the best solution.

```sql
CREATE TABLE employees (
  id          BIGINT,
  first_name  VARCHAR(10),
  salary      DOUBLE PRECISION,
  commission  INTEGER
);

CREATE VIEW virt_employees AS
SELECT id, first_name, salary, commission, 
        (ROUND((salary*(1+commission/100))::numeric,2)) salary2
  FROM employees;
```

References:

* [Generated column](https://pgpedia.info/g/generated-column.html)
* [CREATE TABLE](http://www.postgresql.org/docs/current/static/sql-createtable.html)
* [CREATE VIEW](http://www.postgresql.org/docs/current/static/sql-createview.html)

### Index-Organized Tables

PostgreSQL has no direct equivalent of the _IOT_. 

It's possible to physically sort a table in the same order as one of its indexes
thanks to the `CLUSTER` statement. However, each following write will be written
in any available space in the table, not taking the index's order into account.
It's possible to reduce this phenomenon by reducing the _fillfactor_ to a smaller 
value (than the default 100) to let space to the new versions of a record, as a 
new version is put on the same page as the original one if free space is sufficient. 

An Index Organized Table, or _clustered_ table, can bring a large improvement on 
the _range scan_ type queries, where a consecutive range of records of a table are
accessed by a query. For instance, a query returning all the customers whose 
name start with `MAR (SELECT * FROM clients WHERE nom LIKE 'MAR%' )` will benefit 
from an IOT. Reading will be faster as all records pointed by the index will be 
in the same block, or in a few consecutive blocks.

`CLUSTER` is thus a one-time operation, that should be performed again more or 
less frequently, depending on the write rate in the table, and the data 
maintenance window. One has to keep in mind that this operation is heavy and 
requires an exclusive lock that will block all reads and writes during the 
rebuild. Please note that no reindexation is necessary following a `CLUSTER`, as 
the indexes are already rebuilt during the operation. 

The following example illustrates a significant performance improvement on a 
simple table containing 10 millions of records. The query runtime is divided by 
two after clustering it on the appropriate index. 

Objects creation:

```sql
CREATE TABLE t6 (i SERIAL PRIMARY KEY, v INTEGER);
INSERT INTO t6 (v) SELECT round(random()*10000) FROM generate_series(1, 10000000);
CREATE INDEX idx_t6_v ON t6 (v);
```

The query runtime is consistent, as data is cached:

```sql
SELECT count(*) FROM t6 WHERE v BETWEEN 100 AND 1000;
--  count  
-- --------
--  902274
-- 
-- Time: 350,100 ms
```

This table is then clustered over the `idx_t6_v` index: 

```sql
CLUSTER t6 USING idx_t6_v;
```

The runtime has been divided by a factor of two: 

```sql
SELECT count(*) FROM t6 WHERE v BETWEEN 100 AND 1000;
--  count  
-- --------
--  902274
-- 
-- Time: 148,755 ms
```

References:

* [CLUSTER](http://www.postgresql.org/docs/current/static/sql-cluster.html)

### External tables

External tables make it possible to interact with CSV files from an Oracle
database. This can be emulated through a **Foreign Data Wrapper**, as implemented by
PostgreSQL in respect with the SQL/MED extension to SQL. This new feature makes 
it possible to declare Foreign Data Wrappers (FDW) which allow for remote access 
to objects outside of the database: another database, a CSV file, etc… 

Given an external table's definition for Oracle:

```sql
CREATE OR REPLACE DIRECTORY ext AS '/usr/tmp/';
CREATE TABLE ext_tab (
  empno  CHAR(4),
  ename  CHAR(20),
  job    CHAR(20),
  deptno CHAR(2)
) ORGANIZATION EXTERNAL (
  TYPE oracle_loader
  DEFAULT DIRECTORY ext
    ACCESS PARAMETERS (
    RECORDS DELIMITED BY NEWLINE
    BADFILE 'bad_%a_%p.bad'
    LOGFILE 'log_%a_%p.log'
    FIELDS TERMINATED BY ','
    MISSING FIELD VALUES ARE NULL
    REJECT ROWS WITH ALL NULL FIELDS
    (empno, ename, job, deptno))
    LOCATION ('demo1.dat')
  )
PARALLEL
REJECT LIMIT 0
NOMONITORING;
```

It will be translate this way in PostgreSQL:

```sql
CREATE EXTENSION file_fdw;

CREATE SERVER ext FOREIGN DATA WRAPPER file_fdw;

CREATE FOREIGN TABLE ext_tab (
        empno CHAR(4),
        ename CHAR(20),
        job CHAR(20),
        deptno CHAR(2)
) SERVER ext OPTIONS(filename '/usr/tmp/demo1.dat', format 'csv',
delimiter ',');
```

If the sole pupose is loading data, please note that PostgreSQL's `COPY` statement 
is an alternative to the creation of a foreign data wrapper. For example, to load 
the example's CSV file, one only has to run the following statement:

```sql
COPY ext_table FROM '/usr/tmp/demo1.csv' WITH CSV;
```

References:

* [file_fdw](http://www.postgresql.org/docs/current/static/file-fdw.html)
* [COPY](http://www.postgresql.org/docs/current/static/sql-copy.html)

### Global temporary tables

The Oracle `GLOBAL TEMPORARY TABLE` object has no equivalent in PostgreSQL. As a
reminder, a global temporary table is a table whose structure is permanent, but
whose content is temporary, i.e. specific to each user session.

Several solutions exist to carry these objects.

Depending on usage, it may happen that a classic table can meet the need, with a
simple truncation of the table with `TRUNCATE` command on first use. But this
prevents a use of the table by several users in parallel.

The [pgtt](https://github.com/darold/pgtt) extension, developed by Gilles
Darold, emulates how global temporary tables work. But his use requires the
extension library to be loaded once the user is logged in.

It is also possible to build your own temporary tables globally as follows:

* create a set of tables in a dedicated schema with the structure of global
  temporary tables to emulate; these tables will serve as a model for creating
  temporary tables;
* associate a comment to each table to indicate the mode of operation of the
  temporary table during transaction COMMITs (`ON COMMIT PRESERVE ROWS` or `ON
  COMMIT DROP`);
* create a procedure or function for initializing a temporary table with a
  pre-existing table as a template;
* in PL/pgSQL code, call this procedure or function at the first using the
  global temporary table.

```sql
CREATE SCHEMA IF NOT EXISTS gtt_schema;
CREATE TABLE gtt_schema.gtt_table_1 (
  col1 INTEGER,
  col2 TEXT
);
COMMENT ON TABLE gtt_schema.gtt_table_1 IS 'ON COMMIT PRESERVE ROWS';
```

```sql
CREATE OR REPLACE PROCEDURE prepare_temp_table(p_relname varchar) AS
$$
DECLARE
  v_temp_schema    varchar = 'gtt_schema';
  v_temp_desc      varchar;
BEGIN
  -- Reading the comment associated with the table
  v_temp_desc := pg_catalog.obj_description(
    (format('%s.%s', v_temp_schema, p_relname))::regclass, 'pg_class'
  );

  -- Creating the temporary table
  EXECUTE format(
    'CREATE TEMP TABLE %2$s (LIKE %1$s.%2$s) %3$s',
    v_temp_schema, p_relname,
    CASE
      WHEN v_temp_desc ~* 'delete' THEN 'ON COMMIT DELETE ROWS'
      WHEN v_temp_desc ~* 'drop' THEN 'ON COMMIT DROP'
      ELSE 'ON COMMIT PRESERVE ROWS'
    END
  );

  -- If the temporary table already exists, empty it
  EXCEPTION WHEN SQLSTATE '42P07' THEN
    EXECUTE format('TRUNCATE %s', p_relname);
END;
$$
LANGUAGE plpgsql;

CALL prepare_temp_table('gtt_table_1');
```

References:

* [CREATE TABLE](https://www.postgresql.org/docs/current/sql-createtable.html)