---
title : "Port Forwarding"
date :  "`r Sys.Date()`" 
weight : 7
chapter : false
pre : " <b> 7. </b> "
---
Next, we will create the Route53 module, which includes records and certificates for the records. First, we will declare the **route53** module in the **main.tf** file in the root directory. Some variables are received from other modules, such as **load_balancer_dns** from the **loadbalancer** module and **rds_endpoint** from the **database** module, to create records for these links for easier access.


```
module "route53" {
  source       = "./modules/route53"
  alb_dns_name = module.loadbalancer.load_balancer_dns
  endpoint     = module.database.rds_endpoint
}
```
To create a Load Balancer as described above, we need to create:
- 2 records (1 for web, 1 for db).
- 1 ACM Certificate for the 2 records.
- 2 records for the above certificate.
- 1 Request Validation for those 2 records.

### Records
First, we will retrieve the domain information of the created hosted zone, then create 2 records for the ALB and RDS endpoint in this hosted zone with 2 subdomains: **workshop2** and **db**.

```
data "aws_route53_zone" "domain" {
  name = var.domain_name
}

resource "aws_route53_record" "workshop2" {
  zone_id = data.aws_route53_zone.domain.id
  name    = var.fully_domain_name
  type    = "CNAME"
  ttl     = 300
  records = [var.alb_dns_name]
}

resource "aws_route53_record" "db" {
  zone_id = data.aws_route53_zone.domain.id
  name    = var.db
  type    = "CNAME"
  ttl     = 300
  records = [var.endpoint]
}
```

Next, I will create an **ACM Certification** for those 2 subdomains using the DNS method, then create 2 additional records in the hosted zone to validate access to these subdomains. Finally, I will create a **certificate_validation** to validate the 2 subdomains.


```
resource "aws_acm_certificate" "acm_certificate" {
  domain_name       = var.fully_domain_name
  validation_method = "DNS"

  subject_alternative_names = [
    var.fully_domain_name, 
    var.db 
  ]

  lifecycle {
    create_before_destroy = true
  }
}


resource "aws_route53_record" "web_app_cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.acm_certificate.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      type   = dvo.resource_record_type
      value  = dvo.resource_record_value
    }
  }

  zone_id = data.aws_route53_zone.domain.zone_id
  name    = each.value.name
  type    = each.value.type
  ttl     = 300
  records = [each.value.value]
}

resource "aws_acm_certificate_validation" "cert" {
  certificate_arn         = aws_acm_certificate.acm_certificate.arn
  validation_record_fqdns = [for record in aws_route53_record.web_app_cert_validation : record.fqdn]
}
```

Additionally, we need to output the **certificate** for the ALB to configure HTTPS for the ALB.

```
output "cert_arn" {
  value = aws_acm_certificate.acm_certificate.arn
}
```