---
published: true
date: '2016-07-12 08:30 -0400'
title: Building an IOS XRv Vagrant box for Virtualbox
author: Richard Wellum
excerpt: 'Tools to build your own XR vagrant box  '
tags:
  - vagrant
  - iosxr
  - cisco
  - linux
position: hidden
---
## A new way to try out IOS-XR..
An 'IOS XRv (64-bit)' image will be available for users from IOS XR 6.1.1 onwards. This is the successor to the previous IOS XRv (32-bit) QNX based virtual platform. It is based on the latest IOS XR OS which is built on 64-bit Wind River Linux, and has amongst many other changes, a separate Adminplane and complete access to the underlying linux environment. 

You will likely see newer and better variants of this image as we continue to work on tools for developers.  
Our primary focus will be on consistent tooling and workflows such as Vagrant boxes, open source sample applications and build tools to give full flexibility to an end user.   
  
  We hope that these tools help enable developers to consume and learn IOS-XR, build applications for it and participate in our community on Github:  
Sample Applications and Build tools : <https://github.com/ios-xr>   
Open Source Documentation:  <https://github.com/xrdocs> hosted here:  <https://xrdocs.github.io>



## Linux and XR 6.0.0+

*  This version of IOS XR has an infrastructure that allows people to develop and run their own applications in Linux containers on the router itself.

*  Network and System Automation can be accomplished using shell scripts, puppet,chef, Ansible etc.

*  Customers can tap into telemetry that provides improved visibility into a network at a far granular level and through a much more scalable approach than SNMP.  

*  XR configuration itself can be automated using Model Driven APIs with native, common and OpenConfig  YANG models supported.  


## IOS XRv (64-bit)

### Naming and release
*  The image itself is named: name-features-architecture.format, e.g: iosxrv-fullk9-x64.iso

*  Producing a box and releasing it through devhub.cisco.com allows us to get code to the developer far quicker than the standard process.

*  This platform is provided free but as free software has no support - please read the licenses very carefully.  


### Vagrant VirtualBox
Cisco is providing customers with a Vagrant VirtualBox offering. Vagrant is a superb tool for application development. Amongst others you can use this box to:

*  Test native and container applications on IOS-XR

*  Use configuration management tools like Chef/Puppet/Ansible/Shell as Vagrant provisioners

*  Create complicated topologies and a variety of other use cases

This box is designed to come up fully operational with an embedded Vagrantfile that does all of the work to provide a user and tools access to the box. With a simple 'vagrant add' and 'vagrant up' you will have a IOS XR virtual router to play with. 'vagrant ssh' drops the user directly into the XR Linux namespace as user 'vagrant'. Using vagrant port, you can see which port (usually 2222 with a single node) to ssh to get access to the IOS XR Console/CLI.

The user can design their own Vagrantfiles to do more complex bringups including multiple nodes, and bootstrap configuration. There are examples below.

How to get hold of the box, bring it up etc: [https://xrdocs.github.io/application-hosting/tutorials/iosxr-vagrant-quickstart](https://xrdocs.github.io/application-hosting/tutorials/iosxr-vagrant-quickstart)

You will need an active Cisco CCO id.

For tutorials on some of the cool things you can do with this box see: [https://xrdocs.github.io/application-hosting/tutorials/](https://xrdocs.github.io/application-hosting/tutorials/)

## Cisco has open-sourced the tooling
Finally as was the purpose of this blog, we have open-sourced the code to build the Vagrant VirtualBox from an IOS XR ISO.

[https://github.com/ios-xr/iosxrv-x64-vbox](https://github.com/ios-xr/iosxrv-x64-vbox)

I hope you enjoyed this quick blog. The links above provide far more information. As one of the technical leads behind the new platform, and author of the vagrant tooling I'm very motivated to make this a great platform for Cisco customers. Please contact me at rwellum@cisco.com for any questions or concerns.
