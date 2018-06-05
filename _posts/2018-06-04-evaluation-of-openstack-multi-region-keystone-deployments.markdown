---
layout: post
title: Evaluation of OpenStack Multi-Region Keystone Deployments
date: 2018-06-04
author: Marie Delavergne, Ronan-Alexandre Cherrueau, Adrien Lebre
categories: OpenStack CockroachDB
---

{::options parse_block_html="true" /}
<div class="abstract">
OpenStack is the *de facto* open source solution for the management of
cloud infrastructures and emerging solution for edge infrastructures
(*i.e.*, hundreds of geo-distributed micro Data Centers composed of
dozens of servers interconnected through WAN links &#x2013; from 10 to 300
ms of RTT &#x2013; with intermittent connections and possible network
partitions). Unfortunately, some modules used by OpenStack services do
not fulfill edge infrastructure needs. For instance, core services use
the MariaDB Relational Database Management System (RDBMS) to store
their state. However, a single MariaDB may not cope with thousands of
connections and intermittent network access. Therefore, is OpenStack
technically ready for the management of an edge infrastructure? To
find this out, we built <a href="https://github.com/BeyondTheClouds/juice">Juice</a>. Juice tests and evaluates the
performance of Relational Database Management Systems (RDBMS) in the
context of a distributed OpenStack with high latency network. The
following study presents performance tests results together with an
analysis of these results for three RDBMS: (1) MariaDB, the OpenStack
default RDBMS; (2) Galera, the multi-master clustering solution for
MariaDB; (3) And CockroachDB, a NewSQL system that takes into account
the locality to reduce network latency effects.

Note: this post is currently in draft status. Experiment results lack
of explanation. It is going to be filled in the following days.

</div>
{::options parse_block_html="false" /}

# Table of Contents

<nav id="table-of-contents">
<ul>
<li><a href="#introduction">Introduction</a></li>
<li><a href="#openstack-at-the-edge-the-keystone-use-case">OpenStack at the Edge: the Keystone Use-Case</a></li>
<ul>
<li><a href="#mariadb-and-keystone">MariaDB and Keystone</a></li>
<li><a href="#galera-in-multi-master-replication-mode-and-keystone">Galera in multi-master replication mode and Keystone</a></li>
<li><a href="#cockroachdb-abbr-crdb-and-keystone">CockroachDB (<i>abbr.</i> CRDB) and Keystone</a></li>
</ul>
<li><a href="#experiment-parameters">Experiment Parameters</a></li>
<ul>
<li><a href="#a-note-about-the-gridandrsquo5000-testbed-and-study-conditions">A note about the Grid&rsquo;5000 testbed and study conditions</a></li>
<li><a href="#number-of-openstack-instances">Number of OpenStack instances</a></li>
<li><a href="#delay">Delay</a></li>
</ul>
<li><a href="#load-rally-scenarios">Load: Rally Scenarios</a></li>
<ul>
<li><a href="#a-typical-rally-execution">A typical Rally execution</a></li>
<li><a href="#low-and-high-load">Low and high load</a></li>
<li><a href="#list-of-rally-scenarios">List of Rally scenarios</a></li>
<li><a href="#a-note-about-gauging-the-readswrites-ratio">A note about gauging the %reads/%writes ratio</a></li>
</ul>
<li><a href="#extract-reify-and-query-rally-experiments">Extract, Reify and Query Rally Experiments</a></li>
<ul>
<li><a href="#from-json-files-to-python-objects">From Json files to Python Objects</a></li>
<li><a href="#query-rally-results">Query Rally results</a></li>
</ul>
<li><a href="#experiment-analysis">Experiment Analysis</a></li>
<ul>
<li><a href="#number-of-openstack-instances-impact">Number of OpenStack instances impact</a></li>
<li><a href="#delay-impact">Delay impact</a></li>
<li><a href="#taking-into-account-the-user-locality">Taking into account the user locality</a></li>
</ul>
<li><a href="#experiments-outline">Experiments Outline</a></li>
<li><a href="#appendix">Appendix</a></li>
<ul>
<li><a href="#detailed-experiments-results">Detailed experiments results</a></li>
<li><a href="#detailed-rally-scenarios">Detailed Rally scenarios</a></li>
</ul>

</ul>
</nav>


<a id="orgb635510"></a>

# Introduction

Internet of things, virtual reality or network function virtualization
use-cases all require edge infrastructures. An edge infrastructure
*could* be defined as up to hundreds individually-managed and
geo-distributed micro data centers composed of up to dozens of
servers. The expected latency and bandwidth between elements may
fluctuate, in particular because networks can be wired or wireless.
And disconnections between sites may also occur, leading to network
partitioning situations. This kind of edge infrastructure shares
common basis with cloud computing, notably in terms of management.
Therefore developers and operators (DevOps) of an edge infrastructure
expect to find most features that made current cloud solutions
successful. Unfortunately, there is currently no resource management
system able to deliver all these features for the egde.

