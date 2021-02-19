---
layout: post
theme: jekyll-theme-modernist
title: "Intro to Google Cloud VMware Engine – Connecting a VPC to GCVE"
date: 2021-02-19
comments: true
excerpt: This post will show the process of connecting a VPC to your GCVE environment, and we will use Terraform to do the vast majority of the work.<p>
---

My [previous post]({% link _posts/2021-02-04-gcve-sddc-with-hcx.md %}) walked through deploying an SDDC in Google Cloud VMware Engine (GCVE). This post will show the process of connecting a VPC to your GCVE environment, and we will use Terraform to do the vast majority of the work. The diagram below shows the basic concept of what I will be covering in this post. Once connected, you will be able to communicate from your VPC to your SDDC and vice versa. If you would like to complete this process using the cloud console instead of Terraform, see [Setting up private service access](https://cloud.google.com/vmware-engine/docs/networking/howto-setup-private-service-access) in the VMware Engine documentation.

{: .center}
[![](/resources/2021/02/gcve-vpc-peeing.png)](/resources/2021/02/gcve-vpc-peeing.png){:.drop-shadow}

I’m assuming you have a working SDDC deployed in VMware Engine and some basic knowledge of how Terraform works so you can use the provided Terraform examples. If you have not yet deployed an SDDC, please do so before continuing. If you need to get up to speed with Terraform, browse over to [https://learn.hashicorp.com/terraform](https://learn.hashicorp.com/terraform). All of the code referenced in this post will be available at [https://github.com/shamsway/gcp-terraform-examples](https://github.com/shamsway/gcp-terraform-examples) in the `gcve-network` sub-directory. You will need to have git installed to clone the repo, and I highly recommend using [Visual Studio Code](https://github.com/microsoft/vscode) with the Terraform add-on installed to view the files.

# Private Service Access Overview

GCVE SDDCs can establish connectivity to native GCP services with [private services access](https://cloud.google.com/vpc/docs/private-services-access).  This service can be used to establish connectivity from a VPC to a third-party “service producer,” but in this case, it will simply plumb connectivity between our VPC and SDDC. Configuring private services access requires allocating one or more reserved ranges that cannot be used in your local VPC network. In this case, we will supply the ranges that we have allocated for our SDDC networks. Doing this prevents issues with overlapping IP ranges.

# Leveraging Terraform for Configuration

I have provided Terraform code will do the following:

* Create a VPC network
* Create a subnet in the new VPC network that will be used to communicate with GCVE
* Create two Global Address pools that will be used to reserve addresses used in GCVE
* Create a private connection in the new VPC, using the two address pools as reserved ranges
* Enable import and export of custom routes for the VPC

After Terraform completes configuration, you will be able to establish peering with the new VPC in GCVE.  To get started, clone the example repo with `git clone https://github.com/shamsway/gcp-terraform-examples.git`, then change to the `gcve-network` sub-directory. You will find these files:

* `main.tf` – Contains the primary Terraform code to complete the steps mentioned above
* `variables.tf` – Defines the input variables that will be used in `main.tf`
* `terraform.tfvars` – Supplies values for the input variables defined in `variables.tf`

Let’s take a look at what is happening in `main.tf`, then we will supply the necessary variables in `terraform.tfvars` and run Terraform. You will see `var.[name]` appear over and over in the code, as this is how Terraform references variables. You may think it would be easier to place the desired values directly into `main.tf` instead of defining and supplying variables, but it is worth the time to get used to using variables. Hardcoding values in your code is rarely a good idea, and most Terraform code that I have consumed from other authors use variables heavily.

## main.tf Contents

```tf
provider "google" {
  project = var.project
  region  = var.region
  zone    = var.zone
}
```
The file begins with a provider block, which is common in Terraform. This block defines the Google Cloud project, region, and zone in which Terraform will create resources. The values used are specified in `terraform.tfvars`, which is the same method we will use throughout this example.

```tf
resource "google_compute_network" "vpc_network" {
  name                    = var.network_name
  description             = var.network_descr
  auto_create_subnetworks = false
}
```

The first resource block creates a new VPC in the region and zone specified in the provider block. Setting `auto_create_subnetworks` to `false` specifies that we want a custom VPC instead of auto-creating subnets for each region.

```tf
resource "google_compute_subnetwork" "vpc_subnet" {
  name          = var.subnet_name
  ip_cidr_range = var.subnet_cidr
  region        = var.region
  network       = google_compute_network.vpc_network.id
}
```

The next block creates a subnet in the newly created VPC. Notice that the last line references `google_compute_network.vpc_network.id` for the network value, meaning that it uses the ID value of the VPC created by Terraform.

```tf
resource "google_compute_global_address" "private_ip_alloc_1" {
  name          = var.reserved1_name
  address       = var.reserved1_address
  purpose       = var.address_purpose
  address_type  = var.address_type
  prefix_length = var.reserved1_address_prefix_length
  network       = google_compute_network.vpc_network.id
}
```

This block and the following block (`google_compute_global_address.private_ip_alloc_2`) create a private IP allocation used for the private services configuration.

```tf
resource "google_service_networking_connection" "gcve-psa" {
  network                 = google_compute_network.vpc_network.id
  service                 = var.service
  reserved_peering_ranges = [google_compute_global_address.private_ip_alloc_1.name, google_compute_global_address.private_ip_alloc_2.name]
  depends_on              = [google_compute_network.vpc_network]
}
```

These last two blocks are where things get interesting. The block above configures the private services connection using the VPC network and private IP allocation created by Terraform. `Service` is a specific string, `servicenetworking.googleapis.com`, since Google is the service provider in this scenario. This value is set in `terraform.tfvars`, as we will see in a moment.  If you find this confusing, check the available documentation for this resource, and it should help you to understand it.

```tf
resource "google_compute_network_peering_routes_config" "peering_routes" {
  peering = var.peering
  network = google_compute_network.vpc_network.name
  import_custom_routes = true
  export_custom_routes = true
  depends_on           = [google_service_networking_connection.gcve-psa]
}
```

The final block enables the import and export of custom routes for our VPC peering configuration.

Note that the final two blocks contain an argument that none of the others do: `depends_on`. The Terraform documentation describes `depends_on` in-depth [here](https://www.terraform.io/docs/language/meta-arguments/depends_on.html), but basically, this is a hint for Terraform to describe resources that rely on each other. Typically, Terraform can determine this automatically, but there are occasional cases where this statement needs to be used. Running `terraform destroy` without this argument in place may lead to errors, as Terraform could delete the VPC before removing the private services connection or route peering configuration. 

## terraform.tfvars Contents

`terraform.tfvars` is the file that defines all the variables that are referenced in `main.tf`. All you need to do is supply the desired values for your environment, and you are good to go. Note that the variables below are all examples, so simply copying and pasting may not lead to the desired result.

```python
project                         = "your-gcp-project"
region                          = "us-west2"
zone                            = "us-west2-a"
network_name                    = "gcve-usw2"
network_descr                   = "Network for testing of GCVE in USW2"
subnet_name                     = "gcve-usw2-mgmt"
subnet_cidr                     = "192.168.82.0/24"
reserved1_name                  = "gcve-managemnt-ip-alloc"
reserved1_address               = "192.168.80.0"
reserved1_address_prefix_length = 23
reserved2_name                  = "gcve-workload-ip-alloc"
reserved2_address               = "192.168.84.0"
reserved2_address_prefix_length = 23
address_purpose                 = "VPC_PEERING"
address_type                    = "INTERNAL"
service                         = "servicenetworking.googleapis.com"
peering                         = "servicenetworking-googleapis-com"
```

Additional information on the variables used is available in [README.md](https://github.com/shamsway/gcp-terraform-examples/blob/main/gcve-vpc-peering/README.md). You can also find information on these variables, including their default values should one exist, in `variables.tf`.

## Initializing and Running Terraform

Terraform will use [Application Default Credentials](https://cloud.google.com/sdk/gcloud/reference/auth/application-default) to authenticate to Google Cloud. Assuming you have the `gcloud` cli tool installed, you can set these by running `gcloud auth application-default`. Additional information on authentication can be found in the [Getting Started with the Google Provider](https://registry.terraform.io/providers/hashicorp/google/latest/docs/guides/getting_starte) Terraform documentation. To run the Terraform code, follow the steps below. 

**Following these steps will create resources in your Google Cloud project, and you will be billed for them.**

1. Run `terraform init` and ensure no errors are displayed
2. Run `terraform plan` and review the changes that Terraform will perform
3. Run `terraform apply` to apply the proposed configuration changes

Should you wish to remove everything created by Terraform, run `terraform destroy` and answer `yes` when prompted. This will only remove the VPC network and related configuration created by Terraform. Your GCVE environment will have to be deleted using [these instructions](https://cloud.google.com/vmware-engine/docs/private-clouds/howto-delete-private-cloud), if desired.

# Review VPC Configuration

Once `terraform apply` completes, you can see the results in the [Google Cloud Console](https://console.cloud.google.com/).

{: .center}
[![](/resources/2021/02/network_allocated_ips_edited.png)](/resources/2021/02/network_allocated_ips_edited.png){:.drop-shadow}

IP ranges allocated for use in GCVE are reserved.

{: .center}
[![](/resources/2021/02/network_service_connection_edited.png)](/resources/2021/02/network_service_connection_edited.png){:.drop-shadow}

Private service access is configured.

{: .center}
[![](/resources/2021/02/network_peering_edited.png)](/resources/2021/02/network_peering_edited.png){:.drop-shadow}

Import and export of custom routes on the `servicenetworking-googleapis-com` private connection is enabled.

# Complete Peering in GCVE

The final step is to create the private connection in the VMware Engine portal. You will need the following information to configure the private connection.

* Project ID (found under `Project info` on the console dashboard.) `Project ID` may be different than `Project Name`, so verify you are gathering the correct information.
* Project Number (also found under `Project info` on the console dashboard.)
* Name of the VPC (`network_name` in your `variables.tf` file.)
* Peered project ID from VPC Network Peering screen

Save all of these values somewhere handy, and follow these steps to complete peering

{: .center} 
[![](/resources/2021/02/15b_add_private_connection_edited.png)](/resources/2021/02/15b_add_private_connection_edited.png){:.drop-shadow}

1. Open the VMware Engine portal, and browse to `Network > Private connection`.
2. Click `Add network connection` and paste the required values. Supply the peered project ID in the `Tenant project ID` field, VPC name in the `Peer VPC ID` field, and complete the remaining fields.
3. Choose the region your VMware Engine private cloud is deployed in, and click `submit`.

{: .center}
[![](/resources/2021/02/16_add_private_connection_edited.png)](/resources/2021/02/16_add_private_connection_edited.png){:.drop-shadow}

After a few moments, `Region Status` should show a status of `Connected`. Your VMware Engine private cloud is now peered with your Google Cloud VPC. You can verify peering is working by checking the routing table of your VPC.

# Verify VPC Routing Table

Once peering is completed, you should see routes for networks in your GCVE SDDC in your VPC routing table. You can view these routes in the cloud console or with the `gcloud couple networks peerings list-routes service-networking-googleapis-com –network=[VPC Name] –region=[Region name] –direction=incoming`.

{: .center}
[![](/resources/2021/02/19_gcloud_routes_output.png)](/resources/2021/02/19_gcloud_routes_output.png){:.drop-shadow}
Verifying routes with the gcloud cli

{: .center}
[![](/resources/2021/02/17_peering_imported_routes_edited.png)](/resources/2021/02/17_peering_imported_routes_edited.png){:.drop-shadow}
Viewing routes in the console

# Wrap Up

Well, that was fun! You should now have established connectivity between your VMware Engine SDDC and your Google Cloud VPC, but we are only getting started. My next post will cover creating a bastion host in GCP to manage your GCVE environment, and I may take a look at Cloud DNS as well.

This post comes at a good time, as Google has just announced [several enhancements to GCVE](https://cloud.google.com/blog/products/compute/whats-new-in-google-cloud-vmware-engine-in-february-2021), including multiple VPC peering. I’m planning on exploring these enhancements in future posts.

# Terraform Documentation Links

* [Google Provider Configuration Reference](https://registry.terraform.io/providers/hashicorp/google/latest/docs/guides/provider_reference)
* [google_compute_network Resource](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_network)
* [google_compute_subnetwork Resource](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_subnetwork)
* [google_compute_global_address Resource](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_global_address)
* [google_service_networking_connection Resource](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/service_networking_connection)
* [google_compute_network_peering_routes_config Resource](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_network_peering_routes_config)