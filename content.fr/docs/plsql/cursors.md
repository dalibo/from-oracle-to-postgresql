---
weight: 5
bookFlatSection: false
title: "Curseurs"
previouspage: "code"
nextpage: "specials"
---

## Curseurs

La notation des variables _curseurs_ est quelque peu différente entre Oracle et
PostgreSQL.

### Déclaration des curseurs

Sous Oracle, on déclare un curseur de cette façon : `CURSOR moncurseur`. Il 
est nécessaire d'inverser cette déclaration pour la rendre compatible avec 
PostgreSQL : `moncurseur CURSOR`. Ce cas est traité par Ora2Pg.

Les curseurs de type `REF CURSOR` et `SYS_REFCURSOR` sous Oracle doivent 
également être modifiés en `REFCURSOR` sous PostgreSQL.

Sous Oracle, le mot clé `IN` dans la déclaration des curseurs permet de passer 
des paramètres au curseur. Ce mot clé est inutile avec PostgreSQL, il suffit de 
la supprimer.

La déclaration suivante pour Oracle :

```sql
CURSOR curs_lignes_commande(no_cde IN VARCHAR2) IS 
  SELECT * FROM lignes_commande 
   WHERE num_cde = no_cde;
```

doit être transposé de cette façon pour PostgreSQL :

```sql
curs_lignes_commande CURSOR (no_cde VARCHAR) FOR
  SELECT * FROM lignes_commande 
   WHERE num_cde = no_cde;
```

Références :

* [Types record](https://docs.postgresql.fr/current/plpgsql-declarations.html#PLPGSQL-DECLARATION-RECORDS)

### Retour d'un curseur

Le type retourné lors de la manipulation des curseurs est un enregistrement 
`RECORD` et non pas `nom_curseur%ROWTYPE` sous Oracle. Avec PostgreSQL, il 
est possible à la lecture du curseur de placer cet enregistrement dans une cible 
qui peut être une variable ligne, une variable `record` ou une liste de variables 
simples séparées par des virgules.

Par exemple, la déclaration d'une référence sur un curseur se fait de la façon 
suivante sous Oracle :

```sql
TYPE return_cur IS REF CURSOR RETURN ma_table%ROWTYPE;
p_retcur return_cur;
```

Alors que, sous PostgreSQL, cela s'écrit de la sorte :

```sql
return_cur REFCURSOR;
```

### Sortie d'un curseur

Enfin, le code de sortie d'un curseur doit être modifié. La construction Oracle
`EXIT WHEN ...%NOTFOUND` n'est pas reconnue par PostgreSQL. Elle doit être 
remplacée par une construction de ce type : `IF NOT FOUND THEN EXIT; END IF;`.
La construction `SQL%NOTFOUND` est également à remplacer par `NOT FOUND`. 
Ces deux transformations sont prises en compte par Ora2Pg.

L'extrait de code PL/SQL suivant :

```sql
LOOP
  FETCH c1 INTO my_ename, my_sal, my_hiredate;
  EXIT WHEN c1%NOTFOUND;
  ...
END LOOP;
```

doit être transposé de la façon suivante :

```sql
LOOP
  FETCH c1 INTO my_ename, my_sal, my_hiredate;
  IF NOT FOUND THEN
    EXIT;
  END IF;
  ...
END LOOP;
```
