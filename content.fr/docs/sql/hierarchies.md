---
weight: 4
bookFlatSection: false
title: "Traitement des hiérarchies"
previouspage: "conditions"
nextpage: "transactions"
---

## Traitement des hiérarchies

Oracle propose la fonction `CONNECT BY` qui permet d'explorer un arbre
hiérarchique. Cette fonction spécifique à Oracle possède des fonctionnalités
avancées comme la détection de cycle et propose des pseudos-colonnes comme le
niveau de la hiérarchie et la construction d'un chemin.

Depuis la version 14 de PostgreSQL, il est possible de porter de nouvelles
fonctionnalités avancées. Pour les versions antérieurs, un travail important de
portage doit être réalisé pour porter les requêtes utilisant cette clause.

### CONNECT BY

Soit la requête SQL suivante qui explore la hiérarchie de la table `emp`. La
colonne `mgr` de cette table désigne le responsable hiérarchique d'un employé.
Si elle vaut NULL, alors la personne est au sommet de la hiérarchie (`START WITH
mgr IS NULL`). Le lien avec l'employé et son responsable hiérarchique est
construit avec la clause `CONNECT BY PRIOR empno = mgr` qui indique que la
valeur de la colonne `mgr` correspond à l'identifiant `empno` du niveau de
hiérarchie précédent.

```sql
SELECT empno, ename, job, mgr
  FROM emp
  START WITH mgr IS NULL
  CONNECT BY PRIOR empno = mgr
  ```

Le portage de cette requête est réalisé à l'aide d'une requête récursive (`WITH
RECURSIVE`). La récursion est initialisée dans une première requête qui récupère
les lignes qui correspondent à la condition de la clause `START WITH` de la
requête précédente : `mgr IS NULL`. La récursion continue ensuite avec la
requête suivante qui réalise une jointure entre la table `emp` et la vue
virtuelle `emp_hierarchy` qui est définie par la clause `WITH RECURSIVE`. La
condition de jointure correspond à la clause `CONNECT BY`. La vue virtuelle
`emp_hierarchy` a pour alias `prior` pour mieux représenter la transposition de
la clause `CONNECT BY`.

La requête récursive pour PostgreSQL serait alors écrite de la façon suivante :

```sql
WITH RECURSIVE emp_hierarchy (empno, ename, job, mgr) AS (
  SELECT empno, ename, job, mgr
    FROM emp
    WHERE mgr IS NULL
    UNION ALL
  SELECT emp.empno, emp.ename, emp.job, emp.mgr
    FROM emp
    JOIN emp_hierarchy prior ON (emp.mgr = prior.empno)
)
SELECT * FROM emp_hierarchy;
```

Il faudra néanmoins faire attention à l'ordre des lignes qui sera différent avec
la requête `WITH RECURSIVE`. En effet, Oracle utilise un algorithme
_depth-first_ dans son implémentation du `CONNECT BY`. Ainsi, il explorera
d'abord chaque branche avant de passer à la suivante. L'implémentation `WITH
RECURSIVE` est de type _breadth-first_ qui explore chaque niveau de hiérarchie
avant de descendre.

Il est possible de retrouver l'ordre de tri d'une requête `CONNECT BY` pour une
version antérieure à la 11g d'Oracle en triant sur une colonne `path`, telle
qu'elle est construite pour émuler la clause `SYS_CONNECT_BY_PATH` :

```sql
WITH RECURSIVE emp_hierarchy (empno, ename, job, mgr, path) AS (
  SELECT empno, ename, job, mgr, ARRAY[ename::TEXT] AS path
    FROM emp
   WHERE mgr IS NULL
   UNION ALL
  SELECT emp.empno, emp.ename, emp.job, emp.mgr, 
         prior.path || emp.ename::TEXT AS path
    FROM emp
    JOIN emp_hierarchy prior ON (emp.mgr = prior.empno)
)
SELECT empno, ename, job FROM emp_hierarchy AS emp
ORDER BY path;
```

À partir de la version 11g, Oracle retourne les résultats dans un ordre
différent.

### Pseudo-colonne LEVEL

La clause `LEVEL` permet d'obtenir le niveau de hiérarchie d'un élément.

```sql
SELECT empno, ename, job, mgr, level
  FROM emp
  START WITH mgr IS NULL
  CONNECT BY PRIOR empno = mgr
```

Le portage de la clause `LEVEL` est facile. La requête d'initialisation de la
récursion initialise la colonne `level` à 1. La requête de récursion effectue
ensuite une incrémentation de cette colonne pour chaque niveau de hiérarchie
exploré :

