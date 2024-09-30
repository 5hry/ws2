---
title : "Tạo Load Balancer"
date :  "`r Sys.Date()`" 
weight : 5 
chapter : false
pre : " <b> 5. </b> "
---
Tiếp theo, chúng ta sẽ thực hiện tạo Load Balancer module, gồm ALB, Listener, Target Group để khi ALB lắng nghe trên port đã được cấu hình từ Listener sẽ route traffic đến Target Group. Trước tiên ta sẽ khai báo module Load Balancer trong file **main.tf** ở thư mục gốc. Có những biến được nhận từ các module khác như **vpc_id, internet_gw, public_subnets** từ module **vpc**, **certificate_arn** từ module **route53** để cấu hình HTTPS cho ALB.

```
module "loadbalancer" {
  source            = "./modules/load_balancer"
  public_subnets_id = module.vpc.public_subnets
  vpc_id            = module.vpc.vpc_id
  internet_gw       = module.vpc.internet_gw.id
  certificate_arn = module.route53.cert_arn
}
```
Để tạo một Load Balancer như trên, ta cần tạo:
- 1 ALB
- 1 Listener
- 1 Target Group
- 1 Security Group cho ALB.


### ALB
Ở đây, ta sẽ tạo một Load Balancer với type là Application và facing Internet, được dặt ở 2 public subnets đã tạo trong module **vpc** và được tạo sau khi Internet Gateway đã hoàn tất vì traffic đi qua Internet Gateway mới tới ALB được, cuối cùng ALB này có securiy group được tạo bên dưới. Security Group này đơn giản nhận traffic từ 2 port 80(HTTP) và 443(HTTPS) từ Internet.

```
resource "aws_lb" "my_alb" {
  name               = "my-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb_sg.id]
  subnets            = var.public_subnets_id
  depends_on         = [var.internet_gw]
  tags = {
    Name = "Load Balancer",
  }
}

resource "aws_security_group" "alb_sg" {
  name        = "alb-security-group"
  description = "Security group for Application Load Balancer"
  vpc_id      = var.vpc_id 

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "ALB Security Group"
  }
}

```


Sau đó ta sẽ tạo một Target Group để traffic route đến những EC2 trong group này. Port khi kết nối đến các EC2 trong group này là sẽ 80 với giao thức HTTP. Vì ở đây đã nằm trong VPC của mình rồi nên có thể giao tiếp bằng HTTP port 80. Target group này chính là AutoScaling Group đã tạo ở module EC2, điều này có nghĩa là traffic sẽ được route đến các EC2 nằm trong ASG đó.

```
resource "aws_lb_target_group" "web_app_TG" {
  name     = "web-app-TG"
  port     = 80
  protocol = "HTTP"
  vpc_id   = var.vpc_id
}
```

Tiếp theo, để ALB route traffic đến Target Group, ta cần một Listener. Ta sẽ tạo một resource **aws_lb_listener** từ ALB vừa tạo với giao thức **HTTPS port 443** và **ssl_policy ELBSecurityPolicy-TLS13-1-2-2021-06** (Thông tin này mình lấy từ [trang chủ của AWS](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/describe-ssl-policies.html)) và **certificate_arn** là cert khi request validation bằng ACM cho các record được tạo trong Route53 và forward traffic nhận được trên port 443 đến cho TargetGroup được tạo ở trên.

```
resource "aws_lb_listener" "alb_listener_https" {
  load_balancer_arn = aws_lb.my_alb.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = var.certificate_arn
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.web_app_TG.arn
  }
}
```

Cuối cùng, ta tạo các output cho các modules khác để lấy thông tin từ ALB. Ở đây, có **target_group_arn** để phục vụ cho module **ec2** đã nói ở trước và ngoài ra còn có thêm **load_balancer_dns** để phục vụ việc tạo record trong module **route53**.

```
output "target_group_arn" {
  value = aws_lb_target_group.web_app_TG.arn
}

output "load_balancer_dns" {
  value = aws_lb.my_alb.dns_name
}
```

Để kiểm tra tính đúng đắn từ nãy đến giờ, ta có thể đi thẳng qua module Route53, tạo module Route53 trước sau đó chạy **terraform init** để tải những thư viện cần thiết và chạy **terraform apply** để kiểm tra xem code có sai sót ở đâu hay không. Nếu như không có lỗi gì từ resource thì chạy lệnh **terraform destroy** rồi tiếp tục workshop.