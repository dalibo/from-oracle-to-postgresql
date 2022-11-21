---
weight: 2
bookFlatSection: false
title: "Joins"
---

## Joins

Oracle supports the ISO-standard way of writing joins only since the 9i version.
Previously, joins were written as stipulated by the first SQL standard, with a
proprietary notation for outer joins. PostgreSQL doesn't support this
proprietary notation, but has full support for the standard compliant syntaxes
(old and new).

### Simple join

The following query can be kept as is: 

```sql
SELECT *
  FROM t1, t2
 WHERE t1.col1 = t2.col1
```

However, this syntax doesn't allow for outer joins. It is therefore recommended 
to use only the new notation, which is anyway much easier to read when inner 
and outer joins are combined in a complex query: 

```SQL
SELECT *
  FROM t1
  JOIN t2 ON (t1.col1 = t2.col1)
```

### Left and right outer joins

Oracle uses the `(+)` notation to describe the side of the join where NULLs are 
allowed. For a left join, the `(+)` annotation would be set on the right part of 
the join (and conversely for a right join). This writing isn't supported by
PostgreSQL. These must be rewritten with the standard compliant syntax:
`LEFT OUTER JOIN` or `LEFT JOIN` for a left join, and `RIGHT OUTER JOIN` or
`RIGHT JOIN` for a right join. 

The following query, written the Oracle proprietary way with a left join:

```sql
SELECT *
  FROM t1, t2
 WHERE t1.col1 = t2.col3 (+);
```

has to be rewritten this way: 

```sql
SELECT *
  FROM t1
  LEFT JOIN t2 ON (t1.col1 = t2.col3);
```

Similarly, the following query contains a right join:

```sql
SELECT *
  FROM t1, t2
 WHERE t1.col1 (+) = t2.col3;
```

ant must be rewritten as: 

```sql
SELECT *
  FROM t1
  RIGHT JOIN t2 ON (t1.col1 = t2.col3);
```

### Full outer join

Before Oracle 9i, a `FULL OUTER` JOIN had to be written with a `UNION` between a
left and a right join. Here is an example of such a full outer join: 

```sql
SELECT *
  FROM t1, t2
 WHERE t1.col1 = t2.col3 (+)
UNION ALL
SELECT *
  FROM t1, t2
 WHERE t1.col1 (+) = t2.col3
   AND t1.col IS NULL
```

This query must be rewritten, and will be much simpler: 

```sql
SELECT *
  FROM t1
  FULL OUTER JOIN t2 ON (t1.col1 = t2.col3);
```

### Mixed join syntaxes

While porting from Oracle to PostgreSQL, it could be tempting to only rewrite 
outer joins, and keep other joins as is, to limit the amount of work. 

This should be avoided: on complex queries, accessing a large number of tables, 
the optimizer may have trouble calculating a good execution plan: the content 
of the `WHERE` clause may not be converted as joins. 

This is controlled by the `from_collapse_limit` setting. It is the maximum depth 
where the optimizer will try to re-order joins. The default is 8, which is 
sufficient, most of the time. A higher value may have a huge impact on planning 
time of the queries. 

Here is a very simplified example, where `from_collapse_limit` will be set to 2,
so that the problem appears on a simple query: 

```sql
CREATE TABLE t1(a INT, b INT);
CREATE TABLE t2(b INT, c INT);
CREATE TABLE t3(c INT, d INT);
CREATE TABLE t4(d INT, e INT);
INSERT INTO t1 SELECT generate_series(1,1000000), generate_series(1,1000000);
INSERT INTO t2 SELECT generate_series(1,1000000), generate_series(1,1000000);
INSERT INTO t3 SELECT generate_series(1,1000000), generate_series(1,1000000);
INSERT INTO t4 SELECT generate_series(1,1000000), generate_series(1,1000000);
ALTER TABLE t4 add PRIMARY KEY (a);
ALTER TABLE t1 add PRIMARY KEY (a);
ALTER TABLE t2 add PRIMARY KEY (b);
ALTER TABLE t3 add PRIMARY KEY (c);
ALTER TABLE t4 add PRIMARY KEY (d);

-- Statistics are now up-to-date
analyze; 

-- 4 tables are involved, we are constraining the optimizer's possibilities
set from_collapse_limit TO 2; 
```

