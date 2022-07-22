---
published: false
date: '2022-07-22 09:32 -0700'
title: XR 7.7.1 Feature Release
---
# IOS-XR 7.7.1 

Enter text in [Markdown](http://daringfireball.net/projects/markdown/). Use the toolbar above, or click the **?** button for formatting help.

## Key Technology Updates in 7.7.1: 
Segment Routing, EVPN, BGP, Software Only, RON, Peering, Security, CnBNG

## IOS-XR 7.7.1 Software Availability
IOS-XR 7.7.1 is Generally Available (GA) for release. 8000, ASR9K 64-bit, NCS5500/NCS5700, NCS560, NCS540, XRv9K, and XRd are supported in this release. Whitebox (SW only) is supported in this release. 

## Overview 
- 


## Segment Routing

Cisco Optimization Engine (COE) v4.0 requires optimization in certain capabilities within SR-PCE. XR 7.7.1 features the ability to configure a centralized solution to override PCC-initiated objectives and help the customer deploy constraints as override actions. 

  1.	Improve automation product 
  2.	Help brown-field client
  3.	Remove limitations on platforms 



### SRv6-Services: 
- L2 EVPN QoS Enhancement: Preserve incoming QoS, mark outer
- L3 Services QoS Enhancement

### SR-PCE: Objective override of PCC-Initiated policies
- Several customers (such as Vodafone, Telstra, others) have old devices in the network where, as an example, the old PCC is able to delegate computation of IGP metric policies but does not support a PCC-initiated “low delay” policy.
- In such deployments, a centralized solution (PCE-centric) can be implemented to “override” a PCC-initiated optimization objective (ex: IGP cost) to another objective (ex: delay) and help the customer deploy the intended policy SLA albeit this PCC limitation.

### Crosswork Optimization Engine (COE) 4.0 
- Double COE scale:  
- Enable visualization of he tree-sid use case  (multicast)
- First in industry for customer to visualize their multicast tree live in the network 


