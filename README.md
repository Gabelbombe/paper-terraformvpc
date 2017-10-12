# Building VPC with Terraform in Amazon AWS

### Introduction

[Terraform](https://www.terraform.io/) is a tool for automating infrastructure management. It can be used for a simple task like managing single application instance or more complex ones like managing entire datacenter or virtual cloud. The infrastructure Terraform can manage includes low-level components such as compute instances, storage, and networking, as well as high-level components such as DNS entries, SaaS features and others. It is a great tool to have in a DevOps environment and I find it very powerful but simple to use when it comes to managing infrastructure as a code (IaaC). And the best thing about it is the support for various platforms and providers like AWS, Digital Ocean, OpenStack, Microsoft Azure, Google Cloud etc. meaning you get to use the same tool to manage your infrastructure on any of these cloud providers. See the [Providers](https://www.terraform.io/docs/providers/index.html) page for full list.

Terraform uses its own domain-specific language (DSL) called the Hashicorp Configuration Language (HCL): a fully JSON-compatible language for describing infrastructure as code. Configuration files created by HCL are used to describe the components of the infrastructure we want to build and manage. It generates an execution plan describing what it will do to reach the desired state, and then executes it to build the `terraform.tfstate` file by default. This state file is extremely important; it maps various resource metadata to actual resource IDs so that Terraform knows what it is managing. This file must be saved and distributed to anyone who might run Terraform against the very VPC infrastructure we created so storing this in GitHub repository is the best way to go in order to share a project.

To install terraform follow the simple steps from the install web page [Getting Started](https://www.terraform.io/intro/getting-started/install.html)


### Notes before we start

First let me mention that the below code has been adjusted to work with the latest Terraform and has been successfully tested with version `v0.10.3` It has been running for long time now and been used many times to create production VPC’s in AWS. It’s been based on `CloudFormation` templates I’ve written for the same purpose at some point in 2014 during the quest of converting our infrastructure into code.

What is this going to do is:

  - Create a multi tier VPC (Virtual Private Cloud) in specific AWS region
  - Create 2 Private and 1 Public Subnet in the specific VPC CIDR
  - Create Routing Tables and attach them to the appropriate subnets
  - Create a NAT instance with ASG (Auto Scaling Group) to serve as default gateway for the private subnets
  - Create Security Groups to use with EC2 instances we create
  - Create SNS notifications for Auto Scaling events

This is a step-by-step walk through, the source code will be made available at some point.

We will need an SSH key and SSL certificate (for ELB) uploaded to our AWS account and `awscli` tool installed on the machine we are running terraform before we start.


### Building Infrastructure

After setting up the binaries we create an empty directory that will hold the new project. First thing we do is tell terraform which provider we are going to use. Since we are building Amazon AWS infrastructure we create a `.tfvars` file with our AWS IAM API credentials. For example, `provider-credentials.tfvars` with the following content:

```hcl
provider = {
  access_key  = "<AWS_ACCESS_KEY>"
  secret_key  = "<AWS_SECRET_KEY>"
  region      = "ap-southeast-2"
}
```

We make sure the API credentials are for user that has full permissions to create, read and destroy infrastructure in our AWS account. Check the IAM user and its roles to confirm this is the case.

Then we create a `.tf` file where we create our first resource called `provider`. Lets name the file `provider-config.tf` and put the following content:

```hck
provider "aws" {
  access_key = "${var.provider["access_key"]}"
  secret_key = "${var.provider["secret_key"]}"
  region     = "${var.provider["region"]}"
}
```

for our AWS provider type.

Then we create a `.tf` file `vpc_environment.tf` where we put all essential variables needed to build the VPC, like VPC CIDR, AWS zone and regions, default EC2 instance type and the ssh key and other AWS related parameters:

```hcl
/*=== VARIABLES ===*/
variable "provider" {
  type = "map"
  default = {
    access_key = "unknown"
    secret_key = "unknown"
    region     = "unknown"
  }
}

variable "vpc" {
  type    = "map"
  default = {
    "tag"         = "unknown"
    "cidr_block"  = "unknown"
    "subnet_bits" = "unknown"
    "owner_id"    = "unknown"
    "sns_topic"   = "unknown"
  }
}

variable "azs" {
  type = "map"
  default = {
    "ap-southeast-2" = "ap-southeast-2a,ap-southeast-2b,ap-southeast-2c"
    "eu-west-1"      = "eu-west-1a,eu-west-1b,eu-west-1c"
    "us-west-1"      = "us-west-1b,us-west-1c"
    "us-west-2"      = "us-west-2a,us-west-2b,us-west-2c"
    "us-east-1"      = "us-east-1c,us-west-1d,us-west-1e"
  }
}

variable "instance_type" {
  default = "t1.micro"
}

variable "key_name" {
  default = "unknown"
}

variable "nat" {
  type    = "map"
  default = {
    ami_image         = "unknown"
    instance_type     = "unknown"
    availability_zone = "unknown"
    key_name          = "unknown"
    filename          = "userdata_nat_asg.sh"
  }
}

/* Ubuntu Trusty 14.04 LTS (x64) */
variable "images" {
  type    = "map"
  default = {
    eu-west-1      = "ami-47a23a30"
    ap-southeast-2 = "ami-6c14310f"
    us-east-1      = "ami-2d39803a"
    us-west-1      = "ami-48db9d28"
    us-west-2      = "ami-d732f0b7"
  }
}

variable "env_domain" {
  type    = "map"
  default = {
    name    = "unknown"
    zone_id = "unknown"
  }
}
```
