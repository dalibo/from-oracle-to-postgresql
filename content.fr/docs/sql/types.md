---
weight: 1
bookFlatSection: false
title: "Spécificités des types de données"
---

## Spécificités des types de données

### Manipulation des types VARCHAR

Pour Oracle, une chaine vide est aussi une chaîne `NULL`. Elle a les deux 
propriétés à la fois. PostgreSQL fait la différence : soit la chaîne est inconnue 
(`IS NULL`) soit elle est vide.

Certaines requêtes donnant satisfaction sous Oracle peuvent donner des résultats 
faux lorsqu'elles sont portées directement. Les cas les plus courants sont la 
comparaison de la valeur d'une colonne avec une chaîne vide et la concaténation 
avec une valeur `NULL`.

#### Comparaison

Pour Oracle, si la colonne `col` est un `VARCHAR2`, les deux requêtes 
suivantes vont retourner le même résultat :

```sql
SELECT * FROM TABLE WHERE col = '';
SELECT * FROM TABLE WHERE col IS NULL;
```

Mais sous PostgreSQL, les deux requêtes ne retourneront pas le même résultat. 
Ora2Pg transpose les valeurs `NULL` des colonnes en `VARCHAR2` en valeur 
`NULL` pour un type `VARCHAR`. La première requête ne retournera aucun 
résultat.

Il est donc nécessaire de réécrire la requête concernée en utilisant les opérateurs
`IS NULL` ou `IS NOT NULL`. Dans ce cas, la requête pourra utiliser un index 
pour la recherche.

Si la comparaison est transposée avec une fonction `COALESCE` et en conservant 
la comparaison avec la chaîne vide, il ne sera pas possible d'utiliser un index 
pour la recherche.

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

#### Concaténation

Du fait de la spécificité du type `VARCHAR2` d'Oracle, la concaténation d'une 
valeur NULL dans une chaîne de caractères ne pose pas de problèmes. Elle en pose
cependant dans PostgreSQL. Le standard SQL définit en effet que pour la plupart 
des fonctions, un paramètre NULL entraîne un résultat NULL (on parle de fonctions 
`STRICT` dans PostgreSQL). Dans le cas présent, une valeur NULL dans une 
opération de concaténation sera propagée au résultat : le résultat sera une 
chaîne `NULL` et non la chaîne de caractères attendue.

On prendra donc garde à ajouter la fonction `COALESCE` dans les portions de 
code touchées par ce problème :

```sql
SELECT 'nom employé: ' || COALESCE(ename, `) FROM emp;
```

### Manipulation de dates

#### Format de sortie

Pour Oracle, le format défini par `NLS_DATE_FORMAT` détermine le format des 
dates qui sera utilisé pour la sortie des fonctions `TO_CHAR()` et `TO_DATE()`.

Concernant PostgreSQL, par défaut, le format de sortie d'une date est conforme 
au standard SQL: `YYYY-MM-DD HH24:MI:SS.mmmmmmm+TZ`. Il est cependant possible 
de le modifier avec la variable de configuration `DateStyle` (par défaut `ISO, DMY`).

#### Opérations sur les dates

Oracle autorise l'ajout ou la soustraction d'un nombre entier à une date. 
Par exemple :

```sql
SELECT SYSDATE + 1 FROM DUAL;
```

retournera la date de demain. Pour le type date de PostgreSQL, le comportement
est identique, mais les deux types `date` ne le sont pas.

Pour obtenir le même résultat avec un `TIMESTAMP` (l'équivalent du type `DATE` d'Oracle) 
sous PostgreSQL, il faut utiliser un `INTERVAL` :

```sql
SELECT now() + INTERVAL '1 DAY';
```

De même, la soustraction d'un `TIMESTAMP` à un autre retourne un nombre 
correspondant au nombre de jour entre ces deux dates, alors que, sous PostgreSQL,
cette opération retourne un intervalle.

#### Portage de SYSDATE

Attention au type de données `DATE` sous Oracle, c'est en fait du 
`TIMESTAMP WITHOUT TIMEZONE`.

```sql
SELECT to_char(sysdate,'DD/MM/YYYY HH24:MI:SS') FROM dual;

-- TO_CHAR(SYSDATE,'DD
-- -------------------
-- 11/03/2011 14:58:22
```

Il convient donc d'utiliser `LOCALTIMESTAMP` (et pas `current_timestamp` 
qui, lui, contient la _timezone_…) :

```sql
SELECT LOCALTIMESTAMP;
--          timestamp          
-- ----------------------------
--  2011-03-11 16:59:29.889823
```

Ainsi, toute variable déclarée en DATE dans du PL/SQL d'Oracle doit être 
déclarée en TIMESTAMP dans du PL/pgsql de PostgreSQL.
