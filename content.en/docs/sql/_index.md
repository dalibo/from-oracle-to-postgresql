---
weight: 3
bookFlatSection: false
title: "Porting SQL queries"
url: "porting-sql-queries"
bookCollapseSection: true
---

# Porting SQL queries

## DML statements compatibility

Most DML statements are compatible between Oracle and PostgreSQL. Note though 
that the `MERGE` statement doesn't exist yet in PostgreSQL. Some restrictions 
exist for some window functions and recursive Common Table Expressions (CTE).

## Subquery aliases

Oracle does not handle error messages while subqueries are unaliased unlike
PostgreSQL. For instance, this query need to be rewritten:

```sql
SELECT <columns, ...>
  FROM (SELECT <...>
       )
 WHERE <conditions>
```

as:

```sql
SELECT <columns, ...>
  FROM (SELECT <...>
       ) sub1
 WHERE <conditions>
```

## Implicit conversions

Many implicit conversion to and from a text type have been removed since 
PostgreSQL 8.3. 

For instance, one cannot write this type of query:

```sql
CREATE TABLE depts ( number CHAR(2), dname VARCHAR(25) );

SELECT * FROM depts WHERE number BETWEEN 0 AND 42;
-- ERROR:  operator does not exist: character >= INTEGER
-- LIGNE 1 : SELECT * FROM depts WHERE number BETWEEN 0 AND 42;
```

In order to run this query, one has to explicitly declare the conversion to be 
performed:

```sql
SELECT * FROM depts WHERE number::INTEGER BETWEEN 0 AND 42;
-- or (more SQL compliant)
SELECT * FROM depts WHERE CAST(id AS INTEGER) BETWEEN 0 AND 42;
```

With Oracle, this conversion is implicit (but will mask a potential performance 
problem as it will force a type conversion). 

## HAVING and GROUP BY clauses

Even though Oracle's documentation stipulates that the `GROUP BY` clause precedes 
the `HAVING` clause, Oracle's grammar permits the opposite. Those queries having 
`HAVING` before `GROUP BY` will have to be corrected. 

This query, for example: 

```sql
SELECT * FROM test HAVING count(*) > 3 GROUP BY i;
```

will have to be converted this way to be run in PostgreSQL: 

```sql
SELECT * FROM test GROUP BY i HAVING count(*) > 3;
```

References:

* [The GROUP BY and HAVING Clauses](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-GROUP)

## ROWID

In very rare occasions, some SQL queries use Oracle's `ROWID` column, for instance
to deduplicate records. The `ROWID` column is the physical location of a record in 
a table. PostgreSQL's equivalent is `ctid`. 

More specifically, Oracle's `ROWID` is a record's logical address, in the 
`OOOOOO.FFF.BBBBBB.RRR` form, where `O` is the object's number, `F` the file, `B` the 
block's number, and `R` the row's number in this block. The format may vary if 
the table is in a `BIG FILE TABLESPACE`, but the principle stays the same. 

