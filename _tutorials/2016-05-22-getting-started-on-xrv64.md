---
published: true
date: "2016-05-22 13:49 -0700"
title: "Getting Started on Vagrant IOS-XR"
permalink: "/tutorials/iosxr-vagrant-quickstart"
author: Lisa Roach
excerpt: "Getting started with Cisco's XRv64 Vagrant Instance."
tags: 
  - vagrant
  - iosxr
  - cisco
  - XRv64
position: hidden
---
### Quick Start Using XRv64 Box image
Quick guide to get up and running with XRv64 and vagrant/Virtualbox.

<pre>
function filesize
{
    local file=$1
    <b>size=`stat -c %s $file 2>/dev/null` # linux</b>
    if [[ $? -eq 0 ]]; then
        echo $size
        return 0
    fi

    eval $(stat -s $file) # macos
    if [[ $? -eq 0 ]]; then
        echo $st_size
        return 0
    fi

    echo 0
    return -1
}
</pre>

#### Requirements:
- [Vagrant](https://www.vagrantup.com/) and [Virtualbox](https://www.virtualbox.org/) installed on your laptop

#### Single Node:

	vagrant init xrv64
  ![vagrant init xrv64](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/xrv64_vagrant_init.png)
   
	vagrant box add --name xrv64 <REPLACE THIS WITH PUBLIC VAGRANT BOX URL> --force
    
  ![vagrant box add](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/xrv64_vagrant_add.png)
  
	vagrant up 

This will take some time, possibly over 10 minutes once you see the "Waiting for machine to boot" message.  Look for the green "vagrant up" welcome message to confirm the machine has booted:
	
   ![vagrant up](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/xrv64_vagrant_up_s.png)
    


Now you have two options for accessing the Vagrant instance:

1. Access the XR Linux shell:
  		
       vagrant ssh

  ![vagrant ssh](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/xrv64_vagrant_ssh_s.png)

2.  Access XR Console:
 
        ssh -p <forwarded port> vagrant@127.0.0.1

   * Replace `<forwarded port>` with the port forwarded for your 22 port

      * You can check which ports were forwarded with the command:

		    vagrant port
      
    * The password is “vagrant” 
    
  ![ssh console](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/xrv64_ssh_console_s.png)

#### Multi Node:

![Topology](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/xrv64_topo_m.png)

Copy the multi-node Vagrantfile: `cp xr-cp-vbox/Vagrantfile`

Add the box to vagrant: `vagrant box add --name IOS-XRv iosxrv-fullk9-x64.box --force;`
* Or point to a URL: `vagrant box add --name xrv64 <REPLACE THIS WITH PUBLIC VAGRANT BOX URL> --force`

![cp, add vagrantfile](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/xrv64_cp_vagrantfile_m.png)


`vagrant up` - this will take some time, possibly over 10 minutes once you see the "Waiting for machine to boot" message. Look for the green "vagrant up" welcome message to confirm the three machines have booted:
	
   ![vagrant up- multi](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/xrv64_vagrant_up_m.png)
   
 Post up message:
   ![post up message](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/xrv64_post_up_m.png)


As with the Single Node setup, you have two ways to access the Vagrant instances:

1. Access the XR Linux shell:

		vagrant ssh rtr1 / vagrant ssh rtr2
        
   ![vagrant ssh- multi](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/xrv64_vagrant_ssh_1_m.png)

2. Access XR Console:

         ssh -p <forwarded port> vagrant@127.0.0.1

   * Replace `<forwarded port>` with the port forwarded for your 22 port of the router you wish to access.

      * You can check which ports were forwarded with the command:

		    vagrant port
      
    * The password is “vagrant”.

   * Repeat these steps for each node.
   
 ![ssh console- multi](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/xrv64_ssh_console_m.png)
