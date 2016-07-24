---
published: false
date: '2016-06-06 08:14 +0300'
title: hosted-apps
author: Mike Korshunov
excerpt: Deploy container on IOSXR
tags:
  - vagrant
  - iosxr
  - cisco
---

## Introduction

The goal of this tutorial to deploy container on XR.

In this tutorial we will use techniques from 2 other tutorials:
* [IOS-XR: Ansible and Vagrant]({{ base_path }}/tutorials/IOSXR-Ansible). enable connectivity between machines and have preinstalled Ansible on Ubuntu instance.

* The [XR Toolbox: Bootstrap XR configuration with Vagrant]({{ base_path }}/tutorials/iosxr-vagrant-bootstrap-config)  using the new shell/bash based automation techniques.

The figure below illustrates the basic steps to undertake to launch an lxc container on IOS-XR 6.0+:

![Container workflow](https://raw.githubusercontent.com/xrdocs/xrdocs-images/gh-pages/assets/tutorial-images/mkorshun/hosted_apps/01_workflow_app_hosting.png)

## Pre-requisite

* Vagrant box added for **xrv64** and **Ubuntu**

* Clone the following repository before we start:

```shell
cd ~/
git clone https://github.com/Maikor/IOSXR-Ansible-tutorial.git
cd IOSXR-Ansible-tutorial/app_hosting/
```


We are ready to start, boot the boxes.
{: .notice--info}


### Ubuntu container configuration steps

At first, create new Ubuntu container inside our Ubuntu instance:  
![We need to go deeper](https://raw.githubusercontent.com/xrdocs/xrdocs-images/gh-pages/assets/tutorial-images/mkorshun/hosted_apps/02_we_need_to.png)

```shell
sudo lxc-create -t ubuntu -n cn-01
sudo lxc-start -n cn-01
```
It will take some time to download packages, so after running create command feel free
to grab a cup of coffee{: .notice--info}

Let's add cisco user in the container (existed credentials ubuntu/ubuntu new one are cisco/cisco123):
```shell
sudo adduser cisco
sudo adduser cisco sudo
```

To attach to container use:
```shell
lxc-console -n cn-01
```

Important step to tweak the **sshd_config** file inside the container to start SSH access on a port other than 22.

Port 22 is used by XR for SSH access, and port 57722 by XR linux. In this scenario we set up our container to start SSH access on 58822{: .notice--warning}

```shell
sudo -s
sed -i  s/Port\ 22/Port\ 58822/ /etc/ssh/sshd_config
exit
sudo service ssh restart
```

Now you can ssh to container:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
vagrant@vagrant-ubuntu-trusty-64:~$<mark> sudo lxc-ls --fancy</mark>
NAME   STATE    IPV4        IPV6  AUTOSTART
-------------------------------------------
cn-01  RUNNING  <mark>10.0.3.247</mark>  -     NO

vagrant@vagrant-ubuntu-trusty-64:~$ <mark>ssh -p 58877 cisco@10.0.3.247</mark>
The authenticity of host '[10.0.3.247]:58877 ([10.0.3.247]:58877)' can't be established.
ECDSA key fingerprint is 26:0a:b5:1e:53:e1:48:4f:35:d0:4a:3b:a0:c5:e1:11.
Are you sure you want to continue connecting (yes/no)? yes

</code>
</pre>
</div>

Stop container and prepare it's filesystem to move to XR:
```shell

sudo lxc-stop -n cn-01

sudo -s

cd /var/lib/lxc/cn-01/rootfs

tar -czf cn_rootfs.tar.gz *

```

On XR Linux shell we will create folder for container and unpack the archive:

```shell
sudo cp /home/vagrant/cn_rootfs.tar.gz /misc/app_host/cn_rootfs.tar.gz

cd /misc/app_host
sudo mkdir rootfs
cd rootfs
sudo tar -zxf ../cn_rootfs.tar.gz

```

Last step to create container, we should provide xml file with description:

```shell
/home/vagrant/cn.xml
```

```xml
<domain type='lxc' xmlns:lxc='http://libvirt.org/schemas/domain/lxc/1.0' >
  <name>cn</name>
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
      <source dir='/misc/app_host/rootfs'/>
      <target dir='/'/>
    </filesystem>
    <console type='pty'/>
  </devices>
</domain>
```

Run the container:
```shell
virsh create /home/vagrant/cn.xml
```

Verify container is up:

```shell
xr-vm_node0_RP0_CPU0:/misc/app_host/rootfs$ virsh list
 Id    Name                           State
----------------------------------------------------
 4907  sysadmin                       running
 8087  cn                             running
 12057 default-sdr--1                 running

```

Container is up and running. Now you can SSH to it:
```shell
ssh -p 58822 cisco@127.0.0.1
```
