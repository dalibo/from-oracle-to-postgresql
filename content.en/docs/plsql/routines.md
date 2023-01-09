---
weight: 1
bookFlatSection: false
title: "Porting procedures and functions"
previouspage: "/plsql"
nextpage: "triggers"
---

## Porting procedures and functions

### Functions declaration

Both of Oracle and PostgreSQL allow functions and procedures creation, also
called _routines_ in PostgreSQL's documentation. Procedures can be invoked by
`CALL` word, while functions are called by `SELECT` or `PERFORM`.

Procedures with PostgreSQL do not return any result. All procedures declared 
with `OUT` parameter need to be converted to functions in PostgreSQL.

Please note, amongst the main differences:

* the `RETURN` keyword becomes `RETURNS`;
* the `IS` keyword becomes `AS`;
* the body of a function is delimited by `$$` (or similar) beacons;
* repeating the function's name after the final END keyword is useless;
* local variables are put in a `DECLARE` block in PostgreSQL;
* the language has to be specified in PostgreSQL: `LANGUAGE plpgsql`.

Below, the declaration of the `cs_create_job` in Oracle: 

```sql
CREATE OR REPLACE PROCEDURE cs_create_job(v_job_id IN INTEGER) IS
    a_running_job_count INTEGER;
BEGIN
...
END;
```

It will be rewritten this way in PostgreSQL:

```sql
CREATE OR REPLACE PROCEDURE cs_create_job(v_job_id INTEGER) AS
$$
DECLARE
  a_running_job_count INTEGER;
BEGIN
...
END;
$$ LANGUAGE plpgsql;
```

References:

* [Structure of PL/pgSQL](https://www.postgresql.org/docs/current/plpgsql-structure.html)
* [Declarations](https://www.postgresql.org/docs/current/plpgsql-declarations.html)

### DETERMINISTIC attribute

Oracle functions declared as `DETERMINISTIC` will be converted to PostgreSQL
functions with an `IMMUTABLE` attribute. `IMMUTABLE` and `DETERMINISTIC`
indicate that the function doesn't access the database, and that it will return
the same result given the same parameters. This is supported by Ora2Pg. 

Thus, the following declaration for Oracle:

```sql
CREATE OR REPLACE FUNCTION get_average_char(input_ VARCHAR2)
   RETURN VARCHAR2 DETERMINISTIC
IS
...
END get_average_char;
```

becomes, for PostgreSQL: 

```sql
CREATE OR REPLACE FUNCTION get_average_char(input_ VARCHAR)
   RETURNS VARCHAR
AS $$
...
END;
$$ LANGUAGE plpgsql
IMMUTABLE;
```

References:

* [CREATE FUNCTION](https://www.postgresql.org/docs/current/sql-createfunction.html)

### PIPELINED attribute and PIPE ROW instruction

Oracle's functions declared as `PIPELINED` are equivalent to _Set-Returning
Functions_ (SRF) in PosgreSQL. So, a `PIPELINED` function will be converted to a
`SETOF` function. The `PIPE ROW` instruction will be converted to `RETURN NEXT`.

The following package, extracted from a [PL/SQL example][LNPLS918] from Oracle
uses the `PIPELINED` attribute and the `PIPE ROW` instruction: 

[LNPLS918]: https://docs.oracle.com/cd/E11882_01/appdev.112/e25519/tuning.htm#LNPLS918

```sql
CREATE OR REPLACE PACKAGE pkg1 AS
  TYPE numset_t IS TABLE OF NUMBER;
  FUNCTION f1(x NUMBER) RETURN numset_t PIPELINED;
END pkg1;

CREATE PACKAGE BODY pkg1 AS
  -- FUNCTION f1 RETURNS a collection of elements (1,2,3,... x)
  FUNCTION f1(x NUMBER) RETURN numset_t PIPELINED IS
  BEGIN
    FOR i IN 1..x LOOP
      PIPE ROW(i);
    END LOOP;
    RETURN;
  END f1;
END pkg1;
```

It will be ported quite easily by Ora2Pg:

* the `numset_t` type isn't converted, it will have to be modified manually
  after the Ora2Pg run;
* if the return type was `%ROWTYPE` in Oracle, it would have been ported to `SET
  OF record`;
* the pkg1 package is converted to a pkg1 schema;
* the f1 function is converted as follows.

Here is the function, converted to PL/pgsql: 

```sql
CREATE SCHEMA pkg1;
CREATE OR REPLACE FUNCTION pkg1.f1 (x INTEGER) RETURNS SET OF INTEGER AS $$
DECLARE
  i INTEGER;
BEGIN
  FOR i IN 1..x LOOP
    RETURN NEXT i;
  END LOOP;
  RETURN;
END;
$$ LANGUAGE plpgsql;
```

This function will be call this way: `SELECT pkg1.f1(10);`. 

References:

* [Control Structures](https://www.postgresql.org/docs/current/plpgsql-control-structures.html)

### Functions parameters

Oracle doesn't conform completely to the SQL standard regarding to the
parameters declaration of a function or a procedure. PostgreSQL, though, adheres
to the standard. Minor adaptations are, a consequence, necessary for all
functions declarations.

Each argument is named, has a data type, and a mode determining the behavior of
the parameter In Oracle, the parameter mode is declared after its name. In
PostgreSQL, the parameter mode comes before its name, as specified by the
standard. The `IN` and `OUT` modes are directly transposed, IN `OUT` becomes
`INOUT`. The parameter mode can be completely ignored for `IN`.

Thus, the following parameters declaration in Oracle: 

```sql
CREATE FUNCTION test (p INTEGER) ...
CREATE OR REPLACE PROCEDURE cs_parse_url(
  v_url IN VARCHAR, v_host IN OUT VARCHAR, 
  v_path OUT VARCHAR, v_query OUT VARCHAR
) ...
```

become in PostgreSQL:

```sql
CREATE FUNCTION test (p INTEGER) ...
CREATE OR REPLACE FUNCTION cs_parse_url(
  IN v_url VARCHAR, INOUT v_host VARCHAR, 
  OUT v_path VARCHAR, OUT v_query VARCHAR
) ...
```

References:

* [SQL Functions with Output Parameters](https://www.postgresql.org/docs/current/xfunc-sql.html#XFUNC-OUTPUT-PARAMETERS)

### Default value for a parameter

PostgreSQL accept default values for arguments. Functions declarations using
default values can be copied without modification. 

For instance, a function such as this one in Oracle:

```sql
CREATE FUNCTION fonction1 (a INT, b INT, c INT := 0) ...
```

will be rewritten this way in PostgreSQL: 

```sql
CREATE FUNCTION fonction1 (a INT, b INT, c INT = 0) ...
```

The `DEFAULT` keyword is valid in both languages.

References:

* [CREATE FUNCTION](https://www.postgresql.org/docs/current/sql-createfunction.html)