Building an edge infrastructure resource management system from
scratch seems unreasonable. Hence the Discovery initiative has been
investigating how to deliver such a system leveraging OpenStack, the
cloud computing infrastructure resource manager. From a bird&rsquo;s-eye
view, OpenStack has two type of nodes: data nodes delivering XaaS
capabilities (compute, storage, network, &#x2026;, i.e., data plane) and
control nodes executing OpenStack services (i.e., control plane).
Whenever a user issues a request to OpenStack, the control plane
processes the request which may potentially also affect the data plane
in some manner.

Preliminary studies we conducted enabled us to identify diverse
OpenStack deployment scenarios for the edge: from a fully centralized
control plane to a fully distributed control plane. These deployment
scenarios have been elaborated in the <a href="https://hal.archives-ouvertes.fr/view/index/docid/1812747">Edge Computing Resource
Management System: a Critical Building Block!</a> research paper that will
be presented at the USENIX HotEdge&rsquo;18 Workshop (July 2018).

This post presents the study we conducted regarding the Keystone
Identity Service responsible for authentication and authorization in
OpenStack. By varying the number of OpenStack instances and latency
between instances, we compare the following deployments:

-   One centralized MariaDB handling requests of every Keystone in
    OpenStack instances.
-   A replicated Keystone using Galera Cluster to synchronize databases
    in the different OpenStack instances.
-   A global Keystone using the global geo-distributed CockroachDB
    database.

Note that some results of this study <a href="https://www.openstack.org/videos/vancouver-2018/keystone-in-the-context-of-fogedge-massively-distributed-clouds">have been presented</a> during the
2018 Vancouver OpenStack Summit. This post contains additional
information, though. Notably, this post is based on <a href="https://orgmode.org/">org mode</a> with
<a href="https://orgmode.org/worg/org-contrib/babel/">babel</a>, thus executing it <a href="https://todogh:analyses/notebook.org">TODO:from source</a> computes the following post
including plots for the results.


<a id="org848d5a4"></a>

# OpenStack at the Edge: the Keystone Use-Case

OpenStack comes with several deployment alternatives for the
edge<sup><a id="fnr.1" class="footref" href="#fn.1">1</a></sup>. A <a href="https://www.openstack.org/videos/boston-2017/toward-fog-edge-and-nfv-deployments-evaluating-openstack-wanwide">naive</a> one consists in a
centralized control plane. In this deployment, OpenStack operates an
edge infrastructure as a traditional single data center environment,
except that there is a wide-area network between the control and
compute nodes. A second alternative consists in distributing OpenStack
services by distributing their database and message bus. In this
deployment, all OpenStack instances share the same state thanks to the
distributed database. They also implement intra-service collaboration
thanks to the distributed message bus.

Deploying a multi-region Keystone is <a href="https://www.openstack.org/videos/vancouver-2018/highly-resilient-multi-region-keystone-deployments">a concrete example</a> of these
deployment alternatives. To scale, current OpenStack deployments put
instances of OpenStack in different regions around the globe. But,
operators want to have all of their users and projects available
across all regions. That means, a sort of global Keystone available
from every region. To this aim, an operator got two options. First,
she can centralize Keystone (or its <a href="https://mariadb.org/">MariaDB</a> database) in one region
and make other regions refer to the first one. Second, she can use
<a href="http://galeracluster.com/">Galera Cluster</a> to synchronize Keystones&rsquo; database and thus distribute
the service.

This study targets the *performance evaluation* of a multi-region
Keystone with the MariaDB and Galera deployment alternatives plus a
variant of the Galera alternative based on <a href="https://www.cockroachlabs.com/">CockroachDB</a>. CockroachDB is
a NewSQL database that claims to scale while implementing ACID
transactions. Especially, CockroachDB makes it possible to take into
account the locality of Keystone users to reduce latency effects. The
following section presents each database, their relative Keystone
deployments and expected limitations.

<div class="org-src-container">
<label class="org-src-name"><span class="listing-number">Listing 1: </span>List of Evaluated RDBMS.</label><pre class="src src-python" id="org4460045"><span style="color: #DFAF8F;">RDBMSS</span> = <span style="color: #DCDCCC;">[</span> <span style="color: #CC9393;">'mariadb'</span>, <span style="color: #CC9393;">'galera'</span>, <span style="color: #CC9393;">'cockroachdb'</span> <span style="color: #DCDCCC;">]</span>
</pre>
</div>


<a id="org3bc5c2d"></a>

## MariaDB and Keystone

MariaDB is a fork of MySQL, intended to stay free and under GNU
General Public License. It maintains high compatibility with MySQL,
keeping the same APIs and commands.


#### Multiple OpenStack instances deployment with a single MariaDB

Figure <a href="#org440f085">1</a> depicts the deployment of MariaDB. MariaDB is a
centralized RDBMS and thus, the Keystone backend is centralized in the
first OpenStack instance. Other Keystones of other OpenStack instances
refers to the backend of the first instance.

This kind of deployment comes with two possible limitations. First, a
centralized RDBMS is a SPoF that makes all OpenStack instances
unusable if it crashes. Second, a network disconnection of the,
*e.g.*, third OpenStack instance with the first one makes the third
unusable.


<figure id="org440f085">
<img src='{{ "assets/evaluation-of-openstack-multi-region-keystone-deployments/mariadb.png" | absolute_url }}' alt="mariadb.png">

<figcaption><span class="figure-number">Figure 1: </span>Keystone Deployment with a Centralized MariaDB.</figcaption>
</figure>


<a id="org8b9cf42"></a>

## Galera in multi-master replication mode and Keystone

MariaDB uses Galera Cluster as a synchronous multi-master cluster. It
means that all nodes in the cluster are masters, with active-active
synchronous replication, so it is possible to read or write on every
node at any time. To put it simply, MariaDB Galera Cluster allows
having the same database on every node thanks to synchronous
replication.

To dive more into details, each time a client performs a transaction
request on a node, it is processed as usual until the client issues a
commit. The process is stopped and all changes made to the database in
the transaction are collected in a &ldquo;write-set&rdquo;, along with the primary
keys of the changed rows. The write-set is then broadcasted to every
node with a global identifier<sup><a id="fnr.2" class="footref" href="#fn.2">2</a></sup> to order it regarding
other write-sets. The write-set finally undergoes a deterministic
certification test which uses the given primary keys. It checks all
transactions between the current transaction and the last successful
one to determine whether the primary keys involved conflicts between
each other. If the check fails, Galera rollbacks the whole
transaction, and if it succeeds, Galera applies and commit the
transaction on all nodes.


<figure id="org0f9ec79">
<img src='{{ "assets/evaluation-of-openstack-multi-region-keystone-deployments/commit-galera.gif" | absolute_url }}' alt="commit-galera.gif">

<figcaption><span class="figure-number">Figure 2: </span>Certification Based Replication from <a href="http://galeracluster.com">http://galeracluster.com</a>.</figcaption>
</figure>

This is pretty efficient since it only needs one broadcast to make the
replication, which means the commit does not have to wait for the
responses of the other nodes. But, this means that when it fails, the
entire transaction must be retried and so it may lead to more
conflicts and even deadlocks.


#### Multiple OpenStack instances deployment with Galera

Figure <a href="#org69de1e6">3</a> depicts the deployment of Galera. Galera
synchronizes multiple MariaDB in an active/active fashion. Thus
Keystone&rsquo;s backend of every OpenStack instance is replicated between
all nodes, which allows reads and writes on any instances.

Regarding possible limitations, few rumors stick to Galera. First,
synchronization may suffer from high latency networks. Second,
contention during writes on the database may limit its scalability.


<figure id="org69de1e6">
<img src='{{ "assets/evaluation-of-openstack-multi-region-keystone-deployments/galera.png" | absolute_url }}' alt="galera.png">

<figcaption><span class="figure-number">Figure 3: </span>Keystone Deployment with Synchronization using Galera.</figcaption>
</figure>


<a id="orgac2ad2d"></a>

## CockroachDB (*abbr.* CRDB) and Keystone

CockroachDB is a NewSQL database that uses the Raft protocol (an
alternative version to Lamport&rsquo;s Paxos consensus protocol). It uses
the SQL API to enable SQL requests on every node. These requests are
translated to key-value operations and -if needed- distributed across
the cluster.

CockroachDB implements a single, monolithic sorted map for the keys
and values stored, as seen in <a href="#orge43df05">4</a>. This map is divided in
ranges, which are continuous chunks of this map, with every key being
in a single range, so the ranges will not overlap. Each range is then
replicated (default to three replicas per range) and finally
distributed across the cluster nodes.


<figure id="orge43df05">
<img src='{{ "assets/evaluation-of-openstack-multi-region-keystone-deployments/cockroachdb.png" | absolute_url }}' alt="cockroachdb.png">

<figcaption><span class="figure-number">Figure 4: </span>CockroachDB Ranges, Replicas and Leaseholders (blue points).</figcaption>
</figure>

One of the replicas acts as the leaseholder, a sort of leader that
coordinates all reads and writes for the range. A read only requires
the leaseholder. When a write is requested, the leaseholder prepares
to append it to its log, forward the request to the replicas and when
the quorum is achieved, commit the change by adding it in the log. The
quorum is an agreement from two out of the three replicas to make the
change.


<figure id="orgd95a2dc">
<img src='{{ "assets/evaluation-of-openstack-multi-region-keystone-deployments/commit-cockroachdb.gif" | absolute_url }}' alt="commit-cockroachdb.gif">

<figcaption><span class="figure-number">Figure 5: </span>CockroachDB commit</figcaption>
</figure>


#### Multiple OpenStack instances deployment with CockroachDB

Figure <a href="#orgc0de510">6</a> depicts the deployment of CockroachDB. In this
deployment, each OpenStack instance has its Keystone. The backend is
distributed through key-value stores on every OpenStack instance.
Meaning, the data a Keystone is sought for is not necessarily in its
local key-value store.

CockroachDB is relatively new and we know a few about its limitations,
but first, CockroachDB may suffer from high network latency even
during reads if the current node that gets the requests is not the
leaseholder. Second, as Galera, transaction contention may
dramatically slow down the overall execution. However, CockroachDB
offers locality option to drive the selection of key-value stores
during writes and replication. Thanks to this option it would be
possible to mitigate the impact of latency by ensuring that writes
happen close to the OpenStack operator.


<figure id="orgc0de510">
<img src='{{ "assets/evaluation-of-openstack-multi-region-keystone-deployments/crdb.png" | absolute_url }}' alt="crdb.png">

<figcaption><span class="figure-number">Figure 6: </span>Keystone Deployment with Distributed Backend using CockroachDB.</figcaption>
</figure>


<a id="org30c3d05"></a>

# Experiment Parameters

This section outline the two parameters considered in this study:
first, the number of OpenStack instances and second, network latency.
Later, the section on the locality (see, [Taking into account the user locality](#taking-into-account-the-user-locality)) adds a third
parameter to study heterogeneous network infrastructures.


<a id="orgda231c6"></a>

## A note about the Grid&rsquo;5000 testbed and study conditions

<a href="https://docs.openstack.org/developer/performance-docs/labs/grid5000.html">Grid&rsquo;5000</a> is a large-scale and versatile testbed for experiment-driven
research in all areas of computer science, with a focus on parallel
and distributed computing including Cloud, HPC and Big Data. The
platform gives access to approximately 1000 machines grouped in 30
clusters geographically distributed in 8 sites. This study uses the
<a href="https://www.grid5000.fr/mediawiki/index.php/Nantes:Hardware#ecotype">ecotype cluster</a> made of 48 nodes with each:

-   **CPU:** Intel Xeon E5-2630L v4 Broadwell 1.80GHz (2 CPUs/node, 10
    cores/CPU)
-   **Memory:** 128 GB
-   **Network:** -   eth0/eno1, Ethernet, configured rate: 10 Gbps, model: Intel
        82599ES 10-Gigabit SFI/SFP+ Network Connection, driver: ixgbe
    -   eth1/eno2, Ethernet, configured rate: 10 Gbps, model: Intel
        82599ES 10-Gigabit SFI/SFP+ Network Connection, driver: ixgbe

The deployment of OpenStack relies on <a href="https://github.com/openstack-dev/devstack/tree/stable/pike">devstack stable/pike</a> and uses
default parameter for Keystone (*e.g.,* SQL backend, fernet token,
&#x2026;). Juice deploys a docker version of RDBMS and tweaks a little bit
devstack to ensure Keystone connects to the right RDBMS container.
Note that devstack (and OpenStack) does not support CockroachDB. We
<a href="https://beyondtheclouds.github.io/blog/openstack/cockroachdb/2017/12/22/a-poc-of-openstack-keystone-over-cockroachdb.html">published a post a few months ago</a> about the support of CockroachDB in
Keystone. Note also that RDBMS are stored directly on the memory.


<a id="org9f19b9a"></a>

## Number of OpenStack instances

The OpenStack size (see, lst. <a href="#orgb1865cd">2</a>) defines the number of OpenStack
instances deployed for an experiment. It varies between <code>3</code>, <code>9</code> and
<code>45</code>. A value of <code>3</code>, means Juice deploys OpenStack on three different
nodes, <code>9</code> on nine different nodes, &#x2026; The value of <code>45</code> comes from
the maximum number of nodes available on the ecotype Grid&rsquo;5000
cluster, but Juice is not limited to.

<div class="org-src-container">
<label class="org-src-name"><span class="listing-number">Listing 2: </span>List of Number of OpenStack Instances Deployed.</label><pre class="src src-python" id="orgb1865cd"><span style="color: #DFAF8F;">OSS</span> = <span style="color: #DCDCCC;">[</span> <span style="color: #BFEBBF;">3</span>, <span style="color: #BFEBBF;">9</span>, <span style="color: #BFEBBF;">45</span> <span style="color: #DCDCCC;">]</span>
</pre>
</div>

Experiments that test the impact of the number of OpenStack instances
(see, [Number of OpenStack instances impact](#number-of-openstack-instances-impact)) consider a LAN link
between each OpenStack instances.


<a id="orgbc2e82e"></a>

## Delay

The delay (see, lst. <a href="#org7ba5035">3</a>) defines the network latency between
two OpenStack instances. It is expressed in terms of half the
Round-Trip Times, (*i.e.*, a value of <code>50</code> stands for 100 ms of RTT,
<code>150</code> is 300 ms of RTT). The <code>0</code> value stands for LAN speed which is
approximately 0.08 ms of RTT on the ecotype Grid&rsquo;5000 cluster (10 Gbps
card).

<div class="org-src-container">
<label class="org-src-name"><span class="listing-number">Listing 3: </span>List of Network Latency Between Two OpenStack Instances.</label><pre class="src src-python" id="org7ba5035"><span style="color: #DFAF8F;">DELAYS</span> = <span style="color: #DCDCCC;">[</span> <span style="color: #BFEBBF;">0</span>, <span style="color: #BFEBBF;">50</span>, <span style="color: #BFEBBF;">150</span> <span style="color: #DCDCCC;">]</span>
</pre>
</div>

Juice applies theses network latencies with <code>tc</code> <a href="https://wiki.linuxfoundation.org/networking/netem">netem</a>. Note that
juice applies <code>tc</code> rules on network interfaces dedicated to the RDBMS
communications. Thus, metrics collection and other network
communication are not limited.

Experiments that test the impact of the network latency (see, [Delay
impact](#delay-impact)) are done with 9 OpenStack instances. They make the delay
varies by applying traffic shaping homogeneously between the 9
OpenStack instances.


<a id="org6b5d7b5"></a>

# Load: Rally Scenarios

Juice uses <a href="https://rally.readthedocs.io/en/latest/">Rally</a>, a testing benchmarking tool for OpenStack, to
evaluate how OpenStack control plane behaves at scale. This section
describes Rally scenarios considered in this study. The description
includes the ratio of reads and writes performed on the database. For
a transactional (OLTP) database, depending on the reads/writes ratio,
it could be better to choose one replication strategy to another
(i.e., replicate records on all of your nodes or not).


<a id="org86fa955"></a>

## A typical Rally execution

A Rally executes its load on one instance of OpenStack. Two variables
configure the execution of a Rally scenario: the *times* which is the
number of iteration execution performed for a scenario, and
*concurrency* which is the number of parallel iteration execution.
Thus, a scenario with times of <code>100</code> runs one hundred iterations of
the scenario by a constant load on the OpenStack instance. A
concurrency of <code>10</code> specifies that ten users concurrently achieve the
one hundred iterations. The execution output of such a scenario may
look like this:

    Task 19b09a0b-7aec-4353-b215-8d5b23706cd7 | ITER: 1 START
    Task 19b09a0b-7aec-4353-b215-8d5b23706cd7 | ITER: 2 START
    Task 19b09a0b-7aec-4353-b215-8d5b23706cd7 | ITER: 4 START
    Task 19b09a0b-7aec-4353-b215-8d5b23706cd7 | ITER: 3 START
    Task 19b09a0b-7aec-4353-b215-8d5b23706cd7 | ITER: 5 START
    Task 19b09a0b-7aec-4353-b215-8d5b23706cd7 | ITER: 6 START
    Task 19b09a0b-7aec-4353-b215-8d5b23706cd7 | ITER: 8 START
    Task 19b09a0b-7aec-4353-b215-8d5b23706cd7 | ITER: 7 START
    Task 19b09a0b-7aec-4353-b215-8d5b23706cd7 | ITER: 9 START
    Task 19b09a0b-7aec-4353-b215-8d5b23706cd7 | ITER: 10 START
    Task 19b09a0b-7aec-4353-b215-8d5b23706cd7 | ITER: 4 END
    Task 19b09a0b-7aec-4353-b215-8d5b23706cd7 | ITER: 11 START
    Task 19b09a0b-7aec-4353-b215-8d5b23706cd7 | ITER: 3 END
    Task 19b09a0b-7aec-4353-b215-8d5b23706cd7 | ITER: 12 START
    ...
    Task 19b09a0b-7aec-4353-b215-8d5b23706cd7 | ITER: 100 END


<a id="orgfd329f2"></a>

## Low and high load

The juice tool runs two kinds of load: *low* and *high*. The low load
starts one Rally instance on only one OpenStack instance. The high
load starts as many Rally instances as OpenStack instances.

The high load is named as such because it generates a lot of requests
and thus, a lot of contention on distributed RDBMS. The case of <code>45</code>
Rally instances with a concurrency of <code>10</code> and times of <code>100</code> charges
<code>450</code> constant transactions on the RDBMS up until getting to <code>4,500</code>
iteration.


<a id="org8ff5d81"></a>

## List of Rally scenarios

Here is the complete list of rally scenarios considered in this study.
Values inside the parentheses refer to the percent of reads versus the
percent of writes on the RDBMS. More information about each scenario
is available in the appendix (see, [Detailed Rally scenarios](#detailed-rally-scenarios)).

-   **keystone/authenticate-user-and-validate-token (96.46, 3.54):** Authenticate
    and validate a Keystone token.
-   **keystone/create-add-and-list-user-roles (96.22, 3.78):** Create a
    user role, add it and list user roles for given user.
-   **keystone/create-and-list-tenants (92.12, 7.88):** Create a Keystone
    tenant with random name and list all tenants.
-   **keystone/get-entities (91.9, 8.1):** Get instance of a tenant, user,
    role and service by id&rsquo;s. An ephemeral tenant, user, and role are
    each created. By default, fetches the &rsquo;keystone&rsquo; service.
-   **keystone/create-user-update-password (89.79, 10.21):** Create a
    Keystone user and update her password.
-   **keystone/create-user-set-enabled-and-delete (91.07, 8.93):** Create
    a Keystone user, enable or disable it, and delete it.
-   **keystone/create-and-list-users (92.05, 7.95):** Create a Keystone
    user with random name and list all users.


<a id="orgfbc1a75"></a>

## A note about gauging the %reads/%writes ratio

The %reads/%writes ratio is computed on MariaDB. The gauging code
reads values of status variables <code>Com_xxx</code> that provide statement
counts over all connections (with <code>xxx</code> stands for <code>SELECT</code>, <code>DELETE</code>,
<code>INSERT</code>, <code>UPDATE</code>, <code>REPLACE</code> statements). The SQL query that does
this job is available in listing <a href="#org4083d7a">4</a> and returns the
total number of reads and writes since the database started. Juice
executes that SQL query before and after the execution of one Rally
scenario. After and before values are then subtracted to compute the
number of reads and writes performed during the scenario and finally,
compared to compute the ratio.

<div class="org-src-container">
<label class="org-src-name"><span class="listing-number">Listing 4: </span>Total number of reads and writes performed on MariaDB since the last reboot</label><pre class="src src-sql" id="org4083d7a"><span style="color: #F0DFAF; font-weight: bold;">SELECT</span>
  <span style="color: #DCDCCC; font-weight: bold;">SUM</span><span style="color: #DCDCCC;">(</span>IF<span style="color: #BFEBBF;">(</span>variable_name = <span style="color: #CC9393;">'Com_select'</span>, variable_value, <span style="color: #BFEBBF;">0</span><span style="color: #BFEBBF;">)</span><span style="color: #DCDCCC;">)</span>
     <span style="color: #F0DFAF; font-weight: bold;">AS</span> `Total <span style="color: #F0DFAF; font-weight: bold;">reads</span>`,
  <span style="color: #DCDCCC; font-weight: bold;">SUM</span><span style="color: #DCDCCC;">(</span>IF<span style="color: #BFEBBF;">(</span>variable_name <span style="color: #F0DFAF; font-weight: bold;">IN</span> <span style="color: #D0BF8F;">(</span><span style="color: #CC9393;">'Com_delete'</span>,
                           <span style="color: #CC9393;">'Com_insert'</span>,
                           <span style="color: #CC9393;">'Com_update'</span>,
                           <span style="color: #CC9393;">'Com_replace'</span><span style="color: #D0BF8F;">)</span>, variable_value, <span style="color: #BFEBBF;">0</span><span style="color: #BFEBBF;">)</span><span style="color: #DCDCCC;">)</span>
     <span style="color: #F0DFAF; font-weight: bold;">AS</span> `Total writes`
<span style="color: #F0DFAF; font-weight: bold;">FROM</span>  information_schema.GLOBAL_STATUS;
</pre>
</div>

Note that %reads/%writes may be a little bit more in favor of reads
than what it is presented here because the following also takes into
account the creation/deletion of the Rally context. A basic Rally
context for a Keystone scenario is <code>{"admin_cleanup@openstack":
["keystone"]}</code>. Not sure what does this context do exactly though,
maybe it only creates an admin user&#x2026;

The Juice implementation for this gauging is available on GitHub at
<a href="https://github.com/rcherrueau/juice/blob/02af922a7c3221462d7106dfb2751b3be709a4d5/experiments/read-write-ratio.py">experiments/read-write-ratio.py</a>.


<a id="org12f6a5c"></a>

# Extract, Reify and Query Rally Experiments

The execution of a Rally scenario (such as those seen in the previous
section &#x2013; see [Load: Rally Scenarios](#load-rally-scenarios)) produces a JSON file. The JSON
file contains a list of entries: one for each iteration of the
scenario. An entry then retains the time (in seconds) it takes to
complete all Keystone operations involved in the Rally scenario.

This section provides python facilities to extract and query Rally
results for later analyses. Someone interested by the results and not
by the process to compute them may skip this section and jump to the
next one (see, [Number of OpenStack instances impact](#number-of-openstack-instances-impact)).

An archive with results of all experiments in this study is available
at <a href="https://todo:exp-url">https://todo:exp-url</a>. It contains general metrics collected over the
experiments time such as the CPU/RAM consumption, network
communications (all stored in a influxdb), plus Rally JSON result
files. Let&rsquo;s assume the <code>XPS_PATH</code> variable references the path to the
extracted archive. Listing <a href="#org1372c20">5</a> defines accessors for all
Rally JSON result files thanks to the <a href="https://docs.python.org/3/library/glob.html"><code>glob</code></a> python module. The <code>glob</code>
module finds all paths that match specified UNIX patterns.

<div class="org-src-container">
<label class="org-src-name"><span class="listing-number">Listing 5: </span>Paths to Rally JSON Results File.</label><pre class="src src-python" id="org1372c20"><span style="color: #DFAF8F;">XP_PATHS</span> = <span style="color: #CC9393;">'./ecotype-exp-backoff/'</span>
<span style="color: #DFAF8F;">MARIADB_XP_PATHS</span> = glob.glob<span style="color: #DCDCCC;">(</span>XP_PATHS + <span style="color: #CC9393;">'mariadb-*/rally_home/*.json'</span><span style="color: #DCDCCC;">)</span>
<span style="color: #DFAF8F;">GALERA_XP_PATHS</span> = glob.glob<span style="color: #DCDCCC;">(</span>XP_PATHS + <span style="color: #CC9393;">'galera-*/rally_home/*.json'</span><span style="color: #DCDCCC;">)</span>
<span style="color: #DFAF8F;">CRDB_XP_PATHS</span> = glob.glob<span style="color: #DCDCCC;">(</span>XP_PATHS + <span style="color: #CC9393;">'cockroachdb-*/rally_home/*.json'</span><span style="color: #DCDCCC;">)</span>
</pre>
</div>


<a id="org94a4c9e"></a>

## From Json files to Python Objects

A data class <code>XP</code> retains data of one experiment (i.e., the name of
the Rally scenario, name of database technology, &#x2026; &#x2013; see l.
<a href="#coderef-xp-dataclass-start" class="coderef" onmouseover="CodeHighlightOn(this, 'coderef-xp-dataclass-start');" onmouseout="CodeHighlightOff(this, 'coderef-xp-dataclass-start');">3</a> to <a href="#coderef-xp-dataclass-end" class="coderef" onmouseover="CodeHighlightOn(this, 'coderef-xp-dataclass-end');" onmouseout="CodeHighlightOff(this, 'coderef-xp-dataclass-end');">10</a> of listing <a href="#org491759e">6</a>
for the complete list). Reefing experiment data in a Python object
helps for the later analyses. Indeed, a Python object makes it easier
to filter, sort, map, &#x2026; experiments.

<div class="org-src-container">
<label class="org-src-name"><span class="listing-number">Listing 6: </span>Experiment Data Class.</label><pre class="src src-python" id="org491759e"><span class="linenr"> 1: </span><span style="color: #7CB8BB;">@dataclass</span><span style="color: #DCDCCC;">(</span>frozen=<span style="color: #BFEBBF;">True</span><span style="color: #DCDCCC;">)</span>
<span class="linenr"> 2: </span><span style="color: #F0DFAF; font-weight: bold;">class</span> <span style="color: #7CB8BB;">XP</span>:
<span id="coderef-xp-dataclass-start" class="coderef-off"><span class="linenr"> 3: </span>    scenario: <span style="color: #DCDCCC; font-weight: bold;">str</span>             <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Rally scenario name</span></span>
<span class="linenr"> 4: </span>    rdbms: <span style="color: #DCDCCC; font-weight: bold;">str</span>                <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Name of the RDBMS (e,g, cockcroachdb, galera)</span>
<span class="linenr"> 5: </span>    filepath: <span style="color: #DCDCCC; font-weight: bold;">str</span>             <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Filepath of the JSON file</span>
<span class="linenr"> 6: </span>    oss: <span style="color: #DCDCCC; font-weight: bold;">int</span>                  <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Number of OpenStack instances</span>
<span class="linenr"> 7: </span>    delay: <span style="color: #DCDCCC; font-weight: bold;">int</span>                <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Delay between nodes</span>
<span class="linenr"> 8: </span>    high: <span style="color: #DCDCCC; font-weight: bold;">bool</span>                <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Experiment performed during a high or light load</span>
<span class="linenr"> 9: </span>    success: <span style="color: #DCDCCC; font-weight: bold;">float</span>            <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Success rate (e.g., 1.0)</span>
<span id="coderef-xp-dataclass-end" class="coderef-off"><span class="linenr">10: </span>    dataframe: pd.DataFrame   <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Results in a pandas 2d DataFrame</span></span>
<span class="linenr">11: </span>
<span class="linenr">12: </span>
</pre>
</div>

The <code>XP</code> data class comes with the <code>make_xp</code> function (see, lst.
<a href="#orgeb23e3f">7</a>). It produces an <code>XP</code> object from an experiment file path
(i.e., Rally JSON file). Especially, it uses the python <a href="http://objectpath.org/"><code>objectpath</code></a>
module that provides a DSL to query JSON documents (à la XPath) and
extract only interesting data.

<div class="org-src-container">
<label class="org-src-name"><span class="listing-number">Listing 7: </span>Builds an <code>XP</code> object from a Rally JSON Result File.</label><pre class="src src-python" id="orgeb23e3f"><span class="linenr"> 1: </span><span style="color: #F0DFAF; font-weight: bold;">def</span> <span style="color: #93E0E3;">make_xp</span><span style="color: #DCDCCC;">(</span>rally_path: <span style="color: #DCDCCC; font-weight: bold;">str</span><span style="color: #DCDCCC;">)</span> -&gt; XP:
<span class="linenr"> 2: </span>    <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Find XP name in the `rally_path`</span>
<span class="linenr"> 3: </span>    <span style="color: #DFAF8F;">RE_XP</span> = r<span style="color: #CC9393;">'(?:mariadb|galera|cockroachdb)-[a-zA-Z0-9\-]+'</span>
<span class="linenr"> 4: </span>    <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Find XP params in the `rally_path` (e.g., rdbms, number of OS instances, delay, ...)</span>
<span class="linenr"> 5: </span>    <span style="color: #DFAF8F;">RE_XP_PARAMS</span> = r<span style="color: #CC9393;">'(?P&lt;db&gt;[a-z]+)-(?P&lt;oss&gt;[0-9]+)-(?P&lt;delay&gt;[0-9]+)-(?P&lt;zones&gt;[Z0-9]{6})-(?P&lt;high&gt;[TF]).*'</span>
<span class="linenr"> 6: </span>    <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">JSON path to the rally scenario's name</span>
<span class="linenr"> 7: </span>    <span style="color: #DFAF8F;">JPATH_SCN</span> = <span style="color: #CC9393;">'$.tasks[0].subtasks[0].title'</span>
<span class="linenr"> 8: </span>
<span class="linenr"> 9: </span>    <span style="color: #F0DFAF; font-weight: bold;">with</span> <span style="color: #DCDCCC; font-weight: bold;">open</span><span style="color: #DCDCCC;">(</span>rally_path<span style="color: #DCDCCC;">)</span> <span style="color: #F0DFAF; font-weight: bold;">as</span> rally_json:
<span class="linenr">10: </span>        <span style="color: #DFAF8F;">rally_values</span> = objectpath.Tree<span style="color: #DCDCCC;">(</span>json.load<span style="color: #BFEBBF;">(</span>rally_json<span style="color: #BFEBBF;">)</span><span style="color: #DCDCCC;">)</span>
<span class="linenr">11: </span>        <span style="color: #DFAF8F;">xp_info</span> = re.match<span style="color: #DCDCCC;">(</span>RE_XP_PARAMS, re.findall<span style="color: #BFEBBF;">(</span>RE_XP, rally_path<span style="color: #BFEBBF;">)[</span><span style="color: #BFEBBF;">0</span><span style="color: #BFEBBF;">]</span><span style="color: #DCDCCC;">)</span>.groupdict<span style="color: #DCDCCC;">()</span>
<span class="linenr">12: </span>        <span style="color: #DFAF8F;">success</span> = success_rate<span style="color: #DCDCCC;">(</span>rally_values<span style="color: #DCDCCC;">)</span>
<span class="linenr">13: </span>        <span style="color: #F0DFAF; font-weight: bold;">return</span> XP<span style="color: #DCDCCC;">(</span>
<span class="linenr">14: </span>            scenario = rally_values.execute<span style="color: #BFEBBF;">(</span>JPATH_SCN<span style="color: #BFEBBF;">)</span>,
<span class="linenr">15: </span>            filepath = rally_path,
<span class="linenr">16: </span>            rdbms = xp_info.get<span style="color: #BFEBBF;">(</span><span style="color: #CC9393;">'db'</span><span style="color: #BFEBBF;">)</span>,
<span class="linenr">17: </span>            oss = <span style="color: #DCDCCC; font-weight: bold;">int</span><span style="color: #BFEBBF;">(</span>xp_info.get<span style="color: #D0BF8F;">(</span><span style="color: #CC9393;">'oss'</span><span style="color: #D0BF8F;">)</span><span style="color: #BFEBBF;">)</span>,
<span class="linenr">18: </span>            delay = <span style="color: #DCDCCC; font-weight: bold;">int</span><span style="color: #BFEBBF;">(</span>xp_info.get<span style="color: #D0BF8F;">(</span><span style="color: #CC9393;">'delay'</span><span style="color: #D0BF8F;">)</span><span style="color: #BFEBBF;">)</span>,
<span class="linenr">19: </span>            success = success,
<span class="linenr">20: </span>            high = <span style="color: #BFEBBF;">True</span> <span style="color: #F0DFAF; font-weight: bold;">if</span> xp_info.get<span style="color: #BFEBBF;">(</span><span style="color: #CC9393;">'high'</span><span style="color: #BFEBBF;">)</span> <span style="color: #F0DFAF; font-weight: bold;">is</span> <span style="color: #CC9393;">'T'</span> <span style="color: #F0DFAF; font-weight: bold;">else</span> <span style="color: #BFEBBF;">False</span>,
<span id="coderef-dataframe_per_operations" class="coderef-off"><span class="linenr">21: </span>            dataframe = dataframe_per_operations<span style="color: #BFEBBF;">(</span>rally_values<span style="color: #BFEBBF;">)</span><span style="color: #DCDCCC;">)</span> <span style="color: #F0DFAF; font-weight: bold;">if</span> success <span style="color: #F0DFAF; font-weight: bold;">else</span> <span style="color: #BFEBBF;">None</span></span>
</pre>
</div>

The <a href="#coderef-dataframe_per_operations" class="coderef" onmouseover="CodeHighlightOn(this, 'coderef-dataframe_per_operations');" onmouseout="CodeHighlightOff(this, 'coderef-dataframe_per_operations');"><code>dataframe_per_operations</code></a> (see l. <a href="#coderef-dataframe_per_operations" class="coderef" onmouseover="CodeHighlightOn(this, 'coderef-dataframe_per_operations');" onmouseout="CodeHighlightOff(this, 'coderef-dataframe_per_operations');">21</a>) is a function
that transforms Rally JSON results in a pandas <a href="https://pandas.pydata.org/pandas-docs/stable/generated/pandas.DataFrame.html#pandas.DataFrame"><code>DataFrame</code></a> for result analyses.
The next section will says more on this. Right now, focus on <code>make_xp</code>. With
<code>make_xp</code>, transforming all Rally JSONs into <code>XP</code> objects is as simple as
mapping over experiment paths (see lst. <a href="#org5b0bfdd">8</a>).

<div class="org-src-container">
<label class="org-src-name"><span class="listing-number">Listing 8: </span>From Json Files to Python Objects.</label><pre class="src src-python" id="org5b0bfdd"><span style="color: #DFAF8F;">XPS</span> = seq<span style="color: #DCDCCC;">(</span>MARIADB_XP_PATHS + GALERA_XP_PATHS + CRDB_XP_PATHS<span style="color: #DCDCCC;">)</span>.truth_map<span style="color: #DCDCCC;">(</span>make_xp<span style="color: #DCDCCC;">)</span>
</pre>
</div>

This study also comes with a bunch of predicates in its toolbelt that
ease the filtering and sorting of experiments. For instance, a
function `def is_crdb(xp: XP) -> bool` only keeps CockroachDB experiments. Likewise,
`def xp_csize_rtt_b_scn_order(xp: XP) -> str` returns a comparable value to sort experiments. The
complete list of predicates is available in the source of this study.


<a id="orgbc19c8b"></a>

## Query Rally results

The Rally JSON file contains values that give the scenario completion
time per keystone operations at a certain Rally run. These values must
be analyzed to evaluate which backend best suits an OpenStack for the
edge. And a good python module for data analysis is <a href="https://pandas.pydata.org/">Pandas</a>. Thus, the
function <code>dataframe_per_operations</code> (see lst.
<a href="#org4fb4799">9</a> &#x2013; part of <a href="#orgeb23e3f"><code>make_xp</code></a>) takes the Rally
JSON and returns a Pandas <a href="https://pandas.pydata.org/pandas-docs/stable/generated/pandas.DataFrame.html#pandas.DataFrame"><code>DataFrame</code></a>.

<div class="org-src-container">
<label class="org-src-name"><span class="listing-number">Listing 9: </span>Transform Rally Results into Pandas DataFrame.</label><pre class="src src-python" id="org4fb4799"><span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Json path to the completion time series</span>
<span style="color: #DFAF8F;">JPATH_SERIES</span> = <span style="color: #CC9393;">'$.tasks[0].subtasks[0].workloads[0].data[len(@.error) is 0].atomic_actions'</span>
<span style="color: #F0DFAF; font-weight: bold;">def</span> <span style="color: #93E0E3;">dataframe_per_operations</span><span style="color: #DCDCCC;">(</span>rally_values: objectpath.Tree<span style="color: #DCDCCC;">)</span> -&gt; pd.DataFrame:
    <span style="color: #9FC59F;">"Makes a 2d pd.DataFrame of completion time per keystone operations."</span>
    <span style="color: #DFAF8F;">df</span> = pd.DataFrame.from_items<span style="color: #DCDCCC;">(</span>
        items=<span style="color: #BFEBBF;">(</span>seq<span style="color: #D0BF8F;">(</span>rally_values.execute<span style="color: #93E0E3;">(</span>JPATH_SERIES<span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
               .flatten<span style="color: #D0BF8F;">()</span>
               .group_by<span style="color: #D0BF8F;">(</span>itemgetter<span style="color: #93E0E3;">(</span><span style="color: #CC9393;">'name'</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
               .on_value_domap<span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> it: it<span style="color: #93E0E3;">[</span><span style="color: #CC9393;">'finished_at'</span><span style="color: #93E0E3;">]</span> - it<span style="color: #93E0E3;">[</span><span style="color: #CC9393;">'started_at'</span><span style="color: #93E0E3;">]</span><span style="color: #D0BF8F;">)</span><span style="color: #BFEBBF;">)</span><span style="color: #DCDCCC;">)</span>
    <span style="color: #F0DFAF; font-weight: bold;">return</span> df
</pre>
</div>

The DataFrame is a table that lists all the completion times in
seconds for a specific Rally scenario. A column references a Keystone
operations and row labels (index) references the Rally run.
Listing&nbsp;<a href="#org4677be9">10</a> is an example of the DataFrame for
the [&ldquo;Create and List Tenants&rdquo;](#keystonecreate-and-list-tenants) Rally scenario with <code>9</code> nodes in the
CockroachDB cluster and a <code>LAN</code> delay between each node. The
`lambda` line
<a href="#coderef-crdb_cltenants_lambda" class="coderef" onmouseover="CodeHighlightOn(this, 'coderef-crdb_cltenants_lambda');" onmouseout="CodeHighlightOff(this, 'coderef-crdb_cltenants_lambda');">15</a> takes the DataFrame and transforms it to add a
&ldquo;Total&rdquo; column. Table <a href="#org4cdfd8e">1</a> presents the output of this
DataFrame.

<div class="org-src-container">
<label class="org-src-name"><span class="listing-number">Listing 10: </span>Access to the DataFrame of Rally <code>create_and_list_tenants</code>.</label><pre class="src src-python" id="org4677be9"><span class="linenr"> 1: </span><span style="color: #DFAF8F;">CRDB_CLTENANTS</span> = <span style="color: #DCDCCC;">(</span>XPS
<span class="linenr"> 2: </span>    <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Keep xps for Keystone.create_and_list_tenants Rally scenario</span>
<span class="linenr"> 3: </span>    .<span style="color: #DCDCCC; font-weight: bold;">filter</span><span style="color: #BFEBBF;">(</span>is_keystone_scn<span style="color: #D0BF8F;">(</span><span style="color: #CC9393;">'create_and_list_tenants'</span><span style="color: #D0BF8F;">)</span><span style="color: #BFEBBF;">)</span>
<span class="linenr"> 4: </span>    <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Keep xps for 9 OpenStack instances</span>
<span class="linenr"> 5: </span>    .<span style="color: #DCDCCC; font-weight: bold;">filter</span><span style="color: #BFEBBF;">(</span>when_oss<span style="color: #D0BF8F;">(</span><span style="color: #BFEBBF;">9</span><span style="color: #D0BF8F;">)</span><span style="color: #BFEBBF;">)</span>
<span class="linenr"> 6: </span>    <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Keep xps for CockroachDB backend</span>
<span class="linenr"> 7: </span>    .<span style="color: #DCDCCC; font-weight: bold;">filter</span><span style="color: #BFEBBF;">(</span>is_crdb<span style="color: #BFEBBF;">)</span>
<span class="linenr"> 8: </span>    <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Keep xps for LAN delay</span>
<span class="linenr"> 9: </span>    .<span style="color: #DCDCCC; font-weight: bold;">filter</span><span style="color: #BFEBBF;">(</span>when_delay<span style="color: #D0BF8F;">(</span><span style="color: #BFEBBF;">0</span><span style="color: #D0BF8F;">)</span><span style="color: #BFEBBF;">)</span>
<span class="linenr">10: </span>    <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Keep xps for light load mode</span>
<span class="linenr">11: </span>    .<span style="color: #DCDCCC; font-weight: bold;">filter</span><span style="color: #BFEBBF;">(</span>compose<span style="color: #D0BF8F;">(</span>not_, is_high<span style="color: #D0BF8F;">)</span><span style="color: #BFEBBF;">)</span>
<span class="linenr">12: </span>    <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Get dataframe in xp</span>
<span class="linenr">13: </span>    .<span style="color: #DCDCCC; font-weight: bold;">map</span><span style="color: #BFEBBF;">(</span>attrgetter<span style="color: #D0BF8F;">(</span><span style="color: #CC9393;">'dataframe'</span><span style="color: #D0BF8F;">)</span><span style="color: #BFEBBF;">)</span>
<span class="linenr">14: </span>    <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Add a Total column</span>
<span id="coderef-crdb_cltenants_lambda" class="coderef-off"><span class="linenr">15: </span>    .<span style="color: #DCDCCC; font-weight: bold;">map</span><span style="color: #BFEBBF;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> df: df.assign<span style="color: #D0BF8F;">(</span>Total=df.<span style="color: #DCDCCC; font-weight: bold;">sum</span><span style="color: #93E0E3;">(</span>axis=<span style="color: #BFEBBF;">1</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span><span style="color: #BFEBBF;">)</span></span>
<span class="linenr">16: </span>    .head<span style="color: #BFEBBF;">()</span><span style="color: #DCDCCC;">)</span>
</pre>
</div>

<table id="org4cdfd8e">
<caption class="t-above"><span class="table-number">Table 1:</span> Entries for Rally <code>create_and_list_tenants</code>, 9 CRDB nodes, LAN delay.</caption>

<colgroup>
<col  class="org-right">

<col  class="org-right">

<col  class="org-right">

<col  class="org-right">
</colgroup>
<thead>
<tr>
<th scope="col" class="org-right">Iter</th>
<th scope="col" class="org-right"><code>keystone_v3.create_project</code></th>
<th scope="col" class="org-right"><code>keystone_v3.list_projects</code></th>
<th scope="col" class="org-right">Total</th>
</tr>
</thead>

<tbody>
<tr>
<td class="org-right">0</td>
<td class="org-right">0.134</td>
<td class="org-right">0.023</td>
<td class="org-right">0.157</td>
</tr>


<tr>
<td class="org-right">1</td>
<td class="org-right">0.127</td>
<td class="org-right">0.025</td>
<td class="org-right">0.152</td>
</tr>


<tr>
<td class="org-right">2</td>
<td class="org-right">0.129</td>
<td class="org-right">0.024</td>
<td class="org-right">0.153</td>
</tr>


<tr>
<td class="org-right">3</td>
<td class="org-right">0.134</td>
<td class="org-right">0.023</td>
<td class="org-right">0.157</td>
</tr>


<tr>
<td class="org-right">4</td>
<td class="org-right">0.132</td>
<td class="org-right">0.024</td>
<td class="org-right">0.156</td>
</tr>


<tr>
<td class="org-right">5</td>
<td class="org-right">0.132</td>
<td class="org-right">0.025</td>
<td class="org-right">0.157</td>
</tr>


<tr>
<td class="org-right">6</td>
<td class="org-right">0.126</td>
<td class="org-right">0.024</td>
<td class="org-right">0.150</td>
</tr>


<tr>
<td class="org-right">7</td>
<td class="org-right">0.126</td>
<td class="org-right">0.026</td>
<td class="org-right">0.153</td>
</tr>


<tr>
<td class="org-right">8</td>
<td class="org-right">0.131</td>
<td class="org-right">0.025</td>
<td class="org-right">0.156</td>
</tr>


<tr>
<td class="org-right">9</td>
<td class="org-right">0.130</td>
<td class="org-right">0.025</td>
<td class="org-right">0.155</td>
</tr>
</tbody>
</table>

A pandas DataFrame presents the benefits of efficiently computing a
wide range of analyses. As an example, the listing
<a href="#org6e9fc8a">11</a> computes the number of Rally runs (i.e.,
**count**), mean and standard deviation (i.e., **mean**, **std**), the
fastest and longest completion time (i.e., **min**, **max**), and the
25th, 50th and 75th percentiles (i.e., **25%**, **50%**, **75%**). The
<code>transpose</code> method transposes row labels (index) and columns. Table
<a href="#orgca24dbb">2</a> presents the output of the analysis.

<div class="org-src-container">
<label class="org-src-name"><span class="listing-number">Listing 11: </span>Analyse the DataFrame of Rally <code>create_and_list_tenants</code>.</label><pre class="src src-python" id="org6e9fc8a"><span style="color: #DFAF8F;">CRDB_CLTENANTS_ANALYSIS</span> = CRDB_CLTENANTS.describe<span style="color: #DCDCCC;">()</span>.transpose<span style="color: #DCDCCC;">()</span>
</pre>
</div>

<table id="orgca24dbb">
<caption class="t-above"><span class="table-number">Table 2:</span> Analyses of Rally <code>create_and_list_tenants</code>, 9 CRDB nodes, LAN delay.</caption>

<colgroup>
<col  class="org-left">

<col  class="org-right">

<col  class="org-right">

<col  class="org-right">

<col  class="org-right">

<col  class="org-right">

<col  class="org-right">

<col  class="org-right">

<col  class="org-right">
</colgroup>
<thead>
<tr>
<th scope="col" class="org-left">Operations</th>
<th scope="col" class="org-right">count</th>
<th scope="col" class="org-right">mean</th>
<th scope="col" class="org-right">std</th>
<th scope="col" class="org-right">min</th>
<th scope="col" class="org-right">25%</th>
<th scope="col" class="org-right">50%</th>
<th scope="col" class="org-right">75%</th>
<th scope="col" class="org-right">max</th>
</tr>
</thead>

<tbody>
<tr>
<td class="org-left"><code>keystone_v3.create_project</code></td>
<td class="org-right">10.000</td>
<td class="org-right">0.130</td>
<td class="org-right">0.003</td>
<td class="org-right">0.126</td>
<td class="org-right">0.128</td>
<td class="org-right">0.131</td>
<td class="org-right">0.132</td>
<td class="org-right">0.134</td>
</tr>


<tr>
<td class="org-left"><code>keystone_v3.list_projects</code></td>
<td class="org-right">10.000</td>
<td class="org-right">0.024</td>
<td class="org-right">0.001</td>
<td class="org-right">0.023</td>
<td class="org-right">0.024</td>
<td class="org-right">0.024</td>
<td class="org-right">0.025</td>
<td class="org-right">0.026</td>
</tr>


<tr>
<td class="org-left">Total</td>
<td class="org-right">10.000</td>
<td class="org-right">0.155</td>
<td class="org-right">0.003</td>
<td class="org-right">0.150</td>
<td class="org-right">0.153</td>
<td class="org-right">0.155</td>
<td class="org-right">0.157</td>
<td class="org-right">0.157</td>
</tr>
</tbody>
</table>


<a id="orgc332936"></a>

# Experiment Analysis

This section presents the results of experiments and their analysis.
To avoid lengths of graphics in this report, only a short version of
the results are presented. The entirety of the results, that is,
median time for each Rally scenarios and cumulative distribution
functions for each scenarios, are located in the appendix (see, [Detailed experiments results](#detailed-experiments-results)).

The short results focus on two Rally scenarios. First, *[authenticate
and validate a keystone token (%reads: 96.46, %writes: 3.54)](#keystoneauthenticate-user-and-validate-token)*, which
represents more than 95% of what is actually done on a running
Keystone. Second, *[create user and update password for that user
(%reads: 89.79, %writes: 10.21)](#keystonecreate-user-update-password)*, which gets the highest rate of
writes and thus, the most likely to produce contention on the RDBMS.

In next result plots, the *λ* Greek letter stands for the failure rate
and *σ* for the standard deviation.


<a id="orgb56efe0"></a>

## Number of OpenStack instances impact

This test evaluates how the completion time of Rally Keystone&rsquo;s
scenarios varies, depending on the RDBMS and the number of OpenStack
instances. It measure the capacity of a RDBMS to handle lot of
connections and requests. In this test, the number of OpenStack
instances varies between <code>3</code>, <code>9</code> and <code>45</code> and a <code>LAN</code> link
inter-connects instances. As explain in the [OpenStack at the Edge](#openstack-at-the-edge-the-keystone-use-case)
section, the deployment of the database depends on the RDBMS. With
MariaDB, one instance of OpenStack contains the database, and others
connect to that one. For Galera and CockroachDB, every OpenStack
contains an instance of the RDBMS.

For these experiments, Juice deployed database together with OpenStack
instances and plays Rally scenarios listed in section [List of Rally
scenarios](#list-of-rally-scenarios). Juice runs Rally scenarios under both light and high load.
Results are presented in the two next subsections. The Juice
implementation for these experiments is available on GitHub at
<a href="https://github.com/BeyondTheClouds/juice/blob/8dc04c7fbd371f441f76b3ff73a9a55530b172e4/experiments/cluster-size-impact.py">experiments/cluster-size-impact.py</a>.


### Authenticate and validate a Keystone token (%r: 96.46, %w: 3.54)

Figure <a href="#orga812713">7</a> plots the mean completion time (in
second) of Keystone *authenticate and validate a keystone token (%r:
96.46, %w: 3.54)* scenario in a light Rally mode. The plot displays
results in three tiles. The first tile shows completion time with the
centralized MariaDB, second tile with the replicated Galera and, third
tile with the global CockroachDB. A tile presents results with stacked
bar charts. A bar depicts the total completion time for a specific
number of OpenStack instances (*i.e.*, <code>3</code>, <code>9</code> and <code>45</code>) and stacks
completion times of each Keystone operations. The figure shows that
the trend is similar for the three RDBMS, to the advantage of the
centralized MariaDB, followed by Galera and then CockroachDB.


<figure id="orga812713">
<img src='{{ "assets/evaluation-of-openstack-multi-region-keystone-deployments/oss-impact-auth-light.png" | absolute_url }}' alt="oss-impact-auth-light.png" class="zoom-on-click">

<figcaption><span class="figure-number">Figure 7: </span>Impact of the Number of OpenStack Instances on the Completion Time under Light Load for scenario Authenticate and Validate a Keystone Token (%r: 96.46, %w: 3.54) &#x2013; Mean Time for Every Operations (Lower is Better).</figcaption>
</figure>


Figure <a href="#org00cf6ca">8</a> shows that putting pressure using the
high load has strong effects on MariaDB and Galera while CockroachDB
well handles it. With 45 OpenStack instances, the failure rate of
MariaDB rises from 0 to 98 percent (*i.e.*, λ: 0.98) and, the mean
completion time of Galera rises from 120 to 475 milliseconds (namely,
an increased by a factor of 4).


<figure id="org00cf6ca">
<img src='{{ "assets/evaluation-of-openstack-multi-region-keystone-deployments/oss-impact-auth-high.png" | absolute_url }}' alt="oss-impact-auth-high.png" class="zoom-on-click">

<figcaption><span class="figure-number">Figure 8: </span>Impact of the Number of OpenStack Instances on the Completion Time under High Load for scenario Authenticate and Validate a Keystone Token (%r: 96.46, %w: 3.54) &#x2013; Mean Time for Every Operations (Lower is Better).</figcaption>
</figure>

TODO:explenation (look at MariaDB log).

TODO:explenation (look at Galera log). Plotting the cumulative
distribution function (*i.e.*, the probability that the scenario
complete in less or equal than a certain time &#x2013; see,
fig. <a href="#orgf7e0d9b">9</a>) shows that, with 45 OpenStack
instances, more than 95% of the results should complete in a
reasonable time, but the lasts 5% may take a really long time to
complete (here, up to 30 second).


<figure id="orgf7e0d9b">
<img src='{{ "assets/evaluation-of-openstack-multi-region-keystone-deployments/oss-impact-auth-high-cdf.png" | absolute_url }}' alt="oss-impact-auth-high-cdf.png" class="zoom-on-click">

<figcaption><span class="figure-number">Figure 9: </span>Impact of the Number of OpenStack Instances on the Completion Time under High Load for scenario Authenticate and Validate a Keystone Token (%r: 96.46, %w: 3.54) &#x2013; Cumulative Distribution.</figcaption>
</figure>


### Create a user and update her password (%r: 89.79, %w: 10.21)

Figure <a href="#orga812713">7</a> plots the mean completion time (in
second) of Keystone *Create a user and update her password (%r: 89.79,
%w: 10.21)* scenario in a light Rally mode.


<figure id="org36a9977">
<img src='{{ "assets/evaluation-of-openstack-multi-region-keystone-deployments/oss-impact-pwd-light.png" | absolute_url }}' alt="oss-impact-pwd-light.png" class="zoom-on-click">

<figcaption><span class="figure-number">Figure 10: </span>Impact of the Number of OpenStack Instances on the Completion Time under Light Load for scenario Create a User and Update Her Password (%r: 89.79, %w: 10.21) &#x2013; Mean Time for Every Operations (Lower is Better).</figcaption>
</figure>

High mode


<figure id="org3f4bc70">
<img src='{{ "assets/evaluation-of-openstack-multi-region-keystone-deployments/oss-impact-pwd-high.png" | absolute_url }}' alt="oss-impact-pwd-high.png" class="zoom-on-click">

<figcaption><span class="figure-number">Figure 11: </span>Impact of the Number of OpenStack Instances on the Completion Time under High Load for scenario Create a User and Update Her Password (%r: 89.79, %w: 10.21) &#x2013; Mean Time for Every Operations (Lower is Better).</figcaption>
</figure>

CDF


<figure id="org11ede40">
<img src='{{ "assets/evaluation-of-openstack-multi-region-keystone-deployments/oss-impact-pwd-high-cdf.png" | absolute_url }}' alt="oss-impact-pwd-high-cdf.png" class="zoom-on-click">

<figcaption><span class="figure-number">Figure 12: </span>Impact of the Number of OpenStack Instances on the Completion Time under High Load for scenario Create a User and Update Her Password (%r: 89.79, %w: 10.21) &#x2013; Cumulative Distribution.</figcaption>
</figure>


### Scale outline

TODO:

<table id="org0154195">
<caption class="t-above"><span class="table-number">Table 3:</span> Scale of a RDBMS with the Number OpenStack instances</caption>

<colgroup>
<col  class="org-left">

<col  class="org-left">

<col  class="org-left">
</colgroup>
<thead>
<tr>
<th scope="col" class="org-left">RDBMS</th>
<th scope="col" class="org-left">Scale</th>
<th scope="col" class="org-left">Note</th>
</tr>
</thead>

<tbody>
<tr>
<td class="org-left">MariaDB</td>
<td class="org-left">😺️</td>
<td class="org-left">&#xa0;</td>
</tr>


<tr>
<td class="org-left">Galera</td>
<td class="org-left">😾️</td>
<td class="org-left">&#xa0;</td>
</tr>


<tr>
<td class="org-left">CockroachDB</td>
<td class="org-left">😺</td>
<td class="org-left">&#xa0;</td>
</tr>
</tbody>
</table>


<a id="orge3952f1"></a>

## Delay impact

In this test, the size of the database cluster is 9 and the delay
varies between LAN, 100 and 300 ms of RTT. The test evaluates how the
completion time of Rally scenarios varies, depending of RTT between
nodes of the swarm.

-   TODO: describe the experimentation protocol
-   TODO: Link the github juice code


### Authenticate and validate a Keystone token (%r: 96.46, %w: 3.54)

Auth. Light Mode


<figure id="orgecfe667">
<img src='{{ "assets/evaluation-of-openstack-multi-region-keystone-deployments/delay-impact-auth-light.png" | absolute_url }}' alt="delay-impact-auth-light.png" class="zoom-on-click">

<figcaption><span class="figure-number">Figure 13: </span>Impact of the Network Delay on the Completion Time under Light Load for scenario Authenticate and Validate a Keystone Token (%r: 96.46, %w: 3.54) &#x2013; Mean Time for Every Operations (Lower is Better).</figcaption>
</figure>

High Mode


<figure id="org870432d">
<img src='{{ "assets/evaluation-of-openstack-multi-region-keystone-deployments/delay-impact-auth-high.png" | absolute_url }}' alt="delay-impact-auth-high.png" class="zoom-on-click">

<figcaption><span class="figure-number">Figure 14: </span>Impact of the Network Delay on the Completion Time under High Load for scenario Authenticate and Validate a Keystone Token (%r: 96.46, %w: 3.54) &#x2013; Mean Time for Every Operations (Lower is Better).</figcaption>
</figure>

CDF


<figure id="org91a8ee2">
<img src='{{ "assets/evaluation-of-openstack-multi-region-keystone-deployments/delay-impact-auth-high-cdf.png" | absolute_url }}' alt="delay-impact-auth-high-cdf.png" class="zoom-on-click">

<figcaption><span class="figure-number">Figure 15: </span>Impact of the Network Delay on the Completion Time under High Load for scenario Authenticate and Validate a Keystone Token (%r: 96.46, %w: 3.54) &#x2013; Cumulative Distribution.</figcaption>
</figure>


### Create a user and update her password (%r: 89.79, %w: 10.21)

Create Light Mode


<figure id="org05b2285">
<img src='{{ "assets/evaluation-of-openstack-multi-region-keystone-deployments/delay-impact-pwd-light.png" | absolute_url }}' alt="delay-impact-pwd-light.png" class="zoom-on-click">

<figcaption><span class="figure-number">Figure 16: </span>Impact of the Network Delay on the Completion Time under Light Load for scenario Create a User and Update Her Password (%r: 89.79, %w: 10.21) &#x2013; Mean Time for Every Operations (Lower is Better).</figcaption>
</figure>

High Mode


<figure id="org1e50307">
<img src='{{ "assets/evaluation-of-openstack-multi-region-keystone-deployments/delay-impact-pwd-high.png" | absolute_url }}' alt="delay-impact-pwd-high.png" class="zoom-on-click">

<figcaption><span class="figure-number">Figure 17: </span>Impact of the Network Delay on the Completion Time under High Load for scenario Create a User and Update Her Password (%r: 89.79, %w: 10.21) &#x2013; Mean Time for Every Operations (Lower is Better).</figcaption>
</figure>

CDF


<figure id="orgf0cd4ae">
<img src='{{ "assets/evaluation-of-openstack-multi-region-keystone-deployments/delay-impact-pwd-high-cdf.png" | absolute_url }}' alt="delay-impact-pwd-high-cdf.png" class="zoom-on-click">

<figcaption><span class="figure-number">Figure 18: </span>Impact of the Network Delay on the Completion Time under High Load for scenario Create a User and Update Her Password (%r: 89.79, %w: 10.21) &#x2013; Cumulative Distribution.</figcaption>
</figure>


### Network delay outline

TODO:

<table id="org7264ce1">
<caption class="t-above"><span class="table-number">Table 4:</span> RDBMS Handling of Network Delay</caption>

<colgroup>
<col  class="org-left">

<col  class="org-left">

<col  class="org-left">
</colgroup>
<thead>
<tr>
<th scope="col" class="org-left">RDBMS</th>
<th scope="col" class="org-left">Network Delay</th>
<th scope="col" class="org-left">Note</th>
</tr>
</thead>

<tbody>
<tr>
<td class="org-left">MariaDB</td>
<td class="org-left">😺</td>
<td class="org-left">&#xa0;</td>
</tr>


<tr>
<td class="org-left">Galera</td>
<td class="org-left">😺 reads, 😾 writes</td>
<td class="org-left">&#xa0;</td>
</tr>


<tr>
<td class="org-left">CockroachDB</td>
<td class="org-left">😾</td>
<td class="org-left">&#xa0;</td>
</tr>
</tbody>
</table>


<a id="org9bc2403"></a>

## Taking into account the user locality

A homogeneous delay is sometimes needed but does not map to the edge
reality where some nodes are closed and other are far. To simulate
such heterogeneous network infrastructure &#x2026;

<div class="org-src-container">
<label class="org-src-name"><span class="listing-number">Listing 12: </span>Replication Zones.</label><pre class="src src-python" id="org152ee92"><span style="color: #DFAF8F;">ZONES</span> = <span style="color: #DCDCCC;">[</span> <span style="color: #BFEBBF;">(</span><span style="color: #CC9393;">'Z1'</span>, <span style="color: #CC9393;">'Z2'</span>, <span style="color: #CC9393;">'Z3'</span><span style="color: #BFEBBF;">)</span>, <span style="color: #BFEBBF;">(</span><span style="color: #CC9393;">'Z1'</span>, <span style="color: #CC9393;">'Z1'</span>, <span style="color: #CC9393;">'Z3'</span><span style="color: #BFEBBF;">)</span>,  <span style="color: #BFEBBF;">(</span><span style="color: #CC9393;">'Z1'</span>, <span style="color: #CC9393;">'Z1'</span>, <span style="color: #CC9393;">'Z1'</span><span style="color: #BFEBBF;">)</span> <span style="color: #DCDCCC;">]</span>
</pre>
</div>

<div class="org-src-container">
<label class="org-src-name"><span class="listing-number">Listing 13: </span><code>XP</code> property for Replication Zones</label><pre class="src src-python" id="orgb701898">zones: <span style="color: #DFAF8F;">Tuple</span><span style="color: #DCDCCC;">[</span><span style="color: #DCDCCC; font-weight: bold;">str</span>,<span style="color: #DCDCCC; font-weight: bold;">str</span>,<span style="color: #DCDCCC; font-weight: bold;">str</span><span style="color: #DCDCCC;">]</span> = <span style="color: #DCDCCC;">(</span><span style="color: #CC9393;">'Z1'</span>, <span style="color: #CC9393;">'Z1'</span>, <span style="color: #CC9393;">'Z1'</span><span style="color: #DCDCCC;">)</span> <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Replication Zones</span>
</pre>
</div>

<div class="org-src-container">
<label class="org-src-name"><span class="listing-number">Listing 14: </span>Parsing of replication zones</label><pre class="src src-python" id="orgc8e4756"><span style="color: #DFAF8F;">zones</span> = <span style="color: #DCDCCC; font-weight: bold;">tuple</span><span style="color: #DCDCCC;">(</span><span style="color: #BFEBBF;">[</span>xp_info.get<span style="color: #D0BF8F;">(</span><span style="color: #CC9393;">'zones'</span><span style="color: #D0BF8F;">)[</span>x:x+<span style="color: #BFEBBF;">2</span><span style="color: #D0BF8F;">]</span> <span style="color: #F0DFAF; font-weight: bold;">for</span> x <span style="color: #F0DFAF; font-weight: bold;">in</span> <span style="color: #DCDCCC; font-weight: bold;">range</span><span style="color: #D0BF8F;">(</span><span style="color: #BFEBBF;">0</span>, <span style="color: #DCDCCC; font-weight: bold;">len</span><span style="color: #93E0E3;">(</span>xp_info.get<span style="color: #9FC59F;">(</span><span style="color: #CC9393;">'zones'</span><span style="color: #9FC59F;">)</span><span style="color: #93E0E3;">)</span>, <span style="color: #BFEBBF;">2</span><span style="color: #D0BF8F;">)</span><span style="color: #BFEBBF;">]</span><span style="color: #DCDCCC;">)</span>,
</pre>
</div>

<div class="org-src-container">
<pre class="src src-python"><span style="color: #DFAF8F;">zones_plot</span> = frame_plot<span style="color: #DCDCCC;">(</span>ZONES<span style="color: #DCDCCC;">)</span>
</pre>
</div>


### Delay distribution: uniform & hierarchical

This study considers two kinds of OpenStack instances deployments.
This first one, called *uniform*, defines a uniform distribution of
the network latency between OpenStack instances. For instance, <code>300</code>
ms of RTT between all the <code>9</code> OpenStack instances. The second
deployment, called *hierarchical*, maps to a more realistic view, like
in cloud computing, with groups of OpenStack instances connected
through a low latency network (*e.g.*, <code>3</code> OpenStack instances per
group deployed in the same country, and accessible within <code>20</code> ms of
RTT). And high latency network between groups (*e.g.* <code>150</code> ms of RTT
between groups deployed in different countries).

<div class="org-src-container">
<pre class="src src-python"><span style="color: #DFAF8F;">XPS_ZONES</span> = <span style="color: #DCDCCC;">(</span>XPS
    <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">We are only interested in results with 9 OpenStack instances</span>
    .<span style="color: #DCDCCC; font-weight: bold;">filter</span><span style="color: #BFEBBF;">(</span>when_oss<span style="color: #D0BF8F;">(</span><span style="color: #BFEBBF;">9</span><span style="color: #D0BF8F;">)</span><span style="color: #BFEBBF;">)</span>
    .<span style="color: #DCDCCC; font-weight: bold;">filter</span><span style="color: #BFEBBF;">(</span>when_delay<span style="color: #D0BF8F;">(</span><span style="color: #BFEBBF;">10</span><span style="color: #D0BF8F;">)</span><span style="color: #BFEBBF;">)</span>
    .<span style="color: #DCDCCC; font-weight: bold;">filter</span><span style="color: #BFEBBF;">(</span>compose<span style="color: #D0BF8F;">(</span>not_, is_high<span style="color: #D0BF8F;">)</span><span style="color: #BFEBBF;">)</span>
    <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">.filter(is_keystone_scn('get_entities'))</span>
    <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">.filter(is_crdb)</span>
    <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Also, remove values greater than the 95th percentile</span>
    .<span style="color: #DCDCCC; font-weight: bold;">map</span><span style="color: #BFEBBF;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> xp: xp.set_dataframe<span style="color: #D0BF8F;">(</span>filter_percentile<span style="color: #93E0E3;">(</span>.<span style="color: #BFEBBF;">95</span>, xp.dataframe<span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span><span style="color: #BFEBBF;">)</span>
<span style="color: #DCDCCC;">)</span>
</pre>
</div>

CDF


### Locality outline

<table id="org3dfb2c5">
<caption class="t-above"><span class="table-number">Table 5:</span> RDBMS Handling of Network Delay with Locality</caption>

<colgroup>
<col  class="org-left">

<col  class="org-left">

<col  class="org-left">
</colgroup>
<thead>
<tr>
<th scope="col" class="org-left">RDBMS</th>
<th scope="col" class="org-left">Network Delay w/ Locality</th>
<th scope="col" class="org-left">Note</th>
</tr>
</thead>

<tbody>
<tr>
<td class="org-left">MariaDB</td>
<td class="org-left">😺</td>
<td class="org-left">&#xa0;</td>
</tr>


<tr>
<td class="org-left">Galera</td>
<td class="org-left">😺 reads, 😾 writes</td>
<td class="org-left">&#xa0;</td>
</tr>


<tr>
<td class="org-left">CockroachDB</td>
<td class="org-left">😺</td>
<td class="org-left">&#xa0;</td>
</tr>
</tbody>
</table>


<a id="org0e0ed76"></a>

# Experiments Outline

TODO:

<table id="orge1cb1b7">
<caption class="t-above"><span class="table-number">Table 6:</span> RDBMS for Large Geo-Distributed OpenStacks</caption>

<colgroup>
<col  class="org-left">

<col  class="org-left">

<col  class="org-left">

<col  class="org-left">
</colgroup>
<thead>
<tr>
<th scope="col" class="org-left">RDBMS</th>
<th scope="col" class="org-left">Scale</th>
<th scope="col" class="org-left">Network Delay</th>
<th scope="col" class="org-left">Note</th>
</tr>
</thead>

<tbody>
<tr>
<td class="org-left">MariaDB</td>
<td class="org-left">😺</td>
<td class="org-left">😺</td>
<td class="org-left">&#xa0;</td>
</tr>


<tr>
<td class="org-left">Galera</td>
<td class="org-left">😾</td>
<td class="org-left">😺 reads, 😾 writes</td>
<td class="org-left">&#xa0;</td>
</tr>


<tr>
<td class="org-left">CockroachDB</td>
<td class="org-left">😺</td>
<td class="org-left">😺 with locality, 😾 otherwise</td>
<td class="org-left">&#xa0;</td>
</tr>
</tbody>
</table>

TODO: General purpose support of locality


<a id="orgb58c24d"></a>

# Appendix


<a id="orgacf0611"></a>

## Detailed experiments results

Listing <a href="#org558517a">15</a> computes the mean completion time (in
second) of Rally scenarios in a light mode and plots the results in
figure <a href="#org3b3868e">19</a>. In the following figure, columns presents
results of a specific scenario: the first column presents results for
Authenticate User and Validate Token, the second for Create Add and
List User Role. Rows present results with a specific RDBMS: first row
presents results for MariaDB, second for Galera and third for
CockroachDB. The figure presents results with stacked bar charts. A
bar presents the result for a specific number of OpenStack instances
(*i.e.*, <code>3</code>, <code>9</code> and <code>45</code>) and stacks completion times of each
Keystone operations.


### Number of OpenStack instances impact under light load

<div class="org-src-container">
<pre class="src src-python" id="org558517a">oss_plot<span style="color: #DCDCCC;">(</span><span style="color: #CC9393;">"%s Completion Time (s)"</span>,
         series_stackedbar_plot,
         <span style="color: #CC9393;">'imgs/oss-impact-light'</span>,
         <span style="color: #BFEBBF;">(</span><span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">-- Experiments selection</span>
          XPS
          <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">We are only interested in results where delay is LAN</span>
          .<span style="color: #DCDCCC; font-weight: bold;">filter</span><span style="color: #D0BF8F;">(</span>when_delay<span style="color: #93E0E3;">(</span><span style="color: #BFEBBF;">0</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
          <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">remove values greater than the 95th percentile</span>
          .<span style="color: #DCDCCC; font-weight: bold;">map</span><span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> xp: xp.set_dataframe<span style="color: #93E0E3;">(</span>filter_percentile<span style="color: #9FC59F;">(</span>.<span style="color: #BFEBBF;">95</span>, xp.dataframe<span style="color: #9FC59F;">)</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
          <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Filter light load</span>
          .<span style="color: #DCDCCC; font-weight: bold;">filter</span><span style="color: #D0BF8F;">(</span>compose<span style="color: #93E0E3;">(</span>not_, is_high<span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
          <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Group results by scenario's name, RDBMS technology and</span>
          <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">number of OpenStack instances.</span>
           .group_by<span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> xp: <span style="color: #93E0E3;">(</span>xp.scenario, xp.rdbms, xp.oss<span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
          <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Compute the mean, std and success of the results</span>
          .on_value_domap<span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> xp: <span style="color: #93E0E3;">(</span>xp.dataframe, xp.success<span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
          .on_value<span style="color: #D0BF8F;">(</span>unpack<span style="color: #93E0E3;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> dfs, succs: <span style="color: #9FC59F;">(</span>pd.concat<span style="color: #94BFF3;">(</span>dfs<span style="color: #94BFF3;">)</span>, np.mean<span style="color: #94BFF3;">(</span>succs<span style="color: #94BFF3;">)</span><span style="color: #9FC59F;">)</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
          .on_value<span style="color: #D0BF8F;">(</span>unpack<span style="color: #93E0E3;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> df, succ: <span style="color: #9FC59F;">(</span>df.mean<span style="color: #94BFF3;">()</span>, df.<span style="color: #DCDCCC; font-weight: bold;">sum</span><span style="color: #94BFF3;">(</span>axis=<span style="color: #BFEBBF;">1</span><span style="color: #94BFF3;">)</span>.std<span style="color: #94BFF3;">()</span>, succ<span style="color: #9FC59F;">)</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
          .to_dict<span style="color: #D0BF8F;">()</span><span style="color: #BFEBBF;">)</span><span style="color: #DCDCCC;">)</span>
</pre>
</div>


<figure id="org3b3868e">
<img src='{{ "assets/evaluation-of-openstack-multi-region-keystone-deployments/oss-impact-light.png" | absolute_url }}' alt="oss-impact-light.png" class="zoom-on-click">

<figcaption><span class="figure-number">Figure 19: </span>Impact of the Number of OpenStack Instances on the Completion Time under Light Load &#x2013; Mean Time for Every Operations (Lower is Better).</figcaption>
</figure>

<div class="org-src-container">
<pre class="src src-python" id="org49e84bc">oss_plot<span style="color: #DCDCCC;">(</span><span style="color: #CC9393;">"%s Cumulative Percent"</span>,
         series_linear_plot,
         <span style="color: #CC9393;">'imgs/oss-impact-light-cdf'</span>,
         <span style="color: #BFEBBF;">(</span><span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">-- Experiments selection</span>
          XPS
          <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">We are only interested in results where delay is LAN</span>
          .<span style="color: #DCDCCC; font-weight: bold;">filter</span><span style="color: #D0BF8F;">(</span>when_delay<span style="color: #93E0E3;">(</span><span style="color: #BFEBBF;">0</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
          <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">remove values greater than the 95th percentile</span>
          .<span style="color: #DCDCCC; font-weight: bold;">map</span><span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> xp: xp.set_dataframe<span style="color: #93E0E3;">(</span>filter_percentile<span style="color: #9FC59F;">(</span>.<span style="color: #BFEBBF;">95</span>, xp.dataframe<span style="color: #9FC59F;">)</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
          <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Filter light load</span>
          .<span style="color: #DCDCCC; font-weight: bold;">filter</span><span style="color: #D0BF8F;">(</span>compose<span style="color: #93E0E3;">(</span>not_, is_high<span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
          <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Group results by scenario's name, RDBMS technology and</span>
          <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">number of OpenStack instances.</span>
          .group_by<span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> xp: <span style="color: #93E0E3;">(</span>xp.scenario, xp.rdbms, xp.oss<span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
          <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Compute the CDF of the resuls</span>
          .on_value_domap<span style="color: #D0BF8F;">(</span>attrgetter<span style="color: #93E0E3;">(</span><span style="color: #CC9393;">'dataframe'</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
          .on_value<span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> dfs: pd.concat<span style="color: #93E0E3;">(</span>dfs<span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
          .on_value<span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> df: df.<span style="color: #DCDCCC; font-weight: bold;">sum</span><span style="color: #93E0E3;">(</span>axis=<span style="color: #CC9393;">'columns'</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
          .on_value<span style="color: #D0BF8F;">(</span>make_cumulative_frequency<span style="color: #D0BF8F;">)</span>
          .to_dict<span style="color: #D0BF8F;">()</span><span style="color: #BFEBBF;">)</span>,
         legend=<span style="color: #CC9393;">'all'</span><span style="color: #DCDCCC;">)</span>
</pre>
</div>


<figure id="orgfacab6b">
<img src='{{ "assets/evaluation-of-openstack-multi-region-keystone-deployments/oss-impact-light-cdf.png" | absolute_url }}' alt="oss-impact-light-cdf.png" class="zoom-on-click">

<figcaption><span class="figure-number">Figure 20: </span>Impact of the Number of OpenStack Instances on the Completion Time under Light Load &#x2013; Cumulative Distribution.</figcaption>
</figure>


### Number of OpenStack instances impact under high load

<div class="org-src-container">
<pre class="src src-python" id="org7a78b35">oss_plot<span style="color: #DCDCCC;">(</span><span style="color: #CC9393;">"%s Completion Time (s)"</span>,
         series_stackedbar_plot,
         <span style="color: #CC9393;">'imgs/oss-impact-high'</span>,
         <span style="color: #BFEBBF;">(</span><span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">-- Experiments selection</span>
          XPS
          <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">We are only interested in results where delay is LAN</span>
          .<span style="color: #DCDCCC; font-weight: bold;">filter</span><span style="color: #D0BF8F;">(</span>when_delay<span style="color: #93E0E3;">(</span><span style="color: #BFEBBF;">0</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
          <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">remove values greater than the 95th percentile</span>
          .<span style="color: #DCDCCC; font-weight: bold;">map</span><span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> xp: xp.set_dataframe<span style="color: #93E0E3;">(</span>filter_percentile<span style="color: #9FC59F;">(</span>.<span style="color: #BFEBBF;">95</span>, xp.dataframe<span style="color: #9FC59F;">)</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
          <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Filter high load</span>
          .<span style="color: #DCDCCC; font-weight: bold;">filter</span><span style="color: #D0BF8F;">(</span>is_high<span style="color: #D0BF8F;">)</span>
          <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Group results by scenario's name, RDBMS technology and</span>
          <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">number of OpenStack instances.</span>
          .group_by<span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> xp: <span style="color: #93E0E3;">(</span>xp.scenario, xp.rdbms, xp.oss<span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
          <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Compute the mean, std and success of the results</span>
          .on_value_domap<span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> xp: <span style="color: #93E0E3;">(</span>xp.dataframe, xp.success<span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
          .on_value<span style="color: #D0BF8F;">(</span>unpack<span style="color: #93E0E3;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> dfs, succs: <span style="color: #9FC59F;">(</span>pd.concat<span style="color: #94BFF3;">(</span>dfs<span style="color: #94BFF3;">)</span>, np.mean<span style="color: #94BFF3;">(</span>succs<span style="color: #94BFF3;">)</span><span style="color: #9FC59F;">)</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
          .on_value<span style="color: #D0BF8F;">(</span>unpack<span style="color: #93E0E3;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> df, succ: <span style="color: #9FC59F;">(</span>df.mean<span style="color: #94BFF3;">()</span>, df.<span style="color: #DCDCCC; font-weight: bold;">sum</span><span style="color: #94BFF3;">(</span>axis=<span style="color: #BFEBBF;">1</span><span style="color: #94BFF3;">)</span>.std<span style="color: #94BFF3;">()</span>, succ<span style="color: #9FC59F;">)</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
          .to_dict<span style="color: #D0BF8F;">()</span><span style="color: #BFEBBF;">)</span><span style="color: #DCDCCC;">)</span>
</pre>
</div>


<figure id="org2801e77">
<img src='{{ "assets/evaluation-of-openstack-multi-region-keystone-deployments/oss-impact-high.png" | absolute_url }}' alt="oss-impact-high.png" class="zoom-on-click">

<figcaption><span class="figure-number">Figure 21: </span>Impact of the Number of OpenStack Instances on the Completion Time under High Load &#x2013; Median Time for Every Operations (Lower is Better).</figcaption>
</figure>

<div class="org-src-container">
<pre class="src src-python" id="org67e26a9">oss_plot<span style="color: #DCDCCC;">(</span><span style="color: #CC9393;">"%s Cumulative Percent"</span>,
         series_linear_plot,
         <span style="color: #CC9393;">'imgs/oss-impact-high-cdf'</span>,
         <span style="color: #BFEBBF;">(</span><span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">-- Experiments selection</span>
          XPS
          <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">We are only interested in results where delay is LAN</span>
          .<span style="color: #DCDCCC; font-weight: bold;">filter</span><span style="color: #D0BF8F;">(</span>when_delay<span style="color: #93E0E3;">(</span><span style="color: #BFEBBF;">0</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
          <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">remove values greater than the 95th percentile</span>
          .<span style="color: #DCDCCC; font-weight: bold;">map</span><span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> xp: xp.set_dataframe<span style="color: #93E0E3;">(</span>filter_percentile<span style="color: #9FC59F;">(</span>.<span style="color: #BFEBBF;">95</span>, xp.dataframe<span style="color: #9FC59F;">)</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
          <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Filter high load</span>
          .<span style="color: #DCDCCC; font-weight: bold;">filter</span><span style="color: #D0BF8F;">(</span>is_high<span style="color: #D0BF8F;">)</span>
          <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Group results by scenario's name, RDBMS technology and</span>
          <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">number of OpenStack instances.</span>
          .group_by<span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> xp: <span style="color: #93E0E3;">(</span>xp.scenario, xp.rdbms, xp.oss<span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
          <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Compute the CDF of the resuls</span>
          .on_value_domap<span style="color: #D0BF8F;">(</span>attrgetter<span style="color: #93E0E3;">(</span><span style="color: #CC9393;">'dataframe'</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
          .on_value<span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> dfs: pd.concat<span style="color: #93E0E3;">(</span>dfs<span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
          .on_value<span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> df: df.<span style="color: #DCDCCC; font-weight: bold;">sum</span><span style="color: #93E0E3;">(</span>axis=<span style="color: #CC9393;">'columns'</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
          .on_value<span style="color: #D0BF8F;">(</span>make_cumulative_frequency<span style="color: #D0BF8F;">)</span>
          .to_dict<span style="color: #D0BF8F;">()</span>
         <span style="color: #BFEBBF;">)</span>, legend=<span style="color: #CC9393;">'all'</span><span style="color: #DCDCCC;">)</span>
</pre>
</div>


<figure id="org9254f21">
<img src='{{ "assets/evaluation-of-openstack-multi-region-keystone-deployments/oss-impact-high-cdf.png" | absolute_url }}' alt="oss-impact-high-cdf.png" class="zoom-on-click">

<figcaption><span class="figure-number">Figure 22: </span>Impact of the Number of OpenStack Instances on the Completion Time under High Load &#x2013; Cumulative Distribution.</figcaption>
</figure>


### Delay impact under light load

<div class="org-src-container">
<pre class="src src-python" id="orgef69d5c">delay_plot<span style="color: #DCDCCC;">(</span><span style="color: #CC9393;">"%s Completion Time (s)"</span>,
           series_stackedbar_plot,
           <span style="color: #CC9393;">'imgs/delay-impact-light'</span>,
           <span style="color: #BFEBBF;">(</span><span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">-- Experiments selection</span>
            XPS
            <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">We are only interested in results with 9 OpenStack instances</span>
            .<span style="color: #DCDCCC; font-weight: bold;">filter</span><span style="color: #D0BF8F;">(</span>when_oss<span style="color: #93E0E3;">(</span><span style="color: #BFEBBF;">9</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            .<span style="color: #DCDCCC; font-weight: bold;">filter</span><span style="color: #D0BF8F;">(</span>compose<span style="color: #93E0E3;">(</span>not_, when_delay<span style="color: #9FC59F;">(</span><span style="color: #BFEBBF;">10</span><span style="color: #9FC59F;">)</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Also, remove values greater than the 95th percentile</span>
            .<span style="color: #DCDCCC; font-weight: bold;">map</span><span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> xp: xp.set_dataframe<span style="color: #93E0E3;">(</span>filter_percentile<span style="color: #9FC59F;">(</span>.<span style="color: #BFEBBF;">95</span>, xp.dataframe<span style="color: #9FC59F;">)</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Filter light load</span>
            .<span style="color: #DCDCCC; font-weight: bold;">filter</span><span style="color: #D0BF8F;">(</span>compose<span style="color: #93E0E3;">(</span>not_, is_high<span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Group results by scenario's name, RDBMS technology and</span>
            <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">delay</span>
            .group_by<span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> xp: <span style="color: #93E0E3;">(</span>xp.scenario, xp.rdbms, xp.delay<span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Compute the mean, std and success of the results</span>
            .on_value_domap<span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> xp: <span style="color: #93E0E3;">(</span>xp.dataframe, xp.success<span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            .on_value<span style="color: #D0BF8F;">(</span>unpack<span style="color: #93E0E3;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> dfs, succs: <span style="color: #9FC59F;">(</span>pd.concat<span style="color: #94BFF3;">(</span>dfs<span style="color: #94BFF3;">)</span>, np.mean<span style="color: #94BFF3;">(</span>succs<span style="color: #94BFF3;">)</span><span style="color: #9FC59F;">)</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            .on_value<span style="color: #D0BF8F;">(</span>unpack<span style="color: #93E0E3;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> df, succ: <span style="color: #9FC59F;">(</span>df.mean<span style="color: #94BFF3;">()</span>, df.<span style="color: #DCDCCC; font-weight: bold;">sum</span><span style="color: #94BFF3;">(</span>axis=<span style="color: #BFEBBF;">1</span><span style="color: #94BFF3;">)</span>.std<span style="color: #94BFF3;">()</span>, succ<span style="color: #9FC59F;">)</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            .to_dict<span style="color: #D0BF8F;">()</span>
         <span style="color: #BFEBBF;">)</span><span style="color: #DCDCCC;">)</span>
</pre>
</div>


<figure id="org5a5a573">
<img src='{{ "assets/evaluation-of-openstack-multi-region-keystone-deployments/delay-impact-light.png" | absolute_url }}' alt="delay-impact-light.png" class="zoom-on-click">

<figcaption><span class="figure-number">Figure 23: </span>Impact of the Network Delay on the Completion Time under Light Load &#x2013; Median Time for Every Operations (Lower is Better).</figcaption>
</figure>

<div class="org-src-container">
<pre class="src src-python" id="org6bab319">delay_plot<span style="color: #DCDCCC;">(</span><span style="color: #CC9393;">"%s Cumulative Percent"</span>,
           series_linear_plot,
           <span style="color: #CC9393;">'imgs/delay-impact-light-cdf'</span>,
           <span style="color: #BFEBBF;">(</span><span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">-- Experiments selection</span>
            XPS
            <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">We are only interested in results with 9 OpenStack instances</span>
            .<span style="color: #DCDCCC; font-weight: bold;">filter</span><span style="color: #D0BF8F;">(</span>when_oss<span style="color: #93E0E3;">(</span><span style="color: #BFEBBF;">9</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            .<span style="color: #DCDCCC; font-weight: bold;">filter</span><span style="color: #D0BF8F;">(</span>compose<span style="color: #93E0E3;">(</span>not_, when_delay<span style="color: #9FC59F;">(</span><span style="color: #BFEBBF;">10</span><span style="color: #9FC59F;">)</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Also, remove values greater than the 95th percentile</span>
            .<span style="color: #DCDCCC; font-weight: bold;">map</span><span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> xp: xp.set_dataframe<span style="color: #93E0E3;">(</span>filter_percentile<span style="color: #9FC59F;">(</span>.<span style="color: #BFEBBF;">95</span>, xp.dataframe<span style="color: #9FC59F;">)</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Filter light load</span>
            .<span style="color: #DCDCCC; font-weight: bold;">filter</span><span style="color: #D0BF8F;">(</span>compose<span style="color: #93E0E3;">(</span>not_, is_high<span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Group results by scenario's name, RDBMS technology and</span>
            <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">delay</span>
            .group_by<span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> xp: <span style="color: #93E0E3;">(</span>xp.scenario, xp.rdbms, xp.delay<span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Compute the CDF</span>
            .on_value_domap<span style="color: #D0BF8F;">(</span>attrgetter<span style="color: #93E0E3;">(</span><span style="color: #CC9393;">'dataframe'</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            .on_value<span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> dfs: pd.concat<span style="color: #93E0E3;">(</span>dfs<span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            .on_value<span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> df: df.<span style="color: #DCDCCC; font-weight: bold;">sum</span><span style="color: #93E0E3;">(</span>axis=<span style="color: #CC9393;">'columns'</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            .on_value<span style="color: #D0BF8F;">(</span>make_cumulative_frequency<span style="color: #D0BF8F;">)</span>
            .to_dict<span style="color: #D0BF8F;">()</span>
         <span style="color: #BFEBBF;">)</span>, legend=<span style="color: #CC9393;">'all'</span><span style="color: #DCDCCC;">)</span>
</pre>
</div>


<figure id="org4a78f99">
<img src='{{ "assets/evaluation-of-openstack-multi-region-keystone-deployments/delay-impact-light-cdf.png" | absolute_url }}' alt="delay-impact-light-cdf.png" class="zoom-on-click">

<figcaption><span class="figure-number">Figure 24: </span>Impact of the Network Delay on the Completion Time under Light Load &#x2013; Cumulative Distribution.</figcaption>
</figure>


### Delay impact under high load

<div class="org-src-container">
<pre class="src src-python" id="org0c6927f">delay_plot<span style="color: #DCDCCC;">(</span><span style="color: #CC9393;">"%s Completion Time (s)"</span>,
           series_stackedbar_plot,
           <span style="color: #CC9393;">'imgs/delay-impact-high'</span>,
           <span style="color: #BFEBBF;">(</span><span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">-- Experiments selection</span>
            XPS
            <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">We are only interested in results with 9 OpenStack instances</span>
            .<span style="color: #DCDCCC; font-weight: bold;">filter</span><span style="color: #D0BF8F;">(</span>when_oss<span style="color: #93E0E3;">(</span><span style="color: #BFEBBF;">9</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            .<span style="color: #DCDCCC; font-weight: bold;">filter</span><span style="color: #D0BF8F;">(</span>compose<span style="color: #93E0E3;">(</span>not_, when_delay<span style="color: #9FC59F;">(</span><span style="color: #BFEBBF;">10</span><span style="color: #9FC59F;">)</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Also, remove values greater than the 95th percentile</span>
            .<span style="color: #DCDCCC; font-weight: bold;">map</span><span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> xp: xp.set_dataframe<span style="color: #93E0E3;">(</span>filter_percentile<span style="color: #9FC59F;">(</span>.<span style="color: #BFEBBF;">95</span>, xp.dataframe<span style="color: #9FC59F;">)</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Filter high load</span>
            .<span style="color: #DCDCCC; font-weight: bold;">filter</span><span style="color: #D0BF8F;">(</span>is_high<span style="color: #D0BF8F;">)</span>
            <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Group results by scenario's name, RDBMS technology and</span>
            <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">delay</span>
            .group_by<span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> xp: <span style="color: #93E0E3;">(</span>xp.scenario, xp.rdbms, xp.delay<span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Compute the mean, std and success of the results</span>
            .on_value_domap<span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> xp: <span style="color: #93E0E3;">(</span>xp.dataframe, xp.success<span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            .on_value<span style="color: #D0BF8F;">(</span>unpack<span style="color: #93E0E3;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> dfs, succs: <span style="color: #9FC59F;">(</span>pd.concat<span style="color: #94BFF3;">(</span>dfs<span style="color: #94BFF3;">)</span>, np.mean<span style="color: #94BFF3;">(</span>succs<span style="color: #94BFF3;">)</span><span style="color: #9FC59F;">)</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            .on_value<span style="color: #D0BF8F;">(</span>unpack<span style="color: #93E0E3;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> df, succ: <span style="color: #9FC59F;">(</span>df.mean<span style="color: #94BFF3;">()</span>, df.<span style="color: #DCDCCC; font-weight: bold;">sum</span><span style="color: #94BFF3;">(</span>axis=<span style="color: #BFEBBF;">1</span><span style="color: #94BFF3;">)</span>.std<span style="color: #94BFF3;">()</span>, succ<span style="color: #9FC59F;">)</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            .to_dict<span style="color: #D0BF8F;">()</span>
         <span style="color: #BFEBBF;">)</span><span style="color: #DCDCCC;">)</span>
</pre>
</div>


<figure id="org357b8a9">
<img src='{{ "assets/evaluation-of-openstack-multi-region-keystone-deployments/delay-impact-high.png" | absolute_url }}' alt="delay-impact-high.png" class="zoom-on-click">

<figcaption><span class="figure-number">Figure 25: </span>Impact of the Network Delay on the Completion Time under High Load &#x2013; Median Time for Every Operations (Lower is Better).</figcaption>
</figure>

<div class="org-src-container">
<pre class="src src-python" id="org7683243">delay_plot<span style="color: #DCDCCC;">(</span><span style="color: #CC9393;">"%s Cumulative Percent"</span>,
           series_linear_plot,
           <span style="color: #CC9393;">'imgs/delay-impact-high-cdf'</span>,
           <span style="color: #BFEBBF;">(</span><span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">-- Experiments selection</span>
            XPS
            <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">We are only interested in results with 9 OpenStack instances</span>
            .<span style="color: #DCDCCC; font-weight: bold;">filter</span><span style="color: #D0BF8F;">(</span>when_oss<span style="color: #93E0E3;">(</span><span style="color: #BFEBBF;">9</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            .<span style="color: #DCDCCC; font-weight: bold;">filter</span><span style="color: #D0BF8F;">(</span>compose<span style="color: #93E0E3;">(</span>not_, when_delay<span style="color: #9FC59F;">(</span><span style="color: #BFEBBF;">10</span><span style="color: #9FC59F;">)</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Also, remove values greater than the 95th percentile</span>
            .<span style="color: #DCDCCC; font-weight: bold;">map</span><span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> xp: xp.set_dataframe<span style="color: #93E0E3;">(</span>filter_percentile<span style="color: #9FC59F;">(</span>.<span style="color: #BFEBBF;">95</span>, xp.dataframe<span style="color: #9FC59F;">)</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Filter high load</span>
            .<span style="color: #DCDCCC; font-weight: bold;">filter</span><span style="color: #D0BF8F;">(</span>is_high<span style="color: #D0BF8F;">)</span>
            <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Group results by scenario's name, RDBMS technology and</span>
            <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">delay</span>
            .group_by<span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> xp: <span style="color: #93E0E3;">(</span>xp.scenario, xp.rdbms, xp.delay<span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            <span style="color: #5F7F5F;"># </span><span style="color: #7F9F7F;">Compute the CDF</span>
            .on_value_domap<span style="color: #D0BF8F;">(</span>attrgetter<span style="color: #93E0E3;">(</span><span style="color: #CC9393;">'dataframe'</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            .on_value<span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> dfs: pd.concat<span style="color: #93E0E3;">(</span>dfs<span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            .on_value<span style="color: #D0BF8F;">(</span><span style="color: #F0DFAF; font-weight: bold;">lambda</span> df: df.<span style="color: #DCDCCC; font-weight: bold;">sum</span><span style="color: #93E0E3;">(</span>axis=<span style="color: #CC9393;">'columns'</span><span style="color: #93E0E3;">)</span><span style="color: #D0BF8F;">)</span>
            .on_value<span style="color: #D0BF8F;">(</span>make_cumulative_frequency<span style="color: #D0BF8F;">)</span>
            .to_dict<span style="color: #D0BF8F;">()</span>
         <span style="color: #BFEBBF;">)</span>, legend=<span style="color: #CC9393;">'all'</span><span style="color: #DCDCCC;">)</span>
</pre>
</div>


<figure id="org28ff696">
<img src='{{ "assets/evaluation-of-openstack-multi-region-keystone-deployments/delay-impact-high-cdf.png" | absolute_url }}' alt="delay-impact-high-cdf.png" class="zoom-on-click">

<figcaption><span class="figure-number">Figure 26: </span>Impact of the Network Delay on the Completion Time under High Load &#x2013; Cumulative Distribution.</figcaption>
</figure>


### Take into account the client locality


<a id="org47fa1fd"></a>

## Detailed Rally scenarios


<a id="org38279d5"></a>

### keystone/authenticate-user-and-validate-token

Description: authenticate and validate a keystone token.

Definition Code:
<a href="https://github.com/openstack/rally-openstack/blob/6158c1139c0a4d88cab74481c5cbfc8be398f481/samples/tasks/scenarios/keystone/authenticate-user-and-validate-token.yaml">samples/tasks/scenarios/keystone/authenticate-user-and-validate-token</a>

Source Code:
<a href="https://github.com/openstack/rally-openstack/blob/b1ae405b7fab355f3062cdb56a5b187fc6f2907f/rally_openstack/scenarios/keystone/basic.py#L111-L120">rally\_openstack.scenarios.keystone.basic.AuthenticateUserAndValidateToken</a>

List of keystone functionalities:

1.  keystone\_v3.fetch\_token
2.  keystone\_v3.validate\_token

%Reads/%Writes: 96.46/3.54

Number of runs: 20


### keystone/create-add-and-list-user-roles

Description: create user role, add it and list user roles for given
user.

Definition Code:
<a href="https://github.com/openstack/rally-openstack/blob/6158c1139c0a4d88cab74481c5cbfc8be398f481/samples/tasks/scenarios/keystone/create-add-and-list-user-roles.yaml">samples/tasks/scenarios/keystone/create-add-and-list-user-roles</a>

Source Code:
<a href="https://github.com/openstack/rally-openstack/blob/b1ae405b7fab355f3062cdb56a5b187fc6f2907f/rally_openstack/scenarios/keystone/basic.py#L214-L228">rally\_openstack.scenarios.keystone.basic.CreateAddAndListUserRoles</a>

List of keystone functionalities:

1.  keystone\_v3.create\_role
2.  keystone\_v3.add\_role
3.  keystone\_v3.list\_roles

%Reads/%Writes: 96.22/3.78

Number of runs: 100


<a id="org36b3403"></a>

### keystone/create-and-list-tenants

Description: create a keystone tenant with random name and list all
tenants.

Definition Code:
<a href="https://github.com/openstack/rally-openstack/blob/6158c1139c0a4d88cab74481c5cbfc8be398f481/samples/tasks/scenarios/keystone/create-and-list-tenants.yaml">samples/tasks/scenarios/keystone/create-and-list-tenants</a>

Source Code:
<a href="https://github.com/openstack/rally-openstack/blob/b1ae405b7fab355f3062cdb56a5b187fc6f2907f/rally_openstack/scenarios/keystone/basic.py#L166-L181">rally\_openstack.scenarios.keystone.basic.CreateAndListTenants</a>

List of keystone functionalities:

1.  keystone\_v3.create\_project
2.  keystone\_v3.list\_projects

%Reads/%Writes: 92.12/7.88

Number of runs: 10


### keystone/get-entities

Description: get instance of a tenant, user, role and service by id&rsquo;s.
An ephemeral tenant, user, and role are each created. By default,
fetches the &rsquo;keystone&rsquo; service.

List of keystone functionalities:

1.  keystone\_v3.create\_project
2.  keystone\_v3.create\_user
3.  keystone\_v3.create\_role
    1.  keystone\_v3.list\_roles
    2.  keystone\_v3.add\_role
4.  keystone\_v3.get\_project
5.  keystone\_v3.get\_user
6.  keystone\_v3.get\_role
7.  keystone\_v3.list\_services
8.  keystone\_v3.get\_services

%Reads/%Writes: 91.9/8.1

Definition Code:
<a href="https://github.com/openstack/rally-openstack/blob/6158c1139c0a4d88cab74481c5cbfc8be398f481/samples/tasks/scenarios/keystone/get-entities.yaml">samples/tasks/scenarios/keystone/get-entities</a>

Source Code:
<a href="https://github.com/openstack/rally-openstack/blob/b1ae405b7fab355f3062cdb56a5b187fc6f2907f/rally_openstack/scenarios/keystone/basic.py#L231-L261">rally\_openstack.scenarios.keystone.basic.GetEntities</a>

Number of runs: 100


<a id="org2b70cfd"></a>

### keystone/create-user-update-password

Description: create user and update password for that user.

List of keystone functionalities:

1.  keystone\_v3.create\_user
2.  keystone\_v3.update\_user

%Reads/%Writes: 89.79/10.21

Definition Code:
<a href="https://github.com/openstack/rally-openstack/blob/6158c1139c0a4d88cab74481c5cbfc8be398f481/samples/tasks/scenarios/keystone/create-user-update-password.yaml">samples/tasks/scenarios/keystone/create-user-update-password</a>

Source Code:
<a href="https://github.com/openstack/rally-openstack/blob/b1ae405b7fab355f3062cdb56a5b187fc6f2907f/rally_openstack/scenarios/keystone/basic.py#L306-L320">rally\_openstack.scenarios.keystone.basic.CreateUserUpdatePassword</a>

Number of runs: 100


### keystone/create-user-set-enabled-and-delete

Description: create a keystone user, enable or disable it, and delete
it.

List of keystone functionalities:

1.  keystone\_v3.create\_user
2.  keystone\_v3.update\_user
3.  keystone\_v3.delete\_user

%Reads/%Writes: 91.07/8.93

Definition Code:
<a href="https://github.com/openstack/rally-openstack/blob/6158c1139c0a4d88cab74481c5cbfc8be398f481/samples/tasks/scenarios/keystone/create-user-set-enabled-and-delete.yaml">samples/tasks/scenarios/keystone/create-user-set-enabled-and-delete</a>

Source Code:
<a href="https://github.com/openstack/rally-openstack/blob/b1ae405b7fab355f3062cdb56a5b187fc6f2907f/rally_openstack/scenarios/keystone/basic.py#L75-L91">rally\_openstack.scenarios.keystone.basic.CreateUserSetEnabledAndDelete</a>

Number of runs: 100


### keystone/create-and-list-users

Description: create a keystone user with random name and list all
users.

List of keystone functionalities:

1.  keystone\_v3.create\_user
2.  keystone\_v3.list\_users

%Reads/%Writes: 92.05/7.95

Definition Code:
<a href="https://github.com/openstack/rally-openstack/blob/6158c1139c0a4d88cab74481c5cbfc8be398f481/samples/tasks/scenarios/keystone/create-add-and-list-user-roles.yaml">samples/tasks/scenarios/keystone/create-and-list-users</a>

Source Code:
<a href="https://github.com/openstack/rally-openstack/blob/b1ae405b7fab355f3062cdb56a5b187fc6f2907f/rally_openstack/scenarios/keystone/basic.py#L145-L163">rally\_openstack.scenarios.keystone.basic.CreateAndListUsers</a>.

Number of runs: 100


# Footnotes

<sup><a id="fn.1" href="#fnr.1">1</a></sup> Here we present two deployment
alternatives. But there are several others such as multiple regions,
CellV2 and federation deployment.

<sup><a id="fn.2" href="#fnr.2">2</a></sup> TODO(Comment on global identifier): general way to
implement this is vector clock + causal broadcast (i.e., costly
operation). Actually, galera do not tell how it implement this but say
that it uses a clock that may differ of 1 second from one peer to
another. This means that two concurrent transactions, on two different
peers, done during the same second may end with the same id.
Henceforth, Galera do not ensure first order commit and inconsistency
may arises.
