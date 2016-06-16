---
published: true
date: '2016-06-16 03:17 -0500'
title: 'XR toolbox, Part 4: Deploy a Container (LXC) App on IOS-XR'
author: Akshat Sharma
tags:
  - vagrant
  - iosxr
  - cisco
  - linux
  - lxc
  - containers
position: hidden
excerpt: Launch a Container app (LXC) on IOS-XR
---

{% include toc icon="table" title="Launching a Container App" %}
{% include base_path %}

## Introduction

If you haven't checked out the earlier parts to the XR toolbox Series, then you can do so here:  

>
[XR Toolbox Series]({{ base_path }}/tags/#xr-toolbox)

  
The purpose of this series is simple. Get users started with an IOS-XR setup on their laptop and incrementally enable them to try out the application-hosting infrastructure on IOS-XR.

In this part, we explore how a user can build and deploy their own container (LXC) based applications on IOS-XR.


## Pre-requisites

Before we begin, let's make sure you've set up your development environment.
If you haven't checked it out, go through the "App-Development Topology" tutorial here:  

[XR Toolbox, Part 3: App Development Topology]({{ base_path }}/tutorials/2016-06-06-xr-toolbox-app-development-topology)  
  
Follow the instructions to get your topology up and running as shown below:  

![app dev topo](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/app_dev_topology.png)


If you've reached the end of the above tutorial, you should be able to issue a `vagrant status` to see a rtr (IOS-XR) and a devbox (Ubuntu) instance running.  

```shell
AKSHSHAR-M-K0DS:lxc-app-topo-bootstrap akshshar$ pwd
/Users/akshshar/vagrant-xr/lxc-app-topo-bootstrap
AKSHSHAR-M-K0DS:lxc-app-topo-bootstrap akshshar$ 
AKSHSHAR-M-K0DS:lxc-app-topo-bootstrap akshshar$ vagrant status
Current machine states:

rtr                       running (virtualbox)
devbox                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
AKSHSHAR-M-K0DS:lxc-app-topo-bootstrap akshshar$ 

```

All good? Perfect, let's start building our container application tar ball.
{: .notice--success}  


## Build an LXC tar ball











