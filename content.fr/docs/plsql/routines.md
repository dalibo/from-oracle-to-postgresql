---
weight: 1
bookFlatSection: false
title: "Portage des procédures et fonctions"
---

## Portage des procédures et fonctions

### Déclaration des procédures et fonctions

Les deux SGBD permettent de créer des procédures et des fonctions. Les procédures 
sont appelables par les mêmes requêtes `CALL` et les fonctions par les `SELECT`.

Mais les procédures PostgreSQL ne peuvent pas retourner de valeur. Dans le cadre 
d'une migration, les procédures contenant des paramètres OUT doivent donc être 
transposées en fonctions PostgreSQL.

Il est également possible de transformer une procédure ne retournant rien en une 
fonction avec un type de retour `VOID`.

Parmi les principales différences, il faut noter :

* la clause `RETURN` est transposée en `RETURNS` ;
* le mot clé `IS` est transposé en `AS` ;
* le corps de la fonction/procédure est encadré des symboles `$$` (ou autre) ;
* la répétition du nom de la fonction/procédure après la clause `END` finale est 
inutile ;
* les variables locales sont définies dans PostgreSQL dans un bloc `DECLARE` ;
* le langage est précisé dans PostgreSQL : `LANGUAGE plpgsql`.

Ci-après, la déclaration de la procédure `cs_create_job` dans une base Oracle :

```sql
CREATE OR REPLACE PROCEDURE cs_create_job(v_job_id IN INTEGER) IS
    a_running_job_count INTEGER;
BEGIN
...
END;
```

sera portée de la façon suivante pour PostgreSQL :

```sql
CREATE OR REPLACE PROCEDURE cs_create_job(v_job_id INTEGER) AS
$$
DECLARE
  a_running_job_count INTEGER;
BEGIN
...
END;
$$ LANGUAGE plpgsql;
```

Références :

