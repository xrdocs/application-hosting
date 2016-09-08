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

Clone the gRPC for Python library here: <https://github.com/skkumaravel/iosxr-ez/tree/master/grpc>

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

The file is called grpc_example.py

```python
import json

def main():
    '''
    To not use tls we need to do 2 things. 
    1. Comment the variables creds and options out
    2. Remove creds and options CiscoGRPCClient
    ex: client = CiscoGRPCClient('11.1.1.10', 57777, 10, 'vagrant', 'vagrant')
    '''
    creds = open('ems.pem').read()
    options='ems.cisco.com'
    client = CiscoGRPCClient('11.1.1.10', 57777, 10, 'vagrant', 'vagrant', creds, options)
    #Test 1: Test Get config json requests
    path = '{"Cisco-IOS-XR-ip-static-cfg:router-static": [null]}'
    result = client.getconfig(path)
    print json.dumps(json.loads(result))

if __name__ == '__main__':
    main()
```
We need to change only a few things. We currently don't have TLS activated on our box. So lets get rid of that section in our Python file. Edit the file to make it look like this.

```python
import sys
sys.path.insert(0, '../')
from lib.cisco_grpc_client import CiscoGRPCClient
import json

def main():
    '''
    To not use tls we need to do 2 things.
    1. Comment the variables creds and options out
    2. Remove creds and options CiscoGRPCClient
    ex: client = CiscoGRPCClient('11.1.1.10', 57777, 10, 'vagrant', 'vagrant')
    '''
   # creds = open('ems.pem').read()
   # options='ems.cisco.com'
    client = CiscoGRPCClient('11.1.1.10', 57777, 10, 'vagrant', 'vagrant')
    #Test 1: Test Get config json requests
    path = '{"Cisco-IOS-XR-ip-static-cfg:router-static": [null]}'
    result = client.getconfig(path)
    print json.dumps(json.loads(result))

if __name__ == '__main__':
    main()
```

**Note the fields in the client are the ip address of the router, the port, a timeout, the username, and password**

Now all we have to do is run the file. 

<div class="highlighter-rouge">
<pre class="highlight">
<code>
vagrant@vagrant-ubuntu-trusty-64:~/ios-xr-grpc-python/examples$<mark> python grpc_example.py</mark>
{"Cisco-IOS-XR-ip-static-cfg:router-static": {"default-vrf": {"address-family": {"vrfipv4": {"vrf-unicast": {"vrf-prefixes": {"vrf-prefix": [{"prefix": "0.0.0.0", "vrf-route": {"vrf-next-hop-table": {"vrf-next-hop-interface-name-next-hop-address": [{"interface-name": "MgmtEth0/RP0/CPU0/0", "next-hop-address": "10.0.2.2"}]}}, "prefix-length": 0}]}}}}}}}
</code>
</pre>
</div>


This particular path gave us the static routes associate with router. We can see the information in a yang formatted data.


We are done with this tutorial, feel free to change the path variable and experiment to see what you can do. Some useful links below:

[gRPC getting started](https://github.com/CiscoDevNet/grpc-getting-started)

[Getting Started With OpenConfig](https://github.com/CiscoDevNet/openconfig-getting-started)
