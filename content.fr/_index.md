---
title: "Guide de portage Oracle vers PostgreSQL"
type: docs
bookToc: false
---

# Guide de portage Oracle vers PostgreSQL

## Présentation

Ce document est un guide de migration d'une base de données Oracle vers PostgreSQL.
Une première partie traite des différences entre Oracle et PostgreSQL, une seconde
de la réécriture du schéma de la base de données, une troisième de la réécriture
des requêtes SQL et la quatrième (et dernière) partie du portage de procédures
PL/SQL Oracle en fonctions PL/pgSQL PostgreSQL. 

Un projet de migration est souvent découpé en trois phases :

* le portage de la base de données, avec la reprise des données ;
* le portage de l'application (requêtes, rapports et procédures stockées) ;
* la recette : comparaisons des résultats, tests de non régression, performances,
  etc.

La première des trois phases est couverte par l'outil Ora2Pg. Un [manuel
d'utilisation][ora2pg] lui étant entièrement dédié, cette partie ne sera pas
traitée dans le présent document. En revanche, un certain nombre de conversions
mises en œuvre par Ora2Pg sont décrites dans ce guide de portage. 

[ora2pg]: https://ora2pg.darold.net/documentation.html

Le portage de l'application est une phase très délicate. En effet, le portage des 
requêtes d'une application et des procédures stockées de la base de données peut
être assez difficile du fait des spécificités de chaque base de données.

Ce guide de portage se veut être un outil permettant de faciliter ce travail de
réécriture. La recette est une phase primordiale dans un tel projet. Il est
nécessaire de préparer un jeu d'essai suffisamment exhaustif et de valider le
résultat de chaque requête et de chaque fonction appelée par l'application, si
possible en comparant les résultats entre Oracle et PostgreSQL.

## Remerciements

{{% authors %}}
Ce guide de portage est une initiative collective qui se veut ouverte au plus
grand nombre. Nous remercions chaleureusement ici toutes les personnes qui ont
contribué directement ou indirectement à cet ouvrage, notamment :
{{% /authors %}}
