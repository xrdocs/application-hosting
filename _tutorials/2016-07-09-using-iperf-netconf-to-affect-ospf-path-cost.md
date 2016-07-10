---
published: true
date: '2016-07-09 23:47 -0700'
title: Using iperf + netconf to affect OSPF path cost
position: hidden
author: Akshat Sharma
excerpt: OSPF path remediation using container based iperf and netconf
tags:
  - vagrant
  - iosxr
  - cisco
  - linux
  - iperf
  - ospf
  - netconf
---

{% include toc icon="table" title="Launching a Container App" %}
{% include base_path %}
  

## Introduction

If you haven't checked out the XR toolbox Series, then you can do so here:  

>
[XR Toolbox Series]({{ base_path }}/tags/#xr-toolbox)

  
This series is meant to help a beginner get started with application-hosting on IOS-XR.
In this tutorial we intend to utilize techniques learnt in the above series to solve a path remediation problem:  
  
*  Set up a couple of paths between two routers. In this example we connect the routers back-to-back and set up ECMP (equal cost multipath) links between them. One interface is forcefully selected as the reference link based on OSPF cost configuration.
*  Use a monitoring technique to determine the bandwidth, jitter, latency etc. parameters along the traffic path. In this example we use iperf running inside a container.
*  Simulate network degradation (through link flaps) to test a python app that uses iperf's results to detect the flaps and cause failover by changing the OSPF path cost over netconf.
