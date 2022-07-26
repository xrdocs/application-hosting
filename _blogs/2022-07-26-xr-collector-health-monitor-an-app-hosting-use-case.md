---
published: true
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

This blog post is primarily designed to provide additional information about the application used in the [App Hosting with XR AppMgr and Docker (7.5.1+)]({{ base_path }}/tutorials/2022-07-15-app-hosting-with-xr-appmgr-and-docker-7-5-1) tutorial. Check this resource out if you get the chance.
{: .notice--primary}

## Introduction
In this blog post, I will be discussing the motivation behind leveraging the application hosting capabilities of IOS-XR through the lens of a specific use case. The topic of this post is the XR Collector Health Monitor, which is a third-party application I developed to automate the configuration management of a telemetry stream. Over the course of the article, I will describe the process of designing an application specifically for IOS-XR, while highlighting a few important concepts related to application hosting. This post is specifically about the XR Collector Health Monitor, so I will also be briefly touching on some concepts from telemetry and programmability. 

## Telemetry Snapshot
Before I begin discussing the XR-Collector-Health-Monitor in detail, I would like to get all readers up to speed with telemetry. If you are an expert on telemetry, feel free to skip this section. For those of you who are not, telemetry is the process by which data about the router, its performance, and other network metrics are transferred to an external collector where they can be monitored by network engineers and used to guide decisions about the network. A standard collector utilizes Telegraf to receive the incoming data, InfluxDB or Prometheus to store the data (Time-Series Database), and Grafana to display the statistics in a user-friendly format:

