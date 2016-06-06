---
published: true
date: '2016-06-06 00:13 -0700'
title: 'XR Toolbox: Bring your own Container App'
author: Akshat Sharma
excerpt: Bring up your own Container (LXC) + Application on IOS-XR
tags:
  - vagrant
  - iosxr
  - cisco
  - linux
  - containers
  - LXC
  - iperf
---

{% include base_path %}
{% include toc icon="table" title="Bring your own Container" %}


## Introduction

The Techdoc: [Application Hosting on IOS-XR]({{ base_path}}/techdocs/app_hosting_on_iosxr/introduction) dives deep into the IOS-XR architecture, to help explain how a user can deploy an application:  

*  natively (inside the XR process space) OR
*  as a container (LXC)


In this quick start guide we use the IOS-XR vagrant box to bring up an Ubuntu container on IOS-XR and host an application within it.

So, let's get started!


## Pre-requisites

* Meet the pre-requisites specified in the [IOS-XR Vagrant Quick Start guide: Pre-requisites]({{ base_path }}/tutorials/iosxr-vagrant-quickstart#pre-requisites) 
* Clone the following repository: <https://github.com/akshshar/vagrant-xr>, before we start.

```shell
cd ~/
git clone https://github.com/akshshar/vagrant-xr.git
cd vagrant-xr/
```

You will notice a few directories. We will utilize the `lxc-app-topo-bootstrap` directory in this tutorial.
{: .notice--info}

```shell
AKSHSHAR-M-K0DS:vagrant-xr akshshar$ pwd
/Users/akshshar/vagrant-xr
AKSHSHAR-M-K0DS:vagrant-xr akshshar$ ls lxc-app-topo-bootstrap/
Vagrantfile	configs		scripts
AKSHSHAR-M-K0DS:vagrant-xr akshshar$ 
``` 
 


## Understand the topology

For this tutorial, we'll use a two-node topology: An XR vagrant instance connected to an Ubuntu vagrant instance:

![LXC app bootstrap topo](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/lxc-app-bootstroop-topo.png)


The Vagrantfile to bring up this topology is already in your cloned directory:  

`vagrant-xr/lxc-app-topo-bootstrap/Vagrantfile`

```ruby

Vagrant.configure(2) do |config|
  

 config.vm.define "rtr" do |node|
   node.vm.box =  "IOS-XRv"

   # gig0/0/0 connected to "link1"
   # auto_config is not supported for XR, set to false

   node.vm.network :private_network, virtualbox__intnet: "link1", auto_config: false


   #Source a config file and apply it to XR

   config.vm.provision "file", source: "configs/rtr_config", destination: "/home/vagrant/rtr_config"

   config.vm.provision "shell" do |s|
     s.path =  "scripts/apply_config.sh"
     s.args = ["/home/vagrant/rtr_config"]
   end

 end

 
 config.vm.define "devbox" do |node|
   node.vm.box =  "ubuntu/trusty64"

   # eth1 connected to link1
   # auto_config is supported for an ubuntu instance

   node.vm.network :private_network, virtualbox__intnet: "link1", ip: "11.1.1.20"

 end

end
```

**Notice the   `#Source a config file and apply it to XR` section of the Vagrantfile? This is derived from the [Bootstrap XR configuration with Vagrant]({{ base_path }}/tutorials/iosxr-vagrant-bootstrap-config) tutorial. Check it out if you want to know more about how shell provisioning with XR works**
{: .notice--warning}

>
For now, all you need to know is:
>
*  Config that you want to apply to XR on boot goes into the `lxc-app-topo-bootstrap/configs` directory. It is named  "rtr_config" in this example.
*  The script to apply XR config resides in the `lxc-app-topo-bootstrap/scripts` directory. You will not need to bother about it in this tutorial.
{: .notice--info}



The configuration we wish to apply to XR is pretty simple. We want to configure the XR interface: `GigabitEthernet0/0/0/0`s with the ip-address: `11.1.1.10`:

```shell
AKSHSHAR-M-K0DS:vagrant-xr akshshar$ cat lxc-app-topo-bootstrap/configs/rtr_config 
!! XR configuration
!
interface GigabitEthernet0/0/0/0
  ip address 11.1.1.10/24
  no shutdown
!
end
AKSHSHAR-M-K0DS:vagrant-xr akshshar$ 
```

Take a look at the Vagrantfile above, again. We use the Vagrant auto_config capabilities to make sure "eth1" interface of the Ubuntu VM (called devbox) is configured in the same subnet (11.1.1.20) as XR gig0/0/0/0.
{: .notice--info}


## Bring up the topology

### Download and Add the XR Vagrant box

```
BOX_URL="http://engci-maven-master.cisco.com/artifactory/simple/appdevci-snapshot/XRv64/latest/iosxrv-fullk9-x64.box"

vagrant box add --name IOS-XRv $BOX_URL

```
The `vagrant box add` command will take around 10-15 mins as it downloads the box for you.
{: .notice--info}

Once it completes, you should be able to see the box added as "IOS-XRv" in your local vagrant box list:

```shell

AKSHSHAR-M-K0DS:~ akshshar$ vagrant box list
IOS-XRv (virtualbox, 0)
AKSHSHAR-M-K0DS:~ akshshar$ 

```

### Bring up the topology

Make sure you're in the `lxc-app-topo-bootstrap/` directory and issue a `vagrant up`

<div class="highlighter-rouge">
<pre class="highlight">
<code>
AKSHSHAR-M-K0DS:lxc-app-topo-bootstrap akshshar$<mark>pwd</mark>
<mark>/Users/akshshar/vagrant-xr/lxc-app-topo-bootstrap</mark>
AKSHSHAR-M-K0DS:lxc-app-topo-bootstrap akshshar$<mark> vagrant up </mark>
Bringing machine 'rtr' up with 'virtualbox' provider...
Bringing machine 'devbox' up with 'virtualbox' provider...
==> rtr: Importing base box 'IOS-XRv'...
==> rtr: Matching MAC address for NAT networking...
==> rtr: Setting the name of the VM: lxc-app-topo-bootstrap_rtr_1465208784531_75603
==> rtr: Clearing any previously set network interfaces...
==> rtr: Preparing network interfaces based on configuration...
    rtr: Adapter 1: nat
    rtr: Adapter 2: intnet
==> rtr: Forwarding ports...
    rtr: 57722 (guest) => 2222 (host) (adapter 1)
    rtr: 22 (guest) => 2223 (host) (adapter 1)
==> rtr: Running 'pre-boot' VM customizations...
==> rtr: Booting VM...
==> rtr: Waiting for machine to boot. This may take a few minutes...
    rtr: SSH address: 127.0.0.1:2222
    rtr: SSH username: vagrant
    rtr: SSH auth method: private key
    rtr: Warning: Remote connection disconnect. Retrying...
    rtr: Warning: Remote connection disconnect. Retrying...
    
   

</code>
</pre>
</div>


Once it completes, you should be able to see both the VMs running by using the `vagrant status` command inside the `lxc-app-topo-bootstrap/` directory:

![Vagrant Status output](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/vagrant_status_lxc_app.png)


### Check Reachability


To get into the Ubuntu "devbox", issue a `vagrant ssh devbox`:

```shell
AKSHSHAR-M-K0DS:lxc-app-topo-bootstrap akshshar$ vagrant ssh devbox
Welcome to Ubuntu 14.04.4 LTS (GNU/Linux 3.13.0-87-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

  System information as of Mon Jun  6 11:20:37 UTC 2016

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


vagrant@vagrant-ubuntu-trusty-64:~$ 
```


From the Ubuntu box, you should be able to Ping the XR Gig0/0/0/0 interface:

```shell
vagrant@vagrant-ubuntu-trusty-64:~$ ping 11.1.1.10 -c 2
PING 11.1.1.10 (11.1.1.10) 56(84) bytes of data.
64 bytes from 11.1.1.10: icmp_seq=1 ttl=255 time=1.56 ms
64 bytes from 11.1.1.10: icmp_seq=2 ttl=255 time=1.44 ms

--- 11.1.1.10 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1003ms
rtt min/avg/max/mdev = 1.447/1.504/1.562/0.069 ms
vagrant@vagrant-ubuntu-trusty-64:~$ 
```


To get into XR linux shell, issue `vagrant ssh rtr`

```shell
AKSHSHAR-M-K0DS:lxc-app-topo-bootstrap akshshar$ vagrant ssh rtr
Last login: Mon Jun  6 11:20:58 2016 from 10.0.2.2
xr-vm_node0_RP0_CPU0:~$ 
xr-vm_node0_RP0_CPU0:~$ifconfig Gi0_0_0_0
Gi0_0_0_0 Link encap:Ethernet  HWaddr 08:00:27:46:1f:b2  
          inet addr:11.1.1.10  Mask:255.255.255.0
          inet6 addr: fe80::a00:27ff:fe46:1fb2/64 Scope:Link
          UP RUNNING NOARP MULTICAST  MTU:1514  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:1 errors:0 dropped:3 overruns:0 carrier:1
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:42 (42.0 B)

xr-vm_node0_RP0_CPU0:~$ 
xr-vm_node0_RP0_CPU0:~$ 
xr-vm_node0_RP0_CPU0:~$ 
```

As we can see, `Gi0_0_0_0`, which corresponds to Gig0/0/0/0 (in XR CLI) shows the ip-address 11.1.1.10, assigned by the shell provisioner in the Vagrantfile.
{: .notice--info}


The topology is up!


## Build a Container tar ball

Before we deploy an LXC on XR, we must build an LXC rootfs tar-ball.
For this purpose, install lxc-tools inside the devbox:

```shell
AKSHSHAR-M-K0DS:lxc-app-topo-bootstrap akshshar$ vagrant ssh devbox
Welcome to Ubuntu 14.04.4 LTS (GNU/Linux 3.13.0-87-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

  System information as of Mon Jun  6 11:20:37 UTC 2016



```

```shell
vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$ sudo apt-get -y install lxc
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following extra packages will be installed:
  bridge-utils cgmanager cloud-image-utils debootstrap dnsmasq-base euca2ools
  genisoimage libaio1 libboost-system1.54.0 libboost-thread1.54.0 liblxc1
  libmnl0 libnetfilter-conntrack3 libnspr4 libnss3 libnss3-nssdb librados2
  librbd1 libseccomp2 libxslt1.1 lxc-templates python-distro-info python-lxml
  python-requestbuilder python-setuptools python3-lxc qemu-utils sharutils
  uidmap
Suggested packages:
```

Now bring up an ubuntu LXC inside the devbox:


















