# Chương 3: Đánh giá mã nguồn gốc & Kết quả chạy thử

## 3.1. Thiết lập môi trường và cấu hình chạy thử mã nguồn gốc

Mã nguồn gốc của hệ thống được tổ chức theo kiến trúc **microservices** với các dịch vụ chính gồm: `api-gateway`, `eureka-server`, `user-service`, `product-service`, `order-service`, `notification-service` và `payment-service`. Hệ thống sử dụng Spring Boot làm nền tảng phát triển backend, kết hợp với MongoDB, Redis, Kafka, Docker và Eureka để phục vụ cho việc chạy thử và đánh giá.

### 3.1.1. Môi trường phần mềm sử dụng

Để chạy thử mã nguồn gốc, môi trường được thiết lập với các công cụ sau:

| Thành phần             | Phiên bản / Công cụ    |
| ------------------------ | -------------------------- |
| Hệ điều hành         | Windows 10/11              |
| JDK                      | Java 21 (theo `pom.xml`) |
| Build tool               | Maven                      |
| IDE                      | IntelliJ IDEA / VS Code    |
| Cơ sở dữ liệu        | MongoDB                    |
| Cache                    | Redis                      |
| Message Broker           | Apache Kafka               |
| Containerization         | Docker, Docker Compose     |
| Công cụ kiểm thử API | Postman                    |

### 3.1.2. Cấu hình các dịch vụ chính

Dựa trên mã nguồn gốc và các file `application.properties`, hệ thống sử dụng các cổng mặc định như sau:

| Dịch vụ                |     Cổng | Vai trò                                          |
| ------------------------ | --------: | ------------------------------------------------- |
| `eureka-server`        |  `8761` | Service Discovery                                 |
| `api-gateway`          |  `8181` | Cổng vào chung cho toàn bộ API                |
| `user-service`         |  `8081` | Quản lý người dùng, đăng ký, đăng nhập |
| `product-service`      |  `8082` | Quản lý sản phẩm và tồn kho                 |
| `order-service`        |  `8083` | Xử lý đặt hàng                               |
| `notification-service` |  `8084` | Gửi thông báo email qua Kafka                  |
| `payment-service`      |  `8085` | Tích hợp tạo liên kết thanh toán            |
| `mongo`                | `27017` | Lưu trữ dữ liệu                               |
| `redis`                |  `6379` | Lưu cache / token                                |
| `prometheus`           |  `9090` | Thu thập metrics                                 |
| `zipkin`               |  `9411` | Tracing                                           |
| `kafka-ui`             |  `8080` | Quan sát Kafka cluster                           |

### 3.1.3. Các bước chạy thử mã nguồn gốc

Quá trình thiết lập và chạy thử được thực hiện theo thứ tự sau:

1. **Khởi động hạ tầng phụ trợ** bằng Docker Compose:

```bash
docker compose up -d redis mongo kafka-1 kafka-2 kafka-3 kafka-ui zipkin prometheus eureka-server
```

2. **Build toàn bộ backend** tại thư mục `ProjectWeb-BE`:

```bash
mvn clean install
```

3. **Khởi động các service nghiệp vụ** theo thứ tự khuyến nghị:

- `user-service`
- `product-service`
- `order-service`
- `notification-service`
- `payment-service`
- `api-gateway`

Có thể chạy từng service bằng lệnh:

```bash
cd user-service
mvn spring-boot:run
```

hoặc dùng Docker để chạy đồng thời toàn bộ hệ thống:

```bash
docker compose up -d
```

4. **Kiểm tra đăng ký service** tại địa chỉ:

```text
http://localhost:8761
```

Khi hệ thống hoạt động đúng, các service sẽ xuất hiện trên Eureka Dashboard với trạng thái `UP`.

### 3.1.4. Cấu hình môi trường chạy thử

Trong mã nguồn gốc, các biến cấu hình chính đã được khai báo thông qua `docker-compose.yml` và file cấu hình của từng service, bao gồm:

