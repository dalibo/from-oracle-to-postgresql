---
weight: 3
bookFlatSection: false
title: "Control structures"
previouspage: "triggers"
nextpage: "code"
---

## Control structures

Loops and control structures don't require a lot of work, except for the `GOTO`
and `FORALL` instructions.

### FOR REVERSE loop

The `FOR ... REVERSE` has also the peculiarity of needing to revers the min and
max boundaries in `FOR ... IN ... REVERSE min..max`.

For an Oracle code:

```sql
FOR i IN REVERSE 1..10 BY 2 LOOP
  ...
END LOOP;
```

One should convert to:

```sql
FOR i IN REVERSE 10..1 BY 2 LOOP
  ...
END LOOP;
```

Another important difference is `FOR` loops on queries (other than cursors). In
PL/SQL, the target variables are implicitly declared, while they have to be
declared in PostgresQL. One will have to declare those variables in the `DECLARE`
block of the function. This has the advantage of keeping the variable accessible
out of the loop.

Thus, the following PL/SQL extract:

```sql
DECLARE
  v_date1 DATE, v_date2 DATE;
BEGIN
  FOR r IN (
    SELECT numberprgn FROM t2
     WHERE status = 'ordered' AND assigned_dept = 'departt'
  )
LOOP
  SELECT min(datestamp) INTO v_date1
    FROM T1 WHERE NUMBER = r.numberprgn  AND description = 'state1';

  SELECT min(datestamp) INTO v_date2
    FROM T1 WHERE NUMBER = r.numberprgn  AND description = 'state2';
END LOOP;
```

will be converted this way, declaring `v_date1` and `v_date2` as
`timestamp` data type:

```sql
DECLARE
  v_date1 TIMESTAMP, v_date2 TIMESTAMP;
  r record;
BEGIN
  FOR r IN (
    SELECT numberprgn FROM t2
    WHERE status = 'ordered' AND assigned_dept = 'departt'
  )
LOOP
  SELECT min(datestamp) INTO STRICT v_date1
    FROM T1 WHERE number = r.numberprgn  AND description = 'state1';

  SELECT min(datestamp) INTO STRICT v_date2
    FROM T1 WHERE number = r.numberprgn  AND description = 'state2';
END LOOP;
```

References:

* [Simple Loops](https://www.postgresql.org/docs/current/plpgsql-control-structures.html#PLPGSQL-CONTROL-STRUCTURES-LOOPS)
* [Porting from Oracle PL/SQL](https://www.postgresql.org/docs/current/plpgsql-porting.html)

### GOTO instruction

The `GOTO` instruction has no equivalent and imposes a rewriting. Usage of this
instruction is usually not advised, but it is not that seldom used to step out
of a loop or get to the next iteration.

Given this PL/SQL code:

```sql
LOOP
  FETCH current_cursor INTO current;
  EXIT WHEN curs_courant%NOTFOUND;
  IF v_cat = 'YYY' THEN
    v_cat := current;
    GOTO BEGINNING;
  END IF;
  IF current = 'N1' THEN
    v_cat := current;
    GOTO ENDING;
  END IF;
  ...
<<BEGINNING>>
NULL;
END LOOP;
<<ENDING>>
CLOSE current_cursor;
```

should be rewritten this way:

```sql
LOOP
  FETCH current_cursor INTO current;
  IF NOT FOUND THEN
    EXIT;
  END IF;
  IF v_cat = 'YYY' THEN
    v_cat := current;
    CONTINUE;
  END IF;
  IF current = 'N1' THEN
    v_cat := current;
    EXIT;
  END IF;
  ...
END LOOP;
CLOSE current_cursor;
```

### FORALL loops

A `FORALL` loop is used to iterate over the lines of an Oracle collection. As
there are no collections in PostgreSQL, there is no `FORALL` loop either. It's
implementation is simple enough with PostgreSQL, as it is about iterating over
the array replacing the collection.

Thus, the following PL/SQL procedures:

```sql
CREATE OR REPLACE PROCEDURE allEmployees
IS
  TYPE v_array IS varray(50) OF CHAR(14);
  arr_emp v_array;
BEGIN
  SELECT ename BULK COLLECT INTO arr_emp
  FROM emp
  ORDER BY ename;
  FORALL i IN 1..arr_emp.COUNT
    dbms_output.put_line('|name'||arr_emp..(i)||'i'|| i);
END allEmployees;
```

becomes:

```sql
CREATE OR REPLACE FUNCTION allEmployees()
RETURNS VOID AS $body$
DECLARE
  arr_emp CHAR(14)[];
BEGIN
  arr_emp := array ( SELECT ename FROM emp ORDER BY ename);
  FOR i IN array_lower(arr_emp,1)..array_upper(arr_emp,1) LOOP
    RAISE NOTICE '|name % i %', arr_emp[i], i;
  END LOOP;
END;
$body$ LANGUAGE plpgsql;
```