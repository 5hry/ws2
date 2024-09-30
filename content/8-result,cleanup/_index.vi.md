+++
title = "Kết quả và dọn dẹp tài nguyên  "
date = 2021
weight = 8
chapter = false
pre = "<b>8. </b>"
+++

Sau khi đã hoàn tất phần code, ta sẽ thực hiện chạy lệnh **terraform init** để tải về các thư viện cần thiết và chạy **terraform apply** để provision resources. Vì tạo RDS Instance mất khoảng 5p và RDS Instance mất gần 15p nên mọi người có thể nghỉ ngơi rồi chờ nó chạy xong có thể quay lại kiểm tra kết quả.

Kết quả sau khi chạy lệnh **apply**:

**Module VPC:** Tạo thành công VPC với 6 subnets, 2 route tables, 1 Internet GW, và 1 NAT GW.
![VPC](/images/6.clean/result-vpc.png)
**Module EC2:** Tạo thành công ASG, chạy 4 EC2.
![ASG](/images/6.clean/result-asg.png)
![EC2](/images/6.clean/result-ec2.png)

**Module LoadBalancer:** Tạo thành công ALB, Listener và TargetGroup:
- ALB nằm ở 2 Public Subnets, facing Internet và nằm trong hosted zone được tạo ở phần 2.
![ALB](/images/6.clean/result-alb.png)
- Listener forward traffic đến port 443 của ALB đến các EC2 nằm trong **web_app_TG**.
![ALB Listener](/images/6.clean/result-alb-lis.png)
**Module RDS:** Tạo thành công RDS instance Primary và Replica. Sau đó, cần lấy file init.sql từ S3 để chạy trên RDS. Bước này mình sẽ cập nhật thêm.
![RDS](/images/6.clean/result-rds.png)