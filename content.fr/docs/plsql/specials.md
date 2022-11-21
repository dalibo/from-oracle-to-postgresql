---
weight: 6
bookFlatSection: false
title: "Spécificités PL/SQL"
slug: "specificites-pl-sql"
---

## Spécificités PL/SQL

### Transactions autonomes

On retrouve souvent des `PRAGMA` associés aux transactions autonomes en PL/SQL.
Cette dernière notion n'existe pas dans PostgreSQL.

Il est possible d'émuler les transactions autonomes par le biais d'un _dblink_, 
mais c'est une solution particulièrement contre-performante et consommatrice de 
ressources. En effet, l'utilisation d'un _dblink_ va entraîner une nouvelle 
connexion à la base de données. Le coût d'une nouvelle connexion n'est pas neutre
(vérification des droits, fork d'un nouveau processus, etc.). Cette nouvelle 
connexion nécessitera de bien dimensionner le paramètre `max_connections` et 
consommera de la mémoire et du temps CPU supplémentaire.

Le code PL/SQL Oracle suivant reprend la déclaration d'une procédure autonome :

```sql
CREATE PROCEDURE LOG_ACTION (username VARCHAR2, msg VARCHAR2)
IS
  PRAGMA AUTONOMOUS_TRANSACTION;
BEGIN
  INSERT INTO table_tracking VALUES (username, msg);
  COMMIT;
END log_action;
```

Une réécriture possible avec un dblink implique de renseigner les informations
de connexion à l'instance locale PostgreSQL :

```sql
CREATE EXTENSION dblink;  
 
CREATE OR REPLACE FUNCTION log_action(username TEXT, msg TEXT)
  RETURNS void AS $$
BEGIN  
    perform dblink_connect('pragma', format(
      'dbname=%s user=test password=test', current_database()
    ));  
    perform dblink_exec('pragma', format(
      'insert into table_tracking values (%s, %s);', username, msg
    ));
    perform dblink_exec('pragma','commit;');  
    perform dblink_disconnect('pragma');
END;  
$$ LANGUAGE plpgsql;
```

Une autre alternative, apparue en version 9.5 avec les _background workers_,
consiste à s'appuyer sur l'extension [`pg_background`][pg_background]
pour déporter l'exécution d'une procédure dans une nouvelle transaction à l'aide
d'une procédure _wrapper_ et la méthode `pg_background_launch` de l'extension.

[pg_background]: https://github.com/vibhorkum/pg_background

Une autre réécriture du code précédent ressemblerait à ceci avec
`pg_background`, avec la création d'une procédure appelante :

```sql
-- Créer une fonction que nous déclencherons « à distance »
CREATE OR REPLACE FUNCTION log_action_atx(username TEXT, msg TEXT)
RETURNS void AS $$
BEGIN
  INSERT INTO table_tracking VALUES (username, msg);
END;
$$ LANGUAGE plpgsql;

-- Créer la fonction principale, chargée d'appeler la fonction distante
CREATE OR REPLACE FUNCTION log_action(username TEXT, msg TEXT)
RETURNS void AS $$
DECLARE
  v_query text;
BEGIN
  v_query := format(
    'SELECT true FROM log_action_atx (%s, %s)', 
    quote_nullable(username), quote_nullable(msg)
  );

  PERFORM pg_background_result(
    pg_background_launch(v_query)
  );
END;
$body$
LANGUAGE plpgsql SECURITY DEFINER;
```

La méthode `log_action` se comporte exactement comme si elle exécutait une
instruction dans une transaction autonome.

Références :

* [Support des transactions autonomes dans PostgreSQL](https://blog.dalibo.com/2016/08/19/Support_des_transactions_autonomes_dans_PostgreSQL.html)

### Les collections VARRAY

Les collections `VARRAY` des _packages_ Oracle sont reprises sous la forme d'un 
type tableau, de `n` éléments. Leur définition est reprise par Ora2Pg, mais elles
nécessitent généralement une réécriture du code l'utilisant.

Lorsque la `VARRAY` est un simple tableau d'un type donné, la reprise nécessite 
moins d'intervention que lorsque la `VARRAY` est du type `%ROWTYPE`. Dans ce 
cas, la reprise est bien plus difficile et nécessite une réécriture forte.

Le code suivant :

```sql
DECLARE
   TYPE Calendar IS VARRAY(366) OF DATE;
```

sera transposé de la façon suivante :

```sql
CREATE TYPE calendar AS (date[366]);
```

En revanche, le code suivant sera transposé par Ora2Pg mais inutilisable sans 
modifications :

```sql
TYPE t_tab_emp IS VARRAY (1000) OF emp%ROWTYPE;
...
tab_emp t_tab_emp; -- déclaration de la variable
```

### Les tableaux associatifs et tables imbriquées

Les collections `TABLE OF` sont utilisées généralement pour déclarer des 
fonctions qui retournent un ensemble de données. De ce fait, les types 
`TABLE OF` peuvent être supprimés et remplacés par un `RETURNS TABLE`, voire 
un `RETURNS SETOF type_de_donnees` pour les types de données simples. Se référer 
à la section _Attribut PIPELINED et instruction PIPE ROW_ de la partie « Portage 
des procédures et fonctions » pour un exemple où la collection `TABLE OF` est 
inutile.

Quoi qu'il en soit, Ora2Pg traduit ce type de données par un tableau du type de 
données associé, nécessitant probablement une révision du code porté.

Ainsi, la déclaration suivante :

```sql
CREATE TYPE information IS TABLE OF VARCHAR2(255);
```

sera transposée en tableau de type VARCHAR(255) par Ora2Pg :

```sql
CREATE TYPE information AS VARCHAR(255)[];
```

Cependant, la clause `NUMBER INDEX` n'a pas d'équivalent. Par exemple, la 
déclaration suivante :

```sql
TYPE t_liste_qlf_id IS TABLE OF NUMBER INDEX BY VARCHAR2(5);
```

ne peut être portée directement. Il est possible d'utiliser le module _contrib_ 
`hstore` pour émuler cette fonctionnalité.

Références :

* [Extension hstore](https://docs.postgresql.fr/current/hstore.html)
