---
published: true
date: '2016-05-22 13:49 -0700'
title: 'XR toolbox, Part 1 : IOS-XR Vagrant Quick Start'
permalink: /tutorials/iosxr-vagrant-quickstart
author: Akshat Sharma
excerpt: Getting started with Cisco's IOS-XR Vagrant box
tags:
  - vagrant
  - iosxr
  - cisco
  - xr toolbox
position: top
---

{% include toc icon="table" title="IOS-XR Vagrant: Quick Start" %}
{% include base_path %}

## Introduction

This tutorial is meant to be a quick-start guide to get you up and running with an IOS-XRv Vagrant box.

If you're unfamiliar with Vagrant as a tool for development, testing and design, then here's a quick look at why Vagrant is useful, directly from the folks at Hashicorp:

><https://www.vagrantup.com/docs/why-vagrant/>

>
**To learn more about how to use IOS-XR + Vagrant to**
>
* Test native and container applications on IOS-XR
* Use configuration management tools like Chef/Puppet/Ansible/Shell as Vagrant provisioners   
* Create complicated topologies and a variety of other use cases, 
>
take a look at the rest of the ["XR toolbox" series]({{ base_path }}/tags/#xr-toolbox).
{: .notice--info}

## Pre-requisites:
* [Vagrant](https://www.vagrantup.com/downloads.html) for your Operating System.
* [Virtualbox](https://www.virtualbox.org/wiki/Downloads) for your Operating System.
* A laptop with atleast 4-5G free RAM. (Each XR vagrant instance uses upto 4G RAM, so plan ahead based on the number of XR nodes you want to run)



Tha above items are applicable to all operating systems - Mac OSX, Linux or Windows.  

**If you're using Windows, we would urge you to download a utility like [Git Bash](https://git-scm.com/download/win) so that all the commands provided below work as advertised.**
{: .notice--danger}


## Single Node Bringup

### Download and Add the IOS-XRv vagrant box

>
**IOS-XR Vagrant is currently in Private Beta**  
>
To download the box, you will need an **API-KEY** and a **CCO-ID**
>
To get the API-KEY and a CCO-ID, browse to the following link and follow the steps:  
>
[Steps to Generate API-KEY]({{ site.url }}/getting-started/steps-download-iosxr-vagrant)
{: .notice--danger}


<div class="highlighter-rouge">
<pre class="highlight">
<code>
$ BOXURL="http://devhub.cisco.com/artifactory/appdevci-release/XRv64/latest/iosxrv-fullk9-x64.box"

$ curl <b><mark>-u your-cco-id:API-KEY</mark></b> $BOXURL --output ~/iosxrv-fullk9-x64.box

$ vagrant box add --name IOS-XRv ~/iosxrv-fullk9-x64.box
</code>
</pre>
</div>

Of course, you should replace  your-cco-id with your actual Cisco.com ID and API-KEY with the key you generated and copied using the above [link]({{ site.url }}/getting-started/steps-download-iosxr-vagrant).
{: .notice--danger}

The `vagrant box add` command will take around 10-15 mins as it downloads the box for you.
{: .notice--info}

Once it completes, you should be able to see the box added as "IOS-XRv" in your local vagrant box list:

```shell

AKSHSHAR-M-K0DS:~ akshshar$ vagrant box list
IOS-XRv (virtualbox, 0)
AKSHSHAR-M-K0DS:~ akshshar$ 

```


### Initialize a Vagrantfile

Let's create a working directory (any name would do) for our next set of tasks:

```
mkdir ~/iosxrv; cd ~/iosxrv
```

Now, in this directory, let's initialize a Vagrantfile with the name of the box we added.
 

<div class="highlighter-rouge">
<pre class="highlight">
<code>
AKSHSHAR-M-K0DS:iosxrv akshshar$<mark> vagrant init IOS-XRv </mark>
A `Vagrantfile` has been placed in this directory. You are now
ready to `vagrant up` your first virtual environment! Please read
the comments in the Vagrantfile as well as documentation on
`vagrantup.com` for more information on using Vagrant.
AKSHSHAR-M-K0DS:iosxrv akshshar$
</code>
</pre>
</div>

### Bring up the Vagrant Instance

A simple `vagrant up` will bring up the XR instance

```
vagrant up 
```

This bootup process will take some time, (close to 5 minutes).
{: .notice--info}

You might see some ` Warning: Remote connection disconnect. Retrying...` messages. Ignore them. These messages appear because the box takes longer than a normal linux machine to boot.
{: .notice--warning}

Look for the green "vagrant up" welcome message to confirm the machine has booted:
	

![Vagrant up message](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/vagrant_up_single_node.png)
{: .notice}
 
    
   

Now we have two options to access the Vagrant instance:

### Access the XR Linux shell   

Vagrant takes care of key exchange automatically. We've set things up to make sure that the XR linux shell (running SSH on port 57722) is the environment a user gets dropped into when using `vagrant ssh`

<div class="highlighter-rouge">
<pre class="highlight">
<code>
AKSHSHAR-M-K0DS:simple-mixed-topo akshshar$<mark> vagrant ssh </mark>
xr-vm_node0_RP0_CPU0:~$ 
xr-vm_node0_RP0_CPU0:~$ 
xr-vm_node0_RP0_CPU0:~$ 
</code>
</pre>
</div>

The reason we select the XR linux shell as the default environment and not XR CLI, should be obvious to seasoned users of Vagrant. In the future,Vagrantfiles that integrate chef/puppet/Ansible/Shell as Vagrant provisioners would benefit from linux as the default environment.
{: .notice--info}

### Access XR Console
XR SSH runs on port 22 of the guest IOS-XR instance.  
First, determine the port to which the XR SSH port (port 22) is forwarded by vagrant by using the `vagrant port` command:
  
  
<div class="highlighter-rouge">
<pre class="highlight">
<code>
AKSHSHAR-M-K0DS:simple-mixed-topo akshshar$ <mark> vagrant port </mark>
The forwarded ports for the machine are listed below. Please note that
these values may differ from values configured in the Vagrantfile if the
provider supports automatic port collision detection and resolution.

    <mark>22 (guest) => 2223 (host) </mark>
 57722 (guest) => 2222 (host)
</code>
</pre>
</div>



As shown above, port 22 of XR is fowarded to port 2223:


Use port 2223 to now ssh into XR CLI   

The password is "vagrant"
{: .notice--info}

<div class="highlighter-rouge">
<pre class="highlight">
<code>
AKSHSHAR-M-K0DS:simple-mixed-topo akshshar$<mark> ssh -p 2223 vagrant@localhost</mark>
The authenticity of host '[localhost]:2223 ([127.0.0.1]:2223)' can't be established.
RSA key fingerprint is 65:d1:b8:f6:68:9c:04:a2:d5:db:17:d8:de:04:cb:22.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[localhost]:2223' (RSA) to the list of known hosts.
vagrant@localhost's password: 


RP/0/RP0/CPU0:ios#
RP/0/RP0/CPU0:ios#
RP/0/RP0/CPU0:ios#
</code>
</pre>
</div>
    



## Multi Node Bringup


Let's try to bring up a multi-node topology as shown below:
![Topology](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/vagrant_simple_mixed_topo.png)

### Set up the Vagrantfile
For this purpose, Let's use a Sample vagrantfile located here:  
<https://github.com/ios-xr/vagrant-xrdocs/blob/master/simple-mixed-topo/Vagrantfile>

```
git clone https://github.com/akshshar/vagrant-xrdocs.git
cd vagrant-xrdocs/simple-mixed-topo
```

Shown below is a snippet of the Vagrantfile:

```ruby
Vagrant.configure(2) do |config|

    config.vm.define "rtr1" do |node|
      node.vm.box =  "IOS-XRv"

      # gig0/0/0/0 connected to link2, 
      # gig0/0/0/1 connected to link1, 
      # gig0/0/0/2 connected to link3,
      # auto-config not supported.

      node.vm.network :private_network, virtualbox__intnet: "link2", auto_config: false
      node.vm.network :private_network, virtualbox__intnet: "link1", auto_config: false
      node.vm.network :private_network, virtualbox__intnet: "link3", auto_config: false 

    end

    config.vm.define "rtr2" do |node|
      node.vm.box =  "IOS-XRv"

      # gig0/0/0/0 connected to link1,
      # gig0/0/0/1 connected to link3, 
      # auto-config not supported

      node.vm.network :private_network, virtualbox__intnet: "link1", auto_config: false
      node.vm.network :private_network, virtualbox__intnet: "link3", auto_config: false

    end

```

If you compare this with the topology above it becomes pretty clear how the interfaces of the XR instances are mapped to individual links.   

  

The order of the "private_networks" is important.  
**For each XR node, the first "private_network" corresponds to gig0/0/0/0, the second "private_network" to gig0/0/0/1 and so on.**
{: .notice--warning}

### Bring up the topology
As before, we'll issue a `vagrant up` to bring up the topology.

```
vagrant up
``` 

This will take some time, possibly over 10 minutes. 
{: .notice--warning}

Look for the green "vagrant up" welcome message to confirm the three machines have booted:  

![Vagrant up message](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/vagrant_up_multiple_node.png)
{: .notice}



### Access the nodes

The only point to remember is that in a multinode setup, we "name" each node in the topology.
{: .notice--info}

For example, let's access "rtr2"

* **Access the XR Linux shell**:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
AKSHSHAR-M-K0DS:simple-mixed-topo akshshar$<mark> vagrant ssh rtr2</mark> 
Last login: Tue May 31 05:43:44 2016 from 10.0.2.2
xr-vm_node0_RP0_CPU0:~$ 
xr-vm_node0_RP0_CPU0:~$ 
</code>
</pre>
</div>       


* **Access XR Console**:

Determine the forwarded port for port 22 (XR SSH) for rtr2:


<div class="highlighter-rouge">
<pre class="highlight">
<code>
AKSHSHAR-M-K0DS:simple-mixed-topo akshshar$<mark> vagrant port rtr2 </mark>
The forwarded ports for the machine are listed below. Please note that
these values may differ from values configured in the Vagrantfile if the
provider supports automatic port collision detection and resolution.

    <mark>22 (guest) => 2201 (host)</mark>
 57722 (guest) => 2200 (host)
AKSHSHAR-M-K0DS:simple-mixed-topo akshshar$ 
</code>
</pre>
</div>

For rtr2 port 22, the forwarded port is 2201. So, to get into the XR CLI of rtr2, we use:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
AKSHSHAR-M-K0DS:simple-mixed-topo akshshar$<mark> ssh -p 2201 vagrant@localhost </mark>
The authenticity of host '[localhost]:2201 ([127.0.0.1]:2201)' can't be established.
RSA key fingerprint is 65:d1:b8:f6:68:9c:04:a2:d5:db:17:d8:de:04:cb:22.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[localhost]:2201' (RSA) to the list of known hosts.
vagrant@localhost's password: 


RP/0/RP0/CPU0:ios#
RP/0/RP0/CPU0:ios#
</code>
</pre>
</div>


That's it for the quick-start guide on XR vagrant. Launch your very own XR instance using vagrant and let us know your feedback in the comments below!
{: .notice--success}
