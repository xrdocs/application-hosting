---
published: true
date: '2023-04-18 18:45 -0600'
title: Application Hosting With Appmgr
author: Suhaib Ahmad
excerpt: Technical Marketing Engineer at Cisco
tags:
  - iosxr
  - cisco
  - linux
  - appmgr
  - Application Hosting
position: hidden
---
{% include base_path %}
{% include toc %}

<h1>Third Party Application Hosting on IOS XR</h1>
<br>
Application Hosting on IOS-XR allows users to run third-party applications on Cisco routers running IOS-XR software. This article will introduce the Application Hosting features and provide a step-by-step guide to running your own application on XR.

<h2> Why use third-party applications? </h2>

- You can use a third-party application to extend router capabilities to complement IOS-XR features.
- TPAs help in optimizing the compute resources required in a deployment.
- TPAs can provide operational simplicity by allowing you to use the same CLI tools and programmability tools for managing both IOS-XR and the application.
- By leveraging well-defined programmable interfaces on the router such as gRPC, gNMI, and SL-APIs, TPAs can readily exchange data with the router over secure channels. 
- Certain use-cases such as service monitoring and service assurance benefit from having an application that can send and receive traffic on services from the router itself.  This provides deeper visibility into network.

<h2>App Hosting Components on IOS XR</h2>
Now that we have established some use-cases for hosting applications on routers, let’s dive into IOS XR features that can help us in doing so.

- Docker on IOS XR: The Docker daemon is packaged with IOS XR software on the base Linux OS. This provides native support for running applications inside docker containers on IOS XR. Docker is the preferred way to run TPAs on IOS XR.