![Telemetry Stack](https://xrdocs.github.io/xrdocs-images/assets/blog-images/telemetry-stack.jpg)

Telemetry is **model-driven**, meaning that information transferred in a telemetry stream is described by YANG models. A telemetry stream is made up of three components:

![Telemetry Fundamentals](https://xrdocs.github.io/xrdocs-images/assets/blog-images/telemetry-fundamentals.jpg)

That was a very quick overview of telemetry on IOS-XR. Hopefully, it will be enough to clarify the remainder of this article. If you are curious about telemetry and want to read more, there are plenty of fantastic articles over on the [xrdocs telemetry page](https://xrdocs.io/telemetry).

## What Is the Issue at Hand?
Telemetry streams can become inactive for many reasons, but one of particular concern is in the event of a collector failure. Without alert systems in place, a collector failure can go unnoticed, losing valuable telemetry data until a network engineer notices that the stream is no longer active. Even after the failure is noticed, a network engineer must manually reconfigure all of the routers that were previously streaming to that collector, accruing operational debt. The XR Collector Health Monitor addresses this situation by automating the configuration of the telemetry stream. When a collector fails, the on-box XR Collector Health Monitor application reconfigures the router to resume streaming to a secondary, backup collector. The remainder of this article will focus on the benefits of application hosting, and how they were applied in the case of the XR Collector Health Monitor.

## Why Use Application Hosting?
Before I discuss the XR Collector Health Monitor, I should probably mention why application hosting is well-suited to address this problem, so you know when application hosting is appropriate for your use case. The status of the telemetry stream can only directly be monitored from the collector or the router. It makes sense, then, that we would want to have the router manage its own telemetry stream without the intervention of an external agent. Naturally, this will speed up the response time in the event of a failure and eases the burden on network engineers by automating the reconfiguration process. With the recent release of the XR AppMgr in IOS XR 7.5.1, creating a custom application to perform all of the necessary operations to monitor and control the telemetry stream from the box is easier than ever. The XR AppMgr enables you to run and manage Docker containers directly from the XR Control Plane that have access to APIs at all levels of IOS XR ([read more about XR AppMgr]({{ base_path }}/tutorials/2022-07-15-app-hosting-with-xr-appmgr-and-docker-7-5-1/#introduction)). For the XR Collector Health Monitor, access to the model-driven telemetry APIs is critical to monitor the status of the telemetry stream and reconfigure telemetry on the box when a failure is detected. Because of the rich set of APIs that meet the requirements of this project, along with added simplification of running program directly on-box, application hosting with XR AppMgr was the right choice for this solution

## How Does it Work?
If you want to know how to set up an application for app hosting on XR AppMgr, check out my [tutorial]({{ base_path }}/tutorials/2022-07-15-app-hosting-with-xr-appmgr-and-docker-7-5-1/#registering-and-activating-your-application) on the subject. 
{: .notice--info}

### Architecture
![App-Hosting IOSXR](https://xrdocs.github.io/xrdocs-images/assets/blog-images/app-hosting-iosxr.jpg)

The XR Collector Health Monitor is run as a docker container managed by IOS-XR AppMgr directly on top of the host hypervisor because the docker daemon is run on the host kernel. This means that containerized applications are entirely separate from the LXC that runs IOS-XR. This is a lot of information to take in all at once, so let’s break down those statements and the diagram further. First, IOS-XR exists within a Linux container running a Wind River Linux 7 distro. This is the Linux shell you enter when you use the **bash** command from the XR CLI. All other XR CLI commands are executed within the XR Control Plane, which handles the routing-related activities. Third-party applications (XR Collector Health Monitor in this case) are installed and managed by XR AppMgr, which is within the XR Control Plane, but are run by the Docker daemon. Because of the separation between IOS-XR and third-party containers, we have to employ slightly more advanced concepts to enable communication between them, which will be covered in the next section.

For further details on the application hosting architecture of IOS-XR, check out [this blog post]({{ base_path }}/blogs/2019-08-19-application-hosting-and-packet-io-on-ios-xr-a-deep-dive/#understanding-app-hosting-on-ios-xr).
{: .notice--info}

### Communication
Because third-party applications have no direct means of accessing the data at the heart of IOS-XR, they must manipulate that data through communication protocols (NETCONF or gRPC). In my case, I chose gRPC/gNMI because of its smaller, more efficient messages. gRPC has the additional benefit of making it simple to encrypt data with TLS if you are worried about the security of communications on your network. In Python, there is a library, [pyGNMI](https://github.com/akarneliuk/pygnmi), that handles creating a gRPC/gNMI client which I leveraged in the XR Collector Health Monitor. You can view my source code [here](https://github.com/adhorton-cisco/xr-collector-health-monitor/blob/main/src/gnmi_config.py). When you configure your application, make sure you use the docker run option “\-\-network host” to share the host networking namespace with the container, so the container can talk to the gRPC/NETCONF server in IOS-XR at address 127.0.0.1.  

In IOS-XR, operational and configuration data is accessible via a common format, YANG models. YANG models have been extensively covered in other tutorials, so I will send you to the [programmability](https://xrdocs.io/programmability) or [telemetry](https://xrdocs.io/telemetry) pages to read more if you are interested. The relevant YANG model for the XR Collector Health Monitor that describes the configuration of model-driven telemetry is **Cisco-IOS-XR-telemetry-model-driven-cfg**. The application changes the telemetry stream destination over the lifetime of the program by managing subscriptions through this YANG model.

### Program Logic
This section will discuss the brains of the XR Collector Health Monitor, and how it makes the decision to divert the telemetry stream to the backup collector. The first piece of information to realize, is that the Collector Health Monitor only knows information about the network that it finds in the config file. This is the location where primary and backup collectors are specified as well as information about their telemetry stream formats.  

Telemetry data is streamed at regular intervals, so the Collector Health Monitor only has to check the status of the telemetry stream immediately prior to the data being sent. Luckily, there is a YANG model that describes the status of a telemetry subscription, **Cisco-IOS-XR-telemetry-model-driven-oper**. Should the telemetry stream to the primary collector fail, the Collector Health Monitor reconfigures a new subscription to the destination-group of the backup collector. When the primary collector comes back online, its subscription reactivates automatically (handled by IOS-XR), and the Collector Health Monitor knows to safely remove the subscription to the backup.  

You may also optionally choose to set up TLS, either for communication between the IOS-XR and the Collector Health Monitor, or for the telemetry stream.  

This [tutorial](https://xrdocs.io/telemetry/tutorials/2017-05-08-pipeline-with-grpc/#grpc-dialout-with-tls) describes how to prepare the certificates for telemetry streaming with TLS. If specified in the config file, the Collector Health Monitor will manage the router configuration while the user must place the certificates are in the correct directory.  

You may also configure TLS between the Collector Health Monitor and IOS-XR by configuring gRPC with TLS on the router and copying the certificate in /misc/config/grpc into the directory with the config.yaml file. Although this is nearly useless when the Collector Health Monitor is being run on-box, it can be useful if the Collector Health Monitor is managing a router from an external host (Yes! It can do that too), and you want to ensure that configuration and operational data is private on the network.
