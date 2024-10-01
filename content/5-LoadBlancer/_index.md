---
title : "Create Load Balancer"
date :  "`r Sys.Date()`" 
weight : 5 
chapter : false
pre : " <b> 5. </b> "
---
Next, we will create the Load Balancer module, which includes the ALB, Listener, and Target Group. When the ALB listens on the configured port from the Listener, it will route traffic to the Target Group. First, we will declare the Load Balancer module in the **main.tf** file in the root directory. Some variables are received from other modules, such as **vpc_id, internet_gw, public_subnets** from the **vpc** module, and **certificate_arn** from the **route53** module to configure HTTPS for the ALB.


```
module "loadbalancer" {
  source            = "./modules/load_balancer"
  public_subnets_id = module.vpc.public_subnets
  vpc_id            = module.vpc.vpc_id
  internet_gw       = module.vpc.internet_gw.id
  certificate_arn = module.route53.cert_arn
}
```
To create a Load Balancer as described above, we need to create:
- 1 ALB
- 1 Listener
- 1 Target Group
- 1 Security Group for the ALB.

### ALB
Here, we will create a Load Balancer with type Application, facing the Internet. It will be placed in 2 public subnets created in the **vpc** module, and it will be created after the Internet Gateway is complete, as traffic must pass through the Internet Gateway to reach the ALB. Finally, this ALB will have a security group created below. This Security Group will simply accept traffic on ports 80 (HTTP) and 443 (HTTPS) from the Internet.


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

Next, we will create a Target Group to route traffic to the EC2 instances in this group. The port used to connect to the EC2 instances in this group will be 80 with the HTTP protocol. Since this is already within our VPC, communication can occur over HTTP port 80. This Target Group is linked to the AutoScaling Group created in the EC2 module, meaning that traffic will be routed to the EC2 instances within that ASG.


```
resource "aws_lb_target_group" "web_app_TG" {
  name     = "web-app-TG"
  port     = 80
  protocol = "HTTP"
  vpc_id   = var.vpc_id
}
```
Next, for the ALB to route traffic to the Target Group, we need a Listener. We will create a resource **aws_lb_listener** from the ALB just created, using the protocol **HTTPS port 443** and **ssl_policy ELBSecurityPolicy-TLS13-1-2-2021-06** (this information is obtained from [AWS documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/describe-ssl-policies.html)). The **certificate_arn** is the certificate obtained through ACM when validating records created in Route53, and it will forward traffic received on port 443 to the Target Group created above.

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

Finally, we create the outputs for other modules to retrieve information from the ALB. Here, we have **target_group_arn** to be used by the **ec2** module mentioned earlier, and also **load_balancer_dns** to be used for creating records in the **route53** module.



```
output "target_group_arn" {
  value = aws_lb_target_group.web_app_TG.arn
}

output "load_balancer_dns" {
  value = aws_lb.my_alb.dns_name
}
```
To check the correctness of everything so far, we can move directly to the Route53 module, create the Route53 module first, then run **terraform init** to download the necessary libraries and run **terraform apply** to check for any errors in the code. If there are no resource errors, run **terraform destroy** and continue with the workshop.
