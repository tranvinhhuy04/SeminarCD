# Product Service

Dịch vụ này được thiết kế để quản lý **danh mục sản phẩm và tồn kho**. Đây là service cung cấp dữ liệu sản phẩm cho frontend và đồng thời hỗ trợ các service khác trong việc kiểm tra, cập nhật số lượng hàng hóa.

## Chức năng

- Cung cấp API lấy chi tiết sản phẩm, danh sách sản phẩm có phân trang và API tạo mới sản phẩm.
- Cho phép giảm số lượng tồn kho thông qua endpoint `/{id}/decrease-quantity`.
- Có sử dụng cache Redis cho danh sách sản phẩm nhằm tăng tốc độ phản hồi.
- Đang có thêm các hàm `add-to-cart` và `remove-from-cart` ở mức service.

## Đánh giá mã nguồn

Điểm cốt lõi cần nâng cấp ở dịch vụ này là **ranh giới nghiệp vụ và tính an toàn dữ liệu tồn kho**. Phần catalog sản phẩm được tổ chức khá rõ, nhưng hai hàm `addToCart()` và `removeFromCart()` đang lẫn giữa khái niệm "sản phẩm" và "giỏ hàng"; đặc biệt `removeFromCart()` hiện lại xóa luôn bản ghi sản phẩm khỏi database, đây là rủi ro rất lớn nếu chạy thật. Ngoài ra, thao tác `decrease-quantity` chưa có cơ chế khóa hoặc cập nhật nguyên tử nên vẫn có nguy cơ oversell khi nhiều người mua cùng lúc.

Theo chuẩn "Vibe Code", nên tách giỏ hàng sang một domain/service riêng, bổ sung optimistic locking hoặc stock reservation cho tồn kho, định nghĩa TTL/invalidation rõ ràng cho cache Redis và bỏ endpoint `/test` ra khỏi luồng production để service gọn và sạch hơn.
