---
layout: post
theme: jekyll-theme-modernist
title: "Cloud Connectivity 202 - Extending Layer 2 Into the Cloud"
date: 2021-01-11
comments: true
excerpt: In this post, I will talk about extending layer 2 into the cloud, including scenarios for when it is a good idea, and the numerous dangers involved. I let out a good, long sigh after writing that sentence. Beware, there are dragons ahead.<p>
---

{: .center}
![]({{ "/resources/2021/01/dragons.jpg" | absolute_url }})

In this post, I will talk about extending layer 2 into the cloud, including scenarios for when it is a good idea, and the numerous dangers involved. I let out a good, long sigh after writing that sentence. Beware, there are dragons ahead.

If you’ve been around networks long enough, you’ve probably seen a network taken to its knees by a loop. The [Spanning Tree Protocol](https://en.wikipedia.org/wiki/Spanning_Tree_Protocol) (STP) exists to prevent this issue, but a simple misconfiguration can prevent STP from doing its job. I’ve personally seen a hospital network taken down for days from a network loop, and I’ve heard many similar stories from other network engineers. A lot of time has been spent coming up with alternatives to STP-based networks, and for good reason. A major part of the CCIE Data Center curriculum I studied was alternatives to STP like TRILL and VXLAN EVPN with MP-BGP. Here’s the point: extending layer 2 can be dangerous, especially when done without precautions in place. 

# Why Do We Need Layer 2?

The very first commercially available Ethernet standard, [10Base5](https://en.wikipedia.org/wiki/10BASE5) used a coaxial cable as a shared medium. Multiple devices could be attached to the same cable, and each was identified by its MAC address. Later Ethernet standards kept backward compatibility with this model. Although significant improvements have been made over the years, much of the complexity of layer 2 forwarding semantics stem from this initial design.

Devices can be connected to a hub or a switch without any sort of central coordination. Neighbors are discovered by broadcasting ARP requests, which makes building an ad-hoc network very easy. Things get messier when a broadcast packet enters a looped network. There is no [time to live](https://en.wikipedia.org/wiki/Time_to_live) (TTL) value associated with a layer 2 frame, so it will be forwarded and reforwarded ad infinitum, resulting in a [broadcast storm](https://en.wikipedia.org/wiki/Broadcast_storm). Before we even discuss extending layer 2, it’s important to know that a network can be completely crushed if STP Is improperly configured and a loop is introduced.

# Risks Of Extending Layer 2

Before cloud providers were available, extending layer 2 was typically accomplished over a Data Center Interconnect (DCI) that could serve as a trunk, so one or more VLANs could be extended across the link. The risk here is from fate-sharing. Before extending layer two, any issue would be confined to one site. Once layer two is extended, a problem at one site will extend to the other. There are many good reasons why cloud providers use availability zones, and [fate-sharing](https://en.wikipedia.org/wiki/Fate-sharing) is one of them. The other risk, of course, is that the larger the layer 2 domain, the more opportunities there are to introduce a loop.

Beyond the risks of creating a network outage, extending layer 2 also has some basic disadvantages. A default gateway can only exist at one site, so routed traffic between hosts at the same site may be [tromboned](https://en.wikipedia.org/wiki/Anti-tromboning) across the DCI. If using an overlay to facilitate layer 2 extension, you need to be mindful of the implications that has on MTU across the link. “Silent hosts”, or hosts that don’t properly respond to ARP requests, can also be problematic in this scenario.

# Reasons For Extending Layer 2

Hopefully, I’ve made a good case for why you should be very careful when extending layer 2. There are some good reasons to do so, but it is never something I would recommend doing for a long period of time. The main use cases I see for extending layer 2 are data center evacuation and migrating to the cloud while preserving assigned IP addresses. These are good use cases for layer 2 extension since the extension will be finite. Once the evacuation or migration is complete, the extension can be removed. I have seen layer 2 extensions used for disaster recovery purposes, and I would caution users who want to do this to exercise extreme caution. Indefinite layer 2 extension is a recipe for trouble.

# Methods for Extending Layer 2 to the Cloud

In my post on [cloud connectivity]({% link _posts/2020-11-17-cloud-connectivity-101.md %}), I listed the typical methods for connecting to a cloud provider. You will notice that all of the connections are based on layer 3, apart from a layer 2 VPN, which rides on top of a layer 3 connection. The prior approach of extending VLANs over a circuit simply isn’t an option in the cloud. In the same post, I pointed out that most clouds don’t use traditional layer 2 forwarding in their networks. This certainly presents a problem when trying to extend layer 2 as well! The best solution available is to use an overlay, like [VXLAN](https://en.wikipedia.org/wiki/Virtual_Extensible_LAN) or [GENEVE](https://en.wikipedia.org/wiki/Generic_Networking_Virtualization_Encapsulation), in your cloud of choice. This is where VMware-powered cloud offerings shine since they leverage NSX-T in the cloud-based SDDC. GENEVE, used in NSX-T, will emulate the needed layer 2 functions, as well as provide a layer 2 VPN (L2VPN) appliance to perform the extension. [VMware HCX](https://cloud.vmware.com/vmware-hcx) is also compatible with these solutions and provides layer 2 extension for migrated workloads. 

NSX-T L2VPN and HCX provide guidelines in their documentation that should be carefully studied before deployment. It is also important to know that you cannot use NSX-T L2VPN and HCX at the same time. The HCX documentation states “Virtual machine networks should only be extended with a single solution. For example, HCX Network Extension or NSX L2 VPN can be used to provide connectivity, but both should not be used simultaneously. Using multiple bridging solutions simultaneously can result in a network outage.” Since HCX supports extension to multiple sites, you must deploy the service meshes in a way that does not introduce a loop. See this diagram:

{: .center}
![]({{ "/resources/2021/01/hcx-loop2.png" | absolute_url }})

Other solutions exist beyond VMware products, like [LISP](https://en.wikipedia.org/wiki/Locator/Identifier_Separation_Protocol), but I have not personally seen them used. Ultimately you are restricted to solutions supported by your cloud of choice, which are typically delivered in the form of supported third-party network appliances. If you’re aware of another viable method for extending layer 2 into the cloud, please leave a comment or send me a message on Twitter. I’d love to see what others are using to accomplish this.

# Alternatives To Extending Layer 2

Imagine a world where there was never a hard-coded IP address anywhere. Addresses are assigned automatically, and DNS is instantly updated whenever an address is assigned or changed. Changing addresses is a non-issue since everything relies on DNS name resolution instead of an IP address. Sound too good to be true? It’s not! This is possible today, and it has been for years. Cloud-native networking works exactly in this manner, but it is possible to accomplish on any network, with some effort.

For a variety of reasons, enterprise networks have not operated this way, and it is the primary driver behind the desire to extend layer 2. IPs are frequently hard-coded instead of relying on DHCP for address assignment and DNS for name resolution and service discovery. Running a network is difficult work, and forward-thinking practices like the ones I’m describing aren’t always prioritized. It’s easy to decry this as laziness, or poor planning, but I spent years working in networks similar to what I’ve described. I understand the amount of effort it takes to change the way a network fundamentally works, especially when you’re on a small team managing a huge network. If you never need to migrate a workload to another location, it may not be worth the effort.

The primary alternative to extending layer 2 is to use layer 3 routed connectivity. This is exactly how the cloud was designed to work; why there isn’t any real concept of layer 2 in the cloud, and why cloud providers don’t allow you to extend your VLANs onto their network. Unfortunately, this concept is difficult to swallow if you have a network like the one I describe above - heavily reliant on hard-coded addresses in many places. If this is the case, you will need to work with management to acquire the resources needed to convert to a “cloud friendly” network on premises before moving workloads to the cloud. This may be a small effort or a massive one, depending on the size of your network, but it will position you to be better prepared for the future. The cloud is here to stay, and it’s a wonderful environment to work in once you get the hang of it. 

# Wrap Up

Whether or not it’s a good idea to extend layer 2 is a debate that has been going on for years, and I doubt it will stop any time soon. The guidance I will leave you with is to use layer 3 everywhere you can, and extend layer 2 only if you must. Do everything you can to configure your applications to use DNS instead of hard-coded IPs. Study cloud-native networks and start embracing those concepts in your own network. By doing this, you will be much better prepared for migrating your workloads to the cloud or moving to a hybrid cloud environment.