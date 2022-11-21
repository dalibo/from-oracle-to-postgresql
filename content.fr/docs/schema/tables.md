---
weight: 2
bookFlatSection: false
title: "Reprise des tables"
---

## Reprise des tables

La définition des tables est quasiment identique pour les deux SGBD à la différence
près que PostgreSQL n'a pas de table temporaire globale. Les tables temporaires
sont privées à une session. Les données insérées ne persistent que le temps d'une
transaction (avec la clause `ON COMMIT DELETE ROWS` ou d'une session (avec la clause
`ON COMMIT PRESERVE ROWS`). Sous PostgreSQL c'est la table elle même qui est 
supprimée à la fin de la session ou de la transaction (avec la clause `ON COMMIT DROP`).
Une implémentation équivalente aux tables temporaires globales est proposée plus 
loin dans ce document.

Il n'y a pas non plus de notion de réservation de nombre de transactions allouées
à chaque bloc ou d'_extents_. Il n'existe donc pas d'équivalent à `INITTRANS`
et `MAXEXTENTS`.

L'option `PCTFREE` qui indique (en pourcentage) l'espace que l'on souhaite conserver
dans le bloc pour les mises à jour correspond au _fillfactor_ sous PostgreSQL.
`PCTUSED` n'existe pas (il n'a pas de sens dans l'implémentation de PostgreSQL).

Références :

