---
weight: 2
bookFlatSection: false
title: "Portage des triggers"
previouspage: "routines"
nextpage: "controls"
---

## Portage des triggers

### Structure d'un trigger

Dans Oracle, la déclaration du trigger embarque également le code du trigger. 
Dans PostgreSQL, le trigger et le code du trigger sont deux choses distinctes :
le trigger appelle une fonction trigger selon les évènements sur lesquels il 
doit réagir.

Ainsi, pour l'exemple du trigger `print_salary_changes` pour Oracle :

```sql
CREATE OR REPLACE TRIGGER print_salary_changes
  BEFORE DELETE OR INSERT OR UPDATE ON emp
  FOR EACH ROW
WHEN (new.empno > 0)
BEGIN
...
END;
```

sa déclaration sera scindée en deux au moment du portage :

```sql
CREATE OR REPLACE FUNCTION trigger_fct_print_salary_changes() 
  RETURNS TRIGGER AS $$
BEGIN
...
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE TRIGGER print_salary_changes
  BEFORE DELETE OR INSERT OR UPDATE ON emp
  FOR EACH ROW WHEN (NEW.empno > 0)
     EXECUTE FUNCTION tgr_print_salary_changes();
```

On notera que la fonction trigger n'a pas de paramètres et retourne un objet de 
type `TRIGGER`, spécifique à une fonction trigger. En ce qui concerne les 
triggers exécutés pour chaque ligne affectée par le DML, l'ordre de déclaration 
du trigger est quant à lui quasi-identique à l'ordre Oracle.

Les triggers sur instruction pour Oracle nécessitent une adaptation de l'ordre de 
déclaration du trigger. Dans Oracle, lorsque la mention `FOR EACH ROW` n'est pas 
précisée, le trigger est un trigger équivalent au type `FOR EACH STATEMENT`. 
Dans PostgreSQL, si ce n'est pas précisé, le trigger est un trigger `FOR EACH ROW` 
- les triggers sur instruction nécessitent donc d'être adapté en conséquence. 
Ora2Pg supporte cette opération.

Ainsi, l'équivalent du trigger `FOR EACH STATEMENT` Oracle s'écrit de cette 
manière :

```sql
CREATE OR REPLACE TRIGGER Log_emp_update
AFTER UPDATE ON Emp_tab
BEGIN
    INSERT INTO Emp_log (Log_date, Action)
        VALUES (SYSDATE, 'Emp_tab COMMISSIONS CHANGED');
END;
```

Il nécessitera d'être réécrit de la façon suivante pour PostgreSQL :

```sql
CREATE OR REPLACE FUNCTION trigger_fct_log_emp_update() 
  RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO emp_log (log_date, action)
        VALUES (LOCALTIMESTAMP, 'Emp_tab COMMISSIONS CHANGED');
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE TRIGGER log_emp_update
  AFTER UPDATE ON emp_tab
  FOR EACH STATEMENT
     EXECUTE FUNCTION trigger_fct_log_emp_update();
```

PostgreSQL supporte les triggers DML suivants :

* `BEFORE` et `AFTER` ;
* `FOR EACH ROW` et `FOR EACH STATEMENT` ;
* sur `DELETE`, `UPDATE`, `INSERT` (`COPY` est géré par ce même trigger) et 
`TRUNCATE` ;
* la clause `UPDATE OF nom_colonne_1 [, nom_colonne_2 ... ]` est aussi supportée ;
* la clause `REFERENCING` des triggers `FOR EACH STATEMENT` est aussi supportée 
pour rendre visible sous forme de table l'ensemble des changements apportés par 
une requête ;
* conditionnels, avec la clause `WHEN (condition)`.

Références :

