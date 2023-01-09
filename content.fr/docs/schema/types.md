---
weight: 1
bookFlatSection: false
title: "Correspondance des types de données"
previouspage: "/schema"
nextpage: "tables"
---

## Correspondance des types de données

Le SGBD Oracle ne supporte pas l'ensemble des types de données spécifiés par la
norme SQL. Il s'agit notamment des types numériques _natifs_ tels que `SMALLINT`,
`INT` et `BIGINT`. Le type `VARCHAR2` est également une spécificité Oracle,
notamment dans sa gestion de la valeur NULL. De plus, un certain nombre de types
spécifiques à Oracle sont nommés différemment dans PostgreSQL.

| Types Oracle | Types PostgreSQL |
|--------------|-----------------|
| varchar2, nchar2, nvarchar2, nclob | varchar ou text |
| clob, long | varchar ou text |
| blob, raw, long raw | bytea |
| number | numeric, integer, bigint, smallint, real, double precision |
| date | date ou timestamp |
| binary float | real |
| binary double | double precision |


### Traitement des booléens

Oracle ne possède pas de type de données représentant des booléens. De ce fait,
une chaîne de caractères est souvent utilisée. Ora2Pg permet de convertir des
données représentant un booléen vers PostgreSQL.

Par ailleurs, l'absence de type booléen dans Oracle entraîne des difficultés avec
des ORM tel que Hibernate. En effet, la colonne configurée comme un booléen dans
l'ORM sera représenté physiquement dans la base Oracle comme un type entier ou un
type texte. Or, si le portage de la base de données ne tient pas compte de cette
problématique, l'ORM cherchera un booléen dans la base PostgreSQL mais trouvera
le type équivalent à celui de la base Oracle.

```sql
SELECT true AND false;
-- -[ RECORD 1 ]
-- ?column? | f

SELECT true OR false;
-- -[ RECORD 1 ]
-- ?column? | t
```

### Différences sur les types caractères

Oracle implémente un type de données spécifique, `VARCHAR2`. Parmi ses spécificités,
il confond la chaîne vide et la valeur `NULL`, ce qui est contraire à la norme SQL.
Ainsi, sous Oracle, une comparaison avec `''` correspond à `IS NULL`.

PostgreSQL, quant à lui, propose un type de données `VARCHAR` qui est conforme à
la norme. La comparaison avec `''` est distincte de `IS NULL`. Ce type `VARCHAR`
est limitée à 1 Go. PostgreSQL propose également un type `text` qui est équivalent à
un `VARCHAR` sans limite de taille.

Il faut s'attendre aussi à des différences de comportement lors de la comparaison
entre types `CHAR` et `VARCHAR`. C'est une zone assez floue de la norme SQL, et
chaque moteur a sa propre interprétation.

Lors de la comparaison de ces deux types, PostgreSQL choisit de convertir le
`VARCHAR` en `CHAR`. Il applique ensuite une comparaison avec du remplissage
(_padding_) à droite par des blancs jusqu'à la longueur du `CHAR(x)`.

Par exemple :

```sql
SELECT 'foo'::CHAR(5)='foo '::VARCHAR(5);
--  ?column? 
-- ----------
--  t

SELECT 'foo '::CHAR(5)='foo'::VARCHAR(5);
--  ?column? 
-- ----------
--  t

SELECT 'foo '::VARCHAR(5)='foo'::VARCHAR(5);
--  ?column? 
-- ----------
--  f
```

Ceci peut avoir une grande importance, non seulement sur les résultats, mais aussi
sur les plans d'exécution :

```sql
CREATE TABLE t1 (a VARCHAR(5));

INSERT INTO t1 SELECT generate_series(1,10000);

CREATE INDEX idx1 ON t1(a);

EXPLAIN SELECT * FROM t1 WHERE a='foo'::CHAR(5);
--                      QUERY PLAN                     
-- ----------------------------------------------------
--  Seq Scan on t1  (cost=0.00..170.00 rows=1 width=4)
--    Filter: ((a)::bpchar = 'foo '::character(5))
```

Cette requête ne peut pas utiliser l'index sur la colonne `a` : la colonne est
convertie en `CHAR` (`bpchar` en fait, qui est le type interne de PostgreSQL
pour `CHAR`) pour chaque enregistrement, avant comparaison.

