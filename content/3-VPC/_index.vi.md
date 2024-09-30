---
title : "Khởi tạo Workspace"
date :  "`r Sys.Date()`" 
weight : 3 
chapter : false
pre : " <b> 3. </b> "
---

Đầu tiên, ta cần khởi tạo Workspace như ở [Introduction](../1-Introduction/1.2-Terraform/) ta sẽ tạo một file terraform như **"main.tf"** với nội dung như hình để khai báo version và provider.

```
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-southeast-1"
}
```

Tiếp theo, ta sẽ chạy lệnh **terraform init** để khởi tạo và Terraform tải về máy những thư viện cần thiết.

Vì ở Workshop này sẽ cần tạo khá nhiều resource nên ta cần phải chia ra thành các modules để có thể quản lý các resource dễ hơn.
Ta sẽ tạo một thư mục modules cùng cấp với file main.tf vừa tạo để chứa các modules sắp tạo tiếp theo.

Tiếp theo, chúng ta sẽ thực hiện tạo VPC modules, gồm các subnets để chứa các EC2 chạy ứng dụng web và RDS instances để chạy database.

### Nội dung
3.1. [Tạo VPC](3.1-VPC-main/) \
3.2. [Output cho các modules khác](3.2-VPC-output/) 
