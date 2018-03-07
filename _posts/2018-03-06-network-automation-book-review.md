---
layout: post
theme: jekyll-theme-modernist
title: "Book Review: Network Programability and Automation"
date: 2018-03-06
comments: true
crosspost_to_medium: true
excerpt: A wall of text with my detailed thoughts on this book<p>
---

{: .center}
![](https://covers.oreillystatic.com/images/0636920042082/cat.gif)

I was anxious to get my hands on *[Network Programability and Automation](http://shop.oreilly.com/product/0636920042082.do)* by Jason Edelman, Scott Lowe, and Matt Oswalt as soon as I saw the first announcement that it was being written. Network automation is a subject that I am deeply interested in, so I was excited when the early access edition showed up on Safari Books Online. I was giving a lot of thought to starting this blog when I began reading, and by the time I made it through a few chapters I knew that I wanted a review of this book to be the first "real" post on my blog.

> Disclaimer 1: This review is based off an early access edition of [Network Programability and Automation](http://shop.oreilly.com/product/0636920042082.do) on Safari Books Online. The final version is not available on Safari yet, but based on the Table of Contents at the O'Reilly site for the book, there were some structural differences between the early access and final editions. I have very few criticisms of this book, but I'll do my best to update this review once I get my hands on the final copy.

> Disclaimer 2: I've never written a book review before. I am somewhat writing this for writing's sake. My goal is to build up some writing chops as I ramp up this blog.

I absolutely love this book. If you're a network engineer with any interest in where the industry is going, it is a must read. Automation is a scary topic for a lot of engineers due to the perception that you have to be a programming wizard to do it. *Network Programability and Automation* effectively dispels that myth by providing an easy to follow blueprint, and it doesn't presume any previous programming experience. This book is a fantastic resource for both novices and seasoned programmers. It's worth noting that this book focuses on open source tools like Ansible, Salt and StackStorm. Some of these tools have commercial variants, but if that's not your cup of tea, there are other closed source options on the market. No matter what tool you choose for Network Automation, this book is an amazing resource. For the remainder of this review I'm going to go chapter by chapter to give a taste of what the book covers, as well as some additional useful resources.

**Chapter 1** begins with an overview of current network industry trends: Software Defined Networking, and everything that comes with it. The authors are careful to avoid providing a formal definition of SDN, which is a smart choice. You can ask ten different vendors or engineers what SDN is, and get ten different answers. They do, however, nail down the ecosystem that is driving this software defined trend. The release of the OpenFlow protocol is really the point where SDN started, so the book begins with a brief history and overview of OpenFlow. NFV, VXLAN, virtual switching, APIs, fabrics and whitebox switching are touched on as well. This is a good primer for the engineer that hasn't been keeping up with all of the recent industry hype and hand-waving.

**Chapter 2** provides a high level over view of the what, why and how of network automation. This chapter spells out the benefits of network automation, and typical use cases are covered. We are seeing a shift in the industry away from SNMP and CLIs to APIs and NETCONF, and this is covered as well. It's not mentioned in the book, but VMware NSX is a great example of this. There is no CLI, only API or GUI access for configuration. There is no SNMP polling either - the NSX API is the only way to monitor the environment. Of course, VMware provides commercial tools to monitor NSX, but it's certainly feasible to roll your own monitoring solution once you're familiar with interacting via API.

**Chapter 3** delves in to Linux history, basics, and networking. The early access edition had both a Linux chapter, and an appendix devoted to advanced networking in Linux. I'm assuming that the appendix was rolled into this chapter, which makes perfect sense.

Linux has become the de facto language of the data center, and it is increasingly important that network engineers have an understanding of it. There is a [long list](https://github.com/itdependsnetworks/awesome-network-automation#open-source-projects) of open source projects used for network automation, so the ability to navigate a linux server is table stakes for getting started. The caveat is you could use a commercial network automation product that requires no Linux knowledge, but what fun is that? This chapter is a crucial resource for engineers with no prior Linux experience, and a good refresher for veterans of the operating system.

The early access edition discusses VRFs and Namespaces on Linux in the "Advanced networking in Linux" appendix. At the time of writing, VRF support in Linux was not complete, so the authors focused on Namespaces to provide a VRF-like experience. VRFs are now fully supported on Linux. The folks at Cumulus Linux have [great writeup](https://cumulusnetworks.com/blog/vrf-for-linux/) on this topic.

**Chapter 4** is a Python primer. If Linux is the language of the data center, Python is the language of open source network automation. This chapter explains programming concepts that are foundational for automation – data types, loops, file I/O, and APIs. This chapter goes deep enough into Python to get you started, but there are plenty of additional resources available to further your Python knowledge. The oft-repeated question, "Is coding a requirement for Network Engineers now?" is addressed as well. I'll let you read the book to see how this is answered, and I agree with the answer given in the book.

 If you want to learn more, these are my recommendations.

* [Kirk Byers' free 8-week Python for Network Engineers course](https://pynet.twb-tech.com/email-signup.html)
* [Google's Python Class](https://developers.google.com/edu/python/)

The early access edition of this book included an appendix on [NAPALM](https://napalm-automation.net) (Network Automation and Programmability Abstraction Layer with Multivendor support), a Python library for interacting with network hardware. I do not know if this content was moved to the Python chapter or not, but the NAPALM is mentioned several times in this book and it's widely used by other tools. It's worth your time to read the [docs](https://napalm.readthedocs.io/en/latest/), follow their [blog](https://napalm-automation.net/blog/), and star their [GitHub](https://github.com/napalm-automation/napalm) project. It's safe to say that when you're starting out with network automation, you're going to use NAPALM in some way, shape or form.

**Chapter 5** explains data models like JSON, XML, YAML, and YANG. The data models are equated to programming elements introduced in the previous chapter. This is not the most riveting information in the book, but it becomes very important as it is foundational to the tools introduced later on, as well as when consuming APIs. I wish there was a little more information about YANG in this chapter, but there are plenty of other resources out there. Once you've read through this book, you can find some great YANG examples at these links:

* https://github.com/YangModels/yang
* https://github.com/openconfig/public

**Chapter 6** is similar to the previous chapter. It introduces the Jinja templating language, which is what you will use to build your configuration templates. This chapter is full of plenty of great examples and advice. Jinja was one thing I had very little understanding of before reading this book, and this flipped on the light switch for me. I believe all of the network automation tools introduced later in the book use Jinja, so it is an important concept to understand.

**Chapter 7** goes into great detail on APIs, both RESTful and non-RESTful, and NETCONF. My takeaway from this chapter that NETCONF is more complicated than the other options. Part of this is from having to deal with XML, and more widespread adoption of RESTCONF should ease this a bit. There is a great overview of using the Python `requests` library for interacting with APIs. There are also useful examples using Cisco Nexus NX-API, IOS-XE RESTCONF, Arista EAPI, NETCONF via ncclient, and an intro to the Python `netmiko` library.

**Chapter 8** is a short detour before getting to the real meat of this book. This chapter explains git version control history, terminology and use cases along with several working examples. Understanding git is an important skill since many existing network automation tools and templates are hosted on GitHub. Any new templates or scripts you create should definitely be version controlled with git, whether that is via GitHub or a private git solution.

**Chapter 9** is a lengthy chapter on automation tools. It starts with an architecture review of existing tools: agent-based/agentless, centralized/decentralized, open or proprietary protocol, etc. The rest of the chapter is devoted to the tools themselves.

The first tool examined is Ansible, which is the one I was least familiar with and most interested in. All of the basics are covered: inventory files and groups, variables, playbooks, using Jinja templates, gathering device data, generating config files and pushing configuration to devices. After reading through this session my mind was full of potential use cases, and I felt it provided a strong enough foundation to get me started. It is worth noting that there are some [syntax changes in Ansible 2.5](https://www.ansible.com/blog/coming-soon-networking-features-in-ansible-2.5) to consider if you are planning on using that version. [Ansible Tower](https://www.ansible.com/tower) is the commercial offering for Ansible, which provides a GUI workflow designer and other features. [AWX](https://github.com/ansible/awx) is the open-source/bleeding edge version of Tower, and is worth investigating if you're considering Ansible.

Salt is covered next, and while I was a bit more familiar with this tool, I'd never used it for network automation. Again, the authors do a great job of covering the basics along with providing some examples (you're probably noticing a theme here.) I learned that Salt is much more complex than I'd originally realized. It's completely capable of complex automation use cases, but it does require more thought and planning to effectively use. Some engineers from CloudFlare have published a free eBook that goes into greater detail on using Salt for Network Automation: *[Network Automation at Scale](https://www.cloudflare.com/media/pdf/network-automation-at-scale.pdf)*

StackStorm is covered last. This tool is primarily event driven - it's described as the "If This Then That" of IT infrastructure. My impression is that it could be set up to work like Ansible or Salt, but that is not really its intended use. There are many moving pieces to understand: Actions, Workflows, Sensors, Triggers, Rules, Packs and Workflows. The event driven nature is a good fit for automated remediation, or chatops via Slack and Hubot. StackStorm has native plugins for these tools.

Here's my take:
* Ansible: Not too difficult to get started with - appropriate for simple and complex scenarios.
* Salt: Feature rich but likely takes some planning to effectively use. It is a good option if you want to automatically respond to events.
* StackStorm: More than just an automation tool, StackStorm is a system you could use to drive a complex IT automation and chatops environment.

This chapter was jam packed with goodness. If I could make one small criticism, it is that I wish this had been split up into three separate chapters. So. Much. Content.

We're almost done now, I promise :grin:

**Chapter 10** is an intro to Continuous Integration/Continuous Development (CI/CD). Typical CI/CD tools and workflows are covered, as well as a basic example. The authors make a point to stress the importance of testing your automation with a CI/CD tool if possible. I do wish there were a few more examples in this chapter. This is probably the hardest nut to crack for anyone starting from scratch with Network Automation. I'm going to continue digging into this topic, so hopefully there will be some future blog posts on the subject. Thankfully we're starting to see Docker-based images from many networking vendors, and that will help streamline the CI/CD process.

**Chapter 11** discusses building a culture for Network Automation. Apart from Chapter 9, this is the most important chapter in the book. Knowing how to automate is great, but it's not much help if there is resistance to change in your organization. This chapter covers how to get started in a "traditional" (i.e. old-school) IT organization, the importance of getting executive buy-in, and how to navigate the build vs. buy decision. Some of the best advice in the chapter is around dealing with and embracing failure. No book on automation is complete without addressing the "Will automation take my job?" question, and it is handled beautifully.

## Closing thoughts

There is nothing in this book that is going to increase your understanding of OSPF or BGP. What it will do is give you the tool set to effectively monitor, configure, and remediate your network. This book covers so much information, yet there is still so much more to learn. I'm going to wrap up this incredibly long post with a link to this video from NANOG 72 titled "[Network Automation: Do I Need Expensive Tools To Do Meaningful Automation?](https://www.youtube.com/watch?v=r-TdsK3RKvo)" In it, Patrick Moore gives some great advice for getting on the "automation on-ramp" and the presentation is a great follow up to *Network Programability and Automation*.

In the words of Jason Edelman, Scott Lowe, and Matt Oswalt: be the "automator", not the "automated".

Edit: I'd intended to include an example script I'd written after reading this book. Check it out [here]({ % post_url 2018-03-07-NSX-automation-example % }).
