---
weight: 6
bookFlatSection: false
title: "Reprise des partitions"
---

## Reprise des partitions

PostgreSQL dispose d'un partitionnement déclaratif depuis la version 10, qui ne
cesse de s'améliorer de version majeure en version majeure. Le type du
partitionnement est décisif au moment de la définition du modèle, car il est
très coûteux de le changer au moment de la vie des données. Contrairement à
Oracle, la table partitionnée doit être créée séparément avant que les
partitions puissent être définies.

### Partitionnement par liste

La définition suivante d'une table partitionnée Oracle :

```sql
CREATE TABLE t1 (c1 integer, c2 varchar2(100))
  PARTITION BY LIST (c1) (
    PARTITION t1_a VALUES (1, 2, 3),
    PARTITION t1_b VALUES (4, 5),
    PARTITION t1_default VALUES (DEFAULT)
  );
```

Sera reprise de cette façon avec PostgreSQL :

```sql
CREATE TABLE t1(c1 integer, c2 varchar(100))
  PARTITION BY LIST (c1) ;

CREATE TABLE t1_a PARTITION OF t1 FOR VALUES IN (1, 2, 3);
CREATE TABLE t1_b PARTITION OF t1 FOR VALUES IN (4, 5);
CREATE TABLE t1_default PARTITION of t1 DEFAULT;
```

### Partitionnement par intervalles

Oracle permet la déclaration d'intervalles de valeurs à l'aide de la clause
`LESS THAN` afin de joindre la borne inférieure d'une partition avec la borne
supérieure d'une autre partition. Les valeurs sont donc strictement comprises
entre `-∞` et la plus haute borne supérieure.

```sql
CREATE TABLE t2 (c1 integer, c2 varchar2(100))
  PARTITION BY RANGE (c1) (
    PARTITION t2_a VALUES LESS THAN (0),
    PARTITION t2_b VALUES LESS THAN (100),
    PARTITION t2_c VALUES LESS THAN (MAXVALUE)
  );
```

Les deux bornes de l'intervalle d'une partition doivent être précisées lors de la
déclaration avec PostgreSQL, la borne supérieure étant exclue de l'intervalle.
La table partitionnée sera donc traduite de cette façon :

```sql
CREATE TABLE t2 (c1 integer, c2 varchar(100)) PARTITION BY RANGE (c1);

CREATE TABLE t2_a PARTITION OF t2 FOR VALUES FROM (MINVALUE) TO (0);
CREATE TABLE t2_b PARTITION OF t2 FOR VALUES FROM (0) TO (100);
CREATE TABLE t2_c PARTITION OF t2 FOR VALUES FROM (100) TO (MAXVALUE);
```

### Partitionnement par intervalles automatique

Oracle propose de créer automatiquement les partitions à l'aide de la clause
`INTERVAL` et d'un litéral représentant un intervalle `DAY TO SECOND` ou `YEAR
TO MONTH`. Dans l'exemple ci-dessous, seule la première partition pivot est
définie ; toutes les données insérées provoqueront la création d'une partition
lorsque la valeur de la clé de partitionnement se trouve au-delà de la borne
supérieure de la partition pivot.

```sql
CREATE TABLE t2_auto (c1 number(6), c2 date)
  PARTITION BY RANGE (c2) INTERVAL (NUMTOYMINTERVAL(1, 'MONTH')) (
    PARTITION t2_part VALUES LESS THAN
      (TO_DATE('01-JAN-2020','dd-MON-yyyy'))
  );

INSERT INTO t2_auto VALUES (1, TO_DATE('01-DEC-2000','dd-MON-yyyy'));
INSERT INTO t2_auto VALUES (2, TO_DATE('01-OCT-2022','dd-MON-yyyy'));
INSERT INTO t2_auto VALUES (3, TO_DATE('01-DEC-2022','dd-MON-yyyy'));
```

La vue `USER_TAB_PARTITIONS` présente les partitions, dont deux nouvelles qui
portent un nom généré par le moteur.

| TABLE_NAME   | PARTITION_NAME | HIGH_VALUE |
|--------------|----------------|------------|
| T2_AUTO | T2_PART     | TO_DATE('2020-01-01', 'SYYYY-MM-DD') |
| T2_AUTO | SYS_P472681 | TO_DATE('2021-11-01', 'SYYYY-MM-DD') |
| T2_AUTO | SYS_P472682 | TO_DATE('2023-01-01', 'SYYYY-MM-DD') |

Cette fonctionnalité de création automatique des partitions n'est pas disponible
nativement avec PostgreSQL. Il est cependant possible de se rapprocher du
comportement présenté ci-dessus à l'aide du projet **pg_partman**. Cette extension
repose sur des tables de configuration et d'une routine de maintenance afin de
détecter les tables partitionnées nécessitant de nouvelles partitions.

Contrairement à Oracle, toute nouvelle ligne en dehors d'une des bornes
existantes est insérée dans la partition par défaut. La routine de maintenance se
charge de déplacer les données vers les partitions attribuées de façon
asynchrone.

Prenons l'exemple de la table `t2_auto` qui peut être transformée de la façon
suivante. L'option `premake` permet d'anticiper la création de `n` partitions à
partir du prochain intervalle au moment de l'exécution de la méthode.

```sql
CREATE TABLE t2_auto (c1 integer, c2 date)
  PARTITION BY RANGE (c2);

