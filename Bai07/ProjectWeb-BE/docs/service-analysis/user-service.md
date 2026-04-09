# User Service

Dịch vụ này được thiết kế để xử lý **đăng ký, đăng nhập và quản lý thông tin người dùng**. Đây là service chịu trách nhiệm phát hành JWT, lưu trạng thái token trên Redis và cung cấp dữ liệu email cho các service khác khi cần.

## Chức năng

- Cung cấp API đăng ký (`/register`), đăng nhập (`/login`), đăng xuất (`/logout`) và lấy thông tin user.
- Mã hóa mật khẩu bằng `BCryptPasswordEncoder` trước khi lưu vào MongoDB.
- Sinh JWT sau khi đăng ký hoặc đăng nhập thành công.
- Lưu token vào Redis để phục vụ kiểm tra phiên đăng nhập và logout.
- Hỗ trợ endpoint lấy email theo `userId` để `notification-service` sử dụng.

## Đánh giá mã nguồn

Điểm cốt lõi cần nâng cấp ở dịch vụ này là **vòng đời xác thực và độ ổn định của JWT**. Trong `JWTUtil`, secret key đang được sinh ngẫu nhiên mỗi lần service khởi động bằng `Keys.secretKeyFor(...)`, điều này khiến toàn bộ token cũ mất hiệu lực sau mỗi lần restart hoặc deploy. Bên cạnh đó, nhiều endpoint đang mở khá thoáng và việc logout dùng `GET` cũng chưa đúng tinh thần REST.

Theo chuẩn "Vibe Code", nên chuyển secret JWT sang biến môi trường cố định, thêm filter/interceptor để xác thực token tập trung, giới hạn rõ endpoint public/private, chuyển logout sang `POST`, và chuẩn hóa cơ chế revocation/expiration để phần xác thực không chỉ "phát token" mà còn quản lý vòng đời token một cách nhất quán.
