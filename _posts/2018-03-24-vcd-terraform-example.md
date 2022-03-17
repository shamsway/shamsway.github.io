---
layout: post
theme: jekyll-theme-modernist
title: "Simple cloud automation with vCD, Terraform, ZeroTier and Slack"
date: 2018-03-24
comments: true
crosspost_to_medium: true
excerpt: Earlier this month I had the opportunity to present at the Lexington VMUG on cloud automation. I used this opportunity to pull together a few tools I had been experimenting with and combine them for a simple cloud automation example.<p>
---

Earlier this month I had the opportunity to present at the [Lexington VMUG](https://community.vmug.com/communities/localcommunityhome?CommunityKey=84170a74-5453-4c77-abba-74124cd7dd42) on cloud automation. I used this opportunity to pull together a few tools I had been experimenting with and combine them for a simple cloud automation example.

## Goal

My goal with this demo was to deploy a VM in vCloud Director and automate network connectivity via ZeroTier. During testing I decided to include Slack so I could monitor the progress of my scripts. I chose ZeroTier for connectivity because of it's simplicity. Different cloud providers handle connectivity in different ways. For example, vCD defaults to a "deny any any" firewall ruleset, making it a good fit for ZeroTier. You should follow the best practices of your cloud provider if you're trying this on your own.

## Tools used

* [VMware vCloud Director](https://www.vmware.com/products/vcloud-director.html) - VMware's public cloud solution for service providers in the VMware Cloud Provider Program. We have vCD 9.0 installed in our lab at work, which is where I developed this automation.
* [HashiCorp Terraform](https://www.terraform.io/) - An open source tool written in Go, Terraform allows users to
define infrastructure as code. Many public cloud [providers](https://www.terraform.io/docs/providers/) are supported in Terraform, as well as on prem infrastructure like vSphere.
* [ZeroTier](https://www.zerotier.com/) - “ZeroTier delivers the capabilities of VPNs, SDN, and SD-WAN with a single system. Manage all your connected resources across both local and wide area networks as if the whole world is a single data center.” In other words: simple, free[^1], fast[^2] VPN.
* [Slack](https://slack.com) - This cloud based collaboration tool is an easy way to get feedback from cloud-based infrastructure. Slack’s free tier is great for testing simple automation and receiving notifications from your test/dev projects.
* [GitHub](https://github.com) - I'm hosting scripts on GitHub, but any web host could fill this need. If you choose another host, you should still use Git for version control.

## Prerequisites

Prior to my demo I had installed Terraform and the ZeroTier client on my laptop, created a Linux template in vCD, configured Slack with incoming web-hooks, and uploaded my [ZeroTier install script](https://github.com/shamsway/zerotier-installer) to GitHub. My vCD Org VDC is already configured to allow outbound internet traffic. Sanitized versions of my Terraform files are below.

vCD Terraform configuration - `labvcd.tf`:
```
provider "vcd" {
  user                 = "[username]"
  password             = "[password]"
  org                  = "[org name]"
  url                  = "[vCD URL]/api"
  vdc                  = "[org VDC name]"
  allow_unverified_ssl = "true"
}
```

Terraform syntax to clone a linux template - `tf-demo.tf`:
```
data "template_file" "init" {
  template = "${file("${path.cwd}/setup.sh")}"

  vars {
    ztnetwork   = "[ZeroTier Network ID]"
    ztapi       = "[ZeroTier API Key]"
    slack_webhook_url = "[Slack Webhook URL]"
  }
}

resource "vcd_vapp" "tf-demo" {
  name		= "tf-demo"
  power_on     	= "false"
  network_name 	= "[Org VCD network name]"
  ip            = "allocated"
  catalog_name  = "[Catalog name]"
  template_name = "[Template name]"
  memory        = 4096
  cpus          = 1  
  initscript    = "${data.template_file.init.rendered}"
}
```

The `initscript` line tells Terraform to parse `setup.sh` (below), and sets the contents as the guest customization init script.

VM bootstrap script - `setup.sh`:
```
#!/bin/bash
if [ x$1 = x"precustomization" ]; then
echo "Started doing pre-customization steps..."
echo "Finished doing pre-customization steps."
elif [ x$1 = x"postcustomization" ]; then
echo "Started doing post-customization steps..."
apt-get update && apt-get upgrade -y && apt-get install -y openssh-server jq
sudo systemctl enable ssh
export ZTNETWORK=${ztnetwork}
export ZTAPI=${ztapi}
export SLACK_WEBHOOK_URL=${slack_webhook_url}
wget https://raw.githubusercontent.com/shamsway/zerotier-installer/master/zerotier-installer.sh
chmod +x zerotier-installer.sh
echo "Installing and configuring ZeroTier"
./zerotier-installer.sh
rm zerotier-installer.sh
echo "Finished doing post-customization steps."
fi
```

This script is run via guest customization when the VM powers on for the first time. It runs apt-get update, installs and enables OpenSSH server, sets some environment variables and downloads the ZeroTier installer from GitHub.

Normally I'd put some indentation in the script to make it easier to read, but doing so with guest customization caused the script to break. The `if [ x$1 = x"precustomization" ]; then` and `elif [ x$1 = x"postcustomization" ]; then` lines are mentioned in VMware documentation as the way to control whether the script is run during pre-customization or post-customization. The [documentation from VMware](https://pubs.vmware.com/vcd-820/index.jsp?topic=%2Fcom.vmware.vcloud.user.doc%2FGUID-724EB7B5-5C97-4A2F-897F-B27F1D4226C7.html) uses `==` instead of `=` for comparison, but this failed in the Ubuntu template I was using. Thankfully somebody [blogged about this](http://markhneedham.com/blog/2012/08/06/vcloud-guest-customization-script-postcustomization-unexpected-operator/) error back in 2012.

Using Terraform with vCD 9.0 requires using the latest provider code from GitHub, available at https://github.com/terraform-providers/terraform-provider-vcd. Since I am also using the template provider (https://github.com/terraform-providers/terraform-provider-template), I had to download and install that as well.

## Workflow

Initialize Terraform. `-plugin-dir` instructs Terraform to use the newer provider code that I downloaded.

```
$ terraform init -plugin-dir /usr/local/go/bin/
```

Run Terraform. `parallelism=1` instructs Terraform to run one task at time. By default, it will run tasks simultaneously. This restriction is covered in the [vCD Terraform provider docs](https://www.terraform.io/docs/providers/vcd/r/vapp_vm.html)

```
$ terraform apply -parallelism=1

data.template_file.init: Refreshing state...

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  + vcd_vapp.tf-demo
      id:            <computed>
      catalog_name:  "MattLEX"
      cpus:          "1"
      href:          <computed>
      initscript:    "[snip]"
      ip:            "allocated"
      memory:        "4096"
      name:          "tf-demo"
      network_name:  "MattLEX"
      power_on:      "false"
      template_name: "ubuntu_16.04_small"


Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

vcd_vapp.tf-demo: Creating...
  catalog_name:  "" => "MattLEX"
  cpus:          "" => "1"
  href:          "" => "<computed>"
  initscript:    "" => "[snip]"
  ip:            "" => "allocated"
  memory:        "" => "4096"
  name:          "" => "tf-demo"
  network_name:  "" => "MattLEX"
  power_on:      "" => "false"
  template_name: "" => "ubuntu_16.04_small"
vcd_vapp.tf-demo: Still creating... (10s elapsed)
[snip]
vcd_vapp.tf-demo: Still creating... (7m30s elapsed)
vcd_vapp.tf-demo: Creation complete after 7m32s (ID: tf-demo)
```

vCloud Director clones the template based on my Terraform config. I instruct Terraform to not power on the VM after creation. This is due to a bug in the vCD Terraform provider that tries to apply guest customization after power on if done during cloning. This is not ideal and could be automated via an API call to vCD, but I'm manually powering on the VM.

When the VM boots, guest customization runs the init script (guest customization scripts are limited to 1500 characters, so keep that in mind). The init script downloads a ZeroTier [installer script](https://github.com/shamsway/zerotier-installer/blob/master/zerotier-installer.sh) from GitHub. It is based on the official ZeroTier linux install script at http://intstall.zerotier.com/, but pared down to work on Ubuntu 16.04 only. I've also modified it automatically authorize the new VM via ZeroTier API, and send feedback to Slack.

As the script runs it posts messages in Slack. The final message displays the ZeroTier IP assigned to the VM.

![]({{ "/resources/2018/03/vcd_slack.png" | absolute_url }})

Once I have the ZeroTier IP, I can SSH to the VM to continue whatever setup I would normally complete.

```
$ ssh user@10.244.80.210

Welcome to Ubuntu 16.04.4 LTS (GNU/Linux 4.4.0-116-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

2 packages can be updated.
0 updates are security updates.

Last login: Fri Feb 23 17:27:57 2018 from 192.168.0.4
user@tf-demo:~$ ping www.google.com
PING www.google.com (172.217.6.4) 56(84) bytes of data.
64 bytes from ord38s01-in-f4.1e100.net (172.217.6.4): icmp_seq=1 ttl=47 time=19.4 ms
64 bytes from ord38s01-in-f4.1e100.net (172.217.6.4): icmp_seq=2 ttl=47 time=19.4 ms

```

When I'm finished with the VM, I can destroy it with Terraform. Note that this does not de-authorize the VM in ZeroTier, which is something you would want to do if you were using this in production or on a frequent basis.

```
$ terraform destroy -parallelism=1
data.template_file.init: Refreshing state...
vcd_vapp.tf-demo: Refreshing state... (ID: tf-demo)

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  - destroy

Terraform will perform the following actions:

  - vcd_vapp.tf-demo

Plan: 0 to add, 0 to change, 1 to destroy.

Do you really want to destroy?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: yes

vcd_vapp.tf-demo: Destroying... (ID: tf-demo)
vcd_vapp.tf-demo: Still destroying... (ID: tf-demo, 10s elapsed)
vcd_vapp.tf-demo: Destruction complete after 19s

Destroy complete! Resources: 1 destroyed.
```

## Lessons Learned

vCD support in Terraform is not great, but appears to be getting better. The current Terraform [vCD provider](https://github.com/terraform-providers/terraform-provider-vcd) is based on an older vCD/vCloud Air [Go library](https://github.com/UKCloud/govcloudair) developed for vCD 5.5. There is a [plan](https://github.com/terraform-providers/terraform-provider-vcd/blob/master/v2Plan.md) to develop better support for newer releases of vCD. VMware has also [released their own vCD Terraform Provider](https://github.com/vmware/terraform-provider-vcloud-director), but it is not clear if this will be included in future Terraform releases. One interesting note is the new provider does away with the native Go library in favor of using gRPC + [pyvcloud](https://github.com/vmware/pyvcloud). I am participating in a VMware Cloud Provider Technical Advisory Board meeting next month and I will attempt to get some clarification on the future of vCD with Terraform.

Looking back on this exercise it is clear that there are many hoops to jump through to automate vCD and Terraform. vCD is not _widely_ deployed, but it is used at some of the "second tier" and smaller cloud providers. Over the last two years VMware has recommitted to improving vCD, with the 9.0 and 9.1 releases providing huge improvements to the platform. I believe vCD will continue to improve, as well as gain traction and visibility. On the other hand, there are many other public cloud options, and all of them have robust support in Terraform. Anyone can take the Terraform configuration I wrote for vCD and adapt it to another provider with minimal effort.

## Final Thoughts

* Think about networking connectivity to the cloud - have a plan. Public/HTTPS, IPSec VPN, SSL VPN, ZeroTier, SD-WAN and Direct Connect are all feasible options.
* Find a cloud provider you can use for practice - AWS, Digital Ocean, Azure, or vCD.
* Practice building infrastructure with Terraform or similar tool.
* Use git version control for your scripts.
* Remember: *Don’t put your passwords and API keys in your public github repo*!

[^1]: GPL license / Up to 100 devices / Requires license to embed in commercial products.
[^2]: Quick setup, but actual traffic may proxy through ZeroTier servers. There is no throughput guarantee.
