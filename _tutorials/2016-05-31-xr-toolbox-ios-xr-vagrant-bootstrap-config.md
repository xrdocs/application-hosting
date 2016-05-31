---
published: true
date: "2016-05-31 02:13 -0700"
title: "XR Toolbox: Boostrap XR configuration with Vagrant"
permalink: "/tutorials/iosxr-vagrant-bootstrap-config"
author: Akshat Sharma
excerpt: "Configure an IOS-XR Vagrant box on boot using a Shell provisioner"
tags: 
  - vagrant
  - iosxr
  - cisco
  - xr toolbox
  - configuration
position: hidden
---

{% include toc icon="table" title="IOS-XR Vagrant: Bootstrap Config" %}
{% include base_path %}

## Introduction

The [IOS-XR Vagrant Quick Start guide]({{ base_path }}/tutorials/iosxr-vagrant-quickstart)
showcases how a user can get started with an IOS-XR vagrant box.

**This tutorial will extend the quick-start guide to showcase how one can apply a node-specific configuration to an XR vagrant instance during boot-up itself.  
Make sure you take a look at the quick-start guide before proceeding.**
{: .notice--warning}

>
Bear in mind that the IOS-XR vagrant box is published without a need for any custom plugins.
We thought about it and felt that masking the core functionality of the router with Vagrant workflows could prevent us from showcasing some core functionalities of IOS-XR, namely :
>
* Day 0: ZTP helpers and shell/bash based automation
* Day 1: automation techniques based off YANG models
* Day 2: Streaming Telemetry and application-hosting 

This tutorial invariably ends up using the new shell/bash based automation techniques that have been introduced as part of the Zero Touch provisioning (ZTP) functionality in IOS-XR.


## Bootstrap Configuration: Shell Provisioner

The concept is simple: We'll use the Vagrant shell provisioner to apply a boostrap configuration to an XR instance when we issue a `vagrant up`.  
  
All we need is a shell provisioner section in the Vagrantfile for each node:

```ruby 
 #Source a config file and apply it to XR
      
 config.vm.provision "file", source: "configs/rtr_config", destination: "/home/vagrant/rtr_config"
      
 config.vm.provision "shell" do |s|
   s.path =  "scripts/apply_config.sh"
   s.args = ["/home/vagrant/rtr_config"]
 end
```

We will look at a complete Vagrantfile in a bit. But let's desconstruct the above piece of code.


### Transfer a Configuration file to XR bash
   
```ruby 
config.vm.provision "file", source: "configs/rtr_config", destination: "/home/vagrant/rtr_config"
```

The above line uses the Vagrant "file" provisioner to transfer a file from the host (your laptop) to the XR linux shell (bash).

The root of the source directory is the working directory for your vagrant instance. Hence, the `rtr_config` file is located in the `configs` directory.

### Use a Shell script to Apply XR Config

```ruby
config.vm.provision "shell" do |s|
   s.path =  "scripts/apply_config.sh"
   s.args = ["/home/vagrant/rtr_config"]
 end
```
The shell script will eventually be run on XR bash of the vagrant instance. This script is placed in the `scripts` directory and is named `apply_config.sh`.  

Further, the script needs the location of the router config file as an argument. This is the `destination` parameter in the "file" provisioner above.


So, in short, Vagrant copies a config file to the router bash, and then runs a shell script on the router bash to apply the config file that was copied!
{: .notice--success}

     
      
For example, I'm running my single Vagrant node from the directory ~/iosxrv. My directory structure looks something like:

```shell
AKSHSHAR-M-K0DS:iosxrv akshshar$ pwd
/Users/akshshar/iosxrv
AKSHSHAR-M-K0DS:iosxrv akshshar$ tree ./
./
├── Vagrantfile
├── configs
│   └── rtr_config
└── scripts
    └── apply_config.sh

2 directories, 3 files
```


## Single node bootstrap

Let's assume we're applying a simple XR config that configures Gig0/0/0/0 and enables the grpc server on port 57788:

This configuration will be an **addendum** to the pre-existing configuration on the vagrant instance.
{: .notice--info}

```shell
AKSHSHAR-M-K0DS:iosxrv akshshar$ cat configs/rtr_config 
!! XR configuration
!
interface GigabitEthernet0/0/0/0
  ip address 11.1.1.2/24
  no shutdown
!
grpc
  port 57788
!
end
```














