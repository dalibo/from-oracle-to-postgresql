---
weight: 2
bookFlatSection: false
title: "Jointures"
---

## Jointures

Le SGBD Oracle supporte la syntaxe normalisée d'écriture des jointures seulement 
depuis la version 9i. Auparavant, les jointures étaient exprimées telle que le 
définissait la première version de la norme SQL, avec une notation propriétaire 
pour la gestion des jointures externes. PostgreSQL ne supporte pas cette notation
propriétaire, mais supporte parfaitement la notation portée par la norme SQL.

### Jointure simple

La requête suivante peut être conservée telle qu'elle est écrite :

```sql
SELECT *
  FROM t1, t2
 WHERE t1.col1 = t2.col1
```

Cependant, cette syntaxe ne permet pas d'écrire de jointure externe. Il est donc 
recommandé d'utiliser systématiquement la nouvelle notation, qui est aussi bien 
plus lisible dans le cas où des jointures simples et externes sont mélangées :

```SQL
SELECT *
  FROM t1
  JOIN t2 ON (t1.col1 = t2.col1)
```

### Jointure externe à gauche et à droite

Le SGBD Oracle utilise la notation `(+)` pour décrire le côté où se trouvent 
les valeurs NULL. Pour une jointure à gauche, l'annotation `(+)` serait placée 
du côté droit (et inversement pour une jointure à droite). Cette forme n'est pas 
supportée par PostgreSQL. Il faut donc réécrire les jointures avec la notation 
normalisée : `LEFT OUTER JOIN` ou `LEFT JOIN` pour une jointure à gauche et 
`RIGHT OUTER JOIN` ou `RIGHT JOIN` pour une jointure à droite.

La requête suivante, écrite pour Oracle et qui comporte une jointure à gauche :

```sql
SELECT *
  FROM t1, t2
 WHERE t1.col1 = t2.col3 (+);
```

nécessite d'être réécrite de la manière suivante :

```sql
SELECT *
  FROM t1
  LEFT JOIN t2 ON (t1.col1 = t2.col3);
```

De la même façon, la requête suivante comporte une jointure à droite :

```sql
SELECT *
  FROM t1, t2
 WHERE t1.col1 (+) = t2.col3;
```

et nécessite d'être réécrite de la manière suivante :

```sql
SELECT *
  FROM t1
  RIGHT JOIN t2 ON (t1.col1 = t2.col3);
```

### Jointure externe complète

Dans les versions précédant la version 9i d'Oracle, une jointure externe complète 
(`FULL OUTER JOIN`) devait être exprimée à l'aide d'un `UNION` entre une 
jointure à gauche et une jointure à droite. L'exemple suivant implémente une 
jointure externe complète :

```sql
SELECT *
  FROM t1, t2
 WHERE t1.col1 = t2.col3 (+)
UNION ALL
SELECT *
  FROM t1, t2
 WHERE t1.col1 (+) = t2.col3
   AND t1.col IS NULL
```

Cette requête doit être réécrite et sera par ailleurs simplifiée de la façon 
suivante :

```sql
SELECT *
  FROM t1
  FULL OUTER JOIN t2 ON (t1.col1 = t2.col3);
```

### Mélange de syntaxes de jointure

Lors d'un portage d'Oracle vers Postgres, il est tentant de ne migrer que les 
jointures externes, et de garder les autres à l'ancienne syntaxe, pour limiter
l'effort de réécriture.

C'est à déconseiller : sur des requêtes complexes, impliquant de nombreuses
tables, l'optimiseur risque d'avoir du mal à calculer un plan d'exécution optimal, 
le contenu de la clause `WHERE` n'étant pas forcément réinjecté en tant que jointure.

Ceci est contrôlé par le paramètre `from_collapse_limit` de l'optimiseur. 
Il indique la profondeur maximale à laquelle l'optimiseur va essayer de réordonner 
les jointures du `WHERE`. Il est par défaut à 8, ce qui est la plupart du temps 
une valeur raisonnable. L'augmenter a évidemment un gros impact sur le temps de 
planification des requêtes.

Voici un exemple très simplifié, dans lequel nous allons forcer 
`from_collapse_limit` à 2 pour que le problème se produise sur une requête 
simple :

```sql
CREATE TABLE t1(a INT, b INT);
CREATE TABLE t2(b INT, c INT);
CREATE TABLE t3(c INT, d INT);
CREATE TABLE t4(d INT, e INT);
INSERT INTO t1 SELECT generate_series(1,1000000), generate_series(1,1000000);
INSERT INTO t2 SELECT generate_series(1,1000000), generate_series(1,1000000);
INSERT INTO t3 SELECT generate_series(1,1000000), generate_series(1,1000000);
INSERT INTO t4 SELECT generate_series(1,1000000), generate_series(1,1000000);
ALTER TABLE t4 add PRIMARY KEY (a);
ALTER TABLE t1 add PRIMARY KEY (a);
ALTER TABLE t2 add PRIMARY KEY (b);
ALTER TABLE t3 add PRIMARY KEY (c);
ALTER TABLE t4 add PRIMARY KEY (d);

-- Les statistiques sont maintenant à jour
analyze; 

-- 4 tables sont impliquées, mais on n'autorise qu'une profondeur de 2 à l'optimiseur
set from_collapse_limit TO 2; 
```

