# Notification Service

Dịch vụ này được thiết kế thuần túy dưới dạng một **Consumer** xử lý sự kiện bất đồng bộ. Nó không cần cung cấp API nghiệp vụ ra bên ngoài, mà chủ yếu đảm nhận công việc hậu xử lý sau khi đơn hàng đã được tạo.

## Chức năng

- Liên tục lắng nghe topic Kafka `notificationTopic`.
- Khi `order-service` phát ra thông điệp chứa `message` và `userId`, service sẽ nhận dữ liệu đó.
- Gọi sang `user-service` để lấy email của khách hàng.
- Dùng `JavaMailSender` để gửi email thông báo đơn hàng.

## Đánh giá mã nguồn

Điểm cốt lõi cần nâng cấp ở dịch vụ này là **tính idempotent** và **độ bền lỗi của listener**. Trong môi trường mạng thực tế, một thông điệp có thể bị phát lại nhiều lần; với code hiện tại, cùng một sự kiện hoàn toàn có thể dẫn đến việc gửi trùng nhiều email cho khách hàng. Ngoài ra, service đang phụ thuộc vào lời gọi đồng bộ sang `user-service`, chưa có timeout/retry/dead-letter queue, và thông tin SMTP còn đặt cứng trong `application.properties`.

Theo chuẩn "Vibe Code", nên bổ sung một cơ chế dedupe nội bộ bằng Redis hoặc bảng log theo `eventId/orderId`, ghi nhận trạng thái đã gửi mail, đồng thời thêm retry có backoff, DLQ cho Kafka consumer và exception handling chi tiết hơn để việc lỗi SMTP không làm luồng xử lý bị kẹt hoặc mất dấu vết sự cố.
