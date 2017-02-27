---
published: true
date: '2017-02-26 14:40 -0800'
title: Running Docker Containers on IOS-XR (6.1.2+)
position: hidden
author: Akshat Sharma
excerpt: >-
  Running Docker containers on IOS-XR running as a Vagrant box and on a physical
  NCS5500
tags:
  - vagrant
  - iosxr
  - NCS5500
  - docker
  - xr toolbox
---


{% include toc icon="table" title="Running Docker Containers on IOS-XR" %}
{% include base_path %}

## Introduction

If you haven't checked out the earlier parts to the XR toolbox Series, then you can do so here:  

>
[XR Toolbox Series]({{ base_path }}/tags/#xr-toolbox)

  
The purpose of this series is simple. Get users started with an IOS-XR setup on their laptop and incrementally enable them to try out the application-hosting infrastructure on IOS-XR.

In this part, we explore how a user can spin up Docker containers on IOS-XR. There are multiple ways to do this and we'll explore each one:  

*  **Dockerhub**: Set up reachability from your router (Virtual or physical) to the internet (or specifically to dockerhub: <https://hub.docker.com>).  

*  **Private "secure" registry**: Just set up reachability to your private registry set up with a certificate obtained from a CA. We won't really tackle this scenario separately in this tutorial - once you know how to gain access to dockerhub, the workflow will be similar.  
  
*  **Private "insecure" registry**: Some users may choose to do this, specially if they're running a local docker registry inside a secured part of their network.  
   
*  **Private "self-signed" registry**: This is more secure than the "insecure" setup, and allows a user to enable TLS.
    
*  **Tarball image/container**:  This is the simplest setup - very similar to LXC deployments. In this case, a user may create and set up a container completely off-box, package it up as an image or a container tar ball, transfer it to the router and then load/import it, before running.  


For each case, we will compare IOS-XR running as a Vagrant box with IOS-XR running on a physical box (NCS5500). They should be identical, except for reachability through the Management ports.  
  
  
## Pre-requisite  

### Vagrant IOS-XR box

If you're bringing up the topology on your laptop using the IOS-XR vagrant box, then:

* Meet the pre-requisites specified in the [IOS-XR Vagrant Quick Start guide: Pre-requisites]({{ base_path }}/tutorials/iosxr-vagrant-quickstart#pre-requisites).   
The topology here will require about 5G RAM and 2 cores on the user's laptop.
* Clone the following repository: <https://github.com/ios-xr/vagrant-xrdocs>, before we start.

```shell
cd ~/
git clone https://github.com/ios-xr/vagrant-xrdocs.git
cd vagrant-xrdocs/
```  

You will notice a few directories. We will utilize the `docker-app-topo-bootstrap` directory in this tutorial.
{: .notice--info}

```shell
AKSHSHAR-M-K0DS:vagrant-xrdocs akshshar$ pwd
/Users/akshshar/vagrant-xrdocs
AKSHSHAR-M-K0DS:vagrant-xrdocs akshshar$ ls docker-app-topo-bootstrap/
Vagrantfile	configs		scripts
AKSHSHAR-M-K0DS:vagrant-xrdocs akshshar$ 

```


### Physical (NCS5500 router)  

On the other hand, if you have an NCS5500 lying around (don't we all?), then load up a 6.1.2+ image on the router and connect an Ubuntu server (for the purpose of this tutorial), to the Management network of the router.

The server needs to be reachable from the router over the Management network.



## Understand the topology

The topology I'm using differs slightly between the vagrant setup and the NCS5508 setup I have.
This is owing to the fact that the Management port of the vagrant IOS-XR box is used up in the NAT network. So to show equivalence between the two setups, I directly connect the Gig0/0/0/0 interface of Vagrant ios-xrv64 with eth1 of the devbox as shown in the Vagrantfile.  

The two topologies in use are shown below:  


![vagrant docker topo](https://xrdocs.github.io/xrdocs-images/assets/images/vagrant_docker_topo.png)  
{: notice--warning}
  
    
![NCS5500 docker topo](https://xrdocs.github.io/xrdocs-images/assets/images/ncs5500_docker_topo.png)    
{: notice--warning}






   
   
   
