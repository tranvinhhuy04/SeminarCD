# API Gateway

Dịch vụ này được thiết kế như **cổng vào duy nhất** cho toàn bộ hệ thống microservices. Nó không xử lý nghiệp vụ riêng, mà đóng vai trò tiếp nhận request từ frontend rồi định tuyến đến `user-service`, `product-service`, `order-service` và `payment-service`.

## Chức năng

- Nhận các request từ client tại port `8181`.
- Định tuyến theo path `/v1/api/products/**`, `/v1/api/order/**`, `/v1/api/user/**`, `/v1/api/payment/**`.
- Bật CORS cho frontend và áp dụng giới hạn tốc độ (`RequestRateLimiter`) cho route sản phẩm.
- Có ý đồ cache response qua Redis để giảm tải cho downstream service.

## Đánh giá mã nguồn

Điểm cốt lõi cần nâng cấp ở dịch vụ này là **bảo mật và khả năng chịu lỗi ở lớp gateway**. Hiện tại `SecurityConfig` đang để `anyExchange().permitAll()`, nghĩa là gateway gần như chưa thực hiện vai trò chặn truy cập hay kiểm tra JWT một cách tập trung. Ngoài ra, `CachingGatewayFilter` đang khai báo `RedisTemplate` nhưng chưa được inject đúng cách, nên rất dễ phát sinh `NullPointerException` khi chạy thực tế.

Theo chuẩn "Vibe Code", gateway nên trở thành lớp bảo vệ thật sự của hệ thống: xác thực token ngay từ cửa vào, thêm timeout/circuit breaker/retry cho từng route, chỉ cache các response hợp lệ (`GET`, `200 OK`), và đưa cấu hình CORS/rate-limit ra biến môi trường để dễ triển khai ở nhiều môi trường khác nhau.