- `EUREKA_SERVER_URL=http://eureka-server:8761/eureka/`
- `SPRING_DATA_MONGODB_URI=mongodb://mongo:27017/database`
- `SPRING_DATA_REDIS_HOST=redis`
- `KAFKA_BOOTSTRAP_SERVERS=kafka-1:9092,kafka-2:9094,kafka-3:9096`
- `ZIPKIN_URL=http://zipkin:9411/api/v2/spans`
- `FRONTEND_ALLOWED_ORIGINS=http://localhost:5173,http://frontend:5173`

Nhìn chung, mã nguồn gốc đã có cấu trúc cấu hình tương đối rõ ràng, thuận lợi cho việc dựng lại môi trường chạy thử trên máy cá nhân hoặc trong container.

---

## 3.2. Quá trình chạy thử

Sau khi hoàn tất việc khởi động hệ thống, nhóm tiến hành kiểm tra các API chính bằng **Postman** thông qua `api-gateway` hoặc gọi trực tiếp từng service để xác nhận các chức năng nền tảng của hệ thống hoạt động đúng.

### 3.2.1. Kiểm tra trạng thái hệ thống trên Eureka

Bước đầu tiên là truy cập `http://localhost:8761` để xác nhận các service đã đăng ký thành công. Đây là dấu hiệu cho thấy quá trình khởi động ban đầu ổn định và các service có thể giao tiếp với nhau thông qua cơ chế service discovery.

> **Hình 3.1.** Dashboard Eureka hiển thị các service ở trạng thái `UP`.
>
> *Chèn ảnh chụp màn hình Eureka tại đây.*

### 3.2.2. Kiểm thử nhóm API người dùng

Nhóm API của `user-service` được thử nghiệm đầu tiên vì đây là đầu vào quan trọng cho các chức năng còn lại của hệ thống. Các thao tác được kiểm tra gồm:

- Đăng ký tài khoản mới: `POST /v1/api/user/register`
- Đăng nhập: `POST /v1/api/user/login`
- Lấy thông tin email theo `userId`: `GET /v1/api/user/{id}/email`

Kết quả trả về ở các trường hợp hợp lệ cho thấy service có thể nhận dữ liệu JSON, mã hóa mật khẩu, sinh JWT và trả thông tin người dùng về cho client.

> **Hình 3.2.** API đăng ký / đăng nhập người dùng trả về thành công trên Postman.
>
> *Chèn ảnh chụp màn hình kết quả `200 OK` tại đây.*

### 3.2.3. Kiểm thử nhóm API sản phẩm

Tiếp theo, nhóm thực hiện chạy thử `product-service` với các thao tác cơ bản:

- Lấy danh sách sản phẩm: `GET /v1/api/products?page=0&size=9`
- Lấy chi tiết sản phẩm: `GET /v1/api/products/{id}`
- Thêm sản phẩm mới: `POST /v1/api/products`

Kết quả trả về cho thấy service có thể truy xuất dữ liệu từ MongoDB, thực hiện phân trang và trả về đúng cấu trúc dữ liệu theo yêu cầu của frontend.

> **Hình 3.3.** API lấy danh sách sản phẩm và thêm sản phẩm hoạt động thành công.
>
> *Chèn ảnh chụp màn hình Postman tại đây.*

### 3.2.4. Kiểm thử chức năng đặt hàng

Sau khi có dữ liệu người dùng và sản phẩm, nhóm tiến hành thử API đặt hàng của `order-service`:

- Đặt hàng: `POST /v1/api/order/place-order`
- Xem danh sách đơn hàng theo người dùng: `GET /v1/api/order/{user_id}/get-orders`
- Hủy đơn hàng: `POST /v1/api/order/cancel-order`

Khi request hợp lệ, hệ thống thực hiện gọi sang `product-service` để trừ số lượng tồn kho, lưu đơn hàng vào MongoDB và phát thông điệp Kafka để `notification-service` xử lý gửi email.

