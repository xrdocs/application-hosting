---
published: true
date: '2018-08-06 14:27 -0700'
title: Comprehensive guide to Ansible on IOS-XR
author: Mike Korshunov
excerpt: 'Learn how to utilize Ansible and unleash its power on IOS-XR box. '
position: top
tags:
  - iosxr
  - Ansible
  - Automation
---

{% include toc icon="table" title="Comprehensive guide to Ansible on IOS-XR" %}

{% include base_path %}


Network Automation for the network is crucial nowadays. It changes how network feels and look. This tutorial is going to cover Ansible modules for IOS-XR, some tips and tricks and how to increase performance for Ansible playbooks. 


As per Ansible version 2.6.2, we have following 9 modules for IOS-XR. An excerpt from [Ansible site](https://docs.ansible.com/ansible/2.6/modules/list_of_network_modules.html#iosxr) below:  

- **iosxr_banner** - Manage multiline banners on Cisco IOS XR devices
- **iosxr_command** - Run commands on remote devices running Cisco IOS XR
- **iosxr_config** - Manage Cisco IOS XR configuration sections
- **iosxr_facts** - Collect facts from remote devices running IOS XR
- **iosxr_interface** - Manage Interface on Cisco IOS XR network devices
- **iosxr_logging** - Configuration management of system logging services on network devices
- **iosxr_netconf** - Configures NetConf sub-system service on Cisco IOS-XR devices
- **iosxr_system** - Manage the system attributes on Cisco IOS XR devices
- **iosxr_user** - Manage the aggregate of local users on Cisco IOS XR device


## Prerequisites 

Since Ansible relies on SSH connection to the device, few requirements need to be met.

### k9 security package comitted on the device. 


<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:flamboyant#show install active
Wed Aug  1 18:00:24.162 UTC
Node 0/RP0/CPU0 [RP]
  Boot Partition: xr_lv0
  Active Packages: 9
        ncs5500-xr-6.3.2 version=6.3.2 [Boot image]
        ncs5500-mcast-2.1.0.0-r632
        ncs5500-mpls-2.1.0.0-r632
        ncs5500-mgbl-4.0.0.0-r632
        ncs5500-mpls-te-rsvp-2.2.0.0-r632
        ncs5500-ospf-2.0.0.0-r632
        ncs5500-isis-1.3.0.0-r632
        ncs5500-li-1.0.0.0-r632
        <mark>ncs5500-k9sec-4.1.0.0-r632</mark>

Node 0/0/CPU0 [LC]
  Boot Partition: xr_lcp_lv0
  Active Packages: 9
        ncs5500-xr-6.3.2 version=6.3.2 [Boot image]
        ncs5500-mcast-2.1.0.0-r632
        ncs5500-mpls-2.1.0.0-r632
        ncs5500-mgbl-4.0.0.0-r632
        ncs5500-mpls-te-rsvp-2.2.0.0-r632
        ncs5500-ospf-2.0.0.0-r632
        ncs5500-isis-1.3.0.0-r632
        ncs5500-li-1.0.0.0-r632
        <mark>ncs5500-k9sec-4.1.0.0-r632</mark>

</code>
</pre>
</div>


### RSA key pair is present

RSA key pair needs to be generated. Use the **crypto key generate rsa command** to generate it. You must configure a hostname for the router using the hostname global configuration command.


```

RP/0/RP0/CPU0:flamboyant# crypto key generate rsa keypair-cisco
Wed Aug  1 18:09:23.836 UTC
The name for the keys will be: keypair-cisco
  Choose the size of the key modulus in the range of 512 to 4096 for your General Purpose Keypair. Choosing a key modulus greater than 512 may take a few minutes.

How many bits in the modulus [2048]:
Generating RSA keys ...
Done w/ crypto generate keypair
[OK]

RP/0/RP0/CPU0:flamboyant#

RP/0/RP0/CPU0:flamboyant#! If you want to remove keys from device: 
RP/0/RP0/CPU0:flamboyant#crypto key zeroize  rsa
Wed Aug  1 18:08:37.471 UTC
% Keys to be removed are named keypair-cisco
Do you really want to remove these keys ?? [yes/no]: yes

RP/0/RP0/CPU0:flamboyant#

```

If you want to generate keys without prompt, use following command:

```
RP/0/RP0/CPU0:flamboyant#crypto key generate rsa general-keys modulus
Wed Aug  1 19:31:01.233 UTC
The name for the keys will be: modulus
  Choose the size of the key modulus in the range of 512 to 4096 for your General Purpose Keypair. Choosing a key modulus greater than 512 may take a few minutes.

How many bits in the modulus [2048]: Generating RSA keys ...
Done w/ crypto generate keypair
[OK]

RP/0/RP0/CPU0:flamboyant#

```

### Config for SSH server

Apply the following config on the target device to enable SSH and NETCONF. 

<div class="highlighter-rouge">
<pre class="highlight">
<code>
netconf agent tty
!
netconf-yang agent
 ssh
!
ssh server session-limit 10
ssh server v2
ssh server netconf vrf default
</code>
</pre>
</div>

## Network management specific moments 

Ansible workflow for managing network nodes has its nuances.

Usually, Ansible runs on managed nodes, however, it's not the case for the network modules. Everything stays the same from the user perspective and for accustomed keywords. All the magic happens in the background. Typically network devices lack of Python support (IOS-XR support application hosting concepts. [Read more about it]( {{ base_url }}/application-hosting ). Because of that network module executes locally and CLI/XML instruction sent over to the device.  

One more important aspect for Linux/Unix, configuration files for the system exist in files on hard disks, so the backup for them created in the same directory. It's not the case for the network modules (configuration not stored on disk) and we will see an example in the **iosxr_config** module, where backup folder created on the machine, from which we are running playbooks.  

Additional introduction to Network module - [conditions](https://docs.ansible.com/ansible/2.6/network/user_guide/network_working_with_command_output.html). They allow you to work with output and make value comparison. In conjunction with match parameter, a more sophisticated state can be checked.  


### Communication mechanisms

For network modules, there are 2 main connections: **network_cli** and **netconf**

Previous way to connect to targets was **local**, you can still use old style of declaration in the playbook, however, it's deprecated and will be removed eventually. In official Ansible documentation, you will notice parameter **provider** as an indication that parameter local was used. 

```
---
- name: Configure IOS-XR devices
  hosts: routers
  gather_facts: no
  # connection local instead of network_cli
  connection: local

  tasks:
    - name: collect facts from IOS-XR routers
      iosxr_facts:
        gather_subset:
        - config
        provider: "{{ cli }}"
      register: config

  vars:
    cli:
      host: "{{ ansible_host }}"
      username: "{{ ansible_user }}"
      password: "{{ ansible_ssh_pass }}"

```

If you will run the playbook above with verbose key, in the output you will see 

<div class="highlighter-rouge">
<pre class="highlight">
<code>
$ ansible-playbook head-playbook.yml -i ansible-hosts.ini  --forks 10 -vvv
PLAYBOOK: head-playbook.yml ******************************************************************************
***omitted output***

TASK [include task] **************************************************************************************
task path: /home/cisco/Documents/ansible/head-playbook.yml:16
<172.30.13.70> using connection plugin <mark>network_cli (was local)</mark>
<172.30.13.71> using connection plugin <mark>network_cli (was local)</mark>

***omitted output***

</code>
</pre>
</div>

The choice between network_cli and netconf connection should be made based on documentation [for modules](https://docs.ansible.com/ansible/2.6/modules/list_of_network_modules.html#iosxr) Not all modules support both connection, so first check before usage. 


## Directory structure


All playbooks mentioned in tutorial, available on [Github](https://github.com/Maikor/Ansible-2.5-on-IOS-XR). In Github repo Vagrant folder included, feel free to practice against IOS-XRv image Request it [here](https://xrdocs.io/getting-started/steps-download-iosxr-vagrant). 
{: .notice--info}  

Here is the layout for ansible-playbooks.

```
$ tree ansible 
.
ansible
├── ansible-hosts.ini
├── head-playbook.yml
├── README.md
├── roles
│   ├── get_facts
│   │   └── tasks
│   │       └── main.yml
│   ├── xr_commands
│   │   └── tasks
│   │       └── main.yml
│   └── xr_config
│       ├── backup
│       │   ├── canonball_config.2018-08-06@14:22:09
│       │   └── flamboyant_config.2018-08-06@14:22:09
│       ├── common
│       │   └── router.conf
│       └── tasks
│           └── main.yml
└── xr-passes.yml

9 directories, 12 files
```

Folder content: 

- ansible-hosts.ini contains information about target devices. More on [inventory](https://docs.ansible.com/ansible/2.6/network/getting_started/first_inventory.html);
- head-playbook.yml - main playbook, from which most we will execute tasks;
- roles - Ansible concept to separate list of provisioned apps/configs on the device. In terms of servers think of webserver, database, etc. In networking it's more about services which could be provided via config changes: BGP, VPN, ISIS. In current case, we just demo the ability to separate playbooks into roles; 
- backup directory appears after iosxr_config module triggered with _backup: yes_ parameter. 
- xr-passes.yml - our [Vault]( {{ base_url }}/application-hosting/tutorials/2018-07-24-comprehensive-guide-to-ansible-on-ios-xr/#ansible-vault), described in tutorial. 

## Getting into modules

### Hosts file

We need to define the hosts first. There is a separation of variables from host definition in ini file. In provided example, passwords stored in plain text. To avoid it, [Ansible Vault](https://xrdocs.io/application-hosting/tutorials/2018-08-06-comprehensive-guide-to-ansible-on-ios-xr#ansible-vault) should be used and will be covered later in this tutorial. Another mechanism - [passwordless authentication](https://xrdocs.io/application-hosting/tutorials/2018-08-06-comprehensive-guide-to-ansible-on-ios-xr/#key-based-authentication-to-ios-xr-device), based on keys.


<div class="highlighter-rouge">
<pre class="highlight">
<code>
$ cat ansible-hosts.ini 
[routers]
flamboyant ansible_host=172.30.13.70 ansible_user=root
canonball ansible_host=172.30.13.71  ansible_user=cisco

[routers:vars]
ansible_ssh_pass=cisco
ansible_network_os=iosxr
ansible_port=22
</code>
</pre>
</div>


### NETCONF and Banner modules

As a first example we will run two modules: enable NETCONF on device, set the banner via NETCONF (we just enabled it, why not to utilize it?).

Playbook itself demonstrated below. We will just utilize 2 tasks for now. Name of modules is self-describing. 

For NETCONF module we can provide additionally VRF value, via **netconf_vrf** parameter. Default VRF, if the parameter is omitted - default. 
**State** parameter is responsible for adding or removing NETCONF related knob on the device. **Present** value will enable NETCONF configuration, **Absent** will withdraw config. 

Banner module uses same concept for **State** parameter (present/absent).
Text parameter could start with "|" or ">" as identificator for the multiline string. As per banner itself configuration, you can select would it be **login** or **motd** message. Currently, there is a minor bug with caret return symbol treatment, so please use ">" as a multiline identifier. 

<div class="highlighter-rouge">
<pre class="highlight">
<code>
---
- name: Configure IOS-XR devices
  hosts: routers
  gather_facts: no
  connection: network_cli

  tasks:

    - name: enable netconf service on port 830
      iosxr_netconf:
        listens_on: 830
        state: present

    - name: set welcome banner to device!
      iosxr_banner:
        banner: login
        text: >
          ! Unauthorized access to device:
          {{ inventory_hostname }} restricted.
        #    text: >
        #      "{{ lookup('file', 'raw_banner.txt') }}"
        state: present
      connection: netconf
</code>
</pre>
</div>


Console output after playbook execution. For first time we run in verbose mode, since -vvv specified.  As careful reader may notice, temporary files created directly on the host machine, not on target nodes. This is specific of the network module.  



<div class="highlighter-rouge">
<pre class="highlight">
<code>
$ ansible-playbook head-playbook.yml -i ansible-hosts.ini  --forks 10 -vvv
ansible-playbook 2.6.2
  config file = /home/cisco/.ansible.cfg
  configured module search path = [u'/home/cisco/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/local/lib/python2.7/dist-packages/ansible
  executable location = /usr/local/bin/ansible-playbook
  python version = 2.7.12 (default, Nov 20 2017, 18:23:56) [GCC 5.4.0 20160609]
Using /home/cisco/.ansible.cfg as config file
Parsed /home/cisco/Documents/ansible/ansible-hosts.ini inventory source with ini plugin

PLAYBOOK: head-playbook.yml ******************************************************************************
1 plays in head-playbook.yml
 [WARNING]: Found variable using reserved name: vars_files


PLAY [Configure IOS-XR devices] ****************************************************************************
META: ran handlers

TASK [enable netconf service on port 830] ****************************************************************
task path: /home/cisco/Documents/ansible/head-playbook.yml:9
<172.30.13.70> ESTABLISH LOCAL CONNECTION FOR USER: cisco
<172.30.13.70> EXEC /bin/sh -c '( umask 77 && mkdir -p "` echo /home/cisco/.ansible/tmp/ansible-local-238349mD4bn/ansible-tmp-1533330985.59-76269032799671 `" && echo ansible-tmp-1533330985.59-76269032799671="` echo /home/cisco/.ansible/tmp/ansible-local-238349mD4bn/ansible-tmp-1533330985.59-76269032799671 `" ) && sleep 0'
Using module file /usr/local/lib/python2.7/dist-packages/ansible/modules/network/iosxr/iosxr_netconf.py
Using module file /usr/local/lib/python2.7/dist-packages/ansible/modules/network/iosxr/iosxr_netconf.py
<172.30.13.70> PUT /home/cisco/.ansible/tmp/ansible-local-238349mD4bn/tmpiboupH TO /home/cisco/.ansible/tmp/ansible-local-238349mD4bn/ansible-tmp-1533330985.59-76269032799671/iosxr_netconf.py
<172.30.13.70> EXEC /bin/sh -c 'chmod u+x /home/cisco/.ansible/tmp/ansible-local-238349mD4bn/ansible-tmp-1533330985.59-76269032799671/ /home/cisco/.ansible/tmp/ansible-local-238349mD4bn/ansible-tmp-1533330985.59-76269032799671/iosxr_netconf.py && sleep 0'
<172.30.13.70> EXEC /bin/sh -c 'rm -f -r /home/cisco/.ansible/tmp/ansible-local-238349mD4bn/ansible-tmp-1533330985.59-76269032799671/ > /dev/null 2>&1 && sleep 0'
changed: [flamboyant] => {
    "changed": true,
    "commands": [
        <mark>"netconf-yang agent ssh",
        "ssh server netconf port 830"</mark>
    ],
    "invocation": {
        "module_args": {
            "host": null,
            "listens_on": 830,
            "netconf_port": 830,
            "netconf_vrf": "default",
            "password": null,
            "port": null,
            "provider": null,
            "ssh_keyfile": null,
            "state": "present",
            "timeout": null,
            "username": null
        }
    }
}

TASK [set welcome banner to device!] *********************************************************************
task path: /home/cisco/Documents/ansible/head-playbook.yml:14
<172.30.13.70> ESTABLISH LOCAL CONNECTION FOR USER: cisco
<172.30.13.70> EXEC /bin/sh -c '( umask 77 && mkdir -p "` echo /home/cisco/.ansible/tmp/ansible-local-238349mD4bn/ansible-tmp-1533330997.74-174548583873229 `" && echo ansible-tmp-1533330997.74-174548583873229="` echo /home/cisco/.ansible/tmp/ansible-local-238349mD4bn/ansible-tmp-1533330997.74-174548583873229 `" ) && sleep 0'
Using module file /usr/local/lib/python2.7/dist-packages/ansible/modules/network/iosxr/iosxr_banner.py
<172.30.13.70> PUT /home/cisco/.ansible/tmp/ansible-local-238349mD4bn/tmpECuOQc TO /home/cisco/.ansible/tmp/ansible-local-238349mD4bn/ansible-tmp-1533330997.74-174548583873229/iosxr_banner.py
<172.30.13.70> EXEC /bin/sh -c 'chmod u+x /home/cisco/.ansible/tmp/ansible-local-238349mD4bn/ansible-tmp-1533330997.74-174548583873229/ /home/cisco/.ansible/tmp/ansible-local-238349mD4bn/ansible-tmp-1533330997.74-174548583873229/iosxr_banner.py && sleep 0'
<172.30.13.70> EXEC /bin/sh -c 'https_proxy='"'"''"'"' http_proxy='"'"''"'"' /usr/bin/python /home/cisco/.ansible/tmp/ansible-local-238349mD4bn/ansible-tmp-1533330997.74-174548583873229/iosxr_banner.py && sleep 0'
<172.30.13.70> EXEC /bin/sh -c 'rm -f -r /home/cisco/.ansible/tmp/ansible-local-238349mD4bn/ansible-tmp-1533330997.74-174548583873229/ > /dev/null 2>&1 && sleep 0'
changed: [flamboyant] => {
    "changed": true,
    "invocation": {
        "module_args": {
            "banner": "login",
            "host": null,
            "password": null,
            "port": null,
            "provider": null,
            "ssh_keyfile": null,
            "state": "present",
            <mark>"text": "Unauthorized access to device: flamboyant restricted.\n",</mark>
            "timeout": null,
            "username": null
        }
    },
    "xml": <mark>"<config xmlns:xc=\<br>"urn:ietf:params:xml:ns:netconf:base:1.0\"><banners xmlns=\<br>"http://cisco.com/ns/yang/Cisco-IOS-XR-infra-infra-cfg\"><banner xc:operation=\"merge\"><banner-name>login</banner-name><banner-text><br>! Unauthorized access to device: flamboyant restricted.\n</banner-text></banner></banners></config></mark>"
}
META: ran handlers

PLAY RECAP ***********************************************************************************************
flamboyant                 : ok=2    changed=2    unreachable=0    failed=0


</code>
</pre>
</div>

From the output, we can observe that both changes applied on the device. There are two ways how to check if config properly applied. 

The first approach is a conservative one. Connect to the device and check latest commit changes. Since each task is separate commit, a command issued for last two configuration changes. 

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:flamboyant#show configuration commit changes last 2
Fri Aug  3 21:04:29.739 UTC
Building configuration...
!! IOS XR Configuration version = 6.3.2
banner login Unauthorized access to device: flamboyant restricted.
netconf-yang agent
 ssh
!
end
</code>
</pre>
</div>

More interesting approach to stay in same operations paradigm and use Ansible for config validation: 

<div class="highlighter-rouge">
<pre class="highlight">
<code>

#     1 new task introduced: 
    - name: check if two first tasks 'enable netconf service on port 830' and 'set welcome banner to device!'  successfully applied on the device
      iosxr_command:
        commands:
        - show run | begin netconf
        - show run | include banner
        wait_for:
        - result[0] contains 'netconf-yang agent'
        - result[1] contains 'banner login !'
#        - result[1] contains 'banner login My Not Expected Message'
</code>
</pre>
</div>

Such playbook will pass, and we will be sure, that config presented on the device. If we uncomment last string, the playbook will fail. We can use **match** parameter with value **any** and execution will succeed. The default value is all. Think about them as logical operators AND & OR. If **wait_for** argument included in the task, an output will not be returned until success or number of retries exceeded. 

As you may notice, wait_for used in the last task. More documentation on [this module](https://docs.ansible.com/ansible/2.6/modules/wait_for_module.html) 


<div class="highlighter-rouge">
<pre class="highlight">
<code>

    "stdout_lines": [
        [
            "Building configuration...",
            "netconf-yang agent",
            " ssh",
            "!",
            "ssh server session-limit 10",
            "ssh server v2",
            "ssh server netconf vrf default",
            "end"
        ],
        [
            "Building configuration...",
            "banner login ! Unauthorized access to device: flamboyant restricted."
        ]
    ]

PLAY RECAP ***********************************************************************************************
flamboyant                 : ok=3    changed=0    unreachable=0    failed=0
    
</code>
</pre>
</div>


### Config, Logging and System modules


Next modules to be utilized are iosxr_config, iosxr_logging and iosxr_system. 


iosxr_config doesn't support netconf connection. ⚠️
{: .notice--info}  


To show a more sophisticated flow, we will create a role in our initial playbook. 

File main.yml will include tasks, which need to be executed. router.conf - common configuration for devices.

```
|____xr-passes.yml
|____head-playbook.yml
|____ansible-hosts.ini
|____roles
| |____xr_config
| | |____common
| | | |____router.conf
| | |____tasks
| | | |____main.yml

$ cat roles/xr_config/common/router.conf
router ospf 1
 area 0
  interface HundredGigE0/0/1/0 
  
```

Playbook consists of three tasks. On first task, Ansible will change the device hostname. During second task Ansible applies configuration file on all devices. 
Third task checks that OSPF configuration is properly applied and OSPF process is up and running. 

```
cat roles/xr_config/tasks/main.yml
---
- name: Change hostname
  iosxr_config:
    lines:
      - hostname {{ inventory_hostname }}


- name: Apply config from file
  iosxr_config:
    lines:
      - "{{ lookup('file', '../common/router.conf') }}"
    backup: yes
    

- name: check if OSPF is configured
  iosxr_command:
    commands:
      - show run | i "router ospf"
      - show processes ospf | i "Process state"
    wait_for:
      - result[0] contains "router ospf 1"
      - result[1] contains "Run"
```

Main parameters for the config module are the following:
- _lines & parents_. Lines - ordered set of configs. Parents parameter uniquely identifies, under which block lines should be configured.
- _replace_ values are line (default), block, config. Defines the behavior for task. If set to block and difference exist in lines, whole block will be pushed. 
- _match_  values are line (default), strict, exact, none. Similar parameter to the previous one in terms of comparison. Defines matching algorithm. 
- _backup_ will create the full running configuration in a backup subfolder, before applying new config;
- _before & after_ - will append commands respectively;
- _comment_ - text added to commit description.  Default: configured by iosxr_config;
 

Ready to get some logs out of the device? Logging module is purposed for that.  If you are interested in streaming operation data, check our thorough  [Telemetry tutorials]( {{ base_url }}/telemetry/tutorials )

```
---
- name: Configure IOS-XR devices
  hosts: routers
  gather_facts: no
  connection: network_cli

  tasks:
    - name: configure console logging level
      iosxr_logging:
        dest: console
        level: debugging
        state: present
    - name: configure logging for syslog server host
      iosxr_logging:
        dest: host
        name: 172.30.13.2
        level: critical
        state: present
       
```       


For logging in IOS-XR Guest OS, check [this tutorial]({{ base_url }}/application-hosting/tutorials/2018-05-25-logging-on-ios-xr-guest-os-with-rsyslog-and-elastic-stack/)


iosxr_logging doesn't allow you to specify the port, so you may use config module as a workaround. ⚠️
{: .notice--info}

The third module in the section is system configuration: configure DNS, domain-search, and lookup. State present/absent used for enable/disable configuration piece, like we saw earlier in the tutorial. 

```
---
- name: Configure IOS-XR devices
  hosts: routers
  gather_facts: no
  connection: network_cli

  tasks:
    - name: configure DNS and domain-name (default vrf=default)
      iosxr_system:
        state: present
        domain_name: local.cisco.com
        domain-search:
        - cisco.com
        name_servers:
        # new DNS from CloudFlare, easy to remember ;)
        - 1.1.1.1
        - 8.8.8.8
        - 8.8.4.4
     
```

### Interfaces module

Wonder how to manage interfaces? There is the specific module for interface management. There are 4 states for interface: present(default), absent, up & down. **Up** equal to present + operationally up. **Down** - present + operationally down. 

The aggregate parameter used to configure multiple interfaces in one task. New interface - new line. The first task will enable just one interface, second will enable two interfaces and will set MTU value. 

```
- name: Unshut interface 
  iosxr_interface:
    description: link to RouterXX TenGigE0/0/0/11
    name: TenGigE0/0/0/28
    state: present
        
- name: Configure interfaces using aggregate
  iosxr_interface:
    aggregate:
    - name: TenGigE0/0/0/30
    - name: TenGigE0/0/0/31
    mtu: 512
    state: present

```


### User module

One more module, this time for user management. Password for user provided in the clear text. Public key could be used, but in this case, Python module **base64** required (usually it's included into Python distributions). If **public_key** or **public_key_contents** used and multiple users created, the same key used for every user. 

If you use parameter **purge**, which is boolean, all other users going to be removed from the device, except admin and newly created within the task. 
{: .notice--warning}  



```
- name: set multiple users to group sys-admin
  iosxr_user:
    name: user1
    group: sysadmin
    state: present
    public_key_contents: "{{ lookup('file', '/home/user1/.ssh/id_rsa.pub' }}"


- name: Create multiple users to multiple groups
  iosxr_user:
    aggregate:
      - name: user2
      - name: user3
    configured_password: cisco
    groups:
      - sysadmin
      - root-system
    state: present

# Remove users
- name: user deletion
  iosxr_user:
    aggregate:
      - name: user1
      - name: user2
    state: absent

```



### Get Facts module

Facts is one of the most straightforward module, but also very resource intense because it will prompt device for running configuration. If **gather_subset** supplied, possible values are all, hardware, config, and interfaces. Try to limit usage of _show running-config_ in playbooks to not sacrifice playbook performance. 

```
- name: get config facts and hardware
  iosxr_facts:
    gather_subset:
    - config
    - hardware
  register: hardware

- name: get all facts, except information regarding interfaces, we have the special module for them!
  iosxr_facts:
    gather_subset:
    - all
    - "!interfaces"
```

### Command module

Few examples for this module already provided above.  Important parameters for this module:

- **interval** - default 1 second, sets the timer between retries;
- **retries** - how many attempts Ansible will do before failing on task. Default - 10;
- **wait_for** - example was provided when we enable NETCONF on the system and checked response from the device. Can be used in conjunction with match parameter. 

## Dealing with passwords

### Ansible Vault 

Ansible Vault is a feature to store your secrets and sensitive information in encrypted files, instead of plain text. 

First step to create vault file: 

```
$ ansible-vault create xr-passes.yml
New Vault password:
Confirm New Vault password:
$
```
Populate it with values aka variables, same key-value scheme used: 

```
my_sensitive_pass_vault: cisco_SecUre_P@ss
```

Save file. We can check the encrypted file content: 

```
$ cat xr-passes.yml 
$ANSIBLE_VAULT;1.1;AES256
30303565333538383465393933363636653565636465656438333331303261333866376363373932
6539303964623031383065366135663166356263393937310a316436663938346165376636666631
37623938383264396431326464323632346632636634333966376666353731623530343634373263
6235353763653338370a643737653735396261346264633132383537643137383165346433303937
35386364366566643065306633636361353739363865396166623830643630356535
```

Don't like your Vault password? You can change it. 

```
$ ansible-vault rekey xr-passes.yml
```

Advanced Encryption Standard (AES) is used for default encryption (which is shared-secret based).


<div class="highlighter-rouge">
<pre class="highlight">
<code>
---
- name: Configure IOS-XR devices
  hosts: routers
  gather_facts: no
  connection: local

  roles:
    - get_facts
    - banner_setup
    - xr_config
{% raw %}

  vars:
    # Include vault 
    <mark>vars_files: xr-passes.yml</mark>
    cli:
      host: "{{ inventory_hostname }}"
      username: "{{ ansible_user }}"
      # Password used from Vault 
      <mark>password: "{{ my_sensitive_pass_vault }}"</mark>
{% endraw %}
</code>
</pre>
</div>

To make it possible, you need to include vars_files key into the playbook with vault file value.   

### Key-based authentication to IOS-XR device. 

To establish passwordless authentication on IOS-XR we need to go through multiple steps. The public key should be encoded in base64 format. You can use utility **base64**, part of Linux and macOS distribution, or try to use online tool for encoding, such as [base64decode.org](https://www.base64decode.org/). After copy key to IOS-XR based device and import it. 


Make sure that key-pair generated on the device and create b64 file. 

```
$ ls ~/.ssh/ | grep id_rsa.pub
id_rsa.pub

$ cut -d" " -f2 ~/.ssh/id_rsa.pub | base64 -d > ~/.ssh/id_rsa_pub.b64

$ ls -la ~/.ssh/ | grep b64
-rw-rw-r--  1 cisco cisco   535 Aug  6 10:43 id_rsa_pub.b64

```

Key transfer and import operation: 

```

RP/0/RP0/CPU0:flamboyant#scp cisco@172.30.13.2:/home/cisco/.ssh/id_rsa_pub.b64 /disk0:/id_rsa_pub.b64
Mon Aug  6 17:40:58.958 UTC
Connecting to 172.30.13.2...
Password:
  Transferred 535 Bytes
  535 bytes copied in 0 sec (535000)bytes/sec

RP/0/RP0/CPU0:flamboyant#dir disk0: | i rsa
Mon Aug  6 10:50:14.476 PDT
   56 -rw-r--r-- 1   535 Aug  6 10:41 id_rsa_pub.b64

RP/0/RP0/CPU0:flamboyant#crypto key import authentication rsa disk0:/id_rsa_pub.b64
Mon Aug  6 10:41:03.651 UTC
RP/0/RP0/CPU0:flamboyant#

```

Verification, that key is imported:

```
RP/0/RP0/CPU0:flamboyant#show crypto key authentication rsa
Mon Aug  6 10:51:24.635 PDT
Key label: root
Type     : RSA public key authentication
Size     : 4096
Imported : 10:41:03 PDT Mon Aug 06 2018
Data     :
 30820222 300D0609 2A864886 F70D0101 01050003 82020F00 3082020A 02820201
 00D031B9 C0CD4838 C031D9E8 390C51ED 8B77D3F8 F0637BE3 CB4631C5 5D84A294
 BE475637 8F7CC395 3E4AD022 ABBE538A 5304CD3A EC9F0B19 0876132F 7675B36C
 46ED953D B870F3FB 2EDB9B50 E6C29278 5A48C0B5 66B09AC3 D03A54FB E7F8DE78
 A7733571 660DFED5 FB6D0599 54227601 08924FFD CBB890F7 93DCE02C 13F4FFA2
 E15FF061 9C64E0BF B62CF8B0 C6305613 D714F84F 7DBA3B1D ED93609B 8E8384A8
 EC259CDA EEBBD07E 5931F467 4D86D59A 24B596C7 4AEDE957 FA8866C1 ED2988F5
 7B9945F9 CC308EA3 532A2470 75C8CE23 49C0AA75 A1F03538 BC3DD4DE EACC8150
 6640B368 7D5696A7 15C6D1BA D6534F34 3CD4ED92 A313A8D0 0480A169 4BF9575C
 6BCE836E D72F4E01 E76C94A1 3B35C430 FB6A471B 453B0DE3 ACD28034 2632E111
 192A9CA0 3DBF3410 0E9580C7 E0DE4968 01DB0C43 98254390 FDB43E3E 39429EA2
 9CFA40A5 2D8A89EC 1DA9ED1D 494306D2 96936B1D ABDA1F7C 513B9E89 4E45F1FA
 50B1DB14 A00D4A83 2B72C5EC 4557A975 A76D49D8 AC184BBE 3C75E292 CFE0F032
 2DAE7154 83AE0A21 D4177524 11F33960 56732666 84619C01 BA36E257 93DE4A8B
 B8E1E7F7 67A80F9A 265320F4 949F6151 D67B1B2E BF3F6C61 C98C45CF EE3F2D87
 EE7031D9 AD27C89A 20087789 F711FD69 0957C424 E216E439 51B95831 DCE9008A
 7F02D500 802AEADB 4C7469B9 04E98E1A 4BDC6BC1 C36C191F 31747564 5BC178F6
 CD020301 0001
 
 
```

Try to connect, to make sure, no password required: 

```

ssh root@172.30.13.70

 Unauthorized access to device: flamboyant restricted.

RP/0/RP0/CPU0:flamboyant#
```

Voilà, passwordless authentication works. 

## Performance tips 

### Change strategy for execution

Playbook performance can be increased. In order to achieve this, we need to use strategy plugin and change default value **linear** to **free**. By default, Ansible task execution wait for completion on the host, then goes to another host and after execution on all hosts for the current task complete, a new task started. With **free** strategy, Ansible will create a fork of the process. It wouldn’t wait for execution of task to be completed on all nodes. When task is completed on the node, next task is started on it, without delay. 

It's easy to change strategy in playbook: 

<div class="highlighter-rouge">
<pre class="highlight">
<code>
---
- name: Configure IOS-XR devices
  hosts: routers
  strategy: <mark>free</mark>
  gather_facts: no
  connection: network_cli

  roles:
    - get_facts   
</code>
</pre>
</div>


### Change forks number

Forks - parallel processes spawned by Ansible for communication with remote hosts and task execution. The default amount of processes - 5. Add the following string to *ansible.cfg* to increase number of forks. 

```
# ansible.cfg 
forks = 10
```

Another way to change amount of Forks to specify a value, when you run a playbook: 

```
ansible-playbook head-playbook.yml -i ansible-hosts.ini  --forks 10 
```

## Conclusion 

We cover Ansible modules available today for IOS-XR (particular attention to iosxr_command & iosxr_config modules), prerequisites required for that, created role examples and tweak Ansible performance for faster playbook completion. Start slowly with your automation and increase the complexity as you grow.

Good luck with further automation 🔧
