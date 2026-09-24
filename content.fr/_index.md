---
title: "Guide de portage Oracle vers PostgreSQL"
type: docs
bookToc: false
nextpage: "/general"
---

# Guide de portage Oracle vers PostgreSQL

## Présentation

Ce document est un guide de migration d'une base de données Oracle vers PostgreSQL.
Une première partie traite des différences entre Oracle et PostgreSQL, une seconde
de la réécriture du schéma de la base de données, une troisième de la réécriture
des requêtes SQL et la quatrième (et dernière) partie du portage de procédures
PL/SQL Oracle en fonctions PL/pgSQL PostgreSQL. 

Un projet de migration est souvent découpé en plusieurs phases :

* l'étude de complexité de la base Oracle ou du parc entier à migrer ;
* le portage de la base de données, avec la reprise des données ;
* le portage de l'application (requêtes, rapports et procédures stockées) ;
* la recette : comparaisons des résultats, tests de non régression, performances,
  etc.

La première de ces quatre phases consiste à estimer l'effort (exprimé en heure
ou en jour) que représente la migration de la base, de ses données et de son
code embarqué.

La deuxième phase n’est pas traité dans le présent document.

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
