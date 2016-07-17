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


**The gray region inside the control plane LXC represents the global-vrf network namespace in the XR linux environment. Today, IOS-XR only supports the mapping of global/default VRF in IOS-XR to the global-vrf network namespace in XR linux.**
{: .notice--info}  

>
To get into the XR linux shell (global-vrf network namespace), we have two possible techniques:  
>
*  **From XR CLI**:  Issue the `bash` command to drop into the XR linux shell from the CLI.
*  **Over SSH using port 57722**:  Port 22 is used by XR SSH. To enable a user/tool to drop directly into the XR linux shell, we enable SSH over port 57722. Any reachable IP address of XR could be used for this purpose.


**Once in the XR linux shell, if we issue an ifconfig we should see all the interfaces (that are up/unshut) in the global/default VRF:**  
   
<div class="highlighter-rouge">
<pre class="highlight">
<code>
   RP/0/RP0/CPU0:rtr1#
   RP/0/RP0/CPU0:rtr1#
   RP/0/RP0/CPU0:rtr1#<mark>show  ip int br</mark>
   Sun Jul 17 11:52:15.049 UTC
   
   Interface                      IP-Address      Status          Protocol Vrf-Name
   Loopback0                      1.1.1.1         Up              Up       default 
   GigabitEthernet0/0/0/0         10.1.1.10       Up              Up       default 
   GigabitEthernet0/0/0/1         11.1.1.10       Up              Up       default 
   GigabitEthernet0/0/0/2         unassigned      Shutdown        Down     default
   MgmtEth0/RP0/CPU0/0            10.0.2.15       Up              Up       default 
   RP/0/RP0/CPU0:rtr1#
   RP/0/RP0/CPU0:rtr1#
   RP/0/RP0/CPU0:rtr1#<mark>bash</mark>    
   Sun Jul 17 11:52:22.904 UTC

   [xr-vm_node0_RP0_CPU0:~]$
   [xr-vm_node0_RP0_CPU0:~]$<mark>ifconfig</mark>
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
</code>
</pre>
</div> 
   
  
**Any Linux application hosted in this environment shares the process space with XR, and we refer to it as a `native application`.**  
{: .notice--warning} 

## Spin up the build environment

We're going to spin up a topology with two vagrant instances as shown below:  

![native-app-topo](https://xrdocs.github.io/xrdocs-images/assets/images/native-app-topo.png)  

*  **devbox**: This is the build environment. Since IOS-XR uses a streamlined custom WRL7 distribution, we need to make sure we have the latest WRL7 environment available to build "native" apps. For this reason we have released the <https://atlas.hashicorp.com/ciscoxr/boxes/appdev-xr6.1.1> vagrant box to match IOS-XR release 6.1.1.  You will simply need to reference "ciscoxr/appdev-xr6.1.1" in your Vagrantfile to spin it up.  
  
*  **IOS-XR**: This is the 6.1.1 IOS-XR vagrant instance you would have already downloaded and installed as explained in the vagrant quick-start tutorial:    

   >
   [IOS-XR vagrant box download]({{ base_path }}/tutorials/iosxr-vagrant-quickstart#download-and-add-the-ios-xrv-vagrant-box)  
 
   In the end, the `vagrant box list` must list your IOS-XRv vagrant box:  
   
   ```shell

   AKSHSHAR-M-K0DS:~ akshshar$ vagrant box list
   IOS-XRv (virtualbox, 0)
   AKSHSHAR-M-K0DS:~ akshshar$ 

   ```
   
### Clone the git repo
 
I have set up the git repo with the Vagrantfile and other directories needed to configure the IOS-XR vagrant instance and spin up the app-dev topology. To know more about how to configure the IOS-XR instance, you can check out:  [IOS-XR vagrant bootstrap config]({{ base_path }}/tutorials/iosxr-vagrant-bootstrap-config) 
   
 
<div class="highlighter-rouge">
<pre class="highlight">
<code> 
AKSHSHAR-M-K0DS:~ akshshar$<mark> git clone https://github.com/ios-xr/vagrant-xrdocs.git </mark>
Cloning into 'vagrant-xrdocs'...
remote: Counting objects: 204, done.
remote: Compressing objects: 100% (17/17), done.
remote: Total 204 (delta 4), reused 0 (delta 0), pack-reused 187
Receiving objects: 100% (204/204), 27.84 KiB | 0 bytes/s, done.
Resolving deltas: 100% (74/74), done.
Checking connectivity... done.
AKSHSHAR-M-K0DS:~ akshshar$ 
AKSHSHAR-M-K0DS:~ akshshar$ 
AKSHSHAR-M-K0DS:~ akshshar$<mark> cd vagrant-xrdocs/native-app-topo-bootstrap/</mark>
AKSHSHAR-M-K0DS:native-app-topo-bootstrap akshshar$ pwd
<mark>/Users/akshshar/vagrant-xrdocs/native-app-topo-bootstrap</mark>
AKSHSHAR-M-K0DS:native-app-topo-bootstrap akshshar$ ls
Vagrantfile	configs		scripts
AKSHSHAR-M-K0DS:native-app-topo-bootstrap akshshar$ 
</code>
</pre>
</div> 

Once you're in the right directory, simply issue a `vagrant up`:  

<div class="highlighter-rouge">
<pre class="highlight">
<code> 
AKSHSHAR-M-K0DS:native-app-topo-bootstrap akshshar$<mark> vagrant up </mark>
Bringing machine 'rtr' up with 'virtualbox' provider...
Bringing machine 'devbox' up with 'virtualbox' provider...

--------------------------- snip output ----------------------------
</code>
</pre>
</div> 


## Build iperf from source on devbox

Assuming e

 
   

