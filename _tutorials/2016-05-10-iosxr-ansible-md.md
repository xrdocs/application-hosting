---
permalink: "/tutorials/XR-Ansible"
author: Mike Korshunov
excerpt: Getting started with XR and Ansible playbooks
published: true
title: "IOSXR-Ansible"
---
# Introduction
In this tutorial we are going to cover IOS-XRv configuration with Ansible. We will setup and try simplest configuration with Unix machine connected to IOS-XRv.

## Prerequisites
- Computer with with 4-5GB free RAM;
- Vagrant;
- Ansible;
- [IOS-XRv image](http://engci-maven-master.cisco.com/artifactory/appdevci-snapshot/);

### Vagrant pre-setup

Setup was tested on Windows, but it's the same for other OS. To add XR box, image should be downloaded, after this issue the command:
	
    $ vagrant box add xrv iosxrv-fullk9-x64.box_2016-05-07-19-04-50

Image for Ubuntu will be downloaded from official source:
	
    $ vagrant box add ubuntu/trusty64
    
Let's check for result, we should have box available, box ubuntu/trusty64 and xrv should be displayed:

![Box validation](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/xr-ansible-tutorial/xr_ansible_01_box_list.png)

Vagrantfile with 2 Vagrant boxes should be like:

```
Vagrant.configure(2) do |config|

  config.vm.provision "shell", inline: "echo Hello User"

  config.vm.define "ubuntu" do |ubuntu|
    ubuntu.vm.box = "ubuntu/trusty64"
    ubuntu.vm.network :private_network, virtualbox__intnet: "link1", ip: "10.1.1.10"
  end

  config.vm.define "xr" do |xr|
    xr.vm.box = "xrv"
    xr.vm.network :private_network, virtualbox__intnet: "link1", ip: "10.1.1.20"
  end

end

```


Next step to create Vagrantfile and boot up boxes:

mkorshun@MKORSHUN-2JPYH MINGW64 ~/Documents/workCisco/tutorial
$ vagrant up



### Ubuntu box pre-configuration

To access Ubuntu box just issue the command (no password required):
```
vagrant ssh
```


Install Ansible and all required packages for it. 

```
sudo apt-get install python-setuptools python-dev build-essential git libssl-dev libffi-dev
sudo easy_install pip sshpass
wget https://bootstrap.pypa.io/ez_setup.py -O - | sudo python

git clone http://gitlab.cisco.com/aermongk/iosxr-ansible.git
git clone git://github.com/ansible/ansible.git --recursive
cd ansible/ && sudo python setup.py install
```

Create key to access XR in future:
```
vagrant@vagrant-ubuntu-trusty-64:~$ ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/home/vagrant/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
.our identification has been saved in
.pub.public key has been saved in
The key fingerprint is:
4e:a5:24:de:e1:51:7c:85:78:a1:65:b1:08:17:e3:be vagrant@vagrant-ubuntu-trusty-64
The key's randomart image is:
+--[ RSA 2048]----+
|        ..=o==.  |
|         =o*+.   |
|      . + =o.    |
|     . = *       |
|      . S .      |
|       o   .     |
|        . E      |
|                 |
|                 |
+-----------------+
vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$ cat ~/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCfbpi4N3Lcl5i9Y8gd/g4x05IvnIfoPJkhdmGBW2HMFwqWgJQkJF1BM8SuukWeG8+Su4g0Un5tU4/nvbAcqBDR7wFEmB7z7k8VQrXZUxeB4Lc1jEwdDLbxxGOUitLQO+IXlHFJVpkp9Ps6tT82xopaSOQFXKDq0vYXdeEMD/k3NG++0u5pOsJu+kXLIULy1Ix6qvcFDRKbqfT14fU/K7vBYz8gP8Cl+sql9ySm7aOSb0liCBx46/SueBX6uadnMgecVBc1GyvmEX5PwBppUr3Cpby2yOf69iX4NnNcWTTrPY7o7LWuXPNiAgbSAS3R7jVBPyTltB9zCbGq8We5UuVX vagrant@vagrant-ubuntu-trusty-64
vagrant@vagrant-ubuntu-trusty-64:~$ 
```



### IOS-XRv box pre-configuration

To access XR Linux Shell: 
> vagrant ssh xr

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

Now let's  configure IP address at IOS-XRv box. Issue the command at XR CLI.

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
RP/0/RP0/CPU0:ios#ping 10.1.1.20
Mon May  9 08:36:33.071 UTC
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.1.1.20, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/5/20 ms
RP/0/RP0/CPU0:ios#
```


**Doesn't work for me for now, looking for fix**
Let's copy public part of key from Ubuntu box and allow access without password. For IOS-XRv image 6.1.x commands below:


```
run vi /vagrant/home/.ssh/authorized_keys
```

## Run the playbooks

At Ubuntu box let's configure Ansible prerequisite. 
We need to configure 2 files: ansible_hosts and ansible_env
First file ansible_hosts contain ip address of our xr box. Also we will specify user for reaching out to machine: "ansible_ssh_user=vagrant"
In second file ansible_env we should setup the environment for Ansible. 
We will not care about about [YDK](https://github.com/CiscoDevNet/ydk-py-samples) for now, it's topic for another tutorial. 


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

Samples folder contain various playbooks, files started with "show" using iosxr_cli playbook and passing cmd to XR as parameter. 
