---
title : "Tạo các records và Certs"
date :  "`r Sys.Date()`" 
weight : 7
chapter : false
pre : " <b> 7. </b> "
---
Tiếp theo, chúng ta sẽ thực hiện tạo Route53 module, gồm các records, certificate cho các records. Trước tiên ta sẽ khai báo module **route53** trong file **main.tf** ở thư mục gốc. Có những biến được nhận từ các module khác như **load_balancer_dns** từ module **loadbalancer** và **rds_endpoint** từ module **database** để tạo các record cho các link này để có thể truy cập một cách tiện hơn.

```
module "route53" {
  source       = "./modules/route53"
  alb_dns_name = module.loadbalancer.load_balancer_dns
  endpoint     = module.database.rds_endpoint
}
```
Để tạo một Load Balancer như trên, ta cần tạo:
- 2 records (1 cho web, 1 cho db).
- 1 ACM Certificate cho 2 records trên.
- 2 records cho certificate trên.
- 1 Request Validation cho 2 records đó.

### Records
Đầu tiên, ta sẽ lấy thông tin domain đã tạo hosted zone, sau đó tạo 2 records cho ALB và RDS endpoint trong hosted zone này với 2 subdomains là **workshop2** và **db**

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


Sau đó, mình sẽ tạo **ACM Certification** cho 2 subdomains đó bằng phương thức DNS, tiếp theo tạo thêm 2 records trong hosted zone đó để validate khi truy cập vào 2 subdomains này. Cuối cùng mình sẽ tạo **certificate_validation** để xác thực 2 subdomains.

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

Ngoài ra cần output **certificate** cho ALB để cấu hình HTTPS cho ALB.

```
output "cert_arn" {
  value = aws_acm_certificate.acm_certificate.arn
}
```