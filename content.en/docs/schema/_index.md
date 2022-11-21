---
weight: 2
bookFlatSection: false
title: "Porting database objects"
url: "porting-the-database-schema-to-postgresql"
bookCollapseSection: true
---

# Porting the database schema to PostgreSQL

## DDL compatibility

The majority of DDL commands are compatible between Oracle and PostgreSQL, 
expect the storage clauses. There are however some fundamental differences 
between each RDBMS:

* Oracle does an implicit COMMIT after each DDL command;
* on the contrary, all DDL queries are transnational in PostgreSQL;
* non-blocking DDL are available in Oracle, however PostgreSQL only provides 
  `CREATE INDEX CONCURRENTLY` and `DROP INDEX CONCURRENTLY`. Much effort is 
  also put on PostgreSQ to require minimum locking and rewriting when doing 
  schema changes.

```sql
BEGIN;

CREATE TABLE mytable (a INTEGER PRIMARY KEY, b INTEGER);
-- NOTICE:  CREATE TABLE / PRIMARY KEY will create implicit index "mytable_pkey"
--  for TABLE "mytable"
-- CREATE TABLE

SELECT count(*) FROM mytable;
-- -[ RECORD 1 ]
-- count | 0

ROLLBACK;

SELECT count(*) FROM mytable;
-- ERROR:  relation "mytable" does not exist
-- LINE 1: SELECT count(*) FROM mytable;
```

## Constraint migration

Migrating integrity constraints is of no special difficulty. 

## Synonyms migration

There is no equivalent to Oracle's synonyms in PostgreSQL. 

In case of namesake, a simple way is to set up PostgreSQL's `search_path`
session variable, which specifies the list of schemas to be searched when an 
object name has no schema qualification. This can be specified for each user, 
for a database, or dynamically whenever wanted with the 
`SET search_path = <path list>;` statement. 

Otherwise, synonyms are often replaced by views, in order to expose a table in
another schema with another name, or by using a wrapper function to call an
invisibile function in another schema.

References:

* [The Schema Search Path](https://www.postgresql.org/docs/current/ddl-schemas.html#DDL-SCHEMAS-PATH)