CREATE SCHEMA partman;
CREATE EXTENSION pg_partman WITH SCHEMA partman;

SELECT partman.create_parent(
  p_parent_table => 'public.t2_auto',
  p_control => 'c2',
  p_type => 'native',
  p_interval => 'monthly',
  p_premake => 1
);
```

La requête suivante montre la distribution de trois lignes au sein des
partitions :

```sql
INSERT INTO t2_auto VALUES (1, '2000-12-01');
INSERT INTO t2_auto VALUES (2, '2022-10-01');
INSERT INTO t2_auto VALUES (3, '2022-12-01');

SELECT tableoid::regclass, * from t2_auto;
--     tableoid     | c1 |     c2     
-- -----------------+----+------------
-- t2_auto_p2022_10 |  2 | 2022-10-01
-- t2_auto_default  |  1 | 2000-12-01
-- t2_auto_default  |  3 | 2022-12-01
```

L'outil propose une série de méthodes de maintenance pour ajouter de nouvelles
partitions et redistribuer les lignes présentes dans la table par défaut. Ces
opérations doivent être planifiées par un orchestrateur externe (_cron_,
_pg_cron_, etc.) ou à l'aide du processus d'arrière-plan fourni avec
l'extension. Cette dernière solution nécessite de positionner la valeur
`pg_partman_bgw` au paramètre `shared_preload_libraries` et de redémarrer
l'instance.

```sql
-- Surveille les données insérées dans les tables parents et la 
-- table par défaut
SELECT * FROM partman.check_default();

-- Déplace les données des tables parents ou de la table par défaut
-- vers les enfants appropriés.
SELECT * FROM partman.partition_data_time(p_parent_table => 'public.t2_auto');

-- Crée automatiquement des tables enfants pour les partitions gérées
-- par pg_partman
CALL partman.run_maintenance_proc();
```

Les données sont à présent dans les bonnes partitions.

```sql
SELECT tableoid::regclass, * from t2_auto;
--     tableoid     | c1 |     c2     
-- -----------------+----+------------
-- t2_auto_p2000_12 |  1 | 2000-12-01
-- t2_auto_p2022_10 |  2 | 2022-10-01
-- t2_auto_p2022_12 |  3 | 2022-12-01
```


### Partitionnement par hâchage

Pour répartir équitablement les données entre un nombre fini de partitions,
Oracle et PostgreSQL proposent le même fonctionnement de partitionnement 
par hâchage.

La table partitionnée suivante avec Oracle :

```sql
CREATE TABLE t3 (c1 integer, c2 varchar2(100))
  PARTITION BY HASH (c1) (
    PARTITION t3_a,
    PARTITION t3_b,
    PARTITION t3_c
  );
```

Sera convertie de la sorte avec PostgreSQL :

```sql
CREATE TABLE t3 (c1 integer, c2 varchar(100)) 
  PARTITION BY HASH (c1);

CREATE TABLE t3_a PARTITION OF t3 FOR VALUES WITH (modulus 3, remainder 0);
CREATE TABLE t3_b PARTITION OF t3 FOR VALUES WITH (modulus 3, remainder 1);
CREATE TABLE t3_c PARTITION OF t3 FOR VALUES WITH (modulus 3, remainder 2);
```

Lorsqu'il est nécessaire d'ajouter une ou plusieurs partitions à une table
partitionnée, notamment pour répartir à nouveau le volume de lignes, Oracle
choisit lui-même une partition selon l'algorithme de hâchage pour diviser le
contenu de la partition en deux afin de redistribuer l'une des moitiés sur la
nouvelle partition.

Avec Oracle, l'ordre de maintenance suivant réalise les opérations de façon
transparente :

```sql
ALTER TABLE t3 ADD PARTITION t3_d;
```

Avec PostgreSQL, il revient à l'administrateur de sélectionner la partition
source en élargissant les options `modulus` et `remainder` pour répartir les
deux (ou plus) sous-ensembles de données, dont chaque moitié sera déplacée dans
les nouvelles partitions. Cette opération nécessite de poser un verrou dans une
transaction le temps de la copie vers les deux nouvelles partitions :

```sql
BEGIN;
-- Remplacement d'une partition par deux sous-ensembles
ALTER TABLE t3 DETACH PARTITION t3_c;
CREATE TABLE t3_c_0 PARTITION OF t3 FOR VALUES WITH (modulus 6, remainder 0);
CREATE TABLE t3_c_3 PARTITION OF t3 FOR VALUES WITH (modulus 6, remainder 3);

