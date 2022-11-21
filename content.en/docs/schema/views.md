---
weight: 3
bookFlatSection: false
title: "Views migration"
---

## Views migration

Simple views are ported with no difficulty to PostgreSQL. 

A view is updatable as long as it only references only one table (or another 
updatable view) and doesn't contain more complex operators, group by, join types, 
etc. PostgreSQL even supports views defined with `CHECK OPTION` attribute to
prevent updated or inserted rows to being invisible with view's conditions.

References:

* [CREATE VIEW](https://www.postgresql.org/docs/current/sql-createview.html)

### XML views

Some views return their result as XML. Porting these is quite quick. Only the 
`XMLELEMENT` function is incompatible between Oracle and PostgreSQL, so it's 
necessary to add the `name` directive, which becomes the name of the XML element 
for PostgreSQL. 

Thus, this view definition, in Oracle:

```sql
CREATE VIEW warehouse_view OF XMLTYPE
 XMLSCHEMA "http://www.oracle.com/xwarehouses.xsd" 
    ELEMENT "Warehouse"
    WITH OBJECT ID 
    (extract(OBJECT_VALUE, '/Warehouse/Area/text()').getnumberval())
 AS SELECT XMLELEMENT("Warehouse",
            XMLFOREST(WarehouseID AS "Building",
                      area AS "Area",
                      docks AS "Docks",
                      docktype AS "DockType",
                      wateraccess AS "WaterAccess",
                      railaccess AS "RailAccess",
                      parking AS "Parking",
                      VClearance AS "VClearance"))
  FROM warehouse_table;
```

will be converted to this in PostgreSQL:

```sql
CREATE VIEW warehouse_view
SELECT XMLELEMENT(name "Warehouse",
            XMLFOREST(WarehouseID AS "Building",
                      area AS "Area",
                      docks AS "Docks",
                      docktype AS "DockType",
                      wateraccess AS "WaterAccess",
                      railaccess AS "RailAccess",
                      parking AS "Parking",
                      VClearance AS "VClearance"))
  FROM warehouse_table;
```

References:

* [CREATE VIEW](http://www.postgresql.org/docs/current/static/sql-createview.html)

### Materialized views

Materialized views exist in PostgreSQL, and behave in a similar way than Oracle.
However, they are not automatically refreshed, and queries aren't rewritten to 
use them transparently, though.

The materialized view needs to be refreshed on demand by calling 
`REFRESH MATERIALZED VIEW`, that perform a complete rebuild of the view.

Thus, a Oracle materialized view, defined as below:

```sql
CREATE MATERIALIZED VIEW emp_aggr_mv
  BUILD DEFERRED REFRESH FORCE
  ON DEMAND
AS
  SELECT deptno, SUM(sal) AS sal_by_dept
    FROM emp
   GROUP BY deptno;
```

Will be ported in PostgreSQL with:

```sql
CREATE MATERIALIZED VIEW emp_aggr_mv
AS
SELECT deptno, SUM(sal) AS sal_by_dept
  FROM emp
 GROUP BY deptno
WITH NO DATA;
```

Materialized view logs and other `ON COMMIT` behaviors have not been implemented
yet in PostgreSQL. Emulating incremental refreshment could be done by declaring
triggers or using external projects like `pg_ivm`.

References:

* [CREATE MATERIALIZED VIEW](https://www.postgresql.org/docs/current/sql-creatematerializedview.html)
* [pg_ivm](https://github.com/sraoss/pg_ivm)