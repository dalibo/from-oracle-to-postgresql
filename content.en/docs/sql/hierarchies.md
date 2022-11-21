---
weight: 4
bookFlatSection: false
title: "Hierarchical querying"
---

## Hierarchical querying

Oracle provides a `CONNECT BY` function to explore a hierarchical tree. This
proprietary functionality has advanced features such as loop detection and
provides pseudo-columns such as depth and path.

Since version 14, many advanced features are available with PostgreSQL. For
earlier versions, significant work must be performed to port queries using
theses clauses.

### CONNECT BY

Here is a SQL query exploring the `emp` table's hierarchy. The `mgr` column in
this table points to the manager of the employee. If it is NULL, then this
employee is on top of the hierarchy (`START WITH mgr IS NULL`). The link between
the employee and its manager is built with the `CONNECT BY PRIOR empno = mgr`
clause, indicating that the `mgr` is the `empno` identifier of the precedent
level of hierarchy. 

```sql
SELECT empno, ename, job, mgr
  FROM emp
  START WITH mgr IS NULL
  CONNECT BY PRIOR empno = mgr
  ```

This is converted with the help of a recursive query (`WITH RECURSIVE`).
Recursion is initiated in a first query retrieving the lines matching the `START
WITH` condition of previous query: `mgr is NULL`. Recursion continues then with
the following query doing a join between the `emp` table and the virtual
`emp_hierarchy` table defined by the `WITH RECURSIVE` clause. The join condition
matches the previous `CONNECT BY`. Here, `emp_hierarchy` has been aliased to
`prior`, to better illustrate the conversion. 

Here is a possible rewriting then:

```sql
WITH RECURSIVE emp_hierarchy (empno, ename, job, mgr) AS (
  SELECT empno, ename, job, mgr
    FROM emp
    WHERE mgr IS NULL
    UNION ALL
  SELECT emp.empno, emp.ename, emp.job, emp.mgr
    FROM emp
    JOIN emp_hierarchy prior ON (emp.mgr = prior.empno)
)
SELECT * FROM emp_hierarchy;
```

One should pay attention to the returned order of rows with the `WITH RECURSIVE`
query. Oracle uses by default a _depth-first_ algorithm to implement `CONNECT
BY`. It will explore each branch before going to the next. `WITH RECURSIVE` is
_breadth-first_, exploring each level before going onto the next. 

It's possible to get the `CONNECT BY` order though, by sorting on a path column,
such as the one one would build to emulate `SYS_CONNECT_BY_PATH`: 

```sql
WITH RECURSIVE emp_hierarchy (empno, ename, job, mgr, path) AS (
  SELECT empno, ename, job, mgr, ARRAY[ename::TEXT] AS path
    FROM emp
   WHERE mgr IS NULL
   UNION ALL
  SELECT emp.empno, emp.ename, emp.job, emp.mgr, 
         prior.path || emp.ename::TEXT AS path
    FROM emp
    JOIN emp_hierarchy prior ON (emp.mgr = prior.empno)
)
SELECT empno, ename, job FROM emp_hierarchy AS emp
ORDER BY path;
```

Oracle 11g has changed the default return order of `CONNECT BY` queries. 

### LEVEL pseudo-column

One can get the hierarchy depth of an element using the `LEVEL` pseudo-column. 

```sql
SELECT empno, ename, job, mgr, level
  FROM emp
  START WITH mgr IS NULL
  CONNECT BY PRIOR empno = mgr
```

Porting `LEVEL` is easy. Initialize a column called `level` to 1, then increment
it in each new recursive join:

```sql
WITH RECURSIVE emp_hierarchy (empno, ename, job, mgr, level) AS (
    SELECT empno, ename, job, mgr, 1 AS level
      FROM emp
     WHERE mgr IS NULL
  UNION ALL
    SELECT emp.empno, emp.ename, emp.job, emp.mgr, prior.level + 1
      FROM emp
      JOIN emp_hierarchy prior ON (emp.mgr = prior.empno)
)
SELECT * FROM emp_hierarchy;
```

