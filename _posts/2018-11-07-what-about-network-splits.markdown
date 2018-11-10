---
layout: post
title: What about network splits
date: 2018-11-07
author: Marie Delavergne, Ronan-Alexandre Cherrueau
categories: [OpenStack, network partitions]
tags: [OpenStack, network partitions, network splits, edge]
---

# Living on the edge #

Modern applications in the realm of the internet, such as the internet of things, compel the development for edge infrastructures. The concept of edge computing is to distribute the computation on a large number of edge devices, closer to the places where the data are collected.
We define this as hundreds of auto-managed and geo-distributed micro data center of dozens of servers. Since they are distributed all over the globe, one can expect to have different latency and bandwith across the network. But what can also be expected and have nonetheless unexpected consequences are network partitions that could happen over such a distributed network.

A partition occurs when the link to a resource is severed and this resource becomes isolated from the others. This resource can be a node from a database, a compute node, an entire data center, etc. When this split happens, we can expected either that this isolated part executions can differ from the main partition, which puts it in an incoherent state, or that it can become simply unavailable. The way it is expected to behave is called the partition-tolerant behaviour.

To build an edge infrastructure without reinventing the wheel, the Discovery initiative investigates the use of OpenStack. OpenStack is an open-source cloud computing infrastructure resource manager widely used, and more and more in the context of edge computing. Openstack is built of two types of nodes which constitutes the data plane on one side and the control plane on the other. The data plane
nodes are able to fulfill typical XaaS needs, such as computation, storage, network, etc. The control nodes are required to process the incoming requests which will probably require to communicate with the data plane. Since all these services are distributed across different nodes, we can expect to get a partition between them at some point.

The goal of this study is to shed some light on the behaviour of OpenStack when network partition occurs between a control and a compute node.

# Experimental protocol #
## Base configuration ##

