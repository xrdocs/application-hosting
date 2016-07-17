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
position: top
---

{% include base_path %}  

If you've been following the set of tutorials in the XR toolbox series:  

>
[XR Toolbox Series]({{ base_path }}/tags/#xr-toolbox)
  
  
You might have noticed that we haven't actually delved into the internal architecture of IOS-XR. While there are several upcoming documents that will shed light on the deep internal workings of IOS-XR, I thought I'll take a  quick stab at the internals for the uninitiated.

This is what the internal software architecture and plumbing, replete with the containers, network namespaces and XR interfaces, looks like:  

[![xr-app-hosting-infra](https://xrdocs.github.io/xrdocs-images/assets/images/xr-app-hosting-infra-basic.png)](https://xrdocs.github.io/xrdocs-images/assets/images/xr-app-hosting-infra-basic.png)

  
Alright, back up. The above figure seems pretty daunting to understand, so let's try to deconstruct it:  
  
*  At the bottom of the figure, in gray, we have the host (hypervisor) linux environment. This is a 64-bit linux kernel running the Windriver linux 7 (WRL7) distribution. The rest of the components run as containers (LXCs) on top of the host.    

  ![host-linux-hypervisor](https://xrdocs.github.io/xrdocs-images/assets/images/host_linux_hypervisor.png){: .align-center}
{: .notice}

*  In green, we see the container called the XR Control plane LXC (or XR LXC). This runs a Windriver Linux 7 (WRL7) environment as well and contains the XR control plane and the XR linux environment:  


<a href="https://xrdocs.github.io/xrdocs-images/assets/images/xr-control-plane-lxc.png"><img src="https://xrdocs.github.io/xrdocs-images/assets/images/xr-control-plane-lxc.png" width="350" height="350" class="align-center" /></a>
  
*  Inside the XR control plane LXC, if we zoom in further, the XR control plane processes are represented distinctly in blue as shown below. This is where the XR routing protocols like BGP, OSPF etc. run. The XR CLI presented to the user is also one of the processes.

 <a href="https://xrdocs.github.io/xrdocs-images/assets/images/xr-control-plane.png"><img src="https://xrdocs.github.io/xrdocs-images/assets/images/xr-control-plane.png" width="250" height="250" class="align-center" /></a>

  
*  See the gray box inside the XR control plane LXC ? This is the XR linux shell.  
  
   **P.S. This is what you drop into when you issue a  `vagrant ssh` [[*]]({{ base_path }}/tutorials/iosxr-vagrant-quickstart)**.  
   **Another way to get into the XR linux shell is by issuing a `bash` command in XR CLI.**
   {: .notice--info}  
   
   The XR linux shell that the user interacts with is really the `global-vrf` network namespace 
   inside the control plane container. This corresponds to the global/default-vrf in IOS-XR.  
  
   <img src="https://xrdocs.github.io/xrdocs-images/assets/images/xr-global-vrf-ns.png" width="200" height="250" />{: .align-center}  
   
   **Only the interfaces in global/default vrf in XR appear in the XR linux shell today when you 
   issue an ifconfig:  
   ```shell
   RP/0/RP0/CPU0:rtr1#
   RP/0/RP0/CPU0:rtr1#
   RP/0/RP0/CPU0:rtr1#show  ip int br
   Sun Jul 17 11:52:15.049 UTC
   
   Interface                      IP-Address      Status          Protocol Vrf-Name
   Loopback0                      1.1.1.1         Up              Up       default 
   Loopback1                      6.6.6.6         Up              Up       default 
   GigabitEthernet0/0/0/0         10.1.1.10       Up              Up       default 
   GigabitEthernet0/0/0/1         11.1.1.10       Up              Up       default 
   GigabitEthernet0/0/0/2         unassigned      Shutdown        Down     default 
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
     
   


*  The FIB is programmed by the XR control plane exclusively. The global-vrf network namespace only sees a couple of routes by default:  
    *  A default route pointing to XR FIB. This way any packet with an unknown destination is handed-over by a linux application to XR for routing. This is achieved through a special interface called `fwdintf` as shown in the figure above.  
       
      
    *  Routes in the subnet of the Management Interface:  Mgmt0/RP0/CPU0. The management subnet is local to the global-vrf network namespace.

   To view these routes, simply issue an `ip route` in the XR linux shell:  
  
   <div class="highlighter-rouge">
   <pre class="highlight">
   <code>
   AKSHSHAR-M-K0DS:native-app-bootstrap akshshar$<mark> vagrant ssh rtr </mark>
   xr-vm_node0_RP0_CPU0:~$ 
   xr-vm_node0_RP0_CPU0:~$ 
   xr-vm_node0_RP0_CPU0:~$<mark> ip route </mark>
   default dev fwdintf  scope link  src 10.0.2.15 
   10.0.2.0/24 dev Mg0_RP0_CPU0_0  proto kernel  scope link  src 10.0.2.15 
   xr-vm_node0_RP0_CPU0:~$ 
   xr-vm_node0_RP0_CPU0:~$ 
   </code>
   </pre>
   </div>  
   
   
   
*  Finally, if you followed the [Bring your own Container (LXC) App]({{ base_path }}/tutorials/2016-06-16-xr-toolbox-part-4-bring-your-own-container-lxc-app/), you'll notice that in the XML file meant to launch the lxc, we share the `global-vrf` network namespace with the container; specifically, in this section:  

   
   [Create LXC SPEC XML File]({{ base_path }}/tutorials/2016-06-16-xr-toolbox-part-4-bring-your-   
   own-container-lxc-app/#create-lxc-spec-xml-file) 

   This makes the architecture work seamlessly for `native` and `container` applications. **An 
   LXC app has the same view of the world, the same routes and the same XR interfaces to take 
   advantage of, as any native application with the shared global-vrf namespace**.  

   <img src="https://xrdocs.github.io/xrdocs-images/assets/images/xr-global-vrf-lxc.png" width="250" height="250" />{: .align-center}  

*  You'll also notice my awkward rendering for a linux app:  

   <img src="https://xrdocs.github.io/xrdocs-images/assets/images/linux-app-tpa.png" width="150" height="150" />{: .align-center}  

   Notice the `TPA IP` ? This stands for **T**hird **P**arty **A**pp IP address.  
  
   The purpose of the TPA IP is simple. Set a src-hint for linux applications, so that originating 
   traffic from the applications (native or LXC) could be tied to the loopback IP or any reachable 
   IP of XR.   
  
   This approach mimics how routing protocols like to identify routers in complex topologies: 
   through router-IDs. With the TPA IP, application traffic can be consumed, for example, across 
   an OSPF topology just by relying on XR's capability to distribute the loopback IP address 
   selected as the src-hint.
   {: .notice--warning}  
  
   We go into further detail here: 
   [Set the src-hint for Application traffic](https://xrdocs.github.io/application-hosting/tutorials/2016-06-16-xr-toolbox-part-4-bring-your-own-container-lxc-app/#set-the-src-hint-for-application-traffic)

**That pretty much wraps it up. Remember, XR handles the routing and applications use a only a subset of the routing table to piggy-back on XR for reachability!**
{: .notice--success}
