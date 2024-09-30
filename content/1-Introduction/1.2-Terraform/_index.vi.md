---
title : "Terraform là gì?"
date :  "`r Sys.Date()`" 
weight : 2
chapter : false
pre : " <b> 1.2 </b> "
---
Hiện tại thì Terraform có lẽ là tool IaC thông dụng nhất ở thời điểm hiện tại. Nó là open-source của HashiCorp chuyên dùng để tạo ra infrastructure thông qua những dòng code mà chúng ta viết thay vì chúng ta phải lên console để tạo tốn khá nhiều thời gian.

![Terraform](/images/1.Introduction/terraform.png)

Có tool khác cũng có thể làm được như này, như là Ansible, tuy nhiên Ansible lại là một Configuration Management tool chứ không tập trung vào IaC như Terraform nên việc sử dụng Ansible sẽ cần phải chạy thêm những thứ không cần thiết.

**Ưu điểm:** 
- Open-source, miễn phí.
- Cộng đồng lớn.
- Declarative programing.
- Một ưu điểm khá là lớn: có thể cung cấp hạ tầng cho nhiều Cloud Provider(AWS, GCP, Azure)  trong cùng một file cấu hình.

## Ví dụ sử dụng Terraform:
Trước khi đi vào triển khai Terraform, ta cần cài đặt AWS CLI và Terraform CLI
- Cài AWS CLI. [AWS CLI Configuration](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html)
- Cài Terraform CLI. [Terraform CLI Configuration](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli)

![Terraform life cycle](/images/1.Introduction/01-tf.png)

Để triển khai hạ tầng bằng Terraform, ta cần biết các bước, cũng như một số câu lệnh thông dụng khi sử dụng Terraform:
- Đầu tiên, ta cần tạo một workspace, có thể là một thử mục ở máy local.

- Tiếp theo, viết code khai báo các hạ tầng mình muốn triển khai. Ta có thể khai báo các resources theo cấu trúc sau. Ở dưới, ta sử dụng một block tên là resources, đây là block quan trọng nhất trong terraform, chúng ta sẽ sử dụng block này để tạo resources. **Type** là khai báo loại resource mà ta muốn tạo và cuối cùng là Name là tên mà mình đặt cho resource đó => Phục vụ cho việc truy xuất thông tin.
![Define resources template](/images/1.Introduction/01-intro.webp).

- Như hình ở dưới thì đầu tiên ta cần khai báo provider, ở đây là "aws", tạo thêm một EC2 instance loại "t2.micro" có name là "My EC2 Instance" ở region Singapore.

![Terraform example](/images/1.Introduction/02-tf.png)

Để khởi tạo workspace, ta chạy câu lệnh **terraform init** để tải aws provider xuống workspace hiện tại để Terraform có thể sử dụng những provider này và gọi API để khởi tạo resource cho chúng ta. Sau khi chạy xong thì thư mục .terraform xuất hiện trong workspace và chứa code của các providers mà ta đã khai báo.

![Terraform workspace](/images/1.Introduction/03-tf.png)

{{% notice info %}}
Ở đây ta có thể  chạy lệnh **terraform fmt** để chỉnh lại format của các file terraform.
{{% /notice %}}

{{% notice info %}}
Ngoài ra để kiểm tra xem tính hợp lệ của những đoạn code mà ta đã viết, ta có thể dùng lệnh **terraform validate**
{{% /notice %}}

Sau khi khởi tạo xong workspace bằng **terraform init**, ta chạy câu lệnh **terraform plan** để xem thông những resource nào sẽ được tạo/sửa/xóa. ⇒ Tạo thêm 1 resource, 0 sửa, 0 xóa. 
![Terraform output](/images/1.Introduction/04-tf.png)
Sau khi kiểm tra plan thì ta chạy lệnh **terraform apply** để thực hiện những hành động đó. 

{{% notice warning %}}
Hạn chế sửa/xóa resource được tạo bằng Terraform vì sẽ làm mất đi tính trạng thái của resource được quản lý bởi Terraform.
{{% /notice %}}

{{% notice tip %}}
Nếu trong trường hợp đã apply và khởi tạo resource nhưng có sai sót và muốn sửa thì có thể sửa trực tiếp trong code và chạy lệnh **terraform apply** một lần nữa để Terraform xác định sửa đổi để sửa thông tin mà không cần phải xóa resource chạy lại.
{{% /notice %}}

{{% notice tip %}}
Sau khi chạy **terraform plan** nếu sửa code lại thì lệnh apply vẫn sẽ chạy code mới, nếu muốn vẫn chạy code lúc chạy lệnh **terraform plan** thì thêm “-out=\<filename>”. Sau đó chạy **terraform apply \<filename>**.
{{% /notice %}}

![Terraform result](/images/1.Introduction/05-tf.png)
Sau khi chạy xong lệnh apply thì EC2 Instance với tên "My EC2 Instance" có type "t2.micro" đã được tạo.
![Terraform result](/images/1.Introduction/06-tf.png)

Cuối cùng nếu muốn xóa resource thì chỉ cần chạy lệnh “terraform destroy”. Terraform sẽ xóa toàn bộ những resource đã được triển khai từ đầu. 

⇒ Tóm lại, Terraform có thể được hiểu đơn giản là một công cụ quản lý resource state thông qua file terraform.tfstate được tạo khi chạy lệnh **terraform apply**. Và nó còn giúp thực hiện các hành động CRUD lên các resource của infra mình muốn.