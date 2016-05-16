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

---
## First Steps:

As we've already explained in the [xrdocs How-To guide/First Steps]({{ base_path }}/tutorials/xrdocs-how-to#first-steps) section, 

>
Request access to the [xrdocs/xrdocs-images](https://github.com/xrdocs/xrdocs-images/tree/gh-pages) images repository and set up your author profile by sending an email to the [xrdocs-team](mailto:xrdocs-team@cisco.com) with the following data:  
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


## Document Creation Workflow:
>
As a Guest Blogger, document creation is very similar to an owner/contributor workflow, except:
>
* **Maker sure you keep a Local copy of your markdown content through the review process.  You could use any offline markdown editor or a host of online markdown editors to do so.**
* If you've understood how to use [prose.io](http://prose.io) from the [@xrdocs How-To guide]({{ base_path }}/tutorials/xrdocs-how-to#introducing-proseio) section, then there is **NO DIFFERENCE** between a guest blogger and an owner/contributor workflow during document creation.
* Copy and Paste your markdown content into [prose.io](http://prose.io), in the right repository.
* Make sure you **uncheck** the Published checkbox in the metadata field. Fill up the rest of fields as always. If you do not uncheck the published flag, then you will be asked to do so during the review process.
* Commit.
{: .notice-info}




## Commit and Peer review: Important

The difference in the workflow is post commit. As a Guest Blogger, as soon as you save the markdown content in 
[prose.io](http://prose.io), a **Pull Request** will be generated on the github repository that you contributed to (application-hosting, telemetry etc.)

* Post the commit, the owner/contributors of the github repository can then assign a reviewer of your content on Github. 

* For example if you contributed to the application-hosting xrdocs github repo [xrdocs/application-hosting](https://github.com/xrdocs/application-hosting/tree/gh-pages)

* As a Guest Blogger, Keep a local copy of your markdown content with you.
* Once the Review process is complete, the assigned owner will merge the content, so that your recent changes get reflected in prose.io even after you close your tab.
* Your document will be part of the website, but will remain unpublished until the owner is comfortable.






















