---
title : "Tạo VPC"
date :  "`r Sys.Date()`" 
weight : 1 
chapter : false
pre : " <b> 3.1. </b> "
---

Để tạo một VPC như kiến trúc high availability ở đầu Workshop, ta cần tạo:
- 1 VPC.
- 2 public subnets.
- 2 private subnets cho EC2 chạy ứng dụng web.
- 2 private subnets cho RDS instance chạy database.
- 1 NAT Gateway cho các EC2 có thể truy cập ra ngoài internet.
- 1 Internet Gateway nhận traffic từ internet vào VPC.

### VPC
Trước tiên, ta tạo một thư mục **vpc** trong thư mục modules, tạo file **main.tf** sau đó vào file **main.tf** gốc để khai báo module **VPC** này bằng cách thêm đoạn code như bên dưới để khai báo module vpc nằm trong thư mục **./modules/vpc** .


```
module "vpc" {
  source = "./modules/vpc"
}
```
Sau khi thêm đoạn code này vào file **main.tf** gốc thì ta cần chạy lại lệnh **terraform init** để Terraform nhận biết được là chúng ta đang chạy những file .tf ở thư mục khác và tải về các thư viện cần thiết nếu cần.

Để tạo một VPC, ta chỉ cần khai báo "resource aws_vpc", sau đó thêm vào các thuộc tính của VPC.

```
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"


  tags = {
    Name = "${var.app_name} VPC"
  }
}
```

Ở đây, ta tạo một VPC có Name là main với CIDR block **"10.0.0.0/16"** và đánh tag Name là **"${var.app_name} VPC"** .  **${var.app_name}** là lấy giá trị biến app_name được khai báo trong file variables.tf cùng cấp. Ví dụ biến app_name ở dưới là dạng chuỗi và mặc định giá trị là "Workshop2" thì Name của VPC được tạo ra sẽ là **"Workshop2 VPC"**.

```
variable "app_name" {
  type = string
  default = "Workshop2"
}
```
Vậy là ta đã khai báo được một VPC với CIDR block là "10.0.0.0/16"


### Public Subnets
Sau khi đã có VPC, ta có thể tạo các resource nằm bên trong VPC này. Đầu tiên ta sẽ tạo 2 Public subnets để chứa NAT gateway cho EC2 trong private subnets có thể truy cập ra Internet. Để tạo các subnets, ta khai báo resource aws_subnet trong file **main.tf**. Ở đây ta cần tạo 2 subnets ở 2 zones, thay vì tạo từng cái thì ta có thể thêm thuộc tính **count** bằng độ dài của biến **public_subnet_cidrs** khai báo dải CIDR của các public subnets bằng 2 để tạo 1 lần 2 subnets. **vpc_id** sẽ gán bằng **id** của VPC vừa được tạo. Ta có thể lấy giá trị bằng cách gọi **resource_type.resource_name.feature**, ví dụ như **aws_vpc.main.id**.
CIDR block thì ta có thể dùng hàm **element** để lấy ra giá trị cidr block nằm trong biến **public_subnet_cidrs** thông qua **count.index** để xác định thứ tự cần lấy.
Tương tự với AZ, mỗi subnet sẽ ở một availability zone tương ứng với giá trị **count.index**. Vậy theo đoạn code bên dưới thì sẽ tạo ra 1 subnet với CIDR block là "10.0.1.0/24" ở zone "ap-southeast-1a" với Name "Workshop2 Public Subnets 1" và 1 subnet với CIDR block là "10.0.2.0/24" ở zone "ap-southeast-1b" với Name "Workshop2 Public Subnets 2"


```
resource "aws_subnet" "public_subnets" {
  count             = length(var.public_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = element(var.public_subnet_cidrs, count.index)
  availability_zone = element(var.azs, count.index)
  tags = {
    Name = "${var.app_name} Public Subnets ${count.index + 1}"
  }
}
```
```
variable "public_subnet_cidrs" {
  type        = list(string)
  description = "Public Subnet CIDR values"
  default     = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "azs" {
  type        = list(string)
  description = "Availability Zones Name"
  default     = ["ap-southeast-1a", "ap-southeast-1b"]
}
```

### Private Web Subnets
Bước tiếp theo, ta tạo 2 private subnets để chứa các EC2 chạy ứng web. Tương tự với tạo public subnets, thì ta sẽ tạo ra được 2 private subnets, 1 subnet với CIDR block là "10.0.3.0/24" ở zone "ap-southeast-1a" với Name "Workshop2 Private Subnets 1" và 1 subnet CIDR block là "10.0.4.0/24" ở zone "ap-southeast-1b" với Name "Workshop2 Private Subnets 2"

```
resource "aws_subnet" "private_subnets" {
  count             = length(var.private_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = element(var.private_subnet_cidrs, count.index)
  availability_zone = element(var.azs, count.index)
  tags = {
    Name = "${var.app_name} Private Subnets ${count.index + 1}"
  }
}
```
```
variable "private_subnet_cidrs" {
  type        = list(string)
  description = "Private Subnet CIDR values"
  default     = ["10.0.3.0/24", "10.0.4.0/24"]
}
variable "azs" {
  type        = list(string)
  description = "Availability Zones Name"
  default     = ["ap-southeast-1a", "ap-southeast-1b"]
}
```

