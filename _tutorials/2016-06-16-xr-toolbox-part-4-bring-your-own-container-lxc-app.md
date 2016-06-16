---
published: true
date: '2016-06-16 03:17 -0500'
title: 'XR toolbox, Part 4: Bring your own Container (LXC) App'
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


If you've reached the end of the above tutorial, you should be able to issue a `vagrant status` in the `vagrant-xr/lxc-app-topo-bootstrap` directory to see a rtr (IOS-XR) and a devbox (Ubuntu/trusty) instance running.  

<div class="highlighter-rouge">
<pre class="highlight">
<code>
AKSHSHAR-M-K0DS:lxc-app-topo-bootstrap akshshar$ pwd
<mark>/Users/akshshar/vagrant-xr/lxc-app-topo-bootstrap </mark>
AKSHSHAR-M-K0DS:lxc-app-topo-bootstrap akshshar$ 
AKSHSHAR-M-K0DS:lxc-app-topo-bootstrap akshshar$<mark> vagrant status </mark>
Current machine states:

rtr                       running (virtualbox)
devbox                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
AKSHSHAR-M-K0DS:lxc-app-topo-bootstrap akshshar$ 

</code>
</pre>
</div>


All good? Perfect. Let's start building our container application tar ball.
{: .notice--success}  


## Create a container App  
  
To launch an LXC container we need two things:  

*  A container rootfs tar ball
*  An XML file to launch the container using `virsh/libvirt`

To create them, we'll hop onto our devbox (Ubuntu/trusty) VM in the topology and install lxc-tools. lxc-tools will be used to create a container rootfs tar ball.  


### Install lxc tools on devbox  

SSH into the devbox:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
AKSHSHAR-M-K0DS:lxc-app-topo-bootstrap akshshar$<mark> vagrant ssh devbox</mark>
Welcome to Ubuntu 14.04.4 LTS (GNU/Linux 3.13.0-87-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

  System information as of Thu Jun 16 14:27:47 UTC 2016

  System load:  0.0               Processes:           74
  Usage of /:   3.5% of 39.34GB   Users logged in:     0
  Memory usage: 25%               IP address for eth0: 10.0.2.15
  Swap usage:   0%                IP address for eth1: 11.1.1.20

  Graph this data and manage this system at:
    https://landscape.canonical.com/

  Get cloud support with Ubuntu Advantage Cloud Guest:
    http://www.ubuntu.com/business/services/cloud

0 packages can be updated.
0 updates are security updates.


Last login: Thu Jun 16 14:27:47 2016 from 10.0.2.2
vagrant@vagrant-ubuntu-trusty-64:~$ 
</code>
</pre>
</div> 

  
Install lxc tools inside the devbox

```shell
sudo apt-get update
sudo apt-get -y install lxc
```  

Check that lxc was properly installed:

```shell
vagrant@vagrant-ubuntu-trusty-64:~$ lxc-start --version
1.0.8
vagrant@vagrant-ubuntu-trusty-64:~$ 
```






















