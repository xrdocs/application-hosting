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


{% include toc icon="table" title="Running Pipeline and Kafka on IOS-XR" %}
{% include base_path %}

## Steaming Telemetry

If you haven't checked out the great set of tutorials from [Shelly Cadora](http://twitter.com/scadora) and team on the Telemetry page of xrdocs: <https://xrdocs.github.io/telemetry/tutorials>, it's time to dive in.  

Streaming Telemetry in principle is tied to our need to evolve network device monitoring, above and beyond the capabilities that SNMP can provide.

To get started, check out the following blogs:

*  [The Limits of SNMP](http://blogs.cisco.com/sp/the-limits-of-snmp)

*  [Why you should care about Model Driven Telemetry](http://blogs.cisco.com/sp/why-you-should-care-about-model-driven-telemetry)

*  [Introduction to pipeline](http://blogs.cisco.com/sp/introducing-pipeline-a-model-driven-telemetry-collection-service)


The running theme through the above set of blogs is clear: We need a consistent model driven method of exposing operational data from Network devices (read Yang Models: Openconfig, Vendor specific, and IETF)  and **PUSH** the data over industry accepted transport protocols like [GRPC](http://www.grpc.io/) or plain TCP/UDP to external Telemetry receivers. This is where IOS-XR really excels.    

The move from pull (SNMP style) to a push-based model for gathering Telemetry data is crucial to understand. It allows operational data to be collected at higher rates and higher scale (been shown and tested to be nearly 100x more effective than SNMP).  

Consequently, there is greater focus on tools that can help consume this large amount of data off-box. There are a variety of tools (opensource and otherwise) available for big data consumption:  Apache Kafka, Prometheus, influxDB stack, SignalFX etc.   
  
A tool we recently open-sourced in this space with complete support for Model Driven Telemetry on IOS-XR (6.0.1+) that, as the name suggests, serves as a pipe/conduit between IOS-XR (over TCP, UDP or GRPC) on the input side and a whole host of tools (Kafka, influxDB, prometheus etc.) on the output side is called **Pipeline**. You can find it on [Github](https://github.com/cisco/bigmuddy-network-telemetry-pipeline). Find more about it [here](http://blogs.cisco.com/sp/introducing-pipeline-a-model-driven-telemetry-collection-service) and [here](https://xrdocs.github.io/telemetry/tutorials/2016-10-03-pipeline-to-text-tutorial/).  




## What is on-box Telemetry?


There is no one-size-fits-all technique for monitoring and managing network devices. There are a lot of network operators that will follow the typical approach: Set up the Model Driven Telemetry on IOS-XR and stream operational data to an external receiver or set of receivers. This is shown below. Pipeline as mentioned earlier, is used as a conduit to a set of tools like Kafka,prometheus etc.


[![typical Pipeline deployment](https://xrdocs.github.io/xrdocs-images/assets/images/deploy_pipeline.png)](https://xrdocs.github.io/xrdocs-images/assets/images/deploy_pipeline.png)


However, quite a few of our users have come and asked us if it's possible to have a telemetry receiver run on the router inside a container (lxc or docker) so that applications running locally inside the container can take actions based on Telemetry data.
  
  
This would remove the need to set up external services to process large amounts of data from all routers at the same time, but comes with its own concerns of managing the disk space on each individual router. As always, this is a user's operational decision.


### Docker container to host Pipeline + Kafka

In this tutorial, we look at using a Docker container to host Pipeline and Kafka (with zookeper) as a Telemetry receiver. Further a simple Kafka consumer is written in python to interact with Kafka and take some sample action on a Telemetry data point.

If you haven't had a chance to learn how we enable hosting for Docker containers on IOS-XR platforms and how we set up routing capabilities within the container, take a look at the following section of our detailed Docker guide for IOS-XR:  

>[Understanding Docker Setup on IOS-XR platforms](https://xrdocs.github.io/application-hosting/tutorials/2017-02-26-running-docker-containers-on-ios-xr-6-1-2/#docker-daemon-support-on-ios-xr)


As shown in the platform specific sections below, the pipeline-kafka combination runs as a Docker container onbox. Some specifics on the setup:  

>
*  In IOS-XR 6.1.2 (before 6.3.1) only global-vrf is supported in the linux kernel.  
>
*  The docker container is launched with the global-vrf network namespace mounted inside the container.  
>
*  The pipeline and kafka instances are launched inside the global-vrf network namespace and listen on all visible XR IP addresses in the kernel (Data ports in global-vrf, Management port in Global-vrf, loopback interfaces in global-vrf).  
>
*  The ports and listening IP selected by pipeline can be changed by the user during docker bringup itself by mounting a custom pipeline.conf (shown in subsequent sections).  
>
*  The XR telemetry process is configured to send Telemetry data to pipeline over 
>    * Transport = **UDP** (only UDP is supported for onbox telemetry) and   
>    * Destination address = listening IP address (some local XR IP) for pipeline.    
{: .notice--warning}

   
   
### NCS5500/Vagrant On-Box Telemetry Setup  

The docker daemon on NCS5500, NCS5000, XRv9k and Vagrant XR (IOS-XRv64) platforms runs on the Host layer at the bottom. The onbox telemetry setup will thus look something like: 

[![xr-docker](https://xrdocs.github.io/xrdocs-images/assets/images/docker_onbox_telemetry.png)](https://xrdocs.github.io/xrdocs-images/assets/images/docker_onbox_telemetry.png)  



    
    
### ASR9k On-Box Telemetry Setup


On ASR9k, the setup is the same from the user perspective. But for accuracy, the Docker daemon runs inside the XR VM in this case, as is shown below.


[![xr_asr9k_docker_libvirt](https://xrdocs.github.io/xrdocs-images/assets/images/docker_onbox_telemetry_asr9k.png)](https://xrdocs.github.io/xrdocs-images/assets/images/docker_onbox_telemetry_asr9k.png)  



**It is recommended to host onbox daemons (in this case Kafka, pipeline, zookeeper) on either the all IP address (0.0.0.0)  or on XR loopback IP addresses. This makes sure that these daemons stay up and available even when a physical interface goes down.**  
{: .notice--info}  


## Docker image for Pipeline+Kafka 
    
    

While a user is welcome to build their own custom Docker images, we have a base image that can take care of installation of pipeline and Kafka+zookeeper for you and is already available on Docker hub:


><https://hub.docker.com/r/akshshar/pipeline-kafka/>


This image is built out automatically from the following github repo:

><https://github.com/ios-xr/pipeline-kafka>


We will utilize this image and build our own custom variant to run on an IOS-XR box for onbox telemetry.




### Building a Docker image for IOS-XR
   
   
To build our own Docker image, you need a development environment with Docker engine installed.  
This is basically the devbox environment that we have setup in earlier tutorials. To understand how to do this, follow the steps in the sections below from the Docker guide for IOS-XR:  

* **Pre-requisites:**  [Setup your Vagrant environment and/or physical boxes (ASR9k, NCS5500 etc.)](https://xrdocs.github.io/application-hosting/tutorials/2017-02-26-running-docker-containers-on-ios-xr-6-1-2/#pre-requisites)  

* **Set up your topology:** [Understand the Topology](https://xrdocs.github.io/application-hosting/tutorials/2017-02-26-running-docker-containers-on-ios-xr-6-1-2/#understand-the-topology) 
  
* **Set up the Devbox environment:** [Install docker-engine on the devbox](https://xrdocs.github.io/application-hosting/tutorials/2017-02-26-running-docker-containers-on-ios-xr-6-1-2/#install-docker-engine-on-the-devbox)  



Now that you have a running debox environment, let's clone the github-repo for the pipeline-kafka project:  

**we use --recursive to make sure all the submodules get pulled as well. The submodules are actual github repos for the standalone pipeline and docker-kafka projects.**
{: .notice--info}

```shell



```

