```sql
-- Jointure « moderne »
EXPLAIN ANALYZE 
SELECT * 
  FROM t1 
  JOIN t2 USING (b) 
  JOIN t3 USING (c) 
  LEFT JOIN t4 USING (d) 
 WHERE t1.a between 1 AND 100;
--                          QUERY PLAN                                      
-- -------------------------------------------------------------------------
--  Nested Loop Left Join
--  (cost=1.70..1271.91 rows=101 width=20)
--  (actual time=0.113..4.607 rows=100 loops=1)
--    ->  Nested Loop
--        (cost=1.27..1064.28 rows=101 width=16)
--        (actual time=0.097..3.129 rows=100 loops=1)
--          ->  Nested Loop
--              (cost=0.85..856.40 rows=101 width=12)
--              (actual time=0.081..1.669 rows=100 loops=1)
--                ->  Index Scan using t1_pkey on t1
--                    (cost=0.42..10.45 rows=101 width=8)
--                    (actual time=0.057..0.163 rows=100 loops=1)
--                      Index Cond: ((a >= 1) AND (a <= 100))
--                ->  Index Scan using t2_pkey on t2
--                    (cost=0.42..8.37 rows=1 width=8)
--                    (actual time=0.011..0.012 rows=1 loops=100)
--                      Index Cond: (b = t1.b)
--          ->  Index Scan using t3_pkey on t3
--              (cost=0.42..2.05 rows=1 width=8)
--              (actual time=0.011..0.012 rows=1 loops=100)
--                Index Cond: (c = t2.c)
--    ->  Index Scan using t4_pkey on t4
--        (cost=0.42..2.05 rows=1 width=8)
--        (actual time=0.011..0.012 rows=1 loops=100)
--          Index Cond: (t3.d = d)
--  Total runtime: 4.815 ms
```

```sql
-- Mélange de jointures « modernes » pour les jointures externes et de 
-- jointures SQL89 pour les jointures internes
EXPLAIN ANALYZE 
SELECT * 
  FROM t1,t2,t3 
  LEFT JOIN t4 USING (d) 
 WHERE t1.b=t2.b AND t2.c=t3.c 
   AND t1.a BETWEEN 1 AND 100;
--                          QUERY PLAN                                      
-- -------------------------------------------------------------------------
--  Hash Join
--  (cost=31689.66..79086.67 rows=101 width=28)
--  (actual time=711.708..2369.201 rows=100 loops=1)
--    Hash Cond: (t3.c = t2.c)
--    ->  Hash Left Join
--        (cost=30832.00..74478.00 rows=1000000 width=12)
--        (actual time=711.170..2217.581 rows=1000000 loops=1)
--          Hash Cond: (t3.d = t4.d)
--          ->  Seq Scan on t3
--              (cost=0.00..14425.00 rows=1000000 width=8)
--              (actual time=0.007..266.867 rows=1000000 loops=1)
--          ->  Hash
--              (cost=14425.00..14425.00 rows=1000000 width=8)
--              (actual time=710.802..710.802 rows=1000000 loops=1)
--                Buckets: 131072  Batches: 2  Memory Usage: 19548kB
--                ->  Seq Scan on t4
--                    (cost=0.00..14425.00 rows=1000000 width=8)
--                    (actual time=0.010..297.606 rows=1000000 loops=1)
--    ->  Hash
--        (cost=856.40..856.40 rows=101 width=16)
--        (actual time=0.511..0.511 rows=100 loops=1)
--          Buckets: 1024  Batches: 1  Memory Usage: 5kB
--          ->  Nested Loop
--              (cost=0.85..856.40 rows=101 width=16)
--              (actual time=0.025..0.459 rows=100 loops=1)
--                ->  Index Scan using t1_pkey on t1
--                    (cost=0.42..10.45 rows=101 width=8)
--                    (actual time=0.017..0.046 rows=100 loops=1)
--                      Index Cond: ((a >= 1) AND (a <= 100))
--                ->  Index Scan using t2_pkey on t2
--                    (cost=0.42..8.37 rows=1 width=8)
--                    (actual time=0.003..0.003 rows=1 loops=100)
--                      Index Cond: (b = t1.b)
--  Total runtime: 2370.090 ms
```

Avec `from_collapse_limit` à 8, le problème ne se produit évidemment pas sur 
cette requête. Toutefois, il est plus sûr de réaliser la correction systématiquement 
sur toutes les requêtes que de compter le nombre de tables impliquées. Le problème 
est aussi bien plus dur à diagnostiquer sur une requête complexe faisant usage 
de sous-requêtes, par exemple.

### Produit cartésien

Un produit cartésien peut être exprimé de la façon suivante dans Oracle et
PostgreSQL :

```sql
SELECT *
  FROM t1, t2;
```

Néanmoins, la notation normalisée est moins ambigüe et montre clairement 
l'intention de faire un produit cartésien :

```sql
SELECT *
  FROM t1
  CROSS JOIN t2;
```

Références :

* [Expressions de tables](https://docs.postgresql.fr/current/queries-table-expressions.html)
