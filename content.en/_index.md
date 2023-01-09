---
title: "Porting applications from Oracle to PostgreSQL"
type: docs
bookToc: false
nextpage: "/general"
---

# Porting applications from Oracle to PostgreSQL

## Presentation

This document is a migration guide from an Oracle database to PostgreSQL. The first
part is about differences between Oracle and PostgreSQL, the second about rewriting 
the database's schema, the third about rewriting queries, and the fourth (and last) 
about porting Oracle's PL/SQL procedures to PostgreSQL's PL/pgSQL functions. 

A migration project is often separated into three phases:

* porting the database, with its data;
* porting the program (queries, reports, stored procedures);
* software testing: results comparison, regression testing, benchmarking, etc…

The porting of the database with its data is done with Ora2Pg. As it has its
complete [user's manual][ora2pg], the subject won't be covered by this document.
Nevertheless, some of conversions performed by Ora2Pg are described in this
guide. 

[ora2pg]: https://ora2pg.darold.net/documentation.html

The porting of the application is a very delicate phase. Indeed, the porting of 
an application's queries and stored procedures can be quite complex because of 
each databases' specificities. This porting guide's purpose is to ease this porting. 

Software testing is a crucial phase of such a project. Test data has to cover a very 
large spectrum, to validate the result of each query and function called by the program, 
if possible by a direct comparison between Oracle and PostgreSQL.

## Special thanks

{{% authors %}}
This guide is a collective work that aims to be open to as many people as
possible. We warmly thank here all the people who contributed directly or
indirectly to this work, in particular:
{{% /authors %}}
