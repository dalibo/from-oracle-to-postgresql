---
weight: 3
bookFlatSection: false
title: "Portage des requêtes SQL"
url: "portage-des-requetes-sql"
bookCollapseSection: true
---

## Compatibilité des ordres DML

La plupart des ordres DML sont compatibles entre Oracle et PostgreSQL. On peut 
toutefois noter que l'ordre `MERGE` n'existe pas encore dans PostgreSQL. 
Certaines restrictions s'appliquent aussi pour l'utilisation de certaines 
fonctions de fenêtrage (_window functions_) et des CTEs récursives.

<!--
## Alias de colonnes

Oracle permet l'omission du mot clé `AS` pour définir les alias de colonnes 
dans une requête. PostgreSQL le permet également depuis la version 9.0.
-->

## Alias de sous-requêtes

Oracle ne génère pas de message d'erreur lorsqu'une sous-requête ne possède pas 
d'alias. PostgreSQL retourne une erreur dans ses cas là, ce dernier exige que 
toutes les sous-requêtes disposent d'un alias.

Ainsi la requête Oracle suivante :

```sql
SELECT colonnes...
  FROM (SELECT ...
       )
 WHERE predicats
```

Devra être réécrite en choisissant un alias suffisamment descriptif :

```sql
SELECT colonnes...
  FROM (SELECT ...
       ) sousreq1
 WHERE predicats
```

## Conversions implicites

Les conversions implicites de et vers un champ de type texte ont été supprimées
sous PostgreSQL depuis la version 8.3.

Par exemple, il n'est pas possible de faire ce type de requête :

```sql
CREATE TABLE depts ( numero CHAR(2), nom VARCHAR(25) );

SELECT * FROM depts WHERE numero BETWEEN 0 AND 42;
-- ERROR:  operator does not exist: character >= INTEGER
-- LIGNE 1 : SELECT * FROM depts WHERE numero BETWEEN 0 AND 42;
```

Si l'on veut pouvoir faire faire fonctionner cette requête, il faut préciser
explicitement la conversion à réaliser :

```sql
SELECT * FROM depts WHERE numero::INTEGER BETWEEN 0 AND 42;
-- ou (respect de la norme SQL)
SELECT * FROM depts WHERE CAST(id AS INTEGER) BETWEEN 0 AND 42;
```

Avec Oracle, ce type de conversion est implicite.

## Clauses HAVING et GROUP BY

Bien que la documentation Oracle indique que la clause `GROUP BY` précède la 
clause `HAVING`, la grammaire Oracle autorise l'inverse. Il faut donc corriger 
les requêtes écrites de la façon `HAVING ... GROUP BY`.

Les requêtes de la forme suivante :

```sql
SELECT * FROM test HAVING count(*) > 3 GROUP BY i;
```

seront transposées de la façon suivante pour pouvoir s'exécuter sous PostgreSQL :

```sql
SELECT * FROM test GROUP BY i HAVING count(*) > 3;
```

Références :

