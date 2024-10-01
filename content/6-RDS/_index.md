---
title : "Port Forwarding"
date :  "`r Sys.Date()`" 
weight : 6
chapter : false
pre : " <b> 6. </b> "
---

Next, we will create the RDS module, which includes a Subnet Group to host the Instances in the RDS Cluster, one RDS Instance, and one RDS Instance Replica. Additionally, we need to create a Security Group (SG) for the RDS Instances. First, we will declare the RDS module in the **main.tf** file in the root directory. Some variables are received from other modules, such as **vpc_id** and **database_subnets** from the **vpc** module, and **private_security_groups** from the **ec2** module to configure access, allowing EC2 instances to connect to the database via port 3306.

```
module "database" {
  source                  = "./modules/rds"
  vpc_id                  = module.vpc.vpc_id
  database_subnets        = module.vpc.database_subnets
  private_security_groups = module.ec2.private_security_groups
}
```
To create a Load Balancer as described above, we need to create:
- 1 Subnet Group.
- 1 RDS Instance.
- 1 RDS Instance Replica.
- 1 Security Group.
### Subnet Group
First, we will create a subnet group consisting of 2 database subnets to host the RDS Instances in the Cluster.

```
resource "aws_db_subnet_group" "db-subnet-group" {
  name       = "my-db-subnet-group"
  subnet_ids = var.database_subnets
  tags = {
    Name = "My DB Subnet Group"
  }
}
```
### RDS Instance

In this section, we will first create a Security Group (SG) to open port **3306 (MySQL)** for EC2 instances from the two private subnets to connect.

```
resource "aws_security_group" "rds_sg" {
  vpc_id      = var.vpc_id
  name        = "DB Subnet SG"
  description = "Allow connection from app subnets"


  ingress {
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = var.private_security_groups
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "RDS Database Security Group"
  }
}
```

Next, we will create 2 RDS instances, one as the Primary and the other as a Replica. For the Primary instance, we will allocate 5GB of storage, use MySQL, and choose the t3.micro class. One thing to note is that the username and password are stored in the code, so when you run it, the **tfstate** file will store them as **plaintext** (a security risk if someone gains access). There will be a **backup** every 7 days, and it will have the **security group, subnet group**, etc.

Similarly, for the Replica, the **replicate_source_db** will be the **identifier** of the Primary.

```
resource "aws_db_instance" "mysql-rds" {
  allocated_storage       = 5
  db_name                 = "MyDBInstance"
  engine                  = "mysql"
  engine_version          = "5.7.44"
  instance_class          = "db.t3.micro"
  username                = "dbakaiusername"
  password                = "akaistark1ngothanhsang"
  skip_final_snapshot     = true
  backup_retention_period = 7
  vpc_security_group_ids  = [aws_security_group.rds_sg.id]
  db_subnet_group_name    = aws_db_subnet_group.db-subnet-group.name
  tags = {
    Name = "My DB"
  }
}


resource "aws_db_instance" "replica-mysql-rds" {
  instance_class          = "db.t3.micro"
  skip_final_snapshot     = true
  backup_retention_period = 7
  replicate_source_db     = aws_db_instance.mysql-rds.identifier
  tags = {
    replica = "true"
    env     = "Dev"
  }
}
```