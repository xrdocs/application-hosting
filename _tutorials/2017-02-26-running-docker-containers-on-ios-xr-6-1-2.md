---
published: true
date: '2017-02-26 14:40 -0800'
title: Running Docker Containers on IOS-XR (6.1.2+)
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
  
 
*  **Private "insecure" registry**: Some users may choose to do this, specially if they're running a local docker registry inside a secured part of their network.  
   
*  **Private "self-signed" registry**: This is more secure than the "insecure" setup, and allows a user to enable TLS.


*  **Private "secure" registry**: Set up reachability to your private registry, created using a certificate obtained from a CA. We won't really tackle this scenario separately in this tutorial due to the absence of said certificate :).
    
*  **Tarball image/container**:  This is the simplest setup - very similar to LXC deployments. In this case, a user may create and set up a container completely off-box, package it up as an image or a container tar ball, transfer it to the router and then load/import it, before running.  


For each case, we will compare IOS-XR running as a Vagrant box with IOS-XR running on a physical box (NCS5500). They should be identical, except for reachability through the Management ports.  
  
  
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


### Physical (NCS5500 router)  

On the other hand, if you have an NCS5500 lying around (don't we all?), then load up a 6.1.2+ image on the router and connect an Ubuntu server (for the purpose of this tutorial), to the Management network of the router.

The server needs to be reachable from the router over the Management network.

Further, we're going to enable SSH access in XR CLI and in  XR linux shell to achieve an equivalence between the NCS5508 and Vagrant setup.  

**Enable SSH access in the XR CLI**
{: .notice}

On my NCS5508 setup, I can enable SSH in XR in the default (global) vrf with the following steps and CLI:  

```shell

RP/0/RP0/CPU0:ncs5508#crypto key generate rsa
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


```

**Enable SSH access to XR linux shell**
{: .notice}

This is openssh running in the XR linux environment. Users may choose to keep this disabled based on the kind of operations they intend to have. Enabling it in a given network namespace (equivalent to XR vrf) opens up port 57722 on all the IP addresses reachable in that VRF.

In 6.1.2, only global-vrf (default vrf) is supported in the linux environment for SSH and apps. Post 6.2.11, support for Mgmt vrfs in the linux shell will be brought in.
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

The topology I'm using differs slightly between the vagrant setup and the NCS5508 setup.
This is owing to the fact that the Management port of the vagrant IOS-XR box is used up in the NAT network. So to show equivalence between the two setups, I directly connect the Gig0/0/0/0 interface of Vagrant ios-xrv64 with eth1 of the devbox as shown in the figure below.  

The two topologies in use are:  


![vagrant docker topo](https://xrdocs.github.io/xrdocs-images/assets/images/vagrant_docker_topo.png)  
{: .notice}
  
    
![NCS5500 docker topo](https://xrdocs.github.io/xrdocs-images/assets/images/ncs5500_docker_topo.png)    
{: .notice}  
  
  
  
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


### NCS5508 setup

In this case, the devbox must be provisioned by the user. On an ubuntu devbox, docker-engine can be installed by following the instructions at:  
  
><https://docs.docker.com/engine/installation/linux/ubuntu/>  
  
    

Perfect! Now we're all set with the topology and SSH access. Before we begin, let's understand the docker daemon/client setup inside IOS-XR.
{: .notice--success}  
  
  
## Docker Daemon support on IOS-XR

If you haven't already gone through the basic overview on the application hosting infrastructure on XR, I would urge you to have a quick read:  
  
><{{ base_path }}/blogs/2016-06-28-xr-app-hosting-architecture-quick-look/>  
  

From the above article it becomes fairly clear that internally the IOS-XR architecture involves a Host layer running the libvirtd daemon and IOS-XR runs as an LXC spawned using the daemon.  

Further, the "virsh" client is provided within the XR LXC, so that a user may have client level access to the daemon while sitting inside the XR LXC itself.  

The setup for launching LXCs in IOS-XR is shown below:  

[![xr-lxc](https://xrdocs.github.io/xrdocs-images/assets/images/xr_lxc.png)](https://xrdocs.github.io/xrdocs-images/assets/images/xr_lxc.png)  
  
  
  
The Docker client/daemon setup follows the exact same principle as shown below. Docker Daemon runs on the host and Docker client is made available inside the XR LXC for easy operationalization:  
  
[![xr-docker](https://xrdocs.github.io/xrdocs-images/assets/images/xr_docker.png)](https://xrdocs.github.io/xrdocs-images/assets/images/xr_docker.png)  

Can we verify this?

### Vagrant setup Docker Client Access: 
On your vagrant box, there are two ways to get access to the docker client:

*  **Drop into the "bash" shell from XR CLI:** Using "bash" ensures that the correct environment 
   variables are sourced to gain access to the Docker Daemon on the host:   

   **Password for the XR CLI:   vagrant**
   {: .notice--info}  
   
  
    <div class="highlighter-rouge">
    <pre class="highlight">
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
    <pre class="highlight">
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
    
    

### NCS5508 Setup Docker Client Access.

If you followed the steps in the pre-requisites section above : [Pre-requisites](https://xrdocs.github.io/application-hosting/tutorials/2017-02-26-running-docker-containers-on-ios-xr-6-1-2/#physical-ncs5500-router), you would already have access to your NCS5500 device over XR SSH (CLI, port 22) as well as sshd_operns (XR linux shell, port 57722)


Following the Vagrant model, over XR SSH, we use the "bash" CLI to access the docker client on the NCS5500:

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

### Private "insecure" registry

This is the simplest technique:

*  We spin up an insecure docker registry(which is itself a docker container pulled down from dockerhub) on our devbox.

*  We then modify /etc/sysconfig/docker in XR linux to add the insecure registry information 

*  Set up the route to the registry

*  Populate the registry with some docker images from dockerhub

*  Pull the relevant images from the insecure registry down to XR's docker daemon and spin up containers



**Spin up the insecure Docker registry on Devbox**

We follow the steps described here: <https://docs.docker.com/registry/deploying/>


<div class="highlighter-rouge">
<pre class="highlight">
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

Now let's setup XR's docker daemon to accept the insecure registry located on the directly connected network on Gig0/0/0/0.   


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
# DOCKER_OPTS="lt&;spacegt&;--insecure-registrylt&;spacegt&;foo"
# DOCKER_OPTS+="lt&;spacegt&;--insecure-registrylt&;spacegt&;bar"

<mark>DOCKER_OPTS=" --insecure-registry 11.1.1.20:5000"</mark>
[xr-vm_node0_RP0_CPU0:~]$
</code>
</pre>
</div> 

As the instructions/comments inside the file indicate, make sure there is a space before --insecure-registry flag. Further, in a normal docker daemon setup, a user is supposed to restart the docker daemon when changes to /etc/sysconfig/docker are made. In case of XR, this is not needed. We handle automatic restarts of the docker daemon when a user makes changes to /etc/sysconfig/docker and saves it.
{: .notice--info}


Now issue the docker run command to launch the container on XR.

```shell
RP/0/RP0/CPU0:ios#
RP/0/RP0/CPU0:ios#bash
Mon Mar  6 05:51:14.341 UTC
[xr-vm_node0_RP0_CPU0:~]$
[xr-vm_node0_RP0_CPU0:~]$docker run -itd --name ubuntu -v /var/run/netns --privileged 11.1.1.20:5000/ubuntu bash
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

```
You will notice two peculiar things in the command we run:

*  **Mounting of /var/run/netns**: We mount /var/run/netns into the docker container. This is an option we use to mount all the potential network namespaces that may be created to match the XR vrfs. These network namespaces (XR release 6.2.11+) are created on the host and then bind-mounted into the XR LXC for user convenience. The docker container, running on the host, will simply inherit these network namespaces through the /var/run/netns mount. **Bear in mind that before 6.2.11 release only the `global-vrf` is supported in the XR linux shell**.

*  **--privileged flag**: We're using the `--privileged` flag because even when network namespaces are mounted from the "host" into the docker container, a user can change into a particular network namespace or execute commands in a particular namespace, only if the container is launched with privileged capabilties.

Finally, let's