```sql
-- Modern join
EXPLAIN ANALYZE 
SELECT * 
  FROM t1 
  JOIN t2 USING (b) 
  JOIN t3 USING (c) 
  LEFT JOIN t4 USING (d) 
 WHERE t1.a between 1 AND 100;
--                          QUERY PLAN                                      
-- -------------------------------------------------------------------------
--  Nested Loop Left Join
--  (cost=1.70..1271.91 rows=101 width=20)
--  (actual time=0.113..4.607 rows=100 loops=1)
--    ->  Nested Loop
--        (cost=1.27..1064.28 rows=101 width=16)
--        (actual time=0.097..3.129 rows=100 loops=1)
--          ->  Nested Loop
--              (cost=0.85..856.40 rows=101 width=12)
--              (actual time=0.081..1.669 rows=100 loops=1)
--                ->  Index Scan using t1_pkey on t1
--                    (cost=0.42..10.45 rows=101 width=8)
--                    (actual time=0.057..0.163 rows=100 loops=1)
--                      Index Cond: ((a >= 1) AND (a <= 100))
--                ->  Index Scan using t2_pkey on t2
--                    (cost=0.42..8.37 rows=1 width=8)
--                    (actual time=0.011..0.012 rows=1 loops=100)
--                      Index Cond: (b = t1.b)
--          ->  Index Scan using t3_pkey on t3
--              (cost=0.42..2.05 rows=1 width=8)
--              (actual time=0.011..0.012 rows=1 loops=100)
--                Index Cond: (c = t2.c)
--    ->  Index Scan using t4_pkey on t4
--        (cost=0.42..2.05 rows=1 width=8)
--        (actual time=0.011..0.012 rows=1 loops=100)
--          Index Cond: (t3.d = d)
--  Total runtime: 4.815 ms
```

```sql
-- Mix of modern and old style joins
EXPLAIN ANALYZE 
SELECT * 
  FROM t1,t2,t3 
  LEFT JOIN t4 USING (d) 
 WHERE t1.b=t2.b AND t2.c=t3.c 
   AND t1.a BETWEEN 1 AND 100;
--                          QUERY PLAN                                      
-- -------------------------------------------------------------------------
--  Hash Join
--  (cost=31689.66..79086.67 rows=101 width=28)
--  (actual time=711.708..2369.201 rows=100 loops=1)
--    Hash Cond: (t3.c = t2.c)
--    ->  Hash Left Join
--        (cost=30832.00..74478.00 rows=1000000 width=12)
--        (actual time=711.170..2217.581 rows=1000000 loops=1)
--          Hash Cond: (t3.d = t4.d)
--          ->  Seq Scan on t3
--              (cost=0.00..14425.00 rows=1000000 width=8)
--              (actual time=0.007..266.867 rows=1000000 loops=1)
--          ->  Hash
--              (cost=14425.00..14425.00 rows=1000000 width=8)
--              (actual time=710.802..710.802 rows=1000000 loops=1)
--                Buckets: 131072  Batches: 2  Memory Usage: 19548kB
--                ->  Seq Scan on t4
--                    (cost=0.00..14425.00 rows=1000000 width=8)
--                    (actual time=0.010..297.606 rows=1000000 loops=1)
--    ->  Hash
--        (cost=856.40..856.40 rows=101 width=16)
--        (actual time=0.511..0.511 rows=100 loops=1)
--          Buckets: 1024  Batches: 1  Memory Usage: 5kB
--          ->  Nested Loop
--              (cost=0.85..856.40 rows=101 width=16)
--              (actual time=0.025..0.459 rows=100 loops=1)
--                ->  Index Scan using t1_pkey on t1
--                    (cost=0.42..10.45 rows=101 width=8)
--                    (actual time=0.017..0.046 rows=100 loops=1)
--                      Index Cond: ((a >= 1) AND (a <= 100))
--                ->  Index Scan using t2_pkey on t2
--                    (cost=0.42..8.37 rows=1 width=8)
--                    (actual time=0.003..0.003 rows=1 loops=100)
--                      Index Cond: (b = t1.b)
--  Total runtime: 2370.090 ms
```

With a default `from_collapse_limit` of 8, the problem doesn't occur on this query. 
It is much easier, though, to systematically correct all queries than count the 
involved tables in all queries to determine if the work should be done or not. 
The problem will also be much harder to diagnose on a complex query, involving 
subqueries, for example. 

### Cartesian product

A cartesian product can be written this way in both Oracle and PostgreSQL: 

```sql
SELECT *
  FROM t1, t2;
```

However, the standard-compliant syntax is much more explicit, showing the 
developer really intends on doing a cross join:

```sql
SELECT *
  FROM t1
  CROSS JOIN t2;
```

References:

* [Table Expressions](http://www.postgresql.org/docs/current/static/queries-table-expressions.html)