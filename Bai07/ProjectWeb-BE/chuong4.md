# Chương 4: Tái cấu trúc (Rebuild) theo tiêu chuẩn "Vibe Code"

## 4.1. Chuẩn hóa Cấu trúc và Nền tảng

Sau khi đánh giá mã nguồn gốc ở Chương 3, có thể nhận thấy hệ thống backend đã có nền tảng microservices tương đối đầy đủ, tuy nhiên vẫn còn tồn tại nhiều vấn đề về tính nhất quán, khả năng bảo trì, bảo mật và độ bền khi triển khai thực tế. Vì vậy, chương này đề xuất hướng **tái cấu trúc (rebuild)** mã nguồn theo tiêu chuẩn "Vibe Code" — tức là hướng đến mã nguồn **gọn gàng, dễ hiểu, tách trách nhiệm rõ ràng, dễ mở rộng, dễ kiểm thử và an toàn hơn khi vận hành**.

Trong phạm vi dự án `ProjectWeb-BE`, việc rebuild không nhằm thay đổi toàn bộ kiến trúc, mà tập trung vào các phần quan trọng nhất: chuẩn hóa nền tảng, tối ưu API Gateway, refactor `product-service` và `order-service`, cải tiến `notification-service`, đồng thời nâng cao chất lượng tương tác với cơ sở dữ liệu.

### 4.1.1. Cập nhật phiên bản framework / thư viện

Mã nguồn hiện tại đang sử dụng Spring Boot 3.x và Java 21, đây là nền tảng phù hợp để tiếp tục phát triển. Tuy nhiên, để đồng bộ hóa giữa các service và giảm xung đột phụ thuộc, cần chuẩn hóa lại file `pom.xml` cha của dự án theo hướng:

- Giữ `Java 21` làm chuẩn build chung.
- Sử dụng `spring-cloud-dependencies` để quản lý version đồng bộ cho các thư viện cloud.
- Thay thế các dependency rời rạc bằng các starter chính thức của Spring Boot / Spring Cloud.
- Bổ sung các thư viện cần thiết cho resilience, validation, OpenAPI và JWT theo chuẩn mới.

### Giải pháp đề xuất

- Chuẩn hóa `pom.xml` cha để các module `api-gateway`, `user-service`, `product-service`, `order-service`, `notification-service`, `payment-service`, `eureka-server` dùng chung một nền version.
- Bổ sung `spring-boot-starter-validation`, `resilience4j-spring-boot3`, `springdoc-openapi`, `jjwt-api`, `jjwt-impl`, `jjwt-jackson`.
- Hạn chế khai báo trùng lặp nhiều dependency ở từng service con.

### Code snippet minh họa: Chuẩn hóa `pom.xml` cha

```xml
<properties>
    <java.version>21</java.version>
    <spring-cloud.version>2024.0.1</spring-cloud.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-spring-boot3</artifactId>
    </dependency>

    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.12.6</version>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.12.6</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId>
        <version>0.12.6</version>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

### 4.1.2. Quy chuẩn lại cấu trúc thư mục

Trong mã nguồn gốc, các package `controller`, `service`, `repository`, `config`, `model` đã được tạo tương đối hợp lý. Tuy nhiên, để đạt mức dễ bảo trì cao hơn, mỗi service nên được chuẩn hóa theo cùng một khuôn mẫu để bất kỳ thành viên nào khi đọc code cũng có thể nhanh chóng xác định luồng xử lý.

### Cấu trúc đề xuất cho mỗi service

```text
order-service/
└── src/main/java/vn/tt/practice/orderservice/
    ├── controller/
    ├── service/
    │   ├── OrderCommandService.java
    │   └── OrderQueryService.java
    ├── repository/
    ├── dto/
    │   ├── request/
    │   └── response/
    ├── mapper/
    ├── client/
    ├── event/
    │   ├── payload/
    │   └── producer/
    ├── exception/
    └── config/
