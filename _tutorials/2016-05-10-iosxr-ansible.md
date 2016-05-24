---
permalink: "/tutorials/IOSXR-Ansible"
author: Mike Korshunov
excerpt: Getting started with IOSXR and Ansible playbooks
published: false
title: "IOS-XR Ansible"
date: "2016-05-16 01:18 +0530"
tags: 
  - vagrant
  - iosxr
  - cisco
  - linux
  - Ansible
---

{% include toc icon="table" title="IOS-XR Ansible" %}



# Introduction
In this tutorial we are going to cover IOS-XRv configuration with Ansible. We will set up and try a very simple configuration with a Linux machine connected to IOS-XRv.


## Prerequisites
- Computer with 4-5GB free RAM;
- Vagrant;
- Ansible;
- [IOS-XRv image](http://engci-maven-master.cisco.com/artifactory/appdevci-snapshot/);
- [Vagrantfile and scripts for provisioning](https://github.com/Maikor/IOSXR-Ansible-tutorial)

### Vagrant pre-setup

Setup was tested on Windows, but it's the same for other OS. To add XR box, box file should be downloaded, after this issue the command:
  
    $ vagrant box add xrv iosxrv-fullk9-x64.box_2016-05-07-19-04-50.box

Image for Ubuntu will be downloaded from official source:
  
    $ vagrant box add ubuntu/trusty64
    
Let's check for result, we should have box available, box ubuntu/trusty64 and xrv should be displayed:

![Box validation](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/xr-ansible-tutorial/xr_ansible_01_box_list.png)

Vagrantfile is in [Github](https://github.com/Maikor/IOSXR-Ansible-tutorial) it contain 2 Vagrant boxes and looks like:

```
Vagrant.configure(2) do |config|

  config.vm.provision "shell", inline: "echo Hello User"

  config.vm.define "ubuntu" do |ubuntu|
    ubuntu.vm.box = "ubuntu/trusty64"
    ubuntu.vm.network :private_network, virtualbox__intnet: "link1", ip: "10.1.1.10"
    ubuntu.vm.provision :shell, path: "ubuntu.sh", privileged: false
    ssh_pub_key = File.readlines("#{Dir.home}/.ssh/id_rsa.pub").first.strip
    #ubuntu.vm.provision "shell", inline: "echo #{ssh_pub_key}"
  end

  config.vm.define "xr" do |xr|
    xr.vm.box = "xrv64"
    #xr.vm.provision 'shell', inline: "echo #{ssh_pub_key} >> /home/vagrant/.ssh/authorized_keys", privileged: false
    xr.vm.network :private_network, virtualbox__intnet: "link1", ip: "10.1.1.20"
  end
   
end

```


Copy files from [Github](https://github.com/Maikor/IOSXR-Ansible-tutorial), change dir to this folder and you are ready to boot up boxes:

```
mkorshun@MKORSHUN-2JPYH MINGW64 ~/Documents/workCisco/tutorial
$ ls
ubuntu.sh*  Vagrantfile  xr-config

mkorshun@MKORSHUN-2JPYH MINGW64 ~/Documents/workCisco/tutorial
$ vagrant up
```


### Ubuntu box pre-configuration

To access Ubuntu box just issue the command (no password required):
```
vagrant ssh ubuntu
```
Ubuntu is already configured via file "ubuntu.sh".
The public key is generated for usage in further passwordless authorization. 
Let's review the content of configuration file "ubuntu.sh". First four lines responsible for downloading required packages for Ansible and updating the system. 
```
sudo apt-get update
sudo apt-get install -y python-setuptools python-dev build-essential git libssl-dev libffi-dev sshpass
sudo easy_install pip 
wget https://bootstrap.pypa.io/ez_setup.py -O - | sudo python
```
Now we are ready to download Ansible itself and IOXR-Ansible repo. 
```
git clone -b vagrant http://gitlab.cisco.com/mkorshun/iosxr-ansible.git
git clone git://github.com/ansible/ansible.git --recursive
```

All files are downloaded and ready for installation. Variables from "ansible_env" should be applied in the system.

```
cd ansible/ && sudo python setup.py install
echo "source /home/vagrant/iosxr-ansible/remote/ansible_env" >> /home/vagrant/.profile

```

Last section responsible for generating key for paswordless authorization and generating base 64 version of it:
```
ssh-keygen -t rsa -f /home/vagrant/.ssh/id_rsa -q -P ""
cut -d" " -f2 ~/.ssh/id_rsa.pub | base64 -d > ~/.ssh/id_rsa_pub.b64

```


### IOS-XRv box pre-configuration

To access XR Linux Shell: 
```
vagrant ssh xr
```

To access XR console it takes one additional step to figure out port (credentials for ssh: vagrant/vagrant):
```
mkorshun@MKORSHUN-2JPYH MINGW64 ~/Documents/workCisco/tutorial
$ vagrant port xr
The forwarded ports for the machine are listed below. Please note that
these values may differ from values configured in the Vagrantfile if the
provider supports automatic port collision detection and resolution.
 22 (guest) = 2223 (host)
 57722 (guest) = 2200 (host)
mkorshun@MKORSHUN-2JPYH MINGW64 ~/Documents/workCisco/tutorial
$ ssh -p 2223 vagrant@localhost
vagrant@localhost's password:
RP/0/RP0/CPU0:ios#
```

Now let's configure an IP address on the IOS-XRv instance. Issue the following command on the XR cli:

```
conf t
hostname xr
interface GigabitEthernet0/0/0/0
 ipv4 address 10.1.1.20 255.255.255.0
 no shutdown
!
commit
end
```

Checking connectivity between boxes:
```
RP/0/RP0/CPU0:ios#ping 10.1.1.10
Mon May  9 08:36:33.071 UTC
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.1.1.10, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/5/20 ms
RP/0/RP0/CPU0:ios#
```


Let's copy public part of key from **Ubuntu** box and allow access without password. At first we should connect to Ubuntu and copy file to XR via SCP
```
vagrant ssh ubuntu
scp -P 57722 /home/vagrant/.ssh/id_rsa.pub  vagrant@<XR's interface address>:/home/vagrant/id_rsa_ubuntu.pub

```

Next step to apply copied key inside XR:
```
vagrant ssh xr
cat /home/vagrant/id_rsa_ubuntu.pub >> /home/vagrant/.ssh/authorized_keys
```


If we want paswordless SSH from Ubuntu to XR CLI, issue the following commands on XR:

```
scp vagrant@10.1.1.10:/home/vagrant/.ssh/id_rsa_pub.b64 /disk0:/id_rsa_pub.b64


crypto key import authentication rsa disk0:/id_rsa_pub.b64
```

File id_rsa_pub.b64 was created by provisioning script "Ubuntu.sh", during Vagrant provisioning.


## Run the playbooks

At Ubuntu box let's configure Ansible prerequisites. 
We need to configure 2 files: ansible_hosts and ansible_env
The first file: ansible_hosts contains the ip address of the XR instance.
We also specify a user to connect to the machine: "ansible_ssh_user=root"
In the second file: ansible_env , we set up the environment for Ansible.
We will not care about [YDK](https://github.com/CiscoDevNet/ydk-py-samples) for now, it's topic for another tutorial. Note, that files ansible_hosts and ansible_env preconfigured for our needs. 


```
cd iosxr-ansible/
cd remote/

vagrant@vagrant-ubuntu-trusty-64:~/iosxr-ansible/remote$ cat ansible_hosts
[ss-xr]
10.1.1.20 ansible_ssh_user=vagrant

vagrant@vagrant-ubuntu-trusty-64:~/iosxr-ansible/remote$ cat ansible_env
export BASEDIR=/home/vagrant
export IOSXRDIR=$BASEDIR/iosxr-ansible
export ANSIBLE_HOME=$BASEDIR/ansible
export ANSIBLE_INVENTORY=$IOSXRDIR/remote/ansible_hosts
export ANSIBLE_LIBRARY=$IOSXRDIR/remote/library
export ANSIBLE_CONFIG=$IOSXRDIR/remote/ansible_cfg
export YDK_DIR=$BASEDIR/ydk/ydk-py
export PYTHONPATH=$YDK_DIR

```

Run the playbook:
```
cd ~/iosxr-ansible/remote/
ansible-playbook samples/iosxr_get_facts.yml 
ansible-playbook iosxr_cli.yml -e 'cmd="show interface brief"' 
```

Samples folder contain various playbooks, files started with "show_" using iosxr_cli playbook and passing cmd to XR as parameter. To run playbook as "vagrant" user, playbook should contain string: "become: yes"
Feel free to play with any playbook!