### Private Database Subnets
Tương tự, ta tạo 2 private subnets để chứa 2 RDS instances. Tương tự với tạo ở trên, thì ta sẽ tạo ra được 2 private subnets, 1 subnet với CIDR block là "10.0.5.0/24" ở zone "ap-southeast-1a" với Name "Workshop2 Database Subnets 1" và 1 subnet CIDR block là "10.0.6.0/24" ở zone "ap-southeast-1b" với Name "Workshop2 Database Subnets 2"

```
resource "aws_subnet" "database_subnets" {
  count             = length(var.database_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = element(var.database_subnet_cidrs, count.index)
  availability_zone = element(var.azs, count.index)
  tags = {
    Name = "${var.app_name} Database Subnets ${count.index + 1}"
  }
}
```
```
variable "database_subnet_cidrs" {
  type        = list(string)
  description = "Database Subnet CIDR values"
  default     = ["10.0.5.0/24", "10.0.6.0/24"]
}
variable "azs" {
  type        = list(string)
  description = "Availability Zones Name"
  default     = ["ap-southeast-1a", "ap-southeast-1b"]
}
```





### Internet Gateway
Sau khi ta đã tạo các subnets thì traffic vẫn chưa được route, cần tạo Internet Gateway để traffic có thể ra vào VPC. 
Ta tạo một resource aws_internet_gateway nằm trong vpc trên, được đánh tag Name là "Workshop2 Internet Gateway" để route traffic ra vào VPC.
```
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.app_name} Internet Gateway"
  }
}
```

Sau khi đã có Internet Gateway, ta cần cấu hình route table để có thể route traffic từ public subnet có thể ra vào VPC thông qua Internet Gateway. Ta tạo resource aws_route_table trong vpc main route traffic đến dải CIDR "0.0.0.0/0" đi đến Internet Gateway, và ta sẽ gán 2 public subnets vào route table này dể traffic từ public subnet sẽ được route đến Internet Gateway. Ta tạo thêm resource là **aws_route_table_association** để attach 2 public subnets vào route table đó.
```
resource "aws_route_table" "second_rt" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "${var.app_name} 2nd Route Table"
  }
}

resource "aws_route_table_association" "public_subnets_association" {
  count = length(var.public_subnet_cidrs)
  route_table_id = aws_route_table.second_rt.id
  subnet_id = element(aws_subnet.public_subnets[*].id, count.index)
}
```

### NAT Gateway
Sau khi tạo route table và gán public subnets vào để traffic từ public subnets ra ngoài internet có thể đi qua Internet Gateway, thì nếu EC2 trong Private subnet muốn pull image hay cập nhật bản vá thì cần ra ngoài internet nhưng không thể facing ra internet, vì vậy ta sẽ tạo NAT gateway để EC2 trong Private subnet có thể ra ngoài internet được.
Muốn tạo được NAT gateway thì ta cần phải cấp cho nó một public IP, ta sẽ tạo một Elastic IP để cấp cho NAT gateway này. Tạo một resource aws_eip với **domain = "vpc"** để cấp phát cho NAT trong VPC. Sau đó, khi traffic route đến NAT thì NAT sẽ route đến Internet Gateway để đi ra Internet vì NAT nằm ở public subnets và được cấu hình để ra Internet thông qua Internet Gateway.

```
resource "aws_eip" "nat_gw_eip" {
  domain = "vpc"

  tags = {
    Name = "${var.app_name} EIP NAT"
  }
}
```

Sau khi có được Elastic IP, ta tạo NAT Gateway nằm trong một public subnet đã tạo và được gán cho IP là Elastic IP vừa tạo.

```
resource "aws_nat_gateway" "nat_gw" {
  allocation_id = aws_eip.nat_gw_eip.id
  subnet_id     = aws_subnet.public_subnets[0].id

  tags = {
    Name = "${var.app_name} NAT Gateway"
  }
}
```

Tương tự khi tạo Internet Gateway, ta cần chỉ cho private subnets biết được đường đi ra ngoài Internet bằng cách cấu hình route table route traffic từ Private subnets tới NAT Gateway, đoạn code dưới cũng tương tự đoạn code tạo route table cho Internet Gateway chỉ khác là đổi từ Public subnets thành Private subnet và đổi từ Internet Gateway ID thành NAT Gateway ID.

```
resource "aws_route_table" "nat_gw_rt" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_nat_gateway.nat_gw.id
  }

  tags = {
    Name = "${var.app_name} NAT Route Table"
  }
}

resource "aws_route_table_association" "private_subnet_asso_natgw" {
  count = length(var.private_subnet_cidrs)
  route_table_id = aws_route_table.nat_gw_rt.id
  subnet_id = element(aws_subnet.private_subnets[*].id, count.index)
}
```


Vậy là sau viết những đoạn code để tạo một VPC để chứa web application thì chúng ta sẽ chạy thử để xem kết quả. Mở Terminal chạy **terraform plan** để xem những resources nào sẽ được tạo và **terraform apply** và điền **yes** để xác nhận tạo những resources đó.

![Kết quả sau khi chạy apply](/images/3.initial/result-VPC.png)

Và kết quả là ta đã tạo được một VPC để chứa các tài nguyên, traffic từ private subnets được route đến NAT gateway và traffic từ public subnets được route đến Internet Gateway trong khi database subnet không cần route traffic vì các RDS là Managed Service nên được AWS quản lý hoàn toàn.

Sau khi đã kiểm tra tất cả những resources được tạo ra đã chính xác với mong muốn thì ta có thể dùng lệnh **terraform destroy** để xóa toàn bộ những resources đó để tiết kiệm $$$ rồi tiếp tục viết code để tạo thêm những modules khác.