---
published: false
date: '2022-07-26 09:51 -0500'
title: 'XR Collector Health Monitor: An App Hosting Use Case'
author: Adam Horton
excerpt: >-
  A deep dive into a real-world application of app-hosting and useful insights
  about the process of developing a third-party app.
tags:
  - iosxr
  - cisco
position: hidden
---
{% include toc %}
{% include base_path %}

This blog post is primarily designed to provide additional information about the application used in the [App Hosting with XR AppMgr and Docker (7.5.1+)]({{ base_path }}/application-hosting/tutorials/2022-07-15-app-hosting-with-xr-appmgr-and-docker-7-5-1) tutorial. Check this resource out if you get the chance.
{: .notice--primary}

## Introduction
In this blog post, I will be discussing the motivation behind leveraging the application hosting capabilities of IOS-XR through the lens of a specific use case. The topic of this post is the XR Collector Health Monitor, which is a third-party application I developed to automate the configuration management of a telemetry stream. Over the course of the article, I will describe the process of designing an application specifically for IOS-XR, while highlighting a few important concepts related to application hosting. This post is specifically about the XR Collector Health Monitor, so I will also be briefly touching on some concepts from telemetry and programmability. 
