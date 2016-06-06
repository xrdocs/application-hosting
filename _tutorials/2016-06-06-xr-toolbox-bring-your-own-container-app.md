---
published: true
date: '2016-06-06 00:13 -0700'
title: 'XR Toolbox: Bring your own Container App'
author: Akshat Sharma
excerpt: Bring up your own Container (LXC) + Application on IOS-XR
tags:
  - vagrant
  - iosxr
  - cisco
  - linux
  - containers
  - LXC
  - iperf
---

{% include base_path %}
{% include toc icon="table" title="Bring your own Container" %}


## Introduction

The Techdoc: [Application Hosting on IOS-XR]({{ base_path}}/techdocs/app_hosting_on_iosxr/introduction) dives deep into the IOS-XR architecture, to help explain how a user can deploy an application:  

*  natively (inside the XR process space) OR
*  as a container (LXC)


In this quick start guide we use the IOS-XR vagrant box to bring up an Ubuntu container on IOS-XR and host an application within it.

So, let's get started!


## Pre-requisites

* Meet the pre-requisites specified in the [IOS-XR Vagrant Quick Start guide: Pre-requisites]({{ base_path }}/tutorials/iosxr-vagrant-quickstart#pre-requisites) 
* Clone the following repository: <https://github.com/akshshar/vagrant-xr>, before we start.

```shell
cd ~/
git clone https://github.com/akshshar/vagrant-xr.git
cd vagrant-xr/
```

You will notice a few directories. We will utilize the `lxc-app-topo-bootstrap` directory in this tutorial.
{: .notice--info}

```shell
AKSHSHAR-M-K0DS:vagrant-xr akshshar$ pwd
/Users/akshshar/vagrant-xr
AKSHSHAR-M-K0DS:vagrant-xr akshshar$ ls lxc-app-topo-bootstrap/
Vagrantfile	configs		scripts
AKSHSHAR-M-K0DS:vagrant-xr akshshar$ 
``` 
 


## Understand the topology

For this tutorial, we'll use a two-node topology: An XR vagrant instance connected to an Ubuntu vagrant instance:

![LXC app bootstrap topo](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/lxc-app-bootstroop-topo.png)


The Vagrantfile to bring up this topology is already in your cloned directory:  

`vagrant-xr/lxc-app-topo-bootstrap/Vagrantfile`

```ruby

Vagrant.configure(2) do |config|
  

 config.vm.define "rtr" do |node|
   node.vm.box =  "IOS-XRv"

   # gig0/0/0 connected to "link1"
   # auto_config is not supported for XR, set to false

   node.vm.network :private_network, virtualbox__intnet: "link1", auto_config: false


   #Source a config file and apply it to XR

   config.vm.provision "file", source: "configs/rtr_config", destination: "/home/vagrant/rtr_config"

   config.vm.provision "shell" do |s|
     s.path =  "scripts/apply_config.sh"
     s.args = ["/home/vagrant/rtr_config"]
   end

 end

 
 config.vm.define "devbox" do |node|
   node.vm.box =  "ubuntu/trusty64"

   # eth1 connected to link1
   # auto_config is supported for an ubuntu instance

   node.vm.network :private_network, virtualbox__intnet: "link1", ip: "11.1.1.20"

 end

end
```

**Notice the   `#Source a config file and apply it to XR` section of the Vagrantfile? This is derived from the [Bootstrap XR configuration with Vagrant]({{ base_path }}/tutorials/iosxr-vagrant-bootstrap-config) tutorial. Check it out if you want to know more about how shell provisioning with XR works**
{: .notice--warning}

>
For now, all you need to know is:
>
*  Config that you want to apply to XR on boot goes into the `lxc-app-topo-bootstrap/configs` directory. It is named  "rtr_config" in this example.
*  The script to apply XR config resides in the `lxc-app-topo-bootstrap/scripts` directory. You will not need to bother about it in this tutorial.
{: .notice--info}



The configuration we wish to apply to XR is pretty simple. We want to configure the XR interface: `GigabitEthernet0/0/0/0`s with the ip-address: `11.1.1.10`:

```shell
AKSHSHAR-M-K0DS:vagrant-xr akshshar$ cat lxc-app-topo-bootstrap/configs/rtr_config 
!! XR configuration
!
interface GigabitEthernet0/0/0/0
  ip address 11.1.1.10/24
  no shutdown
!
end
AKSHSHAR-M-K0DS:vagrant-xr akshshar$ 
```

Take a look at the Vagrantfile above, again. We use the Vagrant auto_config capabilities to make sure "eth1" interface of the Ubuntu VM (called devbox) is configured in the same subnet (11.1.1.20) as XR gig0/0/0/0.
{: .notice--info}
















