---
published: true
date: '2019-08-20 11:17 -0700'
title: 'Docker Applications on IOS-XR: Useful Tips and Hints'
position: hidden
author: Akshat Sharma
excerpt: >-
  Planning to deploy custom and open-source tools on IOS-XR routers ? Prefer
  Docker as a deployment technique? Network devices are constrained devices with
  limited resources (disk, cpu, mem) and require some extra care in operations
  to ensure the system remains stable. Read this blog to know more and up-level
  your docker on XR skills with some new tips and hints
tags:
  - iosxr
  - cisco
  - linux
  - docker
  - xr-toolbox
  - xr toolbox
  - operations
  - vrf
  - logs
---
{% include toc %}

# Before we Begin

If you're interested in hosting Linux applications on IOS-XR, it would be useful to run through the various blogs in the ["xr-toolbox"](https://xrdocs.io/application-hosting/tags/#xr-toolbox) series. This series guides a user through the application-hosting and packet-io infrastructure on IOS-XR along with hands-on tutorials to run native, LXC or Docker applications on IOS-XR.

Over the course of the last few years, we have seen most application-friendly Network operators look at Docker as a preferred method of application deployment owing to its highly automatable image-build techniques that integrate well into exising or evolving CI/CD workflows.
Further, the [application-hosting and packet-io infrastructure](https://xrdocs.io/application-hosting/blogs/2019-08-19-application-hosting-and-packet-io-on-ios-xr-a-deep-dive/) in IOS-XR enables Docker containers to have complete access to the IOS-XR networking stack for routing and APIs while allowing Linux processes to run unhindered and unmodified on top of the XR Linux kernel.

With a growing focus on Docker apps in the networking industry, this blog looks to highlight some quick and useful tips to run your favorite Dockerized applications on IOS-XR, keeping in mind the resource constraints (disk, mem, cpu) on network devices as well as the peculiarities of network deployments such as High-Availability (Dual-RP) switchover events, vrf-based isolation and more.

Bear in mind that most of the topics below are equally relevant to Native and LXC applications, but we will tackle these types of applications individually in future blogs.

For the sake of this blog we will run through an NCS55xx setup running IOS-XR 6.5.2 on each router. The topology is shown below:

Router 1 and Router2 are NCS5501 devices and therefore not HA capable. However Router 3 is NCS5508 and contains two Route-Processors (RPs) enabling us to play around with some HA-capable Docker application deployment.

Since the app-hosting infrastructure is platform independent functionally, the same workflow can be attempted

## Choosing the distribution

There is no restriction on choosing the 

## Pulling Docker images 

## Routing through mgmt port

## Routing through data ports

## Opening up TCP/UDP sockets

## Exposing XR vrfs to Docker containers

## Managing CPU and Memory limits

## Mounting config files

## Managing On-Box Application logs

## Handling Router Reloads

## Handling RP Switchover (HA)

## Reducing the size of Docker images

## Demo: Open/R as a Docker App on IOS-XR


