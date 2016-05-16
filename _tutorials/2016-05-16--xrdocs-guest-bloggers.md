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
* **Create a Local copy of your markdown content first. You could use any offline markdown editor or a host of online markdown editors to do so.**  
* **Use [prose.io](http://prose.io) to submit your content for review**
* ** Update your local markdown content based on reviewer's comments.**
{: .notice--info}




## Commit and Peer review: Important

>
As a Guest Blogger, as soon as you save the markdown content in [prose.io](http://prose.io),   
a **Pull Request** will be generated on the github repository that you contributed to (application-hosting, telemetry etc.)
>
* The owner/contributors of the github repository then assign a reviewer of your content on Github using the pull request.  
* Keep updating the local copy of your markdown content based on reviewer's comments.
* Use prose.io as a method to submit your content and create new pull requests.
* Your document will be part of the website, but will remain unpublished until the review process is over.  
* Once, everyone is satisfied, **check** the Published checkbox in the metadata field and your document will be live.  
{: .notice--info}

## Illustration of the above workflow is shown below:

### Create a Local Markdown file

As a guest blogger, I open my markdown content in my markdown editor of choice, let's say the Atom.io editor.

![Local Markdown File](http://xrdocs.github.io/xrdocs-images/assets/tutorial-images/sample_blog_guest.png)
This is my local markdown file, I'll use it to update my content during the review process.
{: .notice}



### Submit the content on prose.io

Let's assume  we're contributing to the application-hosting xrdocs website at <https://xrdocs.github.io/application-hosting>

Open up prose.io and point to the github-pages repo of the repository:
![Prose.io App-hosting gh-pages](http://xrdocs.github.io/xrdocs-images/assets/tutorial-images/proseio_apphosting_ghpages)