```

Cách tổ chức này mang lại các lợi ích sau:

- `controller` chỉ nhận và trả dữ liệu HTTP.
- `service` chỉ xử lý nghiệp vụ.
- `repository` chỉ tương tác với dữ liệu.
- `dto` tách riêng dữ liệu request/response thay vì trả thẳng entity.
- `exception` gom logic xử lý lỗi toàn cục.
- `client` dùng cho Feign/RestTemplate/WebClient gọi sang service khác.
- `event` dùng riêng cho Kafka producer/consumer.

### Code snippet minh họa: `GlobalExceptionHandler`

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<Map<String, Object>> handleBadRequest(IllegalArgumentException ex) {
        return ResponseEntity.badRequest().body(Map.of(
                "timestamp", Instant.now(),
                "message", ex.getMessage()
        ));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<Map<String, Object>> handleInternal(Exception ex) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(Map.of(
                "timestamp", Instant.now(),
                "message", "Internal server error"
        ));
    }
}
```

Việc đưa toàn bộ xử lý lỗi về một nơi giúp controller gọn hơn, response thống nhất hơn và thuận lợi cho frontend khi hiển thị lỗi.

---

## 4.2. Rebuild chi tiết các dịch vụ

### 4.2.1. Tối ưu hóa API Gateway

Trong dự án `ProjectWeb-BE`, `api-gateway` là cổng vào chung của toàn hệ thống. Đây là nơi rất quan trọng vì toàn bộ request từ frontend đều đi qua service này trước khi được điều hướng đến các service nghiệp vụ. Ở phiên bản gốc, gateway đã định nghĩa route cơ bản nhưng vẫn còn các hạn chế như:

- Route cấu hình còn đơn giản, chưa có fallback hoặc circuit breaker.
- Chưa có filter log request/response thống nhất.
- Chưa có cơ chế xác thực token tập trung tại gateway.
- Rate-limit mới áp dụng một phần.

### Giải pháp rebuild

- Chuẩn hóa route theo `application.yml` thay vì khai báo rời rạc khó bảo trì.
- Bổ sung `GlobalFilter` để log method, path, status code và thời gian xử lý.
- Bổ sung `AuthenticationFilter` để kiểm tra `Authorization: Bearer <token>` cho các route cần bảo vệ.
- Kết hợp `Retry`, `CircuitBreaker`, `RequestRateLimiter` cho các service downstream.