* [Structure de PL/pgSQL](https://docs.postgresql.fr/current/plpgsql-structure.html)
* [Déclarations](https://docs.postgresql.fr/current/plpgsql-declarations.html)

### Attribut DETERMINISTIC

Les fonctions Oracle déclarées avec l'attribut `DETERMINISTIC` seront portées 
en fonctions PostgreSQL avec un attribut `IMMUTABLE`. `IMMUTABLE` et 
`DETERMINISTIC` indiquent que la fonction ne peut pas modifier la base de 
données et qu'à arguments constants, la fonction retourne toujours le même 
résultat. Cette fonctionnalité est supportée par Ora2Pg.

Ainsi, la déclaration suivante pour Oracle :

```sql
CREATE OR REPLACE FUNCTION Get_Average_Char(input_ VARCHAR2)
   RETURN VARCHAR2 DETERMINISTIC
IS
...
END Get_Average_Char;
```

sera transposée de la façon suivante pour PostgreSQL :

```sql
CREATE OR REPLACE FUNCTION get_average_char(input_ VARCHAR)
   RETURNS VARCHAR
AS $$
...
END;
$$ LANGUAGE plpgsql
IMMUTABLE;
```

Références :

* [CREATE FUNCTION](https://docs.postgresql.fr/current/sql-createfunction.html)

### Attribut PIPELINED et instruction PIPE ROW

Les fonctions Oracle déclarées avec l'attribut `PIPELINED` sont équivalentes 
aux fonctions `SRF` ou _Set-Returning Functions_ (fonctions retournant des 
ensembles) de PostgreSQL. Ainsi, la fonction avec un attribut `PIPELINED` sera
transposé avec une directive `SETOF`. L'instruction `PIPE ROW` sera transposée 
par `RETURN NEXT`.

Le package suivant, extrait d'un [exemple PL/SQL][LNPLS918] Oracle implémente l'attribut
`PIPELINED` et l'instruction `PIPE ROW` :

[LNPLS918]: https://docs.oracle.com/cd/E11882_01/appdev.112/e25519/tuning.htm#LNPLS918

```sql
CREATE OR REPLACE PACKAGE pkg1 AS
  TYPE numset_t IS TABLE OF NUMBER;
  FUNCTION f1(x NUMBER) RETURN numset_t PIPELINED;
END pkg1;

CREATE PACKAGE BODY pkg1 AS
  -- FUNCTION f1 RETURNS a collection of elements (1,2,3,... x)
  FUNCTION f1(x NUMBER) RETURN numset_t PIPELINED IS
  BEGIN
    FOR i IN 1..x LOOP
      PIPE ROW(i);
    END LOOP;
    RETURN;
  END f1;
END pkg1;
```

Il est porté assez simplement par Ora2Pg :

* le type `numset_t` n'est pas repris, il faudra le modifier manuellement après 
le portage par Ora2Pg ;
* si le type de retour était du type `%ROWTYPE` sous Oracle, il aurait été porté
en `SET OF record` ;
* le package pkg1 est transformé en schéma pkg1 ;
* le code de la fonction f1 est transposé comme ci-après.

Voici le code PL/pgSQL porté :

```sql
CREATE SCHEMA pkg1;
CREATE OR REPLACE FUNCTION pkg1.f1 (x INTEGER) RETURNS SET OF INTEGER AS $$
DECLARE
  i INTEGER;
BEGIN
  FOR i IN 1..x LOOP
    RETURN NEXT i;
  END LOOP;
  RETURN;
END;
$$ LANGUAGE plpgsql;
```

La fonction sera appelée de la façon suivante : `SELECT pkg1.f1(10);`.

Références :

* [Structures de contrôles en PL/pgSQL](https://docs.postgresql.fr/current/plpgsql-control-structures.html)

### Paramètres de fonctions

Oracle ne se conforme pas complètement à la norme SQL quant à la déclaration des 
paramètres d'une fonction ou d'une procédure. PostgreSQL, quant à lui, se conforme 
à la norme SQL. Des adaptations mineures sont donc nécessaires pour ce qui concerne 
le portage de la déclaration des paramètres d'une fonction.

Chaque argument est nommé, a un type de données et un mode déterminant le 
comportement du paramètre. Sous Oracle, le mode du paramètre est déclaré après le 
nom du paramètre. Sous PostgreSQL, le mode d'appel vient avant le nom du paramètre, 
conformément à la norme SQL. Les modes d'appels `IN` et `OUT` sont transposés
directement, sauf `IN OUT` qui devient `INOUT`. Le mode du paramètre peut être 
ignoré pour les paramètres entrants `IN`.

Ainsi, les déclarations de paramètres suivants de fonctions Oracle :

```sql
CREATE FUNCTION test (p INTEGER) ...
CREATE OR REPLACE PROCEDURE cs_parse_url(
  v_url IN VARCHAR, v_host IN OUT VARCHAR, 
  v_path OUT VARCHAR, v_query OUT VARCHAR
) ...
```

seront transposés de cette manière pour PostgreSQL :

```sql
CREATE FUNCTION test (p INTEGER) ...
CREATE OR REPLACE FUNCTION cs_parse_url(
  IN v_url VARCHAR, INOUT v_host VARCHAR, 
  OUT v_path VARCHAR, OUT v_query VARCHAR
) ...
```

Références :

* [Fonctions avec paramètres en sortie](https://docs.postgresql.fr/current/xfunc-sql.html#xfunc-output-parameters)

### Valeurs par défaut d'un argument

PostgreSQL supporte les valeurs par défaut des arguments. Les déclarations 
de fonctions qui possèdent des valeurs par défaut pour les arguments 
peuvent être reprises sans adaptation.

Par exemple, une fonction telle que la suivante sous Oracle :

```sql
CREATE FUNCTION fonction1 (a INT, b INT, c INT := 0) ...
```

sera réécrite de la façon suivante dans PostgreSQL :

```sql
CREATE FUNCTION fonction1 (a INT, b INT, c INT = 0) ...
```

Le mot-clé `DEFAULT` est valable dans les deux langages.

Références :

* [CREATE FUNCTION](https://docs.postgresql.fr/current/sql-createfunction.html)