### SYS_CONNECT_BY_PATH function

`SYS_CONNECT_BY_PATH` is used to display the path taken to reach a record, each
element separated by a given character. For instance, the following query
returns all managers of an employee:

```sql
SELECT empno, ename, job, mgr, SYS_CONNECT_BY_PATH(ename, '/') AS path
  FROM emp
  START WITH mgr IS NULL
  CONNECT BY PRIOR empno = mgr;
```

This is also quite easy to convert. One can add a root path '/' in the first
query of the recursion, then keep concatenating current path to previous path. 

```sql
WITH RECURSIVE emp_hierarchy (empno, ename, job, mgr, path) AS (
    SELECT empno, ename, job, mgr, '/' || ename AS path
      FROM emp
     WHERE mgr IS NULL
  UNION ALL
    SELECT emp.empno, emp.ename, emp.job, emp.mgr,
           prior.path || '/' || emp.ename
      FROM emp
      JOIN emp_hierarchy prior ON (emp.mgr = prior.empno)
)
SELECT * FROM emp_hierarchy;
```

Another way is to use an array to store the path, then convert it to any other
representation (here the same as `SYS_CONNECT_BY_PATH(ename, '/')`). This can be
used to detect loops all at once: 

```sql
WITH RECURSIVE emp_hierarchy (empno, ename, job, mgr, path) AS (
    SELECT empno, ename, job, mgr, ARRAY[ename::TEXT] AS path
      FROM emp
     WHERE mgr IS NULL
  UNION ALL
    SELECT emp.empno, emp.ename, emp.job, emp.mgr, 
           prior.path || emp.ename::TEXT AS path
      FROM emp
      JOIN emp_hierarchy prior ON (emp.mgr = prior.empno)
)
SELECT empno, ename, job, array_to_string(path, '/') AS path 
  FROM emp_hierarchy AS emp;
```

### NOCYCLE clause

The following Oracle query: 

```sql
SELECT empno, ename, job, mgr
  FROM emp
  START WITH mgr IS NULL
  CONNECT BY NOCYCLE PRIOR empno = mgr;
```

will be converted to PostgreSQL this way:

```sql
WITH RECURSIVE emp_hierarchy (empno, ename, job, mgr, path, is_cycle) AS (
    SELECT empno, ename, job, mgr, ARRAY[ename::TEXT] AS path, false AS is_cycle
      FROM emp
     WHERE mgr IS NULL
  UNION ALL
    SELECT emp.empno, emp.ename, emp.job, emp.mgr, 
           prior.path || emp.ename::TEXT AS path, 
           emp.ename = ANY(prior.path) AS is_cycle
      FROM emp
      JOIN emp_hierarchy prior ON (emp.mgr = prior.empno)
     WHERE is_cycle = false
)
SELECT empno, ename, job, mgr
  FROM emp_hierarchy AS emp
 WHERE is_cycle = false;
```

### CONNECT_BY_IS_CYCLE clause

The `CONNECT_BY_IS_CYCLE` clause returns 1 if the current record has a child that
is also its ancestor. Else it will return 0. This can be emulated from the
previous query, with a conditional expression, and by removing the last filter:

```sql
WITH RECURSIVE emp_hierarchy (empno, ename, job, mgr, level, path, is_cycle) AS (
  SELECT empno, ename, job, mgr, 1 AS level,
         ARRAY[ename::TEXT] AS path, false AS is_cycle
    FROM emp
    WHERE mgr = 10000
    UNION ALL
  SELECT emp.empno, emp.ename, emp.job, emp.mgr, 
          prior.level + 1 AS level, 
          prior.path || emp.ename::TEXT AS path,
          emp.ename = ANY(prior.path) AS is_cycle 
    FROM emp
    JOIN emp_hierarchy prior ON (emp.mgr = prior.empno)
    WHERE is_cycle = false
)
SELECT *, is_cycle::int connect_by_is_cycle
  FROM emp_hierarchy AS emp;
```

