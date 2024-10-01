---
title : "Create WebApp Instance"
date : "`r Sys.Date()`"
weight : 4
chapter : false
pre : " <b> 4. </b> "
---


Next, we will create the EC2 module, which includes an AutoScaling Group containing EC2 instances to run the web application, a Security Group, and an IAM Role to pull images from ECR and run the application. First, we will declare the EC2 module in the **main.tf** file located in the root directory. Some variables will be received from other modules, such as **vpc_id**, **private_subnets_id**, and **public_subnets_id** from the **vpc** module, as well as **load_balancer_id** (the ALB ID) and **target_group_arn** (the target group created for ALB to route traffic) from the **load_balancer** module.


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


To create an AutoScaling Group as described in the architecture at the beginning of the workshop, we need to create:
- Launch Template.
- IAM Role for EC2.
- Security groups for EC2 in public and private subnets.
- AutoScaling group, AutoScaling policies.
- Bastion Host.

**ASG = AutoScaling Group**

### Launch Template
We need to create a Launch Template for the AutoScaling Group so that when it adds an EC2 instance, it will be created according to that template. This template will contain attributes such as **Type, AMI, Keypair, Security Group,...**

First, we will create a role to grant the EC2 instance permission to read from ECR, pull the image, and run the container. Additionally, we need to allow the EC2 instance to read the init.sql file stored on S3, which will later be used in RDS, so S3 read permissions are also required.
![EC2 Web App](/images/4.asg/iam-role.png)

Next, we retrieve the **aws_iam_instance_profile** data and name it **web_app_instance_profile**, assigning this role to the EC2 instance. As mentioned earlier, we will then create a launch template to specify the information for the EC2 instance to be created, such as the AMI, key name, and instance type declared in the variables.tf file, as well as the security group that is created below to open port 80, only listening from the public subnets (where the ALB is located). Additionally, there will be user data, which contains the commands to be executed as soon as the EC2 instance is launched.

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

Below is the user data for EC2 when it is launched. It installs Docker, logs in to pull the image from ECR, then runs a container from that image and maps port 80:80.

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
After creating the Launch Template, we will create the AutoScaling Group from that Launch Template. Here, we create a resource **aws_autoscaling_group** with a min_size of 2 and a max_size of 6 ↔ min of 1 and max of 3 EC2 instances per AZ. The health check period is set to 150s (2.5 minutes), and the health check type is ELB. Next, we specify the zones using the private subnets created to host the EC2 instances, and the target group ARN is received from the output variable of the Load Balancer module ⇒ the EC2 instances in this ASG will be in the target group for the ALB to route traffic to. We also declare the **launch_template** created earlier. You can set a tag name for the EC2 instances using a tag key-value pair.

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

And for the ASG to know when to launch more EC2 instances and when to terminate some, we need to create **Autoscaling Policies**. We can create various types of Autoscaling Policies, such as **TargetTrackingScaling**, based on metrics like average CPU, average network traffic in and out, and the number of requests per EC2 instance, as shown in the image below. Alternatively, a Simple Scaling Policy will **Add/Remove/Set** the number of EC2 instances when there is an alarm from CloudWatch, or the Step Scaling Policy involves multiple consecutive Simple Scalings.
![AutoScaling Policies](/images/4.asg/as_policy.png)

In this workshop, the ASG Policy I will use is **TargetTrackingScaling** to monitor the average CPU metric. When it exceeds **60%**, I will add 1 more EC2 instance.


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
Here, I will use a Bastion Host instead of SSM because the EC2 instances are placed in private subnets, and opening Endpoints for both subnets would require up to 6 Endpoints, which is quite expensive.

First, I will create a Security Group (SG) to allow SSH access to the Bastion Host only from my home IP. Port 22 will be opened from the **my_IP** address declared in the **variables.tf** file.

{{% notice info %}}
Since Ingress only accepts **cidr_blocks**, if you want to restrict access to your specific IP address, you should enter **your_IP/32** with a subnet mask of 32, so only your exact IP address will match this **cidr_blocks**.
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
After creating the SG, we will create the Bastion EC2 instance with basic configurations, and it’s important to allocate a Public IP to allow access from the Internet. If desired, we can output the Public IP when running **terraform apply** to quickly retrieve it for connection, instead of going to the console to copy it.

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
# This section is placed in the output.tf file for easier distinction and searching.
```
Since we are in the EC2 module, the output won't be displayed after execution and will only be printed when placed in the root directory. Therefore, we need to create an output in the root directory.

```
output "bastion_IP" {
  description = "Bastion's Public IP address"
  value       = module.ec2.bastion_IP
}
```
Finally, we will create another output, **private_security_groups**, so that when creating the Security Group for the RDS instance, traffic can be allowed from this SG.
