# Building VPC with Terraform in Amazon AWS

### Introduction

[Terraform](https://www.terraform.io/) is a tool for automating infrastructure management. It can be used for a simple task like managing single application instance or more complex ones like managing entire datacenter or virtual cloud. The infrastructure Terraform can manage includes low-level components such as compute instances, storage, and networking, as well as high-level components such as DNS entries, SaaS features and others. It is a great tool to have in a DevOps environment and I find it very powerful but simple to use when it comes to managing infrastructure as a code (IaaC). And the best thing about it is the support for various platforms and providers like AWS, Digital Ocean, OpenStack, Microsoft Azure, Google Cloud etc. meaning you get to use the same tool to manage your infrastructure on any of these cloud providers. See the [Providers](https://www.terraform.io/docs/providers/index.html) page for full list.

Terraform uses its own domain-specific language (DSL) called the Hashicorp Configuration Language (HCL): a fully JSON-compatible language for describing infrastructure as code. Configuration files created by HCL are used to describe the components of the infrastructure we want to build and manage. It generates an execution plan describing what it will do to reach the desired state, and then executes it to build the `terraform.tfstate` file by default. This state file is extremely important; it maps various resource metadata to actual resource IDs so that Terraform knows what it is managing. This file must be saved and distributed to anyone who might run Terraform against the very VPC infrastructure we created so storing this in GitHub repository is the best way to go in order to share a project.

To install terraform follow the simple steps from the install web page [Getting Started](https://www.terraform.io/intro/getting-started/install.html)


### Notes before we start

First let me mention that the below code has been adjusted to work with the latest Terraform and has been successfully tested with version `v0.10.3` It has been running for long time now and been used many times to create production VPC's in AWS. It's been based on `CloudFormation` templates I've written for the same purpose at some point in 2014 during the quest of converting our infrastructure into code.

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

I have created most of the variables as generic and then passing on their values via separate `.tfvars` file `vpc_environment.tfvars`:

```hcl
vpc = {
  tag                   = "TFTEST"
  owner_id              = "<owner-id>"
  cidr_block            = "10.99.0.0/20"
  subnet_bits           = "4"
  sns_email             = "<sns-email>"
}
key_name                = "<ssh-key>"
nat.instance_type       = "m3.medium"
env_domain = {
  name                  = "mydomain.com"
  zone_id               = "<zone-id>"
}
```
Terraform does not support (yet) interpolation by referencing another variable in a variable name (see [Terraform issue #2727](https://github.com/hashicorp/terraform/issues/2727)) nor usage of an array as an element of a map. These are couple of shortcomings but If you have used AWS's CloudFormation you would have faced similar "issues". After all these tools are not really a programming language so we have to accept them as they are and try to make the best of it.

We can see I have separated the provider stuff from the rest of it including the resource so I can easily share my project without exposing sensitive data. For example I can create GitHub repository out of my project directory and put the `provider-credentials.tfvariables` file in `.gitignore` so it never gets accidentally uploaded.


### Initial trial by fire

Now is time to do the first test. After substituting all values in `<>` with real ones we run:

```hcl
$ terraform plan -var-file provider-credentials.tfvars -var-file vpc_environment.tfvars -out vpc.tfplan
```

Inside the directory and check the output. If this goes without any errors then we can proceed to next step, otherwise we have to go back and fix the errors terraform has printed out. To apply the planned changes then we run:

```hcl
$ terraform apply vpc.tfplan
```

but it’s too early for that at this stage since we have nothing to apply yet.

We can start creating resources now, starting with a VPC, subnets and IGW (Internet Gateway). We want our VPC to be created in a region with 3 AZ’s (Availability Zones) so we can spread our future instance nicely for HA. We create a new `.tf` file `vpc.tf`:

```hcl
/*=== VPC AND SUBNETS ===*/
resource "aws_vpc" "environment" {
  cidr_block           = "${var.vpc["cidr_block"]}"
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags {
    Name                = "VPC-${var.vpc["tag"]}"
    Environment         = "${lower(var.vpc["tag"])}"
  }
}

resource "aws_internet_gateway" "environment" {
  vpc_id = "${aws_vpc.environment.id}"
  tags {
    Name                = "${var.vpc["tag"]}-internet-gateway"
    Environment         = "${lower(var.vpc["tag"])}"
  }
}

resource "aws_subnet" "public-subnets" {
  vpc_id                = "${aws_vpc.environment.id}"
  count                 = "${length(split(",", lookup(var.azs, var.provider["region"])))}"
  cidr_block            = "${cidrsubnet(var.vpc["cidr_block"], var.vpc["subnet_bits"], count.index)}"
  availability_zone     = "${element(split(",", lookup(var.azs, var.provider["region"])), count.index)}"
  tags {
    Name                = "${var.vpc["tag"]}-public-subnet-${count.index}"
    Environment         = "${lower(var.vpc["tag"])}"
  }
  map_public_ip_on_launch = true
}

resource "aws_subnet" "private-subnets" {
  vpc_id                = "${aws_vpc.environment.id}"
  count                 = "${length(split(",", lookup(var.azs, var.provider["region"])))}"
  cidr_block            = "${cidrsubnet(var.vpc["cidr_block"], var.vpc["subnet_bits"], count.index + length(split(",", lookup(var.azs, var.provider["region"]))))}"
  availability_zone     = "${element(split(",", lookup(var.azs, var.provider["region"])), count.index)}"
  tags {
    Name                = "${var.vpc["tag"]}-private-subnet-${count.index}"
    Environment         = "${lower(var.vpc["tag"])}"
    Network             = "private"
  }
}

resource "aws_subnet" "private-subnets-2" {
  vpc_id                = "${aws_vpc.environment.id}"
  count                 = "${length(split(",", lookup(var.azs, var.provider["region"])))}"
  cidr_block            = "${cidrsubnet(var.vpc["cidr_block"], var.vpc["subnet_bits"], count.index + (2 * length(split(",", lookup(var.azs, var.provider["region"])))))}"
  availability_zone     = "${element(split(",", lookup(var.azs, var.provider["region"])), count.index)}"
  tags {
    Name                = "${var.vpc["tag"]}-private-subnet-2-${count.index}"
    Environment         = "${lower(var.vpc["tag"])}"
    Network             = "private"
  }
}
```