Ce problème apparaît fréquemment quand la requête est embarquée dans une fonction,
et que les paramètres de cette fonction sont de type `CHAR(x)`. Par exemple :

```sql
CREATE OR REPLACE FUNCTION public.demo_char(p1 CHARACTER)
 RETURNS CHARACTER VARYING
 LANGUAGE plpgsql
AS $function$
DECLARE
  v1 VARCHAR;
BEGIN
  SELECT INTO v1 a FROM t1 WHERE a=p1;
  RETURN v1;
END
$function$
```

Dans cette fonction, le `SELECT` réalisera un parcours complet de `t1`, 
systématiquement : il faut convertir la valeur de `a` vers un `CHAR` pour comparaison.

Références :

* [Types caractères](https://docs.postgresql.fr/current/datatype-character.html)

### Encodage des caractères et collationnement

L'encodage au niveau serveur est défini au niveau d'une base de données PostgreSQL.
La variable de session `client_encoding` permet de définir l'encodage au niveau
du client. La valeur par défaut est héritée de l'encodage de la base de données
sur laquelle le client est connecté, mais elle peut être redéfinie dynamiquement
en début de session avec l'ordre `SET`, par exemple : `SET client_encoding = UTF8`.

Concernant le collationnement, il faut noter un certain nombre d'évolutions
depuis la version 8.4 :

* collationnement par instance avant la version 8.4 ;
* collationnement par base de données depuis la version 8.4 ;
* collationnement par colonne/index/ordre SQL depuis la version 9.1.

Références :

* [Support des collations](https://docs.postgresql.fr/current/collation.html)

### Type de données date et heure

Oracle propose plusieurs types de données pour manipuler les dates :

* `DATE`, qui encode la date et l'heure jusqu'à la seconde ;
* `TIMESTAMP` qui encode la date et l'heure, dont la précision peut aller plus
loin que celle du type équivalent de PostgreSQL ;
* `TIMESTAMP WITH TIME ZONE` qui permet également d'encoder une information
concernant le fuseau horaire ;
* `INTERVAL` qui offre deux précisions : année/mois ou jour/secondes.

PostgreSQL, quant à lui, implémente les types de données suivants :

* `date`, qui encode uniquement une date, conformément au standard SQL ;
* `TIMESTAMP` qui encode la date et l'heure, avec une résolution de 1 microseconde ;
* `TIME` qui encode uniquement l'heure, avec une résolution de 1 microseconde ;
* `INTERVAL` qui offre une précision de 1 microseconde ;
* `TIME` et `TIMESTAMP` peuvent optionnellement embarquer une timezone (fuseau
horaire), en rajoutant `WITH TIME ZONE` à la déclaration du type, comme pour Oracle.

La difficulté du portage d'une colonne de type `DATE` dans une base Oracle réside
dans la difficulté à déterminer si la colonne ne stocke que des dates ou si elle
est utilisée pour stocker des dates avec l'heure.

```sql
SELECT ('1970-01-01'::DATE 
      + '15 YEARS 3 MONTHS 2 DAYS 1 HOUR 23 MINUTES'::INTERVAL) 
    AS calcul_intervalle;
-- -[ RECORD 1 ]-----+--------------------
-- calcul_intervalle | 1985-04-03 01:23:00
```

Références :

* [Types date/heure](https://docs.postgresql.fr/current/datatype-datetime.html)

### Traitement des types composites

L'ensemble des types pouvant être définis par un utilisateur sont supportés avec
plus ou moins d'adaptation. Il peut notamment être nécessaire de redéfinir des
fonctions d'entrée / sortie définissant le comportement lors d'une insertion / lecture
sur les données du type. Dans la plupart des cas, il s'agit de types composites 
ou de tableaux parfaitement supportés par PostgreSQL.

Exemple de type composite version Oracle :

```sql
CREATE OR REPLACE TYPE phone_t AS OBJECT (
    a_code   CHAR(3),
    p_number CHAR(8)
);
```

et version PostgreSQL :

```sql
CREATE TYPE phone_t AS (
    a_code   CHAR(3),
    p_number CHAR(8)
);
```

Exemple d'un tableau de type :

```sql
CREATE OR REPLACE TYPE phonelist AS VARRAY(50) OF phone_t;
```

qui sera traduit en :

```sql
CREATE TYPE phonelist AS (phonelist phone_t[50]);
```