```sql
WITH RECURSIVE emp_hierarchy (empno, ename, job, mgr, level) AS (
    SELECT empno, ename, job, mgr, 1 AS level
      FROM emp
     WHERE mgr IS NULL
  UNION ALL
    SELECT emp.empno, emp.ename, emp.job, emp.mgr, prior.level + 1
      FROM emp
      JOIN emp_hierarchy prior ON (emp.mgr = prior.empno)
)
SELECT * FROM emp_hierarchy;
```

### Clause SYS_CONNECT_BY_PATH

La clause `SYS_CONNECT_BY_PATH` permet d'obtenir un chemin où chaque élément est
séparé de l'autre par un caractère donné. Par exemple, la requête suivante
indique qui sont les différents responsables d'un employé de cette façon :

```sql
SELECT empno, ename, job, mgr, SYS_CONNECT_BY_PATH(ename, '/') AS path
  FROM emp
  START WITH mgr IS NULL
  CONNECT BY PRIOR empno = mgr;
```

Le portage de la clause `SYS_CONNECT_BY_PATH` est également assez facile. La
requête d'initialisation de la récursion construit l'élément racine : `'/' ||
ename AS path`. La requête de récursion réalise quant à elle une concaténation
entre le `path` récupéré de la précédente itération et l'élément à concaténé :
`prior.path || '/' || emp.ename` :

```sql
WITH RECURSIVE emp_hierarchy (empno, ename, job, mgr, path) AS (
    SELECT empno, ename, job, mgr, '/' || ename AS path
      FROM emp
     WHERE mgr IS NULL
  UNION ALL
    SELECT emp.empno, emp.ename, emp.job, emp.mgr, 
           prior.path || '/' || emp.ename
      FROM emp
      JOIN emp_hierarchy prior ON (emp.mgr = prior.empno)
)
SELECT * FROM emp_hierarchy;
```

Une autre façon de faire est d'utiliser un tableau pour stocker le chemin le
temps de la récursion, puis de construire la représentation textuelle de ces
chemins au moment de la sortie des résultats. À noter la conversion de la valeur
de `ename` en type `text` pour chaque élément ajouté dans le tableau `path`.
Cette variante peut être utile pour l'émulation de la clause `NOCYCLE` comme vu
plus bas :

```sql
WITH RECURSIVE emp_hierarchy (empno, ename, job, mgr, path) AS (
    SELECT empno, ename, job, mgr, ARRAY[ename::TEXT] AS path
      FROM emp
     WHERE mgr IS NULL
  UNION ALL
    SELECT emp.empno, emp.ename, emp.job, emp.mgr,
           prior.path || emp.ename::TEXT AS path
      FROM emp
      JOIN emp_hierarchy prior ON (emp.mgr = prior.empno)
)
SELECT empno, ename, job, array_to_string(path, '/') AS path 
  FROM emp_hierarchy AS emp;
```

### Clause NOCYCLE

La requête Oracle suivante :

```sql
SELECT empno, ename, job, mgr
  FROM emp
  START WITH mgr IS NULL
  CONNECT BY NOCYCLE PRIOR empno = mgr;
```

sera transposée pour PostgreSQL de la façon suivante :

```sql
WITH RECURSIVE emp_hierarchy (empno, ename, job, mgr, path, is_cycle) AS (
    SELECT empno, ename, job, mgr, ARRAY[ename::TEXT] AS path, false AS is_cycle
      FROM emp
     WHERE mgr IS NULL
  UNION ALL
    SELECT emp.empno, emp.ename, emp.job, emp.mgr, 
           prior.path || emp.ename::TEXT AS path, 
           emp.ename = ANY(prior.path) AS is_cycle
      FROM emp
      JOIN emp_hierarchy prior ON (emp.mgr = prior.empno)
     WHERE is_cycle = false
)
SELECT empno, ename, job, mgr
  FROM emp_hierarchy AS emp
 WHERE is_cycle = false;
```

### Clause CONNECT_BY_IS_CYCLE

La clause `CONNECT_BY_IS_CYCLE` retourne 1 si la ligne courante a un enfant qui
est également son ancêtre. Dans le cas contraire, elle retourne 0. Il est
possible de retrouver le fonctionnement de cette clause à partir de la requête
précédente, à l'aide d'une expression conditionnelle et en supprimant la
dernière restriction :

