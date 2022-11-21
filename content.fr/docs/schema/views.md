---
weight: 3
bookFlatSection: false
title: "Reprise des vues"
---

## Reprise des vues

Les vues simples sont portées sans difficultés sur PostgreSQL.

PostgreSQL supporte également les vues que l'on peut mettre à jour 
(_updatable views_), y compris avec la clause `WITH CHECK OPTION` qui permet 
de s'assurer que des données insérées ou mises à jour dans la vue satisfont 
toujours les éventuelles conditions de sélection de cette vue.

Références :

* [CREATE VIEW](https://docs.postgresql.fr/current/sql-createview.html)

### Vues XML

Certaines vues retournent le résultat au format XML. Le portage est assez rapide. 
Seule la fonction `XMLELEMENT` n'est pas compatible entre Oracle et PostgreSQL, 
il est nécessaire d'ajouter la directive `name`, qui devient le nom de l'élément 
XML pour PostgreSQL.

Ainsi, la définition de la vue suivante pour Oracle :

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

sera transposée de cette façon pour PostgreSQL :

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

Références :

* [CREATE VIEW](https://docs.postgresql.fr/current/sql-createview.html)

### Vues matérialisées

Les vues matérialisées existent dans PostgreSQL et se comportent globalement 
de la même manière qu'avec Oracle. Néanmoins le rafraîchissement des vues 
n'est possible qu'à la demande (sur commande `REFRESH MATERIALZED VIEW`) et 
reconstruit totalement la vue.

Ainsi, une vue créée vide par exemple avec :

```sql
CREATE MATERIALIZED VIEW emp_aggr_mv
  BUILD DEFERRED REFRESH FORCE
  ON DEMAND
AS
  SELECT deptno, SUM(sal) AS sal_by_dept
    FROM emp
   GROUP BY deptno;
```

sera traduite en :

```sql
CREATE MATERIALIZED VIEW emp_aggr_mv
AS
SELECT deptno, SUM(sal) AS sal_by_dept
  FROM emp
 GROUP BY deptno
WITH NO DATA;
```

Les journaux de vues matérialisées et autres comportements avancées comme le
rafraîchissement `ON COMMIT` ne sont pas encore implémentés avec PostgreSQL.
L'émulation du rafraîchissement nécessite d'opter pour l'usage de triggers ou
d'outils externes comme l'extension `pg_ivm`.

Références :

* [vues matérialisées](https://docs.postgresql.fr/current/sql-creatematerializedview.html)
* [pg_ivm](https://github.com/sraoss/pg_ivm)