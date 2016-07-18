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

Install the latest ncclient code and jinja2 code using pip (required for our app). We also downgrade the cryptography package to 1.2.1 to circumvent a current bug in the package.  


```shell
sudo pip install ncclient jinja2 cryptography==1.2.1

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

<div class="highlighter-rouge">
<pre class="highlight">
<code>
Once everything is up, you should see the three nodes running:  

AKSHSHAR-M-K0DS:vagrant akshshar$ vagrant status
Current machine states:

<mark>rtr1                      running (virtualbox)
devbox                    running (virtualbox)
rtr2                      running (virtualbox)</mark>

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
</code>
</pre>
</div> 


We're all set! Let's test out our application.
{: .notice--success}





## Test out pathchecker!  

Before we begin, let's dump some configuration outputs on rtr1:  

### Check current OSPF cost/path state
<div class="highlighter-rouge">
<pre class="highlight">
<code>
AKSHSHAR-M-K0DS:vagrant akshshar$ vagrant port rtr1
The forwarded ports for the machine are listed below. Please note that
these values may differ from values configured in the Vagrantfile if the
provider supports automatic port collision detection and resolution.

<mark>22 (guest) => 2223 (host)</mark>
 57722 (guest) => 2200 (host)
 58822 (guest) => 58822 (host)
AKSHSHAR-M-K0DS:vagrant akshshar$<mark> ssh -p 2223 vagrant@localhost </mark>
The authenticity of host '[localhost]:2223 ([127.0.0.1]:2223)' can't be established.
RSA key fingerprint is b1:c1:5e:a5:7e:e7:c0:4f:32:ef:85:f9:3d:27:36:0f.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[localhost]:2223' (RSA) to the list of known hosts.
vagrant@localhost's password: 


RP/0/RP0/CPU0:rtr1#
RP/0/RP0/CPU0:rtr1#<mark>show  running-config  router ospf </mark>
Mon Jul 18 15:25:53.875 UTC
router ospf apphost
 area 0
  interface Loopback0
  !
  interface GigabitEthernet0/0/0/0
  !
  <mark>interface GigabitEthernet0/0/0/1
    cost 20</mark>
  !
 !
!

RP/0/RP0/CPU0:rtr1#<mark>show route 2.2.2.2 </mark>
Mon Jul 18 15:26:03.576 UTC

Routing entry for 2.2.2.2/32
  Known via "ospf apphost", distance 110, metric 2, type intra area
  Installed Jul 18 15:18:28.218 for 00:07:35
  Routing Descriptor Blocks
   <mark>10.1.1.20, from 2.2.2.2, via GigabitEthernet0/0/0/0</mark>
      Route metric is 2
  No advertising protos. 
RP/0/RP0/CPU0:rtr1#
</code>
</pre>
</div>   


**We can see that the current OSPF cost on Gig0/0/0/1 is 20, higher than Gig0/0/0/0. Hence as the route to 2.2.2.2 (loopback 0 of rtr2) shows, the current path selected is through Gig0/0/0/0**
{: .notice--warning}  


### Start iperf server on rtr2  

iperf was already installed on rtr2 as a native application (more on native apps here: [XR toolbox part 5: Running a native WRL7 App](https://xrdocs.github.io/application-hosting/tutorials/2016-06-17-xr-toolbox-part-5-running-a-native-wrl7-app/)) during the `vagrant up` process. 

Start iperf server on rtr2 and set it up to accept UDP packets:  

<div class="highlighter-rouge">
<pre class="highlight">
<code>
AKSHSHAR-M-K0DS:vagrant akshshar$<mark> vagrant ssh rtr2 </mark>
Last login: Mon Jul 18 15:57:05 2016 from 10.0.2.2
xr-vm_node0_RP0_CPU0:~$ 
xr-vm_node0_RP0_CPU0:~$<mark> iperf -s -u </mark>
------------------------------------------------------------
Server listening on UDP port 5001
Receiving 1470 byte datagrams
UDP buffer size: 64.0 MByte (default)
------------------------------------------------------------



</code>
</pre>
</div> 

### Start pathchecker on rtr1 (LXC)


SSH into the pathchecker ubuntu container (already brought up as part of vagrant up process) by using port 58822 on your laptop:  

Password for user "ubuntu" : **ubuntu** 
{: .notice--info}

<div class="highlighter-rouge">
<pre class="highlight">
<code>
AKSHSHAR-M-K0DS:vagrant akshshar$ 
AKSHSHAR-M-K0DS:vagrant akshshar$ ssh -p 58822 ubuntu@localhost
The authenticity of host '[localhost]:58822 ([127.0.0.1]:58822)' can't be established.
RSA key fingerprint is 19:54:83:a9:7a:9f:0a:18:62:d1:f3:91:87:3c:e9:0b.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[localhost]:58822' (RSA) to the list of known hosts.
ubuntu@localhost's password: 
Welcome to Ubuntu 14.04.4 LTS (GNU/Linux 3.14.23-WR7.0.0.2_standard x86_64)

 * Documentation:  https://help.ubuntu.com/
Last login: Mon Jul 18 15:19:45 2016 from 10.0.2.2
ubuntu@pathchecker:~$ 
ubuntu@pathchecker:~$ 
ubuntu@pathchecker:~$ 
</code>
</pre>
</div> 

The pc_run.sh script simply runs the pathchecker.py application with a few sample parameters:  

<div class="highlighter-rouge">
<pre class="highlight">
<code>
ubuntu@pathchecker:~$ 
ubuntu@pathchecker:~$<mark> cat ./pathchecker/pc_run.sh </mark>
#!/bin/bash

./pathchecker.py --host 6.6.6.6 -u vagrant -p vagrant --port 830 -c 10 -o apphost -a 0 -i GigabitEthernet0/0/0/0 -s 2.2.2.2  -j 4 -l 5 -f -t 10
ubuntu@pathchecker:~$ 
</code>
</pre>
</div> 

Based on above output, the "-l" option represents the threshold for packet loss and has been set to 5% for this run. Similarly jitter
Start the pathchecker app by running the `pc_run.sh` script in the `pathchecker` repository:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
ubuntu@pathchecker:~$<mark> cd pathchecker/ </mark>
ubuntu@pathchecker:~/pathchecker$<mark> ./pc_run.sh </mark>
Error while opening state file, let's assume low cost state
<mark>Currently, on reference link GigabitEthernet0/0/0/0 </mark>
Starting an iperf run.....
20160718162513,1.1.1.1,62786,2.2.2.2,5001,6,0.0-10.0,1311240,1048992
20160718162513,1.1.1.1,62786,2.2.2.2,5001,6,0.0-10.0,1312710,1048474
20160718162513,2.2.2.2,5001,1.1.1.1,62786,6,0.0-10.0,1312710,1048679,2.453,0,892,0.000,1

bw is
1025.5546875
jitter is
2.453
pkt_loss is
0.000
verdict is
False
Currently, on reference link GigabitEthernet0/0/0/0
Starting an iperf run.....


</code>
</pre>
</div> 

Perfect! The App seems to be running fine on the reference link Gig0/0/0/0.
{: .notice--success}  


### Create impairment on Active path  


With the app running, let's scoot over to "devbox" which will also act as our impairment node.  


<div class="highlighter-rouge">
<pre class="highlight">
<code>
AKSHSHAR-M-K0DS:vagrant akshshar$<mark> vagrant ssh devbox </mark>
Welcome to Ubuntu 14.04.4 LTS (GNU/Linux 3.13.0-87-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

  System information as of Mon Jul 18 16:38:49 UTC 2016

  System load:  0.0               Processes:             76
  Usage of /:   6.3% of 39.34GB   Users logged in:       0
  Memory usage: 32%               IP address for eth0:   10.0.2.15
  Swap usage:   0%                IP address for lxcbr0: 10.0.3.1

  Graph this data and manage this system at:
    https://landscape.canonical.com/

  Get cloud support with Ubuntu Advantage Cloud Guest:
    http://www.ubuntu.com/business/services/cloud


Last login: Mon Jul 18 16:38:50 2016 from 10.0.2.2
vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$ ls
<mark>impair_backup.sh  impair_reference.sh  stop_impair.sh</mark>
vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$<mark> cat impair_reference.sh </mark>
#!/bin/bash
echo "Stopping all current impairments"
sudo tc qdisc del dev eth3 root &> /dev/null
sudo tc qdisc del dev eth4 root &> /dev/null
echo "Starting packet loss on reference link"
<mark>sudo tc qdisc add dev eth3 root netem loss 7% </mark>
vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$ <mark>./impair_reference.sh</mark>
Stopping all current impairments
Starting packet loss on reference link
vagrant@vagrant-ubuntu-trusty-64:~$ 
</code>
</pre>
</div> 


**As we can see, the reference impairment script creates a packet loss of 7% on the reference link**

Take a look at the running pathchecker application on rtr1. It should switch to the backup link once it detects an increase in packet loss beyond 5% (as specified in the pc_run.sh file):  


<div class="highlighter-rouge">
<pre class="highlight">
<code>
<mark>Currently, on reference link GigabitEthernet0/0/0/0</mark>
Starting an iperf run.....
20160718164745,1.1.1.1,60318,2.2.2.2,5001,6,0.0-10.0,1311240,1048992
20160718164745,1.1.1.1,60318,2.2.2.2,5001,6,0.0-10.0,1312710,1048516
20160718164745,2.2.2.2,5001,1.1.1.1,60318,6,0.0-573.0,1312710,18328,5.215,0,892,0.000,1

bw is
1025.5546875
<mark>jitter is
5.215</mark>
pkt_loss is
0.000
verdict is
True
<mark>Woah! iperf run reported discrepancy, increase cost of reference link !
Increasing cost of the reference link GigabitEthernet0/0/0/0</mark>
Currently, on backup link 
Starting an iperf run.....
20160718164755,1.1.1.1,61649,2.2.2.2,5001,6,0.0-10.0,1311240,1048992
20160718164755,1.1.1.1,61649,2.2.2.2,5001,6,0.0-10.0,1312710,1048577
20160718164755,2.2.2.2,5001,1.1.1.1,61649,6,0.0-583.3,1312710,18002,1.627,0,893,0.000,0

bw is
1025.5546875
jitter is
1.627
pkt_loss is
0.000
verdict is
False
<mark>Currently, on backup link</mark>
Starting an iperf run.....
20160718164805,1.1.1.1,59343,2.2.2.2,5001,6,0.0-10.0,1311240,1048992
20160718164805,1.1.1.1,59343,2.2.2.2,5001,6,0.0-10.0,1312710,1048520
20160718164805,2.2.2.2,5001,1.1.1.1,59343,6,0.0-593.4,1312710,17697,2.038,0,893,0.000,0
</code>
</pre>
</div> 


The app initiated the failover! Let's see how the router responded.


### Verify the Failover was successful

<div class="highlighter-rouge">
<pre class="highlight">
<code>
AKSHSHAR-M-K0DS:vagrant akshshar$ ssh -p 2223 vagrant@localhost
vagrant@localhost's password: 


RP/0/RP0/CPU0:rtr1#
RP/0/RP0/CPU0:rtr1#
RP/0/RP0/CPU0:rtr1#show  running-config  router ospf
Mon Jul 18 17:50:47.851 UTC
router ospf apphost
 area 0
  interface Loopback0
  !
  <mark>interface GigabitEthernet0/0/0/0
   cost 30</mark>
  !
  interface GigabitEthernet0/0/0/1
   cost 20
  !
 !
!

RP/0/RP0/CPU0:rtr1#
</code>
</pre>
</div> 

**Great! The Cost of the Gig0/0/0/0 (reference) interface has been increased to 30, greater than the cost of Gig0/0/0/1. This forces the failover to happen to the Gig0/0/0/1 for the iperf traffic (or any traffic destined to rtr2).** 
{: .notice--info}  


<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:rtr1#show route 2.2.2.2
Mon Jul 18 18:01:49.297 UTC

Routing entry for 2.2.2.2/32
  Known via "ospf apphost", distance 110, metric 21, type intra area
  Installed Jul 18 16:47:45.705 for 01:14:03
  Routing Descriptor Blocks
  <mark>11.1.1.20, from 2.2.2.2, via GigabitEthernet0/0/0/1</mark>
      Route metric is 21
  No advertising protos. 
RP/0/RP0/CPU0:rtr1#
</code>
</pre>
</div> 

**It works! The failover happened and the next hop for 2.2.2.2 (loopback0 of rtr2) is now 11.1.1.20 through Gig0/0/0/1 (the backup link).** 
{: .notice--success}  

We leave upto the reader to try impairing the backup link now and see the App switch the path back to the reference interface.





