---
published: true
date: "2016-05-16 16:06 +0530"
title: "@xrdocs Guest Bloggers"
permalink: "/tutorials/xrdocs-guest-bloggers"
author: Akshat Sharma
excerpt: "A Brief walkthrough of the Github based Documentation review process with @xrdocs"
tags: 
  - iosxr
  - cisco
position: hidden
---
{% include toc icon="table" title="Guest Blogging on @xrdocs" %}

{% include base_path %}
[Back to @xrdocs How-To]({{ base_path }}/tutorials/xrdocs-how-to)


## First Steps:

As we've already explained in the [xrdocs How-To/adding images]({{ base_path }}/tutorials/xrdocs-how-to#adding-images-to-your-markdown-content) section, 

>
Request access to the [xrdocs/xrdocs-images](https://github.com/xrdocs/xrdocs-images/tree/gh-pages) images repository by sending an email to the [xrdocs-team](mailto:xrdocs-team@cisco.com) with the following data:  
>
* Full Name
* Github ID
* Email
* attach a bio photo in jpg/jpeg/png format
* twitter handle (optional)
{: .notice--warning}


---

## Quick Recap on types of Users @xrdocs:


>
There are 2 types of users on @xrdocs
>
*   **Owners/contributors** of the corresponding github repos (app-hosting, telemetry etc.). The documents created by these users on prose.io directly get committed to the github repos (hence the website), when saved.
*   **Guest Bloggers**: Do NOT have write access to repositories. All their documents created on prose.io appear as pull requests on github, and are subject to a peer review by the owners/contributors before merging with the website.  

TMEs in the Web Solutions group at Cisco are by default the owners of the internal github repositories that host the XR documentation.

---

## Guest Bloggers

"Guest Blogger" is a generic term we use to define any user that does not have write access to the internal github repository where the documents get stored.

Without going into details of how the github organization (xrdocs) and its repositories (application-hosting, telemetry etc.) are set up, it would be good to understand which repositories a Guest Blogger will be granted access to:


|  Github Repository        | Owner                                      | Guest Write Access| 
| ------------------------- | -----------                                | ----------------- |
| xrdocs/application-hosting|[Akshat Sharma](https://github.com/akshshar)|    No             |
| xrdocs/telemetry          |[Shelly Cadora](https://github.com/scadora) |    No             |
| xrdocs/xrdocs-images      | Cisco Web TMEs                             |    Yes            |   

[xrdocs/xrdocs-images](https://github.com/xrdocs/xrdocs-images/tree/gh-pages)  is the only repository that Guest bloggers will have write access to. Think of this as your image repository in the cloud.   




















