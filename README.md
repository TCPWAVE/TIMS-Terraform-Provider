# 1.	Terraform Overview
Terraform is used to create, manage, and manipulate infrastructure resources. Examples of resources include physical machines, VMs, network switches, containers, etc. Almost any infrastructure noun can be represented as a resource in Terraform.

Terraform is agnostic to the underlying platforms by supporting providers. A Terraform provider is responsible for understanding API interactions and exposing resources. Providers generally are IaaS (e.g. AWS, GCP, Microsoft Azure, OpenStack), PaaS (e.g. Heroku), or SaaS services (e.g. Terraform Enterprise, DNSimple, CloudFlare).

Terraform uses text files to describe infrastructure and to set variables. These text files are called Terraform configurations and end in .tf.

