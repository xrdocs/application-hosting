---
published: true
date: '2016-06-17 17:47 -0700'
title: 'XR toolbox, Part 5: Running a native WRL7 app'
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


## What's a native app?  

I go into some detail with respect to the IOS-XR application hosting architecture in the following blog:
  
>
[XR app-hosting infrastructure: Quick Look]({{ base_path }}/blogs/2016-06-28-xr-app-hosting-architecture-quick-look/)  
  
  
For reference, a part of the architecture is shown below. We focus on the green container in the figure from the original blog:  

<a href="https://xrdocs.github.io/xrdocs-images/assets/images/xr-control-plane-lxc.png"><img src="https://xrdocs.github.io/xrdocs-images/assets/images/xr-control-plane-lxc.png" width="550" height="550" class="align-center" /></a>  
  
This is the XR control plane LXC. XR processes (routing protocols, XR CLI etc.) are all housed in the blue region. We represent XR FIB within the same region to indicate that the XR control plane exclusively handles the data-plane programming and access to the real XR interfaces (Gig, Mgmt etc.)   


**The gray region inside the control plane LXC represents the global-vrf network namespace in the XR linux environment. Today, IOS-XR only supports the mapping of global/default VRF in IOS-XR with the network namespace in XR linux.**
{: .notice--info}  

>
To get into the XR linux shell (global-vrf network namespace), we have two possible techniques:  
>
*  **From XR CLI**:  Issue the `bash` command to drop into the XR linux shell from the CLI.
*  **Over SSH using port 57722**:  Port 22 is used by XR SSH. To enable a user/tool to drop directly into the XR linux shell, we enable SSH over port 57722. Any reachable IP address of XR could be used for this purpose.


** Once in the XR linux shell, if we issue an ifconfig we should see all the interfaces in the global/default VRF:**  
   
```shell
   RP/0/RP0/CPU0:rtr1#
   RP/0/RP0/CPU0:rtr1#
   RP/0/RP0/CPU0:rtr1#show  ip int br
   Sun Jul 17 11:52:15.049 UTC
   
   Interface                      IP-Address      Status          Protocol Vrf-Name
   Loopback0                      1.1.1.1         Up              Up       default 
   GigabitEthernet0/0/0/0         10.1.1.10       Up              Up       default 
   GigabitEthernet0/0/0/1         11.1.1.10       Up              Up       default 
   MgmtEth0/RP0/CPU0/0            10.0.2.15       Up              Up       default 
   RP/0/RP0/CPU0:rtr1#
   RP/0/RP0/CPU0:rtr1#
   RP/0/RP0/CPU0:rtr1#bash    
   Sun Jul 17 11:52:22.904 UTC

   [xr-vm_node0_RP0_CPU0:~]$
   [xr-vm_node0_RP0_CPU0:~]$ifconfig
   Gi0_0_0_0 Link encap:Ethernet  HWaddr 08:00:27:e0:7f:bb  
             inet addr:10.1.1.10  Mask:255.255.255.0
             inet6 addr: fe80::a00:27ff:fee0:7fbb/64 Scope:Link
             UP RUNNING NOARP MULTICAST  MTU:1514  Metric:1
             RX packets:0 errors:0 dropped:0 overruns:0 frame:0
             TX packets:546 errors:0 dropped:3 overruns:0 carrier:1
             collisions:0 txqueuelen:1000 
             RX bytes:0 (0.0 B)  TX bytes:49092 (47.9 KiB)

   Gi0_0_0_1 Link encap:Ethernet  HWaddr 08:00:27:26:ca:9c  
             inet addr:11.1.1.10  Mask:255.255.255.0
             inet6 addr: fe80::a00:27ff:fe26:ca9c/64 Scope:Link
             UP RUNNING NOARP MULTICAST  MTU:1514  Metric:1
             RX packets:0 errors:0 dropped:0 overruns:0 frame:0
             TX packets:547 errors:0 dropped:3 overruns:0 carrier:1
             collisions:0 txqueuelen:1000 
             RX bytes:0 (0.0 B)  TX bytes:49182 (48.0 KiB)

   Mg0_RP0_CPU0_0 Link encap:Ethernet  HWaddr 08:00:27:ab:bf:0d  
             inet addr:10.0.2.15  Mask:255.255.255.0
             inet6 addr: fe80::a00:27ff:feab:bf0d/64 Scope:Link
             UP RUNNING NOARP MULTICAST  MTU:1514  Metric:1
             RX packets:210942 errors:0 dropped:0 overruns:0 frame:0
             TX packets:84664 errors:0 dropped:0 overruns:0 carrier:1
             collisions:0 txqueuelen:1000 
             RX bytes:313575212 (299.0 MiB)  TX bytes:4784245 (4.5 MiB)

   ---------------------------------- snip output -----------------------------------------
```
   
  
   **Any Linux application hosted in this environment shares the process space with XR, and we refer to it as 
   a `native application`.**  
   {: .notice--warning}  



