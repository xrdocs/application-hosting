---
published: true
date: '2016-08-15 10:25 +0300'
title: 'Netmiko and Napalm with IOS-XR: Quick Look'
author: Mike Korshunov
tags:
  - vagrant
  - iosxr
  - Python
excerpt: Python libraries to configure your IOS-XR device
---
{% include toc icon="table" title="IOS-XR: Ansible and Vagrant" %}



## Introduction

To begin with, let's take a look the tools we intend to use:

[Netmiko](https://github.com/ktbyers/netmiko) - multi-vendor ssh tool for device configuration 

[NAPALM](https://github.com/napalm-automation/napalm)- python based automation tool, which provides a common API for different vendor platforms. 

We will use old setup, which consist of devbox (Ubuntu instance) and rtr (IOS-XRv)  [link]({{ site.url }}/tutorials/IOSXR-Ansible).
{: .notice--info}  
![setup](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/mkorshun/netmiko_setup.png){: .align-center}
{: .notice}


We are interested in our interaction with IOS-XRv in particular.


### Module installation

Let's install python modules on devbox:

```shell
sudo pip install netmiko

sudo pip install napalm
```

To start off, we should verify that the setup was successful. 
Run python interpreter:

```shell
vagrant@vagrant-ubuntu-trusty-64:~$ python
Python 2.7.6 (default, Jun 22 2015, 17:58:13)
[GCC 4.8.2] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import netmiko
>>> import napalm
>>>

```

### Example using Netmiko

Create the first file and try to connect to the device. 

```python

from netmiko import ConnectHandler

cisco_ios_xrv = {
    'device_type': 'cisco_xr',
    'ip':   '10.1.1.20',
    'username': 'vagrant',
    'password': 'vagrant',
    'port' : 22,          # optional, defaults to 22
    'secret': 'secret',     # optional, defaults to ''
    'verbose': False,       # optional, defaults to False
}

net_connect = ConnectHandler(**cisco_ios_xrv)

output = net_connect.send_command('show ip int brief')
print(output)

output = net_connect.send_config_set(['hostname my_sweet_rtr', 'commit'])
print(output)

output = net_connect.send_command('show run | b hostname')
print(output)
```

Script output:

```
vagrant@vagrant-ubuntu-trusty-64:~$ python netmiko_tut.py

Fri Jul 15 12:29:07.691 UTC

Interface                      IP-Address      Status          Protocol Vrf-Name
GigabitEthernet0/0/0/0         10.1.1.20       Up              Up       default
MgmtEth0/RP0/CPU0/0            10.0.2.15       Up              Up       default

config term
Fri Jul 15 12:29:09.739 UTC
RP/0/RP0/CPU0:my_sweetest_rtr(config)#hostname my_sweetest_rtr
RP/0/RP0/CPU0:my_sweetest_rtr(config)#commit
Fri Jul 15 12:29:10.332 UTC
end
config term
Fri Jul 15 12:29:12.475 UTC
RP/0/RP0/CPU0:my_sweetest_rtr(config)#show run | include hostname
Fri Jul 15 12:29:13.052 UTC
Building configuration...
hostname my_sweetest_rtr
RP/0/RP0/CPU0:my_sweetest_rtr(config)#
```

Now we can proceed with a more serious example. We will use a separate file with the router configuration. Open this file in python, read the lines individually and make a list of commands from it.

```
vagrant@vagrant-ubuntu-trusty-64:~$ cat tel_conf
telemetry
 encoder json
  policy group FirstGroup
   policy test
    transport tcp
    !
   destination ipv4 10.1.1.10 port 2103
commit
```

Python piece of code to split lines into list with commands:

```python
with open('tel_conf') as f:
    lines = f.read().splitlines()
print lines

tel_out =  net_connect.send_config_set(lines)
print tel_out
```

Run python file: 

```
vagrant@vagrant-ubuntu-trusty-64:~$ python netmiko_tut.py
config term
Thu Jul 14 23:49:25.447 UTC
RP/0/RP0/CPU0:xr(config)#telemetry
RP/0/RP0/CPU0:xr(config-telemetry)# encoder json
RP/0/RP0/CPU0:xr(config-telemetry-json)#  policy group FirstGroup
RP/0/RP0/CPU0:xr(config-policy-group)#   policy test
RP/0/RP0/CPU0:xr(config-policy-group)#    transport tcp
RP/0/RP0/CPU0:xr(config-telemetry-json)#    !
RP/0/RP0/CPU0:xr(config-telemetry-json)#   destination ipv4 10.1.1.10 port 2103
RP/0/RP0/CPU0:xr(config-policy-group)#commit
Thu Jul 14 23:49:26.400 UTC
RP/0/RP0/CPU0:xr(config-policy-group)#
```

Verify that config is here: 

```
RP/0/RP0/CPU0:my_sweet_rtr#show run | begin telemetry
Thu Jul 14 20:58:19.116 UTC
Building configuration...
xml agent ssl
!
xml agent tty
!
telemetry
 encoder json
  policy group FirstGroup
   policy test
   transport tcp
   !
   destination ipv4 10.1.1.10 port 2103
  !
 !
!
end
```

Useful commands:

```
device.send_command('show ip int brief')
```

### Example using NAPALM

File for telemetry configuration will be used. We can change destination port from 2103 to 2109 to try diff command. 


At current moment commands related to configuration management doesn't respond correctly, but informative commands work fine: 

```
vagrant@vagrant-ubuntu-trusty-64:~$ cat napalus.py
from napalm import get_network_driver

driver = get_network_driver('iosxr')

device = driver('10.1.1.20', 'vagrant', 'vagrant')

device.open()
# print device.get_facts() ## doesn't work

print device.get_interfaces()
print ''
print device.get_interfaces_counters()
print ''
print device.get_users()

device.close()
```

Output will look like: 

```
{
    'GigabitEthernet0/0/0/0': {
        'is_enabled': True,
        'description': u '',
        'last_flapped': -1.0,
        'is_up': True,
        'mac_address': u '0800.27b2.5406',
        'speed': 1000
    }
}

{
    'GigabitEthernet0/0/0/0': {
        'tx_multicast_packets': 0,
        'tx_discards': 0,
        'tx_octets': 6929839,
        'tx_errors': 0,
        'rx_octets': 586788,
        'tx_unicast_packets': 10799,
        'rx_errors': 0,
        'tx_broadcast_packets': 0,
        'rx_multicast_packets': 0,
        'rx_broadcast_packets': 3,
        'rx_discards': 0,
        'rx_unicast_packets': 9421
    }
}

{
    u 'vagrant': {
        'password': '',
        'sshkeys': [],
        'level': 15
    }
}
```

For more details and list of available methods: 

[Netmiko github](https://github.com/ktbyers/netmiko)

[NAPALM library](https://github.com/napalm-automation/napalm)

[NAPALM list of commands](https://napalm.readthedocs.io/en/latest/support/index.html)


Good luck with further automation!
