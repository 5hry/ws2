+++
title = "Clean up resources"
date = 2022
weight = 6
chapter = false
pre = "<b>6. </b>"
+++

After completing the code, we will run the **terraform init** command to download the necessary libraries, followed by **terraform apply** to provision the resources. Since creating the RDS Instance takes about 5 minutes and the RDS Instance Replica takes nearly 15 minutes, you can take a break and check the results once it's done.

Results after running the **apply** command:

**Module VPC:** Successfully created VPC with 6 subnets, 2 route tables, 1 Internet GW, and 1 NAT GW.
![VPC](/images/6.clean/result-vpc.png)

**Module EC2:** Successfully created ASG, running 4 EC2 instances.
![ASG](/images/6.clean/result-asg.png)
![EC2](/images/6.clean/result-ec2.png)

**Module LoadBalancer:** Successfully created ALB, Listener, and TargetGroup:
- ALB is placed in 2 Public Subnets, facing the Internet, and located in the hosted zone created in section 2.
![ALB](/images/6.clean/result-alb.png)
- The Listener forwards traffic from port 443 of the ALB to the EC2 instances in **web_app_TG**.
![ALB Listener](/images/6.clean/result-alb-lis.png)

**Module RDS:** Successfully created RDS Primary and Replica instances. Next, the init.sql file from S3 needs to be run on the RDS. I will update this step.
![RDS](/images/6.clean/result-rds.png)
