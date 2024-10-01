---
title : "Introduction"
date :  "`r Sys.Date()`" 
weight : 1 
chapter : false
pre : " <b> 1. </b> "
---
# Deploying a Highly Available Web Application using Terraform

In this workshop, we will build a highly available web application using Terraform on AWS Cloud. We will use Terraform to provision resources for running the web application. We will create an AutoScaling Group consisting of EC2 instances running the web application, connected to an RDS database instance. To ensure high availability, the application will be deployed across two availability zones and will have load balancing capabilities, along with automatic scaling to handle increased traffic and processing loads. To maintain security, we will place the EC2 and RDS instances in private subnets, each with its own security groups, preventing direct access to the EC2 instances by users.

![Architecture](/images/main-arc.png)

### Contents
  - [What is IaC (Infrastructure as Code)?](1.1-Iac/)
  - [Terraform basics](1.2-Terraform/)
