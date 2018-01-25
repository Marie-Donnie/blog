---
layout: post
title: Gap Analysis of the OpenStack codebase w.r.t Edge Computing Challenges - A first step
date: 2018-01-23
categories: OpenStack
author: ad_ri3n_
---

With the next [OpenStack PTG](https://www.openstack.org/ptg/) in mind,
we conducted a first gap analysis of the OpenStack code base with
respect to the Edge Computing Challenges.  The outputs of this
brainstorming session, which has been organized with two folks from
Ericsson, are:
* The identification and classification of a couple of
  features we believe edge administrators/developers may
  expect. Concretely, we define 7 levels of features, from the simplest
  such as being able to start a VM on a specific location, to the most
  advanced ones, including the interoperability of different cloud
  stacks (OpenStack/Kubernetes/...).
* An analysis of different deployment scenarios the OpenStack code
  base provides (cells, regions, federations...)  w.r.t the
  aforementioned levels.

Obviously, this study is an ongoing action that should be refined with
the expertise of additional folks, but at least we started it!
Further information available at
[https://etherpad.openstack.org/p/edge-gap-analysis](https://etherpad.openstack.org/p/edge-gap-analysis).








