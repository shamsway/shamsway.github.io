---
layout: post
theme: jekyll-theme-modernist
title: "Cloud Connectivity 101"
date: 2020-11-17
comments: true
excerpt: With my prior post in mind, we can look at the various methods available for connecting to the cloud.<p>
---

{: .center}
![]({{ "/resources/2020/11/lightning.png" | absolute_url }})

With my [prior post]({% link _posts/2020-11-12-network-princples-cloud.md %}) in mind, we can look at the various methods available for connecting to the cloud. This isn’t intended to be an exhaustive list, but it should cover the vast majority of your options.

## Public IP
**Pros**: Ubiquitous, inexpensive<br>
**Cons**: Potentially insecure, no performance guarantees<br>
**What you’ll need**: A business-class internet circuit<br>

{: .center}
![]({{ "/resources/2020/11/cloud-public-ip.png" | absolute_url }})

Regardless of the other methods listed, you’ll connect to a cloud provider over Public IP the first time you connect to their console to set up your account. Depending on what you host in the cloud, this may be the only connectivity you need. It’s the easiest way to get to the cloud, but also the least secure. If you’re transferring sensitive information over the public internet, make sure you leverage modern application-based encryption, whether over TLS/HTTPS, SSH or some other protocol.

Assuming we’re talking about [Infrastructure as a Service (IaaS)](https://en.wikipedia.org/wiki/Infrastructure_as_a_service) and running virtual instances in the cloud, there are variations between providers around how public IPs are assigned. It’s important to understand how your provider assigns addresses, and how they recommend exposing them to the internet. Frequently a load balancer is used for this, and it’s a good idea to use one even if you’re starting small. It’s easier to start with a load balancer than move to one later.

Public connectivity is not well suited for the typical backend administrative access you’d see in a traditional data center. No security team would recommend exposing all your servers to the internet. Addressing this will likely require adding in one of the connection methods listed below, like a VPN.

Cloud providers allow tight integration between resources you deploy and their hosted DNS solution, but it will still be up to you to configure it properly. Take time to learn the various network services provided, as they will be similar to the network appliances you deploy on prem, but the capabilities and requirements may be different from what you’re used to. As I mentioned previously, there are many options for applying security policy in the cloud, so developing a solid security strategy should be one of the first things you do.

The remaining connection methods, apart from Direct Connection and Network as a Service, build on top of Public IP connectivity. 

## Traditional VPN
**Pros**: Secures sensitive traffic<br>
**Cons**: Requires additional hardware or software, potential performance bottleneck, does not scale well<br>
**What you’ll need**: Hardware or software capable of terminating an IPSec or SSL VPN tunnel<br>

{: .center}
![]({{ "/resources/2020/11/cloud-trad-ipsec.png" | absolute_url }})

Layering a [VPN](https://en.wikipedia.org/wiki/Virtual_private_network) on top of Public IP connectivity will provide a secure connection into your cloud resources. All cloud providers support [IPSec](https://en.wikipedia.org/wiki/IPsec) tunnels, and most support establishing [BGP](https://en.wikipedia.org/wiki/Border_Gateway_Protocol) peering over the tunnel to exchange routes. If you need client-based SSL VPN you will need to deploy a network appliance in your cloud environment, but there are [numerous](https://aws.amazon.com/marketplace/b/Network-Infrastructure/2649366011) [options](https://azuremarketplace.microsoft.com/en-us/marketplace/apps/category/networking?subcategories=firewalls&page=1) [available](https://console.cloud.google.com/marketplace/browse?filter=category:networking).

Using a VPN is the easiest way to build a [hybrid cloud](https://en.wikipedia.org/wiki/Cloud_computing#Hybrid_cloud) solution, and it will give you a secure way to access any instances you’ve deployed via their private IP. If you were paying attention to what I wrote earlier in my previous post, you know that you should have a good IP addressing scheme that includes non-overlapping IP ranges for everything you’re deploying on prem and in the cloud. This becomes even more important if you’re deploying resources into multiple regions in a single cloud provider, or multiple cloud providers.

VPNs, like most technologies, evolve over time, so if you have an aging VPN solution in your environment you may want to consider replacing it with a modern one. Policy-based IPSec VPNs have been around for decades, but cloud providers are encouraging route-based VPNs, which is what you'll have to use if you want dynamic routing over VPN. As your environment scales, dynamic routing becomes increasingly important, so if you aren’t comfortable with BGP now is the time to learn it.

## Layer 2 VPN
**Pros**: IP Mobility<br>
**Cons**: Requires additional hardware or software, potential performance bottleneck, not recommended for long term deployments<br>
**What you’ll need**: Specialized hardware or software capable of building an L2 tunnel<br>

{: .center}
![]({{ "/resources/2020/11/cloud-l2vpn.png" | absolute_url }})

A Layer 2 VPN is used to span a Layer 2 segment, typically a VLAN, across a WAN link. I don’t have scientific data to back this up, but I’d bet a milkshake that if you asked a hundred network engineers if spanning Layer 2 across sites is a good idea, ninety-nine of them would say no. Spanning layer 2 across sites, or over a VPN, introduces complexity and does not scale well. There are some good reasons to do it, but I would never recommend it as a long-term solution.

Normally, a layer 2 VPN is used to migrate existing VMs to another site or cloud provider, while preserving assigned IP addresses. Common scenarios are disaster recovery and data center evacuation. I won’t go on another rant about the importance of DNS, but you can see why I climbed up on that soapbox. Putting that aside, if everyone involved knows the risks introduced by stretched L2, and it’s temporary, it can be a handy tool.

There are a handful of options for L2 VPN, including [VMware HCX](https://cloud.vmware.com/vmware-hcx) and [NSX L2VPN](https://docs.vmware.com/en/VMware-NSX-T-Data-Center/2.5/administration/GUID-86C8D6BB-F185-46DC-828C-1E1876B854E8.html). These have been verified to work in supported cloud providers, but if you choose another solution, be careful to make sure that it is supported by your cloud provider. There is no traditional layer 2 forwarding in most native cloud provider networks, so an overlay like [VXLAN](https://en.wikipedia.org/wiki/Virtual_Extensible_LAN) or [GENEVE](https://en.wikipedia.org/wiki/Generic_Networking_Virtualization_Encapsulation) is used to emulate layer 2 semantics. These overlays encapsulate packets in UDP, so large packets will be fragmented when transmitted over a WAN link. Unless there is a solution for local traffic egress, there will be tromboning of traffic across the L2VPN for any remote endpoints to reach their default gateway.

My advice is to stick to routed layer 3 traffic if possible, even if it takes some work to get there. L2VPN is a tool that can be deployed if absolutely necessary.

## SD-WAN
**Pros**: Flexibility, scalability, potential cost savings<br>
**Cons**: Requires additional hardware or software, potential for vendor lock-in<br>
**What you’ll need**: Hardware or software capable of building an SD-WAN mesh, like [VeloCloud](https://www.vmware.com/products/sd-wan-by-velocloud.html)<br>

{: .center}
![]({{ "/resources/2020/11/cloud-sdwan.png" | absolute_url }})

[Software Defined WAN (SD-WAN)](https://en.wikipedia.org/wiki/SD-WAN) may be the most exciting advancement in the world of networking in the past decade. Building and maintaining VPN connections is a tough job, especially at scale. SD-WAN makes this process much simpler, since all the heavy lifting of creating tunnels, monitoring connectivity, and intelligently routing traffic between locations is handled by a controller. There are potential cost savings as well. Many businesses have replaced their expensive MPLS networks with SD-WAN meshes running over redundant internet connections.

Currently SD-WAN is not standardized, and each vendor offering has its own unique feature set. You will need to do some homework on your own to find the solution that works best for your environment and cloud providers. If you’re considering hybrid-cloud or multi-cloud deployments, you should certainly look at SD-WAN for connecting your environments.

## Direct Connection
**Pros**: High-bandwidth, low-latency cloud connectivity<br>
**Cons**: Cost<br>
**What you’ll need**: A point-to-point circuit from a local telco that provides connectivity to the cloud provider of your choice, and a router or firewall capable of terminating the circuit<br>

{: .center}
![]({{ "/resources/2020/11/cloud-directconnect.png" | absolute_url }})

One of the challenges of working with the cloud is nomenclature. Cloud providers offer so many capabilities, and each offering has a name that has been carefully crafted by their marketing department. You may see a direct connection to a cloud provider referred to as Direct Connect, Express Route, Cloud Interconnect, Fast Connect, or Direct Express Cloud Bonanza. Okay, the last one is fake, but you get the point.

While there is some variation between how the various cloud providers handle direct connections, this is a straightforward path to the cloud. If you are in a standalone data center, you will likely be working with your local telco to provision a circuit from your data center to the cloud provider of your choice. If you have an existing MPLS network, you may be able to have a “leg” connected to a cloud provider as an alternative. Many colocation facilities are offering direct circuits or cross-connects to the closest geographic cloud regions, so check your colocation offerings if that is where your equipment resides.

Review your cloud provider’s documentation for the technical requirements and ordering process for a direct connection. Once the physical circuit is installed, there will be a setup process to complete in the cloud provider portal. Depending on whether you want connection to private resources (e.g. virtual instances deployed with private addresses) or public resources (e.g. provider offerings like object storage), you will need to follow the provider documentation to set up routing across your connection. Most likely this will involve bringing up a BGP peering with the provider between your network and theirs. You may be required to have your own [Autonomous System Number (ASN)](https://en.wikipedia.org/wiki/Autonomous_system_(Internet)) and dedicated public IP address range to access public resources over a direct connection.

I will explore the specifics of the various cloud provider direct connection options in a future post.

## Network as a Service (NaaS)
**Pros**: High-bandwidth, low-latency cloud connectivity, and the ability to dynamically provision cloud connections<br>
**Cons**: Cost, only available in limited locations<br>
**What you’ll need**: A cross connect to the NaaS provider, and compatible hardware to terminate the connection<br>

{: .center}
![]({{ "/resources/2020/11/cloud-naas.png" | absolute_url }})

The last connection method I’ll mention is what some refer to as [Network as a Service (NaaS)](https://en.wikipedia.org/wiki/Network_as_a_service). This is similar to a direct connection, but with much more flexibility. [Megaport](https://www.megaport.com/) and [Equinix Cloud Exchange Fabric](https://www.equinix.com/interconnection-services/cloud-exchange-fabric/) are two examples of this type of service. Typically, you will need to be in a colocation facility to connect to a NaaS provider.  If you’re in a standalone data center, you could provision a circuit to your closest NaaS provider and connect via that method.

Once physically connected to your network, NaaS providers allow you to dynamically provision virtual circuits to cloud providers, managed service providers (MSPs), other data centers, or directly to an ISP. The strength of this solution is in its flexibility. Many NaaS providers provide an API to provision virtual circuits, meaning you could dynamically create and destroy connections to various cloud providers as needed.

If you are looking for high-speed, low-latency connectivity to the cloud, and NaaS is available to you, it’s a great choice.

## Wrap Up
I’ll be exploring additional topics pertaining to cloud connectivity in future posts, but I hope this is a helpful rundown of the options. To recap, you should have a fully developed plan before you provision any sort of cloud connectivity. The actual connectivity, whether it be over the internet, VPN, or direct connection, will depend on several factors. Budget is likely the biggest hurdle for most, and you will pay for better performance. 

When making your decision, consider the words of [Eyvonne Sharp](https://twitter.com/SharpNetwork/status/1326917912347734025), "A myopic focus on cost, instead of business value, is the bane of IT.  It is also a harbinger of irrelevance."