> **Hình 3.4.** API đặt hàng trả về thành công, dữ liệu đơn hàng được lưu và trả về cho client.
>
> *Chèn ảnh chụp màn hình phản hồi JSON của đơn hàng tại đây.*

### 3.2.5. Kiểm thử chức năng thanh toán

Với `payment-service`, nhóm kiểm tra API:

- Tạo liên kết thanh toán: `GET /v1/api/payment/create-payment?amount=100000`

Kết quả trả về là một URL thanh toán chứa các tham số đã được ký theo chuẩn VNPay. Điều này chứng minh service đã thực hiện được bước chuẩn bị giao dịch và chuyển hướng người dùng sang cổng thanh toán bên ngoài.

> **Hình 3.5.** API tạo liên kết thanh toán trả về URL hợp lệ.
>
> *Chèn ảnh chụp màn hình kết quả từ Postman tại đây.*

### 3.2.6. Kiểm tra notification sau đặt hàng

Do `notification-service` không cung cấp API nghiệp vụ trực tiếp, việc chạy thử được quan sát thông qua:

- log ở console khi `@KafkaListener` nhận được message,
- hoặc email được gửi ra môi trường Mailtrap.

Khi `order-service` phát sự kiện đặt hàng thành công, `notification-service` sẽ nhận dữ liệu, gọi sang `user-service` để lấy email và thực hiện gửi thông báo.

> **Hình 3.6.** Log nhận message Kafka / email thông báo sau khi đặt hàng.
>
> *Chèn ảnh chụp màn hình log hoặc hộp thư Mailtrap tại đây.*

### 3.2.7. Tổng hợp kết quả chạy thử

Bảng sau đây tóm tắt nhanh kết quả kiểm tra các nhóm API chính:

| STT | API kiểm thử                         | Phương thức | Kết quả mong đợi                                   |
| --: | -------------------------------------- | -------------- | ------------------------------------------------------ |
|   1 | `/v1/api/user/register`              | `POST`       | Tạo tài khoản mới thành công                     |
|   2 | `/v1/api/user/login`                 | `POST`       | Trả về token và thông tin người dùng            |
|   3 | `/v1/api/products`                   | `GET`        | Trả danh sách sản phẩm có phân trang             |
|   4 | `/v1/api/products`                   | `POST`       | Thêm sản phẩm thành công                          |
|   5 | `/v1/api/order/place-order`          | `POST`       | Tạo đơn hàng và lưu dữ liệu thành công       |
|   6 | `/v1/api/order/{user_id}/get-orders` | `GET`        | Lấy được danh sách đơn hàng theo người dùng |
|   7 | `/v1/api/payment/create-payment`     | `GET`        | Sinh URL thanh toán hợp lệ                          |
|   8 | Kafka `notificationTopic`            | Event          | Notification service nhận và xử lý thông báo     |

Kết quả chạy thử cho thấy mã nguồn gốc đã có thể triển khai được một chuỗi nghiệp vụ cơ bản của một hệ thống thương mại điện tử nhỏ theo hướng microservices, bao gồm: đăng nhập người dùng, xem sản phẩm, đặt hàng, thanh toán và gửi thông báo.

---

## 3.3. Đánh giá chất lượng mã nguồn

Qua quá trình đọc mã nguồn và chạy thử các chức năng chính, có thể đánh giá chất lượng mã nguồn gốc theo hai nhóm: **điểm mạnh** và **điểm yếu** như sau.

### Điểm mạnh

