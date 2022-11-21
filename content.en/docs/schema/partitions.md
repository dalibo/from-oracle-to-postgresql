---
weight: 6
bookFlatSection: false
title: "Partitioning"
---

## Partitioning

PostgreSQL accepts declarative partitioning since version 10 and things keep
getting better from major release to another. A table is devided into partitions
as declared by its partitioned key, which is decisive when defining the model
and could be costly to change during the life of the data. Unlike Oracle, the
partitioned table must be create separately before the partitions can be
defined. 

### List Partitioning

The following partitioned table for Oracle:

```sql
CREATE TABLE t1 (c1 integer, c2 varchar2(100))
  PARTITION BY LIST (c1) (
    PARTITION t1_a VALUES (1, 2, 3),
    PARTITION t1_b VALUES (4, 5),
    PARTITION t1_default VALUES (DEFAULT)
  );
```

will be converted to this in PostgreSQL:

```sql
CREATE TABLE t1(c1 integer, c2 varchar(100))
  PARTITION BY LIST (c1) ;

CREATE TABLE t1_a PARTITION OF t1 FOR VALUES IN (1, 2, 3);
CREATE TABLE t1_b PARTITION OF t1 FOR VALUES IN (4, 5);
CREATE TABLE t1_default PARTITION of t1 DEFAULT;
```

### Range Partitioning

Oracle allows range declaration using `LESS THAN` expression in order to join
the lower bound of one partition with the upper bound of another partition. Data
values are strictly between `-∞` and last defined upper bound.

```sql
CREATE TABLE t2 (c1 integer, c2 varchar2(100))
  PARTITION BY RANGE (c1) (
    PARTITION t2_a VALUES LESS THAN (0),
    PARTITION t2_b VALUES LESS THAN (100),
    PARTITION t2_c VALUES LESS THAN (MAXVALUE)
  );
```

Each bounds of a partition must be specified when declaring with PostgreSQL, the
upper bound is excluded from partition range. The above partitioned table will
be converted this way:

```sql
CREATE TABLE t2 (c1 integer, c2 varchar(100)) PARTITION BY RANGE (c1);

CREATE TABLE t2_a PARTITION OF t2 FOR VALUES FROM (MINVALUE) TO (0);
CREATE TABLE t2_b PARTITION OF t2 FOR VALUES FROM (0) TO (100);
CREATE TABLE t2_c PARTITION OF t2 FOR VALUES FROM (100) TO (MAXVALUE);
```

### Automatic Range Partitioning

Oracle supports automatic partition addition using `INTERVAL` clause and an
interval expression like `DAY TO SECOND` or `YEAR TO MONTH`. In example below,
only the first partition is needed as a transition point; all data inserted will
cause a partition to be created when value is greater that the upper bound of
the transition point.

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

The `USER_TAB_PARTITIONS` view describes table definition, including new
partitions with a generated name.

| TABLE_NAME   | PARTITION_NAME | HIGH_VALUE |
|--------------|----------------|------------|
| T2_AUTO | T2_PART     | TO_DATE('2020-01-01', 'SYYYY-MM-DD') |
| T2_AUTO | SYS_P472681 | TO_DATE('2021-11-01', 'SYYYY-MM-DD') |
| T2_AUTO | SYS_P472682 | TO_DATE('2023-01-01', 'SYYYY-MM-DD') |

This automatique partition management feature is not available natively with
PostgreSQL. However, it is possible to mimic the behaviour presented above using
the **pg_partman** project. This extension relies on configuration tables and a
maintenance routine to detect partitioned tables requiring new partitions.

Unlike Oracle, a new row outside one of existing bound is inserted into default
partition. The maintenance routine takes care of moving rows asynchronously
according to their values in a new partition.

Here an example with the previous table `t2_auto`, which can be managed by
`pg_partman`. The `premake` option allows to prepare `n` partitions from the
interval next to the time of the function call.

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

The following query show the distribution of three rows in existing partitions:

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

`pg_partman` offers a series of maintenance functions to add new partitions and
spread rows from the default partitions to more relevant partitions. These
operations must be scheduled by an external orchestrator (_cron_, pg_cron_,
etc.) or using the background process provided by the extension. This last
solution requires setting the `pg_parman_bgw` value to the
`shared_preload_libraries` parameter and restarting the instance.

