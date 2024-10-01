---
title : "Create VPC"
date : "`r Sys.Date()`"
weight : 1
chapter : false
pre : " <b> 3.1. </b> "
---
To create a VPC following the high availability architecture outlined at the beginning of the workshop, we need to create:
- 1 VPC.
- 2 public subnets.
- 2 private subnets for EC2 instances running the web application.
- 2 private subnets for RDS instances running the database.
- 1 NAT Gateway to allow EC2 instances to access the internet.
- 1 Internet Gateway to handle traffic from the internet into the VPC.

### VPC
First, create a **vpc** directory within the modules folder, then create a **main.tf** file. In the root **main.tf** file, declare the **VPC** module by adding the code below, which references the module located in **./modules/vpc**.


```
module "vpc" {
  source = "./modules/vpc"
}
```

After adding this code to the root **main.tf** file, run the **terraform init** command to let Terraform recognize the .tf files in different directories and download any necessary libraries.

To create a VPC, declare the "resource aws_vpc" and add the VPC attributes.

```
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"


  tags = {
    Name = "${var.app_name} VPC"
  }
}
```

Here, we create a VPC named "main" with the CIDR block **"10.0.0.0/16"**, and tag it with **"${var.app_name} VPC"**. The **${var.app_name}** value is taken from the **app_name** variable declared in the **variables.tf** file in the same directory. For example, if the **app_name** variable is a string with a default value of "Workshop2," the resulting VPC name will be **"Workshop2 VPC"**.

```
variable "app_name" {
  type = string
  default = "Workshop2"
}
```
This completes the VPC declaration with the CIDR block "10.0.0.0/16".

### Public Subnets
After creating the VPC, we can create resources within it. First, we'll create 2 Public subnets to host a NAT gateway, allowing EC2 instances in the private subnets to access the internet. To create subnets, declare the **aws_subnet** resource in **main.tf**. Instead of creating them one at a time, use the **count** attribute, set to the length of the **public_subnet_cidrs** list, to create both subnets in a single operation. The **vpc_id** is assigned the **id** of the VPC created earlier. You can access the value with **resource_type.resource_name.feature**, for example, **aws_vpc.main.id**. The CIDR block is retrieved using the **element** function and **count.index** to determine the order.

```
resource "aws_subnet" "public_subnets" {
  count             = length(var.public_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = element(var.public_subnet_cidrs, count.index)
  availability_zone = element(var.azs, count.index)
  tags = {
    Name = "${var.app_name} Public Subnets ${count.index + 1}"
  }
}
```
```
variable "public_subnet_cidrs" {
  type        = list(string)
  description = "Public Subnet CIDR values"
  default     = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "azs" {
  type        = list(string)
  description = "Availability Zones Name"
  default     = ["ap-southeast-1a", "ap-southeast-1b"]
}
```


### Private Web Subnets
Next, create 2 private subnets for the EC2 instances running the web application. Like the public subnets, the code below will create 1 private subnet with the CIDR block "10.0.3.0/24" in zone "ap-southeast-1a" and another private subnet with the CIDR block "10.0.4.0/24" in zone "ap-southeast-1b".

```
resource "aws_subnet" "private_subnets" {
  count             = length(var.private_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = element(var.private_subnet_cidrs, count.index)
  availability_zone = element(var.azs, count.index)
  tags = {
    Name = "${var.app_name} Private Subnets ${count.index + 1}"
  }
}
```
```
variable "private_subnet_cidrs" {
  type        = list(string)
  description = "Private Subnet CIDR values"
  default     = ["10.0.3.0/24", "10.0.4.0/24"]
}
variable "azs" {
  type        = list(string)
  description = "Availability Zones Name"
  default     = ["ap-southeast-1a", "ap-southeast-1b"]
}
```

### Private Database Subnets
Similarly, create 2 private subnets for the RDS instances. One subnet will have a CIDR block of "10.0.5.0/24" in zone "ap-southeast-1a" with the name "Workshop2 Database Subnets 1," and the other subnet will have a CIDR block of "10.0.6.0/24" in zone "ap-southeast-1b" with the name "Workshop2 Database Subnets 2."

