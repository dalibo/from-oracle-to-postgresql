---
weight: 4
bookFlatSection: false
title: "PL/SQL to PL/pgSQL porting"
url: "pl-sql-to-pl-pgsql-porting"
bookCollapseSection: true
previouspage: "transactions"
nextpage: "routines"
---

# PL/SQL to PL/pgSQL porting

## Main differences between PL/SQL and PL/pgSQL

Although Oracle supports both PL/SQL and Java for server programming, and
PotsgreSQL many languages such as PL/Perl, PL/Python, PL/R, PL/Java, etc., this
part is only about porting from PL/SQL to PL/pgSQL.

PostgreSQL's PL/pgSQL has been initially designed as a language similar to
Oracle's PL/SQL. Nevertheless, these are two different languages, and converting
from one to the other takes some work.

First, please note that there is no package in PostgreSQL. A workaround is
needed to emulate them.

Main differences:

* no package;
* compiled at first execution, by session, not globally;
* no procedures in PostgreSQL, only functions;
* no autonomous transaction;
* no directories or similar functionality… PL/PgSQL doesn't handle files.

The last point can easily be circumvented by using one of the other PL languages
that have this capability. Please also note that the language in which a
function is written is of no importance to the caller.

## Packages migration

An Oracle _package_ is the logical grouping of variables and stored procedures in
a namespace. There is no equivalent in PostgreSQL. To simplify the porting,
Ora2Pg will create a schemas with the package names and put functions in their
respective schemas. This will make it possible to use Oracle's `PACKAGE.PROCEDURE`
notation which will become `SCHEMA.FUNCTION`.

Oracle also permits creating functions nested into other functions. PostgreSQL
doesn't accept this with PL/PgSQL. All those functions will have to be extracted
from their parent's function body and declared separately.

There is no global variable in PL/PgSQL. To emulate this, one can use user
variables, which will have to be declared in the `postgresql.conf` configuration
file, at database or user level or directly used by a session with `SET`
instruction.

Theses variables must be prefixed with an arbitary namespace, so as not
to conflict with system configuration. A good practice is to prefix a 
variable with the schema's name, used to emulate packages. The variable 
scope remains local to a session and can not persist or be shared between
several sessions.

For instance, to create an `id_region` variable, one can use the `SET` command:

```sql
SET myschema.id_region = '38';
```

And to retrieve it:

```sql
SELECT current_setting('myschema.id_region') AS id_region;
```

One can also use a the `set_config` function in a PL/PgSQL function. The third
parameter of set_config specifies if the variable is local to the transaction
(`true`) or global (`false`). 

```sql
PERFORM set_config('myschema.id_region', '38', false);
```

To retrieve its value inside a PL/PgSQL block, `current_setting` can be directly
used in assignment or with `SELECT ... INTO` statement.

```sql
a := current_setting('myschema.id_region');
```

Please note that you can also, obviously, use a table for this. Some other PL
languages, such as PL/Perl, have global variables, and may also be an
interesting alternative. 