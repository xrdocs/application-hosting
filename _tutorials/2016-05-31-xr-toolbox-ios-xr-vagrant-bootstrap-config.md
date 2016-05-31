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
showcases how a user can get started with IOS-XR, running as a vagrant instance, either as a single node or in a multi-node topology.

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


## Bootstrap Configuration

The concept is simple: We'll use the Vagrant shell provisioner to apply a boostrap configuration to an XR instance when we issue a `vagrant up`.  
  
All we need is a shell provisioner section in the Vagrantfile:

```ruby 

 #Source a config file and apply it to XR
      
 config.vm.provision "file", source: "configs/_config", destination: "/home/vagrant/rtr_config"
      
 config.vm.provision "shell" do |s|
   s.path =  "#{rtr_xr_scripts_dir_host}/apply_config.sh"
   s.args = ["#{rtr_xr_cfg_file_remote}"]
 end


```











