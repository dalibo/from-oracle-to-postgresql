---
weight: 1
bookFlatSection: false
title: "Specificities on Data types"
previouspage: "/sql"
nextpage: "joins"
---

## Specificities on Data types

### Varchar handling

For Oracle, an empty string is also a `NULL` string. It is both. PostgreSQL makes 
the difference: either the string is unknown (`IS NULL`), either it is empty. 

So some queries handling strings will behave differently between Oracle and 
PostgreSQL. Most often, they are the consequence of a comparison between a column 
and an empty string, and the concatenation with a `NULL` string. 

#### Comparaison

In Oracle, if the column `col` is of `VARCHAR2` type, both following queries will 
return the same result:

```sql
SELECT * FROM TABLE WHERE col = '';
SELECT * FROM TABLE WHERE col IS NULL;
```

PostgreSQL won't return the same result. Ora2Pg converts `VARCHAR2` null columns
to varchar null columns, so the first query won't return anything. 

It is therefore compulsory to rewrite this query using `IS NULL` or `IS NOT NULL`. 
PostgreSQL will be able to use an index for this search. 

If the comparison is rewritten with a `COALESCE` function, and keeping the 
comparison to the empty string, it won't use an index. 

```sql
EXPLAIN SELECT * FROM emp2 WHERE ename IS NULL;
--                                QUERY PLAN                                
-- -------------------------------------------------------------------------
--  Index Scan using emp2_ename on emp2  (cost=0.00..12.87 rows=1 width=20)
--    Index Cond: (ename IS NULL)

EXPLAIN SELECT * FROM emp2 WHERE COALESCE(ename) = '';
--                          QUERY PLAN                          
-- -------------------------------------------------------------
--  Seq Scan on emp2  (cost=0.00..79144.81 rows=20972 width=20)
--    Filter: ((COALESCE(ename))::text = `::text)
```

#### Concatenation

Because of Oracle's `VARCHAR2` specificity, concatenation with a NULL value in a 
string won't be a problem. It will be in PostgreSQL though. The SQL standard 
defines that for most functions, a NULL parameter will produce a NULL result 
(they are called `STRICT` functions in PostgreSQL). In the present example, a 
NULL value in a concatenation operation will be propagated to the result, which 
will be NULL too, and not the expected string.

`COALESCE` should be added in such portions of code:

```sql
SELECT 'employee name: ' || COALESCE(ename, `) FROM emp;
```

### Date manipulation

#### Ouptut format

In Oracle, the `NLS_DATE_FORMAT` determines the output format of `TO_CHAR()`
and `TO_DATE()` functions.

PostgreSQL, by default, uses the ISO-8601 format for date outputs: 
`YYYY-MM-DD HH24:MI:SS.mmmmmm+TZ`. It cat be modified with the `DateStyle` session
parameter (by default `ISO, DMY`), but won't match Oracle's anyway. 

#### Operation on dates

Oracle permits adding or substracting an integer to or from a date. For example: 

```sql
SELECT SYSDATE + 1 FROM DUAL;
```

will return tomorrow's date. For PostgreSQL's date datatype, the behaviour is 
the same, but it won't be for a timestamp. 

To get the same result with a timestamp (the equivalent type to Oracle's date 
type) with PostgreSQL, one should use an interval:

```sql
SELECT now() + INTERVAL '1 DAY';
```

Similarly, substracting two timestamps from one another returns an integer
corresponding to the number of days between these two days, when it returns an 
interval with the exact difference with PostgreSQL. 

#### SYSDATE

Please note that the DATE datatype, in Oracle, is a `TIMESTAMP WITHOUT TIME ZONE`
in PostgreSQL. 

```sql
SELECT to_char(sysdate,'DD/MM/YYYY HH24:MI:SS') FROM dual;

-- TO_CHAR(SYSDATE,'DD
-- -------------------
-- 11/03/2011 14:58:22
```

One should use `localtimestamp` (and not `current_timestamp` which returns a 
`TIMESTAMP WITH TIME ZONE`):

```sql
SELECT localtimestamp;
--          timestamp          
-- ----------------------------
--  2011-03-11 16:59:29.889823
```

Thus, any variable declared as DATE in Oracle's PL/SQL must be declared as 
timestamp in PostgreSQL's PL/pgsql.