---
published: true
date: '2016-07-09 23:47 -0700'
title: 'Pathchecker:  iperf + netconf for OSPF path failover'
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
  - pathchecker
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

*  Use a monitoring technique to determine the bandwidth, jitter, packet loss etc. parameters along the active traffic path. In this example, we utilize a python app called [pathchecker](https://github.com/ios-xr/pathchecker) that in turn uses iperf to measure link health.  

*  Simulate network degradation to force pathchecker (running inside an LXC) to initiate failover by changing the OSPF path cost over a netconf session.  

This is illustrated below:  

![pathchecker-topo](https://xrdocs.github.io/xrdocs-images/assets/images/ospf-iperf-ncclient.png)

## Understand the topology  

As illustrated above, there are 3 nodes in the topology:  

*  **rtr1** : The router on the left. This is the origin of the traffic. We run the `pathchecker` code inside an ubuntu container on this router. The path failover happens rtr1 interfaces as needed.  

*  **devbox** : This node serves two purposes. We use it to create our ubuntu LXC tar ball with the `pathchecker` code before deploying it to the router. It also houses two bridge networks (one for each path) so that we can create very granular impairment on each path to test our app.  

*  **rtr2** : This is the destination router. `pathchecker` uses an iperf client on rtr1 to get a health estimate of the active path. You need an iperf server running on `rtr2` for the pathchecker app to talk to.



## Pre-requisites 

*  Make sure you have [Vagrant](https://www.vagrantup.com/downloads.html) and [Virtualbox](https://www.virtualbox.org/wiki/Downloads) installed on your system.  

*  The system must have 9-10G RAM available.  

*  Go through the Vagrant quick-start tutorial, if you haven't already, to learn how to use Vagrant with IOS-XR:   [IOS-XR vagrant quick-start]({{ base_path }}/tutorials/iosxr-vagrant-quickstart)  

*  It would be beneficial for the user to go through the [XR Toolbox Series]({{ base_path }}/tags/#xr-toolbox). But it is not a hard requirement. Following the steps in this tutorial should work out just fine for this demo.



Once you have everything set up, you should be able to see the IOS-XRv vagrant box in the `vagrant box list` command:  


```shell
  AKSHSHAR-M-K0DS:~ akshshar$ vagrant box list
  IOS-XRv (virtualbox, 0)
  AKSHSHAR-M-K0DS:~ akshshar$ 
```




## Clone the git repo  

The entire environment can be replicated on any environment running vagrant provided around 9-10G RAM is available. The topology will include 2 IOS-XR routers (8G RAM) and an ubuntu instance (around 512 MB RAM).

Clone the pathchecker code from here:  <https://github.com/ios-xr/pathchecker>

<div class="highlighter-rouge">
<pre class="highlight">
<code>
AKSHSHAR-M-K0DS:~ akshshar$<mark> git clone https://github.com/ios-xr/pathchecker.git </mark>
Cloning into 'pathchecker'...
remote: Counting objects: 46, done.
remote: Compressing objects: 100% (28/28), done.
remote: Total 46 (delta 8), reused 0 (delta 0), pack-reused 18
Unpacking objects: 100% (46/46), done.
Checking connectivity... done.
AKSHSHAR-M-K0DS:~ akshshar$ 
</code>
</pre>
</div> 


## Spin up the devbox

Before we spin up the routers, we need to create the container tar ball for the `pathchecker` code. The way I've set up the launch scripts for `rtr1`, the bringup will fail without the container tar ball in the directory.  

Move to the Vagrant directory and launch only the devbox node:  


<div class="highlighter-rouge">
<pre class="highlight">
<code>
AKSHSHAR-M-K0DS:~ akshshar$ cd pathchecker/
AKSHSHAR-M-K0DS:pathchecker akshshar$ cd vagrant/
AKSHSHAR-M-K0DS:vagrant akshshar$ pwd
<mark>/Users/akshshar/pathchecker/vagrant</mark>
AKSHSHAR-M-K0DS:vagrant akshshar$<mark> vagrant up devbox </mark>
Bringing machine 'devbox' up with 'virtualbox' provider...
==> devbox: Importing base box 'ubuntu/trusty64'...

---------------------------- snip output ---------------------------------

==> devbox: Running provisioner: file...
AKSHSHAR-M-K0DS:vagrant akshshar$ 
AKSHSHAR-M-K0DS:vagrant akshshar$ 
AKSHSHAR-M-K0DS:vagrant akshshar$<mark> vagrant status </mark>
Current machine states:

rtr1                      not created (virtualbox)
<mark>devbox                    running (virtualbox)</mark>
rtr2                      not created (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
AKSHSHAR-M-K0DS:vagrant akshshar$ 
</code>
</pre>
</div> 


## Create the Pathchecker LXC tar ball  


### Launch an Ubuntu LXC inside devbox

SSH into "devbox":  

```shell

vagrant ssh devbox

```

Create the pathchecker lxc template:  

<div class="highlighter-rouge">
<pre class="highlight">
<code>

vagrant@vagrant-ubuntu-trusty-64:~$ <mark> sudo lxc-create -t ubuntu --name pathchecker </mark>
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
vagrant@vagrant-ubuntu-trusty-64:~$<mark> sudo lxc-start --name pathchecker </mark>
&lt;4&gt;init: hostname main process (3) terminated with status 1
&lt;4&gt;init: plymouth-upstart-bridge main process (5) terminated with status 1
&lt;4&gt;init: plymouth-upstart-bridge main process ended, respawning


Ubuntu 14.04.4 LTS nc_iperf console

<mark>pathchecker login: ubuntu</mark>
<mark>Password:      </mark>
Welcome to Ubuntu 14.04.4 LTS (GNU/Linux 3.13.0-87-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

ubuntu@pathchecker:~$ 
</code>
</pre>
</div> 


### Install Application dependencies inside LXC

Install iperf and all the dependencies required to install ncclient inside the container. We'll also install git, will need it to fetch our app.  

```shell
sudo apt-get -y install python-pip python-lxml python-dev libffi-dev libssl-dev iperf git
```  

Install the latest ncclient code and jinja2 code using pip (required for our app):

```shell
sudo pip install ncclient jinja2

```

**Perfect, all the dependencies for our app are now installed.**
{: .notice--success}  

### Fetch the application code from Github
Fetch our app from Github:  

```shell
ubuntu@pathchecker:~$ git clone https://github.com/ios-xr/pathchecker.git
Cloning into 'pathchecker'...
remote: Counting objects: 46, done.
remote: Compressing objects: 100% (28/28), done.
remote: Total 46 (delta 8), reused 0 (delta 0), pack-reused 18
Unpacking objects: 100% (46/46), done.
Checking connectivity... done.
ubuntu@pathchecker:~$ 
```

### Change SSH port inside the container

When we deploy the container to IOS-XR, we will share XR's network namespace. Since IOS-XR already uses up port 22 and port 57722 for its own purposes, we need to pick some other port for our container.  

**P.S. If you check the Vagrantfile, we intend to expose port 58822 to the user's laptop directly, on rtr1.**

Let's change the SSH port to 58822:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
ubuntu@pathchecker:~$<mark> sudo sed -i s/Port\ 22/Port\ 58822/ /etc/ssh/sshd_config </mark>
ubuntu@pathchecker:~$ 
</code>
</pre>
</div>  

Check that your port was updated successfully:

```shell
ubuntu@pathchecker:~$ cat /etc/ssh/sshd_config | grep Port
Port 58822
ubuntu@pathchecker:~$ 
```

We're good!
{: .notice--success}


### Package up the LXC   

Now, shutdown the container:  


<div class="highlighter-rouge">
<pre class="highlight">
<code>
ubuntu@pathchecker:~$<mark> sudo shutdown -h now </mark>
ubuntu@pathchecker:~$ 
Broadcast message from ubuntu@pathchecker
	(/dev/lxc/console) at 10:24 ...

The system is going down for halt NOW!
------------------------------ snip output ------------------------------------
</code>
</pre>
</div> 


You're back on devbox.
{: .notice--info}  


Become root and package up your container tar ball

```shell
sudo -s

cd /var/lib/lxc/pathchecker/rootfs/

tar -czvf /vagrant/pathchecker_rootfs.tar.gz *

```

{% capture "dir-share-text" %}
See what we did there? We packaged up the container tar ball as pathchecker_rootfs.tar.gz under **/vagrant** directory. Why is this important?  
Well, Vagrant also automatically shares a certain directory with your laptop (for most types of guest operating systems). So the **/vagrant** is automatically mapped to the directory in which you launched your vagrant instance. To check this, let's get out of our vagrant instance and issue an `ls` in your launch directory:  
<div class="highlighter-rouge">
<pre class="highlight">
<code>
vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$ exit
logout
Connection to 127.0.0.1 closed.
AKSHSHAR-M-K0DS:vagrant akshshar$ 
AKSHSHAR-M-K0DS:vagrant akshshar$<mark> pwd </mark>
/Users/akshshar/pathchecker/vagrant
AKSHSHAR-M-K0DS:vagrant akshshar$<mark> ls -l pathchecker_rootfs.tar.gz  </mark>
-rw-r--r--  1 akshshar  staff  301262995 Jul 18 07:57 pathchecker_rootfs.tar.gz
AKSHSHAR-M-K0DS:vagrant akshshar$ 
</code>
</pre>
</div> 
{% endcapture %}

<div class="notice--warning">
  {{ dir-share-text | markdownify }}
</div> 



## Launch Router Topology

To launch the two routers in the topology, make sure you are in the vagrant directory under pathchecker and issue a `vagrant up`  


```shell

AKSHSHAR-M-K0DS:vagrant akshshar$ pwd
/Users/akshshar/pathchecker/vagrant
AKSHSHAR-M-K0DS:vagrant akshshar$ 
AKSHSHAR-M-K0DS:vagrant akshshar$ vagrant up
Bringing machine 'rtr1' up with 'virtualbox' provider...
Bringing machine 'devbox' up with 'virtualbox' provider...
Bringing machine 'rtr2' up with 'virtualbox' provider...

-------------------------------- snip output --------------------------------------

```  



