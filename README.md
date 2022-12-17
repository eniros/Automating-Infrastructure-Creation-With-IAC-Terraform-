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

We will now refactor the code to make it dynamic. To do this we will introduce variables.

1. Starting with the provider block, declare a variable named region, give it a default value, and update the provider section by referring to the declared variable.

```
variable "region" {
        default = "us-east-1"
}

provider "aws" {
        region = var.region
}
````

2. Do the same to cidr value in the vpc block, and all the other arguments.

```
variable "region" {
        default = "us-east-1"
    }

variable "vpc_cidr" {
    default = "172.16.0.0/16"
}

variable "enable_dns_support" {
    default = "true"
}

variable "enable_dns_hostnames" {
    default ="true" 
}

variable "enable_classiclink" {
    default = "false"
}

variable "enable_classiclink_dns_support" {
    default = "false"
}

provider "aws" {
region = var.region
}

# Create VPC
resource "aws_vpc" "main" {
cidr_block                     = var.vpc_cidr
enable_dns_support             = var.enable_dns_support 
enable_dns_hostnames           = var.enable_dns_support
enable_classiclink             = var.enable_classiclink
enable_classiclink_dns_support = var.enable_classiclink

}
```

3. We are just going to introduce some interesting concepts. Loops & Data sources

Terraform has a functionality that allows us to pull data which exposes information to us. Let us fetch Availability zones from AWS, and replace the hard coded value in the subnet’s availability_zone section.

```
# Get list of availability zones
data "aws_availability_zones" "available" {
  state = "available"
}
```

4. To make use of this new data resource, we will need to introduce a count argument in the subnet block: Something like this.

```
# Create public subnet1
resource "aws_subnet" "public" { 
    count                   = 2
    vpc_id                  = aws_vpc.main.id
    cidr_block              = "172.16.1.0/24"
    map_public_ip_on_launch = true
    availability_zone       = data.aws_availability_zones.available.names[count.index]

}
```

5.But we still have a problem. If we run Terraform with this configuration, it may succeed for the first time, but by the time it goes into the second loop, it will fail because we still have cidr_block hard coded. The same cidr_block cannot be created twice within the same VPC. So, we have a little more work to do.

We will introduce a function cidrsubnet() to make this happen. It accepts 3 parameters.

```
  # Create public subnet1
  resource "aws_subnet" "public" { 
      count                   = 2
      vpc_id                  = aws_vpc.main.id
      cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
      map_public_ip_on_launch = true
      availability_zone       = data.aws_availability_zones.available.names[count.index]

  }
  ```
  
 6. We can introuduce length() function, which basically determines the length of a given list, map, or string.

Since data.aws_availability_zones.available.names returns a list like ["eu-central-1a", "eu-central-1b", "eu-central-1c"] we can pass it into a lenght function and get number of the AZs.

Now we can simply update the public subnet block like this

```
resource "aws_subnet" "public" { 
    count                   = length(data.aws_availability_zones.available.names)
    vpc_id                  = aws_vpc.main.id
    cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
    map_public_ip_on_launch = true
    availability_zone       = data.aws_availability_zones.available.names[count.index]

}
```

Observations:

What we have now, is sufficient to create the subnet resource required. But if you observe, it is not satisfying our business requirement of just 2 subnets. The length function will return number 3 to the count argument, but what we actually need is 2. Now, let us fix this.

7. Declare a variable to store the desired number of public subnets, and set the default value

```
variable "preferred_number_of_public_subnets" {
  default = 2
}
```
8. Next, update the count argument with a condition. Terraform needs to check first if there is a desired number of subnets. Otherwise, use the data returned by the lenght function. See how that is presented below.

The first part var.preferred_number_of_public_subnets == null checks if the value of the variable is set to null or has some value defined.
The second part ? and length(data.aws_availability_zones.available.names) means, if the first part is true, then use this. In other words, if preferred number of public subnets is null (Or not known) then set the value to the data returned by lenght function.
The third part : and var.preferred_number_of_public_subnets means, if the first condition is false, i.e preferred number of public subnets is not null then set the value to whatever is definied in var.preferred_number_of_public_subnets

Now our main.tf looks this:


```
# Get list of availability zones
data "aws_availability_zones" "available" {
state = "available"
}

