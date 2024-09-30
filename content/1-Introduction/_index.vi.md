---
title : "Giới thiệu"
date :  "`r Sys.Date()`" 
weight : 1 
chapter : false
pre : " <b> 1. </b> "
---
# Triển khai ứng dụng web có tính sẵn sàng cao bằng Terraform


Trong workshop này, chúng ta sẽ xây dựng một ứng dụng web có tính sẵn sàng cao bằng Terraform trên AWS Cloud, chúng ta sẽ dùng Terraform để provision resource chạy ứng dụng web. Tạo AutoScaling Group gồm các EC2 instance chạy ứng dụng web trong kết nối đến database RDS instance. Để đảm bảo tính sẵn sàng cao thì ứng dụng sẽ được deploy trên 2 zones, và có khả năng cân bằng tải cùng với tự động scale khi cần thiết traffic tăng cao, cần xử lí lớn. Để đảm bảo tính bảo mật của ứng dụng ta sẽ đặt EC2 và RDS instance nằm trong các private subnets và có các security groups riêng để người dùng không thể truy cập trực tiếp vào được các EC2 instance được.

![Architecture](/images/main-arc.png)

### Nội dung
  - [IaC(Infrastructure as Code) là gì?](1.1-Iac/)
  - [Terraform basics](1.2-Terraform/)