```
resource "aws_subnet" "database_subnets" {
  count             = length(var.database_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = element(var.database_subnet_cidrs, count.index)
  availability_zone = element(var.azs, count.index)
  tags = {
    Name = "${var.app_name} Database Subnets ${count.index + 1}"
  }
}
```
```
variable "database_subnet_cidrs" {
  type        = list(string)
  description = "Database Subnet CIDR values"
  default     = ["10.0.5.0/24", "10.0.6.0/24"]
}
variable "azs" {
  type        = list(string)
  description = "Availability Zones Name"
  default     = ["ap-southeast-1a", "ap-southeast-1b"]
}
```

### Internet Gateway
After creating the subnets, we need to route traffic by creating an Internet Gateway to allow traffic to flow in and out of the VPC. Create an **aws_internet_gateway** resource in the VPC with the tag "Workshop2 Internet Gateway" to route traffic.

```
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.app_name} Internet Gateway"
  }
}
```


Next, create a route table to route traffic from the public subnets through the Internet Gateway. Use the **aws_route_table** resource to route traffic to CIDR block "0.0.0.0/0" via the Internet Gateway. Then, associate the 2 public subnets with this route table by creating an **aws_route_table_association** resource.


```
resource "aws_route_table" "second_rt" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "${var.app_name} 2nd Route Table"
  }
}

resource "aws_route_table_association" "public_subnets_association" {
  count = length(var.public_subnet_cidrs)
  route_table_id = aws_route_table.second_rt.id
  subnet_id = element(aws_subnet.public_subnets[*].id, count.index)
}
```


### NAT Gateway
After setting up the route table and associating it with the public subnets, EC2 instances in the private subnets will need to access the internet to pull updates or patches. Since these instances can't be publicly exposed, we'll create a NAT Gateway to allow them to access the internet indirectly. First, create an Elastic IP to assign to the NAT Gateway by declaring an **aws_eip** resource with **domain = "vpc"**.
```
resource "aws_eip" "nat_gw_eip" {
  domain = "vpc"

  tags = {
    Name = "${var.app_name} EIP NAT"
  }
}
```

Next, create a NAT Gateway in one of the public subnets, using the Elastic IP created earlier.

```
resource "aws_nat_gateway" "nat_gw" {
  allocation_id = aws_eip.nat_gw_eip.id
  subnet_id     = aws_subnet.public_subnets[0].id

  tags = {
    Name = "${var.app_name} NAT Gateway"
  }
}
```
Finally, create a route table to route traffic from the private subnets through the NAT Gateway. The process is similar to creating the Internet Gateway route table but with **Private subnets** instead of **Public subnets**, and the **NAT Gateway ID** instead of the **Internet Gateway ID**.

```
resource "aws_route_table" "nat_gw_rt" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_nat_gateway.nat_gw.id
  }

  tags = {
    Name = "${var.app_name} NAT Route Table"
  }
}

resource "aws_route_table_association" "private_subnet_asso_natgw" {
  count = length(var.private_subnet_cidrs)
  route_table_id = aws_route_table.nat_gw_rt.id
  subnet_id = element(aws_subnet.private_subnets[*].id, count.index)
}
```

After writing the code to create a VPC for hosting the web application, we can now test the setup. Open the terminal and run **terraform plan** to see which resources will be created, followed by **terraform apply**, and enter **yes** to confirm the creation of those resources.

![Kết quả sau khi chạy apply](/images/3.initial/result-VPC.png)

As a result, we have successfully created a VPC that holds the resources. Traffic from the private subnets is routed to the NAT gateway, and traffic from the public subnets is routed to the Internet Gateway, while the database subnets do not need a route to handle traffic as the RDS instances are Managed Services, fully managed by AWS.

After verifying that all the created resources are as expected, you can use the **terraform destroy** command to delete all the resources and save costs, then proceed to write code for creating other modules.
