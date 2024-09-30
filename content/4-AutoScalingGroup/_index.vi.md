---
title : "Tạo Application Instance"
date :  "`r Sys.Date()`" 
weight : 4 
chapter : false
pre : " <b> 4. </b> "
---
Tiếp theo, chúng ta sẽ thực hiện tạo EC2 module, gồm AutoScaling Group chứa các EC2 instance để chạy ứng dụng web, Security Group và IAM Role để pull image từ ECR về và chạy ứng dụng. Trước tiên ta sẽ khai báo module EC2 trong file **main.tf** ở thư mục gốc. Có những biến được nhận từ các module khác như **vpc_id, private_subnets_id, public_subnets_id,** từ module **vpc**, **load_balancer_id**(id của ALB)và **target_group_arn**(target group được tạo để ALB route traffic đến) từ module **load_balancer**.

```
module "ec2" {
  source             = "./modules/ec2"
  vpc_id             = module.vpc.vpc_id
  private_subnets_id = module.vpc.private_subnets
  public_subnets_id  = module.vpc.public_subnets
  load_balancer_id   = module.loadbalancer.load_balancer_id
  target_group_arn   = module.loadbalancer.target_group_arn
}
```


Để tạo một AutoScaling Group như kiến trúc ở đầu Workshop, ta cần tạo:
- Launch Template.
- IAM Role cho EC2.
- Security groups cho EC2 trong public và private subnets.
- AutoScaling group, AutoScaling policies.
- Bastion Host.

**ASG = AutoScaling Group**

### Launch Template
Ta cần tạo một Launch Template cho AutoScaling Group để khi nó tạo thêm một EC2 thì nó sẽ được tạo theo template đó. Template này sẽ chứa các thuộc tính của EC2 như **Type, AMI, Keypair, Security Group,...**

Trước tiên, ta sẽ tạo 1 role để cấp cho EC2 có quyền đọc ECR để pull image từ ECR và chạy container. Ngoài ra, ta cần cho EC2 này đọc file init.sql được lưu trên S3 để sao đó chạy trong RDS nên cũng cần quyền đọc S3. 
![EC2 Web App](/images/4.asg/iam-role.png)

Sau đó, ta đọc dữ liệu **aws_iam_instance_profile** và đặt tên **web_app_instance_profile**, để gán cho EC2 role này. Như ở trên đã nói, tiếp theo ta sẽ tạo một launch template để khai báo những thông tin của EC2 sẽ được tạo như AMI, key name, instance type được khai báo trong file variables.tf và security group được tạo ở dưới mở port 80 chỉ lắng nghe từ public subnets(nơi đặt ALB), ... Ngoài ra còn có user data, nơi chứa những câu lệnh được chạy ngay khi EC2 được launch.

```
data "aws_iam_instance_profile" "web_app_instance_profile" {
  name = "web-app-ec2"
}

resource "aws_launch_template" "instances_configuration" {
  name_prefix            = "asg-instance"
  image_id               = var.ami_name
  key_name               = var.key_name
  instance_type          = var.instance_type
  user_data              = filebase64("./install_script.sh")
  vpc_security_group_ids = [aws_security_group.private_subnet_sg.id]
  iam_instance_profile {
    name = data.aws_iam_instance_profile.web_app_instance_profile.name
  }
  lifecycle {
    create_before_destroy = true
  }

  tags = {
    Name = "asg-instance"
  }
}

resource "aws_security_group" "private_subnet_sg" {
  vpc_id      = var.vpc_id
  name        = "Private Subnet SG"
  description = "Allow traffic from Public Subnets"

  ingress {
    description = "SSH ingress from public subnets"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    security_groups = [aws_security_group.public_subnet_sg.id, ]
  }

  ingress {
    from_port   = -1          
    to_port     = -1           
    protocol    = "icmp"       
    security_groups = [aws_security_group.public_subnet_sg.id, ]
  }

  ingress {
    description = "Http traffic"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    security_groups = [aws_security_group.public_subnet_sg.id, ]
  }
  egress {
    description = "Allow all traffic to internet"
    cidr_blocks = ["0.0.0.0/0"]
    protocol    = "-1"
    from_port   = 0
    to_port     = 0
  }
  tags = {
    Name = "Private Subnet SG"
  }
}
```

Dưới đây là user data cho EC2 khi được khởi động. Cài docker, và đăng nhập để pull image về từ ECR, sau đó chạy container từ image đó và map port 80:80.
```
#!/bin/bash
sudo yum update 
sudo yum install -y docker
sudo systemctl enable docker
sudo service docker start

aws ecr get-login-password --region ap-southeast-1 | docker login --username AWS --password-stdin 905418377779.dkr.ecr.ap-southeast-1.amazonaws.com
REPOSITORY_URI=905418377779.dkr.ecr.ap-southeast-1.amazonaws.com/ws1-bluegreen-repo
IMAGE_TAG="latest"
docker pull $REPOSITORY_URI:$IMAGE_TAG

CONTAINER_NAME="app-container"
PORT_MAPPING="80:80"

if [ $(docker ps -a -q -f name=$CONTAINER_NAME) ]; then
    echo "Stopping and removing existing container: $CONTAINER_NAME"
    docker stop $CONTAINER_NAME
    docker rm $CONTAINER_NAME
fi

docker run -d --name $CONTAINER_NAME -p $PORT_MAPPING $REPOSITORY_URI:$IMAGE_TAG

if [ $(docker ps -q -f name=$CONTAINER_NAME) ]; then
    echo "Container is running: $CONTAINER_NAME"
else
    echo "Failed to run container: $CONTAINER_NAME"
    exit 1
fi
```

