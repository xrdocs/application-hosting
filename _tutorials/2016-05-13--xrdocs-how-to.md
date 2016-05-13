---
published: true
date: "2016-05-13 16:35 +0530"
title: "@xrdocs How-To"
permalink: "/tutorials/xrdocs-how-to"
author: Akshat Sharma
excerpt: "This tutorial provides a walkthrough of the xrdocs platform for blogging, and creating tutorials and techdocs"
tags: 
  - iosxr
  - cisco
position: " "
---

Welcome to @xrdocs!

This tutorial is meant to get a new blogger/technical-writer, unfamiliar with @xrdocs, up and running in minutes.

Cisco IOS-XR 6.0+ brings with it some key operational enhancements in the domains of monitoring, automation, application-hosting and software management. However, these enhancements require the right tools to aid their consumption. 

In our minds, at a macro level we can classify the tools into two types:

* **User end tools/applications**:  eg. YDK-py, Streaming Telemetry plugins for ELK/Prometheus/SignalFX, support for Config-management tools like Ansible, Puppet and Chef, YUM repositories for WRL7 apps etc.
* **Documentation**: Regular Blogs, Step-by-step tutorials and detailed technical documents have to accompany tools. There is probably nothing more infuriating than a badly documented piece of code/tool/feature.
  
  
  
**@xrdocs** has been designed to address these documentation concerns. 
{: .notice--warning}

####
*   We needed a platform to quickly churn out relevant documentation and make sure it remains up-to-date with all the changes coming up in IOS-XR. 

*   We wanted to peer-review our documentation, raise bugs against it and give control directly to the technical writers. In short, we wanted to treat documentation like code.
{: .notice--info}

{% capture notice-text %}
*   We needed a platform to quickly churn out relevant documentation and make sure it remains up-to-date with all the changes coming up in IOS-XR. 

*   We wanted to peer-review our documentation, raise bugs against it and give control directly to the technical writers. In short, we wanted to treat documentation like code.
{% endcapture %}

<div class="notice--info">
  {{ notice-text | markdownify }}
</div>



The choice was simple: If you've followed the tremendous success of github-pages ever since its inception it would be clear that using [github](https://github.com) as a platform to host and edit our documentation as code, was the way to go.

So here we are, @xrdocs is up and running at [https://xrdocs.github.io] with links to domains of interest.
{: .notice--success}



