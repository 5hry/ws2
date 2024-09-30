---
title : "Các bước chuẩn bị"
date :  "`r Sys.Date()`" 
weight : 2 
chapter : false
pre : " <b> 2. </b> "
---

{{% notice warning %}}
Bạn cần có sẵn một ứng dụng web, nếu như không có thì không thể làm tiếp được. Có thể tham khảo và sử dụng ứng dụng web [ở đây](https://github.com/5hry/e-commerce-web-bluegreen-deploy)
{{% /notice %}}

Ngoài ra, nên có một domain name và tạo trước một hosted zone để tạo những tạo các CNAME record để set các URL của của database, application.

![Route 53 Hosted Zone](/images/2.prerequisite/01-route53.png)
{{% notice info %}}
Bạn có thể mua domain ở **Route53** hoặc mua của bên thứ 3 rồi add vào **Route53**.
{{% /notice %}}
Bây giờ ta có thể follow theo các bước trong Workshop mà không cần lo lắng thiếu gì nữa.

Trong các phần tiếp theo, chúng ta sẽ lần lượt tạo các modules Terraform để provision resources nằm trong các modules đó.