1. **Kiến trúc microservices tương đối rõ ràng**Các chức năng được tách thành nhiều service riêng biệt như người dùng, sản phẩm, đơn hàng, thanh toán và thông báo. Cách tổ chức này giúp dễ mở rộng và dễ phân chia phạm vi phát triển.
2. **Có áp dụng các thành phần phổ biến trong hệ sinh thái cloud-native**Dự án đã sử dụng `Eureka` để service discovery, `Spring Cloud Gateway` để định tuyến, `Kafka` để giao tiếp bất đồng bộ, `Redis` để cache, `Zipkin` và `Prometheus` để hỗ trợ giám sát hệ thống.
3. **Hỗ trợ triển khai container hóa**Việc có sẵn `Dockerfile` cho từng service và `docker-compose.yml` cho toàn bộ hệ thống giúp việc dựng môi trường chạy thử trở nên thuận tiện hơn.
4. **Luồng nghiệp vụ chính đã hình thành tương đối đầy đủ**Hệ thống đã có các chức năng cốt lõi của một website bán hàng như quản lý người dùng, quản lý sản phẩm, tạo đơn hàng, tạo liên kết thanh toán và gửi email thông báo.
5. **Cấu trúc dự án tương đối dễ tiếp cận**
   Các package như `controller`, `service`, `repository`, `config`, `model` đã được phân chia khá quen thuộc, giúp người mới dễ đọc và lần theo luồng xử lý.

### Điểm yếu

1. **Chưa đảm bảo tính an toàn bảo mật ở mức production**Một số thông tin nhạy cảm như tài khoản SMTP hoặc khóa tích hợp thanh toán đang được đặt trực tiếp trong source code / file cấu hình. Điều này tiềm ẩn rủi ro lộ lọt thông tin khi đưa lên Git hoặc triển khai thật.
2. **Xử lý xác thực người dùng còn thiếu tính ổn định**Trong `user-service`, khóa bí mật JWT đang được sinh ngẫu nhiên mỗi lần service khởi động, khiến token cũ bị vô hiệu sau mỗi lần restart. Đây là điểm yếu lớn nếu hệ thống triển khai thực tế.
3. **Thiếu cơ chế đảm bảo nhất quán dữ liệu liên service**Ở `order-service`, số lượng sản phẩm bị trừ trước khi đơn hàng được lưu hoàn tất. Nếu có lỗi ở bước sau, hệ thống có thể rơi vào trạng thái dữ liệu không đồng nhất giữa kho hàng và đơn hàng.
4. **Notification service chưa có cơ chế chống xử lý trùng lặp**Dịch vụ gửi mail hiện mới lắng nghe Kafka và gửi email trực tiếp, chưa có cơ chế idempotent hoặc dead-letter queue. Trong môi trường thật, điều này có thể gây gửi trùng email hoặc khó khôi phục khi lỗi xảy ra.
5. **Một số xử lý nghiệp vụ còn lẫn trách nhiệm**Ví dụ `product-service` đang chứa các thao tác `add-to-cart` và `remove-from-cart`, trong khi giỏ hàng nên là một domain độc lập. Điều này khiến ranh giới nghiệp vụ chưa thật sự rõ ràng.
6. **Thiếu lớp phòng vệ như retry, circuit breaker, timeout rõ ràng**Các lời gọi giữa service hiện chủ yếu là trực tiếp, chưa thấy bổ sung đầy đủ cơ chế chịu lỗi cho môi trường phân tán.
7. **Bộ kiểm thử tự động còn mỏng**
   Dự án đã có một số file test, nhưng phần lớn mới dừng ở mức `contextLoads()` hoặc một vài kiểm thử cơ bản, chưa đủ để bao phủ các tình huống lỗi và hồi quy quan trọng.

### Nhận xét tổng quát

Tổng thể, mã nguồn gốc của hệ thống có **nền tảng kiến trúc khá tốt**, phù hợp để phát triển thành một đồ án hoặc dự án học thuật về microservices. Tuy nhiên, khi đánh giá ở góc độ chất lượng phần mềm và khả năng triển khai thực tế, hệ thống vẫn còn nhiều điểm cần cải thiện về **bảo mật, tính nhất quán dữ liệu, khả năng chịu lỗi và độ hoàn thiện của kiểm thử**. Đây cũng chính là cơ sở để đề xuất các hướng nâng cấp trong các chương tiếp theo.
