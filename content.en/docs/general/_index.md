---
weight: 1
bookFlatSection: false
title: "General differences"
url: "general-differences-between-oracle-and-postgresql"
previouspage: "/"
nextpage: "/schema"
---

# General differences between Oracle and PostgreSQL

This part is about specificities and general differences that have to be taken 
into account while migrating from Oracle to PostgreSQL. 

## Users and schemas

In Oracle Database users and schemas are two closely related objects. A user has 
its own schema, the schema is named after the user's name. PostgreSQL does clearly 
distinguish users and schema objects. In PostgreSQL, a schema is really a real 
namespace for the database objects.

## Case sensibility

Oracle implicitly converts objects name into uppercase while PostgreSQL converts 
them into lowercase. The SQL standard do not make any recommendation on this subject.

If needed, the case can be forced by using englobing the object name with 
double-quotes. This practice is not recommended with PostgreSQL, because each access
to a particular object will each-time require double-quotes:

```sql
CREATE TABLE "MyTable" (a INTEGER PRIMARY KEY, b INTEGER);
-- NOTICE:  CREATE TABLE / PRIMARY KEY will CREATE 
--  implicit INDEX "MyTable_pkey" FOR TABLE "MyTable"
-- CREATE TABLE
 
INSERT INTO mytable (a,b) VALUES (1,1);
-- ERROR:  relation "mytable" does NOT exist
-- LINE 1: INSERT INTO mytable (a,b) VALUES (1,1);
 
INSERT INTO MyTable (a,b) VALUES (1,1);
-- ERROR:  relation "mytable" does NOT exist
-- LINE 1: INSERT INTO MyTable (a,b) VALUES (1,1);
 
INSERT INTO "MyTable" (a,b) VALUES (1,1);
-- INSERT 0 1
```

If one forgets double-quotes in his query, PostgreSQL will implicitly convert 
the object name to lowercase and the query will not work correctly. 

## DUAL Table

The Oracle parser does not accept `SELECT` queries that miss the `FROM` clause. 
PostgreSQL does not have this limitation, all references to the `DUAL` table 
can be removed from the queries.

It is also counter-productive to create an artificial `DUAL` table on PostgreSQL. 
This table will need additional locks acquisition and can be a bottleneck for 
queries using this table. 

```sql
SELECT 1+1 AS resultat;
-- -[ RECORD 1 ]
-- resultat | 2
```

## NULL value

The `VARCHAR2` datatype assimilates an empty string with the `NULL` value.
This is not consistent to the SQL standard. 

Several problems can appear, especially when the query developer wrote query 
predicates with this non-standard behavior in mind. Also, Oracle can concatenate 
a string with a `NULL` value without any problem. With PostgreSQL, and as well with 
other RDBMS, the `NULL` value is propagated in the operations: a string concatenated 
to a `NULL` will give a `NULL`. 

```sql
SELECT 
  'ABC'||NULL AS concatenation_with_null, 
  coalesce('ABC'||NULL,'value if null','value is not null');
-- -[ RECORD 1 ]-----------+---------------
-- concatenation_with_null | 
-- coalesce                | value if null
```

## Database Links

PostgreSQL historically handles database links to other PostgreSQL databases
through the `dblink` extension. However, since version 9.1, PostgreSQL features
support for SQL/MED, in form of **Foreign Data Wrappers** (FDW). A Foreign Data 
Wrapper provides access to external objects in form of tabular data: another 
table in another database, external CSV file, etc. 

An Oracle FDW exists and provides access to Oracle databases from PostgreSQL.
Every new FDW features are supported, as well as spatial datas. In practice, we
need to declare `FOREIGN TABLEs` in PostgreSQL schema, linked to remote tables.
Thus, it's easy to querying external data with SQL, in read and write access.

Foreign Data Wrapper's implementation can collect statistics on remote tables,
and transmit predicates to the driver, plus foreign tables can be updated, as 
long as they have a primary key. These capacities are available in Oracle's FDW, 
giving access to Oracle tables in SQL.

References: 

* [Foreign data wrappers](https://wiki.postgresql.org/wiki/Foreign_data_wrappers)
* [CREATE FOREIGN TABLE](https://www.postgresql.org/docs/current/sql-createforeigntable.html)