* [Fonctions Trigger](https://docs.postgresql.fr/current/plpgsql-trigger.html)
* [CREATE TRIGGER](https://docs.postgresql.fr/current/sql-createtrigger.html)

### Retour d'un trigger

PostgreSQL impose de retourner les enregistrements dans les triggers avant action 
(trigger `BEFORE`). Dans le cas contraire, la valeur `NULL` est retournée, 
bloquant ainsi la mise à jour effective des données. C'est un comportement 
différent de celui d'Oracle, pour lequel le retour est implicite.

Ainsi, le trigger Oracle suivant :

```sql
CREATE TRIGGER gen_id FOR produit
  BEFORE INSERT
  DECLARE noitem INTEGER;
As
BEGIN
  SELECT max(no_produit) INTO noitem FROM produit;
  NEW.no_produit := noitem+1;
END;
```

devra être transformé de la façon suivante :

```sql
CREATE FUNCTION gen_id () RETURNS TRIGGER AS $$
DECLARE
    noitem INTEGER;
BEGIN 
    SELECT INTO noitem max(no_produit) FROM produit;
    IF noitem ISNULL THEN
      noitem:=0;
    END IF;
    NEW.no_produit:=noitem+1;
    RETURN NEW;
  END;
$$
LANGUAGE 'plpgsql';

CREATE TRIGGER trig_before_ins_produit BEFORE INSERT ON produit 
  FOR EACH ROW 
  EXECUTE PROCEDURE gen_id();
```

Références :

* [Fonctions Trigger](https://docs.postgresql.fr/current/plpgsql-trigger.html)

### Triggers DML

Concernant le portage du code des triggers, ces derniers nécessitent plusieurs 
adaptations. Tout d'abord, les pseudo-tables `:new` et `:old` dans un trigger
Oracle doivent être transposés entre `NEW` et `OLD` dans un trigger PostgreSQL.

Enfin, les codes d'opérations `INSERTING`, `UPDATING` et `DELETING` doivent 
être remplacés par une comparaison sur la variable interne `TG_OP` :
`TG_OP = 'INSERT'`, `TG_OP = 'UPDATE'` et `TG_OP = 'DELETE'`.

Le trigger Oracle suivant :

```sql
CREATE OR REPLACE TRIGGER testtgop
BEFORE INSERT OR DELETE OR UPDATE
ON emp
FOR EACH ROW
BEGIN
   IF INSERTING THEN
      ...
   ELSIF UPDATING THEN
      ...
   ELSIF DELETING THEN
      ...
   END IF;
END;
```

sera donc porté de la façon suivante :

```sql
CREATE OR REPLACE FUNCTION trigger_fct_testtgop() RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    ...
  ELSIF TG_OP = 'UPDATE' THEN
    ...
  ELSIF TG_OP = 'DELETE' THEN
    ...
  END IF;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE TRIGGER testtgop
  BEFORE DELETE OR INSERT OR UPDATE ON emp
  FOR EACH ROW
     EXECUTE PROCEDURE trigger_fct_testtgop();
```

Références :

* [Fonctions Trigger](https://docs.postgresql.fr/current/plpgsql-trigger.html)

### Triggers INSTEAD OF

Ce type de trigger nécessite les mêmes adaptations que les triggers DML.

### Triggers DDL et event

Pour reprendre certaines fonctionnalités offertes par les triggers sur DDL et 
sur événement d'Oracle, PostgreSQL dispose d'un mécanisme de triggers sur 
événement, liés aux changements de structure de la base de données 
(instructions DDL de type `CREATE`, `ALTER`, `DROP`, `GRANT`, etc).

Ces triggers se déclenchent sur 4 types d'événement :
* au démarrage d'une commande de DDL ;
* à la terminaison d'une commande de DDL ;
* à la suppression d'un objet ;
* à la réécriture d'une table.

Pour les suppressions d'objet ou les réécritures de table, une interface permet 
à une fonction PL/pgSQL de, par exemple, tracer ou bloquer l'opération.

Néanmoins, pour ce qui est des événements début et fin d'une commande, 
l'exploitation des données de contexte fournies par PostgreSQL nécessite du 
code en C.

Comme pour les triggers DML, l'objet `EVENT TRIGGER` créé doit être associé à 
une fonction retournant un `event_trigger`, définie au préalable.

Références :

* [Event triggers](https://docs.postgresql.fr/current/event-triggers.html)
