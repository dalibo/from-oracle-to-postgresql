---
weight: 4
bookFlatSection: false
title: "Reprise des séquences"
---

## Reprise des séquences

La reprise des séquences ne nécessite pas un travail important. Les séquences
sont implémentées de la même façon dans PostgreSQL et dans Oracle. Cependant,
certains points nécessitent un peu d'attention.

De manière générale, les clauses précédées de `NO` permettant d'utiliser les 
valeurs par défaut nécessitent de séparer le mot clé `NO` de la clause pour être
porté sous PostgreSQL. Par exemple, la clause `NOMAXVALUE` Oracle doit être 
réécrite `NO MAXVALUE` pour PostgreSQL. Toutefois, la clause Oracle `NOCACHE` 
n'a pas d'équivalent direct dans PostgreSQL, mais on peut la transformer en 
`CACHE 1` ou simplement la supprimer. Seules les clauses `ORDER` et `NOORDER` 
ne trouveront aucun équivalent dans PostgreSQL car elles sont spécifiques à
Oracle RAC.

Références :

* [CREATE SEQUENCE](https://docs.postgresql.fr/current/sql-createsequence.html)

### Utilisation des séquences

Les séquences ne s'utilisent pas de la même manière avec Oracle qu'avec PostgreSQL. 
Oracle a une syntaxe `nom_sequence.operation` tandis que PostgreSQL a une syntaxe 
`operation('nom_sequence')`.

Par exemple, l'appel suivant, valide sous Oracle :

```sql
nom_sequence.nextval
```

sera transposé de la façon suivante sous PostgreSQL :

```sql
nextval('nom_sequence')
```

Références :

* [CREATE SEQUENCE](https://docs.postgresql.fr/current/sql-createsequence.html)
* [Fonctions de manipulation de séquences](https://docs.postgresql.fr/current/functions-sequence.html)