```sql
-- Monitor for data getting inserted into parent/default tables
SELECT * FROM partman.check_default();

-- Move data from these parent/default tables into the proper children
SELECT * FROM partman.partition_data_time(p_parent_table => 'public.t2_auto');

-- Automatically create child tables for partition sets configured to use it.
CALL partman.run_maintenance_proc();
```

The data is now in the correct partitions.

```sql
SELECT tableoid::regclass, * from t2_auto;
--     tableoid     | c1 |     c2     
-- -----------------+----+------------
-- t2_auto_p2000_12 |  1 | 2000-12-01
-- t2_auto_p2022_10 |  2 | 2022-10-01
-- t2_auto_p2022_12 |  3 | 2022-12-01
```

### Hash Partitioning

To evenly distribute the data between a finite number of partitions, Oracle and
PostgreSQL offer the same hash partitioning operation.

The following partitioned table with Oracle:

```sql
CREATE TABLE t3 (c1 integer, c2 varchar2(100))
  PARTITION BY HASH (c1) (
    PARTITION t3_a,
    PARTITION t3_b,
    PARTITION t3_c
  );
```

Will be converted with PostgreSQL like this:

```sql
CREATE TABLE t3 (c1 integer, c2 varchar(100)) 
  PARTITION BY HASH (c1);

CREATE TABLE t3_a PARTITION OF t3 FOR VALUES WITH (modulus 3, remainder 0);
CREATE TABLE t3_b PARTITION OF t3 FOR VALUES WITH (modulus 3, remainder 1);
CREATE TABLE t3_c PARTITION OF t3 FOR VALUES WITH (modulus 3, remainder 2);
```

When a new partition need to be added, especially to redistribute rows in a
larger number of partitions, Oracle chooses a partition according to its hashing
algorithm to divide the content in two parts and redistributes one of the halves
into the new partition.

With Oracle, the following instruction performs operations transparently:

```sql
ALTER TABLE t3 ADD PARTITION t3_d;
```

With PostgreSQL, it is up to the administrator to select the source partition by
expanding the `modulus` and `remainder` options to split the two (or more)
subsets of data, each half will be moved into a new partition. This operation
requires getting a lock through a transaction while copying data to the two new
partitions:

```sql
BEGIN;
-- Replacement of a partition by two subsets
ALTER TABLE t3 DETACH PARTITION t3_c;
CREATE TABLE t3_c_0 PARTITION OF t3 FOR VALUES WITH (modulus 6, remainder 0);
CREATE TABLE t3_c_3 PARTITION OF t3 FOR VALUES WITH (modulus 6, remainder 3);

-- Move lines
INSERT INTO t3 SELECT * FROM t3_c;
DROP TABLE t3_c;
COMMIT;
```

### Composite Partitioning

All previous methods can be mixed to meet more precise partitioning needs, on
several levels of partitioned keys. Consider the following table with Oracle
has a high level of list partitioning and a low level of range partitioning,
also known as as composition partitioning or subpartitioning:

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

With PostgreSQL, it is possible to attach partitioned tables to another
partition table. Partitions define the final depth level. The previous example
will be rewritten like this:

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

Partitioning is usually used for several purposes. The first one is performance.
If the partitioning is adequate and the queries correctly written in order to
take advantage of partitioning, then PostgreSQL will limit reads to only the
tables holding the candidate data, and not all partitions. 

Another use case partitioning can fulfill is maintenance. Partitioning can ease
management of historical data. For example, if partitioning per year, when one
wants to get rid of data from a certain year, one should simply drop the
partition containing obsolete data. Furthermore, maintenance operations
(`VACUUM`, `ANALYZE`) will be easier.

However, declarative partitioning in PostgreSQL may have some shortcomings
compared to Oracle functionnalities. For example, foreign key constraints
defined with `ON DELETE ... CASCADE` can have surprising behaviors on
partitioned tables before version 15, when moving a row between two partitions
triggers the unwanted deletion of foreign key related rows.

References :

* [Table Partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html)
* [Partitioning use cases with PostgreSQL](https://blog.anayrat.info/en/2021/09/01/partitioning-use-cases-with-postgresql/)
* [Projet pg_partman](https://github.com/pgpartman/pg_partman)
* [Release 15: Enforce foreign key correctly during cross-partition updates](https://github.com/postgres/postgres/commit/ba9a7e392171c83eb3332a757279e7088487f9a2)
