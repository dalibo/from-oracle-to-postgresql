---
weight: 5
bookFlatSection: false
title: "Gestion des transactions"
previouspage: "hierarchies"
nextpage: "/plsql"
---

## Gestion des transactions

La gestion des transactions et des verrous est assez similaire entre Oracle et 
PostgreSQL. Il faut noter deux différences majeures entre les deux SGBD. Tout 
d'abord, Oracle ouvre implicitement une transaction lorsque l'on commence à 
travailler, tandis que PostgreSQL travaille en `autocommit` par défaut. Il faut 
alors ouvrir explicitement une transaction avec l'ordre `BEGIN`.

L'autre différence majeure concerne l'implémentation du moteur MVCC de PostgreSQL 
qui permet d'avoir un coût nul pour les `ROLLBACK` dans PostgreSQL. En 
contrepartie, les blocs modifiés par une transaction qui effectue un `ROLLBACK` 
seront physiquement présents car le moteur de PostgreSQL aura malgré tout créé 
de nouvelles versions de ces lignes, bien qu'elles aient été annulées. Cet espace 
devra être récupéré ensuite par `VACUUM`.

L'ordre `BEGIN` a plusieurs déclinaisons synonymes :

  * `BEGIN`,
  * `BEGIN WORK`,
  * `BEGIN TRANSACTION`,
  * `START TRANSACTION`.

Références :

