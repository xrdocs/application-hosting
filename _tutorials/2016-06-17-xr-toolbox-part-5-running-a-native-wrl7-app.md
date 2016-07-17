---
published: true
date: '2016-06-17 17:47 -0700'
title: 'XR toolbox, Part 5: Running a native WRL7 app'
author: Akshat Sharma
excerpt: 'Build, Install and launch a native app in IOS-XR (WRL7 RPM or binary)'
position: hidden
tags:
  - vagrant
  - iosxr
  - cisco
  - linux
  - wrl7
  - rpm
---

{% include toc icon="table" title="Launching a Container App" %}
{% include base_path %}
  
Check out Part 4 of the XR toolbox series: [Bring your own Container (LXC) App]({{ base_path }}/tutorials/2016-06-16-xr-toolbox-part-4-bring-your-own-container-lxc-app/).

## Introduction

If you haven't checked out the earlier parts to the XR toolbox Series, then you can do so here:  

>
[XR Toolbox Series]({{ base_path }}/tags/#xr-toolbox)

  
The purpose of this series is simple. Get users started with an IOS-XR setup on their laptop and incrementally enable them to try out the application-hosting infrastructure on IOS-XR.

In this part, we explore how a user can build and deploy native WRL7 RPMs that they may host in the same process space as XR.  


## What's a native app?  

I go into some detail with respect to the IOS-XR application hosting architecture in the following blog:
  
>
[XR app-hosting infrastructure: Quick Look]({{ base_path }}/blogs/2016-06-28-xr-app-hosting-architecture-quick-look/)  
  
  
For reference, a part of the architecture is shown below. We focus on the green container in the figure from the original blog:  

<a href="https://xrdocs.github.io/xrdocs-images/assets/images/xr-control-plane-lxc.png"><img src="https://xrdocs.github.io/xrdocs-images/assets/images/xr-control-plane-lxc.png" width="550" height="550" class="align-center" /></a>

