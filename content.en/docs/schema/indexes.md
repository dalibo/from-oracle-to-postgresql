---
weight: 5
bookFlatSection: false
title: "Index migration"
---

## Index migration

Only the BTree indexes match between both databases. The other index types from
Oracle don't exist in PostgreSQL, but postgreSQL also has some indexes types
of its own. Anyway, most indexes are BTree, as its the default type in both
databases.

### Index on character string

When an index is built to improve searches on the `LIKE` operator on a string type 
column, this index has to be built using the `varchar_pattern_ops` operator class
for a varchar column, `text_pattern_ops` for a text column, or `bpchar_pattern_ops` 
for a char column. These operator classes are used when the database's collation 
isn't `C`. 

The operator class has to be added after the target column name in the index 
creation statement:

```sql
CREATE INDEX emp2_ename ON emp2 (ename varchar_pattern_ops);
```

References:

* [Operator Classes and Operator Families](http://www.postgresql.org/docs/current/static/indexes-opclass.html)

### Bitmap Index

A bitmap index proposed by Oracle is required when a column stored a few distinct
values. Bitmap index use a internal array of bits as a physical representation of
a value in the table. A simple case is biological gender, encoded in an array of
two bits: one for male, another for female. For each row, a bit is adressed in the
bitmap index structure. This one provides extrem compactness only if column has
a few distinct values.

On-disk bitmap indexes dont exist in PostgreSQL. That can be created in-memory 
from a BTree index if required. BTree indexes are much larger than On-disk bitmap 
indexes, but have a much better concurrency. Another technic involves GIN indexes,
used by composite data like arrays or hstore columns.

GIN stands for Generalized Inverted Index. Each possible values are referred as
keys and rows are stored in their associated key (example `gender=F`). Each key 
value is stored only once, so a GIN index is very compact for cases where the
same key appears many times.

For a gender column, GIN is constructed over two _posting lists_ (female and
male). Following example shows differences between Btree and GIN indexes in their
storage:

```sql
-- Use btree_gin extension to manipulate scalar columns
CREATE extension btree_gin;
CREATE TABLE t1 (name VARCHAR, gender CHAR);

-- Gender is equally represented
INSERT INTO t1 
  SELECT i, CASE WHEN i%2 = 0 THEN 'F' ELSE 'M' END
  FROM generate_series(1,100000000) g(i); 

CREATE INDEX idx_gender_gin ON t1 USING gin (sexe);

SELECT pg_size_pretty(pg_table_size('t1'));
--  pg_size_pretty
-- ----------------
--  4223 MB

SELECT pg_size_pretty(pg_table_size('idx_gender_gin'));
--  pg_size_pretty 
-- ----------------
--  102 MB

SELECT pg_size_pretty(pg_table_size('idx_gender_btree'));
--  pg_size_pretty 
-- ----------------
--  2142 MB
```

The large work memory is set to store the whole bitmap in memory. If it had been 
smaller, the bitmap would have become “lossy”, meaning that it would only hold
block numbers, and not records themselves. The sieving through records would have 
required to visit many more blocks.

GIN provides good compactness and concurrency access and could be used to mimic
Bitmap Indexes when column's values are few as possible.

References:

* Article by Hans-Juergen Schoenig: [GIN – Just A Kind Of Index](https://www.cybertec-postgresql.com/en/gin-just-an-index-type/)

### Reverse index

Reverse indexes make it possible to optimize searchs such as `LIKE '%string'`, 
which usually don't benefit from an index. There is no reverse index in PostgreSQL,
but a trigram index with `pg_trgm` extension could be used in place.

The trigram indexes can either use GiST or GIN indexing. GiST is faster for 
Nearest-Neighbor search (called Knn-search in the literature), GIN much faster 
for strict matching, but requires 3 consecutive characters in the pattern (which 
is usually a good idea for fast searchs).

They can be used for LIKE `'%string'`, but also for LIKE `'%string%'`, or 
`LIKE '%str%ing%'`. GIN indexes are usually a bit bigger and cost more to update
than a BTree index.

The `pg_trgm` extension, though not included in PostgreSQL directly, is distributed
along with PostgreSQL (postgresql-contrib package) and maintained by PostgreSQL's 
developers themselves.

This index type is not directly created by Ora2Pg, it has to be performed 
manually. 

```sql
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_emp_ename_trgm ON emp USING gist (ename gist_trgm_ops);
--or
CREATE INDEX idx_emp_ename_trgm ON emp USING gin (ename gin_trgm_ops);
```

The execution plan of a SELET query demonstrates the use of the 
`idx_emp_ename_trgm` index: 

```sql
EXPLAIN SELECT * FROM emp WHERE ename LIKE '%IN%';
--                                        QUERY PLAN                                       
-- ----------------------------------------------------------------------------------------
--  Bitmap Heap Scan on emp  (cost=1442.95..3522.23 rows=32742 width=20)
--    Recheck Cond: ((ename)::text ~~ '%IN%'::text)
--    ->  Bitmap Index Scan on idx_emp_ename_trgm  (cost=0.00..1434.77 rows=32742 width=0)
--          Index Cond: ((ename)::text ~~ '%IN%'::text)
```

These trigram indexes can also be used on case-insensitive matching, using `ILIKE`. 

References:

* [K-Nearest-Neighbor Indexing](https://wiki.postgresql.org/wiki/What%27s_new_in_PostgreSQL_9.1#K-Nearest-Neighbor_Indexing), What's new in PostgreSQL 9.1?
* [pg_trgm](https://www.postgresql.org/docs/current/pgtrgm.html)

### Non blocking index creation

PostgreSQL can create index without blocking concurrent modifications on the 
database, using `CREATE INDEX CONCURRENTLY`. This statement may though 
leave an index to an invalid state if its creation fails. This can happen if 
the index cannot be built, for instance a UNIQUE index that cannot be validated. 

In a same thought, rebuilding an index without blocking is performed with
`REINDEX CONCURRENTLY`. Like creation process, an index can be left in invalid 
state.

References:

* [CREATE INDEX](https://www.postgresql.org/docs/current/sql-createindex.html)
* [REINDEX](https://www.postgresql.org/docs/current/sql-reindex.html)
