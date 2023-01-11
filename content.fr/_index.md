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
code embarqué. L'outil Ora2Pg propose une [série de contrôle][assessment]
permettant d'évaluer la complexité des objets présents dans une base pour
produire un rapport avec le coût associé pour chacun d'eux et l'attribution
d'un score global pour la base.

[assessment]: https://ora2pg.darold.net/documentation.html#Migration-cost-assessment

La deuxième phase est également couverte par l'outil Ora2Pg. Un [manuel
d'utilisation][ora2pg] leur étant entièrement dédié, ces deux premières parties
ne seront pas traitées dans le présent document. Cependant, un certain nombre de
conversions mises en œuvre par Ora2Pg sont décrites dans ce guide de portage.

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
