---
published: true
date: "2016-05-22 13:49 -0700"
title: "Getting Started with IOS-XR Vagrant"
permalink: "/tutorials/iosxr-vagrant-quickstart"
author: Lisa Roach
excerpt: "Getting started with Cisco's XRv64 Vagrant Instance."
tags: 
  - vagrant
  - iosxr
  - cisco
  - XRv64
position: hidden
---

{% include toc icon="table" title="IOS-XR Vagrant: Quick Start" %}
{% include base_path %}

## Introduction

This tutorial is meant to be a quick-start guide to get you up and running with an IOS-XRv Vagrant box.

If you're unfamiliar with Vagrant as a tool for development, testing and design, then here's a quick look at why Vagrant is useful, directly from the folks at Hashicorp:

><https://www.vagrantup.com/docs/why-vagrant/>

For a more detailed walkthrough of Vagrant with IOS-XR, along with examples of topologies and configuration, take a look at the ["XR toolbox" series]({{ base_path }}/tags/#xr-toolbox)
{: .notice--info}


## Pre-requisites:
* [Vagrant](https://www.vagrantup.com/downloads.html) for your Operating system
* [Virtualbox](https://www.virtualbox.org/wiki/Downloads) installed on your laptop
* A laptop with atleast 4-5G free RAM. (Each XR vagrant instance uses upto 4G RAM, so plan ahead based on the number of XR nodes you want to run)



Tha above items are applicable to all operating systems - Mac OSX, Linux or Windows.  

If you're using Windows, we would urge you to download a utility like [Git Bash](https://git-scm.com/download/win) so that "vagrant ssh" works out of the box.
{: .notice--danger}


## Single Node Bringup

### Download and Add the IOS-XRv vagrant box
This can be achieved with a single command as follows:

```
BOX_URL = "http://engci-maven-master.cisco.com/artifactory/simple/appdevci-snapshot/XRv64/latest/iosxrv-fullk9-x64.box"

vagrant box add --name IOS-XRv $BOX_URL

```

### Initialize a Vagrantfile

Let's create a working directory (any name would do) for our next set of tasks:

```
mkdir ~/iosxrv; cd ~/iosxrv
```

Now, in this directory, let's initialize a Vagrantfile with the name of the box we added.
 

```
vagrant init IOS-XRv
  
```

### Bring up the Vagrant Instance

A simple `vagrant up` will bring up the XR instance

```
vagrant up 
```

This bootup process will take some time, (close to 5 minutes).
{: .notice--info}


Look for the green "vagrant up" welcome message to confirm the machine has booted:
	
     
    
   

Now we have two options to access the Vagrant instance:

* **Access the XR Linux shell**:   
  
 
```
vagrant ssh
```
   

* **Access XR Console**:
XR SSH runs on port 22 of the guest IOS-XR instance.  
First, determine the port to which the XR SSH (port 22) is forwarded by vagrant by using the `vagrant port` command:
  
  
<div class="highlighter-rouge">
<pre class="highlight">
<code>
AKSHSHAR-M-K0DS:~ akshshar$ <mark> vagrant port </mark>
The forwarded ports for the machine are listed below. Please note that
these values may differ from values configured in the Vagrantfile if the
provider supports automatic port collision detection and resolution.

    <mark>22 (guest) => 2223 (host) </mark>
 57722 (guest) => 2222 (host)
</code>
</pre>
</div>



As shown above, port 22 of XR is fowarded to port 2223:


Use port 2223 to now ssh into XR CLI
    
```
ssh -p 2223 vagrant@localhost
```
    
The password is "vagrant"
{: .notice--info}



## Multi Node Bringup


Let's try to bring up a multi-node topology as shown below:
![Topology](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/xrv64_topo_m.png)

### Set up the Vagrantfile
For this purpose, Let's use a Sample vagrantfile located here:  
<https://github.com/akshshar/vagrant-xr/blob/master/simple-mixed-topo/Vagrantfile>

```
git clone https://github.com/akshshar/vagrant-xr.git
cd vagrant-xr/simple-mixed-topo
```

Shown below is a snippet of the Vagrantfile:

```
Vagrant.configure(2) do |config|

    config.vm.define "rtr1" do |node|
      node.vm.box =  "IOS-XRv"

      # gig0/0/0/0 connected to link2, 
      # gig0/0/0/1 connected to link1, 
      # gig0/0/0/2 connected to link3,
      # auto-config not supported.

      node.vm.network :private_network, virtualbox__intnet: "link2", auto_config: false
      node.vm.network :private_network, virtualbox__intnet: "link1", auto_config: false
      node.vm.network :private_network, virtualbox__intnet: "link3", auto_config: false 

    end

    config.vm.define "rtr2" do |node|
      node.vm.box =  "IOS-XRv"

      # gig0/0/0/0 connected to link1,
      # gig0/0/0/1 connected to link3, 
      # auto-config not supported

      node.vm.network :private_network, virtualbox__intnet: "link1", auto_config: false
      node.vm.network :private_network, virtualbox__intnet: "link3", auto_config: false

    end

```

If you compare this with the topology above it becomes pretty clear how the interfaces of the XR instances are mapped to individual links.

**The order of the "private_networks" is important. For each XR node, the first "private_network" corresponds to gig0/0/0/0, the second "private_network" to gig0/0/0/1 and so on.**
{: .notice--warning}

### Bring up the topology
As before, we'll issue a `vagrant up` to bring up the topology.

```
vagrant up
``` 

This will take some time, possibly over 10 minutes. 
{: .notice--warning}

Look for the green "vagrant up" welcome message to confirm the three machines have booted:
	
   ![vagrant up- multi](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/xrv64_vagrant_up_m.png)
   
 Post up message:
   ![post up message](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/xrv64_post_up_m.png)


### Access the nodes

The only point to remember is that in a multinode setup, we "name" each node in the topology.
{: .notice--info}

For example, let's access "rtr2"

* Access the XR Linux shell:

```
vagrant ssh rtr2 
```        
![vagrant ssh- multi](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/xrv64_vagrant_ssh_1_m.png)

* Access XR Console:

Determine the forwarded port for port 22 (XR SSH) for rtr2:

```
vagrant port rtr2

```
For rtr2 port 22, the forwarded port is 2201. So, to get into the XR CLI of rtr2, we use:

```
ssh -p 2201 vagrant@localhost
```

![ssh console- multi](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/xrv64_ssh_console_m.png)
