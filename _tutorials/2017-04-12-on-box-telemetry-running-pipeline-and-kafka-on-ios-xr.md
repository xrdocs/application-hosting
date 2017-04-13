---
published: true
date: '2017-04-12 23:53 -0700'
title: 'On-box Telemetry: Running  Pipeline and Kafka on IOS-XR '
author: Akshat Sharma
position: hidden
excerpt: >-
  Running On-box Agents for Model Driven Telemetry on IOS-XR. In this case we
  use pipeline and a standalone Kafka client running inside a docker container
  on XR to carry out local actions based on Telemetry data.
tags:
  - vagrant
  - iosxr
  - cisco
  - docker
  - pipeline
  - telemetry
---


## What is On-Box Telemetry?

If you haven't checked out the great set of tutorials from [Shelly Cadora](http://twitter.com/scadora) and team on the Telemetry page of xrdocs: <https://xrdocs.github.io/telemetry/tutorials>, it's time to dive in.  

Streaming Telemetry in principle is tied to our need to evolve network device monitoring, above and beyond the capabilities that SNMP can provide.

To get started, check out the following blogs:

*  [The Limits of SNMP](http://blogs.cisco.com/sp/the-limits-of-snmp)

*  [Why you should care about Model Driven Telemetry](http://blogs.cisco.com/sp/why-you-should-care-about-model-driven-telemetry)

*  [Introduction to pipeline](http://blogs.cisco.com/sp/introducing-pipeline-a-model-driven-telemetry-collection-service)


The running theme through the above set of blogs is clear: We need a consistent model driven method of exposing operational data from Network devices (read Yang Models: Openconfig, Vendor specific, and IETF)  and **PUSH** the data over industry accepted transport protocols like [GRPC](http://www.grpc.io/) or plain TCP/UDP to external Telemetry receivers. This is where IOS-XR really excels.    

The move from pull (SNMP style) to a push-based model for gathering Telemetry data is crucial to understand. It allows operational data to be collected at higher rates and higher scale (been shown and tested to be nearly 100x more effective than SNMP).  
















