# Order Service

Dịch vụ này được thiết kế như **trung tâm điều phối nghiệp vụ đặt hàng** trong hệ thống. Đây là nơi tiếp nhận yêu cầu tạo đơn từ client, phối hợp với các service khác và phát sự kiện sau khi đơn hàng được ghi nhận.

## Chức năng

- Cung cấp API `/v1/api/order/place-order`, `/v1/api/order/{user_id}/get-orders` và `/v1/api/order/cancel-order`.
- Nhận danh sách sản phẩm từ request đặt hàng.
- Gọi sang `product-service` qua Feign để trừ số lượng tồn kho.
- Lưu đơn hàng vào MongoDB.
- Phát thông điệp Kafka để `notification-service` gửi email cho khách hàng.

## Đánh giá mã nguồn

Điểm cốt lõi cần nâng cấp ở dịch vụ này là **tính nhất quán giao dịch giữa Order và Inventory**. Mã hiện tại đang trừ tồn kho trước rồi mới lưu đơn hàng; nếu bước lưu MongoDB bị lỗi sau đó, hệ thống sẽ rơi vào trạng thái "mất hàng tồn nhưng không có đơn". Ngoài ra, luồng hủy đơn mới chỉ đổi trạng thái sang `CANCELED` mà chưa hoàn lại tồn kho, và event Kafka đang được gửi theo kiểu fire-and-forget nên khó kiểm soát khi downstream gặp lỗi.

Theo chuẩn "Vibe Code", service này nên được nâng lên theo hướng saga/outbox pattern, có cơ chế rollback hoặc bù trừ rõ ràng, thêm validation gắn với người dùng đã đăng nhập thay vì chỉ tin `user_id` từ request, và tách riêng trạng thái thanh toán - trạng thái đơn hàng để đảm bảo luồng nghiệp vụ chặt chẽ hơn.
