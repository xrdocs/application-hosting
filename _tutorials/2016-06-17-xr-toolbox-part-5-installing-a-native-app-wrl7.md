---
published: true
date: '2016-06-17 17:47 -0700'
title: 'XR toolbox, Part 5: Installing a native app (WRL7)'
author: Akshat Sharma
excerpt: 'Build, Install and launch a native app in IOS-XR (WRL7 RPM or binary)'
position: hidden
tags:
  - vagrant
  - iosxr
  - cisco
  - linux
  - wrl7
  - rpm
---

{% include toc icon="table" title="Launching a Container App" %}
{% include base_path %}
  
Check out Part 4 of the XR toolbox series: [Bring your own Container (LXC) App]({{ base_path }}/tutorials/2016-06-16-xr-toolbox-part-4-bring-your-own-container-lxc-app/).

## Introduction

If you haven't checked out the earlier parts to the XR toolbox Series, then you can do so here:  

>
[XR Toolbox Series]({{ base_path }}/tags/#xr-toolbox)

  
The purpose of this series is simple. Get users started with an IOS-XR setup on their laptop and incrementally enable them to try out the application-hosting infrastructure on IOS-XR.

In this part, we explore how a user can build and deploy native WRL7 RPMs that they may host in the same process space as XR.  


## Sounds good, but why?  

As you go through these tutorials, it would be pretty apparent that we've helped you bring up IOS-XR on your laptop, we've created development topologies and even orchestrated ubuntu LXCs on the router.   
  
But we haven't actually delved into the internal architecture of IOS-XR. While there are several upcoming documents that will shed light on the deep internal workings of IOS-XR, before we jump into this tutorial, let's take a quick look at the internals for a bit:  

![xr-app-hosting-infra](https://xrdocs.github.io/xrdocs-images/assets/images/xr-app-hosting-infra-basic.png)  

  
Alright, back up. The above figure seems pretty daunting to understand, so let's try to deconstruct it:  
  
*  At the bottom of the figure, in gray, we have the host (hypervisor) linux environment. This is a 64-bit linux kernel running the Windriver linux 7 (WRL7) environment.  
*  In green, we see the container (called the Control plane LXC or XR LXC) within which XR runs as a collection of processes. This is what presents XR CLI and this is what runs the routing protocols.  
*  Inside the control plane LXC 



