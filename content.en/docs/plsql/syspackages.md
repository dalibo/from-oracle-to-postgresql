---
weight: 7
bookFlatSection: false
title: "Oracle's packages"
previouspage: "specials"
---

## Oracle's packages

Proprietary packages have few or no equivalents in PostgreSQL. It is possible
rewrite them entirely or to rely on free contributions to obtain behavior close
to those proposed by Oracle.

### DBMS_OUTPUT package calls

Calls to Oracle's `DBMS_OUTPUT.put_line`, `DBMS_OUTPUT.put` and
`DBMS_OUTPUT.new_line` output functions are replaced to `RAISE NOTICE` by
Ora2Pg.

Thus, the following procedure, from Oracle:

```sql
CREATE PROCEDURE allAuthors
IS
  TYPE my_array IS varray(100) OF VARCHAR(25);
  temp_arr my_array;
BEGIN
  ...
  DBMS_OUTPUT.put_line(i || ') name: ' || temp_arr..(i));
  ...
END allAuthors;
```

will be translated like this for PostgreSQL:

```sql
CREATE FUNCTION allAuthors() RETURNS VOID
AS $$
DECLARE
  temp_arr VARCHAR(25)[];
BEGIN
  ...
  RAISE NOTICE '% ) name: %', i,  temp_arr..(i);
  ...
END;
$$ LANGUAGE plpgsql;
```

### DBMS_XXX package calls

Some are implemented in the [Orafce](https://github.com/orafce/orafce) library,
as:

* UTL_FILE
* DBMS_PIPE
* DBMS_OUTPUT
* DBMS_ALERT

Some advanced features of these Oracle modules can also be integrated into
non-PostgreSQL tools or extensions:

* Oracle Advanced Queuing => see [PgQ](https://pgq.github.io/extension/pgq/files/external-sql.html)
* Oracle Jobs scheduler => see 
    [pgAgent](https://www.pgadmin.org/docs/pgadmin4/development/pgagent.html) 
  / [pg_timetable](https://pg-timetable.readthedocs.io/en/master/) 
  / [pg_cron](https://github.com/citusdata/pg_cron)

And some others can very easily be written with a language extended procedural
like Perl. For example, if you were using the `UTL_SMTP` module to send emails
from your database, the following code will very simply do the same thing:

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

References:

* [PGQ Tutorial](https://wiki.postgresql.org/wiki/PGQ_Tutorial)