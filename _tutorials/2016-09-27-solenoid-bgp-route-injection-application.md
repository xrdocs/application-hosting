---
published: false
date: '2016-09-27 10:40 -0700'
title: 'Solenoid: BGP Route Injection Application'
author: Lisa Roach
excerpt: >-
  This tutorial is meant to get the Solenoid application up and running on your
  laptop.
tags:
  - iosxr
  - BGP
position: hidden
---
{% include base_path %}

## Introduction

If you haven’t checked out the XR toolbox Series, then you can do so here:

[XR Toolbox Series]({{ base_path }}application-hosting/tags/#xr-toolbox)

This series is meant to help a beginner get started with application-hosting on IOS-XR.

In this tutorial we intend to utilize almost all the techniques learnt in the above series to inject third-party BGP routes into Cisco's RIB table:
	
    -hello{: .text-center}

## Pre-requisites

Make sure you have [Vagrant](https://www.vagrantup.com/downloads.html) and [Virtualbox](https://www.virtualbox.org/wiki/Downloads) installed on your system.

The system must have  4.5GB of space available. The topology includes on IOS-XRv router (3.5G RAM) and an Ubuntu instance (501MB RAM).

Go through the Vagrant quick-start tutorial, if you haven’t already, to learn how to use Vagrant with IOS-XR: [IOS-XR vagrant quick-start]({{ base_path }}/application-hosting/tutorials/iosxr-vagrant-quickstart)

It would be beneficial for the user to go through the [XR Toolbox Series]({{ base_path }}/application-hosting/tags/#xr-toolbox). But it is not a hard requirement. Following the steps in this tutorial should work out just fine for this demo.

Once you have everything set up, you should be able to see the IOS-XRv vagrant box in the `vagrant box list` command:

<div class="highlighter-rouge">
<pre class="highlight">
<code>

lisroach@LISROACH-M-J0AY ~/W/X/S/vagrant> <mark>vagrant box list</mark>
IOS XRv         (virtualbox, 0)

</code>
</pre>
</div>


### Understand the Topology

- **devbox**: The Ubuntu Vagrant box on the right. This is running exaBGP and is peered with the xrv router to its left. exaBGP is sending out 3 BGP neighbor announcements and withdrawals about every 2 seconds.

- **xrv**: The router on the left. This router is running a gRPC server, and is not running any version of Cisco's BGP. It has an Ubuntu LXC as it's third-party container instead, which is running exaBGP and the Solenoid application. 

- **solenoid container**: The Ubuntu LXC that is running on the xrv. exaBGP is peered with the devbox and hears all of its neighbor's announcements and withdrawals. Upon receiving a neighborhood update, exaBGP runs Solenoid, which uses a gRPC client and YANG models to send the new route (or withdrawn route) to the RIB table in the IOS-XR.

### Clone the git repo

The entire environment can be replicated on any environment running vagrant, provided there is at least 4.5GB of space available. The topology includes on IOS-XRv router (3.5G RAM) and an Ubuntu instance (501MB RAM).  

Clone the Solenoid code from here: [https://github.com/ios-xr/Solenoid.git](https://github.com/ios-xr/Solenoid.git)

```
lisroach@LISROACH-M-J0AY ~/Workspace> git clone https://github.com/ios-xr/Solenoid.git
Cloning into 'Solenoid'...
remote: Counting objects: 1539, done.
remote: Compressing objects: 100% (623/623), done.
remote: Total 1539 (delta 884), reused 1508 (delta 866), pack-reused 0
Receiving objects: 100% (1539/1539), 713.76 KiB | 317.00 KiB/s, done.
Resolving deltas: 100% (884/884), done.
Checking connectivity... done.
lisroach@LISROACH-M-J0AY ~/Workspace>
```

### Spin up the Ubuntu devbox:

Move into the vagrant directory and launch only the devbox node:
 ```
 cd Solenoid/vagrant
 vagrant up devbox
 ```
 
### Create Solenoid LXC tar ball

Enter the devbox:
	
`vagrant ssh devbox`

Install LXC:

```
sudo apt-get update
sudo apt -y install lxc
```

Create the Solenoid LXC template:

`sudo lxc-create -t ubuntu --name solenoid`	 

Start the container. You will be dropped into the console once boot is complete.

`sudo lxc-start --name solenoid`

Username: ubuntu

Password: ubuntu

 

### Install Application dependencies inside LXC

Install Solenoid, exaBGP and all of their dependencies inside the container. We’ll also install git, screen, curl, and pip to fetch the repositories and install the requirements. Initiate the following commands:

```
sudo apt-get -y install git curl screen python-dev python-setuptools
sudo easy_install pip
sudo pip install virtualenv exabgp
```

### Fetch the Application code from github

Now, download Solenoid from github. Using the Solenoid directory, we can install the remaining dependencies with the setup.py installation script. 

```
git clone https://github.com/ios-xr/Solenoid.git
cd Solenoid
virtualenv venv
source venv/bin/activate
pip install grpcio
python setup.py install 
```

Perfect! Now all of our dependencies have been installed. 

### Configure Solenoid and exaBGP

Create a solenoid.config file with the following data (in the Solenoid/ top-level directory):

```
[default]
transport: gRPC
ip: 11.1.1.10
port: 57777
username: vagrant
password: vagrant
```

That is all we need to configure Solenoid to work for your system. Now we need to add a configuration file for exaBGP. Navigate to your **home directory**, and add a file named **router.ini**:

```
group demo {
        router-id 11.1.1.10;

        process monitor-neighbors {
            encoder json;
            receive {
                parsed;
                updates;
                neighbor-changes;
            }
            run /usr/bin/env python /home/ubuntu/Solenoid/solenoid/edit_rib.py -f '/home/ubuntu/Solenoid/filter.txt';
        }

        neighbor 11.1.1.20 {
                local-address 11.1.1.10;
                local-as 65000;
                peer-as 65000;
        }
}
```

### Change the SSH port inside the container

When we deploy the container to IOS-XR, we will share XR’s network namespace. Since IOS-XR already uses up port 22 and port 57722 for its own purposes, we need to pick some other port for our container.

**P.S. If you check the Vagrantfile, we intend to expose port 58822 to the user’s laptop directly, on xrv.**

Let’s change the SSH port to 58822:	

`sudo sed -i s/Port\ 22/Port\ 58822/ /etc/ssh/sshd_config`
 
Check that your port was updated successfully:

```
~$ cat /etc/ssh/sshd_config | grep Port
Port 58822
```

We’re good!


### Package up the LXC

Shutdown the container:
`sudo shutdown -h now`

You’re back on the devbox.

Become root and package up your tar ball:

```
sudo -s

cd /var/lib/lxc/solenoid/rootfs/

tar -czvf /vagrant/solenoid.tgz *
```

See what we did there? We packaged up the container tar ball as pathchecker_rootfs.tar.gz under /vagrant directory. Why is this important?

Well, Vagrant also automatically shares a certain directory with your laptop (for most types of guest operating systems). So the /vagrant is automatically mapped to the directory in which you launched your vagrant instance. To check this, let’s get out of our vagrant instance and issue an ls in your launch directory:
```
~$ exit
~$ exit
> pwd
~/Workspace/Solenoid/vagrant
>ls -la solenoid.tgz
-rw-r--r--  1 lisroach  staff  252417007 Aug 2 11:27 solenoid.tgz
>
```

Now you have your solenoid tar ball! You can complete launching your vagrant instances with `ENV=local vagrant up`. 




