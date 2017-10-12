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
