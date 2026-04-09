# Chương 5: Demo, Kiểm thử và Hướng dẫn triển khai

## 5.1. Hướng dẫn khởi chạy hệ thống sau khi Rebuild

Sau khi thực hiện tái cấu trúc theo định hướng ở Chương 4, hệ thống backend `ProjectWeb-BE` vẫn giữ kiến trúc microservices, nhưng được chuẩn hóa hơn về cấu trúc thư mục, cấu hình môi trường và cách khởi chạy. Việc triển khai có thể thực hiện theo hai hình thức chính: **chạy bằng Docker Compose** hoặc **chạy local từng service để phục vụ phát triển và debug**.

### 5.1.1. Cách 1: Khởi chạy bằng Docker Compose

Đây là cách phù hợp nhất khi cần dựng nhanh toàn bộ hệ thống để demo hoặc kiểm thử tích hợp.

#### Bước 1: Chuẩn bị môi trường

Cần cài đặt sẵn các công cụ sau:

- Docker Desktop
- Docker Compose
- JDK 21
- Maven
- Git

#### Bước 2: Di chuyển vào thư mục backend

```bash
cd ProjectWeb-BE
```

#### Bước 3: Build source code

```bash
mvn clean install
```

Lệnh trên giúp build toàn bộ các module gồm:

- `api-gateway`
- `eureka-server`
- `user-service`
- `product-service`
- `order-service`
- `notification-service`
- `payment-service`

#### Bước 4: Khởi động toàn bộ hệ thống bằng Docker

```bash
docker compose up -d --build
```

Khi chạy thành công, hệ thống sẽ khởi tạo đồng thời:

- **Hạ tầng phụ trợ**: `mongo`, `redis`, `kafka-1`, `kafka-2`, `kafka-3`, `kafka-ui`, `zipkin`, `prometheus`
- **Các service backend**: `eureka-server`, `user-service`, `product-service`, `order-service`, `notification-service`, `payment-service`, `api-gateway`

#### Bước 5: Kiểm tra trạng thái hoạt động

Sau khi khởi động, có thể truy cập các địa chỉ sau để kiểm tra:

| Thành phần | Địa chỉ kiểm tra |
|---|---|
| Eureka Dashboard | `http://localhost:8761` |
| API Gateway | `http://localhost:8181` |
| Kafka UI | `http://localhost:8080` |
| Prometheus | `http://localhost:9090` |
| Zipkin | `http://localhost:9411` |

Nếu các service hiển thị trạng thái `UP` trên Eureka thì hệ thống đã sẵn sàng để demo.

---

### 5.1.2. Cách 2: Chạy local từng service

Hình thức này phù hợp khi cần kiểm tra riêng từng service hoặc debug source code trực tiếp trong IDE.

#### Bước 1: Khởi động hạ tầng dùng chung

```bash
docker compose up -d redis mongo kafka-1 kafka-2 kafka-3 kafka-ui zipkin prometheus eureka-server
```

#### Bước 2: Chạy từng service bằng Maven

Mở nhiều terminal và thực hiện lần lượt:

```bash
cd user-service
mvn spring-boot:run
```

```bash
cd product-service
mvn spring-boot:run
```

```bash
cd order-service
mvn spring-boot:run
```

```bash
cd notification-service
mvn spring-boot:run
```

```bash
cd payment-service
mvn spring-boot:run
```

```bash
cd api-gateway
mvn spring-boot:run
```

#### Thứ tự khởi động khuyến nghị

1. `eureka-server`
2. `user-service`
3. `product-service`
4. `order-service`
5. `notification-service`
6. `payment-service`
7. `api-gateway`

Việc khởi động theo thứ tự này giúp hạn chế lỗi đăng ký service hoặc lỗi gọi chéo giữa các service trong giai đoạn khởi tạo.

---

### 5.1.3. Một số lưu ý sau khi rebuild

Để đảm bảo hệ thống chạy ổn định hơn sau khi rebuild, cần lưu ý:

