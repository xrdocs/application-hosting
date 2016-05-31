---
published: true
date: "2016-05-22 13:49 -0700"
title: "Getting Started on Vagrant IOS-XR"
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

{% include toc icon="table" title="@xrdocs How-To" %}
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

Tha above items are applicable to all operating systems - Mac OSX, Linux or Windows.  

If you're using Windows, we would urge you to download a utility like [Git Bash](https://git-scm.com/download/win) so that "vagrant ssh" works out of the box.
{: .notice--danger}


## Single Node Bringup

### Download and Add the IOS-XRv vagrant box
This can be achieved with a single command as follows:

```
BOX_URL = "http://engci-maven-master.cisco.com/artifactory/simple/appdevci-snapshot/XRv64/latest/iosxrv-fullk9-x64.box"

vagrant box add --name iosxr-box $BOX_URL

```

### Initialize a Vagrantfile

Let's create a working directory (any name would do) for our next set of tasks:

```
mkdir ~/iosxrv; cd ~/iosxrv
```

Now, in this directory, let's initialize a Vagrantfile with the name of the box we added.
 

```
vagrant init iosxrv-box
  
```

### Bring up the Vagrant Instance

A simple `vagrant up` will bring up the XR instance

```
vagrant up 
```

This bootup process will take some time, (close to 6 minutes).
{: .notice--info}


Look for the green "vagrant up" welcome message to confirm the machine has booted:
	
     
    
   

Now we have two options to access the Vagrant instance:

* **Access the XR Linux shell**:
  
  ```
  vagrant ssh
  ```
   

* **Access XR Console**:
XR SSH runs on port 22 of the guest IOS-XR instance.  
First, determine the port to which the XR SSH (port 22) is forwarded by vagrant:
    
    ```
    vagrant port
    ```
    As shown in the example below, port 22 of XR is fowarded to port 2223:
    
    
    
    Use port 2223 to now ssh into XR CLI
    
    ```
    ssh -p 2223 vagrant@localhost
    ```
    
    The password is "vagrant"
    {: .notice--info}



#### Multi Node Bringup:

Let's try to bring up a multi-node topology as shown below:
![Topology](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/xrv64_topo_m.png)

For this purpose, we will use a Vagrantfile

Add the box to vagrant: `vagrant box add --name IOS-XRv iosxrv-fullk9-x64.box --force;`
* Or point to a URL: `vagrant box add --name xrv64 <REPLACE THIS WITH PUBLIC VAGRANT BOX URL> --force`

![cp, add vagrantfile](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/xrv64_cp_vagrantfile_m.png)


`vagrant up` - this will take some time, possibly over 10 minutes once you see the "Waiting for machine to boot" message. Look for the green "vagrant up" welcome message to confirm the three machines have booted:
	
   ![vagrant up- multi](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/xrv64_vagrant_up_m.png)
   
 Post up message:
   ![post up message](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/xrv64_post_up_m.png)


As with the Single Node setup, you have two ways to access the Vagrant instances:

1. Access the XR Linux shell:

		vagrant ssh rtr1 / vagrant ssh rtr2
        
   ![vagrant ssh- multi](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/xrv64_vagrant_ssh_1_m.png)

2. Access XR Console:

         ssh -p <forwarded port> vagrant@127.0.0.1

   * Replace `<forwarded port>` with the port forwarded for your 22 port of the router you wish to access.

      * You can check which ports were forwarded with the command:

		    vagrant port
      
    * The password is “vagrant”.

   * Repeat these steps for each node.
   
 ![ssh console- multi](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/xrv64_ssh_console_m.png)
