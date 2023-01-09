---
weight: 3
bookFlatSection: false
title: "Structures de contrôles"
previouspage: "triggers"
nextpage: "code"
---

## Structures de contrôles

Les boucles et les différentes structures de contrôles ne nécessitent pas de 
portage particulier, en dehors des instructions `GOTO` et `FORALL`. 

### Boucle FOR REVERSE

La boucle `FOR ... REVERSE` présente également la particularité de devoir inverser
les bornes `min` et `max` dans les boucles `FOR ... IN ... REVERSE min..max`.

Le code Oracle est le suivant :

```sql
FOR i IN REVERSE 1..10 BY 2 LOOP
  ...
END LOOP;
```

Il devra être modifié de la façon suivante :

```sql
FOR i IN REVERSE 10..1 BY 2 LOOP
  ...
END LOOP;
```

Une autre différence importante concerne les boucles `FOR` sur des requêtes 
(autres que des curseurs). Dans le code PL/SQL, sous Oracle, les variables cibles 
sont déclarées implicitement. Pour adapter une telle fonction à PostgreSQL, il 
est nécessaire de déclarer les variables cibles dans le bloc `DECLARE` de la 
fonction portée. Cela a également pour avantage de laisser cette variable 
accessible à la sortie de la boucle.

Ainsi, l'extrait de code PL/SQL suivant :

```sql
DECLARE
  v_date1 DATE, v_date2 DATE;
BEGIN
  FOR r IN (
    SELECT numberprgn FROM t2 
     WHERE status = 'ordered' AND assigned_dept = 'departt'
  )
LOOP
  SELECT min(datestamp) INTO v_date1
    FROM T1 WHERE NUMBER = r.numberprgn  AND description = 'etat1';

  SELECT min(datestamp) INTO v_date2
    FROM T1 WHERE NUMBER = r.numberprgn  AND description = 'etat2';
END LOOP;
```

sera porté de la façon suivante, en supposant que les variables `v_date1` et 
`v_date2` doivent être portées en type `TIMESTAMP` :

```sql
DECLARE
  v_date1 DATE, v_date2 TIMESTAMP;
  r record;
BEGIN
  FOR r IN (
    SELECT numberprgn FROM t2
    WHERE status = 'ordered' AND assigned_dept = 'departt'
  )
LOOP
  SELECT min(datestamp) INTO STRICT v_date1
    FROM T1 WHERE number = r.numberprgn  AND description = 'etat1';

  SELECT min(datestamp) INTO STRICT v_date2
    FROM T1 WHERE number = r.numberprgn  AND description = 'etat2';
END LOOP;
```

Références :

* [boucle FOR](https://docs.postgresql.fr/current/plpgsql-control-structures.html#plpgsql-integer-for)
* [Portage d'Oracle PL/SQL](https://docs.postgresql.fr/current/plpgsql-porting.html)

### Instruction GOTO

L'instruction `GOTO` n'a pas d'équivalent sous PostgreSQL et nécessite une 
réécriture de code. L'usage de cette instruction est souvent déconseillée, mais 
il n'est pas rare de la retrouver pour contrôler d'exécution d'une boucle, par 
exemple pour aller à l'itération suivante ou simplement sortir de la boucle.

Ainsi, le code PL/SQL suivant :

```sql
LOOP
  FETCH curs_courant INTO courant;
  EXIT WHEN curs_courant%NOTFOUND;
  IF v_cat = 'YYY' THEN
    v_cat := courant;
    GOTO DEBUT;
  END IF;
  IF courant = 'N1' THEN
    v_cat := courant;
    GOTO FIN;
  END IF;
  ...
<<DEBUT>>
NULL;
END LOOP;
<<FIN>>
CLOSE curs_courant;
```

nécessitera d'être réécrit de la façon suivante :

```sql
LOOP
  FETCH curs_courant INTO courant;
  IF NOT FOUND THEN
    EXIT;
  END IF;
  IF v_cat = 'YYY' THEN
    v_cat := courant;
    CONTINUE;
  END IF;
  IF courant = 'N1' THEN
    v_cat := courant;
    EXIT;
  END IF;
  ...
END LOOP;
CLOSE curs_courant;
```

### Boucles FORALL

L'instruction `FORALL` n'a pas d'équivalent dans PostgreSQL. La boucle `FORALL`, 
utilisée pour parcourir les lignes d'une collection sous Oracle, n'existe pas non 
plus sous PostgreSQL. Cependant, son implémentation sous PostgreSQL est simple car 
il s'agit uniquement de parcourir le tableau de la collection.

Ainsi, la procédure PL/SQL suivante :

```sql
CREATE OR REPLACE PROCEDURE allEmployees
IS
  TYPE v_array IS varray(50) OF CHAR(14);
  arr_emp v_array;
BEGIN
  SELECT ename BULK COLLECT INTO arr_emp
  FROM emp
  ORDER BY ename;
  FORALL i IN 1..arr_emp.COUNT
    dbms_output.put_line('|name'||arr_emp..(i)||'i'|| i);
END allEmployees;
```

sera réécrite de la façon suivante :

```sql
CREATE OR REPLACE FUNCTION allEmployees()
RETURNS VOID AS $body$
DECLARE
  arr_emp CHAR(14)[];
BEGIN
  arr_emp := array ( SELECT ename FROM emp ORDER BY ename);
  FOR i IN array_lower(arr_emp,1)..array_upper(arr_emp,1) LOOP
    RAISE NOTICE '|name % i %', arr_emp[i], i;
  END LOOP;
END;
$body$ LANGUAGE plpgsql;
```