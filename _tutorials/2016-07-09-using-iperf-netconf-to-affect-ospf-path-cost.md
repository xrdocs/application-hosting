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
  
  
**In this tutorial** we intend to utilize techniques learnt in the above series to solve a path remediation problem:  
  
*  Set up a couple of paths between two routers. In this example we connect the routers back-to-back and set up ECMP (equal cost multipath) links between them. One interface is forcefully selected as the reference link based on OSPF cost configuration.  

*  Use a monitoring technique to determine the bandwidth, jitter, latency etc. parameters along the traffic path. In this example we use iperf inside a container.  

*  Simulate network degradation (through link flaps) to test a python app (inside the LXC) that uses iperf's results to detect the flaps and causes failover by changing the OSPF path cost over a netconf session.  

This is illustrated below:  

![iperf-ospf-ncclient-demo](https://camo.githubusercontent.com/a30938cc2dd9c0788b701677fbb5398bc5bb6646/68747470733a2f2f7872646f63732e6769746875622e696f2f7872646f63732d696d616765732f6173736574732f7475746f7269616c2d696d616765732f6f7370665f6e635f69706572662e6a7067)  


## Create the container App (tar ball)  

As shown in the above figure, we intend to create an application running inside a container on IOS-XR.  

We're fully aware that most users would like to run things on their own laptop as they develop applications and test them.   
&nbsp;  
Owing to the slightly beefy requirements of IOS-XRv vagrant instances, running more than two vagrant instances may be a problem for most users.   
So, we'll create our container app in a development vagrant instance first, save the app and then destroy the development instance before proceeding with the router topology.
{: .notice--info}


  
  
  
  