* [Clauses GROUP BY et HAVING](https://docs.postgresql.fr/current/queries-table-expressions.html#QUERIES-GROUP)

## Utilisation de ROWID

Dans de très rares cas, des requêtes SQL utilisent la colonne `ROWID` d'Oracle, 
par exemple pour dédoublonner des enregistrements. Le `ROWID` est la localisation 
physique d'une ligne dans une table. L'équivalent dans PostgreSQL est le `ctid`.

Plus précisément, le `ROWID` Oracle représente une adresse logique d'une ligne, 
encodée sous la forme `OOOOOO.FFF.BBBBBB.RRR` où `O` représente le numéro d'objet, 
`F` le fichier, `B` le numéro de bloc et `R` la ligne dans le bloc. Le format est
différent dans le cas d'une table stockée dans un `BIG FILE TABLESPACE`, mais le
principe reste identique.

Quant au `ctid` de PostgreSQL, il ne représente qu'un couple _(numéro du bloc, 
numéro de l'enregistrement)_, aucune autre information de localisation physique 
n'est disponible. Le `ctid` n'est donc unique qu'au sein d'une table. De part 
ce fait, une requête ramenant le `ctid` des lignes d'une table partitionnée peut
présenter des  `ctid` en doublons. On peut dans ce cas utiliser le champ caché 
`tableoid` (l'identifiant unique de la table dans le catalogue) de chaque table 
pour différencier les doublons par partition.

Cette méthode d'accès est donc à proscrire, sauf opération particulière et cadrée.

Références :

* [Colonnes système de PostgreSQL](https://docs.postgresql.fr/current/ddl-system-columns.html)


## Portage de l'opérateur ensembliste MINUS

L'opérateur ensembliste `MINUS` est à transposer en `EXCEPT` pour PostgreSQL. 
Les autres opérateurs ensemblistes `UNION`, `UNION ALL` et `INSERSECT` 
ne nécessitent pas de transposition.

Ainsi, la requête suivante retourne les produits de l'inventaire qui n'ont pas fait
l'objet d'une commande. Elle est exprimée ainsi pour Oracle :

```sql
SELECT product_id FROM inventories
MINUS
SELECT product_id FROM order_items
ORDER BY product_id;
```

La requête sera transposé de la façon suivante pour PostgreSQL :

```sql
SELECT product_id FROM inventories
EXCEPT
SELECT product_id FROM order_items
ORDER BY product_id;
```

Références :

* [Opérateurs UNION, INTERSECT et EXCEPT](https://docs.postgresql.fr/current/queries-union.html)

## Fonctions de fenêtrage

Les clauses permettant d'utiliser les fonctions de fenêtrage ne nécessitent que 
peu d'adaptations.

Oracle propose une clause `ORDER SIBLINGS BY`. Le mot clé `SIBLINGS` n'a pas 
d'équivalent et est de toute façon dédié à la manipulation des hiérarchies, donc 
avec `CONNECT BY`. Une telle requête devra être de toute façon réécrite 
entièrement pour être portée vers PostgreSQL.

La clause `PARTITION BY` ne nécessite aucune adaptation, de même que la clause 
de fenêtrage (`RANGE ...` ou `ROWS ...`).

La plupart des fonctions de fenêtrage généralistes d'Oracle existent pour PostgreSQL. 
Certaines fonctions n'ont malgré tout pas d'équivalent pour l'instant :

* `RATIO_TO_REPORT` nécessite simplement d'utiliser une division sur le résultat
  de la fonction `SUM` ;
* Le portage de la fonction `LISTAGG` nécessite une réécriture avec les
  fonctions `array_agg` et `array_to_string` de PostgreSQL.

À noter toutefois que les capacités d'extensions de PostgreSQL permettent de 
développer des fonctions d'agrégat et des fonctions de fenêtrage, permettant 
ainsi de palier l'absence d'équivalent de certaines fonctions.

Références :

* [Fonctions Window](https://docs.postgresql.fr/current/functions-window.html)
* [Tutoriel fonctions Window](https://docs.postgresql.fr/current/tutorial-window.html)
* [Appels de fonctions Window](https://docs.postgresql.fr/current/sql-expressions.html#syntax-window-functions)
* [Fonctions d'agrégat](https://docs.postgresql.fr/current/functions-aggregate.html)

## CTE récursif

La grammaire Oracle ne distingue pas une CTE simple d'une CTE récursive, au 
contraire de PostgreSQL qui les distingue avec deux syntaxes différentes : 
`WITH` et `WITH RECURSIVE`. Il convient donc de corriger les CTE récursives, 
notamment en repérant l'opérateur `UNION ALL` et, au sein d'une vue, une 
référence de cette même vue.

La requête suivante pour Oracle réalise une récursion très simple :

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
Elle sera réécrite de la façon suivante pour PostgreSQL :

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

La clause `WITH` comprend également des extensions syntaxiques propres à Oracle,
pour décrire la façon de réaliser la récursion (_search_clause_) et la détection
des cycles (_cycle_clause_). Concernant la détection des cycles, un exemple
d'implémentation est donné dans la partie « [Traitement des hiérarchies]({{<
relref "/docs/sql/hierarchies" >}}) ».

Références :

* [SELECT](https://docs.postgresql.fr/current/sql-select.html)
* [Détection de cycles](https://docs.postgresql.fr/current/queries-with.html#QUERIES-WITH-CYCLE)

## MERGE

La requête SQL `MERGE` permet de faire des insertions ou des mises à jour de 
tables en fonction de l'existence préalable ou non des lignes.

La syntaxe typique de l'instruction ressemble à ceci :

```sql
MERGE INTO table_destination
  USING table_source ON (condition)
  WHEN MATCHED THEN update_clause
  WHEN NOT MATCHED THEN insert_clause;
```

Mais les versions antérieures à PostgreSQL 15 ne supportent pas la syntaxe
`MERGE` de la norme SQL. En revanche, on peut émuler cette instruction avec un
`INSERT`. Il faut ici distinguer plusieurs cas, en fonction de la présence ou
non des clauses `WHEN MATCHED` et `WHEN NOT MATCHED`.

S'il n'y a pas de clause `WHEN NOT MATCHED`, il s'agit d'un simple `UPDATE` :

```sql
UPDATE table_destination
  update_clause
  USING table_source
  WHERE conditions_de_jointure;
```

S'il n'y a pas de clause `WHEN MATCHED`, il s'agit d'un `INSERT` sur les lignes 
qui n'existent pas déjà :

```sql
INSERT INTO table_destination
  SELECT ... FROM table_source
    WHERE conditions_de_jointure
    ON CONFLICT DO NOTHING;
```

Si les deux clauses `WHEN MATCHED` et `WHEN NOT MATCHED` sont présentes, la 
traduction devient :

```sql
INSERT INTO table_destination
  SELECT ... FROM table_source
    WHERE conditions_de_jointure
    ON CONFLICT DO
      UPDATE update_clause
```

Quelquefois, sur Oracle, un `DELETE WHERE ...` est présent après l'`UPDATE` de 
la clause `WHEN MATCHED`. Dans ce cas, on peut ajouter une simple requête 
`DELETE` avant ou après l'`INSERT`. On peut aussi mettre ce `DELETE` à 
l'intérieur de l'`INSERT`, sous la forme d'une expression de table commune (CTE).

Références :

* [Clause ON CONFLICT](https://docs.postgresql.fr/current/sql-insert.html#SQL-ON-CONFLICT)


## Traitement des hints

L'optimiseur Oracle supporte des _hints_, qui permettent au DBA de tromper 
l'optimiseur pour lui faire prendre des chemins que l'optimiseur a jugé trop 
coûteux. Ces _hints_ sont exprimés sous la forme de commentaires et ne seront 
donc pas pris en compte par PostgreSQL, qui ne gère pas ces hints.

Néanmoins, une requête comportant un _hint_ pour contrôler l'optimiseur Oracle 
doit faire l'objet d'une attention particulière, et l'analyse de son plan 
d'exécution devra être faite minutieusement, pour s'assurer que, sous PostgreSQL, 
la requête n'a pas de problème particulier, et agir en conséquence le cas échéant. 
C'est notamment vrai lorsque l'une des tables mise en œuvre est particulièrement 
volumineuse. Mais, de manière générale, l'ensemble des requêtes portées devront 
voir leur plan d'exécution vérifié.

Le plan d'exécution de la requête sera vérifiée avec l'ordre `EXPLAIN ANALYZE` 
qui fournit non seulement le plan d'exécution en précisant les estimations de 
sélectivité réalisées par l'optimiseur, mais va également exécuter la requête 
et fournir la sélectivité réelle de chaque nœud du plan d'exécution. Une forte 
divergence entre la sélectivité estimée et réelle permet de détecter un problème. 
Souvent, il s'agit d'un problème de précision des statistiques. Il est possible 
d'agir sur cette précision de plusieurs manières.

Tout d'abord, il est possible d'augmenter le nombre d'échantillons collectés, 
pour construire notamment les histogrammes. Le paramètre `default_statistics_target` 
contrôle la précision de cet échantillon. Pour une base de forte volumétrie, ce 
paramètre sera augmenté systématiquement dans une proportion raisonnable. Pour 
une base de volumétrie normale, ce paramètre sera plutôt augmenté en ciblant une 
colonne particulière avec l'ordre SQL `ALTER TABLE ... ALTER COLUMN ... SET STATISTICS ...;`. 
De plus, il est possible de forcer artificiellement le nombre de valeurs distinctes 
d'une colonne avec l'ordre SQL `ALTER TABLE ... SET COLUMN ... SET n_distinct = ...;`. 
Il est aussi souvent utile d'envisager une réécriture de la requête : si l'optimiseur, 
sous Oracle comme sous PostgreSQL, n'arrive pas à trouver un bon plan, c'est 
probablement qu'elle est écrite d'une façon qui empêche ce dernier de travailler 
correctement.