variable "region" {
      default = "us-east-1"
}

variable "vpc_cidr" {
    default = "172.16.0.0/16"
}

variable "enable_dns_support" {
    default = "true"
}

variable "enable_dns_hostnames" {
    default ="true" 
}

variable "enable_classiclink" {
    default = "false"
}

variable "enable_classiclink_dns_support" {
    default = "false"
}

  variable "preferred_number_of_public_subnets" {
      default = 2
}

provider "aws" {
  region = var.region
}

# Create VPC
resource "aws_vpc" "main" {
  cidr_block                     = var.vpc_cidr
  enable_dns_support             = var.enable_dns_support 
  enable_dns_hostnames           = var.enable_dns_support
  enable_classiclink             = var.enable_classiclink
  enable_classiclink_dns_support = var.enable_classiclink

}

# Create public subnets
resource "aws_subnet" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  vpc_id = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]

}
```

<img width="647" alt="Screenshot 2022-12-17 at 00 49 47" src="https://user-images.githubusercontent.com/61475969/208214325-b4ded800-b084-411d-9699-2dd9b7d3e5e0.png">


Section 5. Introducing variables.tf & terraform.tfvars

1. Instead of havng a long lisf of variables in main.tf file, we can actually make our code a lot more readable and better structured by moving out some parts of the configuration content to other files.

We can create a file called ```variables.tf``` and put all the variables declaration we need in it. Also we will create a ```terraform.tfvars``` file and put all the values of the variables values in it.
Variables.tf file:


```
variable "region" {
      default = "us-east-1"
}

variable "vpc_cidr" {
    default = "172.16.0.0/16"
}

variable "enable_dns_support" {
    default = "true"
}

variable "enable_dns_hostnames" {
    default ="true" 
}

variable "enable_classiclink" {
    default = "false"
}

variable "enable_classiclink_dns_support" {
    default = "false"
}

  variable "preferred_number_of_public_subnets" {
      default = null
}
```

<img width="647" alt="Screenshot 2022-12-17 at 01 00 27" src="https://user-images.githubusercontent.com/61475969/208214838-a93c3054-37fe-4f5e-b415-1c6f2d98f594.png">


terraform.tfvars file:

```
region = "eu-central-1"

vpc_cidr = "172.16.0.0/16" 

enable_dns_support = "true" 

enable_dns_hostnames = "true"  

enable_classiclink = "false" 

enable_classiclink_dns_support = "false" 

preferred_number_of_public_subnets = 2
```

<img width="647" alt="Screenshot 2022-12-17 at 01 01 19" src="https://user-images.githubusercontent.com/61475969/208214885-f375febc-644f-4598-a97a-dcb94282db3e.png">

Main.tf file:

```
# Get list of availability zones
data "aws_availability_zones" "available" {
state = "available"
}

provider "aws" {
  region = var.region
}

# Create VPC
resource "aws_vpc" "main" {
  cidr_block                     = var.vpc_cidr
  enable_dns_support             = var.enable_dns_support 
  enable_dns_hostnames           = var.enable_dns_support
  enable_classiclink             = var.enable_classiclink
  enable_classiclink_dns_support = var.enable_classiclink

}

# Create public subnets
resource "aws_subnet" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  vpc_id = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]
}
```

<img width="809" alt="Screenshot 2022-12-17 at 01 04 29" src="https://user-images.githubusercontent.com/61475969/208215077-9e9d6b3a-b259-45a3-8d5e-15272a3bcc9e.png">

2. You can run ```terraform plan``` now to see the configuration of the infrastructure.

