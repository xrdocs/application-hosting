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

We're going to spin up a topology with 3 vagrant instances as shown below:  

![native-app-topo](https://xrdocs.github.io/xrdocs-images/assets/images/native-app-topo.png)  

*  **WRL7 Build**: Since IOS-XR uses a streamlined custom WRL7 distribution, we need to make sure we have the latest WRL7 environment available to build "native" apps. For this reason we have released the <https://atlas.hashicorp.com/ciscoxr/boxes/appdev-xr6.1.1> vagrant box to match IOS-XR release 6.1.1.  You will simply need to reference "ciscoxr/appdev-xr6.1.1" in your Vagrantfile to spin it up.  
  
*  **IOS-XR**: This is the 6.1.1 IOS-XR vagrant instance you would have already downloaded and installed as explained in the vagrant quick-start tutorial:    

   >
   [IOS-XR vagrant box download]({{ base_path }}/tutorials/iosxr-vagrant-quickstart#download-and-add-the-ios-xrv-vagrant-box)  
 
   In the end, `vagrant box list` must list your IOS-XRv vagrant box:  
   
   ```shell

   AKSHSHAR-M-K0DS:~ akshshar$ vagrant box list
   IOS-XRv (virtualbox, 0)
   AKSHSHAR-M-K0DS:~ akshshar$ 

   ```
*  **devbox**: This is the ubuntu/trusty64 image we have been using in the other tutorials for LXC creation and generic application testing. 

IOS-XR and devbox instances talk to each other over Gig0/0/0/0 and eth1 interfaces respectively.

   
### Clone the git repo
 
Clone the following git repo: <https://github.com/ios-xr/vagrant-xrdocs.git>
   
 
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
Bringing machine 'wrl7_build' up with 'virtualbox' provider...


--------------------------- snip output ----------------------------
</code>
</pre>
</div> 



## Build iperf from source on WRL7 Build Server

Assuming everything came up fine, let's ssh into the wrl7_build instance:  

```shell

AKSHSHAR-M-K0DS:native-app-topo-bootstrap akshshar$ vagrant ssh wrl7_build
localhost:~$ 
localhost:~$ 
localhost:~$ 
localhost:~$ lsb_release -a
LSB Version:	core-4.1-noarch:core-4.1-x86_64
Distributor ID:	wrlinux
Description:	Wind River Linux 7.0.0.2
Release:	7.0.0.2
Codename:	n/a
localhost:~$ 

```

### Fetch iperf source code  

Great! Let's fetch the source code of iperf (iperf2) from its official location:  

Current latest version is:  [**iperf-2.0.9**](https://iperf.fr/download/source/iperf-2.0.9-source.tar.gz)

Download this tar ball into the wrl7_build vagrant instance:  

<div class="highlighter-rouge">
<pre class="highlight">
<code> 
localhost:~$ 
localhost:~$<mark> wget https://iperf.fr/download/source/iperf-2.0.9-source.tar.gz </mark>
--2016-07-17 14:57:13--  https://iperf.fr/download/source/iperf-2.0.9-source.tar.gz
Resolving iperf.fr... 194.158.119.186, 2001:860:f70a::2
Connecting to iperf.fr|194.158.119.186|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 277702 (271K) [application/x-gzip]
Saving to: 'iperf-2.0.9-source.tar.gz'

100%[===================================================================================>] 277,702      345KB/s   in 0.8s   

2016-07-17 14:57:14 (345 KB/s) - 'iperf-2.0.9-source.tar.gz' saved [277702/277702]

localhost:~$ 
localhost:~$ 
localhost:~$ ls
<mark>iperf-2.0.9-source.tar.gz </mark>
localhost:~$
</code>
</pre>
</div> 

**Copy the source code tar ball into the expected location for rpmbuild: `/usr/src/rpm/SOURCES/`**
{: .notice--warning}


```shell
localhost:~$ sudo cp /home/vagrant/iperf-2.0.9-source.tar.gz /usr/src/rpm/SOURCES/
localhost:~$ 
```  


### Set up the SPEC file for rpmbuild  

We will need a spec file to build the RPM. The spec file we intend to use is shown below. The highlighted sections are important.   

**This file is  already available in `/home/vagrant` of wrl7_build server, thanks to the "file" provisioner that run as part of "vagrant up"**.  
{:  .notice--warning}

<div class="highlighter-rouge">
<pre class="highlight">
<code> 
 
Name: iperf 
Version: 2.0.9
Release: XR_6.1.1
License: Copyright (c) 2015 Cisco Systems Inc. All rights reserved.
Packager: cisco
SOURCE0 : %{name}-%{version}-source.tar.gz
Group: 3rd party application
Summary: iperf compiled for WRL7: XR 6.1.1

%description
This is a compiled version of iperf-2.0.9 for WRL7: XR 6.1.1

%prep

%setup -q -n %{name}-%{version}

%build
<mark>./configure
make</mark>

%install
<mark>mkdir -p %{buildroot}%{_sbindir}
install -m755 src/iperf %{buildroot}%{_sbindir}</mark>

%files

%defattr(-,root,root)
%{_sbindir}/iperf

%clean
rm -rf %{buildroot}
</code>
</pre>
</div> 


### Build RPM

Issue the rpmbuild command:  



<div class="highlighter-rouge">
<pre class="highlight">
<code> 

localhost:~$<mark> sudo rpmbuild -ba iperf.spec </mark>
Executing(%prep): /bin/sh -e /var/tmp/rpm-tmp.59743
+ umask 022
+ cd /usr/lib64/rpm/../../src/rpm/BUILD
+ cd /usr/src/rpm/BUILD
+ rm -rf iperf-2.0.9
+ /bin/tar -xf -

----------------------------- snip output -------------------------------

Requires: libc.so.6()(64bit) libc.so.6(GLIBC_2.14)(64bit) libc.so.6(GLIBC_2.2.5)(64bit) libc.so.6(GLIBC_2.3)(64bit) libc.so.6(GLIBC_2.7)(64bit) libgcc_s.so.1()(64bit) libgcc_s.so.1(GCC_3.0)(64bit) libm.so.6()(64bit) libm.so.6(GLIBC_2.2.5)(64bit) libpthread.so.0()(64bit) libpthread.so.0(GLIBC_2.2.5)(64bit) libpthread.so.0(GLIBC_2.3.2)(64bit) librt.so.1()(64bit) librt.so.1(GLIBC_2.2.5)(64bit) libstdc++.so.6()(64bit) libstdc++.so.6(CXXABI_1.3)(64bit) libstdc++.so.6(GLIBCXX_3.4)(64bit) rtld(GNU_HASH)
Checking for unpackaged file(s): /usr/lib64/rpm/check-files /usr/lib64/rpm/../../../var/tmp/iperf-root
Wrote: /usr/src/rpm/SRPMS/iperf-2.0.9-XR_6.1.1.src.rpm
Wrote: /usr/src/rpm/RPMS/x86_64/iperf-2.0.9-XR_6.1.1.x86_64.rpm
localhost:~$ 

</code>
</pre>
</div>

**The final RPM should be available in `/usr/src/rpm/RPMS/x86_64`**:  
  
```shell
localhost:~$ ls -l /usr/src/rpm/RPMS/x86_64/
total 48
-rw-r--r-- 1 root root 48119 Jul 17 16:46 iperf-2.0.9-XR_6.1.1.x86_64.rpm
localhost:~$ 

```


### Transfer the iperf RPM to router  


We can transfer the iperf RPM to the router directly over the management network.  

First determine the forwarded port for XR linux shell (port 57722) for the running router:  

**This command must of course be issued from your laptop running the vagrant environment**
{: .notice--warning}  


<div class="highlighter-rouge">
<pre class="highlight">
<code> 

AKSHSHAR-M-K0DS:native-app-topo-bootstrap akshshar$<mark> vagrant port rtr </mark>
The forwarded ports for the machine are listed below. Please note that
these values may differ from values configured in the Vagrantfile if the
provider supports automatic port collision detection and resolution.

    22 (guest) => 2223 (host)
<mark>57722 (guest) => 2222 (host) </mark>
AKSHSHAR-M-K0DS:native-app-topo-bootstrap akshshar$ 

</code>
</pre>
</div>
 
   
Get back into wrl7_build and use HOST ip address = `10.0.2.2` with port 2222 to transfer the RPM to the router over the management network:  

The password for user vagrant on the router is **"vagrant"**.
{: .notice--info}  

<div class="highlighter-rouge">
<pre class="highlight">
<code>
localhost:~$ 
localhost:~$<mark> scp -P 2222 /usr/src/rpm/RPMS/x86_64/iperf-2.0.9-XR_6.1.1.x86_64.rpm vagrant@10.0.2.2:/home/vagrant/ </mark>
vagrant@10.0.2.2's password: 
iperf-2.0.9-XR_6.1.1.x86_64.rpm                                                                100%   47KB  47.0KB/s   00:00    
localhost:~$ 
</code>
</pre>
</div>  



## Install iperf as native WRL7 app  

Login to the router and install the iperf RPM transferred in the previous step using yum:  

<div class="highlighter-rouge">
<pre class="highlight">
<code>

xr-vm_node0_RP0_CPU0:~$<mark> pwd
/home/vagrant </mark>
xr-vm_node0_RP0_CPU0:~$<mark> ls -l iperf-2.0.9-XR_6.1.1.x86_64.rpm 
-rw-r--r-- 1 vagrant vagrant 48011 Jul 17 21:11 iperf-2.0.9-XR_6.1.1.x86_64.rpm </mark>
xr-vm_node0_RP0_CPU0:~$ 
xr-vm_node0_RP0_CPU0:~$ 
xr-vm_node0_RP0_CPU0:~$<mark> sudo yum install localonly iperf-2.0.9-XR_6.1.1.x86_64.rpm </mark> 
Loaded plugins: downloadonly, protect-packages, rpm-persistence
Setting up Install Process
No package localonly available.
Examining iperf-2.0.9-XR_6.1.1.x86_64.rpm: iperf-2.0.9-XR_6.1.1.x86_64
Marking iperf-2.0.9-XR_6.1.1.x86_64.rpm to be installed
Resolving Dependencies
--> Running transaction check
---> Package iperf.x86_64 0:2.0.9-XR_6.1.1 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

=================================================================================================================================
 Package               Arch                   Version                         Repository                                    Size
=================================================================================================================================
Installing:
 iperf                 x86_64                 2.0.9-XR_6.1.1                  /iperf-2.0.9-XR_6.1.1.x86_64                 103 k

Transaction Summary
=================================================================================================================================
Install       1 Package

Total size: 103 k
Installed size: 103 k
<mark>Is this ok [y/N]: y </mark>
Downloading Packages:
Running Transaction Check
Running Transaction Test
Transaction Test Succeeded
Running Transaction
  Installing : iperf-2.0.9-XR_6.1.1.x86_64                                                                                   1/1 

Installed:
  iperf.x86_64 0:2.0.9-XR_6.1.1                                                                                                  

Complete!
xr-vm_node0_RP0_CPU0:~$ 


</code>
</pre>
</div>  


Check the installation:  

```shell

xr-vm_node0_RP0_CPU0:~$ iperf -v
iperf version 2.0.9 (1 June 2016) pthreads
xr-vm_node0_RP0_CPU0:~$ 

```  


We're all set!
{: .notice--success}  


## Test the Native app  

As we have seen in greater detail in the LXC container app tutorial, we need to set the src-hint for applications to ensure reachability in routed networks.  

[Setting the src-hint for application traffic]({{ base_path }}/tutorials/2016-06-16-xr-toolbox-part-4-bring-your-own-container-lxc-app/#set-the-src-hint-for-application-traffic)  


<div class="highlighter-rouge">
<pre class="highlight">
<code>
AKSHSHAR-M-K0DS:native-app-topo-bootstrap akshshar$<mark> vagrant port rtr </mark>
The forwarded ports for the machine are listed below. Please note that
these values may differ from values configured in the Vagrantfile if the
provider supports automatic port collision detection and resolution.

    22 (guest) => 2223 (host)
<mark>57722 (guest) => 2222 (host) </mark>
AKSHSHAR-M-K0DS:native-app-topo-bootstrap akshshar$<mark> ssh -p 2223 vagrant@localhost </mark>
vagrant@localhost's password: 


RP/0/RP0/CPU0:ios#
RP/0/RP0/CPU0:ios#conf t
Sun Jul 17 21:23:04.140 UTC
RP/0/RP0/CPU0:ios(config)#<mark>tpa address-family ipv4 update-source loopback 0</mark>
RP/0/RP0/CPU0:ios(config)#<mark>commit</mark>
Sun Jul 17 21:23:23.464 UTC
RP/0/RP0/CPU0:ios(config)#end
RP/0/RP0/CPU0:ios#
RP/0/RP0/CPU0:ios#<mark>bash -c ip route</mark>
Sun Jul 17 21:23:35.008 UTC
<mark>default dev fwdintf  scope link  src 1.1.1.1</mark> 
10.0.2.0/24 dev Mg0_RP0_CPU0_0  proto kernel  scope link  src 10.0.2.15 
RP/0/RP0/CPU0:ios#

</code>
</pre>
</div>



