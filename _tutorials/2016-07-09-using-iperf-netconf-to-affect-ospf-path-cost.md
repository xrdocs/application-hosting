---
published: true
date: '2016-07-09 23:47 -0700'
title: Using iperf + netconf to affect OSPF path cost
position: hidden
author: Akshat Sharma
excerpt: OSPF path remediation using container based iperf and netconf
tags:
  - vagrant
  - iosxr
  - cisco
  - linux
  - iperf
  - ospf
  - netconf
---

{% include toc icon="table" title="Launching a Container App" %}
{% include base_path %}
  

## Introduction

If you haven't checked out the XR toolbox Series, then you can do so here:  

>
[XR Toolbox Series]({{ base_path }}/tags/#xr-toolbox)

  
This series is meant to help a beginner get started with application-hosting on IOS-XR.  
  
  
**In this tutorial** we intend to utilize techniques learnt in the above series to solve a path remediation problem:  
  
*  Set up a couple of paths between two routers. In this example we connect the routers back-to-back and set up ECMP (equal cost multipath) links between them. One interface is forcefully selected as the reference link based on OSPF cost configuration.  

*  Use a monitoring technique to determine the bandwidth, jitter, latency etc. parameters along the traffic path. In this example we use iperf inside a container.  

*  Simulate network degradation (through link flaps) to test a python app (inside the LXC) that uses iperf's results to detect the flaps and causes failover by changing the OSPF path cost over a netconf session.  

This is illustrated below:  

![iperf-ospf-ncclient-demo](https://camo.githubusercontent.com/a30938cc2dd9c0788b701677fbb5398bc5bb6646/68747470733a2f2f7872646f63732e6769746875622e696f2f7872646f63732d696d616765732f6173736574732f7475746f7269616c2d696d616765732f6f7370665f6e635f69706572662e6a7067)  


## Create the container App (tar ball)  

As shown in the above figure, we intend to create an application running inside a container on IOS-XR.  

We're fully aware that most users would like to run things on their own laptop as they develop applications and test them.   
&nbsp;  
Owing to the slightly beefy requirements of IOS-XRv vagrant instances, running more than two vagrant instances at a time may be a problem for most users.   
So, we'll create our container app in a development vagrant instance first, save the app and then destroy the development instance before proceeding with the 2 router topology.
{: .notice--info}  


To start off, create a development directory of your choice where we'll spin up an Ubuntu instance for the creation of our container tar ball:  

```shell
AKSHSHAR-M-K0DS:~ akshshar$ mkdir iperf_nc_dev
AKSHSHAR-M-K0DS:~ akshshar$ cd iperf_nc_dev/
AKSHSHAR-M-K0DS:iperf_nc_dev akshshar$ 
```  

Spin up a vagrant ubuntu instance:  

<div class="highlighter-rouge">
<pre class="highlight">
<code>

AKSHSHAR-M-K0DS:iperf_nc_dev akshshar$<mark> vagrant init ubuntu/trusty64 </mark>
A `Vagrantfile` has been placed in this directory. You are now
ready to `vagrant up` your first virtual environment! Please read
the comments in the Vagrantfile as well as documentation on
`vagrantup.com` for more information on using Vagrant.
AKSHSHAR-M-K0DS:iperf_nc_dev akshshar$ 
AKSHSHAR-M-K0DS:iperf_nc_dev akshshar$<mark> vagrant up </mark>
Bringing machine 'default' up with 'virtualbox' provider...
==> default: Importing base box 'ubuntu/trusty64'...
==> default: Matching MAC address for NAT networking...
==> default: Checking if box 'ubuntu/trusty64' is up to date...
------------------------------ snip output ------------------------------------
</code>
</pre>
</div> 


Once it's up, ssh into it and install lxc-tools:  

```shell

vagrant ssh

sudo apt-get install lxc 

```  

Create  an ubuntu LXC container inside your vagrant instance:  


<div class="highlighter-rouge">
<pre class="highlight">
<code>

vagrant@vagrant-ubuntu-trusty-64:~$ <mark> sudo lxc-create -t ubuntu --name nc_iperf </mark>
Checking cache download in /var/cache/lxc/trusty/rootfs-amd64 ... 
Installing packages in template: ssh,vim,language-pack-en
Downloading ubuntu trusty minimal ...
I: Retrieving Release 
I: Retrieving Release.gpg 
I: Checking Release signature
------------------------------ snip output ------------------------------------
</code>
</pre>
</div> 


Start the container. You will be dropped into the console once boot is complete.  
  
Username:  **ubuntu**  
Password:  **ubuntu**  
{: .notice--info}  


<div class="highlighter-rouge">
<pre class="highlight">
<code>
vagrant@vagrant-ubuntu-trusty-64:~$<mark> sudo lxc-start --name nc_iperf </mark>
&lt;4&gt;init: hostname main process (3) terminated with status 1
&lt;4&gt;init: plymouth-upstart-bridge main process (5) terminated with status 1
&lt;4&gt;init: plymouth-upstart-bridge main process ended, respawning


Ubuntu 14.04.4 LTS nc_iperf console

<mark>nc_iperf login: ubuntu</mark>
<mark>Password:      </mark>
Welcome to Ubuntu 14.04.4 LTS (GNU/Linux 3.13.0-87-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

ubuntu@nc_iperf:~$ 
</code>
</pre>
</div> 

Install pip and iperf inside the container  

<div class="highlighter-rouge">
<pre class="highlight">
<code>
ubuntu@nc_iperf:~$<mark> sudo apt-get -y install python-pip iperf</mark>
[sudo] password for ubuntu: 
Reading package lists... Done
Building dependency tree      
------------------------------ snip output ------------------------------------
</code>
</pre>
</div> 
  
  
  
