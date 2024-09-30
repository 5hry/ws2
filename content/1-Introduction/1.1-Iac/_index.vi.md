---
title : "IaC là gì?"
date :  "`r Sys.Date()`" 
weight : 1 
chapter : false
pre : " <b> 1.1 </b> "
---
Thông thường, hầu hết người mới bắt đầu với Cloud nói chung và AWS nói riêng thì đều tiếp xúc với các dịch vụ của AWS qua console, đăng nhập vào console và tương tác với các dịch vụ mình cần. Việc này sẽ khá là nhẹ nhàng thoải mái nếu như chúng ta chỉ tạo một vài ứng dụng nhỏ, không cần phải sử dụng lại cấu hình đó.

Ta có thể hiểu đơn giản theo tên của nó thì ta sẽ viết code để mô tả và chạy những infrastructure của chúng ta. Infrastructure as Code (IaC) giúp chúng ta tương tác với các dịch vụ đó thông qua những dòng code chứ không cần phải lên console để thao tác hoàn toàn bằng tay nữa. 

**Ưu điểm:**
- Có thể tái sử dụng để tạo những ứng dụng khác có nhiều điểm giống nhau để giảm thời gian tạo infra.
- Có cấu trúc rõ ràng, lưu lại trạng của các infra được tạo ra. Bạn hãy thử tưởng tượng việc một hệ thống lớn mà bạn tạo ra infra rồi nhưng qua hôm sau lại quên mất là mình đã tạo chưa.
- Có thể backup lại hạ tầng khi có sự cố xảy ra với hệ thống.