* [BEGIN](https://docs.postgresql.fr/current/sql-begin.html)
* [START TRANSACTION](https://docs.postgresql.fr/current/sql-start-transaction.html)

### Niveau d'isolation

Il est possible d'indiquer le niveau d'isolation d'une transaction en l'indiquant
dans l'ordre d'ouverture d'une transaction :

```
BEGIN [ WORK | TRANSACTION ] [ mode_transaction [, ...] ]
```

où `mode_transaction` est :

```
ISOLATION LEVEL 
  {SERIALIZABLE | REPEATABLE READ | READ COMMITTED | READ UNCOMMITTED }
READ WRITE | READ ONLY
[ NOT ] DEFERRABLE
```

`READ UNCOMMITTED` est un synonyme de `READ COMMITTED` sous PostgreSQL, tout 
comme sous Oracle : les moteurs étant MVCC, le mode `READ UNCOMMITTED` n'a pas 
d'intérêt (les écrivains ne bloquent pas les lecteurs, les lecteurs ne bloquent 
pas les écrivains).

Par ailleurs, Oracle et PostgreSQL implémentent un niveau d'isolation 
`SERIALIZABLE`. PostgreSQL implémente le niveau d'isolation `SERIALIZABLE` 
avec des verrous optimistes afin de garantir un meilleur débit transactionnel. 
La plupart des SGBD implémentent ce niveau d'isolation par le biais de verrous 
pessimistes, grevant ainsi les performances.

Oracle permet de positionner le niveau d'isolation des transactions pour une 
session donnée, c'est-à-dire que toutes les transactions réalisées dans la même 
session. 

L'ordre SQL suivant permet de positionner le niveau d'isolation au niveau de la 
session pour Oracle :

```sql
ALTER SESSION SET ISOLATION LEVEL ...;
```

La même chose est possible avec PostgreSQL au niveau d'un bloc transactionnel :

```sql
BEGIN TRANSACTION ISOLATION LEVEL ...;
...
COMMIT;
```

La façon pour PostgreSQL de déterminer les violations de sérialisations est plus
stricte que celle d'Oracle, mais les deux modes sont très similaires. Par
exemple, l'article ["On Transaction Isolation Levels"][asktom] reprend une
succession d'instructions entre deux sessions, valides avec Oracle (Tableau 7),
qui provoque une erreur dans PostgreSQL.

[asktom]: https://asktom.oracle.com/Misc/oramag/on-transaction-isolation-levels.html

```sql
create table a ( x int );
create table b ( x int );
```

| Time | Session #1            | Session #2
|------|-----------------------|---------------------------
| t1   | `ALTER SESSION SET isolation_level=serializable;` | |
| t2   | | `ALTER SESSION SET isolation_level=serializable;` |
| t3   | `INSERT INTO a SELECT count(*) FROM b;` | |
| t4   | | `INSERT INTO b SELECT count(*) FROM a;` |
| t5   | `COMMIT;` | |
| t6   | | `COMMIT;` |

PostgreSQL n'autorise pas la session #2 à valider ces modifications à cause
d'une violation de son instantané (_snapshot_). Un message d'erreur est ainsi
levé en suggérant de recommencer la transaction pour aboutir correctement.

```
ERROR: could not serialize access due to read/write dependencies among transactions
DETAIL: Reason code: Canceled on identification as a pivot, during commit attempt.
HINT: The transaction might succeed if retried.
```

Références :

* [Isolation des transactions](https://docs.postgresql.fr/current/transaction-iso.html)
* [SET TRANSACTION](https://docs.postgresql.fr/current/sql-set-transaction.html)


### Vérification des contraintes

Les contraintes d'intégrité sont vérifiées à chaque instruction d'écriture,
qu'elle soit exécutée ou non dans une transaction. Pour exiger un respect de ces
contraintes à l'issue de plusieurs transformations dans une transaction, il est
possible de différer ces vérifications  au moment du `COMMIT`.

Oracle et PostgreSQL proposent la même syntaxe SQL pour définir si une
contrainte est différée de façon systématique :

```sql
ALTER TABLE ... CONSTRAINT ...
  [NOT] DEFERRABLE INITIALLY 
    (IMMEDIATE | DEFERRED)
```

Il est également possible de désactiver le contrôle au sein d'une transaction en
cours à l'aide de l'instruction suivante, commune au deux systèmes :

```sql
SET CONSTRAINT (cons_name | ALL) DEFERRED;
```

Enfin, il peut être nécessaire de définir au niveau d'une session que toutes les
prochaines transactions soient différées par défaut. Dans ce cas, les commandes
entre Oracle et PostgreSQL présentent des différences.

Avec Oracle, la syntaxe :

```sql
ALTER SESSION SET CONSTRAINTS = DEFERRED;
ALTER SESSION SET CONSTRAINTS = IMMEDIATE;
```

Devra être remplacée par :

```sql
SET default_transaction_deferrable = ON;
SET default_transaction_deferrable = OFF;
```

* [SET CONSTRAINTS](https://docs.postgresql.fr/current/sql-set-constraints.html)

### SAVEPOINT

Les `SAVEPOINT` fonctionnent sans régression par rapport au SGBD Oracle. Les 
verrous acquis avant la mise en place d'un `SAVEPOINT` ne sont pas relâchés 
si un `SAVEPOINT` est relâché par un `RELEASE SAVEPOINT` ou un
`ROLLBACK TO SAVEPOINT`

La documentation de PostgreSQL met néanmoins en garde contre la modification de 
lignes après le positionnement d'un `SAVEPOINT` alors que ces lignes ont été 
verrouillées par un `SELECT ... FOR UPDATE` avant le positionnement du 
`SAVEPOINT`. En effet, le verrou acquis par le `SELECT ... FOR UPDATE` peut 
être relâché au moment du `ROLLBACK TO SAVEPOINT`. La séquence suivante d'ordres 
SQL est donc à éviter :

```sql
BEGIN;
  SELECT * FROM ma_table WHERE cle = 1 FOR UPDATE;
  SAVEPOINT s;
  UPDATE ma_table SET ... WHERE cle = 1;
ROLLBACK TO SAVEPOINT s;
```

Références :

* [SAVEPOINT](https://docs.postgresql.fr/current/sql-savepoint.html)
* [RELEASE SAVEPOINT](https://docs.postgresql.fr/current/sql-release-savepoint.html)
* [ROLLBACK TO SAVEPOINT](https://docs.postgresql.fr/current/sql-rollback-to.html)

### Gestion des verrous

Bien que PostgreSQL et Oracle partagent de nombreuses similitudes au niveau du 
verrouillage, il faut prendre en compte certaines différences subtiles.

Références :

* [Verrouillage explicite](https://docs.postgresql.fr/current/explicit-locking.html)

#### Verrouillage implicite

Les ordres DML acquièrent des verrous implicites. La différence notable entre 
Oracle et PostgreSQL concerne l'ordre `SELECT` : Oracle n'acquiert aucun verrou, 
tandis que PostgreSQL pose un verrou de type `ACCESS SHARE`. De ce fait, Oracle 
ne protège en aucun cas les lecteurs de modifications telles que la suppression 
d'une table. Une lecture peut être interrompue suite à un `DROP TABLE` concurrent. 
L'acquisition par PostgreSQL d'un verrou `ACCESS SHARE` pour la lecture protège 
de ce genre de problèmes.

Les ordres `INSERT`, `UPDATE` et `DELETE` verrouillent les lignes modifiées.

Références :

* [Dropping a table during SELECT](http://uhesse.com/2009/10/27/dropping-a-table-during-select/), blog de Uwe Hesse

#### Verrouillage explicite

**SELECT FOR UPDATE**

Les ordres `SELECT FOR UPDATE` peuvent nécessiter des adaptations. La syntaxe
Oracle est en effet un peu plus riche que celle de PostgreSQL pour ce qui
concerne cet ordre SQL.

* Oracle propose une syntaxe `WAIT` et `NOWAIT`. PostgreSQL ne propose que la
  clause `NOWAIT`. La clause `WAIT` est implicite si `NOWAIT` n'est pas
  spécifié, il faudra donc la supprimer. La requête `SELECT ... FOR UPDATE
  WAIT;` devient `SELECT ... FOR UPDATE;`.

  * En l'état, la clause `OF` Oracle est incompatible avec le clause `OF` de
  PostgreSQL. Cette clause permet d'indiquer la table verrouillée pour une mise
  à jour ultérieure. Seulement, la clause `OF` d'Oracle désigne une colonne
  d'une table, tandis que la clause `OF` de PostgreSQL désigne une table.

Références :

* [SELECT FOR UPDATE](https://docs.postgresql.fr/current/sql-select.html#SQL-FOR-UPDATE-SHARE)

**LOCK TABLE**

La syntaxe de l'ordre `LOCK TABLE` d'Oracle est compatible avec celle de PostgreSQL 
pour les cas généraux. L'ensemble des modes de verrouillage proposés par Oracle 
existent tous dans PostgreSQL. On peut noter que PostgreSQL propose plus de type
de verrous.

Tout comme pour l'ordre `SELECT FOR UPDATE`, Oracle propose une syntaxe `WAIT` 
et `NOWAIT`. PostgreSQL ne propose aussi que la clause`NOWAIT`. La clause 
`WAIT` est implicite si `NOWAIT` n'est pas spécifié, il faudra donc la 
supprimer. La requête `LOCK TABLE ... WAIT;` devient `LOCK TABLE ...;`.

Les clauses `PARTITION` et `SUBPARTITION` ne peuvent cependant pas être 
reprises. Dans le cas de la mise en œuvre du partitionnement dans PostgreSQL, 
il faut désigner la table correspondant à la partition ciblée par l'acquisition 
d'un verrou.

Références :

* [LOCK TABLE](https://docs.postgresql.fr/current/sql-lock.html)
