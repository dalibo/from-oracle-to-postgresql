---
weight: 4
bookFlatSection: false
title: "PL/SQL code conversion"
slug: "pl-sql-code-conversion"
---

## PL/SQL code conversion

### Empty strings and NULL values

For Oracle, an empty string is `NULL` too. PostgreSQL makes a difference: `IS
NULL` and empty are different.

Some queries working with Oracle may not work as expected if directly copied.
The most frequent problems are comparing a column with an empty string, and
concatenating with a `NULL`. Please refer to what has already been exposed in
the previous chapter about porting queries.

When PL/SQL code is ported to PL/pgSQL, one has to mind the calling code too:
if the application's code hasn't converted `''` to `NULL`, then one has to be
careful about when the function receives an empty string in place of a `NULL`
value.

Here are some examples illustrating this in Oracle:

```sql
IF vidfee IS NULL
  THEN
    ...
```

```sql
IF vidfee IS NOT NULL
  THEN
    ...
```

For Oracle `vidfee` will be `NULL` if it is an empty string **or** if it is
`NULL`, and it won't make any difference. If this calling code accesses a
function with such tests, but passing empty strings, it will work with Oracle.

It will fail with Postgresql, as it is much stricter in this regard. You'll have
to modify the `IF` tests in the converted PL/pgSQL function, to avoid a rewrite
in the calling function.

Here is an example of converted code (first test):

```sql
IF COALESCE(vidfee, '') = ''
  THEN
    ...
```

In this example, if `vidfee` is `NULL`, `coalesce` will return `''`, which is
what we want here.

The second test (`IS NOT NULL`) is a bit less intuitive:

```sql
IF (vidfee IS NOT NULL AND vidfee <> '')
  THEN
    ...
```

One can also start the function with a simple hack, which will greatly simplify
the rest of the tests. This is not applicable to SQL though, only to PL/SQL.

```sql
IF vidfee = '' THEN vidfee := NULL END;
```

Ora2Pg does these code conversions automatically by default, the directive
`NULL_EQUAL_EMPTY` disables this behavior.

### Queries and functions execution

When a `SELECT` with no `INTO` clause exists, it has to be replaced by
`PERFORM`. This is also done automatically by Ora2Pg.

Thus, the following extract from a PL/SQL procedure:

```sql
BEGIN
  SELECT ename, sal FROM EMP
   WHERE empno=7902 FOR UPDATE;
  ...
END;
```

will be converted to:

```sql
BEGIN
  PERFORM ename, sal FROM EMP
    WHERE empno=7902 FOR UPDATE;
  ...
END;
```

In addition, the `EXEC` instruction used to retrieve into a variable the return
code from a PL/SQL function doesn't exist in PostgreSQL. This has to be
rewritten by using `SELECT INTO` This is performed by Ora2Pg.

The following PL/SQL extract:

```sql
EXEC :a := get_version();
```

will be converted to:

```sql
SELECT get_version() INTO a;
```

### Dynamic queries execution

To execute a dynamically built query, Oracle provides an `EXECUTE IMMEDIATE`
clause. In PostgreSQL, the `IMMEDIATE` must be removed, as it in not supported.
Indeed, an `EXECUTE` is always executed immediately. This is performed
automatically by Ora2Pg.

Furthermore, one should use `quote_literal`, `quote_nullable` and `quote_ident`
to build a dynamic SQL query in PostgreSQL. This avoids SQL injection. This
isn't performed by Ora2Pg.

For instance, the following SQL code:

```sql
sql_stmt := 'UPDATE employees SET salary = salary + :1 WHERE '
            || v_column || ' = :2';
EXECUTE IMMEDIATE sql_stmt USING amount, column_value;
```

should be converted this way (note the `:` replaced by `$`):

```sql
sql_stmt := 'UPDATE employees SET salary = salary + $1 WHERE '
            || quote_literal(v_column) || ' = $2';
EXECUTE sql_stmt USING amount, column_value;
```

References:

