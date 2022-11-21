---
weight: 5
bookFlatSection: false
title: "Cursors"
---

## Cursors

The _cursors_ variable's notation is different in Oracle and PostgreSQL.

### Cursors declaration

With Oracle, a cursor is declared this way: `CURSOR mycursor`. This has to be
reverted with PostgreSQL: `mycursor CURSOR`. This is performed by Ora2Pg.

Oracle's `REF CURSOR` and `SYS_REFCURSOR` also have to be modified, both to
`REFCURSOR` for PostgreSQL.

Oracle uses the `IN` keyword to pass parameters to the cursor. This is not
necessary with PostgreSQL, it just has to be removed.

The following declaration, with Oracle:

```sql
CURSOR command_lines_cursor(cd_num IN VARCHAR2) IS
  SELECT * FROM command_lines
   WHERE cd_number = cd_num;
```

should be converted this way:

```sql
command_lines_cursor CURSOR (cd_num VARCHAR) FOR
  SELECT * FROM command_lines
   WHERE cd_number = cd_num;
```

References:

* [Record Types](https://www.postgresql.org/docs/current/plpgsql-declarations.html#PLPGSQL-DECLARATION-RECORDS)

### Returning a cursor

A function can return a cursor with PostgreSQL, as it does with Oracle. Oracle's
return data type is `my_table%ROWTYPE`, while PostgreSQL's is `REFCURSOR`.

For instance, with Oracle, returning a cursor will be declared this way:

```sql
TYPE return_cur IS REF CURSOR RETURN my_table%ROWTYPE;
p_retcur return_cur;
```

Whereas with PostgreSQL, the declaration will be this one:

```sql
return_cur REFCURSOR;
```

### Cursor exit

The exit code of a cursor loop has to be modified. Oracle's construct `EXIT WHEN
...%NOTFOUND` is not accepted by PostgreSQL. It has to be replaced by this kind
of construct: `IF NOT FOUND THEN EXIT; END IF;`. The `SQL%NOTFOUND` also has to
be replaced by `NOT FOUND`. Both these transformations are performed by Ora2Pg.

The following PL/SQL extract:

```sql
LOOP
  FETCH c1 INTO my_ename, my_sal, my_hiredate;
  EXIT WHEN c1%NOTFOUND;
  ...
END LOOP;
```

will be converted this way:

```sql
LOOP
  FETCH c1 INTO my_ename, my_sal, my_hiredate;
  IF NOT FOUND THEN
    EXIT;
  END IF;
  ...
END LOOP;
```
