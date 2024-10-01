---
title : "Connect to EC2 servers"
date : "`r Sys.Date()`"
weight : 3
chapter : false
pre : " <b> 3. </b> "
---

First, we need to initialize the Workspace as shown in the [Introduction](../1-Introduction/1.2-Terraform/). We'll create a file called **"main.tf"** with content similar to the example below to declare the version and provider.


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

Next, we will run the command **terraform init** to initialize and download the necessary libraries to your machine.

Since this workshop requires creating many resources, we will need to divide them into modules for easier resource management.
Create a **modules** directory at the same level as the previously created **main.tf** file to contain the modules we'll create next.

Next, we will create the VPC module, which includes subnets to host the EC2 instances running the web application and RDS instances running the database.

### Contents
3.1. [Creating VPC](3.1-VPC-main/) \
3.2. [Output for other modules](3.2-VPC-output/)