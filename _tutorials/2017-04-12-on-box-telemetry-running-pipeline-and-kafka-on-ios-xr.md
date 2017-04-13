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
  
  
This may be done for different reasons:  

  * Users may choose to simplify their server environment and not run external services (like    
    Kafka, influxDB stack or prometheus/Grafana etc.). Typically, some  
    background in devops engineering is often important to understand how to scale out these 
    services and process large amounts of data coming from all routers at the same time. 

  * The alerts or the remediation actions that a user intends to perform based on the Telemetry 
    data received may be fairly simplistic and can be done on box.


Bear in mind that running onbox does come with its own concerns. Network devices typically have limited compute capacity (CPU/RAM) and limited disk capacity. While CPU/RAM isolation can be achieved using Containers on box, managing the disk space on each individual router does require special care when dealing with Streaming Telemetry applications.
{: .notice--warning}


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




### Building a Pipeline-kafka Docker image for IOS-XR
   
   
To build our own Docker image, you need a development environment with Docker engine installed.  
This is basically the devbox environment that we have setup in earlier tutorials. To understand how to do this, follow the steps below (in order) from the Docker guide for IOS-XR:  


* **Pre-requisites:**  [Setup your Vagrant environment and/or physical boxes (ASR9k, NCS5500 etc.)](https://xrdocs.github.io/application-hosting/tutorials/2017-02-26-running-docker-containers-on-ios-xr-6-1-2/#pre-requisites)  

* **Set up your topology:** [Understand the Topology](https://xrdocs.github.io/application-hosting/tutorials/2017-02-26-running-docker-containers-on-ios-xr-6-1-2/#understand-the-topology) 
  
* **Set up the Devbox environment:** [Install docker-engine on the devbox](https://xrdocs.github.io/application-hosting/tutorials/2017-02-26-running-docker-containers-on-ios-xr-6-1-2/#install-docker-engine-on-the-devbox)  



Now that you have a running devbox environment, let's clone the github-repo for the pipeline-kafka project:  

**we use --recursive to make sure all the submodules get pulled as well. The submodules are actual github repos for the standalone pipeline and docker-kafka projects.**
{: .notice--info}


<div class="highlighter-rouge">
<pre class="highlight" style="white-space: pre-wrap;">
<code>
vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$<mark> git clone --recursive https://github.com/ios-xr/pipeline-kafka</mark>
Cloning into 'pipeline-kafka'...
remote: Counting objects: 38, done.
remote: Compressing objects: 100% (30/30), done.
remote: Total 38 (delta 15), reused 20 (delta 4), pack-reused 0
Unpacking objects: 100% (38/38), done.
Checking connectivity... done.
Submodule 'bigmuddy-network-telemetry-pipeline' (https://github.com/cisco/bigmuddy-network-telemetry-pipeline) registered for path 'bigmuddy-network-telemetry-pipeline'
Submodule 'docker-kafka' (https://github.com/spotify/docker-kafka) registered for path 'docker-kafka'
Cloning into 'bigmuddy-network-telemetry-pipeline'...
remote: Counting objects: 14615, done.
remote: Compressing objects: 100% (8021/8021), done.
remote: Total 14615 (delta 3586), reused 0 (delta 0), pack-reused 3349
Receiving objects: 100% (14615/14615), 43.97 MiB | 2.02 MiB/s, done.
Resolving deltas: 100% (4012/4012), done.
Checking connectivity... done.
Submodule path 'bigmuddy-network-telemetry-pipeline': checked out 'a57e87c59ac220ad7725b6b74c3570243e1a4ac3'
Cloning into 'docker-kafka'...
remote: Counting objects: 98, done.
remote: Total 98 (delta 0), reused 0 (delta 0), pack-reused 98
Unpacking objects: 100% (98/98), done.
Checking connectivity... done.
Submodule path 'docker-kafka': checked out 'fc8cdbd2e23a5cac21e7138d07ea884b4309c59a'
vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$ <mark>cd pipeline-kafka/iosxr_dockerfile/</mark>
vagrant@vagrant-ubuntu-trusty-64:~/pipeline-kafka/iosxr_dockerfile$ ls
Dockerfile  kafka_consumer.py
vagrant@vagrant-ubuntu-trusty-64:~/pipeline-kafka/iosxr_dockerfile$ 

</code>
</pre>
</div>


Let's take a look at the Dockerfile under the `iosxr_dockerfile` folder:  

<div class="highlighter-rouge">
<pre class="highlight" style="white-space: pre-wrap;">
<code>
vagrant@vagrant-ubuntu-trusty-64:~/pipeline-kafka/iosxr_dockerfile$<mark> cat Dockerfile </mark>
FROM akshshar/pipeline-kafka:latest

Maintainer akshshar

# Specify the "vrf" you want to run daemons in during build time
# By default, it is global-vrf

ARG vrf=global-vrf

# Set up the ARG for use by Entrypoint or CMD scripts
ENV vrf_exec "ip netns exec $vrf"

# Add a sample kafka_consumer.py script. User can provide their own
ADD kafka_consumer.py /kafka_consumer.py


CMD $vrf_exec echo "127.0.0.1 localhost" >> /etc/hosts && $vrf_exec supervisord -n
</code>
</pre>
</div>  



Let's break it down:  

All the references below to Dockefile instructions are derived from official Dockerfile Documentation:  
<https://docs.docker.com/engine/reference/builder/#known-issues-run>
{: .notice-warning}  


```Dockerfile
ARG vrf=global-vrf
```

We setup the script to accept arguments from the user during build time. This will allow us to be flexible in specifying the vrf (network namespace) to spin up the daemons in, in the future. Today in 6.1.2 (before 6.3.1), only global-vrf is supported.  
{: .notice--info}


```Dockerfile
ENV vrf_exec "ip netns exec $vrf"
```

In Dockerfiles, the ARG variables are rejected in the ENTRYPOINT or CMD instructions. So we set up an ENV variable (which is honored) to create a command prefix necessary to execute a command in a given network namespace (vrf).  
{: .notice--info}


```Dockerfile
ADD kafka_consumer.py /kafka_consumer.py
```

We place the sample application (in this case written in python) inside the image to act as a consumer of the Telemetry data pushed to Kafka. This application can contain custom triggers to initiate alerts or other actions. For this tutorial, we will initiate the script manually post launch of the container. The user can choose to start the application by default as part of the ENTRYPOINT or CMD instructions in the dockerfile.
{: .notice--info}


```Dockerfile
CMD $vrf_exec echo "127.0.0.1 localhost" >> /etc/hosts && $vrf_exec supervisord -n
```

This specifies the command that will be run inside the container post boot. The first part of the command `$vrf_exec echo "127.0.0.1 localhost" >> /etc/hosts` sets up /etc/hosts with an entry for localhost making it easier for kafka and applications to talk to each other locally. The second part of the command `$vrf_exec supervisord -n` is used to start all the services in the correct vrf (hence the `$vrf_exec`).    
We use supervisord to easily specify multiple daemons that need to be launched (pipeline, kafka, zookeeper).  You can get more details on supervisord here: <http://supervisord.org/>  
{: .notice--info}





