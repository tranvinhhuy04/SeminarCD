# Eureka Server

Dịch vụ này được thiết kế như **trung tâm đăng ký và khám phá service** cho toàn bộ hệ thống. Nó không phục vụ API nghiệp vụ cho người dùng cuối, mà giúp các service khác tìm thấy nhau một cách động thay vì phải hard-code địa chỉ.

## Chức năng

- Cung cấp registry để các service như `api-gateway`, `user-service`, `product-service`, `order-service`, `payment-service`, `notification-service` đăng ký vào.
- Cho phép các service tra cứu lẫn nhau thông qua tên service (`lb://service-name`).
- Đóng vai trò nền tảng cho kiến trúc microservices và cân bằng tải ở mức service discovery.

## Đánh giá mã nguồn

Điểm cốt lõi cần nâng cấp ở dịch vụ này là **độ ổn định vận hành và an toàn cấu hình**. Phần code hiện tại khá gọn, đúng vai trò của một Eureka Server, nhưng đang dùng tài khoản `admin/password` tương đối đơn giản và cấu hình `enable-self-preservation: false`, điều này không phù hợp nếu đưa lên môi trường production thật.

Theo hướng "Vibe Code", service này nên được giữ tối giản nhưng chắc chắn: dùng secret từ biến môi trường, giới hạn truy cập dashboard, bổ sung health-check rõ ràng và cân nhắc triển khai nhiều node để tránh tình trạng registry trở thành điểm lỗi đơn (`single point of failure`).
