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

* Vagrant box added for **IOS-XRv** and **devbox** (usual Ubuntu instance, that also acts as our Ansible Server)

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

### Devbox container creation

>
The user is free to bring their own lxc rootfs tar-ball for deployment on IOS-XR. This section is meant to help a user create a rootfs tar ball from scratch if needed.


![We need to go deeper](https://raw.githubusercontent.com/xrdocs/xrdocs-images/gh-pages/assets/tutorial-images/mkorshun/hosted_apps/02_we_need_to.png)

All the steps required to create a container rootfs are already covered in detail in the tutorial:

>
[XR toolbox, Part 4: Bring your own Container (LXC) App]({{ base_path }}/tutorials/2016-06-16-xr-toolbox-part-4-bring-your-own-container-lxc-app)  
Specifically, head over to the following section of the tutorial:  
[XR toolbox, Part 4.../create-a-container-rootfs]({{ base_path }}/tutorials/tutorials/2016-06-16-xr-toolbox-part-4-bring-your-own-container-lxc-app/#create-a-container-rootfs)
At the end of the section, you should have your very own rootfs (xr-lxc-app-rootfs.tar.gz), ready for deploymenty

Copy and keep the rootfs tar ball in the `/home/vagrant/` directory of your devbox. The Ansible playbook will expect the tar ball in this directory, so make sure an `ls -l` for the tar ball in `/home/vagrant` returns something like:

```shell
ls -l /home/vagrant/xr-lxc-app-rootfs.tar.gz
```

Great! Ansible will copy this tar ball to XR for you.
{: .notice--success}

We will use the management network, since vagrant forward the ssh port for each VM and port 2200 is assigned to XR Linux. IP 10.0.2.2 network gateway. Since the gig interfaces in the Vagrant XR image are rate-limited, to transfer large files, please use management network.  
{: .notice--info}


To create a container, we need an xml file with the specifications for the container. Create the following file in the /home/vagrant directory of your `devbox` :

```shell
cat /home/vagrant/xr-lxc-app.xml
```

```xml
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
      <source dir='/misc/app_host/scratch/xr-lxc-app/'/>
      <target dir='/'/>
    </filesystem>
    <console type='pty'/>
  </devices>
</domain>
```

Ansible playbook contains 5 tasks:  

```shell
cat deploy_container.yml
---
- hosts: ss-xr

  tasks:
  - copy: src=/home/vagrant/xr-lxc-app.xml dest=/home/vagrant/xr-lxc-app.xml owner=vagrant
  
  - copy: src=/home/vagrant/xr-lxc-app-rootfs.tar.gz dest=/misc/app_host/scratch/xr-lxc-app-rootfs.tar.gz owner=vagrant

  - name: Creates directory
    file: path=/misc/app_host/scratch/xr-lxc-app/ state=directory
    become: yes

  - command: tar -zxf /misc/app_host/scratch/xr-lxc-app-rootfs.tar.gz -C /misc/app_host/scratch/xr-lxc-app/
    become: yes
    register: output
    ignore_errors: yes
  - debug: var=output.stdout_lines

  - shell: sudo -i virsh  create /home/vagrant/xr-lxc-app.xml
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

* Task 1 responsible for copy xr-lxc-app.xml to XR ;
* Task 2 creates folder "rootfs" at XR, if it's not existed;
* Task 3 unpack archive with container filesystem (Notice the ignore_errors?- we're simply avoiding the mknod warnings);
* Task 4 create container itself; In XR Linux for virsh used alias (issue
  command "type virsh" on XR Linux to check. "sudo -i" is important, to load up Aliases for the root user);
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
 8087  xr-lxc-app                     running
 12057 default-sdr--1                 running

```

Container is up and running. Now you can SSH to it from your laptop:
```shell
ssh -p 58822 ubuntu@127.0.0.1
```
Congratulations!
