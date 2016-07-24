---
published: true
date: "2016-05-16 09:08 -0700"
title: Getting Started on XRv64
permalink: "https://xrdocs.github.io/application-hosting/tutorials/xrv64"
author: "Lisa Roach, Ric Wellum"
excerpt: "Quick start tutorial on using Cisco's Vagrant router image: XRv64"
tags: 
  - vagrant
  - iosxr
  - cisco
  - XRv64
---
This is a quick guide to get up and running with XRv64 and vagrant/Virtualbox.
 
If you do not have Virtualbox and Vagrant tools installed, please refer to the guide for your OS:

* [Mac OS](http://sourabhbajaj.com/mac-setup/Vagrant/README.html)
* [Windows OS](http://www.sitepoint.com/getting-started-vagrant-windows/)
* [Ubuntu OS](http://www.olindata.com/blog/2014/07/installing-vagrant-and-virtual-box-ubuntu-1404-lts)


### Single Node:

	vagrant init xrv64
	vagrant box add --name xrv64 <REPLACE THIS WITH PUBLIC VAGRANT BOX URL> --force
	vagrant up  

This will take a while – can be upwards of 10 minutes starting when you see the “Waiting for machine to boot” message.
The box will be done when you see the following message:

	default: welcome to the IOS-XR XRv64 VirtualBox. To connect to the XR linux shell, use 'vagrant ssh'. To ssh to the XR Console, use: 'vagrant port' (vagrant version > 1.8) to determine the port that maps to guest port 22, then 'ssh vagrant@localhost -p <forwarded port>'


Now you have two options for accessing the Vagrant:
1. Access the opernns App Hosting/ XR Linux spaces
  		
       vagrant ssh

 2.  Access XR Console:
 
         ssh -p <forwarded port> vagrant@127.0.0.1

   * Replace `<forwarded port>` with the port forwarded for your 22 port

      * You can check which ports were forwarded with the command:

		    vagrant port
      
    * The password is “vagrant” 

### Multi Node:
Copy the multi-node Vagrantfile: `cp xr-cp-vbox/Vagrantfile`

Add the box to vagrant: `vagrant box add --name IOS-XRv iosxrv-fullk9-x64.box --force;`
* Or point to a URL: `vagrant box add --name xrv64 <REPLACE THIS WITH PUBLIC VAGRANT BOX URL> --force`

`vagrant up` - this will take some time, possibly over 10 minutes. Wait until you see a welcome message. 

As with the Single Node setup, you have two ways to access the Vagrant images:
1. Access the opernns App Hosting/ XR Linux spaces:

		vagrant ssh rtr1 / vagrant ssh rtr2

2. Access XR Console:

         ssh -p <forwarded port> vagrant@127.0.0.1

   * Replace `<forwarded port>` with the port forwarded for your 22 port of the router you wish to access.

      * You can check which ports were forwarded with the command:

		    vagrant port
      
    * The password is “vagrant”.

   * Repeat these steps for each node.
