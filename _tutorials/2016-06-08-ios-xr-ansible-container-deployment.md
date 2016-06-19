---
published: false
date: '2016-06-08 19:11 +0300'
title: 'IOS-XR: Ansible container deployment'
author: Mike Korshunov
excerpt: Deploy container at IOS-XR with Ansible
tags:
  - vagrant
  - iosxr
  - cisco
---

## Introduction

The goal of this tutorial to deploy a container (LXC) on XR using an Ansible playbook.

In this tutorial we will use techniques from 2 other tutorials:
* [IOS-XR: Ansible and Vagrant]({{ base_path }}/tutorials/IOSXR-Ansible). enable connectivity between machines and have preinstalled Ansible on devbox instance.

* The [XR Toolbox: Bootstrap XR configuration with Vagrant]({{ base_path }}/tutorials/iosxr-vagrant-bootstrap-config)  using the new shell/bash based automation techniques.

The figure below illustrates the basic steps to undertake to launch an lxc container on IOS-XR 6.0+:

![Container workflow](https://raw.githubusercontent.com/xrdocs/xrdocs-images/gh-pages/assets/tutorial-images/mkorshun/hosted_apps/01_workflow_app_hosting.png)

## Pre-requisite

* Vagrant box added for **IOS-XRv** and **devbox** (usual Ubuntu instance)

* Clone the following repository before we start:

```shell
cd ~/
git clone https://github.com/ios-xr/vagrant-xr.git
cd  vagrant-xr/ansible-tutorials/app_hosting/
```


We are ready to start, boot the boxes.
{: .notice--info}

### Configure Passwordless Access into XR Linux shell
Let's copy public part of key from **devbox** box and allow access without a
password.
First,  connect to the devbox instance and copy file to XR via SCP:

```
vagrant ssh devbox  

scp -P 57722 /home/vagrant/.ssh/id_rsa.pub  vagrant@10.1.1.20:/home/vagrant/id_rsa_ubuntu.pub
```

Now add the copied keys to authorized_keys in XR linux

```
vagrant ssh rtr  

cat /home/vagrant/id_rsa_ubuntu.pub >> /home/vagrant/.ssh/authorized_keys
```

Ansible is ready to work without password.

### Devbox container configuration steps

At first, create new Ubuntu container inside our devbox instance:  
![We need to go deeper](https://raw.githubusercontent.com/xrdocs/xrdocs-images/gh-pages/assets/tutorial-images/mkorshun/hosted_apps/02_we_need_to.png)

All steps to create container covered in details in tutorial:

>
[XR toolbox, Part 4: Bring your own Container (LXC) App]({{ base_path }}/tutorials/2016-06-16-xr-toolbox-part-4-bring-your-own-container-lxc-app)  

Copy the container from devbox to XR Linux.

```shell
sudo scp -P 2200 /var/lib/lxc/cn-01/rootfs/cn_rootfs.tar.gz vagrant@10.0.2.2:/misc/app_host/scratch/cn_rootfs.tar.gz
```

We will use the management network, since vagrant forward the ssh port for each VM and port 2200 is assigned to XR Linux. IP 10.0.2.2 network gateway. There are some speed limitations on traditional copy to XR, so to transfer large files, please use management network.  
{: .notice--info}


To create container, we should provide xml file with description:

```shell
cat /home/vagrant/cn.xml
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
      <source dir='/misc/app_host/scratch/rootfs'/>
      <target dir='/'/>
    </filesystem>
    <console type='pty'/>
  </devices>
</domain>
```

Ansible playbook contain 5 tasks:
```shell
cat deploy_container.yml
---
- hosts: ss-xr

  tasks:
  - copy: src=/home/vagrant/cn.xml dest=/home/vagrant/cn.xml owner=vagrant

  - name: Creates directory
    file: path=/misc/app_host/scratch/rootfs state=directory
    become: yes

  - command: tar -zxf /misc/app_host/scratch/cn_rootfs.tar.gz -C /misc/app_host/scratch/rootfs
    become: yes
    register: output
    ignore_errors: yes
  - debug: var=output.stdout_lines

  - shell: sudo -i virsh  create /home/vagrant/cn.xml
    args:
      warn: no
    register: output
  - debug: var=output.stdout_lines

  - shell: sudo -i virsh list
    args:
      warn: no
    register: output
  - debug: var=output.stdout_lines

```

Tasks overview:

* Task 1 responsible for copy cn.xml to XR ;
* Task 2 creates folder "rootfs" at XR, if it's not existed;
* Task 3 unpack archive with container filesystem;
* Task 4 create container itself; In XR Linux for virsh used alias (issue
  command "type virsh" on XR Linux to check);
* Task 5 verifies, that container is up and running.



Ansible playbook is ready to use. Issue command in devbox:
```shell
ansible-playbook deploy_container.yml
```


Verify container is up from XR Linux shell:

```shell
xr-vm_node0_RP0_CPU0:/misc/app_host/rootfs$ virsh list
 Id    Name                           State
----------------------------------------------------
 4907  sysadmin                       running
 8087  cn                             running
 12057 default-sdr--1                 running

```

Container is up and running. Now you can SSH to it from your laptop:
```shell
ssh -p 58822 ubuntu@127.0.0.1
```
Congratulations!
