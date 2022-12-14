# Automating-Infrastructure-Creation-With-IAC-Terraform-
AUTOMATE INFRASTRUCTURE WITH IAC USING TERRAFORM PART 1

Section 1: IAM/ S3 BUCKET/ AWS CLI ACCESS VIA PYTHON CODE

Create an IAM user, name it terraform (ensure that the user has only programatic access to your AWS account) and grant this user AdministratorAccess permissions.

<img width="811" alt="Screenshot 2022-12-13 at 21 32 37" src="https://user-images.githubusercontent.com/61475969/207448813-183a5569-bd40-481b-8960-861decf3f42d.png">

Create an S3 bucket to store Terraform state file. You can name it something like -dev-terraform-bucket (Note: S3 bucket names must be unique within a region partition

<img width="600" alt="Screenshot 2022-12-13 at 21 43 31" src="https://user-images.githubusercontent.com/61475969/207450226-ae8c1573-d734-474f-9331-4a2ffa158ac0.png">

When you have configured authentication and installed boto3, make sure you can programmatically access your AWS account by running following commands in >python: 

```import boto3
s3 = boto3.resource('s3')
for bucket in s3.buckets.all():
    print(bucket.name)
```

<img width="600" alt="Screenshot 2022-12-13 at 23 54 26" src="https://user-images.githubusercontent.com/61475969/207470741-c676e62f-19ac-4d89-9e0c-d329e5799776.png">

<img width="600" alt="Screenshot 2022-12-13 at 23 55 44" src="https://user-images.githubusercontent.com/61475969/207470866-4bfaa3ba-4d9a-430f-bb47-be2ab37cd053.png">

Section 2: VPC | SUBNETS | SECURITY GROUPS

1. Open VS Code
2. Create a folder called PBL
3. Create a file in the folder, name it main.tf
4. Add AWS as a provider, and a resource to create a VPC in the main.tf file. Provider block informs Terraform that we intend to build infrastructure within AWS while Resource block will create a VPC.

```
provider "aws" {
  region = "us-east-1"
}

# Create VPC
resource "aws_vpc" "main" {
  cidr_block                     = "172.16.0.0/16"
  enable_dns_support             = "true"
  enable_dns_hostnames           = "true"
  enable_classiclink             = "false"
  enable_classiclink_dns_support = "false"
}
```
<img width="600" alt="Screenshot 2022-12-14 at 00 24 35" src="https://user-images.githubusercontent.com/61475969/207474592-c4111c1c-7492-4cab-b06b-5b337985b295.png">


Notice that a new directory has been created: .terraform.... This is where Terraform keeps plugins. Generally, it is safe to delete this folder. It just means that you must execute terraform init again, to download them.

Moving on, let us create the only resource we just defined. aws_vpc. But before we do that, we should check to see what terraform intends to create before we tell it to go ahead and create it.

Run terraform plan

Then, if you are happy with changes planned, execute terraform apply

<img width="770" alt="Screenshot 2022-12-14 at 00 26 32" src="https://user-images.githubusercontent.com/61475969/207474268-612afa29-4a01-4c58-be64-0d3b4a7b1829.png">

Section 3: Subnets resource section

According to our architectural design, we require 6 subnets:

2 public
2 private for webservers
2 private for data layer
Let us create the first 2 public subnets.

Add below configuration to the main.tf file:

```
# Create public subnets1
    resource "aws_subnet" "public1" {
    vpc_id                     = aws_vpc.main.id
    cidr_block                 = "172.16.0.0/24"
    map_public_ip_on_launch    = true
    availability_zone          = "us-east-1a"

}

# Create public subnet2
    resource "aws_subnet" "public2" {
    vpc_id                     = aws_vpc.main.id
    cidr_block                 = "172.16.1.0/24"
    map_public_ip_on_launch    = true
    availability_zone          = "us-east-1b"
}
```

We are using the vpc_id argument to interpolate the value of the VPC id by setting it to aws_vpc.main.id. This way, Terraform knows inside which VPC to create the subnet.

Run terraform plan again to see the changes and execute terraform apply to apply the changes.

NOTE: The setup so far has been full of hardcoded values into the different blocks. We will refactor this in the next section just to make things dynamic and re-usable.

<img width="770" alt="Screenshot 2022-12-14 at 00 33 46" src="https://user-images.githubusercontent.com/61475969/207475338-305afdcf-ddc2-4439-8a8b-97aecd857bce.png">


Run terraform destroy to destroy the infrastructure.

<img width="647" alt="Screenshot 2022-12-14 at 00 34 51" src="https://user-images.githubusercontent.com/61475969/207475466-6ec40c59-a156-4749-bc02-f6524e2699d9.png">

Section 4: FIXING THE PROBLEMS BY CODE REFACTORING


