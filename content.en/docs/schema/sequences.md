---
weight: 4
bookFlatSection: false
title: "Sequences migration"
---

## Sequences migration

There is little work on sequences. Sequences are very similar in PostgreSQL and 
Oracle. A few points need attention, though. 

On a general basis, clauses preceded by `NO` to use default values need a space 
between `NO` and the rest of the clause to be ported to PostgreSQL. For example, 
the `NOMAXVALUE` clause from Oracle has to be translated to `NO MAXVALUE` for 
PostgreSQL. However, Oracle's `NOCACHE` clause has no equivalent in PostgreSQL, 
but it can be converted to `CACHE 1`, or simply removed. Only the `ORDER` and 
`NOORDER` have no equivalent in PostgreSQL, as they are a specificity of 
Oracle RAC. 

De manière générale, les clauses précédées de `NO` permettant d'utiliser les 
valeurs par défaut nécessitent de séparer le mot clé `NO` de la clause pour être
porté sous PostgreSQL. Par exemple, la clause `NOMAXVALUE` Oracle doit être 
réécrite `NO MAXVALUE` pour PostgreSQL. Toutefois, la clause Oracle `NOCACHE` 
n'a pas d'équivalent direct dans PostgreSQL, mais on peut la transformer en 
`CACHE 1` ou simplement la supprimer. Seules les clauses `ORDER` et `NOORDER` 
ne trouveront aucun équivalent dans PostgreSQL car elles sont spécifiques à
Oracle RAC.

References:

* [CREATE SEQUENCE](http://www.postgresql.org/docs/current/static/sql-createsequence.html)

### Sequence usage

Sequences aren't use the same way in PostgreSQL and Oracle. Oracle's syntax is 
`sequence_name.operation`, while PostgreSQL's syntax is `operation('sequence_name')`. 

For instance, the following call, from Oracle:

```sql
sequence_name.nextval
```

has to be rewritten to this for PostgreSQL:

```sql
nextval('sequence_name')
```

References:

* [CREATE SEQUENCE](http://www.postgresql.org/docs/current/static/sql-createsequence.html)
* [Sequence Manipulation Functions](https://www.postgresql.org/docs/current/functions-sequence.html)
