---
title : "Tạo RDS instance"
date :  "`r Sys.Date()`" 
weight : 6
chapter : false
pre : " <b> 6. </b> "
---
Tiếp theo, chúng ta sẽ thực hiện tạo RDS module, gồm Subnet Group để chứa các Instance thuộc RDS Cluster, một RDS Instance và một RDS Instance Replica, ngoài ra cần tạo thêm 1 SG cho các RDS Instances. Trước tiên ta sẽ khai báo module RDS trong file **main.tf** ở thư mục gốc. Có những biến được nhận từ các module khác như **vpc_id, database_subnets** từ module **vpc**, **private_security_groups** từ module **ec2** để cấu hình cho phép EC2 có thể đọc database thông qua port 3306.

```
module "database" {
  source                  = "./modules/rds"
  vpc_id                  = module.vpc.vpc_id
  database_subnets        = module.vpc.database_subnets
  private_security_groups = module.ec2.private_security_groups
}
```
Để tạo một Load Balancer như trên, ta cần tạo:
- 1 Subnet Group.
- 1 RDS Instance.
- 1 RDS Instance Replica.
- 1 Security Group.

### Subnet Group
Đầu tiên, ta sẽ tạo một subnet group gồm 2 db subnet để chứa các RDS Instance trong Cluster.
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

Ở phần này, trước tiên ta sẽ tạo SG để mở port **3306(mysql)** cho EC2 từ 2 private subnets kia kết nối đến.
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

Sau đó, ta sẽ tạo 2 RDS instance với 1 con là Primary, và một con là Replica. Với con Primary thì dung lượng mình sẽ cấp 5GB, và dùng mysql, class t3.micro. Ở đây, có một cái là username và pasword nằm trong code nên khi chạy thì file **tfstate** nó sẽ lưu luôn ở dạng **plaintext**(rủi ro bảo mật khi có ai đọc được), có **backup** mỗi 7 ngày, **security group, subnetgroup**,...
Và tương tự với Replica có **replicate_source_db** là **identifier** của Primary.
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