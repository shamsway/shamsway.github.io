---
layout: post
theme: jekyll-theme-modernist
title: "2022 Update: Simple Cloud Automation with VCD, Terraform, ZeroTier and Slack"
date: 2022-03-10
comments: true
excerpt: This post covers the details of how cloud-init reads its configuration through VMware tools, tips for troubleshooting cloud-init, and some other lessons learned along the way. Of course, I’ll share a working example that deploys a vApp to VCD using cloud-init for customization.<p>
---

In 2018 I wrote a blog titled [Simple cloud automation with vCD, Terraform, ZeroTier and Slack](https://networkbrouhaha.com/2018/03/vcd-terraform-example/). A lot has changed since I wrote that post, so it’s time to update it. The goal is still the same: deploy a VM (inside a vApp) in Cloud Director and automate network connectivity with ZeroTier. Slack is used to monitor the progress and display the IP address assigned by ZeroTier. Overall, I want to be able to deploy a VM that has outbound internet connectivity and be able to connect to it without having to configure any firewall rules, NAT, or SSL/IPsec VPN.

I did make some adjustments to my approach while preparing to write this post. Instead of relying on [Guest Customization](https://docs.vmware.com/en/VMware-vSphere/7.0/com.vmware.vsphere.vm_admin.doc/GUID-58E346FF-83AE-42B8-BE58-253641D257BC.html) with VMware tools, I chose to use [cloud-init](https://cloud-init.io/). This went so poorly that I wrote a dedicated post on it 😂: [Using cloud-init for Customization with VCD and Terraform](https://networkbrouhaha.com/2022/03/cloud-init-vcd/). VCD also has a completely different [Terraform provider](https://registry.terraform.io/providers/vmware/vcd/latest) than the one I demoed in 2018, which I will dig into at the end of this post.

# Tools Used and Prerequisites

* [VMware Cloud Director](https://www.vmware.com/products/cloud-director.html) - VMware’s cloud service delivery platform, typically used by service providers in the VMware Cloud Provider Program. I used VCD 10.3 in my lab when using the Terraform code you will see below.
* [HashiCorp Terraform](https://terraform.io/) - An open-source tool written in Go, Terraform allows users to define infrastructure as code. Many public cloud [providers](https://registry.terraform.io/browse/providers) are supported in Terraform, as well as on-prem infrastructure like [vSphere](https://registry.terraform.io/providers/hashicorp/vsphere/latest) and [NSX-T](https://registry.terraform.io/providers/vmware/nsxt/latest). The Terraform provider for VCD is available at [https://registry.terraform.io/providers/vmware/vcd/latest](https://registry.terraform.io/providers/vmware/vcd/latest).
* [ZeroTier](https://www.zerotier.com/) - The ZeroTier docs state that “ZeroTier is a smart Ethernet switch for planet Earth.” ZeroTier uses an agent to provide connectivity between endpoints connected to the same ZeroTier network. Anyone can create a free account on the ZeroTier website and create multiple networks. Endpoints connected to ZeroTier are managed through the web portal (or API). In other words, ZeroTier is a simple, free[^1], fast[^2] VPN. If you’re wondering how ZeroTier works, check out their awesome [whitepaper](https://docs.zerotier.com/zerotier/manual). My friend and uber-network nerd [Tony Efantis](https://twitter.com/showipintbri) provides a deep dive into ZeroTier on YouTube: [https://www.youtube.com/watch?v=Lao9T_RQTak](https://www.youtube.com/watch?v=Lao9T_RQTak)
* [Slack](https://slack.com/) - I’m assuming everyone is familiar with Slack by now. For this example, Slack is used to provide visibility into the process of connecting a new VM to ZeroTier. Slack’s free tier is great for testing simple automation and receiving notifications via webhooks.
* [GitHub](https://github.com/) - I’m hosting scripts on GitHub, but any web host could fill this need. If you choose another host, you should still use Git for version control for Terraform code and other scripts. The current script I’m using is at [https://github.com/shamsway/zerotier-installer](https://github.com/shamsway/zerotier-installer), and it is a simplified and modified version of the install script provided by ZeroTier at [https://install.zerotier.com/](https://install.zerotier.com/).

Before deploying anything with Terraform, I installed ZeroTier on my local workstation, uploaded an Ubuntu cloud image OVA to my VCD catalog, and configured an incoming webhook for Slack. My VCD environment is preconfigured to allow outbound internet traffic, but nothing else. 

# Terraform Example

Below is the `main.tf `file to create a vApp, attach an existing Org network to the vApp, and clone a VM into the vApp using cloud-init for customization. 

```tf
terraform {
  required_providers {
    vcd = {
      source = "vmware/vcd"
    }
  }
}

variable "ztnetwork" {
  type        = string
  description = "ZeroTier Network to join"
}

variable "ztapi" {
  type        = string
  sensitive   = true
  description = "ZeroTier API Access Token"
}

variable "slack_webhook_url" {
  type        = string
  description = "Slack webhook URL"
  default     = ""
}

variable "vcd_vm_name" {
  type = string
  description = "Name of new vApp created from template"
}

resource "vcd_vapp" "ubuntu" {
  org  = "my-org"
  vdc  = "my-vdc"
  name = "ubuntu"

  power_on = true
}

resource "vcd_vapp_org_network" "ubuntu-network" {
  org = "my-org"
  vdc = "my-vdc"

  vapp_name        = vcd_vapp.ubuntu.name
  org_network_name = "org-network"
}

resource "vcd_vapp_vm" "ubuntu" {
  org           = "my-org"
  vdc           = "my-vdc"
  vapp_name     = "ubuntu"
  catalog_name  = "my-catalog"
  template_name = "ubuntu-2110-cloud"
  name          = "ubuntu-vm"
  memory        = 4096
  cpus          = 1
  os_type       = "ubuntu64Guest"
  power_on      = true

  network {
    type               = "org"
    name               = "org-network"
    ip_allocation_mode = "MANUAL"
    ip                 = "192.168.1.10"
  }

  guest_properties = {
    "user-data" = base64encode(templatefile("cloud-config.yaml", { ztnetwork = var.ztnetwork, ztapi = var.ztapi, slack_webhook_url = var.slack_webhook_url, hostname = var.vcd_vm_name }))
  }
}
```

Most of this is straightforward, but the magic happens in the `guest_properties` block of the `vcd_vapp_vm` resource. The `user-data` property contains a base 64 encoded version of my cloud-init configuration. You can see that the `templatefile()` function is used to insert some values needed for the ZeroTier install script: the ZeroTier network to connect to, an API key for ZeroTier, the webhook URL for Slack, and the VM hostname.

Here is my cloud-config.yaml, which performs the customization of the VM upon first boot:

```yaml
#cloud-config

hostname: ${hostname}
users:
 - name: ubuntu
   sudo: ["ALL=(ALL) NOPASSWD:ALL"]
   groups: [sudo]
   shell: /bin/bash
   ssh_authorized_keys:
    - ssh-rsa alongstringthatisansshkey
manage_resolv_conf: true
packages:
 - python3-pip
 - jq
runcmd:
 - export ZTNETWORK=${ztnetwork}
 - export ZTAPI=${ztapi}
 - export SLACK_WEBHOOK_URL=${slack_webhook_url}
 - wget https://raw.githubusercontent.com/shamsway/zerotier-installer/master/zerotier-installer.sh
 - chmod +x zerotier-installer.sh
 - ./zerotier-installer.sh
 - rm zerotier-installer.sh
final_message: "The system is ready and prepped (took $UPTIME seconds)"
```

This cloud-init config will configure the local ubuntu user with sudo privileges, disable password-based logins, add my desired SSH key and install some necessary packages. The `runcmd` block is the bit that actually downloads my ZeroTier installer from GitHub and executes it, connecting the VM to my ZeroTier network and providing output to Slack.

Now, let’s see this in action.

# Workflow

The output from `terraform apply` looks just as you’d expect if you’ve ever seen Terraform run:

```
Plan: 3 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

vcd_vapp.ubuntu-zt: Creating...
vcd_vapp.ubuntu-zt: Still creating... [10s elapsed]
vcd_vapp.ubuntu-zt: Creation complete after 16s [id=urn:vcloud:vapp:db4d4ee7-b171-45dc-a98a-67cd717db127]
vcd_vapp_org_network.ubuntu-zt-network: Creating...
vcd_vapp_vm.ubuntu: Creating...
vcd_vapp_org_network.ubuntu-zt-network: Creation complete after 5s [id=urn:vcloud:network:1b61037f-dc6d-4ae5-aefc-59962de1e647]
vcd_vapp_vm.ubuntu: Still creating... [10s elapsed]
[snip]
vcd_vapp_vm.ubuntu: Creation complete after 1m58s [id=urn:vcloud:vm:d20caca3-8b80-45da-8435-c4d44c988ccb]

Apply complete! Resources: 3 added, 0 changed, 0 destroyed.
```

VCD creates the vApp, clones a template VM into the vApp, and powers it on. When the VM boots, cloud-init runs and executes each step specified in cloud-config.yaml, which will ultimately connect the new VM to my ZeroTier network. API calls are used to authorize the new VM to connect to my ZeroTier network automatically, so I don’t have to go in and manually accept the new VM in the ZeroTier portal. The process of connecting the VM to ZeroTier is output to Slack, and once complete I can grab the provided IP and immediately connect to the new VM.

{: .center}
[![](/resources/2022/03/vcd-automation-slack.png)](/resources/2022/03/vcd-automation-slack.png){:.drop-shadow}

```
user@ubuntu:~$ ssh ubuntu@172.29.189.205
The authenticity of host '172.29.189.205 (172.29.189.205)' can't be established.
ECDSA key fingerprint is SHA256:sOGaDtQ6D6bvIhmr/YhKt6Olt9EsVNRNGAomfVuIW1o.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '172.29.189.205' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 21.10 (GNU/Linux 5.13.0-28-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Wed Mar 16 23:47:03 UTC 2022

  System load:  0.03              Processes:                   138
  Usage of /:   23.0% of 9.52GB   Users logged in:             0
  Memory usage: 6%                IPv4 address for ens192:     192.168.1.10
  Swap usage:   0%                IPv4 address for ztmjfe5xok: 172.29.189.205

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

ubuntu@ubuntu-impish-21:~$ ping google.com
PING google.com (142.250.191.238) 56(84) bytes of data.
64 bytes from ord38s32-in-f14.1e100.net (142.250.191.238): icmp_seq=1 ttl=113 time=3.54 ms
64 bytes from ord38s32-in-f14.1e100.net (142.250.191.238): icmp_seq=2 ttl=113 time=3.60 ms
```

Notice that SSH key-based authentication is used instead of a password, which is common practice for instances running in the cloud.

So there it is - a VM deployed into VCD and automatically connected to ZeroTier, making it available without having to configure any sort of inbound firewall rules, NAT, or IPSec/SSL VPN. 

# State of the VCD Terraform Provider in 2022

When I wrote about this in 2018, the VCD Terraform provider was written by HashiCorp and was based on a go library named `govcloudair`. This library was not maintained by VMware and it was not actively developed, meaning that the VCD provider supported a limited number of features. I am happy to report that the [current VCD provider](https://registry.terraform.io/providers/vmware/vcd/latest) is in a much better state. The provider is actively developed by VMware along with the underlying go library, [go-vcloud-director](https://github.com/vmware/go-vcloud-director). As of March 2022, there were **over 2 million installs** of the VCD Terraform provider, and new features are being added regularly. Many of the workarounds and caveats I mentioned in my 2018 post are no longer required. Huzzah!

{: .center}
![](https://media.giphy.com/media/d7qN2d6ktQphUeDoQ4/giphy.gif)

# Final Thoughts

Here are a few random thoughts/potential improvements:

* This same workflow could be used in any cloud environment. It would require outbound internet access to be enabled, and cloud-init is well supported across cloud providers. Each cloud provider’s Terraform provider documentation should contain examples for using cloud-init.
* Cloud-init could be used to install ZeroTier and send the output to Slack, but I didn’t want to spend the time to convert my install script. Initially, I used a script hosted on GitHub because there was a limit on the size of a script that can be used with Guest Customization, but cloud-init does not have that limit. I may convert my install script over to cloud-init at a later date.
* The ZeroTier install script uses [https://github.com/philippbosch/slack-webhook-cli](https://github.com/philippbosch/slack-webhook-cli) to send messages to Slack, which requires Python to be installed. Installing Python adds time to the process. Sending messages to Slack is just a webhook, so a bash script could be used instead. This would remove the requirement to install Python and the whole process would be a bit faster.


# Resources

* VCD Terraform provider: [https://registry.terraform.io/providers/vmware/vcd/latest](https://registry.terraform.io/providers/vmware/vcd/latest)
* Go-vcloud-director library: [https://github.com/vmware/go-vcloud-director](https://github.com/vmware/go-vcloud-director)
* ZeroTier documentation: [https://docs.zerotier.com/zerotier/manual/](https://docs.zerotier.com/zerotier/manual/)
* ZeroTier overview on Wikipedia: [https://en.wikipedia.org/wiki/ZeroTier](https://en.wikipedia.org/wiki/ZeroTier)
* How Does ZeroTier Actually Work? [https://www.youtube.com/watch?v=Lao9T_RQTak](https://www.youtube.com/watch?v=Lao9T_RQTak)

[^1]: GPL license / Up to 100 devices / Requires license to embed in commercial products.
[^2]: Quick setup, but actual traffic may proxy through ZeroTier servers. There is no throughput guarantee.