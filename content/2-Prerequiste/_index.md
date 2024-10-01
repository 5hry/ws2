---
title : "Preparation "
date : "`r Sys.Date()`"
weight : 2
chapter : false
pre : " <b> 2. </b> "
---

{{% notice warning %}}
You need to have a web application ready; without it, you won't be able to proceed. You can reference and use the web application [here](https://github.com/5hry/e-commerce-web-bluegreen-deploy).
{{% /notice %}}

Additionally, it's recommended to have a domain name and create a hosted zone in advance to set up CNAME records for your database and application URLs.

![Route 53 Hosted Zone](/images/2.prerequisite/01-route53.png)

{{% notice info %}}
You can purchase a domain via **Route 53** or buy one from a third party and add it to **Route 53**.
{{% /notice %}}

Now, we can follow the steps in the workshop without worrying about missing anything.

In the next sections, we will sequentially create Terraform modules to provision resources within those modules.