- Tất cả thông tin nhạy cảm như JWT secret, SMTP credential, VNPay secret nên được đưa ra biến môi trường.
- Các endpoint public/private nên được kiểm soát qua `api-gateway`.
- Nên kiểm tra log ở `order-service` và `notification-service` khi demo luồng đặt hàng.
- Nếu dùng Mailtrap để demo email, cần cấu hình tài khoản thử nghiệm hợp lệ.

---

## 5.2. Kịch bản kiểm thử (Test Cases)

Sau khi hệ thống được khởi động thành công, tiến hành demo và kiểm thử theo các kịch bản chính dưới đây.

### 5.2.1. Test Case 1: Tạo sản phẩm mới

#### Mục tiêu

Kiểm tra `product-service` có nhận request tạo sản phẩm mới, lưu dữ liệu vào MongoDB và trả về kết quả thành công hay không.

#### API sử dụng

- **Method**: `POST`
- **Endpoint**: `http://localhost:8181/v1/api/products`

#### Request body mẫu

```json
{
  "name": "Laptop Gaming MSI",
  "price": 25000000,
  "description": "Laptop phục vụ chơi game và học tập",
  "image": "https://example.com/msi.jpg",
  "checkToCart": false,
  "quantity": 10,
  "productCode": "MSI-01"
}
```

#### Kết quả mong đợi

- Server trả về `200 OK`.
- Response chứa thông báo thêm sản phẩm thành công.
- Sản phẩm mới được lưu vào MongoDB.
- Khi gọi `GET /v1/api/products`, sản phẩm vừa tạo phải xuất hiện trong danh sách.

#### Bảng test case

| Thành phần | Nội dung |
|---|---|
| Mã test case | `TC-01` |
| Tên test | Tạo mới sản phẩm |
| Tiền điều kiện | `product-service` và `api-gateway` đang chạy |
| Input | JSON thông tin sản phẩm |
| Kết quả kỳ vọng | Sản phẩm được tạo thành công và truy vấn lại được |
| Trạng thái | Đạt / Không đạt |

> **Hình 5.1.** Kết quả Postman khi tạo sản phẩm mới thành công.  
> *Chèn ảnh chụp màn hình tại đây.*

---

### 5.2.2. Test Case 2: Tạo đơn hàng và kiểm tra luồng liên lạc `Order -> Product -> Notification`

#### Mục tiêu

Kiểm tra toàn bộ chuỗi nghiệp vụ quan trọng nhất của hệ thống sau khi rebuild, bao gồm:

1. Người dùng gửi request đặt hàng vào `order-service`.
2. `order-service` gọi sang `product-service` để giảm tồn kho.
3. `order-service` lưu đơn hàng vào MongoDB.
4. `order-service` phát event lên Kafka.
5. `notification-service` nhận event và thực hiện gửi email thông báo.

#### API sử dụng

- **Method**: `POST`
- **Endpoint**: `http://localhost:8181/v1/api/order/place-order`

#### Request body mẫu

```json
{
  "user_id": "USER_ID_DEMO",
  "paymentMethod": "COD",
  "status": "PENDING",
  "items": [
    {
      "id": "PRODUCT_ID_DEMO",
      "name": "Laptop Gaming MSI",
      "price": 25000000,
      "quantity": 1
    }
  ]
}
```

#### Kết quả mong đợi

- `order-service` trả về `200 OK` cùng dữ liệu đơn hàng.
- Số lượng tồn kho của sản phẩm trong `product-service` giảm đi tương ứng.
- Trong log của `order-service` xuất hiện thông tin gửi event Kafka thành công.
- `notification-service` nhận được message và thực hiện gửi email.
- Nếu dùng Mailtrap, email thông báo sẽ xuất hiện trong hộp thư test.

#### Bảng test case tích hợp

| Thành phần | Nội dung |
|---|---|
| Mã test case | `TC-02` |
| Tên test | Đặt hàng và kiểm tra luồng tích hợp |
| Tiền điều kiện | Đã có user và product hợp lệ trong database |
| Input | Request JSON tạo đơn hàng |
| Kết quả kỳ vọng | Đơn hàng được lưu, tồn kho giảm, event được phát, notification được xử lý |
| Trạng thái | Đạt / Không đạt |