- Appmgr: While the docker daemon comes packaged with IOS XR, docker applications can only managed using appmgr. Appmgr allows users to install applications packaged as rpms and then manage their lifecycle using IOS XR CLI and programmable models. We will discuss appmgr features in depth in later sections.
A public repository (https://github.com/ios-xr/xr-appmgr-build) contains scripts that package docker images into appmgr supported rpms. We will use these scripts to build appmgr rpms in a later section of this article.

- PacketIO: This is the router infrastructure that implements the packet path between TPAs and IOS XR running on the same router. It allows TPAs to leverage XR forwarding to send and receive traffic. A future article in this series will discuss the architecture and features of PacketIO in-depth.

<h2>TPA Security</h2>

IOS XR comes with built in guardrails to prevent Third Party Applications from interfering with its functions as a Network OS.

- While IOS XR does not limit the number of TPAs that can be run simultaneously, it does restrict the resources available to the docker daemon for the following parameters:
	- CPU: ¼ CPU per core available in the platform.
	- RAM: 1G Maximum
	- Disk is limited by the partition size which varies by platform. Can be checked by executing “run df -h” and looking at the size of the /misc/app_host or /var/lib/docker mounts.

- All packets to and from the application are policed by the XR control protection, LPTS. Read this blog to learn more about XR's LPTS [Introduction to NCS 55XX and NCS 5xx LPTS](https://xrdocs.io/ncs5500/tutorials/introduction-to-ncs55xx-and-ncs5xx-lpts/)

- Signed Applications are supported on IOS XR. Users can sign their own applications by onboarding Owner Certificate (OC) using Ownership Voucher based workflows described in RFC 8366. After onboarding an Owner Certificate, users can sign applications with GPG keys based on the Owner Certificate which can then be verified while installing the application on the router.

<center><img src="{{base_path}}/images/secure-app-workflow.png" width="200" height="150" alt="RFC 8366 app installation" style="padding:1px;border:thin solid black;"></center>

<h2>IOS XR appmgr</h2>

<img src="{{base_path}}/images/appmgr-intro.png" width="700" height="800" alt="appmgr comparison" style="padding:1px;border:thin solid black;">


Command reference:

#### Application Install
```
appmgr
|- package 
|    |- install <rpm>
|    |- uninstall <rpm>
```
- ```<rpm>``` is a path to the application rpm.
- Install: Installs the rpm and the application is ready to be configured.
- Uninstall: Uninstalls the application that was un-configured.
- From a docker standpoint, executing appmgr install will load the docker image from the rpm into the local repository. Executing appmgr uninstall will remove the docker image from the local repository.

#### Application Activation
```
configure

appmgr
|- application <name>
|    |- type <type>
|    |- source <source>
|    |- run-opts <run-options>
|    |- run-cmd <cmd>

commit
```
- Type: Type of app [docker, native]
- Source: app source file [docker image name]
- Run-opts: docker run options [docker only]
-- Not all docker run opts are supported
- Docker-run-cmd – Command to run in container [docker only]
- Once committed the container will be started by the App Manager

#### Application Action
```
appmgr
|- application
|    |- copy
|    |- start
|    |- stop
|    |- kill
|    |- exec

```

- Start: Start an app [Docker equivalent: docker start]
- Stop: Stop an app [Docker equivalent: docker stop]
- Kill: Kill an app [Docker equivalent: docker rm]
- Exec: run command in container [Docker equivalent: docker exec]
- Copy: copy files to/from container


#### Application Monitoring

```

 show appmgr
  |- application-table
  |- application name <name>
  |    |- info [summary|detail]
  |    |- stats
  |    |- logs
  |- source-table
  |- source name <name>

```

- Application-table: Shows a consolidated view of all applications and their status
- Application info: Show application status and information
- Application stats: Show app stats [Docker equivalent: docker stats/top]
- Application logs: Show app logs [Docker equivalent: docker logs/journalctl]
- Source table: Show information about available sources.

#### Docker hello-world example

Let us try running a docker hello-world container on a Cisco router running IOS XR using appmgr.

To get started, clone the appmgr-build repository on your development environment.

```git clone https://github.com/ios-xr/xr-appmgr-build```

The cloned directory structure should look like:
```
xr-appmgr-build
├── appmgr_build
├── clean.sh
├── docker
│   ├── build_rpm.sh
│   ├── Centos.Dockerfile
│   └── WRL7.Dockerfile
├── examples
│   └── alpine
│       ├── alpine.tar.gz
│       ├── build.yaml
│       ├── config
│       │   └── config.json
│       └── data
│           └── certs
│               ├── cert.pem
│               ├── client.key
│               ├── server.crt
│               └── server.key
├── LICENSE
├── Makefile
├── README.md
└── release_configs
    ├── eXR_7.3.1.ini
    └── ThinXR_7.3.15.ini
```

Inside the xr-appmgr-build directory, let us create a directory specific to our application (hello-world). In the hello-world directory, we need to have the following:

- build.yaml file that will specify the build parameters for our application.
- A tarball of the docker image
- [Optional] Config directory containing any configuration files for our application.
- [Optional] Data directory containing any other files for our application.

For our example, we will not set any config or data directories.

```
cisco@cisco:~/demo$ cd xr-appmgr-build/
cisco@cisco:~/demo/xr-appmgr-build$ mkdir hello-world
cisco@cisco:~/demo/xr-appmgr-build$ cd hello-world/
cisco@cisco:~/demo/xr-appmgr-build/hello-world$ touch build.yaml
cisco@cisco:~/demo/xr-appmgr-build/hello-world$ docker pull hello-world
Using default tag: latest
latest: Pulling from library/hello-world
Digest: sha256:aa0cc8055b82dc2509bed2e19b275c8f463506616377219d9642221ab53cf9fe
Status: Image is up to date for hello-world:latest
docker.io/library/hello-world:latest
cisco@cisco:~/demo/xr-appmgr-build/hello-world$ docker save hello-world:latest > hello-world.tar
cisco@cisco:~/demo/xr-appmgr-build/hello-world$ ls
build.yaml  hello-world.tar
cisco@cisco:~/demo/xr-appmgr-build/hello-world$
```

The build.yaml contains parameters that specify how to build the rpm. 
- name and version are used to tag the RPM we are building. 

- Release should correspond to an entry in the release_configs directory. Currently, ThinXR_7.3.15 corresponds to LNT platforms and eXR_7.3.1 corresponds to eXR platforms. 

- We must specify the name and path to the docker tarball under sources. 

- [Optional] We specify the name and path to the config directory and data directory under config-dir and data-dir respectively. We are not using data or config directories in this example.
 
Example build.yaml: 
 ```
packages: 
- name: "hello-world" 
  release: "ThinXR_7.3.15" 
  version: "0.1.0" 
  sources: 
    - name: hello-world 
      file: hello-world/hello-world.tar
 ```
 
Once the steps above are completed, we can use the appmgr_build script to build the rpm. We run the script using the following command: 
 
```./appmgr_build -b hello-world/build.yaml ```
 
In general: 
 
```./appmgr_build -b <path to target build.yaml> ```
 
Once the build process is complete, the rpm should be present in the /RPMS/x86_64/ directory. 
After building the application, we can copy it to the router using scp (ssh must be enabled on the router).

```scp ./hello-world-0.1.0-ThinXR_7.3.15.x86_64.rpm user@routerIP:/misc/disk1/ ```

After copying the rpm onto the router, we can install it using appmgr CLI commands.

```appmgr package install rpm /misc/disk1/hello-world-0.1.0-ThinXR_7.3.15.x86_64.rpm```

We can verify if the packaged was installed using:

```
RP/0/RP0/CPU0:8201#show appmgr packages installed 
Thu Jan 19 15:59:17.243 PST
Package                                                     
------------------------------------------------------------
hello-world-0.1.0-ThinXR_7.3.15.x86_64.rpm
```

Once installed, the application can be activated using

```appmgr application hello-world activate type docker source hello-world docker-run-opts “<YOUR DOCKER RUN OPTS>”```

Running applications can be viewed in the application table.

```
RP/0/RP0/CPU0:8201#show appmgr application-table 
Thu Jan 19 16:00:29.878 PST
Name        Type   Config State Status                                         
----------- ------ ------------ --------------------------------
hello-world Docker  Activated   Running                        
bgpfs2acl_1 Docker  Activated   Exited (137) 4 months ago                      
bgpfs2acl_2 Docker  Activated   Exited (2) 2 months ago                        
```
