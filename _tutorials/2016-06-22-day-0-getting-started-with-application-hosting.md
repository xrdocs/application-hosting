---
published: true
date: '2016-06-22 19:41 +0530'
title: 'Day 0: Getting Started with Application Hosting'
author: Anne Sequeira
tags:
  - iosxr
excerpt: Getting Started with App Hosting
---
## What is IOS XR 64-bit?

Before you begin, it is essential that you are familiar with IOS XR 64-bit.

The 64-bit IOS XR uses a hypervisor to offer you the provision of creating your own build environment, spinning up your own Linux Container (LXC), or using a docker for hosting applications. XR uses a Windriver Linux Yocto distribution, and is designed to be flexible to the needs of an Application Developer.

![IOS XR 64-bit Architecture](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/364966_app%20hosting%20architecture.jpg)


## What are the different types of application hosting offered by IOS XR 64-bit? 

When you start using XR, you are offered three distinct options for hosting applications:

1. Building your own environment natively in the control plane of XR.
2. Spinning up your own container on the XR.
3. Using a docker.


## Choosing a type of application hosting
You can use the following workflow to determine the best application hosting approach for your requirement.
![CHOOSING THE TYPE OF APPLICATION HOSTING ](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/DAY%200%20WORKFLOW.png)