* [CREATE TABLE](https://docs.postgresql.fr/current/sql-createtable.html)

### Colonnes virtuelles

Il existe deux façons de porter les colonnes virtuelles Oracle, définies avec la 
clause `VIRTUAL`.

La plus simple consiste à transposer la colonne virtuelle en une colonne 
calculée. Contrairement à la version Oracle, dans la version PostgreSQL la 
colonne de la table occupera de la place sur disque. Mais l'utilisation de la 
table est ensuite strictement identique.

Voici un exemple de définition de colonne virtuelle sous Oracle :

```sql
CREATE TABLE employees (
  id          NUMBER,
  first_name  VARCHAR2(10),
  salary      NUMBER(9,2),
  commission  NUMBER(3),
  salary2     NUMBER GENERATED ALWAYS 
    AS (ROUND(salary*(1+commission/100),2)) VIRTUAL,
);
```

Cette table sera portée dans PostgreSQL de la façon suivante :

```sql
CREATE TABLE employees (
  id          BIGINT,
  first_name  VARCHAR(10),
  salary      DOUBLE PRECISION,
  commission  INTEGER,
  salary2     DOUBLE PRECISION GENERATED ALWAYS 
    AS (ROUND((salary*(1+commission/100))::numeric,2)) STORED
);
```

Si le stockage de la colonne sur disque est considérée comme gênant, il est 
aussi possible de créer une vue.

```sql
CREATE TABLE employees (
  id          BIGINT,
  first_name  VARCHAR(10),
  salary      DOUBLE PRECISION,
  commission  INTEGER
);

CREATE VIEW virt_employees AS
SELECT id, first_name, salary, commission, 
        (ROUND((salary*(1+commission/100))::numeric,2)) salary2
  FROM employees;
```

Références :

* [CREATE TABLE](https://docs.postgresql.fr/current/sql-createtable.html)
* [CREATE VIEW](https://docs.postgresql.fr/current/sql-createview.html)

### Index-Organized Tables

PostgreSQL ne propose pas d'équivalent direct des _IOT_.

Il est néanmoins possible d'organiser une table selon un index à l'aide de l'ordre
`CLUSTER`. Cependant, toutes les écritures ultérieures à la réorganisation ne
tiendront pas du tout compte de l'ordre de l'index. Il est possible de diminuer
ce phénomène en appliquant un paramètre `fillfactor` inférieur à 100 pour laisser
de la place aux nouvelles versions des lignes mises à jour car les mises à jour
sont conservées sur la même page si l'espace libre est suffisant.

Une table organisée selon un index, ou table _clusterisée_, peut apporter un gain 
significatif en performance sur les requêtes de type _range scan_, où des portions
consécutives de la table sont ramenées par la requête. Par exemple, une requête
qui ramène tous les clients dont le nom commence par `MAR`
(`SELECT * FROM clients WHERE nom LIKE 'MAR%';`) bénéficiera d'une table organisée
selon un index. La lecture sera alors plus rapide car les différentes lignes
pointées par l'index sont normalement dans le même bloc de données.

`CLUSTER` est donc une opération ponctuelle, à renouveler plus ou moins fréquemment
selon le taux d'écriture dans la table et des fenêtres de maintenance des applications.
Il faut garder à l'esprit que cette opération est une opération lourde qui nécessite
l'acquisition d'un verrou d'accès exclusif qui bloquera toutes les lectures et
écritures durant le temps de la réorganisation. Par ailleurs, aucune réindexation
n'est nécessaire à la suite d'un `CLUSTER` car les index sont reconstruits à
l'issue de la réorganisation.

L'exemple ci-dessous montre un gain significatif des performances sur une table
simple comportant 10 millions de lignes. Le temps d'exécution de la requête est
divisé par deux après avoir effectué un `CLUSTER` sur l'index approprié.

Création des objets :

```sql
CREATE TABLE t6 (i SERIAL PRIMARY KEY, v INTEGER);
INSERT INTO t6 (v) SELECT round(random()*10000) FROM generate_series(1, 10000000);
CREATE INDEX idx_t6_v ON t6 (v);
```

Le temps d'exécution de la requête suivante est constant, les données étant en
cache :

```sql
SELECT count(*) FROM t6 WHERE v BETWEEN 100 AND 1000;
--  count  
-- --------
--  902274
-- 
-- Time: 350,100 ms
```

La table est réorganisée selon l'index `idx_t6_v` :

```sql
CLUSTER t6 USING idx_t6_v;
```

Le temps d'exécution de la requête précédente est divisé par deux :

```sql
SELECT count(*) FROM t6 WHERE v BETWEEN 100 AND 1000;
--  count  
-- --------
--  902274
-- 
-- Time: 148,755 ms
```

Références :

* [CLUSTER](https://docs.postgresql.fr/current/sql-cluster.html)

### Tables externes

Les tables externes permettent de manipuler des fichiers de type CSV depuis une
base de données Oracle. Cette fonctionnalité peut être émulée en utilisant un
_Foreign Data Wrapper_. Dans le chapitre précédent relatif aux Database Links, 
nous avons déjà présenté ces mécanismes d'accès à des données distantes. Il 
s'agit ici de les appliquer pour des fichiers plats, par exemple au format CSV.

Soit la définition de la table externe suivante pour Oracle :

```sql
CREATE OR REPLACE DIRECTORY ext AS '/usr/tmp/';
CREATE TABLE ext_tab (
  empno  CHAR(4),
  ename  CHAR(20),
  job    CHAR(20),
  deptno CHAR(2)
) ORGANIZATION EXTERNAL (
  TYPE oracle_loader
  DEFAULT DIRECTORY ext
    ACCESS PARAMETERS (
    RECORDS DELIMITED BY NEWLINE
    BADFILE 'bad_%a_%p.bad'
    LOGFILE 'log_%a_%p.log'
    FIELDS TERMINATED BY ','
    MISSING FIELD VALUES ARE NULL
    REJECT ROWS WITH ALL NULL FIELDS
    (empno, ename, job, deptno))
    LOCATION ('demo1.dat')
  )
PARALLEL
REJECT LIMIT 0
NOMONITORING;
```

Elle sera convertie de la façon suivante dans PostgreSQL :

```sql
CREATE EXTENSION file_fdw;

CREATE SERVER ext FOREIGN DATA WRAPPER file_fdw;

CREATE FOREIGN TABLE ext_tab (
        empno CHAR(4),
        ename CHAR(20),
        job CHAR(20),
        deptno CHAR(2)
) SERVER ext OPTIONS(filename '/usr/tmp/demo1.dat', format 'csv',
delimiter ',');
```

S'il s'agit seulement de charger des données, il est à noter que l'ordre SQL
`COPY` de PostgreSQL est également une solution alternative à la création d'un
_foreign data wrapper_. Par exemple, pour charger le fichier CSV de l'exemple,
il suffit d'exécuter l'ordre SQL suivant :

```sql
COPY ext_table FROM '/usr/tmp/demo1.csv' WITH CSV;
```

Références :

* [file_fdw](https://docs.postgresql.fr/current/file-fdw.html)
* [COPY](https://docs.postgresql.fr/current/sql-copy.html)

### Tables temporaires Globales

L'objet `GLOBAL TEMPORARY TABLE` Oracle n'a pas d'équivalent dans PostgreSQL. 
Pour rappel, une table temporaire globale est une table dont la structure est 
permanente, mais dont le contenu est temporaire, c'est à dire propre à chaque 
session utilisateur.

Plusieurs solutions existent pour porter ces objets.

En fonction de l'usage qui en est fait, il peut arriver qu'une table classique 
puisse répondre au besoin, avec un simple vidage de la table par une 
commande `TRUNCATE` lors de la première utilisation. Mais ceci empêche une 
utilisation de la table par plusieurs utilisateurs en parallèle.

L'extension [pgtt](https://github.com/darold/pgtt), développée par Gilles 
Darold émule le fonctionnement des tables temporaires globales. Mais son 
utilisation nécessite que soit chargée la bibliothèque de l'extension une 
fois que l'utilisateur est connecté.

Il est également possible de simuler le fonctionnement des tables temporaires 
globales de la manière suivante :

* créer un jeu de tables dans un schéma dédié avec la structure des tables
  temporaires globales à émuler ; ces tables vont servir de modèle pour la
  création de tables temporaires ;
* associer à chaque table un commentaire pour indiquer le mode de fonctionnement
  de la table temporaire lors des COMMIT de transaction (`ON COMMIT PRESERVE
  ROWS` ou `ON COMMIT DROP`) ;
* créer une procédure ou fonction d'initialisation d'une table temporaire sur la
  base d'une table modèle préexistante ;
* dans le code PL/pgSQL, appeler cette procédure ou fonction à la première
  utilisation de la table temporaire globale.

```sql
CREATE SCHEMA IF NOT EXISTS gtt_schema;
CREATE TABLE gtt_schema.gtt_table_1 (
  col1 INTEGER,
  col2 TEXT
);
COMMENT ON TABLE gtt_schema.gtt_table_1 IS 'ON COMMIT PRESERVE ROWS';
```

```sql
CREATE OR REPLACE PROCEDURE prepare_temp_table(p_relname varchar) AS
$$
DECLARE
  v_temp_schema    varchar = 'gtt_schema';
  v_temp_desc      varchar;
BEGIN
  -- Lecture du commentaire associé à la table
  v_temp_desc := pg_catalog.obj_description(
    (format('%s.%s', v_temp_schema, p_relname))::regclass, 'pg_class'
  );

  -- Création de la table temporaire
  EXECUTE format(
    'CREATE TEMP TABLE %2$s (LIKE %1$s.%2$s) %3$s',
    v_temp_schema, p_relname, 
    CASE
      WHEN v_temp_desc ~* 'delete' THEN 'ON COMMIT DELETE ROWS'
      WHEN v_temp_desc ~* 'drop' THEN 'ON COMMIT DROP'
      ELSE 'ON COMMIT PRESERVE ROWS'
    END
  );

  -- Si la table temporaire existe déjà, on la vide
  EXCEPTION WHEN SQLSTATE '42P07' THEN
    EXECUTE format('TRUNCATE %s', p_relname);
END;
$$
LANGUAGE plpgsql;

CALL prepare_temp_table('gtt_table_1');
```

Références :

* [CREATE TABLE](https://docs.postgresql.fr/current/sql-createtable.html).