---
weight: 5
bookFlatSection: false
title: "Reprise des index"
---

## Reprise des index

Pour les index, seule la forme `BTREE` correspond. Les autres types d'index
d'Oracle ne sont pas implémentées mais PostgreSQL dispose lui-aussi d'autres
types d'index. Quoiqu'il en soit, la plupart des index utilisés sont des index
de type `BTREE` car c'est la méthode par défaut utilisée lors de la création
d'un index.

### Index sur chaînes de caractère

Lorsqu'un index sert à accélérer les recherches avec l'opérateur `LIKE` sur
une colonne de type chaîne de caractères, il est nécessaire de le créer avec
la classe d'opérateur `varchar_pattern_ops` pour une colonne de type `VARCHAR`, 
ou `text_pattern_ops` pour une colonne de type `text`, ou encore 
`bpchar_pattern_ops` pour une colonne de type `CHAR`. Ces opérateurs sont 
utiles lorsque les paramètres de localisation du serveur sont différents de `C`.

La classe d'opérateur adéquate est à ajouter après le nom de la colonne concernée 
dans l'ordre de création de l'index :

```sql
CREATE INDEX emp2_ename ON emp2 (ename varchar_pattern_ops);
```

Références :

* [Classes et familles d'opérateurs](https://docs.postgresql.fr/current/indexes-opclass.html)

### Index Bitmap

Le cas d'utilisation des index bitmap, sous Oracle, est celui d'une colonne ayant
très peu de valeurs différentes (faible cardinalité). Il crée donc un tableau de 
bits, chaque bit représentant la présence ou non d'une valeur dans un enregistrement 
de la table. Grossièrement, pour chaque enregistrement (`ROWID`), un bit est associé
à la valeur indexée.

Le cas classique, pour le sexe, par exemple, on aura 2 bits, l'un pour homme, 
l'autre pour femme. Et on remplira un tableau de 2 bits, avec une entrée par 
`ROWID` potentiel de la table, ce qui fait donc un tableau à deux dimensions. 
L'avantage de cette méthode, c'est bien sûr que c'est extrêmement compact si la 
colonne a peu de valeurs différentes.

Il y a par contre plusieurs gros défauts :

* tout ajout de colonne déclenche la réécriture de tout l'index. C'est long et 
surtout, ça verrouille l'index pendant ce temps.
* la concurrence est très faible: l'index étant tout petit, il est très probable 
que les sessions voudront mettre à jour les mêmes pages en même temps,
* au-delà de 8 valeurs différentes, les performances se dégradent très vite.

PostgreSQL ne dispose pas de ces index. Il dispose par contre des index GIN, qui 
sont normalement utilisés surtout pour indexer des données non scalaires: tableaux, 
listes de mots, etc… L'article en référence détaille de manière assez détaillée 
le fonctionnement des index GIN.

Le principe de GIN est d'être un index inversé. Pour chaque valeur possible de 
l'attribut, on crée une liste des enregistrements vérifiant le prédicat 
(par exemple `sexe='F'`). Dans le cas d'un tableau, le même enregistrement peut se 
retrouver simultanément dans plusieurs listes (une liste différente par valeurs 
dans un tableau, par exemple).

Si on revient à notre cas H/F, on va avoir deux listes (ce qu'on appelle des 
_posting lists_, voir l'article en référence), l'une avec la liste des 
enregistrements où `sexe='F'`, l'autre avec la liste des enregistrements où `sexe='H'`.
Ces deux listes, depuis PostgreSQL 9.4, sont compressées, ce qui les rend très 
compactes. Pas autant qu'un bitmap, mais très compactes tout de même (et meilleure 
en concurrence d'accès).

La séquence suivante montre la différence de volumétrie entre un index Btree et 
un index GIN pour l'exemple évoqué :

```sql
-- Pour pouvoir indexer des scalaires, et pas uniquement des tableaux, avec gin
CREATE extension btree_gin;
CREATE TABLE t1 (name VARCHAR, sexe CHAR);

-- 100 millions d'enregistrements, le nom est un numéro, et le sexe est 50/50
INSERT INTO t1 
  SELECT i, CASE WHEN i%2 = 0 THEN 'F' ELSE 'M' END
  FROM generate_series(1,100000000) g(i); 

CREATE INDEX idx_sexe_gin ON t1 USING gin (sexe);

SELECT pg_size_pretty(pg_table_size('t1'));
--  pg_size_pretty
-- ----------------
--  4223 MB

SELECT pg_size_pretty(pg_table_size('idx_sexe_gin'));
--  pg_size_pretty 
-- ----------------
--  102 MB

SELECT pg_size_pretty(pg_table_size('idx_sexe_btree'));
--  pg_size_pretty 
-- ----------------
--  2142 MB
```

Les index GIN ne sont donc pas aussi compact que des index bitmaps. Par contre, 
d'un point de vue de l'accès concurrentiel, ils sont supérieurs (organisés de 
façon similaires à des BTrees). C'est donc un compromis un peu différent, mais 
ils permettent de répondre à la même problématique.

De même que les index BTree, les index GIN peuvent être utilisés pour réaliser 
des opérations de bitmap, afin d'utiliser plusieurs index simultanément pour 
restreindre le volume de données à consulter.

Références :

* [GIN – Just A Kind Of Index](http://www.cybertec.at/gin-just-an-index-type/), par Hans-Juergen Schoenig.

### Index inverse

Les index inverses (ou _reverse_) permettent d'accélérer des recherches de type 
`LIKE '%chaine'`, opérations qui ne bénéficient pas ordinairement 
de la présence d'un index. L'index inverse n'existe pas directement dans PostgreSQL 
mais il est possible d'utiliser l'extension `pg_trgm` et un index `KKN-GiST`
pour émuler un index inverse et même le surpasser. En effet, cette extension offre 
également la possibilité d'utiliser un index pour les recherches telles que 
`LIKE '%chaine%'` ou même `LIKE '%chaine%chaine%'`. 
Il faut toutefois garder à l'esprit que la mise à jour d'un index `GiST` est 
plus coûteuse que la mise à jour d'un index `BTREE`.

L'extension `pg_trgm`, bien qu'elle ne soit pas incluse dans le cœur de PostgreSQL,
est distribuée avec PostgreSQL (paquet postgresql-contrib) et est maintenue par 
les développeurs de PostgreSQL.

Ce type d'index n'est pas créé automatiquement par Ora2Pg, il nécessite une 
intervention manuelle.

```sql
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_emp_ename_trgm ON emp USING gist (ename gist_trgm_ops);
--or
CREATE INDEX idx_emp_ename_trgm ON emp USING gin (ename gin_trgm_ops);
```

Le plan d'exécution d'une requête SELECT montre bien l'utilisation de l'index 
`idx_emp_ename_trgm` :

```sql
EXPLAIN SELECT * FROM emp WHERE ename LIKE '%IN%';
--                                        QUERY PLAN                                       
-- ----------------------------------------------------------------------------------------
--  Bitmap Heap Scan on emp  (cost=1442.95..3522.23 rows=32742 width=20)
--    Recheck Cond: ((ename)::text ~~ '%IN%'::text)
--    ->  Bitmap Index Scan on idx_emp_ename_trgm  (cost=0.00..1434.77 rows=32742 width=0)
--          Index Cond: ((ename)::text ~~ '%IN%'::text)
```

Ces index peuvent également être employés pour des recherches insensibles à la
casse, comme `ILIKE`. 

Références :

* [Indexation des K plus proches voisins](http://wiki.postgresql.org/wiki/What%27s_new_in_PostgreSQL_9.1/fr#K-Nearest-Neighbor_Indexing.2FIndexation_des_k_plus_proches_voisins), Quoi de neuf dans PostgreSQL 9.1 ?
* [Extension pg_trgm](https://docs.postgresql.fr/current/pgtrgm.html)

### Création d'index sans blocage

PostgreSQL permet de créer des index sans bloquer les accès concurrents, grâce 
à l'ordre `CREATE INDEX CONCURRENTLY`. Cet ordre présente néanmoins le risque 
de laisser l'index invalide à l'issue de sa création. Cela arrive si l'index 
ne peut pas être construit, s'il s'agit par exemple d'un index UNIQUE, et 
que la contrainte d'unicité n'est pas respectée.

De la même manière, il est possible de recréer un index sans bloquer les accès 
concurrents, à l'aide de l'ordre `REINDEX CONCURRENTLY`. Comme pour la création, 
il peut arriver que l'index soit laissé invalide.

Références :

* [CREATE INDEX](https://docs.postgresql.fr/current/sql-createindex.html)
* [REINDEX](https://docs.postgresql.fr/current/sql-reindex.html)
