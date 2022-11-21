---
weight: 3
bookFlatSection: false
title: "Conditional expressions"
---

## Conditional expressions

Although Oracle as support for the different conditional expressions as 
specified by the SQL standard, far too many SQL queries still use Oracle's 
historical functions. 

### DECODE

Oracle's `DECODE` function is a proprietary equivalent of the standard compliant
`CASE` clause. 

Here is an example of `DECODE` use: 

```sql
SELECT emp_name,
       decode(
          trunc((yrs_of_service + 3) / 4),
          0, 0.04,
          1, 0.04,
          0.06
       ) AS perc_value
  FROM employees;
```

This should be rewritten this way: 

```sql
SELECT emp_name,
       CASE WHEN trunc(yrs_of_service + 3) / 4 = 0 THEN 0.04
            WHEN trunc(yrs_of_service + 3) / 4 = 1 THEN 0.04
            ELSE 0.06
       END
  FROM employees;
```

Here is another example:

```sql
DECODE('user_status','active','username',NULL)
```

This should be rewritten as: 

```sql
CASE WHEN user_status='active' THEN username ELSE NULL END
```

Pay attention to the comments between `WHEN` and `THEN`, that are not supported 
by PostgreSQL. 

References:

* [CASE](https://www.postgresql.org/docs/current/functions-conditional.html#FUNCTIONS-CASE)

### NVL

Oracle's `NVL` function is still often used, although the standard-compliant 
`COALESCE` is also available. These two functions return the first non-NULL 
argument of a list. Of course, PostgreSQL only implements the standard compliant 
function, `COALESCE`. Replacing NVL with `COALESCE` should be sufficient. 

Thus, the following query:

```sql
SELECT NVL(description, short_description, '(none)') FROM articles;
```

will be easily converted to:

```sql
SELECT COALESCE(description, short_description, '(none)') FROM articles;
```

References:

* [COALESCE](https://www.postgresql.org/docs/current/functions-conditional.html#FUNCTIONS-COALESCE-NVL-IFNULL)

### ROWNUM

Oracle presents a `ROWNUM` pseudo-column which can be used to number a query's 
result lines. The `ROWNUM` can be used to number the lines, or to limit the number 
of lines returned by a query. 

#### ROWNUM for numbering

In the first case, numbering the lines of a result set, the query should be 
rewritten to make use of the `row_number()` window function. Though Oracle 
recommends using the standard compliant `row_number()` function, `ROWNUM` is still 
frequently used in a query: 

```sql
SELECT ROWNUM, * FROM employees;
```

This will be rewritten as:


```sql
SELECT row_number() OVER () AS rownum, * FROM employees;
```

One should pay attention to a query using `ORDER BY` and `ROWNUM` to number the 
lines returned by a query. Indeed, the sort is performed after the rownum 
assignment. These queries should be double-checked.

#### ROWNUM and limiting results

To limit the set returned by a query, the ROWNUM predicates should be replaced
by `LIMIT/OFFSET`.

The following query returns the first 10 lines of the `employees` table in
Oracle:

```sql
SELECT *
  FROM employees
 WHERE ROWNUM < 11;
```

It will be rewritten as this in PostgreSQL:

```sql
SELECT *
  FROM employees
 LIMIT 10;
```

If the result need to be sorted in a particular way, Oracle will operate a
sorting node with `ORDER BY` after adding the `ROWNUM` pseudo-column (`COUNT
STOPKEY`), as shown in the execution plan of the previous query:

```
---------------------------------------------------------------------------------
| Id  | Operation            | Name      | Rows | Bytes | Cost (%CPU)| Time     |
---------------------------------------------------------------------------------
|   0 | SELECT STATEMENT     |           |   10 |   690 |     3  (34)| 00:00:01 |
|   1 |  SORT ORDER BY       |           |   10 |   690 |     3  (34)| 00:00:01 |
|*  2 |   COUNT STOPKEY      |           |      |       |            |          |
|   3 |    TABLE ACCESS FULL | EMPLOYEES |   10 |   690 |     2   (0)| 00:00:01 |
---------------------------------------------------------------------------------
```

A query limiting with Oracle is generally written this way (to work around the
fact that `ROWNUM` is performed before the sorting): 

```sql
SELECT ROWNUM, r.*
  FROM (SELECT *
          FROM t1
         ORDER BY col) r
 WHERE ROWNUM BETWEEN 1 AND 10;
```

On the opposite, PostgreSQL will first sort, then limit the result. When
PostgreSQL has both a `LIMIT` and a sort with `ORDER BY`, the sort will be
performed first. This should be written this way in PostgreSQL:

```sql
SELECT *
  FROM employees
 ORDER BY salary DESC 
 LIMIT 10;
```

```
                               QUERY PLAN                                
-------------------------------------------------------------------
 Limit (cost=81.44..81.46 rows=10 width=8)
   -> Sort (cost=81.44..87.09 rows=2260 width=8)
      Sort Key: salary DESC
      -> Seq Scan on employees (cost=0.00..32.60 rows=2260 width=8)
```

References:

* [LIMIT and OFFSET](https://www.postgresql.org/docs/current/queries-limit.html)
* [Window Functions](https://www.postgresql.org/docs/current/functions-window.html)
