---
title : "VPC Output cho các module khác"
date :  "`r Sys.Date()`" 
weight : 2 
chapter : false
pre : " <b> 3.2. </b> "
---

Sau khi tạo xong module VPC, ta sẽ tiếp tục tạo các modules khác. Tuy nhiên, giả sự như ta tạo một EC2, ta cần biết được private subnet, ... những thông tin từ module VPC, vì vậy ta cần phải cho các modules khác biết được những thông tin đó thông qua file **output.tf**.


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

Ở trên ta sẽ output ra **vpc_id** cho các module như EC2, Load Balancer và Database để tạo những resource liên quan trong VPC này. Ngoài ra ta cũng output ra những subnets id được tạo để phục vụ cho việc tạo Bastion Host, AutoScaling Group, RDS instance,... Và cuối cùng sẽ output **internet_gw** để ALB nhận traffic thông qua nó. 