---
title : "Connect to Private instance"
date : "`r Sys.Date()`"
weight : 2
chapter : false
pre : " <b> 3.2. </b> "
---
After completing the VPC module, we will proceed to create other modules. However, suppose we need to create an EC2 instance; in that case, we will need information from the VPC module, such as private subnets, etc. Therefore, we must make this information available to other modules through the **output.tf** file.

```
output "vpc_id" {
  value = aws_vpc.main.id
}

output "private_subnets" {
  value = aws_subnet.private_subnets[*].id
}

output "database_subnets" {
  value = aws_subnet.database_subnets[*].id
}

output "public_subnets" {
  value = aws_subnet.public_subnets[*].id
}

output "internet_gw" {
  value = aws_internet_gateway.igw
}

```


In the code above, we are outputting the **vpc_id** for modules like EC2, Load Balancer, and Database to create resources within this VPC. Additionally, we output the IDs of the subnets created, which will be useful for creating a Bastion Host, AutoScaling Group, RDS instance, etc. Finally, we output the **internet_gw** so that the ALB can receive traffic through it.
