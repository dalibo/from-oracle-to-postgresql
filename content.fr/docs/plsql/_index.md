---
weight: 4
bookFlatSection: false
title: "Portage du code PL/SQL vers PL/pgSQL"
url: "portage-du-code-pl-sql-vers-pl-pgsql"
bookCollapseSection: true
---

# Portage du code PL/SQL vers PL/pgSQL

## Principales différences entre PL/SQL et PL/pgSQL

Bien que le SGBD Oracle supporte également la programmation Java côté serveur, 
et le SGBD PostgreSQL de nombreux langages (PL/Perl, PL/Python, PL/R, etc.), 
cette partie ne traite que du portage du PL/SQL en code PL/pgSQL.

Le langage PL/pgSQL de PostgreSQL est conçu à l'origine pour être ressemblant au 
langage PL/SQL d'Oracle. Cependant, ce sont deux langages différents qui nécessitent 
un certain travail pour passer de l'un à l'autre.

Tout d'abord, on note l'absence de _package_ dans PostgreSQL. Il est donc 
nécessaire d'utiliser un moyen de contournement pour émuler leur fonctionnement.

Principales différences :

* pas de packages ;
* code compilé à la première exécution, pas de façon globale ;
* contrairement aux fonctions, les procédures ne peuvent pas retourner de données ;
* pas de transaction autonome ;
* pas de fonctionnalités comme les directories… PL/PgSQL ne manipule pas de fichiers.

Pour ce dernier point, les autres langages PL comblent ce manque (et plus). Il 
est aussi à noter que le langage d'écriture de la fonction est transparent pour
l'appelant.

## Portage des packages

Un _package_ Oracle, ou paquet de fonctions, est le regroupement logique de 
variables et de procédures stockées. Il n'existe pas de notion de paquets de 
fonctions dans PostgreSQL. Pour simplifier le portage, Ora2Pg va créer un schéma 
portant le nom du paquet et importer les fonctions dans ce schéma. Ceci permet 
de garder la notation Oracle `PACKAGE.PROCEDURE` qui sera en fait sous 
PostgreSQL `SCHEMA.FONCTION`.

Oracle permet également de définir des fonctions à l'intérieur d'autres fonctions. 
PostgreSQL ne le permet pas avec PL/PgSQL. Elles devront être extraites du corps 
de leur fonction parente et déclarées comme les autres fonctions.

La notion de variable globale n'existe pas sous PL/PgSQL. Pour pouvoir émuler le
comportement des variables globales, on peut utiliser les variables
utilisateurs, qu'il faudra définir dans le fichier de configuration
`postgresql.conf`, au niveau de la base, de l'utilisateur ou de la session.

Celles-ci doivent être préfixée par un espace de nom au choix, pour ne pas
rentrer en conflit avec les variables systèmes. Une bonne pratique consiste à
préfixer une variable par le nom du schéma qui imite le _package_. La variable
reste locale à une session et ne persiste pas à sa fermeture ni ne peut se
partager entre plusieurs sessions.

Par exemple, pour créer une variable globale nommée `id_region`, il suffit 
d'utiliser la commande `SET` :

```sql
SET monschema.id_region = '38';
```

et pour utiliser sa valeur dans la même session :

```sql
SELECT current_setting('monschema.id_region') AS id_region;
```

Il est également possible d'utiliser la fonction `set_config` à l'intérieur
d'une fonction PL/pgSQL. Le troisième paramètre de la méthode permet de
déterminer si la variable est locale à la transaction ou globale dans la
session.

```sql
PERFORM set_config('monschema.id_region', '38', false);
```

Dans un bloc PL/pgSQL, la méthode `current_setting` peut être directement
employée pour assignée la valeur dans une variable ou à travers le mot-clé
`INTO`.

```sql
a := current_setting('monschema.id_region');
```

Il est également possible d'utiliser une table pour définir ces variables et 
leurs valeurs. Par ailleurs, certains langages PL, comme PL/Perl par exemple, 
disposent quant à eux, de variables globales.