```sql
WITH RECURSIVE emp_hierarchy (empno, ename, job, mgr, level, path, is_cycle) AS (
    SELECT empno, ename, job, mgr, 1 AS level, ARRAY[ename::TEXT] AS path, false AS is_cycle
      FROM emp
     WHERE mgr = 10000
  UNION ALL
    SELECT emp.empno, emp.ename, emp.job, emp.mgr, 
           prior.level + 1 AS level, 
           prior.path || emp.ename::TEXT AS path,
           emp.ename = ANY(prior.path) AS is_cycle 
      FROM emp
      JOIN emp_hierarchy prior ON (emp.mgr = prior.empno)
     WHERE is_cycle = false
)
SELECT *, is_cycle::int AS connect_by_is_cycle
  FROM emp_hierarchy AS emp;
```

### Clause ORDER SIBLINGS BY

L'émulation de la clause `ORDER SIBLINGS BY`, qui effectue un tri, nécessite de
reprendre la requête récursive émulant la clause `SYS_CONNECT_BY_PATH` et
d'appliquer un tri sur la colonne `path` résultante de la requête :

```sql
WITH RECURSIVE emp_hierarchy (empno, ename, job, mgr, path) AS (
  SELECT empno, ename, job, mgr, '/' || ename AS path
    FROM emp
   WHERE mgr IS NULL
   UNION ALL
  SELECT emp.empno, emp.ename, emp.job, emp.mgr, prior.path || '/' || emp.ename
    FROM emp
    JOIN emp_hierarchy prior ON (emp.mgr = prior.empno)
)
SELECT * FROM emp_hierarchy
 ORDER BY path;
```

Cependant, cette émulation ne fonctionne que dans le cadre d'un tri ascendant.
L'application d'un tri descendant ne retourne pas le résultat escompté.

### Clause CONNECT_BY_ROOT

La clause `CONNECT_BY_ROOT` retourne la racine de chaque élément de la
hiérarchie. Dans l'exemple ci-dessous, la dernière colonne retournera le nom de
la personne la plus élevée dans la hiérarchie de l'employé concerné :

```sql
SELECT empno, ename, job, mgr AS direct_mgr, 
       CONNECT_BY_ROOT ename AS mgr
  FROM emp
  START WITH mgr IS NULL
  CONNECT BY mgr = PRIOR empno
  ORDER SIBLINGS BY ename DESC;
```

La requête est transposée de la même manière que pour le cas de
`SYS_CONNECT_BY_PATH`. Le tableau `path` est utilisé pour obtenir l'élément
racine de la hiérarchie.

```sql
WITH RECURSIVE emp_hierarchy (empno, ename, job, mgr, path) AS (
    SELECT empno, ename, job, mgr, ARRAY[ename::TEXT] AS path
      FROM emp
     WHERE mgr IS NULL
  UNION ALL
    SELECT emp.empno, emp.ename, emp.job, emp.mgr, 
           prior.path || emp.ename::TEXT AS path
      FROM emp
      JOIN emp_hierarchy prior ON (emp.mgr = prior.empno)
)
SELECT empno, ename, job, path[1] AS connect_by_root 
  FROM emp_hierarchy AS emp;
```

### Clause CONNECT_BY_ISLEAF

La clause `CONNECT_BY_ISLEAF` ne prend aucun argument et précise si la ligne en
question (_feuille_) n'est plus connectée à une nouvelle ligne descendante à
travers l'arbre de hiérarchie. La valeur `0` est retournée s'il existe une
connexion et `1` s'il s'agit de la dernière ligne de la hiérarchie.

```sql
SELECT empno, ename, job, mgr,
       CONNECT_BY_ROOT ename AS mgr,
       CONNECT_BY_ISLEAF AS isleaf, 
       SYS_CONNECT_BY_PATH(ename, '/') AS path
  FROM emp
  START WITH mgr IS NULL
  CONNECT BY mgr =  PRIOR empno
```

L'émulation de la clause `CONNECT_BY_ISLEAF` nécessite une auto-jointure sur le
résultat de la requête récursive pour déterminer si la ligne est une _feuille_
de l'arbre. Il est nécessaire d'utiliser la colonne de type tableau `path` pour
pouvoir trier le résultat.

```sql
WITH RECURSIVE emp_hierarchy (empno, ename, job, mgr, path) AS (
  SELECT empno, ename, job, mgr, ARRAY[ename::TEXT] AS path
    FROM emp
    WHERE mgr IS NULL
    UNION ALL
  SELECT emp.empno, emp.ename, emp.job, emp.mgr, 
         prior.path || emp.ename::TEXT AS path
    FROM emp
    JOIN emp_hierarchy prior ON (emp.mgr = prior.empno)
)
SELECT emp.empno, emp.ename, emp.job, 
       CASE WHEN leaf.empno IS NULL THEN 1 ELSE 0 END AS isleaf
  FROM emp_hierarchy AS emp
  LEFT JOIN emp_hierarchy AS leaf ON (emp.empno = leaf.mgr)
 ORDER BY emp.path;
```