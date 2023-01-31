---
layout: post
theme: jekyll-theme-modernist
title: "Introducing IP Spaces for VMware Cloud Director"
date: 2023-01-31
comments: true
excerpt: This post provides an introduction to IP Spaces, a new IP Address Management scheme for VMware Cloud Director.<p>
---

{: .center}
[![](/resources/2023/01/sd-computer-network.png)](/resources/2023/01/sd-computer-network.png){:.drop-shadow}

Welcome! This blog post is about a new feature in [VMware Cloud Director](https://www.vmware.com/products/cloud-director.html) (VCD), IP Spaces. As a VMware employee, I want to make it clear that the thoughts and opinions expressed in this post are my own and do not necessarily reflect the position of my employer. With that out of the way, let's try to wrap our heads around IP Spaces in Cloud Director!

I find myself asking the question “Why?” frequently in customer conversations (shout out to [Simon Sinek](https://www.ted.com/talks/simon_sinek_how_great_leaders_inspire_action) and the [Golden Circle](https://simonsinek.com/books/start-with-why/)!) In this blog post, my goal is to get to the “why” of IP Spaces. I will touch on the “how” and “what”, but these are fully covered in the Cloud Director documentation and other blog posts, which are linked at the bottom of this post.

{: .center}
[![](/resources/2023/01/golden-circle.png){: width="400" }](/resources/2023/01/golden-circle.png){:.drop-shadow}
<br>*Simon Sinek's Golden Circle*

# The Background 

When backed by NSX-V, IP address management in Cloud Director is simple. The typical architecture consists of an external network with tenant edge gateways connected. The provider specifies a block of usable IPs that can be assigned to the external interface of each edge. If needed, additional IPs can be pulled from the block and assigned to the edge external interface for NAT, Load Balancing VIPs, VPN endpoints, etc. Everything the tenant needs to connect to the outside world can be accomplished by assigning one or more IPs to an edge interface and routing is very simple.

{: .center}
[![](/resources/2023/01/vcd-nsxv-connectivity.png)](/resources/2023/01/vcd-nsxv-connectivity.png){:.drop-shadow}
<br>*Cloud Director External Connectivity with NSX-V*

External connectivity is quite different when Cloud Director is backed by NSX-T. External networking is provided via a T0 Gateway, which is created by the provider and imported into Cloud Director. Each tenant edge gateway is a T1 router that is connected to the T0 (or in some cases, a T0 VRF). Addresses used by the tenant are no longer assigned to an interface, but rather assigned via endpoint IP, which is essentially a loopback address assigned to the T1. Since there are now multiple hops to get from the data center network, through the T0, to the tenant T1, dynamic routing (e.g. BGP) is typically used to advertise the endpoint IPs that are assigned to the T1. These endpoint IPs can be used to SNAT workloads to the internet or terminate IPsec tunnels, providing very similar functionality to what is available in NSX-V.

This change in behavior led to IP address sprawl and providers struggled to keep track of which tenants were using which IPs. To address this challenge, IP Spaces was born.

{: .center}
[![](/resources/2023/01/vcd-nsxt-connectivity.png)](/resources/2023/01/vcd-nsxt-connectivity.png){:.drop-shadow}
<br>*Cloud Director External Connectivity with NSX-T*

# IP Spaces Overview 

In VCD 10.4.1, there is a new configuration section to define IP Spaces. IP Spaces can be Public, Private, or Shared. Public IP Spaces are defined by the provider and specify what public IPs can be consumed by tenants. Private IP Spaces are defined by the tenant and are intended to simplify the process of connecting a tenant virtual data center (VDC) to a corporate WAN. Shared IP Spaces are like Private IP Spaces, allowing providers a streamlined way to provide dedicated services to tenants, such as NTP, software repos, managed services, etc.

The scope of an IP range defines which networks are internal or external, or in other words, which networks are local to VCD, and which are remote. If you are familiar with the old Cisco terminology for NAT, think inside and outside networks. Relating this to NAT is helpful because that is one of the primary reasons that these scopes are defined. In future VCD releases, this information may be used to automatically create NAT and NONAT rules to simplify the configuration of typical architectures.

Rounding out the concepts that are included in an IP Spaces are IP ranges, IP prefixes, and quota settings. IP ranges can be supplied in list form or CIDR notation and must be within the range defined as the internal scope. Tenants can request individual IPs out of the range to assign for services like NAT or a load balancer VIP. IP prefixes are also constrained to the internal scope, and they define specific subnets that tenants can consume. Quota settings define how many individual IPs and prefixes each tenant can use.

# The Why

Defining these parameters – IP Space type, scope, ranges, prefixes, and quotas – provides VCD with far more information than was available with the basic IP address management in previous versions. Providers have fine-grained control over exactly which IP addresses and ranges tenants are allowed to consume. This also means that future VCD releases will have enough information to potentially configure NAT/NONAT rules, firewall rules, and BGP policy (prefix lists/filtering/etc.) for a variety of common topologies. The initial release of IP Spaces is just the beginning, providing a much more manageable and coherent IP address management system for providers and tenants. I am looking forward to seeing what other new capabilities will be unlocked as this feature evolves.

# Helpful Links

Release Notes: [https://docs.vmware.com/en/VMware-Cloud-Director/10.4.1/rn/vmware-cloud-director-1041-release-notes/index.html](https://docs.vmware.com/en/VMware-Cloud-Director/10.4.1/rn/vmware-cloud-director-1041-release-notes/index.html)

Documentation: [https://docs.vmware.com/en/VMware-Cloud-Director/10.4/VMware-Cloud-Director-Tenant-Portal-Guide/GUID-FB230D89-ACBC-4345-A11A-D099D359ED1B.html](https://docs.vmware.com/en/VMware-Cloud-Director/10.4/VMware-Cloud-Director-Tenant-Portal-Guide/GUID-FB230D89-ACBC-4345-A11A-D099D359ED1B.html)

Other blog posts on IP Spaces:

* New Networking Features in VMware Cloud Director 10.4.1: [https://fojta.wordpress.com/2022/12/16/new-networking-features-in-vmware-cloud-director-10-4-1/](https://fojta.wordpress.com/2022/12/16/new-networking-features-in-vmware-cloud-director-10-4-1/)
* IP Spaces in VMware Cloud Director 10.4.1 – Part 1 – Introduction & Public IP Spaces: [https://kiwicloud.ninja/?p=69005](https://kiwicloud.ninja/?p=69005)
* IP Spaces in VMware Cloud Director 10.4.1 – Part 2 – Private IP Spaces: [https://kiwicloud.ninja/?p=69028](https://kiwicloud.ninja/?p=69028)
* IP Spaces in VMware Cloud Director 10.4.1 – Part 3 – Tenant Experience, Compatibility & Summary: [https://kiwicloud.ninja/?p=69044](https://kiwicloud.ninja/?p=69044)

# Notes

The [image](/resources/2023/01/sd-computer-network.png) at the top of this post is what you get when you prompt an [AI image generator](https://en.wikipedia.org/wiki/Stable_Diffusion) to create a picture with computer networking and clouds. I find it weird, and I like it.