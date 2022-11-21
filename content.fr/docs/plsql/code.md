---
weight: 4
bookFlatSection: false
title: "Reprise du code PL/SQL"
slug: "reprise-du-code-pl-sql"
---

## Reprise du code PL/SQL

### Chaînes vides et valeur NULL

Pour Oracle, une chaîne vide est aussi une valeur `NULL`. Le SGBD PostgreSQL 
fait la différence : soit la chaîne est nulle (`IS NULL`) soit elle est vide.

Certaines requêtes donnant satisfaction sous Oracle peuvent donner des résultats
faux lorsqu'elles sont portées directement. Les cas les plus courants sont la 
comparaison de la valeur d'une colonne avec une chaîne vide et la concaténation 
avec une valeur `NULL`. Il est nécessaire de se reporter à la partie concernant
le portage des requêtes pour le traitement de ces deux cas.

Lorsque l'on porte du code PL/SQL en PL/pgSQL, il faut faire attention au code qui 
l'appelle, côté applicatif : si le développeur de l'application n'a pas pris garde 
de migrer les `''` par des `NULL`, alors il faut prévoir le cas 
où la fonction reçoit des chaînes vides en lieu et place de la valeur `NULL`.

Ainsi, les codes Oracle suivant peuvent poser des problèmes du fait de cette 
confusion entre la chaîne vide et la valeur `NULL` :

```sql
IF vidtarif IS NULL
  THEN
    ...
```

```sql
IF vidtarif IS NOT NULL
  THEN
    ...
```

Pour Oracle, `vidtarif` sera `NULL` si elle est une chaine vide **ou** si
elle est à `NULL`, et ce, indifféremment. Si le code appelle une fonction qui 
a ce type de traitement, mais en lui passant des chaînes vides en argument, 
cela fonctionnera avec Oracle.

Mais il en sera tout autrement avec PostgreSQL. Ce dernier est beaucoup plus 
strict. Il faut donc étendre les tests dans le code migré en PL/pgSQL sous 
PostgreSQL, pour se prémunir de toute écriture non migré dans l'application.

Voici un exemple de portage du premier test donné en exemple plus haut :

```sql
IF COALESCE(vidtarif, '') = ''
  THEN
    ...
```

Dans cet exemple, si `vidtarif` est `NULL`, alors `coalesce` va choisir la 
valeur suivante, soit la chaine vide `''`. L'égalité sera alors vraie. Et si 
`vidtarif` est une chaine vide, l'égalité sera vraie aussi.

Concernant le second test `IS NOT NULL` vu plus haut, son portage est plus subtil :

```sql
IF (vidtarif IS NOT NULL AND vidtarif <> '')
  THEN
    ...
```

Il est également possible d'utiliser l'astuce suivante en début de fonction, ce
qui simplifiera grandement l'écritures des conditions. Ceci ne s'applique pas à
SQL cependant, uniquement à PL/SQL.

```sql
IF vidtarif = '' THEN vidtarif := NULL END;
```

Ora2Pg fait ces conversions de code automatiquement par défaut, la directive 
`NULL_EQUAL_EMPTY` permet de désactiver ce comportement.

### Exécution de requêtes et de fonctions

Lorsqu'un `SELECT` sans clause `INTO` est présent, il doit être remplacé par 
`PERFORM`. Cette transformation est également prise en charge par Ora2Pg.

Ainsi, l'extrait suivant d'une procédure PL/SQL :

```sql
BEGIN
  SELECT ename, sal FROM EMP
   WHERE empno=7902 FOR UPDATE;
  ...
END;
```

sera réécrit de la façon suivante :

```sql
BEGIN
  PERFORM ename, sal FROM EMP
    WHERE empno=7902 FOR UPDATE;
  ...
END;
```

Par ailleurs, l'instruction `EXEC` permettant de récupérer le code retour d'une 
fonction PL/SQL dans une variable n'existe pas dans PostgreSQL. Il faut la réécrire
en utilisant l'instruction `SELECT INTO`. Cette transformation est prise en charge
par Ora2Pg.

Ainsi, l'extrait de code PL/SQL ci-dessous :

```sql
EXEC :a := get_version();
```

sera réécrit de la façon suivante :

```sql
SELECT get_version() INTO a;
```

### Exécution de requêtes dynamiques

