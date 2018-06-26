# 1.	Terraform Overview
Terraform is used to create, manage, and manipulate infrastructure resources. Examples of resources include physical machines, VMs, network switches, containers, etc. Almost any infrastructure noun can be represented as a resource in Terraform.

Terraform is agnostic to the underlying platforms by supporting providers. A Terraform provider is responsible for understanding API interactions and exposing resources. Providers generally are IaaS (e.g. AWS, GCP, Microsoft Azure, OpenStack), PaaS (e.g. Heroku), or SaaS services (e.g. Terraform Enterprise, DNSimple, CloudFlare).

Terraform uses text files to describe infrastructure and to set variables. These text files are called Terraform configurations and end in .tf.

![terraform execution stages](https://user-images.githubusercontent.com/4006576/41900548-9cfc59da-794c-11e8-8cc1-677b1616686e.png)
Figure 1: Terraform execution stages
# 2.	TIMS Terraform Provider Architectural Diagram
![terraform provider diagram](https://user-images.githubusercontent.com/4006576/41908424-87d1f50a-7961-11e8-9665-7cebee839d54.png)

# 3.	TIMS Terraform Provider Details
TCPWave IPAM is integrated with Terraform to provide resources that can create or delete VPC, Subnet, VM in Cloud and corresponding Network, Subnet, Object and Resource Records in IPAM. terraform-provider-tims acts as Terraform Provider for TCPWave IPAM.

Currently Create and Delete functions are supported for the resources.

Provider is present in /opt/tcpwave/terraform folder. There is also a sample template createCloudNetwork.tf in the same folder. User can modify this .tf file and use it to create or delete resources.

terraform.exe is present in /opt/tcpwave/bin.

Example below shows the format of .tf file and execution steps involved in using tims terraform provider. The resource tims_cloud_vpc is taken as example. tims_cloud_vpc creates next available VPC in AWS with the given mask or given cidr.

![example picture](https://user-images.githubusercontent.com/4006576/41908498-cfb16d38-7961-11e8-88db-255e753882b4.png)


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
|-------------|-------------|-------------|
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
This resource will create a Subnet in the given VPC in AWS Cloud account provided. If cidr is not given, next available Subnet will be created for the given mask. 

Algorithm to calculate next available Subnet: While calculating next available Subnet in a given VPC, next available Subnet in AWS will be calculated first, this value will be compared with all the Subnets present in IPAM for the given Network in the given Organization. If this value clashes with existing Subnet in IPAM, next Subnet will be taken into account. This loop repeats until we get non-duplicated Subnet. 

This resource must have dependency on tims_network.

|Param Name|Description|Properties|
|-------------|-------------|-------------|
|cloud_provider|The name of the cloud provider created in IPAM in the specified org_name|Required|
|name|The name of the Subnet to be created.|Optional|
|cidr|IP Address of the Subnet in CIDR format (For Ex: 10.1.10.0/24). If this value is provided, Subnet will be created for this CIDR.|Either of cidr or mask must be provided.|
|mask|Mask of the VPC to be created. Either of CIDR or Mask must be provided. If only Mask is provided, Subnet will be created with next available CIDR of the given mask. |Either of cidr or mask must be provided.|
|org_name|Organization Name created in IPAM|Required|
|vpc_id|VPC Id in which Subnet is to be created. VPC Id of the VPC created using tims_cloud_vpc can be used here. 
Example: "${tims_cloud_vpc.resourceName.vpc_id}"|Required|
|v4_Address|The IPv4 CIDR block for the created Subnet (For ex: 10.1.10.0/24)|Computed|
|subnet_id|The ID of the created Subnet.| Computed|
|available_ips_count|Available Ip’s count of the created Subnet.| Computed|
|state|The current state of the created Subnet.|Computed|
| Ip| IP part of the Subnet created. Ex: 10.1.10.0|Computed|

### Example:
        resource "tims_cloud_subnet" "ecsubnet" {
        cloud_provider = "aws"
        name = "TerraformSubnet"
        mask = "24"
        vpc_id = "${tims_cloud_vpc.evpc.vpc_id}"
        org_name = "Internal"
        depends_on = ["tims_network.enw"]|
        }
## 5.4 tims_subnet
This resource creates a subnet in IPAM with the given IP and mask in the given network in the organization.  If router_addess is not specified, First Object in the Subnet will be created as a Router Object. 

This resource must have dependency on tims_cloud_subnet.

|Param Name|Description|Properties|
|-------------|-------------|-------------|
|Ip|IP Address part of the Subnet (Example: 10.1.10.0). This can be taken from output parameter: ip of tims_cloud_subnet. 
Example:
 "${tims_cloud_subnet. resoruceName.ip}"|Required|
|Name|Name of the Subnet in IPAM|Optional|
|org_name|Organization in IPAM. |Required|
|Mask|Mask of the network.|Required|
|router_address|Router Address in the Subnet. If it is not provided, first IP will be created as Router Object in IPAM.|Optional|
|network_address|IP Address part of the Network (Example: 10.1.10.0). This can be taken from output parameter: ip of tims_cloud_vpc. 
Example:
 "${tims_cloud_vpc. resoruceName.ip}"| Required|
|primary_domain|Resource records will be updated in this domain.| Required|
|cloud_provider|If Cloud provider is provided, Objects can be imported from Cloud. Once AWS Instances are imported from Cloud, these Objects will be associated with AWS Instances. User can change state of the AWS Instance or terminate or view details from IPAM Object grid.|Optional|

### Example:
        resource "tims_subnet" "esubnet" {
        ip =  "${tims_cloud_subnet.ecsubnet1.ip}"
        name = "CloudSubnet"
        mask = "24"
        org = "Internal"
        network_address =  "${tims_cloud_vpc.evpc.ip}"
        primary_domain = "tcpwave.com"
        cloud_provider = "aws"
        depends_on = ["tims_network.enw"]
        }
## 5.5	tims_object
This resource creates an Object with the specified IP Address in the specified Subnet and Organization in IPAM. 

This resource must have dependency on tims_subnet.

|Param Name|Description|Properties|
|-------------|-------------|-------------|
|Ip|IP Address of the Object to be created. Example: 10.1.10.7| Required|
|Name|Name of the Object in IPAM.|Required|
|org_name|Organization in IPAM.|Required|
|domain_name|Domain name into which this object will be allocated.|Required|
|object_type|The type of the Object.|Required|
|description|The description of the Object|Optional|
|Ttl|TTL value of the Object record in the specified domain|Required|
|allocation_type|Allocation type of the Object. Valid values are 1 and 2. 1 represents Static Object. 2 represents dynamic Object that contributes into DHCP scope.|Required|
|subnet_address|Subnet full address. Example: 10.1.10.0/24.
This can be derived from tims_cloud_subnet resource. 
Example: "${tims_cloud_subnet.resourceName.v4_Address}"|Required|

### Example:
        resource "tims_object" "eobj01" {
        ip = "1.2.0.6"
        name = "CloudObject1206"
        org_name = "Internal"
        domain_name = "tcpwave.com"
        object_type = "AWS Instance"
        description = "Object Created by Terraform Tims Provider"
        ttl = "1400"
        allocation_type = "1"
        subnet_address = "${tims_cloud_subnet.ecsubnet1.v4_Address}"
        depends_on = ["tims_subnet.esubnet"]
        }
        
## 5.6	tims_object_rr
This resource creates resource record at object/network/zone level with the specified details. The parameter scope determines the context in which resource record will be created.

|Param Name|Description|Properties|
|-------------|-------------|-------------|
|scope|	Takes 'object', 'zone' or 'network'. Defines the context in which the resource record is being added. |	Required|
|Owner|	Owner name of the resource record. Should point to a valid A record for records of type 'SRV'. Should be a valid domain name for records of type 'A'. Should be a valid alias for records of type CNAME. Should be a valid IP Address for records of type PTR. Should be a valid domain name for records of type NS|Optional|
|Class|	Indicates the class of the resource record. Support only 'IN' currently|Required|
|Type|	Indicates the type of the resource record. Takes one of 'A','CNAME', MX','SRV','NS','TXT','NAPTR','PTR'	|Required|
|Ttl|	Indicates the time-to-live value specified in number of seconds for the resource record.|Required|
|Ip|	IP address of the target object in TCPWave IPAM when defining resource record of type 'A'.|	Required|
|Domain|Domain name in data part of a PTR resource record.|Optional|
|organization|Organization name to be specified for resource records of type NS. If this argument is omitted, the root zone will be selected from the organization the user is associated with. This argument is mandatory if user is 'FADM'.|Required|
|Service|Service name associated with an SRV resource record.|Optional|
|Protocol|Protocol associated with an SRV resource record|Optional|
|host_name|Host name in data part of a PTR resource record|Optional|
|Cname|	CNAME data part of a CNAME record.|Optional|
|pref_num|Preference number associated with an MX resource record.|Optional|
|mail_host|Name of the server hosting the mail service associated with an MX resource record.|Optional|
|Priority|Priority number associated with an SRV resource record.|Optional|
|Weight|Weight associated with an SRV resource record.|Optional|
|Port|	Port number associated with an SRV resource record.|Optional|
|srvc_host|Name of the server hosting the service associated with an SRV record.|Optional|
|Txt|Text associated with a TXT resource record.|Optional|
|Regexp|Regexp value associated with an NAPTR resource record.|Optional|
|Flag|Flag value associated with an NAPTR resource record.|Optional|
|Order|	Order number associated with an NAPTR resource record.|	Optional|
|Params|Params value associated with an NAPTR resource record.|	Optional|
|Replace|Replace field associated with an NAPTR resource record.|Optional|
|name_server|Name Server or data part a NS resource record|Optional|
|view_name|DNS view name in which resource record is being created. This argument is applicable when scope is zone or object|Optional|
|zone_name|Zone name of the target zone in TCPWave IPAM when the scope is zone.	|Optional|
|is_external_rr|Takes 'true' or 'false'. If this argument is specified as 'true’, resource record being added will be added as an external resource record. This argument is applicable when scope is zone else it will be ignored.|Optional|
|proxy_root_zone|DNS Proxy root zone flag. It takes 'true' or 'false'. If it is specified as 'true’, resource record being added will be added as a proxy root zone resource record. If it is specified as 'false' resource record being added will be added as a root zone resource record. This argument is applicable when scope is zone and zone_name is. (dot).||
|network_address|IP Address of the network in TCPWave IPAM when scope is network.||
|Description|Description for the resource record.||

### Example:    
        resource "tims_object" "eobj01" {
        ip = "1.2.0.6"
        name = "CloudObject1206"
        org_name = "Internal"
        domain_name = "tcpwave.com"
        object_type = "AWS Instance"
        description = "Object Created by Terraform Tims Provider"
        ttl = "1400"
        allocation_type = "1"
        subnet_address = "${tims_cloud_subnet.ecsubnet1.v4_Address}"
        depends_on = ["tims_subnet.esubnet"]
        }
# 6.	Non - duplicate VPC in Multiple Cloud Providers
To create non-duplicate VPCs in multiple cloud providers, user needs to make sure to follow below guidelines while creating template.
1.	org_name value must be same for all resources.
2.	Creation of VPC must be followed by creation of the same network in IPAM.
3.	All the resources must be executed sequentially. This can be achieved in Terraform by providing dependency on the previous resource using depends_on parameter. 
Example:  depends_on = ["tims_subnet.esubnet"]

# 7.	Sample .tf file 
Below sample .tf file creates a next available VPC in AWS for the given mask length, next available subnet in AWS for the given mask length in the VPC created, Network and Subnet in IPAM, Objects in IPAM as well as Virtual Machines in AWS and Object resource record.
       
       provider "tims" {
         TIMS-Session-Token = "5e8cf446-9a2d-46aa-87c5-e3e21b8885"
        URL = "https://12.1.10.1:7443/tims"
        }       
        resource "tims_cloud_vpc" "vpc" {
        cloud_provider = "AWS"
        name = "TerraformNetwork"
        mask = "16"
        tenancy = "default"
        org_name = "TCPWave"
        }
        resource "tims_network" "enw" {
        ip = "${tims_cloud_vpc.vpc.ip}"
        name = "TerraformNetwork"
        mask = 16
        org_name = "TCPWave"
        email = "admin@tcpwave.com"
        desc =  "Test"
        zone_template = "TCPWave Standard Template"
        depends_on = ["tims_cloud_vpc.vpc"]
        }
        resource "tims_cloud_subnet" "cloudSubnet" {
        cloud_provider = "AWS"
        name = "TerraformSubnet"
        mask = "24"
        vpc_id = "${tims_cloud_vpc.vpc.vpc_id}"
        org_name = "TCPWave"
        depends_on = ["tims_network.enw"]
        }
        resource "tims_subnet" "esubnet" {
        ip =  "${tims_cloud_subnet.cloudSubnet.ip}"
        name = "TerraformSubnet"
        mask = "24"
        org_name = "TCPWave"
        network_address =  "${tims_cloud_vpc.vpc.ip}"
        primary_domain = "adomain.com"
        cloud_provider = "AWS"
        depends_on = ["tims_cloud_subnet.cloudSubnet"]
        }
        resource "tims_object" "vm001" {
        ip = "2.0.0.7"
        name = "TerraformInstance1"
        org_name = "TCPWave"
        domain_name = "adomain.com"
        object_type = "AWS Instance"
        description = "Object Created by Terraform Tims Provider"
        ttl = "1400"
        allocation_type = "1"
        subnet_address = "2.0.0.0/24"
        cloud_instance_template = "AWS instance template"
                depends_on = ["tims_subnet.esubnet"]
        }
        resource "tims_object" "vm002" {
        ip = "2.0.0.8"
        name = "TerraformInstance2"
        org_name = "TCPWave"
        domain_name = "adomain.com"
        object_type = "AWS Instance"
        description = "Object Created by Terraform Tims Provider"
        ttl = "1400"
        allocation_type = "1"
        subnet_address = "2.0.0.0/24"
        cloud_instance_template = "AWS instance template"
                depends_on = ["tims_subnet.esubnet"]
        }
        resource "tims_object" "vm003" {
        ip = "2.0.0.9"
        name = "TerraformInstance2"
        org_name = "TCPWave"
        domain_name = "adomain.com"
        object_type = "AWS Instance"
        description = "Object Created by Terraform Tims Provider"
        ttl = "1400"
        allocation_type = "1"
        subnet_address = "2.0.0.0/24"
        cloud_instance_template = "AWS instance template"
                depends_on = ["tims_subnet.esubnet"]
        }
        resource "tims_object" "vm004" {
        ip = "2.0.0.10"
        name = "TerraformInstance4"
        org_name = "TCPWave"
        domain_name = "adomain.com"
        object_type = "AWS Instance"
        description = "Object Created by Terraform Tims Provider"
        ttl = "1400"
        allocation_type = "1"
        subnet_address = "2.0.0.0/24"
        cloud_instance_template = "AWS instance template"
                depends_on = ["tims_subnet.esubnet"]
        }
        resource "tims_object" "vm005" {
        ip = "2.0.0.11"
        name = "TerraformInstance5"
        org_name = "TCPWave"
        domain_name = "adomain.com"
        object_type = "AWS Instance"
        description = "Object Created by Terraform Tims Provider"
        ttl = "1400"
        allocation_type = "1"
        subnet_address = "2.0.0.0/24"
        cloud_instance_template = "AWS instance template"
                depends_on = ["tims_subnet.esubnet"]
        }
        resource "tims_object" "vm006" {
        ip = "2.0.0.12"
        name = "TerraformInstance6"
        org_name = "TCPWave"
        domain_name = "adomain.com"
        object_type = "AWS Instance"
        description = "Object Created by Terraform Tims Provider"
        ttl = "1400"
        allocation_type = "1"
        subnet_address = "2.0.0.0/24"
        cloud_instance_template = "AWS instance template"
                depends_on = ["tims_subnet.esubnet"]
        }
        resource "tims_object_rr" "objrr001" {
        scope = "object"
        cname = "TerraformInstance1.adomain.com."
        class = "IN"
        type = "CNAME"
        ttl = "1200"
        owner =  "one.adomain.com."
        ip = "2.0.0.7"
        domain = "adomain.com"
        organization = "TCPWave"
        description = "CNAME Record"
                depends_on = ["tims_object.vm001"]
                }

        resource "tims_object" "eobj01" {
        ip = "1.2.0.6"
        name = "CloudObject1206"
        org_name = "Internal"
        domain_name = "tcpwave.com"
        object_type = "AWS Instance"
        description = "Object Created by Terraform Tims Provider"
        ttl = "1400"
        allocation_type = "1"
        subnet_address = "${tims_cloud_subnet.ecsubnet1.v4_Address}"
        depends_on = ["tims_subnet.esubnet"]
        }
        
# 8.	Use cases
Few examples to show the usage and purpose of resources is provided below.
<picture>
        
# 9.	Conclusion
By using tims terraform provider, user can leverage the capability to manage VPCs, Subnets, VMs and Resource records from IPAM after infrastructure is created using Terraform. Single terraform template can manage multiple cloud providers.




