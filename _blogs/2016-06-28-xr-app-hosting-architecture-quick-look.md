---
published: true
date: '2016-06-28 01:54 -0700'
title: 'XR App-hosting architecture: Quick Look!'
author: Akshat Sharma
excerpt: Quick Walkthrough of the XR app-hosting architecture
tags:
  - iosxr
  - cisco
  - architecture
  - xr toolbox
position: hidden
---
If you've been following along the set of tutorials in the XR toolbox series:  

>
[XR Toolbox Series]({{ base_path }}/tags/#xr-toolbox)
  
  
You might have noticed that we haven't actually delved into the internal architecture of IOS-XR. While there are several upcoming documents that will shed light on the deep internal workings of IOS-XR, I thought I'll take a  quick stab at the internals for the uninitiated.

This is what the internal software architecture and plumbing, replete with the containers, network namespaces and XR interfaces, looks like:  

![xr-app-hosting-infra](https://xrdocs.github.io/xrdocs-images/assets/images/xr-app-hosting-infra-basic.png)  

  
Alright, back up. The above figure seems pretty daunting to understand, so let's try to deconstruct it:  
  
*  At the bottom of the figure, in gray, we have the host (hypervisor) linux environment. This is a 64-bit linux kernel running the Windriver linux 7 (WRL7) environment.  
   ![host-linux-hypervisor](https://xrdocs.github.io/xrdocs-images/assets/images/host-linux-   hypervisor.png)
{: .notice}{: .align-center}  

*  In green, we see the container (called the Control plane LXC or XR LXC) within which XR runs as a collection of processes (represented in sky blue). This is what presents the XR CLI and this is what runs the routing protocols.  
*  Inside the control plane LXC, you are presented with the XR linux shell. This is what you drop into when you issue a `vagrant ssh`.  
*  The XR linux shell that the user interacts with is really the `global-vrf` network namespace inside the control plane container. This corresponds to the global-vrf/default-vrf in IOS-XR. **Only the interfaces in global-vrf appear in the XR linux shell today when you issue an ifconfig. But the infrastructre is in place to map each custom user VRF to a network namespace in the future.**  

*  The FIB is programmed by the XR control plane exclusively. The global-vrf network namespace only sees a couple of routes:  
  *  A default route pointing to XR FIB. This way any packet with an unknown destination is handed by a linux application to XR for routing.  
  *  Routes in the subnet of the Management Interface:  Mgmt0/RP0/CPU0