### Code snippet minh họa: Route gateway thông minh hơn

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/v1/api/user/**

        - id: product-service
          uri: lb://product-service
          predicates:
            - Path=/v1/api/products/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
                key-resolver: "#{@ipKeyResolver}"
            - name: CircuitBreaker
              args:
                name: productCircuitBreaker
                fallbackUri: forward:/fallback/products
```

### Code snippet minh họa: Filter log tại `api-gateway`

```java
@Component
@Slf4j
public class RequestLoggingFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        long start = System.currentTimeMillis();
        String method = exchange.getRequest().getMethod().name();
        String path = exchange.getRequest().getURI().getPath();

        return chain.filter(exchange)
                .doFinally(signal -> {
                    long duration = System.currentTimeMillis() - start;
                    HttpStatusCode status = exchange.getResponse().getStatusCode();
                    log.info("[Gateway] {} {} -> {} ({} ms)", method, path, status, duration);
                });
    }

    @Override
    public int getOrder() {
        return -1;
    }
}
```

### Code snippet minh họa: Filter xác thực token

```java
@Component
public class AuthenticationFilter {

    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getURI().getPath();

        if (path.startsWith("/v1/api/user/login") || path.startsWith("/v1/api/user/register")) {
            return chain.filter(exchange);
        }

        String authHeader = exchange.getRequest().getHeaders().getFirst(HttpHeaders.AUTHORIZATION);
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        return chain.filter(exchange);
    }
}
```

Nhờ đó, `api-gateway` không chỉ làm nhiệm vụ route đơn thuần mà còn trở thành lớp bảo vệ đầu tiên của hệ thống.

### 4.2.2. Refactor logic `Product Service` và `Order Service`

Hai service `product-service` và `order-service` là phần lõi của nghiệp vụ thương mại điện tử. Vì vậy, đây cũng là nơi cần ưu tiên refactor mạnh nhất theo nguyên tắc **SOLID**.

#### Các vấn đề ở phiên bản gốc

- `ProductController` đang chứa một số logic không nên thuộc về domain sản phẩm, ví dụ `add-to-cart`, `remove-from-cart`.
- `removeFromCart()` hiện có rủi ro xóa luôn sản phẩm khỏi database.
- `OrderService.placeOrder()` đang vừa gọi service khác, vừa kiểm tra dữ liệu, vừa lưu đơn, vừa phát sự kiện — vi phạm nguyên tắc một trách nhiệm.
- Chưa có cơ chế tách command/query rõ ràng.

### Hướng rebuild theo SOLID

- Controller giữ ở mức mỏng, chỉ nhận request và trả response.
- Dùng `OrderCommandService` để xử lý đặt/hủy đơn.
- Dùng `OrderQueryService` để truy vấn danh sách đơn hàng.
- Dùng `InventoryGateway` hoặc `ProductClient` như một abstraction riêng cho việc gọi xuống `product-service`.
- Trong `product-service`, tách logic quản lý sản phẩm khỏi logic giỏ hàng.

### Code snippet minh họa: Controller mỏng hơn

```java
@RestController
@RequestMapping("/v1/api/order")
@RequiredArgsConstructor
public class OrderController {

    private final OrderCommandService orderCommandService;
    private final OrderQueryService orderQueryService;

    @PostMapping("/place-order")
    public ResponseEntity<OrderResponse> placeOrder(@Valid @RequestBody CreateOrderRequest request) {
        return ResponseEntity.ok(orderCommandService.placeOrder(request));
    }

    @GetMapping("/{userId}/get-orders")
    public ResponseEntity<List<OrderResponse>> getOrders(@PathVariable String userId) {
        return ResponseEntity.ok(orderQueryService.getOrdersByUserId(userId));
    }
}
```

### Code snippet minh họa: `OrderCommandService` gọn gàng và dễ bảo trì hơn

```java
@Service
@RequiredArgsConstructor
public class OrderCommandService {

    private final InventoryGateway inventoryGateway;
    private final OrderRepository orderRepository;
    private final OrderMapper orderMapper;
    private final OrderEventPublisher orderEventPublisher;

    public OrderResponse placeOrder(CreateOrderRequest request) {
        validateRequest(request);

        request.items().forEach(item ->
                inventoryGateway.reserve(item.productId(), item.quantity())
        );

        Order order = orderMapper.toEntity(request);
        order.setStatus(OrderStatus.PENDING.name());

        Order savedOrder = orderRepository.save(order);
        orderEventPublisher.publishOrderCreated(savedOrder);

        return orderMapper.toResponse(savedOrder);
    }

    private void validateRequest(CreateOrderRequest request) {
        if (request.userId() == null || request.items().isEmpty()) {
            throw new IllegalArgumentException("Invalid order request");
        }
    }
}
```

Ở phiên bản rebuild, từng bước đã được tách thành các hàm nhỏ hơn, rõ ý nghĩa hơn và dễ viết test hơn.

### Code snippet minh họa: cập nhật tồn kho nguyên tử trong `product-service`

```java
@Service
@RequiredArgsConstructor
public class ProductInventoryService {

    private final MongoTemplate mongoTemplate;

    public void decreaseStock(String productId, int amount) {
        Query query = Query.query(
                Criteria.where("_id").is(productId)
                        .and("quantity").gte(amount)
        );

        Update update = new Update().inc("quantity", -amount);

        Product updated = mongoTemplate.findAndModify(
                query,
                update,
                FindAndModifyOptions.options().returnNew(true),
                Product.class
        );

        if (updated == null) {
            throw new IllegalArgumentException("Product not found or insufficient stock");
        }
    }
}
```

Cách làm này tốt hơn so với mô hình `findById() -> setQuantity() -> save()` vì tránh được lỗi cạnh tranh dữ liệu khi nhiều request cùng mua một sản phẩm tại cùng một thời điểm.

### 4.2.3. Cải tiến `Notification Service`

Về bản chất, `notification-service` đã là một service chạy bất đồng bộ nhờ Kafka. Tuy nhiên, ở mã nguồn gốc, consumer vẫn đang làm quá nhiều việc trong một lớp duy nhất: parse JSON, gọi `user-service`, xử lý lỗi, gửi email. Điều này khiến code khó mở rộng và khó đảm bảo idempotent.

### Giải pháp rebuild

- Tách lớp `KafkaConsumer` ra khỏi lớp xử lý thông báo thực tế.
- Tạo `NotificationProcessor` riêng để xử lý nghiệp vụ gửi email.
- Áp dụng `@Async` + `@Retryable` để service chịu lỗi tốt hơn.
- Thêm cơ chế chống gửi trùng bằng Redis hoặc bảng `processed_events`.

### Code snippet minh họa: tách consumer và processor

```java
@Service
@RequiredArgsConstructor
public class OrderCreatedConsumer {

    private final NotificationProcessor notificationProcessor;

    @KafkaListener(topics = "notificationTopic", groupId = "notification-group")
    public void onMessage(OrderCreatedEvent event) {
        notificationProcessor.process(event);
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class NotificationProcessor {

    private final ProcessedEventStore processedEventStore;
    private final EmailService emailService;

    @Async("notificationExecutor")
    @Retryable(maxAttempts = 3, backoff = @Backoff(delay = 2000))
    public void process(OrderCreatedEvent event) {
        if (processedEventStore.alreadyProcessed(event.eventId())) {
            return;
        }

        emailService.sendOrderPlacedMail(event.email(), event.orderId());
        processedEventStore.markProcessed(event.eventId());
    }
}
```

### Lợi ích đạt được

- Không gửi email trùng khi Kafka phát lại message.
- Logic rõ ràng: consumer chỉ nhận event, processor mới xử lý nghiệp vụ.
- Có thể mở rộng thêm SMS, push notification hoặc webhook mà không làm rối code cũ.

---

## 4.3. Cải thiện Database Interactions (Tối ưu query, sử dụng ORM / Spring Data best practices)

Dự án `ProjectWeb-BE` đang sử dụng MongoDB làm cơ sở dữ liệu chính cho nhiều service. Vì vậy, việc cải thiện tương tác dữ liệu nên tập trung vào các nguyên tắc của **Spring Data MongoDB**, thay vì chỉ nhìn theo tư duy CRUD cơ bản.

### 4.3.1. Tối ưu hóa truy vấn

Ở phiên bản gốc, một số thao tác vẫn còn truy vấn theo kiểu chưa chọn lọc hoặc load nhiều dữ liệu hơn mức cần thiết. Để tối ưu, cần áp dụng các nguyên tắc sau:

- Chỉ lấy đúng dữ liệu cần dùng thay vì trả cả document.
- Dùng phân trang (`Pageable`) cho các danh sách lớn.
- Tạo index cho các trường truy vấn thường xuyên như `email`, `productCode`, `userId`, `status`.
- Dùng query nguyên tử cho cập nhật tồn kho.

### Code snippet minh họa: thêm index cho các document quan trọng

```java
@Document(collection = "users")
public class User {

    @Id
    private String id;

    @Indexed(unique = true)
    private String email;

    private String username;
    private String password;
}
```

```java
@Document(collection = "products")
public class Product {

    @Id
    private String id;

    @Indexed(unique = true)
    private String productCode;

    private String name;
    private int quantity;

    @Version
    private Long version;
}
```

Việc thêm `@Indexed` giúp tăng tốc các truy vấn thường xuyên, còn `@Version` hỗ trợ optimistic locking để hạn chế ghi đè dữ liệu khi có nhiều request đồng thời.

### 4.3.2. Sử dụng Repository theo đúng best practice

Thay vì xử lý quá nhiều logic thủ công ở service, nên tận dụng khả năng sinh query tự động từ `Spring Data Repository`.

### Code snippet minh họa: Repository trả về projection thay vì full entity

```java
public interface OrderSummaryProjection {
    String getId();
    String getStatus();
    BigDecimal getTotalPrice();
    Instant getCreatedAt();
}

public interface OrderRepository extends MongoRepository<Order, String> {
    Page<OrderSummaryProjection> findByUserIdOrderByCreatedAtDesc(String userId, Pageable pageable);
}
```

Cách làm này giúp response gọn hơn, giảm lượng dữ liệu truyền đi và giảm coupling giữa entity trong database với dữ liệu gửi cho frontend.

### 4.3.3. Chuẩn hóa dữ liệu ghi log và audit

Trong hệ thống thương mại điện tử, dữ liệu như thời gian tạo đơn, thời gian cập nhật trạng thái, người thao tác là các thông tin rất quan trọng. Vì vậy, các document nên có thêm trường audit để phục vụ debug và truy vết sau này.

### Code snippet minh họa: Audit fields

```java
@Document(collection = "orders")
public class Order {

    @Id
    private String id;

    private String userId;
    private String status;

    @CreatedDate
    private Instant createdAt;

    @LastModifiedDate
    private Instant updatedAt;
}
```

Khi chuẩn hóa audit field, hệ thống sẽ thuận lợi hơn trong việc:

- sắp xếp đơn hàng theo thời gian,
- thống kê báo cáo,
- truy vết lỗi,
- đồng bộ dữ liệu với các hệ thống khác.

### 4.3.4. Tổng kết hướng cải thiện database

Các cải tiến về dữ liệu trong giai đoạn rebuild nên tập trung vào 4 mục tiêu chính:

1. **Đúng dữ liệu**: tránh cập nhật sai, thiếu hoặc ghi đè không kiểm soát.  
2. **Nhanh hơn**: thêm index, projection, pagination.  
3. **An toàn hơn**: hỗ trợ locking và kiểm tra điều kiện cập nhật.  
4. **Dễ bảo trì hơn**: repository rõ ràng, DTO tách biệt, ít viết query thủ công trùng lặp.

---

## Kết luận chương 4

Việc tái cấu trúc backend theo tiêu chuẩn "Vibe Code" không chỉ là làm cho mã nguồn đẹp hơn, mà quan trọng hơn là giúp hệ thống `ProjectWeb-BE` tiến gần hơn đến một kiến trúc **ổn định, dễ bảo trì, dễ mở rộng và phù hợp triển khai thực tế**. Các đề xuất ở chương này đã bám sát đúng cấu trúc dự án hiện tại, bao gồm `api-gateway`, `product-service`, `order-service`, `notification-service`, `user-service` và các thành phần hạ tầng liên quan.

Nếu được áp dụng đầy đủ, các thay đổi này sẽ giúp hệ thống cải thiện rõ rệt ở các mặt:

- bảo mật tốt hơn,
- code dễ đọc và dễ test hơn,
- luồng nghiệp vụ rõ ràng hơn,
- khả năng chịu lỗi tốt hơn trong môi trường microservices,
- hiệu năng truy vấn và xử lý dữ liệu ổn định hơn.

Đây cũng là tiền đề để tiếp tục triển khai các bước nâng cấp sâu hơn ở các giai đoạn sau như: bổ sung CI/CD hoàn chỉnh, chuẩn hóa logging tập trung, tăng độ bao phủ test và triển khai production-ready trên Kubernetes.
