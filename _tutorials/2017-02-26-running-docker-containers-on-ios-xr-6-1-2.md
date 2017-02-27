---
published: true
date: '2017-02-26 14:40 -0800'
title: Running Docker Containers on IOS-XR (6.1.2+)
position: hidden
author: Akshat Sharma
excerpt: >-
  Running Docker containers on IOS-XR running as a Vagrant box and on a physical
  NCS5500
tags:
  - vagrant
  - iosxr
  - NCS5500
  - docker
  - xr toolbox
---


{% include toc icon="table" title="Running Docker Containers on IOS-XR" %}
{% include base_path %}

## Introduction

If you haven't checked out the earlier parts to the XR toolbox Series, then you can do so here:  

>
[XR Toolbox Series]({{ base_path }}/tags/#xr-toolbox)

  
The purpose of this series is simple. Get users started with an IOS-XR setup on their laptop and incrementally enable them to try out the application-hosting infrastructure on IOS-XR.

In this part, we explore how a user can spin up Docker containers on IOS-XR. There are multiple ways to do this and we'll explore each one:  

*  **Dockerhub**: Set up reachability from your router (Virtual or physical) to the internet (or specifically to dockerhub: <https://hub.docker.com>).  

*  **Private "secure" registry**: Just set up reachability to your private registry set up with a certificate obtained from a CA. We won't really tackle this scenario separately in this tutorial - once you know how to gain access to dockerhub, the workflow will be similar.  
  
*  **Private "insecure" registry**: Some users may choose to do this, specially if they're running a local docker registry inside a secured part of their network.  
   
*  **Private "self-signed" registry**: This is more secure than the "insecure" setup, and allows a user to enable TLS.
    
*  **Tarball image/container**:  This is the simplest setup - very similar to LXC deployments. In this case, a user may create and set up a container completely off-box, package it up as an image or a container tar ball, transfer it to the router and then load/import it, before running.  


For each case, we will compare IOS-XR running as a Vagrant box with IOS-XR running on a physical box (NCS5500). They should be identical, except for reachability through their Management ports.






   
   
   
