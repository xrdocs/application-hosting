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

*  **Private "secure" registry**: Set up reachability to your private registry, created using a certificate obtained from a CA. We won't really tackle this scenario separately in this tutorial - once you know how to gain access to dockerhub, the workflow will be similar.  
  
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

The topology I'm using differs slightly between the vagrant setup and the NCS5508 setup.
This is owing to the fact that the Management port of the vagrant IOS-XR box is used up in the NAT network. So to show equivalence between the two setups, I directly connect the Gig0/0/0/0 interface of Vagrant ios-xrv64 with eth1 of the devbox as shown in the figure below.  

The two topologies in use are:  


![vagrant docker topo](https://xrdocs.github.io/xrdocs-images/assets/images/vagrant_docker_topo.png)  
{: .notice}
  
    
![NCS5500 docker topo](https://xrdocs.github.io/xrdocs-images/assets/images/ncs5500_docker_topo.png)    
{: .notice}  
  
  
  
## Install docker-engine on the devbox

### Vagrant setup
For the Vagrant setup, you will see a script called `docker_install.sh` under the scripts folder:  

```shell
AKSHSHAR-M-K0DS:docker-app-topo-bootstrap akshshar$ pwd
/Users/akshshar/vagrant-xrdocs/docker-app-topo-bootstrap
AKSHSHAR-M-K0DS:docker-app-topo-bootstrap akshshar$ ls
Vagrantfile	configs		scripts
AKSHSHAR-M-K0DS:docker-app-topo-bootstrap akshshar$ ls scripts/
apply_config.sh		docker_install.sh
AKSHSHAR-M-K0DS:docker-app-topo-bootstrap akshshar$ 

```

This is the vagrant provisioner for the devbox and will install docker-engine on boot (vagrant up).  


### NCS5508 setup

In this case, the devbox must be provisioned by the user. On an ubuntu devbox, docker-engine can be installed by following the instructions at:  
  
><https://docs.docker.com/engine/installation/linux/ubuntu/>  
  
    
    
Perfect! Now we're all set with the topology. Before we begin, let's understand the docker daemon/client setup inside IOS-XR.
{: .notice--success}  
  
  
## Docker Daemon support on IOS-XR

If you haven't already gone through the basic overview on the application hosting infrastructure on XR, I would urge you to have a quick read:  
  
><{{ base_path }}/blogs/2016-06-28-xr-app-hosting-architecture-quick-look/>  
  

From the above article it becomes fairly clear that internally the IOS-XR architecture involves a Host layer running the libvirtd daemon and IOS-XR runs as an LXC spawned using the daemon.  

Further, the "virsh" client is provided within the XR LXC, so that a user may have client level access to the daemon while sitting inside the XR LXC itself.  

The setup for launching LXCs in IOS-XR is shown below:  

[![xr-lxc](https://xrdocs.github.io/xrdocs-images/assets/images/xr_lxc.png)](https://xrdocs.github.io/xrdocs-images/assets/images/xr_lxc.png)  
  
  
  
The Docker client/daemon setup follows the exact same principle as shown below. Docker Daemon runs on the host and Docker client is made available inside the XR LXC for easy operationalization:  
  
[![xr-docker](https://xrdocs.github.io/xrdocs-images/assets/images/xr_docker.png)](https://xrdocs.github.io/xrdocs-images/assets/images/xr_docker.png)  

Can we verify this?

### Vagrant setup 

#### Docker Client Access: 
On your vagrant box, there are two ways to get access to the docker client:

*  Drop into the "bash" shell from XR CLI. Using "bash" ensures that the correct environment 
   variables are sourced to gain access to the Docker Daemon on the host:   

   **Password for the XR CLI: vagrant**
   {: .notice--info}  
   
  
    <div class="highlighter-rouge">
    <pre class="highlight">
    <code>
    AKSHSHAR-M-K0DS:docker-app-topo-bootstrap akshshar$<mark> vagrant port rtr </mark>
    The forwarded ports for the machine are listed below. Please note that
    these values may differ from values configured in the Vagrantfile if the
    provider supports automatic port collision detection and resolution.

         <mark>22 (guest) => 2223 (host)</mark>
     57722 (guest) => 2222 (host)
    AKSHSHAR-M-K0DS:docker-app-topo-bootstrap akshshar$ 
    AKSHSHAR-M-K0DS:docker-app-topo-bootstrap akshshar$ 
    AKSHSHAR-M-K0DS:docker-app-topo-bootstrap akshshar$ 
    AKSHSHAR-M-K0DS:docker-app-topo-bootstrap akshshar$<mark> ssh -p 2223 vagrant@localhost</mark>
    The authenticity of host '[localhost]:2223 ([127.0.0.1]:2223)' can't be established.
    RSA key fingerprint is SHA256:uHev9uiAa0LM36RnnxDYuRyKywra8Oe/G5Gt34OiBqk.
    Are you sure you want to continue connecting (yes/no)? yes
    Warning: Permanently added '[localhost]:2223' (RSA) to the list of known hosts.
    vagrant@localhost's password: 


    RP/0/RP0/CPU0:ios#
    RP/0/RP0/CPU0:ios#
    RP/0/RP0/CPU0:ios#<mark>bash</mark>
    Sun Mar  5 18:17:18.380 UTC

    [xr-vm_node0_RP0_CPU0:~]$
    [xr-vm_node0_RP0_CPU0:~]$
    [xr-vm_node0_RP0_CPU0:~]$<mark>docker ps</mark>
    CONTAINER ID    IMAGE      COMMAND      CREATED       STATUS        PORTS         NAMES
    [xr-vm_node0_RP0_CPU0:~]$
    </code>
    </pre>
    </div> 

*  Drop directly into the Linux shell over SSH (port 57722)

### NCS5508 setup