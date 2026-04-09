# Payment Service

Dịch vụ này được thiết kế để **tích hợp cổng thanh toán VNPay** cho hệ thống. Nó đóng vai trò sinh URL thanh toán và tiếp nhận callback trả về sau khi người dùng hoàn tất hoặc hủy giao dịch.

## Chức năng

- Cung cấp API `/v1/api/payment/create-payment` để tạo URL thanh toán.
- Ký dữ liệu bằng HMAC SHA-512 theo chuẩn của VNPay.
- Cung cấp endpoint `/v1/api/payment/payment-return` để kiểm tra chữ ký phản hồi và redirect về frontend.

## Đánh giá mã nguồn

Điểm cốt lõi cần nâng cấp ở dịch vụ này là **an toàn cấu hình và tính hoàn chỉnh của quy trình thanh toán**. Hiện tại các thông tin nhạy cảm như `vnp_TmnCode` và `secretKey` đang được đặt trực tiếp trong `PaymentConfig`, rất rủi ro nếu source code bị lộ. Bên cạnh đó, service mới dừng ở mức tạo URL và redirect sau callback, chưa liên kết chặt với `order-service` để cập nhật trạng thái thanh toán của đơn hàng.

Theo chuẩn "Vibe Code", nên đưa toàn bộ secret ra biến môi trường hoặc secret manager, thêm cơ chế đối soát giao dịch/IPN rõ ràng, log theo `orderId/transactionId`, và bảo đảm rằng kết quả thanh toán sẽ cập nhật đúng trạng thái đơn hàng thay vì chỉ redirect người dùng về frontend.
