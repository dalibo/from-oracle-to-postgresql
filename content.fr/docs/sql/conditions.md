---
weight: 3
bookFlatSection: false
title: "Expressions conditionnelles"
previouspage: "joins"
nextpage: "hierarchies"
---

## Expressions conditionnelles

Bien qu'Oracle implémente les différentes expressions conditionnelles telles 
qu'elles sont spécifiées dans la norme SQL, encore trop de requêtes SQL utilisent 
les fonctions _historiques_ Oracle.

### Portage de DECODE

La fonction `DECODE` d'Oracle est un équivalent propriétaire de la clause 
`CASE`, qui est normalisée.

La construction suivante utilise la fonction `DECODE` :

```sql
SELECT emp_name,
       decode(
          trunc((yrs_of_service + 3) / 4),
          0, 0.04,
          1, 0.04,
          0.06
       ) AS perc_value
  FROM employees;
```

Cette construction doit être réécrite de cette façon :

```sql
SELECT emp_name,
       CASE WHEN trunc(yrs_of_service + 3) / 4 = 0 THEN 0.04
            WHEN trunc(yrs_of_service + 3) / 4 = 1 THEN 0.04
            ELSE 0.06
       END
  FROM employees;
```

Cet autre exemple :

```sql
DECODE('user_status','active','username',NULL)
```

sera transposé de cette façon :

```sql
CASE WHEN user_status='active' THEN username ELSE NULL END
```

Attention aux commentaires entre le `WHEN` et le `THEN` qui ne sont pas 
supportés par PostgreSQL.

Références :

* [Expressions conditionnelles](https://docs.postgresql.fr/current/functions-conditional.html)

### Portage de NVL

La fonction `NVL` d'Oracle est encore souvent utilisée, bien que la fonction 
normalisée `COALESCE` soit également implémentée. Ces deux fonctions retournent 
le premier argument qui n'est pas `NULL`. Bien évidemment, PostgreSQL 
n'implémente que la fonction normalisée `COALESCE`. Un simple remplacement de
l'appel de `NVL` par un appel à `COALESCE` est suffisant.

Ainsi, la requête suivante :

```sql
SELECT NVL(description, description_courte, '(aucune)') FROM articles;
```

se verra portée facilement de cette façon :

```sql
SELECT COALESCE(description, description_courte, '(aucune)') FROM articles;
```

Références :

* [Expressions conditionnelles](https://docs.postgresql.fr/current/functions-conditional.html)

### Utilisation de ROWNUM

Oracle propose une pseudo-colonne `ROWNUM` qui permet de numéroter les lignes 
du résultat d'une requête SQL. La clause `ROWNUM` peut être utilisée soit pour
numéroter les lignes de l'ensemble retourné par la requête. Elle peut aussi être 
utilisée pour limiter l'ensemble retourné par une requête.

#### ROWNUM pour numéroter les lignes

Dans le premier cas, à savoir numéroter les lignes de l'ensemble retourné par la
requête, il faut réécrire la requête pour utiliser la fonction de fenêtrage 
`row_number()`. Bien qu'Oracle préconise d'utiliser la fonction normalisée 
`row_number()`, il est fréquent de trouver `ROWNUM` dans une requête issue
d'une application s'appuyant sur une ancienne version d'Oracle :

```sql
SELECT ROWNUM, * FROM employees;
```

La requête sera réécrite de la façon suivante :

```sql
SELECT row_number() OVER () AS rownum, * FROM employees;
```

Il faut toutefois faire attention à une clause `ORDER BY` dans une requête 
employant `ROWNUM` pour numéroter les lignes retournées par une requête. En 
effet, le tri commandé par `ORDER BY` est réalisé après l'ajout de la 
pseudo-colonne `ROWNUM`. Il faudra vérifier le plan d'exécution de la requête 
sous Oracle et PostgreSQL pour vérifier qu'elles retourneront des résultats 
identiques.

#### ROWNUM pour limiter le résultat

Pour limiter l'ensemble retourné par une requête, il faut supprimer les
prédicats utilisant `ROWNUM` dans la clause et les transformer en couple
`LIMIT`/`OFFSET`.

La requête suivante retourne les 10 premières lignes de la table `employees`
sous Oracle :

```sql
SELECT *
  FROM employees
 WHERE ROWNUM < 11;
```

Elle sera réécrite de la façon suivante lors du portage de la requête pour
PostgreSQL :

```sql
SELECT *
  FROM employees
 LIMIT 10;
```

Avec Oracle, si cette même requête est triée à l'aide de la clause `ORDER BY`,
la pseudo-colonne `ROWNUM` est calculée (`COUNT STOPKEY`) pour chaque ligne afin
de garantir le tri qui est réalisé juste après. Le plan d'exécution ci-dessous
montre l'ordre des nœuds qui est retenu pour satisfaire la requête :

```
---------------------------------------------------------------------------------
| Id  | Operation            | Name      | Rows | Bytes | Cost (%CPU)| Time     |
---------------------------------------------------------------------------------
|   0 | SELECT STATEMENT     |           |   10 |   690 |     3  (34)| 00:00:01 |
|   1 |  SORT ORDER BY       |           |   10 |   690 |     3  (34)| 00:00:01 |
|*  2 |   COUNT STOPKEY      |           |      |       |            |          |
|   3 |    TABLE ACCESS FULL | EMPLOYEES |   10 |   690 |     2   (0)| 00:00:01 |
---------------------------------------------------------------------------------
```

Cette particularité est connue des développeurs et pour palier au comportement
de l'optimiseur Oracle, il est possible de rencontrer cette forme de requête
avec sous-requête, comme suit :

```sql
SELECT ROWNUM, r.*
  FROM (SELECT *
          FROM t1
         ORDER BY col) r
 WHERE ROWNUM BETWEEN 1 AND 10;
```

Au contraire, PostgreSQL va appliquer le tri avant la limitation du résultat.
Lorsque PostgreSQL rencontre une clause `LIMIT ` et un tri avec `ORDER BY `, il
appliquera d'abord le tri avant de limiter le résultat. La requête peut alors
être simplifiée de cette façon :

```sql
SELECT *
  FROM employees
 ORDER BY salary DESC 
 LIMIT 10;
```
```
                               QUERY PLAN                                
-------------------------------------------------------------------
 Limit (cost=81.44..81.46 rows=10 width=8)
   -> Sort (cost=81.44..87.09 rows=2260 width=8)
      Sort Key: salary DESC
      -> Seq Scan on employees (cost=0.00..32.60 rows=2260 width=8)
```

Références :

* [LIMIT et OFFSET](https://docs.postgresql.fr/current/queries-limit.html)
* [Fonctions Window](https://docs.postgresql.fr/current/functions-window.html)