### ORDER SIBLINGS BY clause

The `ORDER SIBLINGS BY` clause can be emulated by starting from the recursive
query used for `SYS_CONNECT_BY_PATH`, and sort on the `path` array column, as
arrays are sorted by comparing elements from first to last:

```sql
WITH RECURSIVE emp_hierarchy (empno, ename, job, mgr, path) AS (
  SELECT empno, ename, job, mgr, '/' || ename AS path
    FROM emp
   WHERE mgr IS NULL
   UNION ALL
  SELECT emp.empno, emp.ename, emp.job, emp.mgr, prior.path || '/' || emp.ename
    FROM emp
    JOIN emp_hierarchy prior ON (emp.mgr = prior.empno)
)
SELECT * FROM emp_hierarchy
 ORDER BY path;
```

However, this emulation only works with an ascending sort. Applying a descending
sort does not return the expected result.

### CONNECT_BY_ROOT clause

The `CONNECT_BY_ROOT` clause returns the root of the hierarchy for each element.
In the following example, the last column will return the topmost person in the
hierarchy of the current employee: 

```sql
SELECT empno, ename, job, mgr AS direct_mgr, 
       CONNECT_BY_ROOT ename AS mgr
  FROM emp
  START WITH mgr IS NULL
  CONNECT BY mgr = PRIOR empno
  ORDER SIBLINGS BY ename DESC;
```

This is converted the same way as the `SYS_CONNECT_BY_PATH` query. The path
array can simply be used to get the topmost element.

```sql
WITH RECURSIVE emp_hierarchy (empno, ename, job, mgr, path) AS (
    SELECT empno, ename, job, mgr, ARRAY[ename::TEXT] AS path
      FROM emp
     WHERE mgr IS NULL
  UNION ALL
    SELECT emp.empno, emp.ename, emp.job, emp.mgr, 
           prior.path || emp.ename::TEXT AS path
      FROM emp
      JOIN emp_hierarchy prior ON (emp.mgr = prior.empno)
)
SELECT empno, ename, job, path[1] AS connect_by_root 
  FROM emp_hierarchy AS emp;
```

### CONNECT_BY_ISLEAF clause

`CONNECT_BY_ISLEAF` takes no arguments. It specifies whether current row (called
_leaf_) is no longer connected to a deeper row through the hierarchy tree. The
value `0` is returned if deeper rows exist and `1` if current row is the last in
the hierarchy.

```sql
SELECT empno, ename, job, mgr,
       CONNECT_BY_ROOT ename AS mgr,
       CONNECT_BY_ISLEAF ASisleaf, 
       SYS_CONNECT_BY_PATH(ename, '/') AS path
  FROM emp
  START WITH mgr IS NULL
  CONNECT BY mgr =  PRIOR empno
```

`CONNECT_BY_ISLEAF` is a bit harder to emulate, as we are using a
_breadth-first_ algorithm: we can't know if a record is a leaf before having
done the next iteration of the recursive join. We'll have to identify leaf
records with a second pass. 

For instance, we can use a window function to compare each found path to the
following (sorted by `path`): if the next path is longer than the current, we
are not a leaf. Else, we are a leaf. Here is a rewrite doing this: 

```sql
WITH RECURSIVE emp_hierarchy (empno, ename, job, mgr, path) AS (
  SELECT empno, ename, job, mgr, ARRAY[ename::TEXT] AS path
    FROM emp
    WHERE mgr IS NULL
    UNION ALL
  SELECT emp.empno, emp.ename, emp.job, emp.mgr, 
         prior.path || emp.ename::TEXT AS path
    FROM emp
    JOIN emp_hierarchy prior ON (emp.mgr = prior.empno)
)
SELECT emp.empno, emp.ename, emp.job, 
       CASE WHEN leaf.empno IS NULL THEN 1 ELSE 0 END AS isleaf
  FROM emp_hierarchy AS emp
  LEFT JOIN emp_hierarchy AS leaf ON (emp.empno = leaf.mgr)
 ORDER BY emp.path;
```