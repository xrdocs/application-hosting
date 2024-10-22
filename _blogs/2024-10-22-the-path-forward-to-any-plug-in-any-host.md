---
published: true
date: '2024-10-22 17:29 +0200'
title: 'The Path Forward to Any Plug in Any Host '
author: Maurizio Gazzola
excerpt: >-
  Blog provides an update about industry directions about how to have different
  hosts supporting different Third Party DCO Pluggables
tags:
  - cisco
  - DCO Pluggable
  - Pluggable Management
---


Until recently, optical systems have been closed and proprietary. They came as a package that included client optics, transponders, a line system, and a management system. In the traditional optical architecture, these components were provided by a single vendor, and interfaces between those functions were closed and proprietary. While the concept of disaggregated or open optical components is not new, technolgy advances have made it the optimal solution in some networksallowing them to be sold separately. This enables providers to assemble a system themselves in the manner they choose.
The integration of Coherent WDM interfaces directly in the Routing Switching layer is providing large benefits to network operators but is also raising the question. “What is the best approach to manage it?”.

Leveraging existing management interface standards between host and pluggable, network operators are requesting full interoperability between any pluggable vendor and any hosts vendor in a seamless way.  

This concept is also called “Any Plug in Any Host” 

## What is the real status of “Any Plug in Any Host”?
OIF, Open ZR+ and OpenROADM provided a solid interoperability base but only on the WDM physical layer.
For host to pluggable management Interface everything is defined by the [Common Management Interface Specification (CMIS)](https://www.oiforum.com/technical-work/hot-topics/management/) standard defined in [OIF](https://www.oiforum.com/). CMIS is focused on addressing four keys areas of functionality to support interoperability: Advertising, Monitoring, Provisioning, and Upgrades.

However, the coherent DWDM optic is significantly more complicated than a short reach client optic, and these standards are evolving to cope with such level of complexity.  The implication is that, even in the case where both the optic and the host platform claim standards compliance, there will still be significant integration effort required on the part of the host.
It has to be noted that CMIS compliance is a key element to shorten the eventual integration effort and some customers are starting asking solutions to test CMIS compliance on modules and hosts.

Additionally, even in the case where the optics and the host appear to work together, there is no guarantee that this will remain true across software releases unless the host vendor is committed to this interoperability. For example, the optics and host may only work with specific brands or models of routers or switches, but they may not be compatible with other routers or networking hardware. Cisco has validated this.

Some examples:
- Module recognition/configuration:
Routers use the Application code to understand if it is ZR or ZR+. Additionally, routers and modules must reference the two evolving standards to interpret the Applications codes. The CMIS standard, which defines the register syntax and organization. The SNIA SFF-8024 standard, which defines the Application ID values.
Until CMIS 5.2 version, standards defined ONLY 15 application codes. While for ZR these were enough for ZR+ they were not. This implies that today ZR+ and complex DSP configurations are NOT standardized. 
- Link flap and host side configuration.
CMIS has several flexible capabilities but NOT all the vendors support the same features:
1. Capability to squelch the host side when media side is in LOS/LOF
2. Capability to have an “hold off timer” before doing squelch.
 
- Behavior of alarms and PM during media side LOS/LOF  condition is not standardized yet.
 
- Loopback capability. There are registers in CMIS that advertise and configure the loopback capability, but the variation between vendor interpretation and expectation regarding the sequence of events is not consistent and sometimes it doesn’t work.

And the list of examples can be longer. Does this mean that there is no hope to have the “Any Plug into Any Host” compatibility?

## One Way to Guarantee Compatibility  
When both the optics and the router are from the same vendor, they are designed to work together and often provide a more robust solution. This means that they have been tested and verified to meet certain performance standards, and they are guaranteed to be compatible(for example hardware/ environmental issues like if an optic draws more power than allowed or doesn’t have sufficient passive cooling). This can help ensure optimal performance and reduce the risk of compatibility issues that could result in reduced network availability or increased downtime. This has been validated during multiple lab and interoperability tests performed in the past two years by Cisco.

## Another Approach: AppSel Code
Recently there has been movement in the right direction:
The official release of [OIF CMIS 5.3](https://www.oiforum.com/wp-content/uploads/OIF-CMIS-05.3.pdf) and publication of OIF White Paper : “[CMIS: Path to Plug and Play](https://www.oiforum.com/wp-content/uploads/OIF-CMIS-Plug-and-Play-01.0.pdf)” 
The whitepaper describes how a host (a router for example) can use the advertised capabilities to configure a module for different applications in a generic manner. 
The proposal is that Host platforms adopting two enhancements:
1. Hosts use module advertising to create a table of applications and 		present that table in human readable form. Utilizing new CMIS Common 		Data Block (CBD) commands to extract the applications details. A 			description of the host/media codes stores in the optics module itself 		which enhances the solution by not requiring the host to know how to 		interpret the hex values
2.	Hosts allow users to provision the module based on the application 		descriptor from the table created in enhancement #1.

Table 1 shows an example of a subset of application code that a host could use to configure the coherent pluggable.

![](https://raw.githubusercontent.com/xrdocs/xrdocs-images/0b6adef7d32a8f25d1f43b5b2c2bb70f1137d768/assets/images/Table%201%20blog%20AnyPlugAnyHost.jpg)
Table 1 - AppSel Code Table

It has to be noted that the Appselcode number is not univocally set for all the vendors. What is standardized is the Media Code and the Host Code but it could happen that AppSel code 1 for vendor A is equivalent to AppSel code 3 for a Vendor B.
Two possible scenarios could be evaluated: 
1.	Host checks the available capability of the coherent pluggable 			reading the above table, and then configures the right operational mode 	to the pluggable setting the AppSel Code in the first column as it 			combines those parameters in a single code.
2.	Host checks the advetised capability of the pluggable reading the 		above table, and then configures the right operational mode to the 			pluggable setting the 2 parameters that univocally identify the 			Operational mode : Media Code and Host Code.

Both options could be valid and have pros and cons in terms of development efficiency but the important aspect is that this change drammatically reduces the dependancy of the pluggable from the Host software capability. At a high level it is required just to dump the table and the set a code to the pluggable.
Of course this solution is not completely solving the interoperability Plug to Host issues as there will always be a custom mode requiring specific implementation. For example, thermal issues that cannot be solved without a full hardware test. However, it provides a clear improvement on the capability to have the host to accept many different coherent pluggables. 
While advancements have been made for multi-vendor interoperability with the [OIF CMIS 5.3](https://www.oiforum.com/wp-content/uploads/OIF-CMIS-05.3.pdf) release and white paper, work will continue in this area and Cisco will be part of it.

