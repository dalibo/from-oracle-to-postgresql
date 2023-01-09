---
weight: 6
bookFlatSection: false
title: "PL/SQL specificities"
slug: "pl-sql-specificities"
previouspage: "cursors"
nextpage: "syspackages"
---

## PL/SQL specificities

### Autonomous transactions

We often find `PRAGMA` associated with autonomous transactions in PL/SQL. This
last notion does not exist in PostgreSQL.

It is possible to emulate autonomous transactions through a _dblink_, but it is
a particularly counter-efficient solution that consumes resources. Indeed, the
use of a _dblink_ will cause a new database connection. The cost of a new
connection is significant and a additional connection will require to properly
size the `max_connections` parameter and will consume additional memory and CPU
time.

The following Oracle PL/SQL code shows the declaration of a procedure with 
`AUTONOMOUS_TRANSACTION` pragma:

```sql
CREATE PROCEDURE LOG_ACTION (username VARCHAR2, msg VARCHAR2)
IS
  PRAGMA AUTONOMOUS_TRANSACTION;
BEGIN
  INSERT INTO table_tracking VALUES (username, msg);
  COMMIT;
END log_action;
```

A possible rewrite with a dblink involves filling in the information
to connect to the local PostgreSQL instance:

```sql
CREATE EXTENSION dblink;  
 
CREATE OR REPLACE FUNCTION log_action(username TEXT, msg TEXT)
  RETURNS void AS $$
BEGIN  
    perform dblink_connect('pragma', format(
      'dbname=%s user=test password=test', current_database()
    ));  
    perform dblink_exec('pragma', format(
      'insert into table_tracking values (%s, %s);', username, msg
    ));
    perform dblink_exec('pragma','commit;');  
    perform dblink_disconnect('pragma');
END;  
$$ LANGUAGE plpgsql;
```

Another alternative, which appeared in version 9.5 with _background workers_, is
to rely on the extension [`pg_background`][pg_background] to deport the
execution of a procedure to a new transaction using of a _wrapper_ procedure and
the `pg_background_launch` method of the extension.

[pg_background]: https://github.com/vibhorkum/pg_background

Another rewrite of the previous code would look like this with `pg_background`,
with the creation of a calling procedure:

```sql
-- Create the function we will be "remotely" calling:
CREATE OR REPLACE FUNCTION log_action_atx(username TEXT, msg TEXT)
RETURNS void AS $$
BEGIN
  INSERT INTO table_tracking VALUES (username, msg);
END;
$$ LANGUAGE plpgsql;

-- Now we can write the calling function
CREATE OR REPLACE FUNCTION log_action(username TEXT, msg TEXT)
RETURNS void AS $$
DECLARE
  v_query text;
BEGIN
  v_query := format(
    'SELECT true FROM log_action_atx (%s, %s)', 
    quote_nullable(username), quote_nullable(msg)
  );

  PERFORM pg_background_result(
    pg_background_launch(v_query)
  );
END;
$body$
LANGUAGE plpgsql SECURITY DEFINER;
```

The `log_action` method behaves exactly as if it were executing a
statement in a autonomous transaction.

References:

* [Autonomous transaction support in PostgreSQL](https://blog.dalibo.com/2016/08/19/Autonoumous_transactions_support_in_PostgreSQL.html)

### VARRAY collections

`VARRAY` collections in Oracle _packages_ are migrated to PostgreSQL arrays.
Their definition is created by Ora2Pg, but they usually need a rewrite of the
code using them.

When a `VARRAY` is a simple array of a scalar datatype, there is less rewriting
work to be done than when dealing with a `%ROWTYPE VARRAY`. In this case, there
is quite a lot of work.

The following code:

```sql
DECLARE
   TYPE Calendar IS VARRAY(366) OF DATE;
```

will be converted to: 

```sql
CREATE TYPE calendar AS (date[366]);
```

The next one, though converted by Ora2Pg, won't work without modification: 

```sql
TYPE t_tab_emp IS VARRAY (1000) OF emp%ROWTYPE;
...
tab_emp t_tab_emp;
```

### Associative arrays and nested tables

`TABLE OF` collections are usually used to declare functions returning a set of
records. As a consequence, the `TABLE OF` types can be replaced by a `RETURNS
TABLE` or a `RETURNS SETOF data_type` for simple data types. Please refer to
_PIPELINED attribute and PIPE ROW instruction_ for an example where a `TABLE OF`
collection is not necessary.

Anyway, Ora2Pg translates this data type to an array of the associated type, and
will probably need some rework of the translated code.

Thus, the following declaration:

```sql
CREATE TYPE information IS TABLE OF VARCHAR2(255);
```

will be converted to an array of `varchar(255)` by Ora2Pg: 

```sql
CREATE TYPE information AS VARCHAR(255)[];
```

The `NUMBER INDEX` clause has no equivalent, though. For instance, the following
declaration: 

```sql
TYPE t_list_qlf_id IS TABLE OF NUMBER INDEX BY VARCHAR2(5);
```

cannot be directly converted. One can use the hstore contrib `module`, or the
JSONB datatype, to emulate this feature. 

References:

* [hstore](https://www.postgresql.org/docs/current/hstore.html)