### Luồng xử lý mong đợi

```text
Client
   -> API Gateway
      -> Order Service
         -> Product Service (decrease quantity)
         -> MongoDB (save order)
         -> Kafka (publish order event)
            -> Notification Service
               -> User Service (get email)
               -> Email Service / Mailtrap
```

### Các bước kiểm tra cụ thể

1. Gửi request tạo đơn hàng bằng Postman.
2. Gọi lại API lấy danh sách đơn hàng theo `user_id` để xác nhận đơn đã được lưu.
3. Gọi API lấy chi tiết sản phẩm để kiểm tra `quantity` đã giảm.
4. Kiểm tra console log của `notification-service`.
5. Kiểm tra email test trên Mailtrap (nếu cấu hình sẵn).

> **Hình 5.2.** Kết quả API đặt hàng thành công.  
> *Chèn ảnh chụp màn hình phản hồi JSON tại đây.*
>
> **Hình 5.3.** Log tại `notification-service` khi nhận event Kafka.  
> *Chèn ảnh log tại đây.*
>
> **Hình 5.4.** Email thông báo hiển thị trên Mailtrap.  
> *Chèn ảnh chụp màn hình tại đây.*

---

### 5.2.3. Đánh giá kết quả kiểm thử

Qua hai test case trọng tâm, có thể xác nhận rằng phiên bản backend sau khi rebuild có thể được đánh giá theo ba khía cạnh chính:

- **Chạy được nghiệp vụ cốt lõi**: tạo sản phẩm, đặt hàng, gửi thông báo.
- **Thể hiện được sự phối hợp giữa nhiều microservice** thay vì chỉ kiểm thử từng service đơn lẻ.
- **Phù hợp để demo đồ án** vì có cả API đồng bộ và luồng xử lý bất đồng bộ.

---

## 5.3. Link Video Demo (Yêu cầu bắt buộc)

Để đáp ứng yêu cầu của báo cáo, cần đính kèm một video quay lại toàn bộ quá trình:

- phân tích nhanh mã nguồn,
- khởi động hệ thống,
- chạy thử các API,
- demo luồng đặt hàng và gửi notification.

### Thông tin link video demo

- **YouTube**: `Dán link YouTube tại đây`
- **Google Drive**: `Dán link Google Drive tại đây`

> Ví dụ trình bày trong báo cáo:  
> **Video demo hệ thống:** `https://youtu.be/your-demo-link`  
> hoặc  
> **Video demo hệ thống:** `https://drive.google.com/file/d/your-demo-id/view`

### Nội dung nên có trong video demo

1. Giới thiệu nhanh cấu trúc dự án `ProjectWeb-BE`.
2. Mô tả các service chính: `api-gateway`, `user-service`, `product-service`, `order-service`, `notification-service`, `payment-service`.
3. Chạy hệ thống bằng Docker hoặc local.
4. Thực hiện test case tạo sản phẩm.
5. Thực hiện test case tạo đơn hàng.
6. Mở log để chứng minh luồng `Order -> Product -> Notification` hoạt động.
7. Kết luận ngắn về kết quả rebuild.

---

## Kết luận chương 5

Chương 5 đã trình bày cách triển khai hệ thống sau khi rebuild, đồng thời xây dựng các kịch bản kiểm thử trọng tâm để phục vụ demo đồ án. Việc lựa chọn hai test case chính — **tạo sản phẩm mới** và **tạo đơn hàng có liên lạc liên service** — là phù hợp vì vừa thể hiện được nghiệp vụ cốt lõi, vừa chứng minh được tính chất microservices của hệ thống. Bên cạnh đó, phần video demo là minh chứng trực quan bắt buộc, giúp người xem đánh giá rõ hơn chất lượng và mức độ hoàn thiện của giải pháp.
