---
published: false
date: '2016-08-22 18:29 -0400'
title: 'IOSXR: Puppet And Vagrant'
author: Sushrut Shirole
excerpt: Getting started with IOSXR and Puppet
tags:
  - vagrant
  - iosxr
  - cisco
  - linux
  - Puppet
permalink: /tutorials/IOSXR-Puppet/
---
{% include toc icon="table" title="IOS-XR: Puppet" %}

## Introduction
The goal of this tutorial is to set up Puppet Master and Puppet Agent on an Ubuntu and IOS-XRv vagrant instances respectively. This setup was tested on OSX, but the workflow is the same for other environments.  
{: .notice--info}

## Prerequisites
- [Vagrant 1.8.4](https://releases.hashicorp.com/vagrant/1.8.4/) for your Operating System.
- [Virtualbox 5.0.x](http://download.virtualbox.org/virtualbox/) for your Operating System.
- A computer with atleast 8G free memory.
- [Vagrantfile and scripts for provisioning](https://github.com/ios-xr/vagrant-xrdocs/tree/master/puppet-tutorials/app_hosting)

Vagrant 1.8.5 sets the permissions on ~vagrant/.ssh/authorized_keys to 0644 (world-readable) when replacing the insecure public key with a newly generated one. Since sshd will only accept keys readable just by their owner, vagrant up returns an error, since it cannot connect with the new key and it already removed the insecure key. This is Vagrant bug #7610, which affects CentOS Puppet-Master. You can either downgrade to Vagrant 1.8.4 or add ``config.ssh.username = "vagrant"`` and ``config.ssh.password = "vagrant"`` lines to Vagrantfile. More information [here](https://seven.centos.org/2016/08/updated-centos-vagrant-images-available-v1607-01/).
{: .notice--danger}

## The ciscoyang Puppet Module

The `ciscoyang` module allows configuration of IOS-XR through Cisco supported [YANG data models](https://github.com/YangModels/yang/tree/master/vendor/cisco/xr) in JSON/XML format. This module bundles the `cisco_yang` and `cisco_yang_netconf` Puppet types, providers, Beaker tests, and sample manifests to enable users to configure and manage IOS-XR.

This GitHub [repository](https://github.com/cisco/cisco-yang-puppet-module/) contains the latest version of the `ciscoyang` module source code. Supported versions of the `ciscoyang` module are available at Puppet Forge.

### Description

This module enables management of supported Cisco Network Elements through the `cisco_yang` and `cisco_yang_netconf` Puppet types and providers.

A typical role-based architecture scenario might involve a network administrator who uses a version control system to manage various YANG-based configuration files. An IT administrator who is responsible for the puppet infrastructure can simply reference the YANG files from a puppet manifest in order to deploy the configuration

## Setup

### Pre-setup

Clone the vagrant-xrdocs repository with puppet tutorial:

```
$ cd ~
$ git clone https://github.com/ios-xr/vagrant-xrdocs.git
$ cd ~/vagrant-xrdocs/puppet-tutorials/app_hosting/centos-pm/
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

We should now have IOS-XR box available, Use the ``vagrant box list`` command to display the current set of boxes on your system as shown below:

```
$ vagrant box list
IOS-XRv         (virtualbox, 0)
```

The Vagrantfile contains 2 Vagrant boxes; PuppetMaster and IOS-XRv.
If you go to app_hosting directory, you will find that we have two different setups of puppetmaster.

```
$ cd ~/iosxr/vagrant-xrdocs/puppet-tutorials/app_hosting/
$ ls
centos-pm       ubuntu-pm
```

centos-pm and ubuntu-pm has puppetserver installed on CentOS and Ubuntu respectivley. CentOS workflow installs beaker package to run beaker test. So consider centos-pm for development purpose.

Boot up the IOS-XR and Puppet-Master boxes:

```
$ cd ~/vagrant-xrdocs/puppet-tutorials/app_hosting/centos-pm/
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

### Puppet Master  

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
Next, script clones the Puppet-Yang github repository and installs `ciscoyang` puppet module:
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

### Puppet Agent / IOS-XRv   

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


## Usage

### Puppet Manifest

This section explains puppet manifest. This section is only for the user's information. To apply manifest, jump to apply sample manifest section.
{: .notice--warning}

The following example manifest shows how to use `ciscoyang` to configure two VRF instances on a Cisco IOS-XR device. 

```json
node 'default' {
  cisco_yang { 'my-config':
    ensure => present,
    target => '{"Cisco-IOS-XR-infra-rsi-cfg:vrfs": [null]}',
    source => '{"Cisco-IOS-XR-infra-rsi-cfg:vrfs": {
          "vrf":[
            {
                "vrf-name":"VOIP",
                "description":"Voice over IP",
                "vpn-id":{"vpn-oui":875, "vpn-index":3},
                "create":[null]
            },
            {
                "vrf-name":"INTERNET",
                "description":"Generic external traffic",
                "vpn-id":{"vpn-oui":875,"vpn-index":22},
                "create":[null]
            }]
      }
    }',
  }
}
```
The following example manifest shows how to copy a file from the Puppet master to the agent and then reference it from the manifest.

```json
  file { '/root/bgp.json': source => 'puppet:///modules/ciscoyang/models/bgp.json' }

  cisco_yang { '{"Cisco-IOS-XR-ipv4-bgp-cfg:bgp": [null]}':
    ensure => present,
    mode   => replace,
    source => '/root/bgp.json',
  }
}
```

The following example manifest shows how to use `ciscoyang` to configure two VRF instances on a Cisco IOS-XR device using the Yang NETCONF type.

```xml
node 'default' {
  cisco_yang_netconf { 'my-config':
    target => '<vrfs xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-infra-rsi-cfg"/>',
    source => '<vrfs xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-infra-rsi-cfg">
                 <vrf>
                   <vrf-name>VOIP</vrf-name>
                   <create/>
                   <description>Voice over IP</description>
                   <vpn-id>
                     <vpn-oui>875</vpn-oui>
                     <vpn-index>3</vpn-index>
                   </vpn-id>
                 </vrf>
                 <vrf>
                   <vrf-name>INTERNET</vrf-name>
                   <create/>
                   <description>Generic external traffic</description>
                   <vpn-id>
                     <vpn-oui>875</vpn-oui>
                     <vpn-index>22</vpn-index>
                   </vpn-id>
                 </vrf>
              </vrfs>',
    mode => replace,
    force => false,
  }
}
```

### The cisco_yang Puppet Type

Allows IOS-XR to be configured using YANG models in JSON format via gRPC.

**Parameters**

* target

The model path of the target node in YANG JSON format, or a reference to a local file containing the model path. For example, to configure the list of vrfs in IOS-XR, you could specify a `target` of `'{"Cisco-IOS-XR-infra-rsi-cfg:vrfs": [null]}'` or reference a file which contained the same JSON string.

* mode

Determines which mode is used when setting configuration via ensure=>present. Valid values are `replace` and `merge` (which is the default). If `replace` is specified, the current configuration will be replaced by the configuration in the source property (corresponding to the ReplaceConfig gRPC operation). If `merge` is specified, the configuration in the source property will be merged into the current configuration (corresponding to the MergeConfig gRPC operation).

* force

Valid values are `true` and `false` (which is the default). If `true` is specified, then the config in the source property is set on the device regardless of the current value. If `false` is specified (or no value is specified), the default behavior is to set the configuration only if it is different from the running configuration.

**Properties**

* ensure

Determines whether a certain configuration should be present or not on the device. Valid values are `present` and `absent`.

* source

The model data in YANG JSON format, or a reference to a local file containing the model data. This property is only used when ensure=>present is specified. In addition, if `source` is not specified when ensure=>present is used, `source` will default to the value of the `target` parameter. This removes some amount of redundancy when the `source` and `target` values are the same (or very similar).

### The cisco_yang_netconf Puppet Type

Allows IOS-XR to be configured using YANG models in XML format via NETCONF.

**Parameters**

* target

The Yang Netconf XML formatted string or file location containing the filter used to query the existing configuration. For example, to configure the list of vrfs in IOS-XR, you could specify a `target` of '' or reference a file which contained the equivalent Netconf XML string.

* mode

Determines which mode is used when setting configuration. Valid values are `replace` and `merge` (which is the default). If `replace` is specified, the current configuration will be replaced by the configuration in the source property. If `merge` is specified, the configuration in the source property will be merged into the current configuration.

* force

Valid values are `true` and `false` (which is the default). If `true` is specified, then the config in the source property is set on the device regardless of the current value. If `false` is specified (or no value is specified), the default behavior is to set the configuration only if it is different from the running configuration.

**Properties**

* source

The model data in YANG XML Netconf format, or a reference to a local file containing the model data. The Netconf protocol does not allow deletion of configuration subtrees, but instead requires addition of 'operation="delete"' attributes in the YANG XML specifed in the `source` property.

## Apply Sample Puppet Manifest  

**Create Sample Manifest**
A sample manifest file is included in Puppet-Yang git repository. Copy sample manifest file at right location on puppet master.

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

**Apply Sample Manifest**

The sample manifest above requires /root/temp directory on puppet agent to copy XR configuration file vrfs.json. 

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
