---
layout: post
title: Adding compute at will
date: 2019-05-16
author: Marie Delavergne
categories: OpenStack crowdsourced edge
---

The
[Inria&rsquo;s Discovery initiative](https://beyondtheclouds.github.io/) focus
on challenges around the Edge. In this study, we deal with the
challenge of adding/removing remote compute resources in an automatic
manner.

Concretely, we discuss the framework we developped to enable the
addition/removal of any remote compute resource WANWide, leveraging an
OpenVPN flat network when needed. This framework can be
found [here](https://github.com/badock/enos_openvpn). It is based
on [Enos](https://github.com/BeyondTheClouds/enos)
and [OpenVPN](https://openvpn.net/), as the name would suggest. The
original enos openvpn was made
by [Jonathan Pastor](http://www.jonathanpastor.fr/) using bash
scripts, and I have translated them to ansible before enabling the
adding, removal and rejoining of a node.


The example here is to have an ISP that would provide compute
resources. With the routers getting more and more sophisticated, we
could imagine that they could be used for computation when not
used. But this usage must be driven by the will of the owner of the
router to join this infrastructure, maybe for some compensation.

The idea is to have a master that deploy nodes to have a fully
deployed OpenStack, as presented in the Figure below. The first
deployed node is the one that handles Enos and OpenVPN, but it can be
physically located on the same node as the master (see informations
relative to the <a href="#host-file">host file for details</a>).


```
+-------------------------------------------------------------------+
|                       ENOS/                                       |
|        MASTER         OPENVPN_NODE      CONTROL        NET.       |
|   +--------------+   +-----------+     +-------+     +-------+    |
|   |              |   |           |     |       |     |       |    |
|   |              |   |           |     |       |     |       |    |
|   | enos openvpn |   |           |     |       |     |       |    |
|   |              |   |           |     +-------+     +-------+    |
|   |              |   +-----------+                +---------------+
|   |              |                       C1       |
|   +--------------+                     +-------+  |
|               ^                        |       |  |
|               |                        |       |  |
|               |                        |       |  |
|               |                        +-------+  |
+---------------------------------------------------+
MINIMUM         |                         x.x.x.x
INSTALL         |                        +-------+
                |                        |       |
                +------------------------+       |
                    request addition     |       |
                       to openvpn        +-------+
                     and openstack
```
<figure>
<figcaption>
<span class="figure-number">Figure 1: </span>
Schematic representation of the architecture of enos openvpn and a node requesting to join the infrastructure. NET. is the network node, C1 the first compute. MASTER is the node executing enos openvpn commands, and ENOS/OPENVPN_NODE the one that manages Enos deployment and OpenVPN server.
</figcaption>
</figure>

## Mechanisms ##

We will now see how Enos OpenVPN is used.

### Basic configuration ###

You can use running machines (<a href="#host-file">below</a>) or get some resources and deploy on [Grid'5000](https://www.grid5000.fr/w/Grid5000:Home) using the deploy command (<a href="#g5k-deploy">Get resources and deploy everything needed on g5k</a>)

#### Configure the hosts that will be used (without G5K) ####
<a name="host-file" id="host-file"></a>

Put the servers' addresses in the *current/hosts* as follow (assuming you have respectively an Enos/OpenVPN node, a control node (Openstack control), a network node and a compute):

``` bash
me@computer:~/enos_openvpn$ cat current/hosts
econome-20.nantes.grid5000.fr
econome-21.nantes.grid5000.fr
econome-22.nantes.grid5000.fr
econome-3.nantes.grid5000.fr
```
If you need to put Enos/OpenVPN node on the same node as the master, just put localhost in the first line of the file.

**Ensure that you can connect as root via SSH on each of these server.**

#### Get resources and deploy everything needed on G5K ####
<a name="g5k-deploy" id="g5k-deploy"></a>

If you are using Grid'5000, use the *deploy* command to prepare the hosts:
``` bash
usage: eov deploy [options]

Claim resources from G5k and launch the deployment

Options:
    -n, --xp-name NAME               Name of the experiment [default: enos_openvpn]
    -w, --walltime WALLTIME          Length, in time, of the experiment [default: 08:00:00]
    -c, --cluster CLUSTER            Cluster to deploy onto [default: ecotype]
    -r, --reservation RESERVATION    When to make the reservation (format is 'yyyy-mm-dd hh:mm:ss')
    --nodes NUMBER                   Number of nodes [default: 5]
```
So, to deploy an experience of adding a node to the infrastructure of 6 paravance nodes for 2 hours on October 26th, 1985 at 1:21AM, you will enter the command:

```
python eov.py deploy -n 'adding-node' --nodes 6 -c paravance -w 02:00:00 -r '1985-26-10 01:21:00'
```
(note that G5K can't go back to the future, so you will probably get an error for the date)


#### OpenVPN ####

To put OpenVPN in place on the hosts, simply use the `openvpn` command:

``` bash
Usage: eov openvpn

Deploy openvpn on resources from current/hosts
```

### Configure Enos ###

We need to create a Python virtual environment on a service node, install Enos and its dependencies and configure a reservation.yaml file for Enos.

##### reservation.yaml file #####

Configure the reservation.yaml that uses the [*static*](https://enos.readthedocs.io/en/stable/provider/static.html) provider. You have to specify the nodes of your infrastructure (controllers nodes, network nodes, compute nodes) in the reservation.yaml file. Keep in mind that the IP addresses should be IPs of the management, which by convention is "11.8.0.0/24".

### Run Enos ###

Once all the previous steps have been **successfully** completed, simply run the following command:

``` bash
Usage: eov enos [options]

Deploy enos on hosts

Options:
    --g5k              Deploying on g5k [default: false]

```

If you want to deploy in "production", you might want to change the passwords in Enos ([password file used by Kolla](https://github.com/BeyondTheClouds/enos/blob/master/enos/ansible/roles/bootstrap_kolla/files/passwords.yml))

### (Bonus) fix live migrations ###

To enable live migrations, the hosts files of nova-libvirt containers must contain the addresses of all compute nodes of the OpenStack infrastructure that you deployed.

To do so, run the following script:

``` bash
bash fix_live_migration.sh
```

### Add a node ###


On the node where you deployed enos openvpn, use:
``` bash
export FLASK_APP=eov.py
flask run --host=0.0.0.0
```
This will allow the node to accept requests from joining computes.

Run the node normally.

(To get another one on G5K, you can use for example:
``` bash
oarsub -I -l nodes=1,walltime=<TIME_OF_NODE_RUNNING> -p "cluster='<CLUSTER>'" -t deploy
kadeploy3 -e debian9-x64-nfs -f $OAR_NODE_FILE -k
```
)

Then ssh on it.

If the node is not on G5K, you will need to get the public key from the "master" node:
``` bash
curl http://<MASTER_IP>:5000/ >> .ssh/authorized_keys
```
This will get the public key from the master node and add it to autorized keys.


Then, you just have to request to be added to the OpenStack with:
``` bash
http://<MASTER_IP>:5000/addnode/<G5K>/<NODE_IP>
```
where:
* `MASTER_IP` is the ip of the master node (where you have ran enos openvpn, you can get it with `ip a` on the master node).
* `G5K` is a boolean (True or False), whether the node is on G5K or not
* `NODE_IP` is the ip of the node you want to add

And voilà. The node is on the VPN network and the infrastructure as a compute.

#### Alternative: client commands ####

First, execute the following commands on the new node:

``` bash
apt update; apt install git python-pip; git clone https://github.com/badock/enos_openvpn.git ; cd enos_openvpn; pip install -r requirements.txt
```

You have access to the client.py file, that has its own API.
Second, `add_ssh` command:
``` bash
Usage: client add_ssh MASTER

Add master ssh key to the node. MASTER is the ip of the master node
```

Then, `openvpn`:
``` bash
Usage: client openvpn MASTER [options]

Connect to the openvpn network. MASTER is the ip of the master node

Options:
    -n, --name NAME     Name of the node to add [default: new_node]
```

Finally, you can use `enos` to add the node, remove it or rejoin the infrastructure:
``` bash
Usage: client enos MASTER [options]

Join the existing Openstack. MASTER is the ip of the master node

Options:
    -n, --name NAME      Name of the node to act on [default: new_node]
    --g5k                Specify if the node is on g5k [default: False]
    -a, --action ACTION  Action to run (add, remove or rejoin)
```

For example, to add `ecotype-47.nantes.grid5000.fr` to an infrastructure which has a master with ip `192.31.58.160`:
``` bash
python client.py add_ssh 192.31.58.160:5000
python client.py openvpn 192.31.58.160:5000 -n ecotype-47.nantes.grid5000.fr
python client.py enos 192.31.58.160:5000 -a add --g5k -n ecotype-47.nantes.grid5000.fr
```

## Note ##

There is currently a version being worked on to enable a node to autojoin the infrastructure without having to contact any node.