Oracle a un ordre `EXECUTE IMMEDIATE` pour exécuter une requête construite 
dynamiquement. Dans PostgreSQL, le mot clé `IMMEDIATE` doit être supprimé car
il n'est pas supporté. En effet, un ordre `EXECUTE` est toujours réalisé 
immédiatement. Cette transformation est réalisée par Ora2Pg.

Par ailleurs, il est préférable de modifier la construction de l'ordre SQL 
dynamique pour utiliser les fonctions `quote_literal` et `quote_ident` de 
PostgreSQL, respectivement pour encadrer les valeurs littérales et les identifiants
(noms d'objets). Cette adaptation permet de se protéger des injections SQL. Elle 
n'est pas prise en charge par Ora2Pg.

Par exemple, l'extrait de code SQL suivant :

```sql
sql_stmt := 'UPDATE employees SET salary = salary + :1 WHERE ' 
            || v_column || ' = :2';
EXECUTE IMMEDIATE sql_stmt USING amount, column_value;
```

devrait être porté de la façon suivante (à noter également, les `:` remplacés 
par `$`) :

```sql
sql_stmt := 'UPDATE employees SET salary = salary + $1 WHERE ' 
            || quote_literal(v_column) || ' = $2';
EXECUTE sql_stmt USING amount, column_value;
```

Références :

* [Exécution dynamique de commandes](https://docs.postgresql.fr/current/plpgsql-statements.html#plpgsql-statements-executing-dyn)

### COMMIT dans une routine

Il n'est pas possible de coder des instructions de validation/invalidation de la
transaction courante, `COMMIT`/`ROLLBACK`, dans une fonction ou dans une
procédure appelée par une fonction.

Dans ces cas de figure il convient de remonter la gestion de la transaction dans
du code de niveau supérieur.

Notons que, de par sa gestion du MVCC, PostgreSQL peut supporter des longues
transactions plus facilement qu'Oracle.  Aussi, certaines instructions de COMMIT
intermédiaires peuvent souvent être purement et simplement supprimées.

### Gestion des exceptions

Le traitement des exceptions diffère quelque peu entre Oracle et PostgreSQL. La
variable `SQLCODE` d'Oracle est le presque équivalent de `SQLSTATE` dans
PostgreSQL. Il est donc nécessaire de transformer `SQLCODE` en `SQLSTATE`, ce
que fait Ora2Pg.

Oracle et PostgreSQL sont fondamentalement différents dans le traitement des
exceptions. La différence la plus notable est la façon dont l'erreur est gérée.
Si une erreur est déclenchée dans un bloc PL/SQL, seule l'instruction
déclenchante est annulée. En conséquence, on voit souvent des points de
sauvegarde déclarés au début du bloc et des instructions `ROLLBACK TO SAVEPOINT`
émises dans le bloc d'exception.

Avec PostgreSQL, quand une exception est récupérée par une clause `EXCEPTION`,
toutes les modifications de la base de données depuis le bloc `BEGIN` sont
automatiquement annulées. C'est un point important à prendre en compte quant au
portage d'une fonction ou d'une procédure PL/SQL.

Dans ce cas, des constructions PL/SQL utilisant des `SAVEPOINT` seront portés
très simplement en supprimant les instructions de traitements des points de
reprises.

Ainsi, le code PL/SQL suivant :

```sql
BEGIN
  SAVEPOINT s1;
  ...
EXCEPTION
  WHEN ... THEN
    ROLLBACK TO s1;
    ...
  WHEN ... THEN
    ROLLBACK TO s1;
    ...
END;
```

sera traduit de la façon suivante en PL/pgSQL :

```sql
BEGIN
  ...
EXCEPTION
  WHEN ... THEN
    ...
  WHEN ... THEN
    ...
END;
```

Le nom de certaines exceptions doit être transposé. Le tableau ci-dessous donne 
les correspondances entre les exceptions Oracle et les exceptions PostgreSQL 
qu'il est nécessaire de modifier :

| Exception Oracle | Exception PostgreSQL |
|------------------|----------------------|
| STORAGE_ERROR    | OUT_OF_MEMORY        |
| ZERO_DIVIDE      | DIVISION_BY_ZERO     |
| INVALID_CURSOR   | INVALID_CURSOR_STATE |
| dup_val_on_index | unique_violation     |

Enfin, la fonction Oracle `raise_application_error` doit être transformée en 
`RAISE EXCEPTION`. Le code Oracle suivant :

```sql
raise_application_error(
  -20000, 
  'Unable to create a new job: a job is currently running.'
);
```

doit être transposé de la façon suivante :

```sql
RAISE EXCEPTION 
  'Unable to create a new job: a job is currently running';
```

Références :

* [Codes d'erreurs de PostgreSQL](https://docs.postgresql.fr/current/errcodes-appendix.html)
* [Annulation implicite après une exception](https://docs.postgresql.fr/current/plpgsql-porting.html#PLPGSQL-PORTING-EXCEPTIONS)

### SELECT INTO

L'instruction `SELECT ... INTO ...` nécessite d'être adaptée pour qu'elle se 
comporte sous PostgreSQL de la même façon qu'elle le ferait sous Oracle. En 
effet, la documentation de PostgreSQL indique : « L'option `STRICT` correspond au 
comportement du `SELECT INTO` d'Oracle PL/SQL et des instructions relatives. ».

Il est donc nécessaire d'ajouter le mot clé `STRICT` après `INTO` lorsqu'une
exception sur `NO_DATA_FOUND` ou `TOO_MANY_ROWS` est traitée dans le même bloc
de code.

Ainsi, l'extrait suivant d'une procédure PL/SQL Oracle :

```sql
BEGIN   
  SELECT idgroupetarif INTO vgroupevente
    FROM groupetarif
   WHERE classetarif = 'V'
     AND isdefaut = 1
     AND ispublic = vispublic;
EXCEPTION
  WHEN NO_DATA_FOUND THEN    
    vidartpxvte := '-040'; 
    RETURN vidartpxvte;
  WHEN TOO_MANY_ROWS THEN    
    vidartpxvte := '-045'; 
    RETURN vidartpxvte;
END;
```

doit être migré de la façon suivante pour PostgreSQL :

```sql
BEGIN   
  SELECT idgroupetarif INTO STRICT vgroupevente
    FROM groupetarif
   WHERE classetarif = 'V'
     AND isdefaut = 1
     AND ispublic = vispublic;
EXCEPTION
  WHEN NO_DATA_FOUND THEN    
    vidartpxvte := '-040'; 
    RETURN vidartpxvte;
  WHEN TOO_MANY_ROWS THEN    
    vidartpxvte := '-045'; 
    RETURN vidartpxvte;
END;
```

Références :

* [Exécuter une requête avec une seule ligne de résultats](https://docs.postgresql.fr/current/plpgsql-statements.html#PLPGSQL-STATEMENTS-SQL-ONEROW)

### BULK COLLECT

La notion de `BULK COLLECT` n'existe pas sous PostgreSQL. Cette fonctionnalité 
d'Oracle charge le résultat d'une requête dans un tableau et permet de parcourir 
ensuite ce tableau.

Par exemple, ce code Oracle :

```sql
CREATE PROCEDURE tousLesAuteurs
IS
  TYPE my_array IS varray(100) OF VARCHAR(25);
  temp_arr my_array;
BEGIN
  SELECT nom BULK COLLECT INTO temp_arr FROM auteurs ORDER BY nom;
  FOR i IN temp_arr.first .. temp_arr.last LOOP
    DBMS_OUTPUT.put_line(i || ') nom: ' || temp_arr..(i));
  END LOOP;
END tousLesAuteurs;
```

peut être traduit sous PostgreSQL de la façon suivante :

```sql
CREATE FUNCTION tousLesAuteurs() RETURNS VOID 
AS $$
DECLARE
  temp_arr VARCHAR(25)[];
BEGIN
  temp_arr := (SELECT nom FROM auteurs ORDER BY nom);
  FOR i IN array_lower(temp_arr,1) .. array_upper(temp_arr,1) LOOP
    RAISE NOTICE '% ) nom: %', i,  temp_arr..(i);
  END LOOP;
END;
$$ LANGUAGE plpgsql;
```

### Fonction instr

Oracle propose une fonction `instr`. La documentation de PostgreSQL propose une
implémentation en PL/pgSQL équivalente, dans l'annexe de la section concernant
le [portage de PL/SQL vers PL/pgSQL][plpgsql-porting].

[plpgsql-porting]: https://docs.postgresql.fr/current/plpgsql-porting.html#PLPGSQL-PORTING-APPENDIX