-- Déplacement des lignes
INSERT INTO t3 SELECT * FROM t3_c;
DROP TABLE t3_c;
COMMIT;
```

### Partitionnement composite

Les précédentes méthodes peuvent être couplées pour répondre à des besoins de
partitionnement plus précis, sur plusieurs niveaux de clés de partitionnement.
Prenons la table suivante avec Oracle qui présente des partitions par liste et
des sous-partitions par intervalles :

```sql
CREATE TABLE t4 (c1 char(1), c2 date)
  PARTITION BY LIST (c1)
  SUBPARTITION BY RANGE (c2) (
    PARTITION t4_a VALUES ('A') (
      SUBPARTITION t4_a_2020 VALUES LESS THAN 
        (TO_DATE('01-JAN-2021','dd-MON-yyyy')),
      SUBPARTITION t4_a_2021 VALUES LESS THAN 
        (TO_DATE('01-JAN-2022','dd-MON-yyyy')),
      SUBPARTITION t4_a_2022 VALUES LESS THAN 
        (TO_DATE('01-JAN-2023','dd-MON-yyyy'))
    ),
    PARTITION t4_b VALUES ('B') (
      SUBPARTITION t4_b_2020 VALUES LESS THAN 
        (TO_DATE('01-JAN-2021','dd-MON-yyyy')),
      SUBPARTITION t4_b_2021 VALUES LESS THAN 
        (TO_DATE('01-JAN-2022','dd-MON-yyyy')),
      SUBPARTITION t4_b_2022 VALUES LESS THAN 
        (TO_DATE('01-JAN-2023','dd-MON-yyyy'))
    )
  );
```

Avec PostgreSQL, il sera suffisant de créer des tables partitionnées rattachées
à d'autres tables partitionnées. Les partitions définissent le niveau de
profondeur final. L'exemple précédent sera réécrit de cette manière :

```sql
CREATE TABLE t4 (c1 char(1), c2 date)
  PARTITION BY LIST (c1);

CREATE TABLE t4_a PARTITION OF t4 
  FOR VALUES IN ('A') PARTITION BY RANGE (c2);
CREATE TABLE t4_a_2020 PARTITION OF t4_a 
  FOR VALUES FROM (MINVALUE) TO ('2021-01-01');
CREATE TABLE t4_a_2021 PARTITION OF t4_a 
  FOR VALUES FROM ('2021-01-01') TO ('2022-01-01');
CREATE TABLE t4_a_2022 PARTITION OF t4_a 
  FOR VALUES FROM ('2022-01-01') TO ('2023-01-01');

CREATE TABLE t4_b PARTITION OF t4 
  FOR VALUES IN ('B') PARTITION BY RANGE (c2);
CREATE TABLE t4_b_2020 PARTITION OF t4_b 
  FOR VALUES FROM (MINVALUE) TO ('2021-01-01');
CREATE TABLE t4_b_2021 PARTITION OF t4_b 
  FOR VALUES FROM ('2021-01-01') TO ('2022-01-01');
CREATE TABLE t4_b_2022 PARTITION OF t4_b 
  FOR VALUES FROM ('2022-01-01') TO ('2023-01-01');
```

---

Le partitionnement permet de répondre à plusieurs problématiques. L'une d'entre
elle concerne les performances, si le partitionnement est bien conçu et les
requêtes écrites de façon à tirer partie du partitionnement, alors PostgreSQL ne
lira qu'une ou quelques partitions et non pas la totalité des partitions.

Une autre problématique adressée concerne la maintenance. Le partitionnement
permet en effet de simplifier les opérations de suppressions de données. Par
exemple, la suppression des données relatives à une année se résumera à la
simple suppression d'une table si la table est partitionnée sur l'année. Les
opérations de maintenance (`VACUUM`, `ANALYZE`) sont également bien plus
efficace.

Cependant, le partitionnement déclaratif avec PostgreSQL peut souffrir
d'insuffisances fonctionnelles par rapport aux avancées que présente Oracle. Par
exemple, les contraintes de clés étrangères définies avec des actions telles que
`ON DELETE ... CASCADE` peuvent avoir des comportements surprenants sur des
tables partitionnées avant la version 15, où le déplacement d'une ligne entre
deux partitions déclenche la suppression en cascade des lignes de la clé
étrangère.

Références :

* [Partitionnement](https://docs.postgresql.fr/current/ddl-partitioning.html)
* [Cas d'usages du partitionnement natif dans PostgreSQL](https://blog.anayrat.info/2021/09/01/cas-dusages-du-partitionnement-natif-dans-postgresql)
* [Projet pg_partman](https://github.com/pgpartman/pg_partman)
* [Release 15: Enforce foreign key correctly during cross-partition updates](https://github.com/postgres/postgres/commit/ba9a7e392171c83eb3332a757279e7088487f9a2)
