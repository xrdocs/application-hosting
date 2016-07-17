---
published: true
date: '2016-07-09 23:47 -0700'
title: iperf + netconf to influence OSPF path cost
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
  
  
**In this tutorial** we intend to utilize almost all the techniques learnt in the above series to solve a path remediation problem:  
  
*  Set up a couple of paths between two routers. Bring up OSPF neighborship on both links. One link is forced to be the reference link by increasing the ospf cost of the other link.

*  Use a monitoring technique to determine the bandwidth, jitter, latency etc. parameters along the traffic path. In this example we use iperf running inside a container.  

*  Simulate network degradation to test a python app (inside the LXC) that uses iperf's results to detect  and causes failover by changing the OSPF path cost over a netconf session.  

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

### Launch the development Vagrant instance (devbox)
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


### Launch an Ubuntu LXC inside devbox

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


### Install Application dependencies inside LXC

Install iperf and all the dependencies required to install ncclient inside the container. We'll also install git, wil need it to fetch our app.  

```shell
sudo apt-get install python-pip python-lxml python-dev libffi-dev libssl-dev iperf git
```  

Install the latest ncclient code and jinja2 code using pip (required for our app):

```shell
sudo pip install ncclient jinja2

```

**Perfect, all the dependencies for our app are now installed.**
{: .notice--info}  

### Fetch the application code from Github
Fetch our app from Github:  

```shell
ubuntu@nc_iperf:~$ git clone https://github.com/ios-xr/ospf-iperf-ncclient
Cloning into 'ospf-iperf-ncclient'...
remote: Counting objects: 15, done.
remote: Total 15 (delta 0), reused 0 (delta 0), pack-reused 14
Unpacking objects: 100% (15/15), done.
Checking connectivity... done.
```

### Package up the LXC   

Now, shutdown the container:  


<div class="highlighter-rouge">
<pre class="highlight">
<code>
ubuntu@nc_iperf:~$<mark> sudo shutdown -h now </mark>
ubuntu@nc_iperf:~$ 
Broadcast message from ubuntu@nc_iperf
	(/dev/lxc/console) at 10:24 ...

The system is going down for halt NOW!
------------------------------ snip output ------------------------------------
</code>
</pre>
</div> 


You're back on your development Vagrant instance. 
{: .notice--info}  


Become root and package up your container tar ball

```shell
sudo -s

cd /var/lib/lxc/nc_iperf/rootfs/

tar -czvf /vagrant/nc_iperf_rootfs.tar.gz *

```

{% capture "dir-share-text" %}
See what we did there? We packaged up the container tar ball as nc_iperf_rootfs.tar.gz under **/vagrant** directory. Why is this important?  
Well, Vagrant also automatically shares certain directories with your laptop (for most types of guest operating systems). So the **/vagrant** is automatically mapped to the directory in which you launched your vagrant instance. To check this, let's get out of our vagrant instance and issue an `ls` in your launch directory:  
<div class="highlighter-rouge">
<pre class="highlight">
<code>
vagrant@vagrant-ubuntu-trusty-64:~$ exit
logout
Connection to 127.0.0.1 closed.
AKSHSHAR-M-K0DS:iperf_nc_dev akshshar$ 
AKSHSHAR-M-K0DS:iperf_nc_dev akshshar$<mark> pwd</mark>
/Users/akshshar/iperf_nc_dev
AKSHSHAR-M-K0DS:iperf_nc_dev akshshar$ ls
Vagrantfile		<mark>nc_iperf_rootfs.tar.gz</mark>
AKSHSHAR-M-K0DS:iperf_nc_dev akshshar$ 
</code>
</pre>
</div> 
{% endcapture %}

<div class="notice--warning">
  {{ dir-share-text | markdownify }}
</div> 


### Destroy devbox

Now we can safely destroy our development Vagrant instance and get ready to launch the router topology.  

<div class="highlighter-rouge">
<pre class="highlight">
<code>
AKSHSHAR-M-K0DS:iperf_nc_dev akshshar$<mark> vagrant destroy </mark>
    default: Are you sure you want to destroy the 'default' VM? [y/N] y
==> default: Forcing shutdown of VM...
==> default: Destroying VM and associated drives...
AKSHSHAR-M-K0DS:iperf_nc_dev akshshar$<mark> ls </mark>
Vagrantfile		<mark>nc_iperf_rootfs.tar.gz</mark>
AKSHSHAR-M-K0DS:iperf_nc_dev akshshar$ 
</code>
</pre>
</div>   



## Launch Router Topology

We've created our app and are ready to launch the 2 router topology. You'll need 2 IOS-XR routers connected over 2 back-to-back links and running OSPF on both links.  

  
To set this up, simply clone the following repository to your laptop:    
<https://github.com/ios-xr/vagrant-xrdocs>

```shell
git clone https://github.com/ios-xr/vagrant-xrdocs  

```  

We are interested in the `ospf-iperf-topo/` directory under the cloned `vagrant-xrdocs` directory.   

```shell
AKSHSHAR-M-K0DS:vagrant-xrdocs akshshar$ pwd
/Users/akshshar/vagrant-xrdocs
AKSHSHAR-M-K0DS:vagrant-xrdocs akshshar$ cd ospf-iperf-topo/
AKSHSHAR-M-K0DS:ospf-iperf-topo akshshar$ ls
Vagrantfile	configs		scripts
AKSHSHAR-M-K0DS:ospf-iperf-topo akshshar$   
```  

This section assumes you already have an IOS-XR vagrant box set up locally. If you don't, follow along here: 
[Download and Add the IOS-XRv vagrant box](https://xrdocs.github.io/application-hosting/tutorials/iosxr-vagrant-quickstart#download-and-add-the-ios-xrv-vagrant-box)).  
  
Once you have an IOS-XR vagrant box, bring up the topology:  

<div class="highlighter-rouge">
<pre class="highlight">
<code>
  
</code>
</pre>
</div>