* [Executing Dynamic Commands](https://www.postgresql.org/docs/current/plpgsql-statements.html#PLPGSQL-STATEMENTS-EXECUTING-DYN)

### COMMIT in a routine

There is no way with PostgreSQL to `COMMIT` or `ROLLBACK` instructions inside a
function or even in a procedure called by a function.

In these cases, it is advisable to perform transaction management in higher
level code.

Note that, due to MVCC implementations, PostgreSQL can support long transactions
more easily than Oracle. Also, some intermediate `COMMIT` statements can often
be purely and simply deleted.

### Exceptions management

Exceptions' handling is quite different between Oracle and PostgreSQL. First,
Oracle's `SQLCODE` is almost equivalent to PostgreSQL's `SQLSTATE`. One has thus to
be replaced by the other, which is performed by Ora2Pg.

But the most notable difference is the way the error is handled. If an error is
triggered in a PL/SQL block, only the triggering statement is rollbacked. As a
consequence, one often sees savepoints declared at the beginning of the block
and `ROLLBACK TO SAVEPOINT` statements issued in the exception block.

PostgreSQL's way of handling exceptions is different. When an error is trigger
in a PL/pgSQL block, the whole block is rollbacked by the exception. PL/SQL
constructs using `SAVEPOINTS` with thus be simply ported, by removing the
`ROLLBACK TO SAVEPOINT` instructions.

The following PL/SQL code:

```sql
BEGIN
  SAVEPOINT s1;
  ...
EXCEPTION
  WHEN ... THEN
    ROLLBACK TO s1;
    ...
  WHEN ... THEN
    ROLLBACK TO s1;
    ...
END;
```

will be simplified this way:

```sql
BEGIN
  ...
EXCEPTION
  WHEN ... THEN
    ...
  WHEN ... THEN
    ...
END;
```

Some exceptions' names will also have to be replaced. Here are the errors that
will have to be replaced:

| Oracle           | PostgreSQL           |
|------------------|----------------------|
| STORAGE_ERROR    | OUT_OF_MEMORY        |
| ZERO_DIVIDE      | DIVISION_BY_ZERO     |
| INVALID_CURSOR   | INVALID_CURSOR_STATE |
| dup_val_on_index | unique_violation     |

Finally, Oracle's `raise_application_error` will be converted to `RAISE
EXCEPTION`. The following code from Oracle:

```sql
raise_application_error(
  -20000,
  'Unable to create a new job: a job is currently running.'
);
```

must be converted this way: 

```sql
RAISE EXCEPTION
  'Unable to create a new job: a job is currently running';
```

References:

* [PostgreSQL Error Codes](https://www.postgresql.org/docs/current/errcodes-appendix.html)
* [Implicit Rollback after Exceptions](https://www.postgresql.org/docs/current/plpgsql-porting.html#PLPGSQL-PORTING-EXCEPTIONS)

### SELECT INTO

The `SELECT ... INTO ...` statement will need adaptation to behave with
PostgreSQL the same way it does with Oracle. PostgreSQL's `STRICT` option is
Oracle's default behavior. One will have to add the `STRICT` keyword after
`INTO` when an exception on `NO_DATA_FOUND` or `TOO_MANY_ROWS` exists in the
same code block.

Thus, this procedure in PL/SQL:

```sql
BEGIN
  SELECT feegroupid INTO vsalegroup
    FROM feegroup
   WHERE feeclass = 'V'
     AND isdefault = 1
     AND ispublic = vispublic;
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    vidartpxvte := '-040';
    RETURN vidartpxvte;
  WHEN TOO_MANY_ROWS THEN
    vidartpxvte := '-045';
    RETURN vidartpxvte;
END;
```

has to be converted this way for PostgreSQL: 

```sql
BEGIN
  SELECT feegroupid INTO STRICT vsalegroup
    FROM feegroup
   WHERE feeclass = 'V'
     AND isdefault = 1
     AND ispublic = vispublic;
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    vidartpxvte := '-040';
    RETURN vidartpxvte;
  WHEN TOO_MANY_ROWS THEN
    vidartpxvte := '-045';
    RETURN vidartpxvte;
END;
```

References:

* [Executing a Command with a Single-Row Result](https://www.postgresql.org/docs/current/plpgsql-statements.html#PLPGSQL-STATEMENTS-SQL-ONEROW)

### BULK COLLECT

There is no `BULK COLLECT` in PostgreSQL. This is used in Oracle to load the
content of a query in an array, and then iterate over this array.

For example, this Oracle code:

```sql
CREATE PROCEDURE AllAuthors
IS
  TYPE my_array IS varray(100) OF VARCHAR(25);
  temp_arr my_array;
BEGIN
  SELECT author_name BULK COLLECT INTO temp_arr 
    FROM authors ORDER BY author_name;

  FOR i IN temp_arr.first .. temp_arr.last LOOP
    DBMS_OUTPUT.put_line(i || ') author_name: ' || temp_arr..(i));
  END LOOP;
END AllAuthors;
```

can be translated to: 

```sql
CREATE FUNCTION AllAuthors() RETURNS VOID
AS $$
DECLARE
  temp_arr VARCHAR(25)[];
BEGIN
  temp_arr := (SELECT author_name FROM authors ORDER BY author_name);

  FOR i IN array_lower(temp_arr,1) .. array_upper(temp_arr,1) LOOP
    RAISE NOTICE '% ) author_name: %', i,  temp_arr..(i);
  END LOOP;
END;
$$ LANGUAGE plpgsql;
```

### instr function

PostgreSQL's documentation proposes a PL/PgSQL implementation of Oracle's `instr`
in its chapter about [porting from Oracle PL/SQL][plpgsql-porting].

[plpgsql-porting]: https://www.postgresql.org/docs/current/plpgsql-porting.html#PLPGSQL-PORTING-APPENDIX