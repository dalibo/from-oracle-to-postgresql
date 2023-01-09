---
weight: 1
bookFlatSection: false
title: "Différences générales"
url: "differences-generales-entre-oracle-et-postgresql"
previouspage: "/"
nextpage: "/schema"
---

# Différences générales entre Oracle et PostgreSQL

Cette partie traite des spécificités et des différences générales à prendre en
considération dans le cadre de la migration d'une base de données Oracle vers
PostgreSQL.


## Utilisateur et schéma

Oracle confond la notion d'utilisateur et de schéma. La création d'un utilisateur
implique la création d'un schéma portant le même nom.

PostgreSQL distingue clairement un utilisateur d'un schéma. Le schéma est un réel
espace de nommage pour les objets.

## Sensibilité à la casse

Oracle convertit les noms d'objets en majuscule, tandis que PostgreSQL les convertit
en minuscule. La norme SQL ne faisant aucune recommandation à ce sujet. 

La casse peut être forcée au besoin en encadrant le nom de l'objet manipulé par
deux guillemets doubles (_double-quotes_).

Cependant, cette pratique est déconseillée fortement sous PostgreSQL, dans la
mesure où la table `"MaTable"` ne peut être accessible qu'avec les guillemets doubles,
de manière systématique :

```sql
CREATE TABLE "MaTable" (a INTEGER PRIMARY KEY, b INTEGER);
-- NOTICE:  CREATE TABLE / PRIMARY KEY will create implicit index "MaTable_pkey"
--  for table "MaTable"
-- CREATE TABLE

INSERT INTO matable (a,b) VALUES (1,1);
-- ERROR:  relation "matable" does not exist
-- LINE 1: INSERT INTO matable (a,b) values (1,1);
     
INSERT INTO MaTable (a,b) VALUES (1,1);
-- ERROR:  relation "matable" does not exist
-- LINE 1: INSERT INTO MaTable (a,b) values (1,1);
  
INSERT INTO "MaTable" (a,b) VALUES (1,1);
-- INSERT 0 1
```

Ainsi, si on oublie les guillemets doubles dans le code, PostgreSQL convertira
systématiquement le nom de table saisi en minuscules.

## Table DUAL

L'analyseur d'Oracle n'accepte pas l'absence d'une clause `FROM` sur un `SELECT`.
PostgreSQL n'a pas cette limitation, les références à la table `DUAL` dans une
requête peuvent être supprimées.

À noter qu'il est contre-performant de créer artificiellement une table `DUAL`
dans PostgreSQL car elle nécessitera l'acquisition inutile d'un verrou et peut
être source de contentions pour les requêtes utilisant cette table.

```sql
SELECT 1+1 AS resultat;
-- -[ RECORD 1 ]
-- resultat | 2
```

## Traitement de la valeur NULL

Le type de données `VARCHAR2` assimile la chaîne vide à la valeur `NULL`.
Ce comportement n'est pas respectueux de la norme SQL.

Il peut s'ensuivre des problèmes pour la comparaison écrite avec ces spécificités
en tête. De même, Oracle permet de concaténer une chaîne avec une valeur `NULL`
sans problèmes. Avec PostgreSQL, la valeur `NULL` est propagée dans les opérations :
une valeur `NULL` concaténée à une chaîne de caractère donne `NULL`.

```sql
SELECT 
  'ABC'||NULL AS concatenation_avec_null, 
  coalesce('ABC'||NULL,'valeur si null','valeur si non null');
-- -[ RECORD 1 ]-----------+---------------
-- concatenation_avec_null | 
-- coalesce                | valeur si null
```

## Database Links

PostgreSQL propose un module externe nommé `dblink` pour accéder à une base
PostgreSQL distante.

PostgreSQL implémente également le sous-ensemble SQL/MED de la norme SQL. Ces 
fonctionnalités permettent de définir des _Foreign Data Wrapper_ (FDW) qui 
permettent d'accéder à des objets externes à la base de données : une autre 
base de données, un fichier plat, et de nombreuses autres sources de données.

Parmi ceux-ci, on trouve PostgreSQL, naturellement, mais aussi Oracle. Les 
FDW permettent ainsi d'intégrer une base PostgreSQL dans un eco-système de 
bases Oracle. Concrétement, on définit dans la base PostgreSQL locale des 
`FOREIGN TABLEs` , associées à des tables hébergées sur un serveur distant. 
Dès lors, on peut facilement requêter ces tables en SQL, que ce soit pour 
de la lecture ou de la mise à jour.

Un certain nombre de mécanismes, tels que la collecte de statistiques sur 
les tables distantes ou la transmission de prédicats de requête sur le serveur 
distant, rend ces accès assez efficaces. Néanmoins, il faut garder en tête que 
les performances obtenues sont généralement moindres que dans le cas où toutes 
les données sont localisées dans la même base.

Références :

* [Liste de foreign data wrappers](https://wiki.postgresql.org/wiki/Foreign_data_wrappers)
* [CREATE FOREIGN TABLE](https://docs.postgresql.fr/current/sql-createforeigntable.html) PostgreSQL 14.