PostgreSQL's `ctid` contains only the block number and the row's number in this 
block. There is no other localisation information. The `ctid` is only unique 
inside a table. Because of that, a query returning `ctid`s from a partitioned table
may have duplicated. In this peculiar case, one can add the `tableoid` column 
(the table's id in the catalog), which will distinguish between duplicates 
coming from different partitions. 

Another difference is that an update doesn't change a record's rowid in Oracle, 
while it does change the ctid in PostgreSQL.

This access method should be avoided as much as possible.

References:

* [System Columns](https://www.postgresql.org/docs/current/ddl-system-columns.html)


## Minus operator conversion

The `MINUS` operator has to be converted to `EXCEPT` for PostgreSQL. The other
set operations `UNION`, `UNION ALL`, and `INTERSECT` don't need any conversion. 

Thus, the following query returns all inventory products which have never been
in a command. It can be written like this for Oracle: 

```sql
SELECT product_id FROM inventories
MINUS
SELECT product_id FROM order_items
ORDER BY product_id;
```

It will be written this way for PostgreSQL: 

```sql
SELECT product_id FROM inventories
EXCEPT
SELECT product_id FROM order_items
ORDER BY product_id;
```

References:

* [Combining Queries (UNION, INTERSECT, EXCEPT)](http://www.postgresql.org/docs/current/static/queries-union.html)

## Window Functions

The queries using window functions don't usually need a lot of work. 

Oracle proposes an `ORDER SIBLINGS BY` clause. The `SIBLINGS` keyword has no
equivalent and is anyway only used for hierarchy processing, with `CONNECT BY`.
This kind of query has to be rewritten anyway.

The `PARTITION BY` don't need any adaptation, neither the windowing clause
(`RANGE ...` or `ROWS ...`). 

Most of Oracle's general usage window functions exist in PostgreSQL. Some
functions still have no equivalent for the moment.

* `RATIO_TO_REPORT` will need to be rewritten (using a division on the sum on
  the window).
* `LISTAGG` function will need to be rewritten using `array_agg` and
  `array_to_string`.

Please note though that PostgreSQL's extension capabilities make it possible to
write new aggregate and window functions, making it possible to write the
missing functions. 

References:

* [Window Functions](https://www.postgresql.org/docs/current/functions-window.html)
* [Tutorial Window Functions](https://www.postgresql.org/docs/current/tutorial-window.html)
* [Window Function Calls](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS)
* [Aggregate Functions](https://www.postgresql.org/docs/current/functions-aggregate.html)

## Recursive CTE

Oracle's grammar makes no difference between a standard CTE and a recursive CTE,
while PostgreSQL does and requires a correct usage of `WITH` and `WITH
RECURSIVE`. One will need to correct recursive CTE, which should include a
`UNION ALL` and a reference to the common table expression itself.

The following query does a very simple recursion: 

```sql
WITH recursion (a) AS (
  SELECT 1 AS a
    FROM dual
UNION ALL
  SELECT a + 1
    FROM recursion
   WHERE a < 10
)
SELECT * FROM recursion;
```

Here it is, rewritten for PostgreSQL:

```sql
WITH RECURSIVE recursion (a) AS (
  SELECT 1 AS a
UNION ALL
  SELECT a + 1
    FROM recursion
   WHERE a < 10
)
SELECT * FROM recursion;
```

The `WITH` clause also proposes some extensions in Oracle, to describe how to
perform the recursion (_search_clause_) and the loop detection (_cycle_clause_).
Cycle detection has already been tackled in "[Hierarchical querying]({{< relref 
"/docs/sql/hierarchies" >}}) " section of this document. 

References:

* [SELECT](https://www.postgresql.org/docs/current/sql-select.html)
* [Cycle Detection](https://www.postgresql.org/docs/current/queries-with.html#QUERIES-WITH-CYCLE)

## MERGE

`MERGE` command allows to make insertions or updates of tables depending on
whether the rows already exist or not. The typical syntax for the statement
looks like this:

```sql
MERGE INTO destination_table
  USING source_table ON (condition)
  WHEN MATCHED THEN update_clause
  WHEN NOT MATCHED THEN insert_clause;
```

But versions prior to PostgreSQL 15 do not support the syntax `MERGE` from the
SQL standard. On the other hand, we can emulate this instruction with a `INSERT`.
It is necessary to distinguish here several cases, according to the presence or
not of `WHEN MATCHED` and `WHEN NOT MATCHED` clauses.

If there is no `WHEN NOT MATCHED` clause, a simple `UPDATE` is required:

```sql
UPDATE destination_table
  update_clause
  USING source_table
  WHERE join_condition;
```

If there is no `WHEN MATCHED` clause, using an `INSERT` on non existing rows is
more relevant:

```sql
INSERT INTO destination_table
  SELECT ... FROM source_table
    WHERE join_condition
    ON CONFLICT DO NOTHING;
```

If the two clauses `WHEN MATCHED` and `WHEN NOT MATCHED` are present, the
translation becomes:

```sql
INSERT INTO destination_table
  SELECT ... FROM source_table
    WHERE join_condition
    ON CONFLICT DO 
      UPDATE update_clause
```

Sometimes, on Oracle, a `DELETE WHERE ...` is present after the `UPDATE` of the
`WHEN MATCHED` clause. In this case, we can add a simple query `DELETE` before
or after the `INSERT`. We can also set this `DELETE` to inside the `INSERT`, as
a common table expression (CTE).

References:

* [ON CONFLICT Clause](https://www.postgresql.org/docs/current/sql-insert.html#SQL-ON-CONFLICT)


## Hints management

Oracle's optimizer accepts _hints_, which make it possible for the DBA to force
the optimizer into using a plan it considers isn't the best. Those hints are
expressed as commends and will be ignored by PostgreSQL, which has no hints.

Nevertheless, a query making use of a hint should have its execution plan
analyzed carefully, to be sure that it will work with PostgreSQL.

The plan will be checked with `EXPLAIN ANALYZE`, which displays both the
estimates of the optimizer and what really occurred during execution. One should
look for a large discrepancy between estimated and real selectivity, for each
node. This will indicate what the optimizer doesn't understand in the query.
Often, this is only the consequence of too imprecise statistics. This can be
corrected in several way.

First, it's possible to make statistics collection more thorough, by raising the
amount of sampled data. This is controlled by the `default_statistics_target`
parameter. This can be done globally or per column of each table. A higher value
will make statistics collection consume more resources, and will make query
planning longer, as more statistics will have to be analyzed to make a decision.
The default value of 100 is good for most tables and data sets, so one usually
only change statistics on the very few columns requiring it. This is done with
`ALTER TABLE … ALTER COLUMN … SET STATISTICS …`. It's also possible to
artificially force the number of distinct values statistic on a column with
`ALTER TABLE … SET COLUMN … SET n_distinct = …`, as this statistic is quite
difficult to get right for the statistics collector.

Sometimes, a query rewrite is the solution: queries can be written in ways that
prevent the optimizer from doing good estimations. This is as true for
PostgreSQL as it is for Oracle.