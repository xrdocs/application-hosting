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

*  **Pull docker images from dockerhub**: Set up reachability from your router (Virtual or physical) to the internet (or specifically to dockerhub: <https://hub.docker.com>).  

*  **Pull docker images from a local registry**:  

*  **Spin up a docker image from a tarball image**:  



