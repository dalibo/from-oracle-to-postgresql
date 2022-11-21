---
weight: 2
bookFlatSection: false
title: "Triggers conversion"
---

## Triggers conversion

### Structure of a trigger

In Oracle, a trigger declaration embeds the triggers code. In PostgreSQL, a
trigger and a trigger function are two distinct objects: the trigger calls a
trigger function depending on the events it must act upon.

For example, the `print_salary_changes` trigger in Oracle:

```sql
CREATE OR REPLACE TRIGGER print_salary_changes
  BEFORE DELETE OR INSERT OR UPDATE ON emp
  FOR EACH ROW
WHEN (new.empno > 0)
BEGIN
...
END;
```

will be split in two objects during the rewrite:

```sql
CREATE OR REPLACE FUNCTION trigger_fct_print_salary_changes() 
  RETURNS TRIGGER AS $$
BEGIN
...
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE TRIGGER print_salary_changes
  BEFORE DELETE OR INSERT OR UPDATE ON emp
  FOR EACH ROW WHEN (NEW.empno > 0)
     EXECUTE FUNCTION tgr_print_salary_changes();
```

Please note that the trigger function has no parameter and returns a `TRIGGER`
type, specific to a trigger function. The trigger declaration is almost the same
between both RDBMS.

Triggers on statements need some rewriting on their declaration. In Oracle, when
the `FOR EACH ROW` clause is not present, the resulting trigger is a `FOR EACH
STATEMENT`. In PostgreSQL, the default trigger is `FOR EACH ROW`. So `FOR EACH
STATEMENT` triggers have to be converted. Ora2Pg takes care of this. 

This `FOR EACH STATEMENT` trigger in Oracle: 

```sql
CREATE OR REPLACE TRIGGER Log_emp_update
AFTER UPDATE ON Emp_tab
BEGIN
    INSERT INTO Emp_log (Log_date, Action)
        VALUES (SYSDATE, 'Emp_tab COMMISSIONS CHANGED');
END;
```

will be rewritten this way in PostgreSQL:

```sql
CREATE OR REPLACE FUNCTION trigger_fct_log_emp_update() 
  RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO emp_log (log_date, action)
        VALUES (LOCALTIMESTAMP, 'Emp_tab COMMISSIONS CHANGED');
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE TRIGGER log_emp_update
  AFTER UPDATE ON emp_tab
  FOR EACH STATEMENT
     EXECUTE FUNCTION trigger_fct_log_emp_update();
```

PostgresQL accepts the following DML triggers:

* `BEFORE` after `AFTER`;
* `FOR EACH ROW` and `FOR EACH STATEMENT`;
* on `DELETE`, `UPDATE`, `INSERT` (`COPY` is managed by the `INSERT` trigger)
  and `TRUNCATE`;
* the `UPDATE OF nom_colonne_1 [, nom_colonne_2 … ]` clause is also supported;
* conditional, with the `WHEN (condition)` clause.

References:

* [Trigger Functions](https://www.postgresql.org/docs/current/plpgsql-trigger.html)
* [CREATE TRIGGER](https://www.postgresql.org/docs/current/sql-createtrigger.html)

### Trigger return value

PostgreSQL requires that the record is returned in a `BEFORE` trigger. If `NULL`
is returned (or nothing at all), the record is not updated. This is different
from Oracle, where this return value is implicit.

Thus, the following trigger from Oracle: 

```sql
CREATE TRIGGER gen_id FOR produit
  BEFORE INSERT
  DECLARE noitem INTEGER;
As
BEGIN
  SELECT max(no_produit) INTO noitem FROM produit;
  NEW.no_produit := noitem+1;
END;
```

will have to be converted this way: 

```sql
CREATE FUNCTION gen_id () RETURNS TRIGGER AS $$
DECLARE
    noitem INTEGER;
BEGIN 
    SELECT INTO noitem max(no_produit) FROM produit;
    IF noitem ISNULL THEN
      noitem:=0;
    END IF;
    NEW.no_produit:=noitem+1;
    RETURN NEW;
  END;
$$
LANGUAGE 'plpgsql';

CREATE TRIGGER trig_before_ins_produit BEFORE INSERT ON produit 
  FOR EACH ROW 
  EXECUTE PROCEDURE gen_id();
```

### DML Triggers

Those triggers will need some work. First, `:new` and `:old` pseudo-tables in an
Oracle trigger will have to be converted to `NEW` and `OLD` in PostgreSQL.

Finally, the `INSERTING`, `UPDATING`, and `DELETING` operation codes must be
replaced with a test on the `TG_OP` valiable: `TG_OP = 'INSERT'`, `TG_OP =
'UPDATE'` and `TG_OP = 'DELETE'`.

Oracle's following trigger:

```sql
CREATE OR REPLACE TRIGGER testtgop
BEFORE INSERT OR DELETE OR UPDATE
ON emp
FOR EACH ROW
BEGIN
   IF INSERTING THEN
      ...
   ELSIF UPDATING THEN
      ...
   ELSIF DELETING THEN
      ...
   END IF;
END;
```

will be converted this way:

```sql
CREATE OR REPLACE FUNCTION trigger_fct_testtgop() RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    ...
  ELSIF TG_OP = 'UPDATE' THEN
    ...
  ELSIF TG_OP = 'DELETE' THEN
    ...
  END IF;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE TRIGGER testtgop
  BEFORE DELETE OR INSERT OR UPDATE ON emp
  FOR EACH ROW
     EXECUTE PROCEDURE trigger_fct_testtgop();
```

### INSTEAD OF triggers

This trigger type requires the same adaptation as the DML triggers. 

### DDL and event triggers

PostgreSQL has event triggers which are functionally similar to Oracle's DDL
triggers. Unlike regular triggers, which are attached to a single table and
capture only DML events, event triggers are global to a particular database and
are capable of capturing DDL events.

An event trigger fires whenever the event with which it is associated occurs in
the database in which it is defined. Currently, only four events are supported:

* before the execution of a DDL command (CREATE, ALTER, DROP, SECURITY LABEL,
  COMMENT, GRANT or REVOKE);
* after the execution of this same set of commands;
* any operation that drops database objects;
* before a table is rewritten by some actions of the commands ALTER TABLE and
  ALTER TYPE.

Event triggers require a complete rewrite in any procedural language that
includes event trigger support, or in C, but not in plain SQL.

References:

* [Event triggers](https://www.postgresql.org/docs/current/event-triggers.html)
