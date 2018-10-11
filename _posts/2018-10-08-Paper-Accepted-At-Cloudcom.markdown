---
layout: post
title:  "Paper Have Been Accepted at Cloudcom 2018"
date:   2018-10-08
categories: publications
author: Javier Rojas Balderrama
---

## Scalability and Locality Awareness of Remote Procedure Calls: An Experimental Study in Edge Infrastructures

*Javier Rojas Balderrama, and Matthieu Simonin*

Cloud computing depends on communication mechanisms implying location
transparency. Transparency is tied to the cost of ensuring scalability and an
acceptable request responses associated to the locality. Current
implementations, as in the case of OpenStack, mostly follow a centralized
paradigm but they lack the required service agility that can be obtained in
decentralized approaches.

In an edge scenario, the communicating entities of an application can be
dispersed. In this context, we focus our study on the inter-process
communication of OpenStack when its agents are geo-distributed. More precisely,
we are interested in the different Remote Procedure Calls (RPCs) implementations
of OpenStack and their behaviours with regards to three classical communication
patterns: anycast, unicast and multicast. We discuss how the communication
middleware can align with the geo-distribution of the RPC agents regarding two
key factors: scalability and locality. We reached up to ten thousands
communicating agents, and results show that a router-based deployment offers a
better trade-off between locality and load-balancing. Broker-based suffers from
its centralized model which impact the achieved locality and scalability.

*Paper available on [HAL](https://hal.inria.fr/hal-01891567).*
