---
published: true
date: '2017-02-26 14:40 -0800'
title: 'XR toolbox, Part 6: Running Docker Containers on IOS-XR (6.1.2+)'
position: hidden
author: Akshat Sharma
excerpt: >-
  Running Docker containers on IOS-XR running as a Vagrant box and on a physical
  NCS5500
tags:
  - vagrant
  - iosxr
  - NCS5500
  - docker
  - xr toolbox
---


{% include toc icon="table" title="Running Docker Containers on IOS-XR" %}
{% include base_path %}

## Introduction

If you haven't checked out the earlier parts to the XR toolbox Series, then you can do so here:  

>
[XR Toolbox Series]({{ base_path }}/tags/#xr-toolbox)

  
The purpose of this series is simple. Get users started with an IOS-XR setup on their laptop and incrementally enable them to try out the application-hosting infrastructure on IOS-XR.

In this part, we explore how a user can spin up Docker containers on IOS-XR. There are multiple ways to do this and we'll explore each one:  
  
*  **Public  Dockerhub Registry**: This is the simplest setup that most docker users would be well aware of. All you need to do is set up reachability to dockerhub with the correct dns resolution.

*  **Private "insecure" registry**: Some users may choose to do this, specially if they're running a local docker registry inside a secured part of their network.  
   
*  **Private "self-signed" registry**: This is more secure than the "insecure" setup, and allows a user to enable TLS.

*  **Private "secure" registry**: Set up reachability to your private registry, created using a certificate obtained from a CA. The steps used to set this up are identical to a private self-signed registry except for the creation of the certificate. We won't really tackle this scenario separately in this tutorial due to the absence of said certificate :).
    
*  **Tarball image/container**:  This is the simplest setup - very similar to LXC deployments. In this case, a user may create and set up a container completely off-box, package it up as an image or a container tar ball, transfer it to the router and then load/import it, before running.  


For each case, we will compare IOS-XR running as a Vagrant box with IOS-XR running on a physical box (NCS5500 and ASR9k). They should be identical, except for reachability through the Management ports.  
  
  
## Pre-requisites

### Vagrant IOS-XR box

If you're bringing up the topology on your laptop using the IOS-XR vagrant box, then:

* Meet the pre-requisites specified in the [IOS-XR Vagrant Quick Start guide: Pre-requisites]({{ base_path }}/tutorials/iosxr-vagrant-quickstart#pre-requisites).   
The topology here will require about 5G RAM and 2 cores on the user's laptop.
* Clone the following repository: <https://github.com/ios-xr/vagrant-xrdocs>, before we start.

```shell
cd ~/
git clone https://github.com/ios-xr/vagrant-xrdocs.git
cd vagrant-xrdocs/
```  

You will notice a few directories. We will utilize the `docker-app-topo-bootstrap` directory in this tutorial.
{: .notice--info}

```shell
AKSHSHAR-M-K0DS:vagrant-xrdocs akshshar$ pwd
/Users/akshshar/vagrant-xrdocs
AKSHSHAR-M-K0DS:vagrant-xrdocs akshshar$ ls docker-app-topo-bootstrap/
Vagrantfile	configs		scripts
AKSHSHAR-M-K0DS:vagrant-xrdocs akshshar$ 

```


### Physical (NCS5500 and ASR9k)  

On the other hand, if you have an NCS5500 or ASR9k lying around (don't we all?), then load up a 6.1.2+ image on the router and connect an Ubuntu server (for the purpose of this tutorial), to the Management network of the router.

The server needs to be reachable from the router over the Management network.

Further, we're going to enable SSH access in XR CLI and in  XR linux shell to achieve an equivalence between the NCS5500/ASR9k and Vagrant setup.  

**Note**: NCS5500 steps are described, but ASR9k works in exactly the same way. 
{: .notice--warning}  

**Enable SSH access in the XR CLI**
{: .notice}

On my NCS5500 setup, I can enable SSH in XR in the default (global) vrf with the following steps and CLI:  

<div class="highlighter-rouge">
<pre class="highlight" style="white-space: pre-wrap;">
<code>

RP/0/RP0/CPU0:ncs5508#<mark>crypto key generate rsa</mark>
Mon Mar  6 05:28:57.184 UTC
The name for the keys will be: the_default
  Choose the size of the key modulus in the range of 512 to 4096 for your General Purpose Keypair. Choosing a key modulus greater than 512 may take a few minutes.

How many bits in the modulus [2048]: 
Generating RSA keys ...
Done w/ crypto generate keypair
[OK]

RP/0/RP0/CPU0:ncs5508#
RP/0/RP0/CPU0:ncs5508#show  running-config ssh
Mon Mar  6 05:29:51.819 UTC
ssh server v2
ssh server vrf default

RP/0/RP0/CPU0:ncs5508#

</code>
</pre>
</div>

**Enable SSH access to XR linux shell**
{: .notice}

This is openssh running in the XR linux environment. Users may choose to keep this disabled based on the kind of operations they intend to have. Enabling it in a given network namespace (equivalent to XR vrf) opens up port 57722 on all the IP addresses reachable in that VRF.

In 6.1.2, only global-vrf (default vrf) is supported in the linux environment for SSH and apps. Post 6.3.1, support for Mgmt vrfs in the linux shell will be brought in.
{: .notice-warning} 

To enable SSH access in the XR linux shell for a sudo user, we'll take 3 steps:

*  Enable the "sudo" group permissions in /etc/sudoers

   Open up /etc/sudoers using vi in the XR bash shell and uncomment the following line:

   ```
   # %sudo ALL=(ALL) ALL
   ```

   Save and exit (:wq in vi).  

*  Create a non-root user. This is important. For security reasons, root user access over SSH (SSH    in the linux shell) is disabled. Only the root XR user can create new (sudo or non-sudo) users,    so use the "bash" cli to get into the shell:

   ```
   RP/0/RP0/CPU0:ncs5508#
   RP/0/RP0/CPU0:ncs5508#bash
   Mon Mar  6 06:16:01.391 UTC
  
   [ncs5508:~]$
   [ncs5508:~]$adduser cisco
   Login name for new user []:cisco

   User id for cisco [ defaults to next available]:

   Initial group for cisco [users]:

   Additional groups for cisco []:sudo

   cisco's home directory [/home/cisco]:

   cisco's shell [/bin/bash]:

   cisco's account expiry date (MM/DD/YY) []:

   OK, Im about to make a new account. Heres what you entered so far:
   New login name: cisco
   New UID: [Next available]
   Initial group: users
   /usr/sbin/adduser: line 68: [: -G: binary operator expected
   Additional groups: sudo
   Home directory: /home/cisco
   Shell: /bin/bash
   Expiry date: [no expiration]
   This is it... if you want to bail out, you'd better do it now.

   Making new account...
   useradd: user 'cisco' already exists
   Changing the user information for cisco
   Enter the new value, or press ENTER for the default
	   Full Name []: 
	   Room Number []: 
	   Work Phone []: 
	   Home Phone []: 
	   Other []: 
   Enter new UNIX password: 
   Retype new UNIX password: 
   passwd: password updated successfully
   Done...
   [ncs5508:~]$

   ```



*  Finally enable SSH access by starting the sshd_operns service:  

   ```
   [ncs5508:~]$service sshd_operns start
   Mon Mar 6 06:21:53 UTC 2017 /etc/init.d/sshd_operns: Waiting for OPERNS interface creation...
   Mon Mar 6 06:21:53 UTC 2017 /etc/init.d/sshd_operns: Press ^C to stop if needed.
   Mon Mar 6 06:21:54 UTC 2017 /etc/init.d/sshd_operns: Found nic, Mg0_RP0_CPU0_0
   Mon Mar 6 06:21:54 UTC 2017 /etc/init.d/sshd_operns: Waiting for OPERNS management interface      creation...
   Mon Mar 6 06:21:54 UTC 2017 /etc/init.d/sshd_operns: Found nic, Mg0_RP0_CPU0_0
   Mon Mar 6 06:21:54 UTC 2017 /etc/init.d/sshd_operns: OPERNS is ready
   Mon Mar 6 06:21:54 UTC 2017 /etc/init.d/sshd_operns: Start sshd_operns
   Starting OpenBSD Secure Shell server: sshd
     generating ssh RSA key...
     generating ssh ECDSA key...
     generating ssh DSA key...
     generating ssh ED25519 key...
   [ncs5508:~]$
   ```

   Check that the sshd_operns service is now listening on port 57722 in the global-vrf network        namespace:  

   netns_identify utility is to check which network namespace a process is in. `$$` gets the pid      of the current shell. In the output below, tpnns is a symbolic link of  global-vrf. So they        both mean the same thing - XR default VRF mapped to a network namespace in linux. All XR          interfaces in the default(global) vrf will appear in the linux shell in this network namespace.    Issuing an `ifconfig` will show up these interfaces.
   {: .notice--info}


   ```shell

   [ncs5508:~]$netns_identify $$
   tpnns
   global-vrf
   [ncs5508:~]$netstat -nlp | grep 57722
   tcp        0      0 0.0.0.0:57722           0.0.0.0:*               LISTEN      622/sshd        
   tcp6       0      0 :::57722                :::*                    LISTEN      622/sshd        
   [ncs5508:~]$
   [ncs5508:~]$ifconfig
   Mg0_RP0_CPU0_0 Link encap:Ethernet  HWaddr 80:e0:1d:00:fc:ea  
             inet addr:11.11.11.59  Mask:255.255.255.0
             inet6 addr: fe80::82e0:1dff:fe00:fcea/64 Scope:Link
             UP RUNNING NOARP MULTICAST  MTU:1514  Metric:1
             RX packets:3830 errors:0 dropped:0 overruns:0 frame:0
             TX packets:4 errors:0 dropped:0 overruns:0 carrier:3
             collisions:0 txqueuelen:1000 
             RX bytes:1288428 (1.2 MiB)  TX bytes:280 (280.0 B)

   fwd_ew    Link encap:Ethernet  HWaddr 00:00:00:00:00:0b  
             inet6 addr: fe80::200:ff:fe00:b/64 Scope:Link
             UP RUNNING NOARP MULTICAST  MTU:1500  Metric:1
             RX packets:18 errors:0 dropped:10 overruns:0 frame:0
             TX packets:2 errors:0 dropped:1 overruns:0 carrier:0
             collisions:0 txqueuelen:1000 
             RX bytes:486 (486.0 B)  TX bytes:140 (140.0 B)
  
   fwdintf   Link encap:Ethernet  HWaddr 00:00:00:00:00:0a  
             inet6 addr: fe80::200:ff:fe00:a/64 Scope:Link
             UP RUNNING NOARP MULTICAST  MTU:1500  Metric:1
             RX packets:0 errors:0 dropped:0 overruns:0 frame:0
             TX packets:2 errors:0 dropped:1 overruns:0 carrier:0
             collisions:0 txqueuelen:1000 
             RX bytes:0 (0.0 B)  TX bytes:140 (140.0 B)

   lo        Link encap:Local Loopback  
             inet addr:127.0.0.1  Mask:255.0.0.0
             inet6 addr: ::1/128 Scope:Host
             UP LOOPBACK RUNNING NOARP MULTICAST  MTU:65536  Metric:1
             RX packets:0 errors:0 dropped:0 overruns:0 frame:0
             TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
             collisions:0 txqueuelen:0 
             RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

   lo:0      Link encap:Local Loopback  
             inet addr:1.1.1.1  Mask:255.255.255.255
             UP LOOPBACK RUNNING NOARP MULTICAST  MTU:65536  Metric:1

   [ncs5508:~]$

 
   ```

Awesome! Now let's test SSH access directly into the linux shell:

As seen from the above output, the Mgmt port (Mg0_RP0_CPU0_0) has an IP 11.11.11.59 and the port 57722 is open all the IP addresses in the corresponding network namespace. 

From the directly connected "devbox" or jumpserver I can then issue an ssh as follows:

```
cisco@dhcpserver:~$ ssh cisco@11.11.11.59 -p 57722
cisco@11.11.11.59's password: 
-sh: /var/log/boot.log: Permission denied
ncs5508:~$ 
ncs5508:~$ sudo -i
Password: 
[ncs5508:~]$ 
[ncs5508:~]$ whoami
root
[ncs5508:~]$ 
```

Works like a charm!
{: .notice--success}


## Understand the topology

The topology I'm using differs slightly between the vagrant setup and the NCS5500 setup.
This is owing to the fact that the Management port of the vagrant IOS-XR box is used up in the NAT network. So to show equivalence between the two setups, I directly connect the Gig0/0/0/0 interface of Vagrant ios-xrv64 with eth1 of the devbox as shown in the figure below.  

The two topologies in use are:  

### Vagrant Setup

<div class="notice">
<img src="https://xrdocs.github.io/xrdocs-images/assets/images/vagrant_docker_topo.png" alt="vagrant docker topo" style="padding:1px;border:thin solid black;">
</div>

### NCS5500 and ASR9k Setup

<div class="notice">
<img src="https://xrdocs.github.io/xrdocs-images/assets/images/ncs5500_docker_topo.png" alt="NCS5500 docker topo" style="padding:1px;border:thin solid black;">
</div>


  
  
## Install docker-engine on the devbox

### Vagrant setup
For the Vagrant setup, you will see a script called `docker_install.sh` under the scripts folder:  

```shell
AKSHSHAR-M-K0DS:docker-app-topo-bootstrap akshshar$ pwd
/Users/akshshar/vagrant-xrdocs/docker-app-topo-bootstrap
AKSHSHAR-M-K0DS:docker-app-topo-bootstrap akshshar$ ls
Vagrantfile	configs		scripts
AKSHSHAR-M-K0DS:docker-app-topo-bootstrap akshshar$ ls scripts/
apply_config.sh		docker_install.sh
AKSHSHAR-M-K0DS:docker-app-topo-bootstrap akshshar$ 

```

This is the vagrant provisioner for the devbox and will install docker-engine on boot (vagrant up).  


### NCS5500 and ASR9k setup

In this case, the devbox must be provisioned by the user. On an ubuntu devbox, docker-engine can be installed by following the instructions at:  
  
><https://docs.docker.com/engine/installation/linux/ubuntu/>  
  
    

Perfect! Now we're all set with the topology and SSH access. Before we begin, let's understand the docker daemon/client setup inside IOS-XR.
{: .notice--success}  
  
  
## Docker Daemon support on IOS-XR


### Vagrant and NCS5500 architecture  

If you haven't already gone through the basic overview on the application hosting infrastructure on XR, I would urge you to have a quick read:  
  
><{{ base_path }}/blogs/2016-06-28-xr-app-hosting-architecture-quick-look/>  
  
>
**Relevant Platforms**: The LXC architecture described above and expanded on below is relevant to the following platforms:  
>
  *  NCS5500 (NCS5501, NCS5501-SE, NCS5502, NCS5502-SE, NCS5508, NCS5516)
  *  NCS5000
  *  NCS5011
  *  XRv9k
  *  IOS-XRv64 (Vagrant box and ISO)  
  
  
From the above article it becomes fairly clear that internally the IOS-XR architecture involves a Host layer running the libvirtd daemon and IOS-XR runs as an LXC spawned using the daemon.  

Further, the "virsh" client is provided within the XR LXC, so that a user may have client level access to the daemon while sitting inside the XR LXC itself.  

The setup for launching LXCs in IOS-XR is shown below:  

[![xr-lxc](https://xrdocs.github.io/xrdocs-images/assets/images/xr_lxc.png)](https://xrdocs.github.io/xrdocs-images/assets/images/xr_lxc.png)  
  
  
  
The Docker client/daemon setup follows the exact same principle as shown below. Docker Daemon runs on the host and Docker client is made available inside the XR LXC for easy operationalization:  
  
[![xr-docker](https://xrdocs.github.io/xrdocs-images/assets/images/xr_docker.png)](https://xrdocs.github.io/xrdocs-images/assets/images/xr_docker.png)  

### ASR9k architecture  

The ASR9k architecture is slightly different. In ASR9k, IOS-XR runs inside its own VM on the 64-bit Linux host to be able to support ISSU requirements relevant to traditional Service Provider deployments.

In this case, the libvirtd and docker daemons are available inside the XR control plane VM itself.
This does not change the user experience from a docker client or virsh client perspective. 
The difference is mainly how one may interact with the docker daemon as we'll touch upon in subsequent sections.  

This is what the architecture looks like for ASR9k:  


**ASR9k LXC/libvirt Setup:**

[![xr_asr9k_libvirt_libvirt](https://xrdocs.github.io/xrdocs-images/assets/images/xr_asr9k_libvirt_architecture.png)](https://xrdocs.github.io/xrdocs-images/assets/images/xr_asr9k_libvirt_architecture.png)

Libvirt daemon is local to the XR control plane VM.


**ASR9k Docker Setup:**  

[![xr_asr9k_docker_libvirt](https://xrdocs.github.io/xrdocs-images/assets/images/xr_asr9k_docker_architecture.png)](https://xrdocs.github.io/xrdocs-images/assets/images/xr_asr9k_docker_architecture.png)

Docker daemon is local to the XR control plane VM.


Alright, so can we verify this?  



### Vagrant setup Docker Client Access 

On your vagrant box, there are two ways to get access to the docker client:

*  **Drop into the "bash" shell from XR CLI:** Using "bash" ensures that the correct environment 
   variables are sourced to gain access to the Docker Daemon on the host:   

   **Password for the XR CLI:   vagrant**
   {: .notice--info}  
   
  
    <div class="highlighter-rouge">
    <pre class="highlight" style="white-space: pre-wrap;">
    <code>
    AKSHSHAR-M-K0DS:docker-app-topo-bootstrap akshshar$<mark> vagrant port rtr </mark>
    The forwarded ports for the machine are listed below. Please note that
    these values may differ from values configured in the Vagrantfile if the
    provider supports automatic port collision detection and resolution.

        <mark>22 (guest) => 2223 (host)</mark>
     57722 (guest) => 2222 (host)
    AKSHSHAR-M-K0DS:docker-app-topo-bootstrap akshshar$ 
    AKSHSHAR-M-K0DS:docker-app-topo-bootstrap akshshar$ 
    AKSHSHAR-M-K0DS:docker-app-topo-bootstrap akshshar$ 
    AKSHSHAR-M-K0DS:docker-app-topo-bootstrap akshshar$<mark> ssh -p 2223 vagrant@localhost</mark>
    The authenticity of host '[localhost]:2223 ([127.0.0.1]:2223)' can't be established.
    RSA key fingerprint is SHA256:uHev9uiAa0LM36RnnxDYuRyKywra8Oe/G5Gt34OiBqk.
    Are you sure you want to continue connecting (yes/no)? yes
    Warning: Permanently added '[localhost]:2223' (RSA) to the list of known hosts.
    vagrant@localhost's password: 


    RP/0/RP0/CPU0:ios#
    RP/0/RP0/CPU0:ios#
    RP/0/RP0/CPU0:ios#<mark>bash</mark>
    Sun Mar  5 18:17:18.380 UTC

    [xr-vm_node0_RP0_CPU0:~]$
    [xr-vm_node0_RP0_CPU0:~]$
    [xr-vm_node0_RP0_CPU0:~]$whoami
    <mark>root</mark>
    [xr-vm_node0_RP0_CPU0:~]$
    [xr-vm_node0_RP0_CPU0:~]$<mark>docker ps</mark>
    CONTAINER ID    IMAGE      COMMAND      CREATED       STATUS        PORTS         NAMES
    [xr-vm_node0_RP0_CPU0:~]$
    </code>
    </pre>
    </div> 
    
    Bear in mind that when you drop into the XR linux shell using the "bash" CLI, you are droppped
    in as `root`. This is why you can access the docker client without any hassle. 
    For any other  user, you will need to first become root (using sudo).  
    {: .notice--warning}  
      
      
*  **Drop directly into the Linux shell over SSH (port 57722):**  

   From the above output for `vagrant port rtr`, the port 57722 on XR (running openssh in the XR linux shell) is accessible via port 2222 on the host machine (laptop):  
   
   Use either `vagrant ssh rtr` or `ssh -p 2222 vagrant@localhost` to drop into the XR linux shell
   
   **Username: vagrant  
   Password: vagrant** 
   {: .notice--warning}  
    

    <div class="highlighter-rouge">
    <pre class="highlight" style="white-space: pre-wrap;">
    <code>
   
    AKSHSHAR-M-K0DS:docker-app-topo-bootstrap akshshar$<mark> vagrant ssh rtr</mark>
    Last login: Sun Mar  5 18:55:20 2017 from 10.0.2.2
    xr-vm_node0_RP0_CPU0:~$ 
    xr-vm_node0_RP0_CPU0:~$whoami
    <mark>vagrant</mark>
    xr-vm_node0_RP0_CPU0:~$ 
    xr-vm_node0_RP0_CPU0:~$<mark> sudo -i </mark>
    [xr-vm_node0_RP0_CPU0:~]$ 
    [xr-vm_node0_RP0_CPU0:~]$ whoami
    <mark>root</mark>
    [xr-vm_node0_RP0_CPU0:~]$ 
    [xr-vm_node0_RP0_CPU0:~]$<mark> docker ps</mark>
    CONTAINER ID      IMAGE       COMMAND       CREATED       STATUS       PORTS        NAMES
    [xr-vm_node0_RP0_CPU0:~]$ 
    </code>
    </pre>
    </div> 
    
    As shown above, we become root by using `-i` flag for `sudo` to make sure the correct environment variables are sourced.
    {: .notice--info}  
    
    

### NCS5500 and ASR9k Docker Client Access

If you followed the steps in the pre-requisites section above : [Pre-requisites](https://xrdocs.github.io/application-hosting/tutorials/2017-02-26-running-docker-containers-on-ios-xr-6-1-2/#physical-ncs5500-router), you would already have access to your NCS5500/ASR9k device over XR SSH (CLI, port 22) as well as sshd_operns (XR linux shell, port 57722)


Following the Vagrant model, over XR SSH, we use the "bash" CLI to access the docker client on the NCS5500/ASR9k:

**Note**: The steps for ASR9k are identical. NCS5500 steps are shown below. 
{: .notice--warning}  


```
cisco@dhcpserver:~$ 
cisco@dhcpserver:~$ ssh root@11.11.11.59
The authenticity of host '11.11.11.59 (11.11.11.59)' can't be established.
RSA key fingerprint is 8a:42:49:bf:4c:cd:f9:3c:e1:19:f9:02:b6:3a:ad:01.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '11.11.11.59' (RSA) to the list of known hosts.
Password: 


RP/0/RP0/CPU0:ncs5508#
RP/0/RP0/CPU0:ncs5508#
RP/0/RP0/CPU0:ncs5508#bash
Mon Mar  6 09:36:37.221 UTC

[ncs5508:~]$whoami
root
[ncs5508:~]$docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
[ncs5508:~]$
[ncs5508:~]$

```

Similarly, for direct access to the linux shell, we ssh over 57722, become sudo and then access the docker client:

SSH password and sudo password for user cisco will be whatever you've set up during the Pre-requisites stage.
{: .notice-info} 

```
cisco@dhcpserver:~$ ssh cisco@11.11.11.59 -p 57722
cisco@11.11.11.59's password: 
Permission denied, please try again.
cisco@11.11.11.59's password: 
Last login: Mon Mar  6 06:30:47 2017 from 11.11.11.2
-sh: /var/log/boot.log: Permission denied
ncs5508:~$ 
ncs5508:~$ 
ncs5508:~$ sudo -i
Password: 
[ncs5508:~]$ 
[ncs5508:~]$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
[ncs5508:~]$ 
```



## Launch a Docker Container

As discussed earlier, we'll showcase a few different techniques through which a user may spin up a docker container on IOS-XR.


## Public Dockerhub registry

This is the simplest setup that most docker users would know already. The obvious configuration necessary would be to make sure connectivity to the internet is available from the router.

This may not be the preferred setup for production deployments, understandably, since direct connectivity to the internet from a production router is not typical. The next few techniques with private registries or tarball based docker container bringup might be more your cup of tea, in that case.
{: .notice--info} 


### Vagrant Setup  

The vagrant IOS-XR box comes with connectivity to the internet already. All you need to do is set up the domain name-server in the global-vrf (before 6.3.1, we only support the global/default vrf for the docker daemon image downloads).  

Remember that we're setting up this domain name on per vrf basis. In the future, we intend to sync this through XR CLI for all vrfs to the corresponding network namespaces. Before 6.3.1, of course only global-vrf may be used.

Update `/etc/netns/global-vrf/resolv.conf` to point to a reachable nameserver, in this case 8.8.8.8:

```
[xr-vm_node0_RP0_CPU0:~]$cat /etc/netns/global-vrf/resolv.conf
nameserver 8.8.8.8
[xr-vm_node0_RP0_CPU0:~]$
```

Again, become root with the correct environment (sudo -i)  to execute the relevant docker commands to spin up the container.

<div class="highlighter-rouge">
<pre class="highlight" style="white-space: pre-wrap;">
<code>
[xr-vm_node0_RP0_CPU0:~]$sudo -i
[xr-vm_node0_RP0_CPU0:~]$ 
[xr-vm_node0_RP0_CPU0:~]$ whoami    
root
[xr-vm_node0_RP0_CPU0:~]$ 
[xr-vm_node0_RP0_CPU0:~]$
[xr-vm_node0_RP0_CPU0:~]$docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
[xr-vm_node0_RP0_CPU0:~]$docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
[xr-vm_node0_RP0_CPU0:~]$
[xr-vm_node0_RP0_CPU0:~]$
[xr-vm_node0_RP0_CPU0:~]$
[xr-vm_node0_RP0_CPU0:~]$ docker run -itd --name ubuntu -v /var/run/netns/global-vrf:/var/run/netns/global-vrf --cap-add=SYS_ADMIN ubuntu bash
Unable to find image 'ubuntu:latest' locally
latest: Pulling from library/ubuntu
d54efb8db41d: Pull complete 
f8b845f45a87: Pull complete 
e8db7bf7c39f: Pull complete 
9654c40e9079: Pull complete 
6d9ef359eaaa: Pull complete 
Digest: sha256:dd7808d8792c9841d0b460122f1acf0a2dd1f56404f8d1e56298048885e45535
Status: Downloaded newer image for ubuntu:latest
495ec2ab0b201418999e159b81a934072be504b05cc278192d8152efd4965635
[xr-vm_node0_RP0_CPU0:~]$ 
[xr-vm_node0_RP0_CPU0:~]$ 
[xr-vm_node0_RP0_CPU0:~]$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
495ec2ab0b20        ubuntu              "bash"              7 minutes ago       Up 7 minutes                            ubuntu
[xr-vm_node0_RP0_CPU0:~]$ 
```  
</code>
</pre>
</div>

>
You will notice two peculiar things in the command we run:
>
*  **Mounting of /var/run/netns/&lt;vrf-name&gt;**: We mount /var/run/netns/&lt;vrf-name&gt; into the docker container. This is an option we use to mount the appropriate network namespace(s) (one or more -v options may be used) into the container. These network namespaces (XR release 6.3.1+) are created on the host and then bind-mounted into the XR LXC for user convenience. In case of ASR9k, these network namespaces are local. The docker container, running on the host (inside XR VM in case of ASR9k), will simply inherit these network namespaces through the /var/run/netns/&lt;vrf-name&gt; mount. **Each Network namespace may correspond to a VRF in XR (CLI option to achieve this will be available post 6.3.1. Bear in mind that before 6.3.1 release only the `global-vrf` is supported in the XR linux shell**.  
>
*  **--cap-add=SYS_ADMIN flag**: We're using the `--cap-add=SYS_ADMIN` flag because even when network namespaces are mounted from the "host" (or XR VM in case of ASR9k) into the docker container, a user can change into a particular network namespace or execute commands in a particular namespace, only if the container is launched with privileged capabilties.


Yay! The container's running. We can get into the container by starting bash through a docker exec. If you're running container images that do not support a shell, try docker attach instead.

```shell
[xr-vm_node0_RP0_CPU0:~]$
[xr-vm_node0_RP0_CPU0:~]$
[xr-vm_node0_RP0_CPU0:~]$docker exec -it ubuntu bash
root@bf408eb70f88:/# 
root@bf408eb70f88:/# cat /etc/*-release 
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=16.04
DISTRIB_CODENAME=xenial
DISTRIB_DESCRIPTION="Ubuntu 16.04.2 LTS"
NAME="Ubuntu"
VERSION="16.04.2 LTS (Xenial Xerus)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 16.04.2 LTS"
VERSION_ID="16.04"
HOME_URL="http://www.ubuntu.com/"
SUPPORT_URL="http://help.ubuntu.com/"
BUG_REPORT_URL="http://bugs.launchpad.net/ubuntu/"
VERSION_CODENAME=xenial
UBUNTU_CODENAME=xenial
root@bf408eb70f88:/# 

```

### NCS5500 and ASR9k Setup
  
Remember the topology for the NCS5508/ASR9k setup?: [NCS5500 and ASR9k Setup Topology](https://xrdocs.github.io/application-hosting/tutorials/2017-02-26-running-docker-containers-on-ios-xr-6-1-2/#physical-ncs5500-and-asr9k)

In order to reach the internet, the NCS5508/ASR9k needs to be configured with a default route through the Management port which is NAT-ted (using iptables Masquerade rules, not shown here) to the outside world through devbox.

**Note:** Steps below are applicable to ASR9k as well.
{: .notice--warning}

Read the note below if you need a refresher on the routing in XR's linux kernel: 
  
  
>
**Setting up Default routes in the Linux Kernel:**
>
For those who understand the basic principle behind the IOS-XR Packet I/O architecture for Linux application traffic (see here: [Application hosting Infrastructure in IOS-XR](><{{ base_path }}/blogs/2016-06-28-xr-app-hosting-architecture-quick-look/>) ), it might be clear that routes in the linux kernel are controlled through the "tpa" CLI.
>
This leads to 3 types of routes:
1.  Default route through "fwdintf" : To allow packets through the front panel ports by default. Herein the update-source CLI is used to set the source IP address of the packets.
2.  East-West route through "fwd_ew" : This enables packets to flow between XR and a linux app running in a given vrf (network namespace - only global-vrf supported before 6.3.1 release).
3.  Management Subnet:  The directly connected subnet for the Management port as well non-default routes in the RIB through the Management port.


To set up a default route through the Management port:

**Prior to 6.3.1 release**

Prior to 6.3.1, there is no direct knob in the tpa CLI to help set this up. So we drop into the linux shell directly and set the default route ourselves:

```
RP/0/RP0/CPU0:ncs5508#bash
Wed Mar  8 02:06:54.590 UTC

[ncs5508:~]$
[ncs5508:~]$
[ncs5508:~]$ip route
default dev fwdintf  scope link  src 1.1.1.1 
10.10.10.10 dev fwd_ew  scope link  src 1.1.1.1 
11.11.11.0/24 dev Mg0_RP0_CPU0_0  proto kernel  scope link  src 11.11.11.59 
[ncs5508:~]$
[ncs5508:~]$
[ncs5508:~]$ip route del default
[ncs5508:~]$ip route add default via 11.11.11.2 dev Mg0_RP0_CPU0_0
[ncs5508:~]$
[ncs5508:~]$ip route
default via 11.11.11.2 dev Mg0_RP0_CPU0_0 
10.10.10.10 dev fwd_ew  scope link  src 1.1.1.1 
11.11.11.0/24 dev Mg0_RP0_CPU0_0  proto kernel  scope link  src 11.11.11.59 
[ncs5508:~]$

```
Having done the above change, set up the DNS server in global-vrf network namespace, much like in the Vagrant setup:


```
[ncs5508:~]$cat /etc/netns/global-vrf/resolv.conf
nameserver ######
[ncs5508:~]$
```

Of course, use an actual IP address of the DNS server in your network, and not #####. I use it to simply hide the private DNS IP in my setup :)
{: .notice--info}


<div class="highlighter-rouge">
<pre class="highlight" style="white-space: pre-wrap;">
<code>
[ncs5508:~]$
[ncs5508:~]$
[ncs5508:~]$docker run -itd --name ubuntu --cap-add=SYS_ADMIN -v /var/run/netns:/var/run/netns ubuntu bash
Unable to find image 'ubuntu:latest' locally
latest: Pulling from library/ubuntu
d54efb8db41d: Pull complete 
f8b845f45a87: Pull complete 
e8db7bf7c39f: Pull complete 
9654c40e9079: Pull complete 
6d9ef359eaaa: Pull complete 
Digest: sha256:dd7808d8792c9841d0b460122f1acf0a2dd1f56404f8d1e56298048885e45535
Status: Downloaded newer image for ubuntu:latest
67b781a19b5a164d77ee7ed95201c422e70be57c9ee6547a7e8e9457f8db514b
[ncs5508:~]$
[ncs5508:~]$
[ncs5508:~]$docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
67b781a19b5a        ubuntu              "bash"              3 minutes ago       Up 3 minutes                            ubuntu
[ncs5508:~]$

</code>
</pre>
</div>

**Post 6.3.1 release**

Post 6.3.1, the default route wouldn't have to be set using the linux command (ip route default...). We have introduced a default-route CLI under tpa (along with vrfs, but more on that in another blog).   

The CLI will look something like :

<div class="highlighter-rouge">
<pre class="highlight">
<code>
tpa
  vrf &lt;vrf-name&gt;
    address-family ipv4[ipv6]
      default-route east-west
</code>
</pre>
</div>


The advantage of introducing a CLI is that it helps handle the routes in the linux kernel across reloads and switchovers as well.



## Private "insecure" registry

This is a straightforward technique when a user expects to bring up private registries for their docker images in a secure part of the network (so that connection between the registry and the router doesn't necessarily need to be secured) :

*  We spin up an insecure docker registry(which is itself a docker container pulled down from dockerhub) on our devbox.

*  We then modify /etc/sysconfig/docker in XR linux to add the insecure registry information 

*  Set up the route to the registry

*  Populate the registry with some docker images from dockerhub

*  Pull the relevant images from the insecure registry down to XR's docker daemon and spin up containers



### Setting up the insecure registry

Let's begin by spinning up a registry on the devbox in our Vagrant setup. The same exact steps are relevant to the devbox environment on the NCS5500/ASR9k setup as well. 
We follow the steps described here: <https://docs.docker.com/registry/deploying/>


<div class="highlighter-rouge">
<pre class="highlight" style="white-space: pre-wrap;">
<code>
AKSHSHAR-M-K0DS:docker-app-topo-bootstrap akshshar$<mark>vagrant ssh devbox </mark>
Welcome to Ubuntu 14.04.5 LTS (GNU/Linux 3.13.0-95-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

 System information disabled due to load higher than 1.0

  Get cloud support with Ubuntu Advantage Cloud Guest:
    http://www.ubuntu.com/business/services/cloud

0 packages can be updated.
0 updates are security updates.

New release '16.04.2 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$<mark> sudo -s </mark>
root@vagrant-ubuntu-trusty-64:~# 
root@vagrant-ubuntu-trusty-64:~# <mark>docker run -d -p 5000:5000 --restart=always --name registry registry:2</mark>
Unable to find image 'registry:2' locally
2: Pulling from library/registry
709515475419: Pull complete 
df6e278d8f96: Pull complete 
16218e264e88: Pull complete 
16748da81f63: Pull complete 
8d73e673c34c: Pull complete 
Digest: sha256:28be0609f90ef53e86e1872a11d672434ce1361711760cf1fe059efd222f8d37
Status: Downloaded newer image for registry:2
b6a2a5fef7b7c201ee4d162b56f1e35054e25225ad27ad3fbf3a267d2ef9fb7a
root@vagrant-ubuntu-trusty-64:~# 
root@vagrant-ubuntu-trusty-64:~#<mark> docker pull ubuntu && docker tag ubuntu localhost:5000/ubuntu</mark>
Using default tag: latest
latest: Pulling from library/ubuntu
d54efb8db41d: Pull complete 
f8b845f45a87: Pull complete 
e8db7bf7c39f: Pull complete 
9654c40e9079: Pull complete 
6d9ef359eaaa: Pull complete 
Digest: sha256:dd7808d8792c9841d0b460122f1acf0a2dd1f56404f8d1e56298048885e45535
Status: Downloaded newer image for ubuntu:latest
root@vagrant-ubuntu-trusty-64:~# 
root@vagrant-ubuntu-trusty-64:~#<mark> docker push localhost:5000/ubuntu </mark>
The push refers to a repository [localhost:5000/ubuntu]
56827159aa8b: Pushed 
440e02c3dcde: Pushed 
29660d0e5bb2: Pushed 
85782553e37a: Pushed 
745f5be9952c: Pushed 
latest: digest: sha256:6b079ae764a6affcb632231349d4a5e1b084bece8c46883c099863ee2aeb5cf8 size: 1357
root@vagrant-ubuntu-trusty-64:~# 
root@vagrant-ubuntu-trusty-64:~# 

 </code>
 </pre>
 </div> 


In the above steps, we've simply set up the registry on the devbox, pulled down an ubuntu docker image from dockerhub and pushed the image to the local registry.


### Vagrant Setup

Before we start let's come back to square-one on our Vagrant setup. Delete the previously running container and downloaded image:

```
[xr-vm_node0_RP0_CPU0:~]$ docker stop ubuntu && docker rm ubuntu
ubuntu
ubuntu
[xr-vm_node0_RP0_CPU0:~]$ docker rmi ubuntu
Untagged: ubuntu:latest
Deleted: sha256:0ef2e08ed3fabfc44002ccb846c4f2416a2135affc3ce39538834059606f32dd
Deleted: sha256:0d58a35162057295d273c5fb8b7e26124a31588cdadad125f4bce63b638dddb5
Deleted: sha256:cb7f997e049c07cdd872b8354052c808499937645f6164912c4126015df036cc
Deleted: sha256:fcb4581c4f016b2e9761f8f69239433e1e123d6f5234ca9c30c33eba698487cc
Deleted: sha256:b53cd3273b78f7f9e7059231fe0a7ed52e0f8e3657363eb015c61b2a6942af87
Deleted: sha256:745f5be9952c1a22dd4225ed6c8d7b760fe0d3583efd52f91992463b53f7aea3
[xr-vm_node0_RP0_CPU0:~]$ 
```


Now let's set up XR's docker daemon to accept the insecure registry located on the directly connected network on Gig0/0/0/0.   


Based off the config applied via the Vagrantfile, the reachable IP address of the registry running on devbox = 11.1.1.20, port 5000.
{: .notice--info}


Log into XR CLI. We will first make sure that the request from XR's docker daemon originates with a source IP that is reachable from the docker registry. So set the TPA ip address = Gig0/0/0/0 ip address (directly connected subnet):  


```
RP/0/RP0/CPU0:ios(config)#tpa
RP/0/RP0/CPU0:ios(config-tpa)#address-family ipv4 ?
  update-source  Update the Source for Third Party
  <cr>           
RP/0/RP0/CPU0:ios(config-tpa)#address-family ipv4 
RP/0/RP0/CPU0:ios(config-tpa-afi)#update-source gigabitEthernet 0/0/0/0 
RP/0/RP0/CPU0:ios(config-tpa-afi)#commit
Mon Mar  6 05:08:32.436 UTC
RP/0/RP0/CPU0:ios(config-tpa-afi)#

```

This should lead to the following routes in the linux kernel:

```
RP/0/RP0/CPU0:ios#
RP/0/RP0/CPU0:ios#bash
Mon Mar  6 05:35:49.459 UTC

[xr-vm_node0_RP0_CPU0:~]$ip route
default dev fwdintf  scope link  src 11.1.1.10 
10.0.2.0/24 dev Mg0_RP0_CPU0_0  proto kernel  scope link  src 10.0.2.15 
[xr-vm_node0_RP0_CPU0:~]$

```

Before we launch the container, we need to configure the XR docker daemon to disregard security for our registry. This is done by modifying `/etc/sysconfig/docker` inside the XR LXC. My eventual configuration looks something like:


<div class="highlighter-rouge">
<pre class="highlight">
<code>
[xr-vm_node0_RP0_CPU0:~]$cat /etc/sysconfig/docker
# DOCKER_OPTS can be used to add insecure private registries to be supported 
# by the docker daemon
# eg : DOCKER_OPTS="--insecure-registry foo --insecure-registry bar"

# Following are the valid configs
# DOCKER_OPTS="&lt;space&gt;--insecure-registry&lt;space&gt;foo"
# DOCKER_OPTS+="&lt;space&gt;--insecure-registry&lt;space&gt;bar"

<mark>DOCKER_OPTS=" --insecure-registry 11.1.1.20:5000"</mark>
[xr-vm_node0_RP0_CPU0:~]$
</code>
</pre>
</div> 

As the instructions/comments inside the file indicate, make sure there is a space before --insecure-registry flag. Further, in a normal docker daemon setup, a user is supposed to restart the docker daemon when changes to /etc/sysconfig/docker are made. In case of XR, this is not needed. We handle automatic restarts of the docker daemon when a user makes changes to /etc/sysconfig/docker and saves it.   
Further, since the docker daemon will be automatically restarted, wait for about 10-15 seconds before issuing any docker commands.
{: .notice--info}


Now issue the docker run command to launch the container on XR.

<div class="highlighter-rouge">
<pre class="highlight" style="white-space: pre-wrap;">
<code>
RP/0/RP0/CPU0:ios#
RP/0/RP0/CPU0:ios#bash
Mon Mar  6 05:51:14.341 UTC
[xr-vm_node0_RP0_CPU0:~]$
[xr-vm_node0_RP0_CPU0:~]$docker run -itd --name ubuntu -v /var/run/netns --cap-add=SYS_ADMIN 11.1.1.20:5000/ubuntu bash
Unable to find image '11.1.1.20:5000/ubuntu:latest' locally
latest: Pulling from ubuntu
fec6b243e075: Pull complete 
190e0e9a3e79: Pull complete 
0d79cf192e4c: Pull complete 
38398c307b51: Pull complete 
356665655a72: Pull complete 
Digest: sha256:6b079ae764a6affcb632231349d4a5e1b084bece8c46883c099863ee2aeb5cf8
Status: Downloaded newer image for 11.1.1.20:5000/ubuntu:latest
bf408eb70f88c8050c29fb46610d354a113a46edbece105acc68507e71442d38
[xr-vm_node0_RP0_CPU0:~]$
[xr-vm_node0_RP0_CPU0:~]$
[xr-vm_node0_RP0_CPU0:~]$docker ps
CONTAINER ID        IMAGE                   COMMAND             CREATED             STATUS              PORTS               NAMES
bf408eb70f88        11.1.1.20:5000/ubuntu   "bash"              8 seconds ago       Up 8 seconds                            ubuntu
[xr-vm_node0_RP0_CPU0:~]$

</code>
</pre>
</div>

There, you've launched a docker container on XR using a private "insecure" registry.
{: .notice--success}  


### NCS5500 setup  

The workflow is more or less identical to the Vagrant setup.
In this case we're setting up the registry to be reachable over the Management network (and over the same subnet). For this, you don't need to set the TPA IP.  

If you've followed the steps above in the [Setting up the Insecure Registry](https://xrdocs.github.io/application-hosting/tutorials/2017-02-26-running-docker-containers-on-ios-xr-6-1-2/#setting-up-the-insecure-registry) section, then you should have an insecure registry already running on the devbox environment, along with a "pushed" ubuntu image.  


Now hop over to the NCS5500 and issue the "bash" CLI. Your "ip route" setup should look something like this:


<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:ncs5508#bash
Tue Mar  7 00:29:56.416 UTC

[ncs5508:~]$ip route
<mark>default dev fwdintf  scope link  src 1.1.1.1</mark>
10.10.10.10 dev fwd_ew  scope link  src 1.1.1.1 
<mark>11.11.11.0/24 dev Mg0_RP0_CPU0_0  proto kernel  scope link  src 11.11.11.59</mark>
[ncs5508:~]$
[ncs5508:~]$
[ncs5508:~]$

</code>
</pre>
</div> 

We won't be leveraging the tpa setup for the fwdintf interface (meant for reachability over front panel/data ports) and instead just use the local management network subnet (11.11.11.0/24) for reachability to the docker registry.


Further, much like before, set up `/etc/sysconfig/docker` to disregard security for our registry.

```
[ncs5508:~]$cat /etc/sysconfig/docker
# DOCKER_OPTS can be used to add insecure private registries to be supported 
# by the docker daemon
# eg : DOCKER_OPTS="--insecure-registry foo --insecure-registry bar"

# Following are the valid configs
# DOCKER_OPTS="<space>--insecure-registry<space>foo"
# DOCKER_OPTS+="<space>--insecure-registry<space>bar"

DOCKER_OPTS=" --insecure-registry 11.11.11.2:5000"
[ncs5508:~]$
```

When you make the above change,the docker daemon will be automatically restarted. Wait for about 10-15 seconds before issuing any docker commands.
{: .notice--info}

Now we can issue a docker run (or docker pull followed by a docker run) to download and launch the docker ubuntu image from the registry.

<div class="highlighter-rouge">
<pre class="highlight" style="white-space: pre-wrap;">
<code>
[ncs5508:~]$docker run -itd --name ubuntu -v /var/run/netns --cap-add=SYS_ADMIN 11.11.11.2:5000/ubuntu
Unable to find image '11.11.11.2:5000/ubuntu:latest' locally
latest: Pulling from ubuntu
d54efb8db41d: Pull complete 
f8b845f45a87: Pull complete 
e8db7bf7c39f: Pull complete 
9654c40e9079: Pull complete 
6d9ef359eaaa: Pull complete 
Digest: sha256:dd7808d8792c9841d0b460122f1acf0a2dd1f56404f8d1e56298048885e45535
Status: Downloaded newer image for 11.11.11.2:5000/ubuntu:latest
aa73f6a81b9346131118b84f30ddfc2d3bd981a4a54ea21ba2e2bc5c3d18d348
[ncs5508:~]$
[ncs5508:~]$docker ps
CONTAINER ID        IMAGE                    COMMAND             CREATED             STATUS              PORTS               NAMES
aa73f6a81b93        11.11.11.2:5000/ubuntu   "/bin/bash"         4 hours ago         Up 4 hours                              ubuntu
[ncs5508:~]$

</code>
</pre>
</div>



### ASR9k setup

The ASR9k setup for an insecure docker registry is slightly different from Vagrant IOS-XR or NCS platforms. There is no automatic mechanism to restart the docker daemon.

**The user must restart the docker daemon once they modify the /etc/sysconfig/docker file.**
{: notice--info}  


Again, we're setting up the registry to be reachable over the Management network (and over the same subnet). For this, you don't need to set the TPA IP.  


Now hop over to the ASR9k and issue the "bash" CLI. Your "ip route" setup should look something like this:


<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RSP1/CPU0:asr9k#bash
Tue Mar  7 00:29:56.416 UTC

[asr9k:~]$ip route
<mark>default dev fwdintf  scope link  src 1.1.1.1</mark>
10.10.10.10 dev fwd_ew  scope link  src 1.1.1.1 
<mark>11.11.11.0/24 dev Mg0_RP0_CPU0_0  proto kernel  scope link  src 11.11.11.59</mark>
[asr9k:~]$
[asr9k:~]$
[asr9k:~]$

</code>
</pre>
</div> 

We won't be leveraging the tpa setup for the fwdintf interface (meant for reachability over front panel/data ports) and instead just use the local management network subnet (11.11.11.0/24) for reachability to the docker registry.


Further, much like before, set up `/etc/sysconfig/docker` to disregard security for our registry.

```
[asr9k:~]$cat /etc/sysconfig/docker
# DOCKER_OPTS can be used to add insecure private registries to be supported 
# by the docker daemon
# eg : DOCKER_OPTS="--insecure-registry foo --insecure-registry bar"

# Following are the valid configs
# DOCKER_OPTS="<space>--insecure-registry<space>foo"
# DOCKER_OPTS+="<space>--insecure-registry<space>bar"

DOCKER_OPTS=" --insecure-registry 11.11.11.2:5000"
[asr9k:~]$
```

**Important:** For the ASR9k, you need to restart the docker daemon for the above config change to take effect.    
{: .notice--warning}  


```
[asr9k:~]$service docker restart
docker stop/waiting
docker start/running, process 12276
[asr9k:~]$
```

Now we can issue a docker run (or docker pull followed by a docker run) to download and launch the docker ubuntu image from the registry.

<div class="highlighter-rouge">
<pre class="highlight" style="white-space: pre-wrap;">
<code>
[asr9k:~]$docker run -itd --name ubuntu -v /var/run/netns --cap-add=SYS_ADMIN 11.11.11.2:5000/ubuntu
Unable to find image '11.11.11.2:5000/ubuntu:latest' locally
latest: Pulling from ubuntu
d54efb8db41d: Pull complete 
f8b845f45a87: Pull complete 
e8db7bf7c39f: Pull complete 
9654c40e9079: Pull complete 
6d9ef359eaaa: Pull complete 
Digest: sha256:dd7808d8792c9841d0b460122f1acf0a2dd1f56404f8d1e56298048885e45535
Status: Downloaded newer image for 11.11.11.2:5000/ubuntu:latest
aa73f6a81b9346131118b84f30ddfc2d3bd981a4a54ea21ba2e2bc5c3d18d348
[ncs5508:~]$
[ncs5508:~]$docker ps
CONTAINER ID        IMAGE                    COMMAND             CREATED             STATUS              PORTS               NAMES
aa73f6a81b93        11.11.11.2:5000/ubuntu   "/bin/bash"         4 hours ago         Up 4 hours                              ubuntu
[asr9k:~]$

</code>
</pre>
</div>


## Private Self-Signed Registry


This technique is a bit more secure than the insecure registry setup and may be used to more or less secure the connection between the router's docker daemon and the docker registry running externally. The basic steps involved are:


* Generate your own certificate on the devbox

* Use the result to start your docker registry with TLS enabled

* Copy the certificates to the /etc/docker/certs.d/ folder on the router

* Don’t forget to restart the Docker daemon for the ASR9k. In case of other platforms, the restart is automatic  

*  Set up the route to the registry

*  Populate the registry with some docker images from dockerhub

*  Pull the relevant images from the registry down to XR's docker daemon and spin up containers


### Setting up a self-signed Docker Registry  

<div class="highlighter-rouge">
<pre class="highlight" style="white-space: pre-wrap;">
<code>
AKSHSHAR-M-K0DS:docker-app-topo-bootstrap akshshar$ vagrant ssh devbox
Welcome to Ubuntu 14.04.5 LTS (GNU/Linux 3.13.0-95-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

 System information disabled due to load higher than 1.0

  Get cloud support with Ubuntu Advantage Cloud Guest:
    http://www.ubuntu.com/business/services/cloud

0 packages can be updated.
0 updates are security updates.

New release '16.04.2 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$<mark> mkdir -p certs && openssl req -newkey rsa:4096 -nodes -sha256 -keyout certs/domain.key -x509 -days 365 -out certs/domain.crt </mark>
Generating a 4096 bit RSA private key
.......................++
..........................++
writing new private key to 'certs/domain.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:
State or Province Name (full name) [Some-State]:
Locality Name (eg, city) []:
Organization Name (eg, company) [Internet Widgits Pty Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:<mark>devbox.com</mark>
Email Address []:
vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$ cd certs/
vagrant@vagrant-ubuntu-trusty-64:~/certs$ ls
domain.crt  domain.key
vagrant@vagrant-ubuntu-trusty-64:~/certs$ 
vagrant@vagrant-ubuntu-trusty-64:~/certs$ 
vagrant@vagrant-ubuntu-trusty-64:~/certs$ 
vagrant@vagrant-ubuntu-trusty-64:~/certs$ cd ..
vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$ <mark>sudo docker run -d -p 5000:5000 --restart=always --name registry -v `pwd`/certs:/certs -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key registry:2 </mark>
Unable to find image 'registry:2' locally
2: Pulling from library/registry
709515475419: Pull complete 
df6e278d8f96: Pull complete 
16218e264e88: Pull complete 
16748da81f63: Pull complete 
8d73e673c34c: Pull complete 
Digest: sha256:28be0609f90ef53e86e1872a11d672434ce1361711760cf1fe059efd222f8d37
Status: Downloaded newer image for registry:2
c423ae398af2ec05fabd9c1efc29b846b21c63af71ed0b59ba6ec7f4d13a6762
vagrant@vagrant-ubuntu-trusty-64:~$ <mark>sudo docker ps </mark>
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
c423ae398af2        registry:2          "/entrypoint.sh /e..."   5 seconds ago       Up 4 seconds        0.0.0.0:5000->5000/tcp   registry
vagrant@vagrant-ubuntu-trusty-64:~$ 

</code>
</pre>
</div>

Now pull an ubuntu image (just an example) from dockerhub and push it to the local registry: 

<div class="highlighter-rouge">
<pre class="highlight" style="white-space: pre-wrap;">
<code>
vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$ sudo -s
root@vagrant-ubuntu-trusty-64:~# 

root@vagrant-ubuntu-trusty-64:~#<mark> docker pull ubuntu && docker tag ubuntu localhost:5000/ubuntu </mark>
Using default tag: latest
latest: Pulling from library/ubuntu
d54efb8db41d: Pull complete 
f8b845f45a87: Pull complete 
e8db7bf7c39f: Pull complete 
9654c40e9079: Pull complete 
6d9ef359eaaa: Pull complete 
Digest: sha256:dd7808d8792c9841d0b460122f1acf0a2dd1f56404f8d1e56298048885e45535
Status: Downloaded newer image for ubuntu:latest
root@vagrant-ubuntu-trusty-64:~# 
root@vagrant-ubuntu-trusty-64:~# 
root@vagrant-ubuntu-trusty-64:~# 
root@vagrant-ubuntu-trusty-64:~# <mark> docker push localhost:5000/ubuntu </mark>
The push refers to a repository [localhost:5000/ubuntu]
56827159aa8b: Layer already exists 
440e02c3dcde: Layer already exists 
29660d0e5bb2: Layer already exists 
85782553e37a: Layer already exists 
745f5be9952c: Layer already exists 
latest: digest: sha256:6b079ae764a6affcb632231349d4a5e1b084bece8c46883c099863ee2aeb5cf8 size: 1357
root@vagrant-ubuntu-trusty-64:~# 
root@vagrant-ubuntu-trusty-64:~#<mark> docker images </mark>
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
registry                2                   047218491f8c        5 weeks ago         33.2 MB
ubuntu                  latest              0ef2e08ed3fa        5 weeks ago         130 MB
localhost:5000/ubuntu   latest              0ef2e08ed3fa        5 weeks ago         130 MB
root@vagrant-ubuntu-trusty-64:~# 
</code>
</pre>
</div>


### Vagrant Setup

All we have to do get out docker daemon on the router working with the self-signed docker registry is to make sure the certificate is available in the right directory: /etc/docker/certs.d/ in the XR shell.

Hop over to the router and create folder with name = "&lt;Common Name of the certificate&gt;:5000" in the folder `/etc/docker/certs.d/` as shown below:


Hop into the router shell from your host/laptop:
  
```
AKSHSHAR-M-K0DS:docker-app-topo-bootstrap akshshar$ vagrant ssh rtr
Last login: Sun Apr  2 13:45:29 2017 from 10.0.2.2
xr-vm_node0_RP0_CPU0:~$ 
xr-vm_node0_RP0_CPU0:~$ 
xr-vm_node0_RP0_CPU0:~$ 
xr-vm_node0_RP0_CPU0:~$ sudo -i
[xr-vm_node0_RP0_CPU0:~]$ 
[xr-vm_node0_RP0_CPU0:~]$ 

```

Create a folder named `devbox.com:5000` under `/etc/docker/certs.d`.   

The folder name = `&lt;Common Name of the certificate&gt;:&lt;Port opened by the registry&gt;`

```

[xr-vm_node0_RP0_CPU0:~]$ mkdir /etc/docker/certs.d/devbox.com:5000
[xr-vm_node0_RP0_CPU0:~]$ 
[xr-vm_node0_RP0_CPU0:~]$ 

```

Add the dns entry for devbox.com in /etc/hosts of the vrf you're working in. Since before 6.3.1, we only support global-vrf in the linux kernel, we set up `/etc/hosts` of the global-vrf network namespace to create a pointer to `devbox.com`. To do this change into the correct network namespace (global-vrf) and edit /etc/hosts as shown below: 

Another way to do this would be to edit  `/etc/netns/global-vrf/hosts` file and then change into the network namespace for the subsequent scp to immediately work.
{: .notice--info}  


<div class="highlighter-rouge">
<pre class="highlight" style="white-space: pre-wrap;">
<code>

[xr-vm_node0_RP0_CPU0:~]$ 
[xr-vm_node0_RP0_CPU0:~]$ ip netns exec global-vrf bash
[xr-vm_node0_RP0_CPU0:~]$
[xr-vm_node0_RP0_CPU0:~]$ cat /etc/hosts 
127.0.0.1	localhost.localdomain		localhost
<mark>11.1.1.20       devbox.com </mark>
[xr-vm_node0_RP0_CPU0:~]$ 

</code>
</pre>
</div>

Here, 11.1.1.20 is the IP address of the directly connected interface of the devbox on the port Gi0/0/0/0 of the IOS-XR instance.
{: .notice--info}  

<div class="highlighter-rouge">
<pre class="highlight" style="white-space: pre-wrap;">
<code>

[xr-vm_node0_RP0_CPU0:~]$ 
[xr-vm_node0_RP0_CPU0:~]$ scp vagrant@devbox.com:~/certs/domain.crt /etc/docker/certs.d/devbox.com\:5000/ca.crt
vagrant@devbox.com's password: 
domain.crt                                                                                                                                                              100% 1976     1.9KB/s   00:00    
[xr-vm_node0_RP0_CPU0:~]$ 

</code>
</pre>
</div>


Perfect. Now wait about 5-10 seconds as the certificate gets automatically sync-ed to the underlying host layer (remember, the docker daemon is running on the host).  
{: .notice--info}   


Pull the docker image from the registry:  


<div class="highlighter-rouge">
<pre class="highlight" style="white-space: pre-wrap;">
<code>
[xr-vm_node0_RP0_CPU0:~]$ <mark>docker pull devbox.com:5000/ubuntu</mark>
Using default tag: latest
latest: Pulling from ubuntu
fec6b243e075: Pull complete 
190e0e9a3e79: Pull complete 
0d79cf192e4c: Pull complete 
38398c307b51: Pull complete 
356665655a72: Pull complete 
Digest: sha256:6b079ae764a6affcb632231349d4a5e1b084bece8c46883c099863ee2aeb5cf8
Status: Downloaded newer image for devbox.com:5000/ubuntu:latest
[xr-vm_node0_RP0_CPU0:~]$ 
[xr-vm_node0_RP0_CPU0:~]$ 

[xr-vm_node0_RP0_CPU0:~]$<mark> docker images </mark>
REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
devbox.com:5000/ubuntu   latest              0ef2e08ed3fa        4 weeks ago         130 MB
[xr-vm_node0_RP0_CPU0:~]$ 
[xr-vm_node0_RP0_CPU0:~]$ 

</code>
</pre>
</div>

Spin it up! :  

<div class="highlighter-rouge">
<pre class="highlight" style="white-space: pre-wrap;">
<code>
[xr-vm_node0_RP0_CPU0:~]$
[xr-vm_node0_RP0_CPU0:~]$<mark> docker run -itd --name ubuntu -v /var/run/netns/global-vrf:/var/run/netns/global-vrf --cap-add=SYS_ADMIN devbox.com:5000/ubuntu bash</mark>
b50424bbe195fd4b79c0d375dcc081228395da467d1c0d5367897180c421b41d
[xr-vm_node0_RP0_CPU0:~]$ 
[xr-vm_node0_RP0_CPU0:~]$ 
[xr-vm_node0_RP0_CPU0:~]$ docker ps
CONTAINER ID        IMAGE                    COMMAND             CREATED             STATUS              PORTS               NAMES
b50424bbe195        devbox.com:5000/ubuntu   "bash"              4 seconds ago       Up 3 seconds                            ubuntu
[xr-vm_node0_RP0_CPU0:~]$ 

</code>
</pre>
</div>


### NCS5500 and ASR9k Setup

The setup of the self-signed registry is already covered above in the [Setting up a Self-Signed Docker Registry](https://xrdocs.github.io/application-hosting/tutorials/2017-02-26-running-docker-containers-on-ios-xr-6-1-2/#setting-up-a-self-signed-docker-registry) section.  

The steps for NCS5500 and ASR9k are identical from hereon and match what we did for the Vagrant setup. To be thorough, here are the steps on an NCS5500 setup:  

Hop over to the router and issue the "bash" CLI.

Now change into the network namespace (explicitly) and set up `/etc/hosts` (In my setup, the devbox is reachable over the management port on IP=11.11.11.2) :    

```
[ncs5508:~]$ip netns exec global-vrf bash 
[ncs5508:~]$cat /etc/hosts
127.0.0.1	localhost.localdomain		localhost

127.0.1.1    ncs5508.cisco.com    ncs5508

11.11.11.2 devbox.com
[ncs5508:~]$
[ncs5508:~]$
[ncs5508:~]$

```

Set up the directory to store the certificates created for the docker registry: 

```
[ncs5508:~]$
[ncs5508:~]$mkdir /etc/docker/certs.d/devbox.com:5000
[ncs5508:~]$
[ncs5508:~]$

```

scp over the self-signed certificate from the devbox into the above directory:  

<div class="highlighter-rouge">
<pre class="highlight" style="white-space: pre-wrap;">
<code>
[ncs5508:~]$scp cisco@devbox.com:~/certs/domain.crt /etc/docker/certs.d/devbox.com\:5000/ca.crt
Warning: Permanently added 'devbox.com,11.11.11.2' (ECDSA) to the list of known hosts.
cisco@devbox.com's password: 
domain.crt                                    100% 1976     1.9KB/s   00:00    
[ncs5508:~]$

</code>
</pre>
</div>


Now pull the docker image from the registry and spin it up:  


<div class="highlighter-rouge">
<pre class="highlight" style="white-space: pre-wrap;">
<code>
[ncs5508:~]$<mark>docker pull devbox.com:5000/ubuntu</mark>
Using default tag: latest
latest: Pulling from ubuntu

d54efb8db41d: Pull complete 
f8b845f45a87: Pull complete 
e8db7bf7c39f: Pull complete 
9654c40e9079: Pull complete 
6d9ef359eaaa: Pull complete 
Digest: sha256:dd7808d8792c9841d0b460122f1acf0a2dd1f56404f8d1e56298048885e45535
Status: Downloaded newer image for devbox.com:5000/ubuntu:latest
[ncs5508:~]$
[ncs5508:~]$ <mark> docker run -itd --name ubuntu -v /var/run/netns/global-vrf:/var/run/netns/global-vrf --cap-add=SYS_ADMIN devbox.com:5000/ubuntu bash</mark>
3b4721fa053a97325ccaa2ac98b3dc3fd9fb224543e0ed699be597f773ab875d
[ncs5508:~]$
[ncs5508:~]$
[ncs5508:~]$docker ps
CONTAINER ID        IMAGE                    COMMAND             CREATED             STATUS              PORTS               NAMES
3b4721fa053a        devbox.com:5000/ubuntu   "bash"              5 seconds ago       Up 4 seconds                            ubuntu
[ncs5508:~]$

</code>
</pre>
</div>


## Using a container/image tar ball

This is the potentially the easiest secure technique if you don't want to meddle around with certificates on a docker registry and potentially don't want a registry at all.

### Create a docker image tarball

As a first step, on your devbox create a docker image tar ball. This can be done by first pulling the relevant docker image into your devbox (From dockerhub) or building it on your own on the devbox (we will not delve into this here), and then issuing a `docker save` to save the image into a loadable tar-ball.  

This is shown below. We assume you know how to get images into the devbox environment already:  

<div class="highlighter-rouge">
<pre class="highlight" style="white-space: pre-wrap;">
<code>

vagrant@vagrant-ubuntu-trusty-64:~$ sudo docker images 
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
registry                2                   047218491f8c        5 weeks ago         33.2 MB
localhost:5000/ubuntu   latest              0ef2e08ed3fa        5 weeks ago         130 MB
ubuntu                  latest              0ef2e08ed3fa        5 weeks ago         130 MB
vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$ <mark>sudo docker save ubuntu > ubuntu.tar </mark>
vagrant@vagrant-ubuntu-trusty-64:~$ 

</code>
</pre>
</div>


### Vagrant Setup

Login to your Router (directly into the shell or by issuing the `bash` command in XR CLI). We first scp the docker image tar ball into an available volume on the router and then load it up.  


<div class="highlighter-rouge">
<pre class="highlight" style="white-space: pre-wrap;">
<code>
[xr-vm_node0_RP0_CPU0:~]$ 
[xr-vm_node0_RP0_CPU0:~]$ 
[xr-vm_node0_RP0_CPU0:~]$ df -h  /misc/app_host/
Filesystem                       Size  Used Avail Use% Mounted on
/dev/mapper/app_vol_grp-app_lv0  3.9G  260M  3.5G   7% /misc/app_host
[xr-vm_node0_RP0_CPU0:~]$ <mark>scp vagrant@11.1.1.20:~/ubuntu.tar /misc/app_host/</mark>
vagrant@11.1.1.20's password: 
ubuntu.tar                                                                                                                                                             100%  129MB 107.7KB/s   20:31    
[xr-vm_node0_RP0_CPU0:~]$ 
[xr-vm_node0_RP0_CPU0:~]$ 
[xr-vm_node0_RP0_CPU0:~]$ <mark> docker load &lt; /misc/app_host/ubuntu.tar </mark>
[xr-vm_node0_RP0_CPU0:~]$ 
[xr-vm_node0_RP0_CPU0:~]$ docker images
REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
ubuntu                   latest              0ef2e08ed3fa        4 weeks ago         130 MB
[xr-vm_node0_RP0_CPU0:~]$ 

</code>
</pre>
</div>

Now go ahead and spin it up as shown earlier:  


<div class="highlighter-rouge">
<pre class="highlight" style="white-space: pre-wrap;">
<code>
[xr-vm_node0_RP0_CPU0:~]$
[xr-vm_node0_RP0_CPU0:~]$<mark> docker run -itd --name ubuntu -v /var/run/netns/global-vrf:/var/run/netns/global-vrf --cap-add=SYS_ADMIN ubuntu bash</mark>
b50424bbe195fd4b79c0d375dcc081228395da467d1c0d5367897180c421b41d
[xr-vm_node0_RP0_CPU0:~]$ 
[xr-vm_node0_RP0_CPU0:~]$ 
[xr-vm_node0_RP0_CPU0:~]$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
108a5ad711ca        ubuntu              "bash"              3 seconds ago       Up 2 seconds                            ubuntu
[xr-vm_node0_RP0_CPU0:~]$ 
</code>
</pre>
</div>



### NCS5500 and ASR9k setup.

NCS5500 and ASR9k follow the exact same steps as the Vagrant box above. For completeness, though:

<div class="highlighter-rouge">
<pre class="highlight" style="white-space: pre-wrap;">
<code>
[ncs5508:~]$
[ncs5508:~]$scp cisco@11.11.11.2:~/ubuntu.tar /misc/app_host/
cisco@11.11.11.2's password: 
ubuntu.tar                                    100%  317MB  10.2MB/s   00:31    
[ncs5508:~]$
[ncs5508:~]$docker load &lt; /misc/app_host/ubuntu.tar 
[ncs5508:~]$
[ncs5508:~]$
[ncs5508:~]$
[ncs5508:~]$docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
[ncs5508:~]$
[ncs5508:~]$<mark> docker run -itd --name ubuntu -v /var/run/netns/global-vrf:/var/run/netns/global-vrf --cap-add=SYS_ADMIN ubuntu bash</mark>
ffc95e05e05c6e2e6b8e4aa05b299f513fd5df6d1ca8fe641cfa7f44671e6f07
[ncs5508:~]$
[ncs5508:~]$
[ncs5508:~]$docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED              STATUS              PORTS               NAMES
ffc95e05e05c        ubuntu              "bash"              About a minute ago   Up About a minute                       ubuntu
[ncs5508:~]$
</code>
</pre>
</div>



**And there you have it! We've successfully tried all the possible techniques through which a docker image can be pulled into the router before we spin up the container.** 
{: .notice--success}


## Network access inside Docker container

As a user you might be wondering:  What can processes inside the spun-up Docker container really do?
The answer: everything.
You have a distribution of your choice with complete access to XR RIB/FIB (through routes in the kernel) and interfaces (data and management) to bind to.  

**Docker images by default are extremely basic and do not include most utilities. To be able to showcase the kind of access that a container has, I pull in a special ubuntu docker image with pre-installed iproute2**  
{: .notice--warning}  

Assuming you've selected one of the techniques above to spin up the docker container, I'm going to  let's exec into the container using `docker exec`:  

We're executing the steps on an NCS5500. The steps are identical for ASR9k, NCS5500/NCS5000 and Vagrant setups.  
{: .notice--info}  





 




