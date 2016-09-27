---
published: true
date: '2016-09-08 09:18 -0700'
title: gRPC in Python for IOS-XR
author: Karthik Kumaravel
excerpt: gRPC in Python for IOS-XR
---
{% include toc icon="table" title="IOS-XR: gRPC in Python" %}

## Introduction
The goal of this tutorial is to set up gRPC in Python to send gRPC commands to an IOS-XR box. This tutorial assumes that you have gone through the XR Toolbox Series before. If you haven't checked out the earlier parts to the XR toolbox Series, then you can do so here:  

>
[XR Toolbox Series]({{ base_path }}/tags/#xr-toolbox)

## Pre-requisites

Before we begin, let's make sure you've set up your development environment.
If you haven't checked it out, go through the "App-Development Topology" tutorial here:  

>
[XR Toolbox, Part 3: App Development Topology]({{ base_path }}/tutorials/2016-06-06-xr-toolbox-app-development-topology)  
  
Follow the instructions to get your topology up and running as shown below:  

![app dev topo](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/app_dev_topology.png)


If you've reached the end of the above tutorial, you should be able to issue `vagrant status` in the `vagrant-xrdocs/lxc-app-topo-bootstrap` directory to see a rtr (IOS-XR) and a devbox (Ubuntu/trusty) instance running.  

**Ensure you have the latest version of IOS-XRv.**
{: .notice--danger}

## Installing gRPC on the devbox 

First log into the devbox

```shell
vagrant ssh devbox
```

Let's start by installing a few developer tools on the devbox.

```shell
sudo apt-get update
sudo apt-get -y install python-dev python-pip git
```

Now that we have installed the developer tools, lets install gRPC for Python on the box.

```shell
sudo pip install grpcio
```

Thats it! gRPC for Python is installed on the devbox.
{: .notice--success}  

## Cloning the git repo

Now that we have gRPC for Python installed on the devbox. We need to get the bindings associated with IOS-XR. Let's use the library that has these bindings done.

Clone the gRPC for Python library here: <https://github.com/cisco-grpc-connection-libs/ios-xr-grpc-python>

```shell
cd ~/
git clone https://github.com/cisco-grpc-connection-libs/ios-xr-grpc-python.git
cd ios-xr-grpc-python
```

## Understanding the topology

We need to ensure that gRPC is turned on in the devbox and take note of the port.

```shell
ssh vagrant@11.1.1.10
```

**Note the password would be vagrant**
{: .notice--warning}

From here, use `show run` to find check that gRPC is running and find the configured port.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:ios#<mark>show run</mark>            
Wed Sep  7 16:59:19.241 UTC
Building configuration...
!! IOS XR Configuration version = 6.1.1.19I
!! Last configuration change at Wed Sep  7 16:18:14 2016 by UNKNOWN
!
telnet vrf default ipv4 server max-servers 10
username vagrant
 group root-lr
 group cisco-support
 secret 5 $1$0Lrs$AInvgKCO262qZh6hLMfis0
!
tpa
 address-family ipv4
  update-source MgmtEth0/RP0/CPU0/0
 !
!
interface MgmtEth0/RP0/CPU0/0
 ipv4 address dhcp
!
interface GigabitEthernet0/0/0/0
 ipv4 address 11.1.1.10 255.255.255.0
!
router static
 address-family ipv4 unicast
  0.0.0.0/0 MgmtEth0/RP0/CPU0/0 10.0.2.2
 !
!
ssh server v2
ssh server vrf default
<mark>grpc
port 57777</mark>
!
end

RP/0/RP0/CPU0:ios#
</code>
</pre>
</div>

We can see that gRPC is turned on, and is on port 57777.
{: .notice--info}

Let's exit out and continue getting our gRPC client working.

```
RP/0/RP0/CPU0:ios#exit
Connection to 11.1.1.10 closed.
vagrant@vagrant-ubuntu-trusty-64:~/ios-xr-grpc-python$
```

## Creating a Python gRPC Call

There is an example call already created. We can find it in the examples folder.

```shell
cd examples
```
Lets first understand the json file we are going to use. The json below is based off the yang model provided by Cisco: <https://github.com/YangModels/yang/blob/master/vendor/cisco/xr/611/Cisco-IOS-XR-ipv4-bgp-cfg.yang>. 

You can walk through the heirachy using pyang, and create a json model similar to the example below. <https://github.com/mbj4668/pyang/wiki/TreeOutput>

This json model is for a bgp cfg. We can see that it is defining a bgp instance and a single neighbor.

```shell
vagrant@vagrant-ubuntu-trusty-64:/vagrant/ios-xr-grpc-python/examples$ cat snips/bgp_start.json
{
 "Cisco-IOS-XR-ipv4-bgp-cfg:bgp": {
  "instance": [
   {
    "instance-name": "default",
    "instance-as": [
     {
      "as": 0,
      "four-byte-as": [
       {
        "as": 65400,
        "bgp-running": [
         null
        ],
        "default-vrf": {
         "global": {
          "router-id": "11.1.1.10",
          "global-afs": {
           "global-af": [
            {
             "af-name": "ipv4-unicast",
             "enable": [
              null
             ],
             "sourced-networks": {
              "sourced-network": [
               {
                "network-addr": "11.1.1.0",
                "network-prefix": 24
               }
              ]
             }
            }
           ]
          }
         },
         "bgp-entity": {
          "neighbors": {
           "neighbor": [
            {
             "neighbor-address": "11.1.1.20",
             "remote-as": {
              "as-xx": 0,
              "as-yy": 65450
             },
             "neighbor-afs": {
              "neighbor-af": [
               {
                "af-name": "ipv4-unicast",
                "activate": [
                 null
                ],
                "next-hop-self": true
               }
              ]
             }
            }
           ]
          }
         }
        }
       }
      ]
     }
    ]
   }
  ]
 }
}
```

Now let's use the client, there is a python example that uses the client called grpc_cfg.py. There are 5 helper functions. The init creates a gRPC client object, then there are 4 other functions, each using a different method in gRPC for IOS-XR. They read in a json file and pass it to the gRPC server for configs.

```python
'''
Note:
This is an example to show replace and merge configs work with a get command.
The example is using XRdocs vagrant topology for all the configurations
'''

import sys
sys.path.insert(0, '../')
from lib.cisco_grpc_client import CiscoGRPCClient
import json
from time import sleep

class Example:
    def __init__(self):
        self.client = CiscoGRPCClient('11.1.1.10', 57777, 10, 'vagrant', 'vagrant')
    def get(self):
        path = '{"Cisco-IOS-XR-ipv4-bgp-cfg:bgp": [null]}'
        result = self.client.getconfig(path)
        print result

    def replace(self):
        path = open('snips/bgp_start.json').read()
        result = self.client.replaceconfig(path)
        print result # If this is sucessful, then there should be no errors.

    def merge(self):
        path = open('snips/bgp_merge.json').read()
        result = self.client.mergeconfig(path)
        print result # If this is sucessful, then there should be no errors.

    def delete(self):
        path = open('snips/bgp_start.json').read()
        result = self.client.deleteconfig(path)
        print result # If this is sucessful, then there should be no errors.

```

**Note the fields in the client are the ip address of the router, the port, a timeout, the username, and password**

Let's start with using the python interpreter to import the Example class and initialize it.

```shell
vagrant@vagrant-ubuntu-trusty-64:/vagrant/ios-xr-grpc-python/examples$ python
Python 2.7.6 (default, Jun 22 2015, 17:58:13)
[GCC 4.8.2] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> from grpc_cfg import Example
>>> example = Example()
```

We are going to start with a replace config, to add a base BGP config using the json file we looked at earlier.

```shell
>>> example.replace()
```

A blank response means there are no errors.
Now let's use a get request to see what is on the router.

```shell
>>> example.get()
```

If this worked correctly you should see the json file we looked at, and the resopnse from the get should be identical 
Now let's use a merge request to add another neighbor with the second json file.

```shell
>>> example.merge()

>>> example.get()
```

The resulting config should be the first config plus the second, or in other words there are 2 neighbors defined. 

At this point you can see how gRPC is easy to use to get, replace, and merge configs. You can even remove a config completely using delete.

We are done with this tutorial, feel free to change the path variable and experiment to see what you can do. Some useful links below:

[gRPC getting started](https://github.com/CiscoDevNet/grpc-getting-started)

[Getting Started With OpenConfig](https://github.com/CiscoDevNet/openconfig-getting-started)