To make the required tests, we used [Enos](https://github.com/BeyondTheClouds/enos), a tool previously developed by the Discovery Initiative, and deployed on [Grid'5000](https://www.grid5000.fr/mediawiki/index.php/Grid5000:Home), a testbed dedicated to research.  The platform gives access to approximately 1000 machines grouped in 30 clusters geographically distributed in 8 sites. This study uses the [paravance cluster](https://www.grid5000.fr/mediawiki/index.php/Rennes:Hardware#paravance) composed of 72 nodes with each:
- **CPU:** Intel Xeon E5-2630 v3 (Haswell, 2.40GHz, 2 CPUs/node, 8 cores/CPU)
- **Memory:** 128 GB
- **Network:**
  - eth0/eno1, Ethernet, configured rate: 10 Gbps, model: Intel 82599ES 10-Gigabit SFI/SFP+ Network Connection, driver: ixgbe
  - eth1/eno2, Ethernet, configured rate: 10 Gbps, model: Intel 82599ES 10-Gigabit SFI/SFP+ Network Connection, driver: ixgbe

Enos enables us to get the resources, deploy the required topology and configure OpenStack according to our needs. We used the following configuration:
```python
topology:
  grp1:
    paravance:
      control: 1
      network: 1
  grp2:
    paravance:
      compute: 1
  grp3:
    paravance:
      compute: 2
```
We thus created 3 different groups of 5 paravance servers, connected by two virtual LANs and plugged to each nodes using two different NICs. This servers are 3 compute nodes, one to handle Neutron (network) and one control to manage everything (Glance, Keystone, MariaDB, Horizon, etc.), as shown on Figure<a href="#os_topo_base">1</a>.

Every VM we booted were placed on the compute nodes, numbered from 1 to 3. As seen on the figure and the topology, the nodes are divided into 3 groups: grp1 for the control and the network nodes, grp1 for the compute node which will be isolated and the grp3 acting as compute nodes to make the experiment as well as control samples.

<figure id="os_topo_base">
<img src='{{ "assets/what-about-network-splits/openstack_topology_base.svg" | absolute_url }}' alt="openstack base topology">
<figcaption style="text-align:center"><span class="figure-number">Figure 1: </span>Openstack base topology</figcaption>
</figure>

Enos deployed everything as displayed in the previous figure. It created two VLans, each one associated to a node's NIC.

## OpenStack topology ##

We then used a [heat template file]({{ "assets/what-about-network-splits/heat.hot" | absolute_url }}) to deploy 6 VMs distributed equally across the computes (i.e. 2 VMs per compute) and across 2 private networks (i.e. 3 VMs per network).
<figure id="net_topo">
<img src='{{ "assets/what-about-network-splits/network_topology.svg" | absolute_url }}' alt="Network topology">
	<figcaption style="text-align:center"><span class="figure-number">Figure 2: </span>Network topology</figcaption>
</figure>

The figure [2](#net_topo) presents the network topology in a similar way to Horizon dashboard. We can see that VMs 1, 3 and 5 are in the first network (Nw1) and the 3 others in the second network (Nw2). The yellow/purple/orange represents the compute on which the VMs are located. Thus, VMs 1 and 4 are collocated and VM 3 and 6 are separated from each other. To sum up the positioning of VMs across computes, groups and network:

| VM  | Compute | Group | Network |
|:---:| :---:   | :---: | :---:   |
| 1   | 1       | 2     | 1       |
| 2   | 2       | 3     | 2       |
| 3   | 3       | 3     | 1       |
| 4   | 1       | 2     | 2       |
| 5   | 2       | 3     | 1       |
| 6   | 3       | 3     | 2       |

This architecture allows us to test communications between two networks, two computes and to the outside.
We consider three types of communications:
  * **L2/L3**: In **L2** configuration, everything happens in the same network, whereas in **L3** the messages goes from a network to another.
  * **Full/Dense**: This is about the positioning of the VMs towards the compute. In **dense** mode, the VMs are on the same compute. In **full mode**, they are scattered on two different computes.
  * **East-West/North-South**: Whether the VMs talk to each other (**East-West**) or to an external address, e.g. 8.8.8.8 (**North-South**)

## Cutting the edge ##

We then applied some tc rules using `enos tc` from Enos (these lines are simply put in the configuration file of Enos):
```python
network_constraints:
  enable: true
  default_delay: 0ms
  default_rate: 10gbit
  default_loss: 0%
  constraints:
    - src: grp1
      dst: grp2
      loss: 100%
      symetric: true
      network: 'network_interface'
```
These rules use traffic control to drop all messages (`loss:100%`) on the nodes of grp1 going go or coming from the ip adresses of grp2 nodes and vice-versa (`symetric:true`). This only concerns the `network_interface` vLAN, which means it is only for one NIC (used by OpenStack). As you can see in Figure[3](#os_topo) and from both configurations, it is worth precising that when we talk about cutting the link between compute1 and the control node, we in fact cut the link between control-network nodes and compute1, to avoid the ability to pass requests through Neutron. In the rest of the article, we will refer to this cut as the cut between control and compute1, but keep in mind we also cut the link between network and compute1 nodes.

We can nonetheless represent the topology depicted in Figure [3](#os_topo), as far as OpenStack is concerned:
<figure id="os_topo">
<img src='{{ "assets/what-about-network-splits/openstack_topology.svg" | absolute_url }}' alt="openstack topology">
<figcaption style="text-align:center"><span class="figure-number">Figure 3: </span>Openstack topology</figcaption>
</figure>


# Experiments #

We deployed our topology and pinged the VMs from one another (East-West) or the public network (North-South) to ensure everything worked as intended. We then cut the network using `enos tc`. This had several consequences:
1. On the horizon dashboard and through the command `openstack hypervisor list`, compute1 became down after a few seconds
2. The VMs became unattainable through the command `openstack server ssh`
3. The VMs were not marked as down or shutoff on the horizon dashboard nor through the command line `openstack server list`.

We will now dive a little more into the experiments and the results.


## A bit of pinging ##

### L3 Full East-West/North-South ###

<figure id="L3_full">
<img src='{{ "assets/what-about-network-splits/L3_full.svg" | absolute_url }}' alt="L3 full">
<figcaption style="text-align:center"><span class="figure-number">Figure 6: </span>L3 Full</figcaption>
</figure>


| Domain | Colocation | Ping type   |  Source         | Destination    | Result |
| :----: | :--------: | :---------: | :-------------: | :------------: | :----: |
| L3     | full       | East-West   |  VM1 (C1-Nw1)   | VM2 (C2-Nw2)   | <span style="color:red">X</span>      |
| L3     | full       | East-West   |  VM2 (C2-Nw2)   | VM1 (C1-Nw1)   | <span style="color:red">X</span>      |
| L3     | full       | North-South |  VM1 (C1-Nw1)   | 8.8.8.8        | <span style="color:red">X</span>      |
| L3     | full       | North-South |  VM2 (C2-Nw1)   | 8.8.8.8        | <span style="color:green">V </span>     |

As expected, compute1's VMs are unreachable so no communication were possible through the VMs. Indeed, the associated floating ip can't be used by OpenStack so there are no way to connect to the VMs through OpenStack.


### L3 Dense East-West/North-South ###

<figure id="L3_dense">
<img src='{{ "assets/what-about-network-splits/L3_dense.svg" | absolute_url }}' alt="L3 dense">
<figcaption style="text-align:center"><span class="figure-number">Figure 7: </span>L3 Dense</figcaption>
</figure>

| Domain | Colocation | Ping type   |  Source         | Destination    | Result |
| :----: | :--------: | :---------: | :-------------: | :------------: | :----: |
| L3     | dense      | East-West   |  VM1 (C1-Nw1)   | VM4 (C1-Nw2)   | <span style="color:red">X</span>      |
| L3     | dense      | East-West   |  VM4 (C1-Nw2)   | VM1 (C1-Nw1)   | <span style="color:red">X</span>      |
| L3     | dense      | North-South |  VM1 (C1-Nw1)   | 8.8.8.8        | <span style="color:red">X</span>      |
| L3     | dense      | North-South |  VM4 (C1-Nw2)   | 8.8.8.8        | <span style="color:red">X</span>      |

As previously, since both VMs are on the unreachable compute, we can not make any ping from them.


### L2 Full East-West/North-South ###

<figure id="L2_full">
<img src='{{ "assets/what-about-network-splits/L2_full.svg" | absolute_url }}' alt="L2 full">
<figcaption style="text-align:center"><span class="figure-number">Figure 4: </span>L2 Full</figcaption>
</figure>

| Domain | Colocation | Ping type   |  Source         | Destination    | Result |
| :----: | :--------: | :---------: | :-------------: | :------------: | :----: |
| L2     | full       | East-West   |  VM1 (C1-Nw1)   | VM3 (C3-Nw1)   | <span style="color:red">X</span>   |
| L2     | full       | East-West   |  VM3 (C3-Nw1)   | VM1 (C1-Nw1) private address  | <span style="color:green">V </span>     |
| L2     | full       | East-West   |  VM3 (C3-Nw1)   | VM1 (C1-Nw1) public address   | <span style="color:red">X</span>     |
| L2     | full       | North-South |  VM1 (C1-Nw1)   | 8.8.8.8        | <span style="color:red">X</span>      |
| L2     | full       | North-South |  VM3 (C3-Nw1)   | 8.8.8.8        | <span style="color:green">V </span>     |

Here the VM on the compute2 (VM3) seems to be able to ping the one on compute1 (VM1) through its private ip, since it does not need to go through the floating public ip. Indeed, they are on the same private network so nothing has to go through Neutron functionalities.

### L2 Dense East-West/North-South ###

This had to be done using a different topology as we did not have a compute that was cut off the network and had two VMs on the same network. The principle remains entirely the same.

<figure id="L2_dense">
<img src='{{ "assets/what-about-network-splits/L2_dense.svg" | absolute_url }}' alt="L2 dense">
<figcaption style="text-align:center"><span class="figure-number">Figure 5: </span>L3 Dense</figcaption>
</figure>


| Domain | Colocation | Ping type   |  Source         | Destination    | Result |
| :----: | :--------: | :---------: | :-------------: | :------------: | :----: |
| L2     | dense      | East-West   |  VM1 (C1-Nw1)   | VM2 (C1-Nw1)   | <span style="color:red">X</span>      |
| L2     | dense      | East-West   |  VM2 (C1-Nw1)   | VM1 (C1-Nw1)   | <span style="color:red">X</span>      |
| L2     | dense      | North-South |  VM1 (C1-Nw1)   | 8.8.8.8        | <span style="color:red">X</span>      |
| L2     | dense      | North-South |  VM2 (C1-Nw1)   | 8.8.8.8        | <span style="color:red">X</span>      |

As for the previous dense experiments, nothing can be done since we can not reach the VMs on compute1.


## Playing on the edge

As stated before, when a network gets partitioned, the resources have to implement a partition-tolerant behaviour in order to avoid an interruption of service. In this part, we tried to poke around OpenStack and see how well it behaves under a split.


First thing we have done is to cut the network between the control and compute1 node and check how Openstack would detect it. Then when reconnected the nodes and checked what happened.
After around a minute of disconnection, we obtained:
```
$ openstack hypervisor list
+----+------------------------------------------+-----------------+------------+-------+
| ID | Hypervisor Hostname                      | Hypervisor Type | Host IP    | State |
+----+------------------------------------------+-----------------+------------+-------+
|  1 | paravance-45-kavlan-4.rennes.grid5000.fr | QEMU            | 10.24.8.45 | down  |
|  2 | paravance-36-kavlan-4.rennes.grid5000.fr | QEMU            | 10.24.8.36 | up    |
|  3 | paravance-49-kavlan-4.rennes.grid5000.fr | QEMU            | 10.24.8.49 | up    |
+----+------------------------------------------+-----------------+------------+-------+
```
Then, almost as soon as the connection came back -though sometimes it takes up to 20 seconds-, the node was up again:
```
$ enos tc --reset
[...]
$ openstack hypervisor list
+----+------------------------------------------+-----------------+------------+-------+
| ID | Hypervisor Hostname                      | Hypervisor Type | Host IP    | State |
+----+------------------------------------------+-----------------+------------+-------+
|  1 | paravance-45-kavlan-4.rennes.grid5000.fr | QEMU            | 10.24.8.45 | up    |
|  2 | paravance-36-kavlan-4.rennes.grid5000.fr | QEMU            | 10.24.8.36 | up    |
|  3 | paravance-49-kavlan-4.rennes.grid5000.fr | QEMU            | 10.24.8.49 | up    |
+----+------------------------------------------+-----------------+------------+-------+
```

Second thing was to test the consistence of VM status. Here, we:
1. Created the network as previously
2. Cut the link between control-network and compute1 node
3. Kill the VMs on the partitioned compute. This can be done since we still can access the compute through Grid'5000 and is done thanks to `virsh destroy <VM_number>`.
4. Plug the compute back to OpenStack by resetting traffic control rules thanks to `enos tc :-reset`.
As before, the compute node reappear as active in OpenStack in a matter of seconds. Second, the VMs appeared as shutoff almost as soon as the compute rejoined the network.
This is the status when checking with the command line when the link is cut off:
```
$ openstack server list
+--------------------------------------+------+--------+---------------------------------+------------+----------+
| ID                                   | Name | Status | Networks                        | Image      | Flavor   |
+--------------------------------------+------+--------+---------------------------------+------------+----------+
| f1608a3a-38c0-4254-8458-fd0a46ecb243 | VM5  | ACTIVE | Network1=10.1.0.7, 10.24.90.17  | cirros.uec | m1.small |
| 2a7f93cc-3d67-4801-8d34-17a89e6299d1 | VM3  | ACTIVE | Network1=10.1.0.4, 10.24.90.16  | cirros.uec | m1.small |
| d7576000-9774-4a1f-b23b-035583d51dc8 | VM4  | ACTIVE | Network2=10.2.0.10, 10.24.90.3  | cirros.uec | m1.small |
| 087bed99-01a8-4407-a551-35f567387fec | VM2  | ACTIVE | Network2=10.2.0.11, 10.24.90.10 | cirros.uec | m1.small |
| 1524183e-2ab4-48b1-8cd1-37401b12013b | VM1  | ACTIVE | Network1=10.1.0.9, 10.24.90.6   | cirros.uec | m1.small |
| 3ebf4597-cdb2-4380-b9f7-b4b96da98a66 | VM6  | ACTIVE | Network2=10.2.0.9, 10.24.90.7   | cirros.uec | m1.small |
+--------------------------------------+------+--------+---------------------------------+------------+----------+
```
And the same command ran as soon as the compute is marked as `up`:
```
$ openstack server list
+--------------------------------------+------+---------+---------------------------------+------------+----------+
| ID                                   | Name | Status  | Networks                        | Image      | Flavor   |
+--------------------------------------+------+---------+---------------------------------+------------+----------+
| f1608a3a-38c0-4254-8458-fd0a46ecb243 | VM5  | ACTIVE  | Network1=10.1.0.7, 10.24.90.17  | cirros.uec | m1.small |
| 2a7f93cc-3d67-4801-8d34-17a89e6299d1 | VM3  | ACTIVE  | Network1=10.1.0.4, 10.24.90.16  | cirros.uec | m1.small |
| d7576000-9774-4a1f-b23b-035583d51dc8 | VM4  | SHUTOFF | Network2=10.2.0.10, 10.24.90.3  | cirros.uec | m1.small |
| 087bed99-01a8-4407-a551-35f567387fec | VM2  | ACTIVE  | Network2=10.2.0.11, 10.24.90.10 | cirros.uec | m1.small |
| 1524183e-2ab4-48b1-8cd1-37401b12013b | VM1  | ACTIVE  | Network1=10.1.0.9, 10.24.90.6   | cirros.uec | m1.small |
| 3ebf4597-cdb2-4380-b9f7-b4b96da98a66 | VM6  | ACTIVE  | Network2=10.2.0.9, 10.24.90.7   | cirros.uec | m1.small |
+--------------------------------------+------+---------+---------------------------------+------------+----------+
```

# What has been observed #

Though most of this study has been done manually aside from the deployment of OpenStack over the physical servers and the launching of the VMs, it gives a notion of the behaviour of OpenStack data plane when a network partition occurs. Though the VMs involved in the partition are no longer available, it seems that no consistency errors occur during this partioning. The `active` status of the VMs when their compute is considered down is arguably a good option, but feels pertinent since there are no way to known whether they are indeed down themselves or continue running during the partition and potentially continue some tasks they were computing.

We would like to retry those tests using [shaker](https://github.com/openstack/shaker) to get more detailled informations about the behaviour under such partitioning.

There are also still some ongoing work about the partition between compute nodes and distributed databases to check the consistency of the overall system.



Disclaimer: this article might get some modifications in the following weeks
