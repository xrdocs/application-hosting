---
published: true
date: '2016-06-22 19:41 +0530'
title: 'Day 0: Getting Started with Application Hosting'
author: Anne Sequeira
tags:
  - iosxr
---
## What you need to know to begin using IOS XR 64-bit

Before you begin, it is essential that you are familiar with the IOS XR 64-bit Linux Shell.

The 64-bit IOS XR uses a hypervisor to offer you the provision of creating your own build environment, or spinning up your own Linux Container (LXC) for hosting applications. XR uses a Windriver Yocto distribution and is designed to work well with embedded systems.

When you start using XR, you are offered three distinct options for hosting applications:

1. Building your own environment natively in the control plane of XR.
2. Spinning up your own container on the XR.
3. Using a docker.
