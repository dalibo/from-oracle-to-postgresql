---
weight: 7
bookFlatSection: false
title: "Packages propriétaires"
previouspage: "specials"
---

## Packages propriétaires

Les paquets de fonctions et de procédures fournis par Oracle, n'ont pas ou peu
d'équivalents dans PostgreSQL. Il est possible de chercher à les réécrire
entièrement ou bien de s'appuyer sur des contributions libres pour obtenir un
comportement proche de ceux proposés par Oracle.

### Appels au package DBMS_OUTPUT

Les appels aux fonctions de sortie Oracle `DBMS_OUTPUT.put_line`, 
`DBMS_OUTPUT.put` et `DBMS_OUTPUT.new_line` sont remplacés par `RAISE NOTICE` 
par Ora2Pg.

Ainsi, la procédure Oracle suivante :

```sql
CREATE PROCEDURE tousLesAuteurs
IS
  TYPE my_array IS varray(100) OF VARCHAR(25);
  temp_arr my_array;
BEGIN
  ...
  DBMS_OUTPUT.put_line(i || ') nom: ' || temp_arr..(i));
  ...
END tousLesAuteurs;
```

sera traduite de la façon suivante pour PostgreSQL :

```sql
CREATE FUNCTION tousLesAuteurs() RETURNS VOID 
AS $$
DECLARE
  temp_arr VARCHAR(25)[];
BEGIN
  ...
  RAISE NOTICE '% ) nom: %', i,  temp_arr..(i);
  ...
END;
$$ LANGUAGE plpgsql;
```

### Appels aux packages DBMS_XXX

Certains sont implémentés dans la librairie
[Orafce](https://github.com/orafce/orafce), comme :

* UTL_FILE
* DBMS_PIPE
* DBMS_OUTPUT
* DBMS_ALERT

Certaines fonctionnalités avancées de ces modules Oracle peuvent aussi être 
intégrées dans des outils externes à PostgreSQL ou des extensions :

* Oracle Advanced Queuing => voir [PgQ](https://pgq.github.io/extension/pgq/files/external-sql.html)
* Oracle Jobs scheduler => voir 
    [pgAgent](https://www.pgadmin.org/docs/pgadmin4/development/pgagent.html) 
  / [pg_timetable](https://pg-timetable.readthedocs.io/en/master/) 
  / [pg_cron](https://github.com/citusdata/pg_cron)


Et certaines autres peuvent très facilement être écrites avec un language
procédural étendu comme Perl. Par exemple, si vous utilisiez le module
`UTL_SMTP` pour envoyer des emails depuis votre base de données, le code suivant
fera très simplement la même chose :

```sql
CREATE OR REPLACE FUNCTION send_email(name, INET, TEXT, TEXT, TEXT)
RETURNS INTEGER AS $body$
  use Net::SMTP;
  my ($Db, $Ip, $sendTo, $Subject, $Message) = @_;
  my $smtp = Net::SMTP->new("mailhost", Timeout => 60);
  $smtp->mail("$Db\@$Ip");
  $smtp->recipient($sendTo);
  $smtp->data();
  $smtp->datasend("To: $sendTo\n");
  $smtp->datasend("Subject: $Subject\n");
  $smtp->datasend("Content-Type: text/plain;\n\n");
  $smtp->datasend("$Message\n");
  $smtp->dataend();
  $smtp->quit();
  return 1;
$body$ language 'plperlu';

SELECT send_email(
  current_database(), inet_server_addr(), 
  'dba@dom.com', 'test pg_utl_smtp', 'Just a test'
);
```

Références :

* [PGQ Tutorial](https://wiki.postgresql.org/wiki/PGQ_Tutorial)
