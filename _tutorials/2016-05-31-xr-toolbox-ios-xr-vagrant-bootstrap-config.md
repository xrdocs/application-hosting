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

**This tutorial will extend the quick-start guide to showcase how one can apply a node-specific configuration to an XR vagrant instance during boot-up itself.**
{: .notice--info}

>
Bear in mind that the IOS-XR vagrant box is published without a need for any custom plugins.
We thought about it and felt that masking the core functionality of the router with Vagrant workflows could prevent us from showcasing some core functionalities of IOS-XR :
>
* Day 0: ZTP helpers and shell/bash based automation
* Day 1: automation techniques based off YANG models
* Day 2: Streaming Telemetry and application-hosting 








