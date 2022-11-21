---
weight: 2
bookFlatSection: false
title: "Portage du schéma"
url: "portage-du-schema-de-la-base-de-donnees"
bookCollapseSection: true
---

# Portage du schéma de la base de données

## Compatibilités des ordres DDL

Dans l'ensemble, les ordres DDL sont compatibles entre Oracle et PostgreSQL.

On notera une différence fondamentale entre les deux SGBD :

  * Oracle effectue un COMMIT implicite après chaque ordre DDL ;
  * au contraire, les ordres DDL sont transactionnels dans PostgreSQL ;
  * les DDL concurrents sont exclusifs à Oracle, mais PostgreSQL propose un
    `CREATE INDEX CONCURRENTLY`.

```sql
BEGIN;

CREATE TABLE matable (a INTEGER PRIMARY KEY, b INTEGER);
-- NOTICE:  CREATE TABLE / PRIMARY KEY will create implicit index "matable_pkey"
--  for TABLE "matable"
-- CREATE TABLE

SELECT count(*) FROM matable;
-- -[ RECORD 1 ]
-- count | 0

ROLLBACK;

SELECT count(*) FROM matable;
-- ERROR:  relation "matable" does not exist
-- LINE 1: SELECT count(*) FROM matable;
```

## Reprise des contraintes

La reprise des contraintes d'intégrité ne pose pas de difficulté particulière.

## Reprise des synonymes

Les synonymes Oracle n'ont pas d'équivalent direct dans PostgreSQL. 

Deux cas de figure sont à traiter pour reprendre les fonctionnalités attachées aux 
synonymes.

Si le synonyme est en fait un homonyme, permettant par exemple de rendre visible 
une table d'un schéma dans un autre schéma, alors, il suffira de valoriser 
judicieusement la variable de session PostgreSQL `search_path` qui indique la 
liste de schémas dans laquelle PostgreSQL doit chercher les objets dont le 
schéma n'est pas qualifié. Il est possible de spécifier cette variable pour 
chaque utilisateur de la base de données, ou dynamiquement à n'importe quel 
moment dans une session avec l'ordre 
`SET search_path = liste schémas ...;`.

Si le synonyme n'est pas un homonyme, il est commun de créer une vue pour 
rendre visible une table dans un autre schéma et sous un autre nom, ou une 
fonction d'encapsulation pour rendre visible une fonction dans un autre 
schéma sous un autre nom.

Références :

* [Chemin de parcours des schémas](https://docs.postgresql.fr/current/ddl-schemas.html#ddl-schemas-path)

