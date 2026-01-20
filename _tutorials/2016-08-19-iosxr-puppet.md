---
published: false
date: '2016-08-19 18:09 -0400'
title: IOSXR-Puppet
author: Sushrut Shirole
excerpt: Getting started with IOSXR and Puppet
tags:
  - vagrant
  - iosxr
  - cisco
  - linux
  - Puppet
  - OSX
  - Windows
permalink: /tutorials/IOSXR-Puppet
---
{% include toc icon="table" title="IOS-XR: Puppet and Vagrant" %}

## Introduction
The goal of this tutorial is to set up Puppet Master on an Ubuntu vagrant instance and Puppet Agent on an IOS-XRv vagrant instance. This setup was tested on OSX, but the workflow is the same for other environments.  
{: .notice--info}

## Prerequisites
- [Vagrant 1.8+](https://www.vagrantup.com/downloads.html) for your Operating System.
- [Virtualbox 5.1+](https://www.virtualbox.org/wiki/Downloads) for your Operating System.
- A computer with atleast 8G free memory.
- [Vagrantfile and scripts for provisioning](https://github.com/ios-xr/vagrant-xrdocs/tree/master/puppet-tutorials/app_hosting)

### Vagrant pre-setup

Clone the vagrant-xrdocs repository with puppet tutorial:

```
$ cd ~
$ git clone https://github.com/ios-xr/vagrant-xrdocs.git
$ cd ~/vagrant-xrdocs/puppet-tutorials/app_hosting/
$ ls
Vagrantfile  iosxrv.sh  scripts  xr_config  configs  puppetmaster.sh
```

To add an IOS-XR box, you need to download it.

>
**IOS-XR Vagrant is currently in Private Beta**  
>
To download the box, you will need an __API-KEY__ and a __CCO-ID__
>
To get the API-KEY and a CCO-ID, browse to the following link and follow the steps:   
>
[Steps to Generate API-KEY]({{ site.url }}/getting-started/steps-download-iosxr-vagrant)
{: .notice--danger}

<div class="highlighter-rouge">
<pre class="highlight">
<code>
$ BOXURL="http://devhub.cisco.com/artifactory/appdevci-release/XRv64/latest/iosxrv-fullk9-x64.box"
$ curl <b><mark>-u CCO-ID:API-KEY</mark></b> $BOXURL --output ~/iosxrv-fullk9-x64.box
$ vagrant box add --name IOS-XRv ~/iosxrv-fullk9-x64.box
</code>
</pre>
</div>

Of course, you should replace  CCO-ID with your cisco.com ID and API-KEY with the key you generated and copied using the above [link]({{ site.url }}/getting-started/steps-download-iosxr-vagrant).
{: .notice--danger}  

Download Ubuntu image for Puppet Master:

```
$ vagrant box add ubuntu/xenial64
```

We should now have both the boxes available, Use the ``vagrant box list`` command to display the current set of boxes on your system as shown below:

```
$ vagrant box list
IOS-XRv         (virtualbox, 0)
ubuntu/xenial64 (virtualbox, 20160815.0.0)
```

The Vagrantfile contains 2 Vagrant boxes; PuppetMaster and IOS-XRv.
Boot up the boxes:

```
$ cd ~/vagrant-xrdocs/puppet-tutorials/app_hosting/
$ ls
Vagrantfile  iosxrv.sh  scripts  xr_config  configs  puppetmaster.sh
$ vagrant up
Bringing machine 'puppetmaster' up with 'virtualbox' provider...
Bringing machine 'iosxrv' up with 'virtualbox' provider...
```

This will take some time. If guest OS logs a message to stderr then you might see few red lines. Ignore them.
{: .notice--warning}

Look for “vagrant up” welcome message to confirm the machine has booted:

```
==> iosxrv: Machine 'iosxrv' has a post `vagrant up` message. This is a message
==> iosxrv: from the creator of the Vagrantfile, and not from Vagrant itself:
==> iosxrv:
==> iosxrv:
==> iosxrv:     Welcome to the IOS XRv (64-bit) VirtualBox.
==> iosxrv:     To connect to the XR Linux shell, use: 'vagrant ssh'.
==> iosxrv:     To ssh to the XR Console, use: 'vagrant port' (vagrant version > 1.8)
==> iosxrv:     to determine the port that maps to guestport 22,
==> iosxrv:     then: 'ssh vagrant@localhost -p <forwarded port>'
==> iosxrv:
==> iosxrv:     IMPORTANT:  READ CAREFULLY
==> iosxrv:     The Software is subject to and governed by the terms and conditions
==> iosxrv:     of the End User License Agreement and the Supplemental End User
==> iosxrv:     License Agreement accompanying the product, made available at the
==> iosxrv:     time of your order, or posted on the Cisco website at
==> iosxrv:     www.cisco.com/go/terms (collectively, the 'Agreement').
==> iosxrv:     As set forth more fully in the Agreement, use of the Software is
==> iosxrv:     strictly limited to internal use in a non-production environment
==> iosxrv:     solely for demonstration and evaluation purposes. Downloading,
==> iosxrv:     installing, or using the Software constitutes acceptance of the
==> iosxrv:     Agreement, and you are binding yourself and the business entity
==> iosxrv:     that you represent to the Agreement. If you do not agree to all
==> iosxrv:     of the terms of the Agreement, then Cisco is unwilling to license
==> iosxrv:     the Software to you and (a) you may not download, install or use the
==> iosxrv:     Software, and (b) you may return the Software as more fully set forth
==> iosxrv:     in the Agreement.
```

## Puppet Master box configuration  

To access the Puppet Master box just issue the ``vagrant ssh`` command (no password required):

```
$ vagrant ssh puppetmaster
```

The Puppet Master instance is already configured via file ["puppetmaster.sh"](https://github.com/ios-xr/vagrant-xrdocs/blob/master/puppet-tutorials/app_hosting/puppetmaster.sh). This section is only for the user's information.
{: .notice--warning}

>
Let's review the ["puppetmaster.sh"](https://github.com/ios-xr/vagrant-xrdocs/blob/master/puppet-tutorials/app_hosting/puppetmaster.sh) script.
The first line adds Puppet Master and IOS-XRv host information in /etc/hosts file.
>
```shell
yes | sudo cp /home/ubuntu/hosts /etc/hosts > /dev/null 2>&1
```
Next, downloads required packages for Puppet Master and updates the system.
>
```shell
wget -q https://apt.puppetlabs.com/puppetlabs-release-pc1-xenial.deb
sudo dpkg -i puppetlabs-release-pc1-xenial.deb > /dev/null 2>&1
sudo apt update -qq > /dev/null 2>&1
sudo apt-get install puppetserver -qq > /dev/null
```
Next, script clones the Puppet-Yang github repository and installs cisco-ciscoyang puppet module:
>
```shell
git clone https://github.com/cisco/cisco-yang-puppet-module.git -q
cd cisco-yang-puppet-module
/opt/puppetlabs/puppet/bin/puppet module build > /dev/null
sudo /opt/puppetlabs/puppet/bin/puppet module install pkg/*.tar.gz
```
>
The last section creates a puppet configuration file and ensures that puppetserver service is running on the Puppet Master
>
```shell
yes | sudo cp /home/ubuntu/puppet.conf /etc/puppetlabs/puppet/puppet.conf
sudo /opt/puppetlabs/bin/puppet resource service puppetserver ensure=running enable=true > /dev/null
```
{: .notice--info}

## IOS-XRv box configuration  

To access the IOS-XRv bash shell just issue the ``vagrant ssh`` command (no password required):

```
$ vagrant ssh iosxrv
```

To access the XR console on IOS-XRv requires an additional step to figure out the ssh port:

```
$ vagrant port iosxrv
The forwarded ports for the machine are listed below. Please note that
these values may differ from values configured in the Vagrantfile if the
provider supports automatic port collision detection and resolution.

    22 (guest) => 2223 (host)
 57722 (guest) => 2200 (host)
 
$ ssh -p 2223 vagrant@localhost # password: vagrant
vagrant@localhost's password:
RP/0/RP0/CPU0:xrv9k#
```

The IOS-XRv instance is already configured via ["iosxrv.sh"](https://github.com/ios-xr/vagrant-xrdocs/blob/master/puppet-tutorials/app_hosting/iosxrv.sh). This section is only for the user's information.
{: .notice--warning}

>
Let's review the ["iosxrv.sh"](https://github.com/ios-xr/vagrant-xrdocs/blob/master/puppet-tutorials/app_hosting/iosxrv.sh) script.
The first section installs puppet agent on IOS-XRv.
>
```shell
sudo rpm --import http://yum.puppetlabs.com/RPM-GPG-KEY-puppetlabs
sudo rpm --import http://yum.puppetlabs.com/RPM-GPG-KEY-reductive
wget -q https://yum.puppetlabs.com/puppetlabs-release-pc1-cisco-wrlinux-7.noarch.rpm
sudo yum install -y puppetlabs-release-pc1-cisco-wrlinux-7.noarch.rpm > /dev/null
sudo yum update -y > /dev/null
sudo yum install -y puppet > /dev/null
```
Next, downloads and installs grpcs gem. 
>
```shell
export PATH=/opt/puppetlabs/puppet/bin:$PATH
wget -q https://rubygems.org/downloads/grpc-0.15.0-x86_64-linux.gem
sudo /opt/puppetlabs/puppet/bin/gem install --no-rdoc --no-ri grpc > /dev/null
```
Next, copies configuration files:
>
```shell
yes | sudo cp /home/vagrant/puppet.conf /etc/puppetlabs/puppet/puppet.conf
yes | sudo cp /home/vagrant/hosts /etc/hosts
yes | sudo cp /home/vagrant/cisco_yang.yaml /etc/cisco_yang.yaml
```
{: .notice--info}


## Applying puppet manifest

### Create puppet manifest file on puppet master.  

A sample manifest file is included in Puppet-Yang git repository. Copy sample manifest file at right location.

```
$ vagrant ssh puppetmaster
$ find . -name site.pp
./cisco-yang-puppet-module/examples/site.pp
$ sudo cp ./cisco-yang-puppet-module/examples/site.pp /etc/puppetlabs/code/environments/production/manifests/
$ exit
```

The sample puppet manifest looks like: 

```json
node 'default' {
  file { "/root/temp/vrfs.json":
    source => "puppet:///modules/ciscoyang/models/defaults/vrfs.json"}

  # Configure two vrfs (VOIP & INTERNET)
  cisco_yang { '{"Cisco-IOS-XR-infra-rsi-cfg:vrfs": [null]}':
    ensure => present,
    source => '/root/temp/vrfs.json',
  }
}
```
{: .notice}

### Run puppet agent on IOS-XRv.  

The sample manifest requires /root/temp directory to copy XR configuration file vrfs.json. 

```
$ vagrant ssh iosxrv
$ sudo mkdir /root/temp/
$ exit
```

The vrfs.json file:

```json
{
   "Cisco-IOS-XR-infra-rsi-cfg:vrfs":{
      "vrf":[{
            "vrf-name":"VOIP",
            "description":"Voice over IP",
            "vpn-id":{"vpn-oui":87, "vpn-index":3},
            "create":[null]
         },
         {
            "vrf-name":"INTERNET",
            "description":"Generic external traffic",
            "vpn-id":{"vpn-oui":85, "vpn-index":22},
            "create":[null]
         }]
   }
}
```

Run puppet agent ``puppet agent -t`` to apply configuration on IOS-XRv. 

```
$ vagrant ssh iosxrv
$ sudo puppet agent -t
$ exit
```

Verify the applied configuration:

```
$ ssh -p 2223 vagrant@localhost # password: vagrant
vagrant@localhost's password:

RP/0/RP0/CPU0:xrv9k#show running-config vrf

Fri Aug 19 00:02:40.505 UTC
vrf VOIP
 description Voice over IP
 vpn id 57:3
!
vrf INTERNET
 description Generic external traffic
 vpn id 55:16
!
$ exit
```
{: .notice--success}
