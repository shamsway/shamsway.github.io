---
layout: post
theme: jekyll-theme-modernist
title: "Using cloud-init for Customization with VCD and Terraform"
date: 2022-03-10
comments: true
excerpt: This post covers the details of how cloud-init reads its configuration through VMware tools, tips for troubleshooting cloud-init, and some other lessons learned along the way. Of course, I’ll share a working example that deploys a vApp to VCD using cloud-init for customization.<p>
---

Recently I decided to update a blog post I wrote in 2018, [Simple cloud automation with vCD, Terraform, ZeroTier and Slack](https://networkbrouhaha.com/2018/03/vcd-terraform-example/). At a very high level, this blog post walks through deploying a vApp to VCD that is customized to run a script at first boot. In the original blog post, I relied on [Guest Customization](https://docs.vmware.com/en/VMware-vSphere/7.0/com.vmware.vsphere.vm_admin.doc/GUID-58E346FF-83AE-42B8-BE58-253641D257BC.html) with VMware tools to accomplish this. For a variety of reasons - primarily curiosity - I decided to use [cloud-init](https://cloud-init.io/) to run the script instead. Cloud-init is quite flexible and well supported, but in hindsight, my choice led me down quite a rabbit hole. This post covers the details of how cloud-init reads its configuration through VMware tools, tips for troubleshooting cloud-init, and some other lessons learned along the way. Of course, I’ll share a working example that deploys a vApp to VCD using cloud-init for customization.

The act that set the stage for this post is something I have done many times: I uploaded an Ubuntu ISO to a VCD catalog and used it to create a vApp. That vApp, and the single VM it contained, would be added to the same VCD catalog as a vApp template. This was my first mistake, but it took me several hours to figure out why. 

{: .center}
![](https://media.giphy.com/media/xUPGcl3ijl0vAEyIDK/giphy.gif)

Before we get into that, let’s level set on how cloud-init works.

# The Basics of cloud-init

Here is how cloud-init describes itself:

"Cloud-init is the industry standard multi-distribution method for cross-platform cloud instance initialization. 
It is supported across all major public cloud providers, provisioning systems for private cloud infrastructure, and bare-metal installations.” 
-[https://cloudinit.readthedocs.io/](https://cloudinit.readthedocs.io/)


Taking a look at [the provided configuration examples](https://cloudinit.readthedocs.io/en/latest/topics/examples.html) makes it clear what the capabilities are:

* Add/configure users
* Create files
* Install or update software
* Configure networking
* Configure Certificate Authorities
* Run scripts/arbitrary commands
* And [much more](https://cloudinit.readthedocs.io/en/latest/topics/modules.html)

The typical scenario for cloud-init is that a config file is supplied when a server boots, is read by cloud-init and executed. The cloud-init docs refer to the config file as `user-data`. So, how is `user-data` supplied? The details vary, but a datasource is the vehicle to deliver configuration files cloud-init. Cloud-init supports several [datasources](https://cloudinit.readthedocs.io/en/latest/topics/datasources.html) to deliver `user-data` (there are datasources available for major cloud providers), but in a VMware environment the most promising options are [OVF](https://cloudinit.readthedocs.io/en/latest/topics/datasources/ovf.html) and [VMware](https://cloudinit.readthedocs.io/en/latest/topics/datasources/vmware.html).

* The [VMware datasource docs](https://cloudinit.readthedocs.io/en/latest/topics/datasources/vmware.html) state that it supports `GuestInfo` keys for supplying `user-data`. `GuestInfo` is metadata in the form of key/value pairs set in a VM’s `extraConfig` property, which can be read by VMware tools. As long as this metadata can be set via the VCD Terraform provider, this sounds like the datasource that would be used by cloud-init.
* The [OVF datasource docs](https://cloudinit.readthedocs.io/en/latest/topics/datasources/ovf.html) state "The OVF Datasource provides a datasource for reading data from on an Open Virtualization Format ISO transport." That sounds less promising. I’m not interested in building an ISO to bootstrap cloud-init.

Queue my surprise when I finally got cloud-init working, and the logs indicated that it used the OVF datasource. The datasource used by cloud-init can be checked with the `cloud-id` command, and this was the output I received:


```
ubuntu@ubuntu-impish-21:~$ cloud-id
ovf
```

Since all of the cloud-init code is available on GitHub, it’s not too difficult to see how the various data sources work. After a bit of snooping, it’s clear that the OVF datasource also reads the `extraConfig` metadata through VMware tools. In this case, it appears that the cloud-init docs are out of date. That was one of many valuable lessons during this process. Let me share two important ones with you.


# Lesson #1: Check GitHub issues

The VCD Terraform Provider docs have a [section on guest customization](https://registry.terraform.io/providers/vmware/vcd/latest/docs/guides/vm_guest_customization), but it doesn’t mention cloud-init specifically. It does show an example of configuring metadata with the provider, so I felt confident that I could supply cloud-init `user-data` with that method. I mentioned in the intro that I made a mistake by attempting to use cloud-init with an Ubuntu server that I built from an ISO. I’m quite sure there is a way to make it work, but I kept hitting roadblocks. Had I skimmed the resolved issues in the VCD Terraform Provider repo, I would have found [this helpful comment](https://github.com/vmware/terraform-provider-vcd/issues/667#issuecomment-844030920):


```
The problem that I had was the OVA machine I tried to use.
A standard version of Ubuntu.
First part to make this working correctly is to download the cloud image at:
http://cloud-images.ubuntu.com/
```


The commenter then goes on to provide a working example of using cloud-init with the VCD Terraform Provider. Normally I do a search through GitHub issues when I’m troubleshooting something. In this case, inexplicably, I did not. If I had read that comment first, I would have saved a lot of time. However, I would not have learned so many useful strategies for troubleshooting cloud-init.

{: .center}
![](https://media.giphy.com/media/3o7aD4ubUVr8EkgQF2/giphy.gif)


# Lesson #2: Use a Cloud Image

I was aware cloud images existed, but I was set in my ways. I’d used a bootable ISO to build a Linux VM template so many times and I didn’t consider that there was an easier option. I also assumed cloud images were purely for cloud providers, and I didn’t bother to check if there was a VMware flavor available. Lesson learned. There’s a great post on using the Ubuntu cloud image on vSphere here: [https://d-nix.nl/2021/04/using-the-ubuntu-cloud-image-in-vmware/](https://d-nix.nl/2021/04/using-the-ubuntu-cloud-image-in-vmware/). That only covers the vSphere side of things, but that post is a great explainer.


# Deploying and Customizing a VCD vApp with Terraform

With those (rather obvious) lessons learned, **let's do this thing**. 

{: .center}
![](https://media.giphy.com/media/tyxovVLbfZdok/giphy.gif)

You will need the following:

* A `cloud-config.yaml` file, containing the cloud-init `user-data`. The file extension is a clue that this is a YAML-formatted file. If you have cloud-init installed locally, you can verify that it is a valid config with `cloud-init devel schema -c cloud-init.yaml`. I highly recommend that you do this.
* A cloud image OVA downloaded on your local workstation. For Ubuntu, these are available at [http://cloud-images.ubuntu.com/](http://cloud-images.ubuntu.com/)

## Creating a Catalog

Creating a catalog in VCD with Terraform is pretty simple. Here is an example:


```tf
resource "vcd_catalog" "mycatalog" {
 org = "my-org"

 name             = "my-catalog"
 description      = "Catalog created by Terraform"
 delete_recursive = "true"
 delete_force     = "true"
}
```



## Uploading an OVA to a Catalog

Similarly, adding the cloud image OVA to the new catalog is straightforward. The upload time will be dependent on the bandwidth available, but the Ubuntu 21.10 cloud image is only about 540 MB.


```tf
resource "vcd_catalog_item" "ubuntu-2110-cloud" {
 org     = vcd_catalog.mycatalog.org
 catalog = vcd_catalog.mycatalog.name

 name                 = "ubuntu-2110-cloud"
 description          = "Ubuntu 21.10 cloud image"
 ova_path             = "./impish-server-cloudimg-amd64.ova"
 upload_piece_size    = 10
}
```



## Deploying the vApp

This is the final step, and it requires a few different Terraform resources, but it’s not too difficult to follow.


```tf
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
 vapp_name     = vcd_vapp.ubuntu.name
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
   "user-data" = base64encode("cloud-config.yaml")
 }
}

```

* The `vcd_vapp` resource creates the new vApp that contain a single VM running the cloud image template in my catalog 
* The `vcd_vapp_org_network` resource attaches an existing org network to the new vApp
* The `vcd_vapp_vm `resource provides all of the configuration for the single VM that will be in the new vApp, including the cloud-init `user-data`

Most of the config in the `vcd_vapp_vm `resource is what you’d expect - compute, memory, and networking settings. The `guest_properties` section is the important bit. It configures the `extraConfig` property on the VM, which is where cloud-init will read the `user-data` from. Notice that the [base64encode()](https://www.terraform.io/language/functions/base64encode) function is used to convert the `cloud-config.yaml` file into a single, long, encoded string. This is how cloud-init expects the `user-data` to be passed over. 

If you have values in your `cloud-config.yaml` file that you need to change on the fly, like credentials or API keys, you can use the [templatefile()](https://www.terraform.io/language/functions/templatefile) function to insert those values into the config file before encoding it. It’s possible that `user-data` will contain sensitive data and it is trivial to decode base64. In a production environment, you should remove the `user-data` from the VM after first boot.

I traveled down a winding road to get here, but I finally assembled all of the pieces needed to do what I set out for originally: [update an old blog post]({% link _posts/2022-03-16-vcd-verraform-example.md %}). If all you needed was some tips on using cloud-init with Terraform and VCD, you can go along your merry way. Stick around if you want some tips on troubleshooting cloud-init.


# Troubleshooting cloud-init

Here are some basic troubleshooting steps for cloud-init with vSphere/VCD:



* Make sure you have a recent version of VMware Tools installed. This is required to read the metadata associated with the VM.
* Make sure you are using a cloud image _or_ you have taken the steps to ensure that your VM is properly configured to work with cloud-init. You can see an example of this with the `govc` tool at [https://github.com/vmware/govmomi/blob/master/govc/USAGE.md#vmchange](https://github.com/vmware/govmomi/blob/master/govc/USAGE.md#vmchange). 
* Verify that VMware Tools is able to access VM metadata. You can use the command `vmware-rpctool 'info-get guestinfo.ovfEnv' `to check this. If the command returns a slew of XML, it is working as expected.
* Verify the VM metadata. You can view this in vSphere by browsing to the `VM -> Settings -> vApp Options`. Base64 encoded `user-data` should be visible under the properties section, and you can click the “View OVF Environment” button to see the XML formatted version of the metadata. This is the same information you should see from running the `vmware-rpctool` command on the VM. You can also view these properties in VCD by viewing the Guest Properties section in the VM properties.
* Check the cloud-init logs at `/var/log/cloud-init.log` and `/var/log/cloud-init-output.log` for errors and warnings.
* Run `cloud-id` to verify that the correct datasource is being used. If the output is `fallback` or `none`, cloud-init was not able to detect the datasource.
* `ds-identify` is used by cloud-init to find all available datasources. Check the logs at `/run/cloud-init/ds-identify.log` to see why the desired datasource is not found.
* While troubleshooting, you can completely reset cloud-init with `sudo cloud-init clean --logs`, and reboot to have cloud-init run again. This saves time over redeploying a template.


# Resources



* Terraform VCD provider: [https://registry.terraform.io/providers/vmware/vcd/3.5.1](https://registry.terraform.io/providers/vmware/vcd/3.5.1)
* `vcd_catalog `resource: [https://registry.terraform.io/providers/vmware/vcd/latest/docs/resources/catalog](https://registry.terraform.io/providers/vmware/vcd/latest/docs/resources/catalog)
* `vcd_catalog_item `resource: [https://registry.terraform.io/providers/vmware/vcd/latest/docs/resources/catalog_item](https://registry.terraform.io/providers/vmware/vcd/latest/docs/resources/catalog_item)
* `vcd_vapp `resource: [https://registry.terraform.io/providers/vmware/vcd/latest/docs/resources/vapp](https://registry.terraform.io/providers/vmware/vcd/latest/docs/resources/vapp)
* `vcd_vapp_org_network `resource: [https://registry.terraform.io/providers/vmware/vcd/latest/docs/resources/vapp_org_network](https://registry.terraform.io/providers/vmware/vcd/latest/docs/resources/vapp_org_network)
* `vcd_vapp_vm `resource: [https://registry.terraform.io/providers/vmware/vcd/latest/docs/resources/vapp_vm](https://registry.terraform.io/providers/vmware/vcd/latest/docs/resources/vapp_vm)
* OVF Runtime Environment: [https://williamlam.com/2012/06/ovf-runtime-environment.html](https://williamlam.com/2012/06/ovf-runtime-environment.html)
* Using the Ubuntu Cloud Image in VMware: [https://d-nix.nl/2021/04/using-the-ubuntu-cloud-image-in-vmware/](https://d-nix.nl/2021/04/using-the-ubuntu-cloud-image-in-vmware/)
* Terraform, vSphere, and Cloud-Init oh my! [https://grantorchard.com/terraform-vsphere-cloud-init/](https://grantorchard.com/terraform-vsphere-cloud-init/)
* Cloud-init config examples: [https://cloudinit.readthedocs.io/en/latest/topics/examples.html](https://cloudinit.readthedocs.io/en/latest/topics/examples.html)