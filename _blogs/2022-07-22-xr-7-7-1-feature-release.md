---
published: false
date: '2022-07-22 09:32 -0700'
title: XR 7.7.1 Feature Release
---
# IOS-XR 7.7.1 


## Key Technology Updates in 7.7.1: 
Segment Routing, EVPN, BGP, Software Only, RON, Peering, and Security

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
- L2 EVPN QoS Enhancement
- L3 Services QoS Enhancement

### SR-PCE: Objective override of PCC-Initiated policies
- Several customers (such as Vodafone, Telstra, others) have old devices in the network where, as an example, the old PCC is able to delegate computation of IGP metric policies but does not support a PCC-initiated “low delay” policy.
- In such deployments, a centralized solution (PCE-centric) can be implemented to “override” a PCC-initiated optimization objective (ex: IGP cost) to another objective (ex: delay) and help the customer deploy the intended policy SLA albeit this PCC limitation.

### Crosswork Optimization Engine (COE) 4.0 
- Enable visualization of tree-sid use cases in multicast
- First in industry for customer to visualize their multicast tree live in the network 
- Double COE scale


## EVPN 

## X-Leaf

## Security

## Coming Soon

# IOS-XR 7.7.1 Software PIDs

#### Ordering Information ASR 9000: 
|PID |Description|
|---|---|
| XR-A9K-X64-07.7  |  Cisco IOS XR IP/MPLS Core Software - IOS-XR 64 Bit |
| XR-A9K-X64K9-07.7  | Cisco IOS XR IP/MPLS Core Software 3DES - IOS-XR 64 Bit  |





	
