---
layout: post
title:  "Two More Papers Have Been Accepted (CCGRID and ICFEC 2018)"
date:   2018-02-10
categories: publications
author: Adrien Lebre
---

## Towards Lazy and Locality-Aware Overlays for Decentralized Clouds

*Genc Tato, Marin Bertier, and Cédric Tedeschi*

Current cloud computing infrastructures and their management are highly centralized, and therefore they suffer from limitations in terms of network latency, energy consumption, and possible legal restrictions. Decentralizing the Cloud has been recently proposed as an alternative. However, the efficient management of a geographically dispersed platform brings new challenges related to service localization, network utilization and locality-awareness. We here consider a cloud topology composed of many small datacenters geographically dispersed within the backbone network.

In this paper, we present the design, development and experimental validation of Koala, a novel overlay network that specifically targets such a geographically distributed cloud platform. The three key characteristics of Koala are laziness, latency-awareness and topology-awareness.

By using application traffic, Koala maintains the overlay lazily while it takes locality into account in each routing decision. Although Koala’s performance depends on application traffic, through simulation experiments we show that for a uniformly distributed traffic, Koala delivers similar routing complexity and reduced latency compared to a traditional proactive protocol, such as Chord. Additionally, we show that despite its passive maintenance, Koala can appropriately deal with churn by keeping the number of routing failures low, without significantly degrading the routing performance. Finally, we show how such an overlay adapts to a decentralized cloud composed of multiple small datacenters.

## Nitro: Network-Aware Virtual Machine Images Management in Geo-Distributed Clouds

*By Jad Darrous, Shadi Ibrahim, Amelie Chi Zhou, and Christian Perez*

Recently, most large cloud providers, like Amazon and Microsoft, replicate their Virtual Machine Images (VMIs) on multiple geo-graphically distributed data centers to offer fast service provisioning. Provisioning a service may require to transfer a VMI over the wide-area network and there- fore is dictated by the distribution of VMIs and the network bandwidth in-between sites. Nevertheless, existing methods to facilitate VMIs management (i.e., retrieving VMIs) overlook network heterogeneity in geo-distributed clouds.

In this paper, we design, implement and evaluate Nitro, a novel VMI management system that helps to minimize the transfer time of VMIs over a heterogeneous WAN. To achieve this goal, Nitro incorporates two complementary features. First, it makes use of deduplication to reduce the amount of data which will be transferred due to the high similarities within an image and in-between images. Second, Nitro is equipped with a network-aware data transfer strategy to effectively exploit links with high bandwidth when acquiring data and thus expedites the provisioning time. Our results show that the network-aware data transfer strategy offers optimal solution when acquiring VMIs while introducing minimal overhead. Moreover, Nitro outperforms state-of-the-art VMI storage systems (e.g., Openstack Swift) by up to 77%.
