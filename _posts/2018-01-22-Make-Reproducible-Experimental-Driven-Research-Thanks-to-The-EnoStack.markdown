---
layout: post
title: Make Reproducible Experimental-Driven Research Thanks to The EnoStack 
date: 2018-01-22
categories: EnosStack
author: avankemp
---

Experimental studies are an integral part of the Inria's Discovery initiative as we strongly believe that the solutions we design have to be validated in practical contexts. However, designing, deploying and validating an experiment can quickly become a tedious and time-consuming endeavor, particularly when targeting large scale and real testbeds (e.g. Chameleon, Grid’5000, etc.). Rings a bell... ?

In order to help experimenters in this daunting task, we created the EnosStack, an open source software stack especially designed for reproducible scientific experiments. It enables an experimenter to easily  describe experimental  workflows meant to be re-used, while  abstracting  the  underlying  infrastructure  running  them. In other words, an experimenter can switch from  a  local  to  a  real  testbed deployment with very few effort, so that the code development and validation time is greatly reduced.


We are actually using the EnosStack in the context of the OpenStack performance working group to help us conducting the numerous experiments of this [test plan](https://review.openstack.org/#/c/491818/).
We just published a report detailing its concepts and its implementation details, if you are interested it's [there](https://hal.inria.fr/hal-01689726).
Hoping that the EnosStack will save some of your time too !