### AutoScaling Group
Sau khi có Launch Template, ta sẽ tạo AutoScaling Group từ launch Template đó. Ở đây, ta tạo một resource **aws_autoscaling_group** có min_size là 2, max_size là 6 ↔ min mỗi AZ là 1 và max là 3 EC2, health check period là 150s = 2,5m và health bằng ELB, sau đó khai báo zone bằng những private subnets được tạo để chứa EC2, và target group ARN là nhận từ biến được output từ module Load Balancer ⇒ các EC2 nằm trong ASG này sẽ ở trong target group để ALB route traffic đến, và khai báo **launch_template** được tạo ở trên. Ta có thể tạo tag name cho những EC2 bằng tag key value.
```
resource "aws_autoscaling_group" "asg" {
  name                      = "asg"
  min_size                  = 2
  max_size                  = 6
  desired_capacity          = 2
  health_check_grace_period = 150
  health_check_type         = "ELB"
  vpc_zone_identifier       = var.private_subnets_id
  target_group_arns         = [var.target_group_arn]
  launch_template {
    id      = aws_launch_template.instances_configuration.id
    version = "$Latest"
  }

  tag {
    key                 = "Name"
    value               = "Web Application"
    propagate_at_launch = true
  }
}
```

Và để ASG biết khi nào thì launch thêm EC2, và khi nào thì terminate bớt đi thì ta cần tạo những **Autoscaling Policies**. Có thể tạo nhiều loại Autoscaling Policies như **TargetTrackingScaling** dựa trên các metrics như trung bình CPU, trung bình network traffic in và out, và số request trên từng EC2 như hình dưới. Hoặc Policy Simple Scaling sẽ **Add/Remove/Set to** số lượng EC2 khi có alarm từ CloudWatch hoặc  Policy Step Scaling sẽ là nhiều Simple Scaling liên tiếp nhau.
![AutoScaling Policies](/images/4.asg/as_policy.png)

Ở workshop này, ASG Policy mình sẽ dùng **TargetTrackingScaling** để theo dõi metric CPU trung bình khi lớn hơn **60%**, mình sẽ tăng thêm 1 EC2.

```
resource "aws_autoscaling_policy" "avg_cpu_policy_greater" {
  name                   = "avg-cpu-policy-greater"
  policy_type            = "TargetTrackingScaling"
  autoscaling_group_name = aws_autoscaling_group.asg.id
  
  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value = 60.0
  }
}
```

### Bastion Host
Ở đây, mình sẽ dùng Bastion Host thay thì SSM vì mình đặt các EC2 trong private subnets nên việc mở Endpoint cho cả 2 subnets sẽ tốn tới 6 Endpoints khá tốn kém.

Trước tiên, mình sẽ tạo một SG để có thể SSH đến Bastion Host chỉ bằng IP của nhà mình. Mở port 22 từ địa chỉ **my_IP** được khai báo trong file **variables.tf**.

{{% notice info %}}
Vì trong Ingress chỉ nhận vào **cidr_blocks** nên nếu muốn để một địa chỉ IP duy nhất của mình thì ta sẽ nhập **your_IP/32** với subnet mask là 32 thì sẽ chỉ có duy nhất địa chỉ IP của bạn khớp với **cidr_blocks** này.
{{% /notice %}}

```
resource "aws_security_group" "public_subnet_sg" {
  vpc_id      = var.vpc_id
  name        = "Public Subnet SG"
  description = "Allow SSH from my home"

  ingress {
    description = "SSH ingress"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.my_IP, ]
  }
  egress {
    description = "Allow all traffic to internet"
    cidr_blocks = ["0.0.0.0/0"]
    protocol    = "-1"
    from_port   = 0
    to_port     = 0
  }
  tags = {
    Name = "Public Subnet SG"
  }
}
```

Sau khi tạo SG, ta sẽ tạo EC2 Bastion với các thông số cơ bản, và đặc biệt lưu ý cấp phát Public IP cho nó để có thể truy cập từ ngoài Internet. Nếu muốn ta có thể output ra khi chạy **terraform apply** để lấy IP và connect đến nhanh hơn thay vì lên console để copy. 

```
resource "aws_instance" "bastion_ec2" {
  ami                         = var.ami_name
  key_name                    = var.key_name
  instance_type               = var.instance_type
  vpc_security_group_ids      = [aws_security_group.public_subnet_sg.id]
  subnet_id                   = var.public_subnets_id[0]
  associate_public_ip_address = true
}
```

```
output "bastion_IP" {
  value = aws_instance.bastion_ec2.public_ip
}
# Đoạn này được đặt trong file output.tf để phân biệt và tìm kiếm dễ hơn.
```
Vì mình đang ở module EC2 nên output vẫn chưa được in ra khi chạy xong và chỉ được in ra khi nằm trong thư mục gốc. Vậy ta cần tạo lên một output tại thư mục gốc.
```
output "bastion_IP" {
  description = "Bastion's Public IP address"
  value       = module.ec2.bastion_IP
}
```

Cuối cùng, ta sẽ tạo một output nữa là private_security_groups để khi tạo Security Group cho RDS instance có thể cho traffic từ SG này đến.