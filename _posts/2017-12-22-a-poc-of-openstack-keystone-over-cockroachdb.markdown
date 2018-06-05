---
layout: post
title: A POC of OpenStack Keystone over CockroachDB
date: 2017-12-22
categories: OpenStack CockroachDB
author: Ronan-Alexandre Cherrueau
---
In the [Inria&rsquo;s Discovery initiative](https://beyondtheclouds.github.io/), we get in touch with
[CockroachLabs](https://www.cockroachlabs.com/docs/stable/transactions.html#transaction-retries) guys with an idea: make Keystone supports CockorachDB.
So we give it a try and you can find a very first result on our
[GitHub](https://github.com/BeyondTheClouds/openstack-cockroachdb-dev/). The GitHub project consists of a Vagrant file that spawns a VM
with a CockroachDB database and then installs Keystone with Devstack
using CockroachDB as backend.

CockroachDB claims to support the PostgreSQL protocol. It also
provides support for SQLAlchemy that mostly inherits from the
PostgreSQL one. So, making Keystone working with CockroachDB should be
easy peasy right? Almost! The rest of this post starts by explaining
why we choose Keystone for our POC. It then describes what we have
done to make Keystone works over CockorachDB.


# Focus on Keystone first, conquer all services then

OpenStack is a huge project, made of dozens of services. All services
are developed independently, but built with common principals. In
particular, a service stores its entire state in a database relying on
a library called [`oslo.db`](https://docs.openstack.org/oslo.db/latest/) that abstracts databases binding. Under the
hood, `oslo.db` wraps SQLAclhemy. Thus, OpenStack supports MariaDB and
PostgreSQL.

From all OpenStack services, we decided to focus first on [Keystone](https://docs.openstack.org/keystone/latest/).
Keystone is one of the core services of OpenStack. It manages all the
authentication part of the OpenStack process. We focus on Keystone
with the following idea: if we can make Keystone work with
CockroachDB, then we can hopefully make all services work with
CockroachDB thanks to `oslo.db` abstraction.

We choose Keystone because its DB requests are quite simple: Keystone
splits the authentication management in 10 concerns (credentials,
token, domain, &#x2026;), whose code and DB requests are relatively easy to
follow. Keystone presents another advantage when you want to test
OpenStack at scale. To scale, current deployments put instances of
OpenStack in different regions around the globe. Across these regions,
operators use Galera to synchronise Keystones&rsquo; database. Thus, it
makes authentication the same, wherever a client authenticates
herself. However, Galera presents major limitations in case of high
latency such as in Edge computing. So focusing on Keystone will let us
test how the combo Keystone/CockroachDB outshines these limitations.


# sqlalchemy-migrate doesn&rsquo;t work with CockroachDB

Keystone uses a database migration tool named [sqlalchemy-migrate](https://github.com/openstack/sqlalchemy-migrate). This
tool is called during the deployment of Keystone to migrate the
database from its initial versions to its actual version.
Unfortunately, migration failed with the following (partial)
stacktrace:

    DEBUG migrate.versioning.repository [-] Config: OrderedDict([('db_settings', OrderedDict([('__name__', 'db_settings'), ('repository_id', 'keystone'), ('version_table', 'migrate_version'), ('required_dbs', '[]'), ('use_timestamp_numbering', 'False')]))]) __init__ /usr/local/lib/python2.7/dist-packages/migrate/versioning/repository.py:83
    INFO migrate.versioning.api [-] 66 -> 67...
    CRITICAL keystone [-] KeyError: 'cockroachdb'
    ...
    TRACE keystone     visitorcallable = get_engine_visitor(engine, visitor_name)
    TRACE keystone   File "/usr/local/lib/python2.7/dist-packages/migrate/changeset/databases/visitor.py", line 47, in get_engine_visitor
    TRACE keystone     return get_dialect_visitor(engine.dialect, name)
    TRACE keystone   File "/usr/local/lib/python2.7/dist-packages/migrate/changeset/databases/visitor.py", line 62, in get_dialect_visitor
    TRACE keystone     migrate_dialect_cls = DIALECTS[sa_dialect_name]
    TRACE keystone KeyError: 'cockroachdb'

Actually, sqlalchemy-migrate contains dedicated SQL backend codes and
there is no such code for CockroachDB. So, we [developed one](https://github.com/rcherrueau/sqlalchemy-migrate). The code
circumvents some CockroachDB limitations. For instance, CockroachDB
does not support the add of Primary Key constraint. So, we implement
this migration by, first making a new temporary table with the new
primary keys. Then, copying elements of the original table into the
temporary one. And finally, dropping the original table and renaming
the temporary one. You can find that specific snippet at [lines 238-257
of our fork of sqlalchemy-migrate](https://github.com/rcherrueau/sqlalchemy-migrate/blob/95aef2ea10e2b01776a4b5f3c06dfd26b1240994/migrate/changeset/databases/cockroach.py#L238-L257).

Honestly, the support is only partial and not well-tested. A `tox
-epy27` shows that the code successfully passes 90% of the unit tests,
but still fails on cleaning some index. However, our patch lets us
deploy Keystone over CockroachDB with Devstack, which is good enough
for our POC.


# oslo.db contains backend specific code for error handling

The `oslo.db` library contains one file with backend-specific codes
([`oslo_db/sqlalchemy/exc_filters.py`](https://github.com/openstack/oslo.db/blob/stable/pike/oslo_db/sqlalchemy/exc_filters.py)). This file aims at abstracting
database exceptions that differ but target the same class of error
(because each backend produces a specific exception with a specific
error message). Intercepted exceptions are wrapped into a common OS
exception to abstract backend errors in the rest of `oslo.db`.
CockroachDB doesn&rsquo;t produce same errors as PostgreSQL, so we have to
update this class. Note that our POC is not exhaustive since we only
added error messages we saw during Tempest/Rally tests.

You can look at the differences between OpenStack/oslo.db/stable/pike
and our fork [on GitHub](https://github.com/openstack/oslo.db/compare/openstack:stable/pike...BeyondTheClouds:cockroachdb/pike). We only add two lines!


# CockroachDB manages isolation without lock

CockroachDB lets you write transactions that respect ACID properties.
Regarding the &ldquo;I&rdquo; (Isolation), CockroachDB offers snapshot and
serializable isolation but doesn&rsquo;t rely on locks to do that. So every
time there is concurrent editing transactions that end in a conflict,
then you have to retry the transactions. Fortunately, `oslo.db`
already offers [a decorator](https://github.com/openstack/oslo.db/blob/stable/pike/oslo_db/api.py#L84) to do that automatically. But, based on
tests we run with Tempest/Rally, we figured out that some Keystone&rsquo;s
SQL requests needed the decorator.

You can look at the differences between OpenStack/keystone/stable/pike
and our fork [on GitHub](https://github.com/openstack/keystone/compare/stable/pike...BeyondTheClouds:cockroachdb/pike).


# What&rsquo;s next?

You can drop yourself on the VM as a stack user and run Tempest/Rally
tests (see README). Note that modifications we made are minimal. This
is promising for the adoption of CockroachDB by other core services.
Nonetheless:

-   We have to look through the performance prism: Keystone over
    CockroachDB vs. Keystone over Galera. This part may involve more
    modifications of `oslo.db`. One thing we have in mind is the
    management of retrying transactions since CockroachDB&rsquo;s
    documentation [suggests an approach](https://www.cockroachlabs.com/docs/stable/transactions.html#transaction-retries) that doesn&rsquo;t match with actual
    `oslo.db` implementation.

-   We recently start a similar work for [Nova](https://docs.openstack.org/nova/latest/), the compute service of
    OpenStack. Nova produces SQL query that are far more complex than
    Keystone. In particular, Nova produces correlated subqueries,
    *i.e.*, the subqueries are ran against each row returned by the
    main query. Unfortunately, correlated subqueries are not supported
    by CockroachDB. So keep in touch, we will see in a future post how
    we cope with these queries.
