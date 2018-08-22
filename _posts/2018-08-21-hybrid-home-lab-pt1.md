---
layout: post
theme: jekyll-theme-modernist
title: "Hybrid Home Lab Pt. 1"
date: 2018-08-21
comments: true
crosspost_to_medium: true
excerpt: Over the last few weeks I've been working on standing up my version of a "real" lab. I've got enough information together to start putting together some blog posts, so let's dive right in.<p>
---

Over the last few weeks I've been working on standing up my version of a "real" lab. I've got enough information together to start putting together some blog posts, so let's dive right in. Previously, my home lab was just a custom built linux server with plenty of memory and software RAID. This was enough to do some small-scale network labs and run the few applications I needed, but it really doesn't qualify as a true home lab. There's no way for me to work with a vSphere or KVM cluster, let alone NSX-v or NSX-T. I've laid out a few goals for my "Hybrid Home Lab":

* On Prem Resources
  * 2x UCS C220 M3
  * Re-purpose existing server as a home NAS
    * Utilize hardware RAID and serve LUNs via iSCSI or NFS
  * [Ubiquiti EdgeSwitch 16XG](https://www.ubnt.com/edgemax/edgeswitch-16-xg/)
  * [Ubiquiti EdgeRouter PoE](https://www.ubnt.com/edgemax/edgerouter-poe/)
  * Purpose: Compute Virtualization Lab (vSphere or KVM), Network Virtualization Lab (NSX-V, NSX-T, EVE-NG, VIRL, GNS3), Kubernetes backup cluster
* Cloud Resources
   * Hosted in [vCloud Director](https://www.vmware.com/products/vcloud-director.html)
   * [Rancher 2.0](https://rancher.com/blog/2018/2018-05-01-rancher-ga-announcement-sheng-liang/) Kubernetes cluster
   * [OPNsense](https://opnsense.org) firewall
     * Also provides [ZeroTier](https://www.zerotier.com) VPN/SD-WAN and [HAproxy](https://haproxy.org) load balancing
     * Replaces NSX edge in vCD
   * [Gluster](https://www.gluster.org) for persistent Kubernetes storage
   * Purpose: Learn Kubernetes, deliver applications independent of on prem resources, test OPNsense as a "cloud router" and ZeroTier for hybrid cloud scenarios
     * Applications I'll try to run: Gitlab, Netbox, Zabbix, Grafana, MariaDB/Postgres, StackStorm, other automation tools, and custom

I will be publishing detailed blog posts on the setup of these components - stay tuned!

# But, Why?

{: .center}
![]({{ "/resources/2018/08/ytho.jpg" | absolute_url }}){:height="35%" width="35%"}

{: .center}
(my daughter approves the use of this meme)

**Why vCD?** I have access to a vCD lab at work. I have to keep a small footprint, but this is much more economical than using another cloud provider. We've run vCD at my [day job](https://thinksis.cmo) for quite a while, and I've become fond of it. It's come a _long_ way since we initially deployed it, and it continues to improve. [#LongLiveVCD](https://twitter.com/search?f=tweets&vertical=default&q=%23LongLiveVCD)

**Why Rancher?** This is another product that we're using at work, so I have some motivation to learn it. It definitely is "training wheels" for Kubernetes, and I'm already getting the itch to experiment with vanilla Kubernetes or OpenShift. For now it does what I need it to, and it's not terribly difficult to take all my YAML files and load them in another Kubernetes cluster later.

**Why are you running stateful applications in Kubernetes?** I understand that Kubernetes is mainly for stateless applications and microservices, but it does support stateful workloads. This is a lab, and sometimes it is fun to push the limits.

**Why Gluster?** Persistent storage in Kubernetes is a PITA if you're not using one of the major cloud providers, or leveraging storage that provides a Kubernetes plugin. [Heketi](https://github.com/heketi/heketi) provides an API interface for GlusterFS that Kubernetes can leverage. I'll provide more information in a later blog post, but this was the easiest way to provide redundant persistent storage for my Rancher cluster.

**Why OPNsense?** Yes, vCD provides an NSX edge. In vCD 9.1, it is full featured and suitable for most workloads. I'm a network nerd so this is one of the areas where I want more flexibility than what NSX can provide. The [feature list](https://opnsense.org/about/features/) for OPNsense is impressive, and most importantly for me, it has built in support for ZeroTier.

**Why ZeroTier?** Please see my previous post on [cloud automation]({% link _posts/2018-03-24-vcd-terraform-example.md %}). Future posts will go into more detail on this as well.

# Show me the diagram

IP addresses have been changed to protect the innocent.

{: .center}
[![hybrid lab diagram]({{ "/resources/2018/08/hybrid_lab_diagram.png" | absolute_url }})]({{ "/resources/2018/08/hybrid_lab_diagram.png" | absolute_url }}){:height="75%" width="75%"}

{: .center}
(Click to embiggen)
