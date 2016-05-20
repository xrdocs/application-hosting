---
permalink: "/tutorials/IOSXR-Ansible"
author: Mike Korshunov
excerpt: Getting started with IOSXR and Ansible playbooks
published: false
title: "IOS-XR Ansible"
date: "2016-05-16 01:18 +0530"
---

{% include toc icon="table" title="IOS-XR Ansible" %}



# Introduction
In this tutorial we are going to cover IOS-XRv configuration with Ansible.We will set up and try a very simple configuration with a linux machine connected to IOS-XRv.


## Prerequisites
- Computer with with 4-5GB free RAM;
- Vagrant;
- Ansible;
- [IOS-XRv image](http://engci-maven-master.cisco.com/artifactory/appdevci-snapshot/);
- [Vagrantfile and scripts for provisioning](https://cisco.box.com/s/wykoquj1z76gbedq2mj0pgt0sfhwjvhv)

### Vagrant pre-setup

Setup was tested on Windows, but it's the same for other OS. To add XR box, image should be downloaded, after this issue the command:
  
    $ vagrant box add xrv iosxrv-fullk9-x64.box_2016-05-07-19-04-50

Image for Ubuntu will be downloaded from official source:
  
    $ vagrant box add ubuntu/trusty64
    
Let's check for result, we should have box available, box ubuntu/trusty64 and xrv should be displayed:

![Box validation](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/xr-ansible-tutorial/xr_ansible_01_box_list.png)

Vagrantfile is in [Box](https://cisco.box.com/s/wykoquj1z76gbedq2mj0pgt0sfhwjvhv) it contain 2 Vagrant boxes and looks like:

```
Vagrant.configure(2) do |config|

  config.vm.provision "shell", inline: "echo Hello User"

  config.vm.define "ubuntu" do |ubuntu|
    ubuntu.vm.box = "ubuntu/trusty64"
    ubuntu.vm.network :private_network, virtualbox__intnet: "link1", ip: "10.1.1.10"
    ubuntu.vm.provision :shell, path: "ubuntu.sh", privileged: false
	ssh_pub_key = File.readlines("#{Dir.home}/.ssh/id_rsa.pub").first.strip
  	ubuntu.vm.provision "shell", inline: "echo #{ssh_pub_key}"

  end

  config.vm.define "xr" do |xr|
    xr.vm.box = "xrv64"
    xr.vm.network :private_network, virtualbox__intnet: "link1", ip: "10.1.1.20"
  end

end


```


Copy files from [Box](https://cisco.box.com/s/wykoquj1z76gbedq2mj0pgt0sfhwjvhv), change dir to this folder and you are ready to boot up boxes:

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
Ubuntu is already configured via file "ubuntu.sh". Ansible and requred packages are installed.
Public key is generated for usage in futher passwordless authorization. 


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
interface GigabitEthernet0/0/0/0
 ipv4 address 10.1.1.20 255.255.255.0
 no shutdown
!
commit
end
```

Finally, let's check connectivity between boxes:
```
RP/0/RP0/CPU0:ios#ping 10.1.1.10
Mon May  9 08:36:33.071 UTC
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.1.1.10, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/5/20 ms
RP/0/RP0/CPU0:ios#
```


Let's copy public part of key from **Ubuntu** box and allow access without password. For IOS-XRv image 6.1.x  open file for edit and paste key located in /home/vagrant/.ssh/id_rsa.pub.


```
run nano /home/vagrant/.ssh/authorized_keys
```

If we want paswordless SSH from Ubuntu to XR CLI, issue the following commands on XR:

```
scp vagrant:vagrant@10.1.1.10:/home/vagrant/.ssh/id_rsa_pub.b64 /disk0:/id_rsa_pub.b64


crypto key import authentication rsa disk0:/id_rsa_pub.b64
```

File id_rsa_pub.b64 was created by provisioning script "Ubuntu.sh", during Vagrant provisioning.


## Run the playbooks

At Ubuntu box let's configure Ansible prerequisites. 
We need to configure 2 files: ansible_hosts and ansible_env
The first file: ansible_hosts contains the ip address of the XR instance.
We also specify a user to connect to the machine: "ansible_ssh_user=root"
In the second file: ansible_env , we set up the environment for Ansible.
We will not care about about [YDK](https://github.com/CiscoDevNet/ydk-py-samples) for now, it's topic for another tutorial. Note, that files ansible_hosts and ansible_env preconfigured for our needs. 


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

source ansible_env
```

Run the playbook:
```
cd ~/iosxr-ansible/remote/
ansible-playbook samples/iosxr_get_facts.yml 
ansible-playbook iosxr_cli.yml -e 'cmd="show interface brief"' 
```

Samples folder contain various playbooks, files started with "show_" using iosxr_cli playbook and passing cmd to XR as parameter.
Feel free to play with any playbook!
