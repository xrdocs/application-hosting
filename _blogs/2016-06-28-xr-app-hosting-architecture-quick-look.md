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


  <img src="https://xrdocs.github.io/xrdocs-images/assets/images/xr-control-plane-lxc.png" width="250" height="250" />{: .align-center}
  
*  Inside the XR control plane LXC, if we zoom in further, the XR control plane processes are represented distinctly in blue as shown below. This where the XR routing protocols like BGP, OSPF etc. run. The XR CLI presented to the user is also one of the processes.

  <img src="https://xrdocs.github.io/xrdocs-images/assets/images/xr-control-plane.png" width="250" height="250" />{: .align-center}

  
*  See the gray box inside the XR control plane LXC ? This is the XR linux shell .  
   **P.S. This is what you drop into when you issue a  `vagrant ssh` [[*]]({{ base_path }}/tutorials/iosxr-vagrant-quickstart)**.  
   {: .notice--info}  
   
   The XR linux shell that the user interacts with is really the `global-vrf` network namespace 
   inside the control plane container. This corresponds to the global/default-vrf in IOS-XR.  
  
   <img src="https://xrdocs.github.io/xrdocs-images/assets/images/xr-global-vrf-ns.png" width="200" height="250" />{: .align-center}  

   
   **Any Linux application hosted in this environment shares the process space with XR, and we refer to it as 
   a** `native application`.

   **Only the interfaces in global/default vrf in XR appear in the XR linux shell today when you 
   issue an ifconfig. But the infrastructure is in place to map each custom user VRF to a network 
   namespace in the future.**   
   



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
