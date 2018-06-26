# 1.	Terraform Overview
Terraform is used to create, manage, and manipulate infrastructure resources. Examples of resources include physical machines, VMs, network switches, containers, etc. Almost any infrastructure noun can be represented as a resource in Terraform.

Terraform is agnostic to the underlying platforms by supporting providers. A Terraform provider is responsible for understanding API interactions and exposing resources. Providers generally are IaaS (e.g. AWS, GCP, Microsoft Azure, OpenStack), PaaS (e.g. Heroku), or SaaS services (e.g. Terraform Enterprise, DNSimple, CloudFlare).

Terraform uses text files to describe infrastructure and to set variables. These text files are called Terraform configurations and end in .tf.

![terraform execution stages](https://user-images.githubusercontent.com/4006576/41900548-9cfc59da-794c-11e8-8cc1-677b1616686e.png)
Figure 1: Terraform execution stages
# 2.	TIMS Terraform Provider Architectural Diagram
<Picture Terraform Provider diagram>

# 3.	TIMS Terraform Provider Details
TCPWave IPAM is integrated with Terraform to provide resources that can create or delete VPC, Subnet, VM in Cloud and corresponding Network, Subnet, Object and Resource Records in IPAM. terraform-provider-tims acts as Terraform Provider for TCPWave IPAM.

Currently Create and Delete functions are supported for the resources.

Provider is present in /opt/tcpwave/terraform folder. There is also a sample template createCloudNetwork.tf in the same folder. User can modify this .tf file and use it to create or delete resources.

terraform.exe is present in /opt/tcpwave/bin.

Example below shows the format of .tf file and execution steps involved in using tims terraform provider. The resource tims_cloud_vpc is taken as example. tims_cloud_vpc creates next available VPC in AWS with the given mask or given cidr.
<Example picture>

TIMS-Session-Token is used to authenticate the user.  This token can be created in ‘Session Tokens’ page in IPAM. Tims Terraform Provider uses the URL mentioned to connect to IPAM and perform further actions. 
# 4.	Terraform Execution Steps
### Step 1:
Go to the directory where terraform-provider-tims.exe is present and save the .tf file in the same directory.Execute the command terraform plan. This shows the execution plan.

### Step 2:
Execute the command terraform apply. This will execute the plan and add, modify or deletes resources.
# 5.	Resources
Current version of terraform-provider-tims provides create/delete functionality of the resources explained below. 
## 5.1 tims_cloud_vpc
This resource will create VPC in the AWS Cloud account provided. If cidr is not given, next available VPC will be created for the given mask. 

Algorithm to calculate next available VPC: While calculating next available VPC,  next available VPC in AWS will be calculated first, this value will be compared with all the networks present in IPAM for the given Organization. If this value clashes with existing network in IPAM, next VPC will be taken into account. This loop repeats until we get non-duplicated VPC. 

Note: If multiple VPC’s have to be added in multiple cloud providers with non-duplicated CIDR’s, 
1.    The org_name provided in all the resources must be same. Because IPAM supports overlapped networks in different organizations.
2.    Network must be created in IPAM after VPC is created. Otherwise IPAM will lose sync with Cloud provider VPC’s. tims_network   resource can be used to create network in IPAM.

| Param Name  | Description | Properties  |
|-------------|-------------|-------------|
|cloud_provider|The name of the cloud provider created in IPAM in the specified org_name|Required|
|name|The name of the VPC to be created.|Optional|
|cidr|IP Address of the VPC in CIDR format (For Ex: 10.1.10.0/24). If this value is provided, VPC will be created for this CIDR.|Either of cidr or mask must be provided.|
|mask|Mask of the VPC to be created. Either of CIDR or Mask must be provided. If only Mask is provided, VPC will be created with next available CIDR of the given mask. |Either of cidr or mask must be provided.|
|org_name|Organization Name created in IPAM|Required|
|tenancy|The tenancy options for instances launched into the VPC. For default, instances are launched with shared tenancy by default. You can launch instances with any tenancy into a shared tenancy VPC. For dedicated, instances are launched as dedicated tenancy instances by default. You can only launch instances with a tenancy of dedicated or host into a dedicated tenancy VPC. Permitted values are default and dedicated. Values are case sensitive.|Optional|
|v4_Address|The IPv4 CIDR block for the created VPC (For ex: 10.1.10.0/24)|Computed|
|vpc_id|The ID of the created VPC |Computed|
|dhcp_options_id|The ID of the set of DHCP options associated with the created VPC.|Computed|
|instance_tenancy|The allowed tenancy of instances launched into the created VPC.|Computed|
|State|The current state of the created VPC. Valid Values are pending and available|Computed|
|Ip|IP part of the VPC created. Ex: 10.1.10.0|Computed|

### Example:
        resource "tims_cloud_vpc" "evpc" {
        cloud_provider = "aws"
        name = "TerraformNetwork"
        mask = 24
        tenancy = "default"
        org_name = "Internal"
        }
       
       
 
## 5.2 tims_network
This resource creates a network in IPAM with the given IP and mask in the given organization. If Zone template which is associated with AWS Cloud providers is provided, the resource records of Network/Object will be updated in those Cloud providers. 
This resource must have dependency on tims_cloud_vpc.

| Param Name | Description | Properties |
| Ip | IP Address part of the network (Example: 10.1.10.0). This can be taken from output parameter: ip of tims_cloud_vpc. 
Example: "${tims_cloud_vpc. resoruceName.ip}" | Required  |
| name | Name of the network in IPAM  | Required|
|Org_name|Organization in IPAM. This must be same as the org_name in tims_cloud_vpc in order to get non-duplicated VPC’s in multiple cloud providers.|Required|
|mask|Mask of the network.|Required|
|email|Email id of the Admin|Required
|desc|Description of the network|Optional|
|zone_template|Zone Template to be associated with the network.|Optional|

### Example:
    resource "tims_network" "enw" {
        ip = "${tims_cloud_vpc.evpc.ip}"
        name = "CloudNetwork"
        mask = 24
        org = "Internal"
        email = "admin@tcpwave.com"
        desc =  "Test"
        zone_template = "cloud_zone_template"
        depends_on = ["tims_cloud_vpc.evpc"]
        }
## 5.3 tims_cloud_subnet
