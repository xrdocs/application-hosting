---
published: true
date: '2016-06-16 03:17 -0500'
title: 'XR toolbox, Part 4: Bring your own Container (LXC) App'
author: Akshat Sharma
tags:
  - vagrant
  - iosxr
  - cisco
  - linux
  - lxc
  - containers
position: hidden
excerpt: Launch a Container app (LXC) on IOS-XR
---

{% include toc icon="table" title="Launching a Container App" %}
{% include base_path %}

## Introduction

If you haven't checked out the earlier parts to the XR toolbox Series, then you can do so here:  

>
[XR Toolbox Series]({{ base_path }}/tags/#xr-toolbox)

  
The purpose of this series is simple. Get users started with an IOS-XR setup on their laptop and incrementally enable them to try out the application-hosting infrastructure on IOS-XR.

In this part, we explore how a user can build and deploy their own container (LXC) based applications on IOS-XR.


## Pre-requisites

Before we begin, let's make sure you've set up your development environment.
If you haven't checked it out, go through the "App-Development Topology" tutorial here:  

>
[XR Toolbox, Part 3: App Development Topology]({{ base_path }}/tutorials/2016-06-06-xr-toolbox-app-development-topology)  
  
Follow the instructions to get your topology up and running as shown below:  

![app dev topo](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/app_dev_topology.png)


If you've reached the end of the above tutorial, you should be able to issue a `vagrant status` in the `vagrant-xr/lxc-app-topo-bootstrap` directory to see a rtr (IOS-XR) and a devbox (Ubuntu/trusty) instance running.  

<div class="highlighter-rouge">
<pre class="highlight">
<code>
AKSHSHAR-M-K0DS:lxc-app-topo-bootstrap akshshar$ pwd
<mark>/Users/akshshar/vagrant-xr/lxc-app-topo-bootstrap </mark>
AKSHSHAR-M-K0DS:lxc-app-topo-bootstrap akshshar$ 
AKSHSHAR-M-K0DS:lxc-app-topo-bootstrap akshshar$<mark> vagrant status </mark>
Current machine states:

rtr                       running (virtualbox)
devbox                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
AKSHSHAR-M-K0DS:lxc-app-topo-bootstrap akshshar$ 

</code>
</pre>
</div>


All good? Perfect. Let's start building our container application tar ball.
{: .notice--success}  


## Create a container App  

{% capture notice-text %}
#### Using a custom rootfs tar ball  

The technique presented here focuses on the creation of a container from scratch (using a base ubuntu template) followed by the installation of an application for first-time users.
A user can easily use their own pre-built rootfs tar ball and ignore this section altogether.
    
The only point to remember is that if you expect to use SSH access into the container after deployment to XR, then change the default SSH port in /etc/ssh/sshd_config in your rootfs to something other than 22/57722 (or any other port you expect XR to use based on your config).  

This is showcased in the following section below:  

[Change SSH port inside your container]({{ base_path }}/tutorials/2016-06-16-xr-toolbox-part-4-bring-your-own-container-lxc-app/#change-ssh-port-inside-your-container)  

{% endcapture %}

<div class="notice--warning">
  {{ notice-text | markdownify }}
</div>

  
To launch an LXC container we need two things:  

*  A container rootfs tar ball
*  An XML file to launch the container using `virsh/libvirt`

To create them, we'll hop onto our devbox (Ubuntu/trusty) VM in the topology and install lxc-tools. lxc-tools will be used to create a container rootfs tar ball.  


### Install lxc tools on devbox  

SSH into the devbox:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
AKSHSHAR-M-K0DS:lxc-app-topo-bootstrap akshshar$<mark> vagrant ssh devbox</mark>
Welcome to Ubuntu 14.04.4 LTS (GNU/Linux 3.13.0-87-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

  System information as of Thu Jun 16 14:27:47 UTC 2016

  System load:  0.0               Processes:           74
  Usage of /:   3.5% of 39.34GB   Users logged in:     0
  Memory usage: 25%               IP address for eth0: 10.0.2.15
  Swap usage:   0%                IP address for eth1: 11.1.1.20

  Graph this data and manage this system at:
    https://landscape.canonical.com/

  Get cloud support with Ubuntu Advantage Cloud Guest:
    http://www.ubuntu.com/business/services/cloud

0 packages can be updated.
0 updates are security updates.


Last login: Thu Jun 16 14:27:47 2016 from 10.0.2.2
vagrant@vagrant-ubuntu-trusty-64:~$ 
</code>
</pre>
</div> 

  
Install lxc tools inside the devbox

```shell
sudo apt-get update
sudo apt-get -y install lxc
```  

Check that lxc was properly installed:

```shell
vagrant@vagrant-ubuntu-trusty-64:~$ sudo lxc-start --version
1.0.8
vagrant@vagrant-ubuntu-trusty-64:~$ 
```

### Launch an LXC container on the devbox

Using the standard ubuntu template available with lxc, let's create and start the ubuntu container inside devbox:  

<div class="highlighter-rouge">
<pre class="highlight">
<code>
vagrant@vagrant-ubuntu-trusty-64:~$<mark> sudo lxc-create -t ubuntu --name xr-lxc-app</mark>
Checking cache download in /var/cache/lxc/trusty/rootfs-amd64 ... 
Installing packages in template: ssh,vim,language-pack-en
Downloading ubuntu trusty minimal ...
I: Retrieving Release 
I: Retrieving Release.gpg 
I: Checking Release signature
I: Valid Release signature (key id 790BC7277767219C42C86F933B4FE6ACC0B21F32)
I: Retrieving Packages 

------------------------------ snip output ------------------------------------
</code>
</pre>
</div> 


This process will take some time as the ubuntu rootfs template is downloaded for you by the lxc tools. 
{: .notice--info}  


Once the container template is installed successfully, it should show up in the lxc-ls output:


<div class="highlighter-rouge">
<pre class="highlight">
<code>
vagrant@vagrant-ubuntu-trusty-64:~$<mark> sudo lxc-ls --fancy </mark>
NAME        STATE    IPV4  IPV6  AUTOSTART  
------------------------------------------
xr-lxc-app  STOPPED  -     -     NO         
vagrant@vagrant-ubuntu-trusty-64:~$ 
</code>
</pre>
</div> 


Now let's start the container:


<div class="highlighter-rouge">
<pre class="highlight">
<code>
vagrant@vagrant-ubuntu-trusty-64:~$<mark> sudo lxc-start --name xr-lxc-app </mark>
&lt;4&gt;init: plymouth-upstart-bridge main process (5) terminated with status 1
&lt;4&gt;init: plymouth-upstart-bridge main process ended, respawning
&lt;4&gt;>init: hwclock main process (7) terminated with status 77
&lt;4&gt;>init: plymouth-upstart-bridge main process (15) terminated with status 1
&lt;4&gt;>init: plymouth-upstart-bridge main process ended, respawning

------------------------------ snip output ------------------------------------

</code>
</pre>
</div> 

You will taken to the login prompt.

The Default credentials are:  
Username:  **ubuntu**  
Password:  **ubuntu**
{: .notice--info}  


```shell
Ubuntu 14.04.4 LTS xr-lxc-app console

xr-lxc-app login: <4>init: setvtrgb main process (428) terminated with status 1
<4>init: plymouth-upstart-bridge main process (23) killed by TERM signal


Ubuntu 14.04.4 LTS xr-lxc-app console

xr-lxc-app login: ubuntu
Password: 
Welcome to Ubuntu 14.04.4 LTS (GNU/Linux 3.13.0-87-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

ubuntu@xr-lxc-app:~$ 


```

Perfect! You've launched an ubuntu container on your devbox.
{: .notice--success}


### Create/Install your app

In this example we'll install iperf as a sample application.You may choose to skip this step if you have another app in mind.

sudo password:  **ubuntu**
{: .notice--info}   


```shell

ubuntu@xr-lxc-app:~$ sudo apt-get -y install iperf
[sudo] password for ubuntu: 
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following NEW packages will be installed:
  iperf
0 upgraded, 1 newly installed, 0 to remove and 0 not upgraded.
Need to get 56.3 kB of archives.
After this operation, 174 kB of additional disk space will be used.
Get:1 http://archive.ubuntu.com/ubuntu/ trusty/universe iperf amd64 2.0.5-3 [56.3 kB]
Fetched 56.3 kB in 2s (23.5 kB/s)
Selecting previously unselected package iperf.
(Reading database ... 14629 files and directories currently installed.)
Preparing to unpack .../iperf_2.0.5-3_amd64.deb ...
Unpacking iperf (2.0.5-3) ...
Setting up iperf (2.0.5-3) ...
ubuntu@xr-lxc-app:~$ 
ubuntu@xr-lxc-app:~$ 
ubuntu@xr-lxc-app:~$ 
ubuntu@xr-lxc-app:~$ iperf -v
iperf version 2.0.5 (08 Jul 2010) pthreads
ubuntu@xr-lxc-app:~$ 

```  


### Change SSH port inside your container

When we deploy the container to IOS-XR, we will share XR's network namespace. Since IOS-XR already uses up port 22 and port 57722 for its own purposes, we need to pick some other port for our container.  

**Our recommendation? - Pick some port in the 58xxx range.**
{: .notice--info}  

Let's change the SSH port to 58822:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
ubuntu@xr-lxc-app:~$<mark> sudo sed -i s/Port\ 22/Port\ 58822/ /etc/ssh/sshd_config </mark>
ubuntu@xr-lxc-app:~$ 
</code>
</pre>
</div>  

Check that your port was updated successfully:

```shell
ubuntu@xr-lxc-app:~$ cat /etc/ssh/sshd_config | grep Port
Port 58822
ubuntu@xr-lxc-app:~$ 
```

We're good!
{: .notice--success}


### Shutdown and package your container  

  
  
Issue a shutdown to escape 

```shell
ubuntu@xr-lxc-app:~$ sudo shutdown -h now
ubuntu@xr-lxc-app:~$ 
Broadcast message from ubuntu@xr-lxc-app
	(/dev/lxc/console) at 19:37 ...

The system is going down for halt NOW!

------------------------------ snip output ------------------------------------

mount: cannot mount block device /dev/sda1 read-only
 * Will now halt
vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$ 
```  

We're back on our devbox.
Now hop over to the directory `/var/lib/lxc/xr-lxc-app` and package the rootfs into a tar ball.
In the end we transfer the tar ball to the home directory (~/ or /home/vagrant)

**You will need to be root for this operation**
{: .notice--warning}


<div class="highlighter-rouge">
<pre class="highlight">
<code>
vagrant@vagrant-ubuntu-trusty-64:~$<mark> sudo -s </mark>
root@vagrant-ubuntu-trusty-64:~# 
root@vagrant-ubuntu-trusty-64:~#<mark> whoami </mark>
root
root@vagrant-ubuntu-trusty-64:~#<mark> cd /var/lib/lxc/xr-lxc-app/ </mark>
root@vagrant-ubuntu-trusty-64:/var/lib/lxc/xr-lxc-app# ls
config  fstab  rootfs
root@vagrant-ubuntu-trusty-64:/var/lib/lxc/xr-lxc-app# <mark>cd rootfs </mark>
root@vagrant-ubuntu-trusty-64:/var/lib/lxc/xr-lxc-app/rootfs# 
root@vagrant-ubuntu-trusty-64:/var/lib/lxc/xr-lxc-app/rootfs#<mark> tar -czf xr-lxc-app-rootfs.tar.gz * </mark>
tar: dev/log: socket ignored
root@vagrant-ubuntu-trusty-64:/var/lib/lxc/xr-lxc-app/rootfs#
root@vagrant-ubuntu-trusty-64:/var/lib/lxc/xr-lxc-app/rootfs#<mark>mv xr-lxc-app-rootfs.tar.gz /home/vagrant</mark>
root@vagrant-ubuntu-trusty-64:/var/lib/lxc/xr-lxc-app/rootfs#<mark>ls -l /home/vagrant</mark>
total 119984
-rw-r--r-- 1 root root 122863332 Jun 16 19:41 xr-lxc-app-rootfs.tar.gz
</code>
</pre>
</div>  



## Create LXC SPEC XML File

We need to create an XML file that will define different parameters (cpu, mem, rootfs location etc.) for the container launch on IOS-XR (which uses libvirt).
On the devbox, use your favorite editor (vi, nano, pico etc.) to create a new file called   
`xr-lxc-app.xml` under `/home/vagrant` of the devbox with the following content:  

```html
<domain type='lxc' xmlns:lxc='http://libvirt.org/schemas/domain/lxc/1.0' >
<name>xr-lxc-app</name>
<memory>327680</memory>
<os>
<type>exe</type>
<init>/sbin/init</init>
</os>
<lxc:namespace>
<sharenet type='netns' value='global-vrf'/>
</lxc:namespace>
<vcpu>1</vcpu>
<clock offset='utc'/>
<on_poweroff>destroy</on_poweroff>
<on_reboot>restart</on_reboot>
<on_crash>destroy</on_crash>
<devices>
<emulator>/usr/lib64/libvirt/libvirt_lxc</emulator>
<filesystem type='mount'>
<source dir='/misc/app_host/xr-lxc-app/'/>
<target dir='/'/>
</filesystem>
<console type='pty'/>
</devices>
</domain>

```

>
A couple of configuration knobs seem interesting in the above XML file:  
>
*  The netns (network namespace) setting:  
>
   ```html
   `<sharenet type='netns' value='global-vrf'/>`;
   ```
>
   In IOS-XR the **'global-vrf' network namespace houses all the XR Gig/Mgmt interfaces that are 
   in the global/default VRF.** The sharenet setting above makes sure that the container on launch 
   will also have access to all of XR's interfaces natively 
>
*  The rootfs mount volume:  
>
   ```html
   `<source dir='/misc/app_host/xr-lxc-app/'/>`;
   ```
>
   **/misc/app_host/ in IOS-XR is a special mount volume** that is designed to provide nearly 3.9G 
   of Disk space on IOS-XRv and varying amounts on other platforms (NCS5508, ASR9k) etc. This 
   mount volume may be used to host custom container rootfs and other large files without using up 
   XR's disk space. **In this case we expect the rootfs to be untarred in the 
   /misc/app_host/xr-lxc-app/ directory**
{: .notice--info} 

>
Your LXC app is now ready to be deployed! You should have the following two components in the home directory of the devbox:  
>
```shell
root@vagrant-ubuntu-trusty-64:~# pwd
/home/vagrant
root@vagrant-ubuntu-trusty-64:~# ls -l
total 119988
-rw-r--r-- 1 root root 122863332 Jun 16 19:41 xr-lxc-app-rootfs.tar.gz
-rw-r--r-- 1 root root       590 Jun 16 23:29 xr-lxc-app.xml
root@vagrant-ubuntu-trusty-64:~# 
```
{: .notice--success}  


## Transfer rootfs and XML file to XR

We can either use the XR Gig or Mgmt interface to transfer the files.
IOS-XR runs openssh in the linux environment on port 57722.  

>
We need to transfer the files to the /misc/app_host volume on IOS-XR.
However, /misc/app_host is owned by root and root access over SSH is not allowed, for obvious security reasons.  
>
Hence, to enable the transfer of custom files to IOS-XR, we provide a `/misc/app_host/scratch` directory which is owned by the app_host group. Any user transferring files over SSH to this directory must be part of the app_host group to have access.The user `vagrant` is already part of the app_host group.
{: .notice--info}

**Transfer using the Gig interface:**  

```
scp -P 57722 /home/vagrant/* vagrant@11.1.1.10:/misc/app_host/scratch/
```  
Where 11.1.1.10 is the directly connected Gig0/0/0/0 interface of IOS-XR instance (this config was explained in the [XR Toolbox, Part 3: App Development Topology]({{ base_path }}/tutorials/2016-06-06-xr-toolbox-app-development-topology) tutorial).
But this process might be slow since Gig interfaces in the Vagrant IOS-XR image are rate-limited.
  
**Transfer using the Mgmt interface**
Vagrant forwards the port 57722 to some host port for IOS-XR over the management port. In Virtualbox, the IP address of the host (your laptop) is always 10.0.2.2 for the NAT'ed port.

So determine the forwarded port for port 57722 for XR on your laptop shell:

```shell
AKSHSHAR-M-K0DS:lxc-app-topo-bootstrap akshshar$ vagrant port rtr
The forwarded ports for the machine are listed below. Please note that
these values may differ from values configured in the Vagrantfile if the
provider supports automatic port collision detection and resolution.

    22 (guest) => 2223 (host)
 57722 (guest) => 2222 (host)
AKSHSHAR-M-K0DS:lxc-app-topo-bootstrap akshshar$ 

```

Now use port `2222` to transfer the files over the management port using the host IP = 10.0.2.2

```shell
vagrant@vagrant-ubuntu-trusty-64:~$ scp -P 2222 /home/vagrant/* vagrant@10.0.2.2:/misc/app_host/scratch
The authenticity of host '[10.0.2.2]:2222 ([10.0.2.2]:2222)' can't be established.
ECDSA key fingerprint is db:25:e2:27:49:2a:7b:27:e1:76:a6:7a:e4:70:f5:f7.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[10.0.2.2]:2222' (ECDSA) to the list of known hosts.
vagrant@10.0.2.2's password: 
xr-lxc-app-rootfs.tar.gz                                                                                                                                                100%  117MB  16.7MB/s   00:07    
xr-lxc-app.xml                                                                                                                                                          100%  590     0.6KB/s   00:00    
vagrant@vagrant-ubuntu-trusty-64:~$ 

```


## Untar rootfs under /misc/app_host/


## Use virsh to launch container


## Test your app!






















