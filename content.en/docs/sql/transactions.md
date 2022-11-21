---
weight: 5
bookFlatSection: false
title: "Transaction management"
---

## Transaction management

Transactions and locks are very similar between Oracle and PostgreSQL. There are
two major differences. First, Oracle implicitely starts a new transaction when a
statement is run and keeps it running until COMMIT, while PostgreSQL is using
`autocommit` by default. A transaction must be explicitely started with `BEGIN`.

The other difference is the way MVCC is implemented in both databases.
PostgreSQL's version has the benefit that a `ROLLBACK` is instantaneous. In
return, the modified blocs of a rollbacked transaction will be physically
present in, as PostgreSQL will have created new versions of updated lines,
although they have been cancelled. This space will have to be reclaimed
afterwards by `VACUUM`.

The BEGIN statement has several synonyms:

  * `BEGIN`;
  * `BEGIN WORK`;
  * `BEGIN TRANSACTION`;
  * `START TRANSACTION`.

References:

* [BEGIN](https://www.postgresql.org/docs/current/sql-begin.html)
* [START TRANSACTION](https://www.postgresql.org/docs/current/sql-start-transaction.html)

### Isolation Level

One can specify an isolation level by specifying it in the beginning statement
of a transaction: 

```sql
BEGIN [ WORK | TRANSACTION ] [ mode_transaction [, ...] ]
```

where `transaction_mode` is:

```
ISOLATION LEVEL 
  {SERIALIZABLE | REPEATABLE READ | READ COMMITTED | READ UNCOMMITTED }
READ WRITE | READ ONLY
[ NOT ] DEFERRABLE
```

`READ UNCOMMITTED` is a synonym of `READ COMMITTED` under PostgreSQL and Oracle:
MVCC engines don't need the `READ UNCOMMITTED` mode, as writers and readers dont
block each other.

Furthermore, Oracle and PostgreSQL both implement the `SERIALIZABLE` level.
PostgreSQL and Oracle implement this level with optimistic locking, in order to
improve transactional throughput. Most RDBMS implement this level through
pessimistic locking, severely impairing throughput.

With Oracle, one can set the isolation level of all future transactions at the
session level. This is done with this statement:

```sql
ALTER SESSION SET ISOLATION LEVEL ...;
```

This is done inside a transaction block with PostgreSQL: 

```sql
BEGIN TRANSACTION ISOLATION LEVEL ...;
...
COMMIT;
```

PostgreSQL's way of determining serialization failures is a bit stricter than
Oracle's, but the two modes are very similar. For instance, the example given in
table 7 of this document ["On Transaction Isolation Levels"][asktom] generates a
serialization error in PostgreSQL, while Oracle doesn't catch it. 

[asktom]: https://asktom.oracle.com/Misc/oramag/on-transaction-isolation-levels.html

```sql
create table a ( x int );
create table b ( x int );
```

| Time | Session #1            | Session #2
|------|-----------------------|---------------------------
| t1   | `ALTER SESSION SET isolation_level=serializable;` | |
| t2   | | `ALTER SESSION SET isolation_level=serializable;` |
| t3   | `INSERT INTO a SELECT count(*) FROM b;` | |
| t4   | | `INSERT INTO b SELECT count(*) FROM a;` |
| t5   | `COMMIT;` | |
| t6   | | `COMMIT;` |

PostgreSQL does not allow session #2 to commit its changes due to snapshot
violation. A error message is thrown to user and suggest to retry the complete
transaction block to succeed.

```
ERROR: could not serialize access due to read/write dependencies among transactions
DETAIL: Reason code: Canceled on identification as a pivot, during commit attempt.
HINT: The transaction might succeed if retried.
```

References:

* [Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
* [SET TRANSACTION](https://www.postgresql.org/docs/current/sql-set-transaction.html)

### Constraint checking

Integrity constraints are checked on every modification, whether or not it is
executed in a transaction. To require compliance with these constraints within a
complex transaction, it is possible to defer these verifications at the time of
the `COMMIT`.

Oracle and PostgreSQL provide the same SQL syntax for defining whether a
constraint is always deferred:

```sql
ALTER TABLE ... CONSTRAINT ...
  [NOT] DEFERRABLE INITIALLY 
    (IMMEDIATE | DEFERRED)
```

It is also possible to disable checking within an ongoing transaction using the
following statement, common to both systems:

```sql
SET CONSTRAINT (cons_name | ALL) DEFERRED;
```

Finally, it may be necessary to define at a session level that all future
transactions are deferred by default. In this case, the commands between Oracle
and PostgreSQL have differences.

With Oracle, the following syntax should be changed from:

```sql
ALTER SESSION SET CONSTRAINTS = DEFERRED;
ALTER SESSION SET CONSTRAINTS = IMMEDIATE;
```

To:

```sql
SET default_transaction_deferrable = ON;
SET default_transaction_deferrable = OFF;
```

* [SET CONSTRAINTS](https://www.postgresql.org/docs/current/sql-set-constraints.html)

### SAVEPOINT

`SAVEPOINTS` work the same way in Oracle and PostgreSQL. Locks acquired before a
`SAVEPOINT` aren't released if a `SAVEPOINT` is released by a `RELEASE
SAVEPOINT` or a `ROLLBACK TO SAVEPOINT`. 

PostgreSQL's documentation warns against the modification of lines after a
`SAVEPOINT` has been put if those lines have been locked with a `SELECT ... FOR
UPDATE` before the `SAVEPOINT`. Indeed, the lock acquired by the `SELECT ... FOR
UPDATE` may be released during the `ROLLBACK TO SAVEPOINT`. The following sequence
of SQL statements should thus be avoided:

```sql
BEGIN;
  SELECT * FROM my_table WHERE key = 1 FOR UPDATE;
  SAVEPOINT s;
  UPDATE my_table SET ... WHERE key = 1;
ROLLBACK TO SAVEPOINT s;
```

References:

* [SAVEPOINT](https://www.postgresql.org/docs/current/sql-savepoint.html)
* [RELEASE SAVEPOINT](https://www.postgresql.org/docs/current/sql-release-savepoint.html)
* [ROLLBACK TO SAVEPOINT](https://www.postgresql.org/docs/current/sql-rollback-to.html)

### Lock management

Although PostgreSQL and Oracle are very similar as far as locking is concerned,
some subtle differences have to be taken into account. 

References:

* [Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)

#### Implicit locking

DML statements acquire implicit locks. The most notable difference between
Oracle and PostgreSQL is the `SELECT` statement: Oracle acquires no lock, while
PostgreSQL takes an `ACCESS SHARE` lock. As a consequence, Oracle doesn't
protect readers from operations such as a table drop. A `SELECT` can be
interrupted following a `DROP TABLE` in another session. PostgreSQL's locking
prevents that. 

`INSERT`, `UPDATE` and `DELETE` lock the modified lines, in the same way as
Oracle: directly in the record, and not in memory. 

References:

* [Dropping a table during SELECT](http://uhesse.com/2009/10/27/dropping-a-table-during-select/), blog post from Uwe Hesse

#### Explicit locking

**SELECT FOR UPDATE**

`SELECT FOR UPDATE` statements may need modification. Oracle's syntax is indeed
richer than PostreSQL's as far as this statement is concerned.

* Oracle's syntax accepts both `WAIT` and `NOWAIT`. PostgreSQL accepts only
  `NOWAIT`, `WAIT` being the default behaviour. SELECT … FOR UPDATE WAIT becomes
  `SELECT … FOR UPDATE`.

* Oracle's `OF` clause is incompatible with PostgreSQL's. This is used to
  specify the table to be locked for the coming update. The difference is the
  Oracle's `OF` clause specify a column, while PostgreSQL's specifies a table.

References:

* [SELECT FOR UPDATE](https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE)

**LOCK TABLE**

Oracle's `LOCK TABLE` syntax is compatible with PostgreSQL's in most cases. All
of Oracle's locking modes exist in PostgreSQL, and PostgreSQL has a few more of
them.

As for the `SELECT FOR UPDATE` statement, Oracle proposes both `WAIT` and
`NOWAIT`, while PostgreSQL only has `NOWAIT`, `WAIT` being the default.

Oracle's `PARTITION` and `SUBPARTITION` clauses cannot be converted though. If
the target database also uses partitioning, the child table must be targeted for
the locking. 

References:

* [LOCK TABLE](https://www.postgresql.org/docs/current/sql-lock.html)
