# Báo Cáo Công Việc Tuần 2 — Phan Thị Thùy Linh

**Đề tài:** Hệ thống đặt phòng khách sạn với kiến trúc microservices trên ECS  
**Sinh viên:** Phan Thị Thùy Linh — MSSV `24110271`  
**Học phần:** Điện toán đám mây (Cloud) — Đợt 1 năm học 2026–2027  
**Giảng viên hướng dẫn:** Thầy Huỳnh Xuân Phụng  
**Môi trường:** Docker Desktop trên Windows & AWS Academy Learner Lab, Region `us-east-1` (Account ID: `0556-5295-1832`)  
**Repository:** https://github.com/pttlinh32/hotel-booking-microservices  
**Phân công Tuần 2:** Lập trình Backend lõi (Spring Data JPA, REST call nội bộ, AWS SDK SNS/SQS/DynamoDB), Đóng gói Docker & Đẩy image ECR (`v1.0.0`), Kiểm thử thành phần và phối hợp tích hợp ECS/ALB.  
**Số giờ thực hiện:** 9.0 giờ theo worklog  
**Trạng thái:** Hoàn thành  
**Ngày thực hiện:** 14–17/10/2026  

---

### Bảng phân công công việc tuần 2 (Worklog)

| STT | Thành viên | MSSV | Vai trò chính trong Tuần 2 | Số giờ | Trạng thái |
|:---:|---|:---:|---|:---:|:---:|
| 1 | **Phan Thị Thùy Linh** | `24110271` | **Backend & Containerization:** Nâng cấp dependencies JPA/AWS SDK v2; lập trình mã nguồn 3 microservices (`room`, `booking`, `notification`); cơ chế gọi REST nội bộ, bù trừ transaction, Idempotency; đóng gói Docker & đẩy ECR `v1.0.0`; phối hợp kiểm thử E2E. | 9.0 giờ | Hoàn thành |
| 2 | **Cao Hoàng Phúc** | `24110303` | **Cloud Infrastructure & DevOps:** Khởi tạo RDS PostgreSQL (2 logical DBs), DynamoDB Audit Table, SNS/SQS/DLQ, cấu hình ALB Path-based Routing, dựng ECS Fargate Cluster, Task Definitions & Services. | 9.0 giờ | Hoàn thành |

Từ tuần 2, nhóm thống nhất triển khai trực tiếp trên tài khoản AWS Academy Learner Lab của Phan Thị Thùy Linh (`055652951832`) theo sự đồng ý của GVHD Huỳnh Xuân Phụng.

---

## 1. Nâng cấp Dependencies & Cấu hình ứng dụng

Để chuyển đổi từ khung dự án (skeleton) ở Tuần 1 sang hệ thống microservices thực sự có khả năng xử lý nghiệp vụ, lưu trữ dữ liệu quan hệ, ghi nhận audit log và truyền thông điệp bất đồng bộ, em đã tiến hành nâng cấp toàn bộ file cấu hình Maven `pom.xml` cho cả 3 services:

### 1.1. Bổ sung các thư viện phụ thuộc (Dependencies)
1. **Spring Data JPA & PostgreSQL Driver:**
   - Tích hợp `spring-boot-starter-data-jpa` vào `room-service` và `booking-service` để tự động ánh xạ quan hệ đối tượng (ORM), quản lý giao dịch (@Transactional) và tương tác với cơ sở dữ liệu Amazon RDS PostgreSQL thông qua thư viện `org.postgresql:postgresql`.
2. **AWS SDK for Java v2:**
   - Khai báo BOM quản lý phiên bản tập trung `software.amazon.awssdk:bom` trong `<dependencyManagement>` để đồng bộ phiên bản SDK.
   - Bổ sung các client chuyên biệt:
     - `software.amazon.awssdk:dynamodb`: Cho `booking-service` ghi nhận nhật ký kiểm toán (Audit Trail).
     - `software.amazon.awssdk:sns`: Cho `booking-service` phát hành sự kiện đặt phòng `BookingCreatedEvent`.
     - `software.amazon.awssdk:sqs`: Cho `notification-service` lắng nghe và tiếp nhận thông điệp từ hàng đợi.
3. **Jackson Datatype JSR310:**
   - Bổ sung `com.fasterxml.jackson.datatype:jackson-datatype-jsr310` để ObjectMapper tuần tự hóa (serialize) và giải mã (deserialize) chính xác các trường kiểu thời gian của Java 8 (`LocalDate`, `Instant`) trong sự kiện JSON.
4. **Spring Boot Starter Actuator:**
   - Cung cấp probe giám sát sức khỏe `/actuator/health` phục vụ kiểm tra trạng thái tự động (Health Check) của AWS ALB Target Groups và ECS Fargate.

### 1.2. Thiết kế cấu hình linh hoạt qua biến môi trường (`application.properties`)
Mọi thông số kết nối tới RDS, DynamoDB, SNS, SQS và endpoint nội bộ đều được thiết kế đọc từ biến môi trường (Environment Variables) kèm giá trị mặc định hợp lý, giúp ứng dụng có thể chạy kiểm thử tại máy cục bộ hoặc triển khai trong ECS container mà không phải biên dịch lại mã nguồn:

- **`room-service`:**
  ```properties
  spring.datasource.url=${SPRING_DATASOURCE_URL:jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:room_db}}
  spring.datasource.username=${SPRING_DATASOURCE_USERNAME:${DB_USERNAME:room_user}}
  spring.datasource.password=${SPRING_DATASOURCE_PASSWORD:${DB_PASSWORD:room_password}}
  spring.jpa.hibernate.ddl-auto=update
  ```
- **`booking-service`:**
  ```properties
  spring.datasource.url=${SPRING_DATASOURCE_URL:jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:booking_db}}
  spring.datasource.username=${SPRING_DATASOURCE_USERNAME:${DB_USERNAME:booking_user}}
  spring.datasource.password=${SPRING_DATASOURCE_PASSWORD:${DB_PASSWORD:booking_password}}
  room.service.url=${ROOM_SERVICE_URL:http://room-service.hotel.local:8080}
  aws.region=${AWS_REGION:us-east-1}
  aws.sns.topic.booking=${AWS_SNS_TOPIC_ARN:arn:aws:sns:us-east-1:055652951832:hotel-booking-events}
  aws.dynamodb.table.audit=${DYNAMODB_AUDIT_TABLE:hotel-booking-audit-logs}
  ```
- **`notification-service`:**
  ```properties
  aws.region=${AWS_REGION:us-east-1}
  aws.sqs.queue.url=${AWS_SQS_QUEUE_URL:https://sqs.us-east-1.amazonaws.com/055652951832/booking-notification-queue}
  notification.polling.enabled=${NOTIFICATION_POLLING_ENABLED:true}
  ```

---

## 2. Phát triển mã nguồn lõi cho 3 Microservices

### 2.1. `room-service` (Quản lý trạng thái và danh mục phòng)
Dịch vụ đảm nhiệm việc quản lý phòng khách sạn, cung cấp API tra cứu danh mục và quản lý trạng thái khóa/giải phóng phòng.

- **Entity `Room`:** Ánh xạ vào bảng `rooms` trong cơ sở dữ liệu `room_db`, gồm các thuộc tính:
  - `id` (Long, Primary Key tự tăng).
  - `roomNumber` (String, Unique): Mã số phòng (ví dụ: `A-101`, `B-201`).
  - `type` (String): Hạng phòng (`SINGLE`, `DOUBLE`, `DELUXE`, `SUITE`).
  - `pricePerNight` (BigDecimal): Giá phòng một đêm (USD).
  - `status` (String): Trạng thái phòng (`AVAILABLE`, `RESERVED`, `OCCUPIED`, `MAINTENANCE`).
  - `description`, `createdAt`, `updatedAt`.
- **Repository `RoomRepository`:** Kế thừa `JpaRepository<Room, Long>`, bổ sung phương thức `findByStatusIgnoreCase(String status)` để hỗ trợ lọc phòng trống theo trạng thái.
- **Xử lý nghiệp vụ tại `RoomService` & `RoomController`:**
  - `GET /api/rooms`: Trả về danh sách toàn bộ phòng hoặc lọc theo tham số `?status=AVAILABLE`.
  - `GET /api/rooms/{id}`: Tra cứu chi tiết một phòng; nếu không tìm thấy sẽ ném ngoại lệ `RoomNotFoundException` (được `GlobalExceptionHandler` ánh xạ thành mã lỗi `404 Not Found`).
  - `PUT /api/rooms/{id}/status`: Cập nhật trạng thái phòng. Em đã thiết lập cơ chế kiểm tra tranh chấp (Concurrency/Conflict Check): Khi nhận yêu cầu đổi trạng thái sang `RESERVED`, nếu phòng **không ở trạng thái `AVAILABLE`** (ví dụ đã bị đặt hoặc đang bảo trì), service lập tức ném `RoomConflictException` và trả về mã HTTP **`409 Conflict`**. Điều này ngăn chặn triệt để tình trạng đặt trùng phòng (Overbooking).
- **Bộ khởi tạo dữ liệu mẫu (`DataInitializer`):** Tự động khởi tạo 4 phòng mẫu (`A-101` Deluxe $120, `A-102` Single $60, `B-201` Double $90, `B-202` Suite $250) ngay khi ứng dụng khởi động lần đầu nếu bảng dữ liệu đang rỗng, giúp thuận tiện cho quá trình kiểm thử tự động.

---

### 2.2. `booking-service` (Điều phối đặt phòng & Quản lý giao dịch)
Dịch vụ trung tâm của hệ thống, đóng vai trò điều phối quy trình đặt phòng phức hợp kết hợp cả giao tiếp đồng bộ REST và bất đồng bộ sự kiện.

- **Entity `Booking`:** Ánh xạ vào bảng `bookings` trong cơ sở dữ liệu `booking_db`, gồm các trường:
  - `id` (String UUID): Định danh duy nhất của đơn đặt.
  - `bookingCode` (String Unique): Mã đơn thân thiện cho người dùng (ví dụ: `BK-123456`).
  - `customerId`, `customerName`, `customerEmail`, `customerPhone`.
  - `roomId` (Long): Mã phòng tham chiếu sang `room-service`.
  - `checkInDate`, `checkOutDate`, `totalPrice`, `status` (`CONFIRMED`, `CANCELLED`).
  - `createdAt`, `updatedAt`.
- **Giao tiếp REST nội bộ qua `RoomClient`:**
  - Sử dụng đối tượng `RestClient` hiện đại của Spring Boot với cấu hình `SimpleClientHttpRequestFactory` thiết lập chặt chẽ: **Connect Timeout = 3.000 ms** và **Read Timeout = 5.000 ms**, ngăn chặn tình trạng một service treo làm cạn kiệt luồng kết nối của hệ thống.
  - Base URL trỏ tới `http://room-service.hotel.local:8080` (sử dụng Private Hosted Zone của Route 53 trong VPC).
  - Phương thức `reserveRoom(roomId, bookingCode)`: Gửi yêu cầu `PUT /api/rooms/{id}/status` với `targetStatus=RESERVED`. Bắt chính xác mã lỗi `409 Conflict` từ `room-service` để chuyển thành `RoomNotAvailableException`.
  - **Cơ chế Giao dịch bù trừ (Compensating Transaction):** Phương thức `releaseRoom(roomId, bookingCode)` gửi yêu cầu đưa trạng thái phòng trở lại `AVAILABLE`. Trong khối lệnh `try...catch` của `BookingService`, nếu sau khi khóa phòng thành công mà các bước tiếp theo (ghi DB, DynamoDB, SNS) gặp sự cố nghiêm trọng, service sẽ tự động kích hoạt `releaseRoom` để hoàn trả phòng, tránh tình trạng phòng bị treo vĩnh viễn ở trạng thái `RESERVED`.
- **Ghi nhật ký kiểm toán qua `AuditLogService` (Amazon DynamoDB):**
  - Tích hợp `DynamoDbClient` để lưu lại vết trạng thái (Audit Trail) tốc độ cao vào bảng NoSQL `hotel-booking-audit-logs`.
  - Khóa chính Partition Key `bookingId` và Sort Key `timestamp`. Ghi nhận chi tiết: `action: BOOKING_CREATED`, `previousStatus: PENDING`, `newStatus: CONFIRMED`, `actor: booking-service`, cùng thông tin metadata (giá tiền, email, mã phòng).
- **Phát hành sự kiện qua `EventPublisherService` (Amazon SNS):**
  - Chuẩn hóa payload sự kiện theo chuẩn CloudEvents envelope với class `BookingCreatedEvent`:
    - Header: `eventId` (UUID), `eventType: BOOKING_CREATED`, `timestamp`.
    - Payload `data`: Đầy đủ thông tin chi tiết về đơn đặt, khách hàng, số đêm nghỉ và giá tiền.
  - Sử dụng `SnsClient.publish()` để đẩy sự kiện vào topic `hotel-booking-events`, từ đó SNS sẽ tự động fanout phân phối sang hàng đợi SQS của dịch vụ thông báo.
- **Quy trình đặt phòng hoàn chỉnh tại `createBooking`:**
  1. Kiểm tra tính hợp lệ của ngày nhận/trả phòng (`checkOutDate > checkInDate`).
  2. Tra cứu thông tin phòng từ `room-service` qua REST nội bộ.
  3. Tính toán tổng tiền theo đơn giá phòng và số đêm lưu trú.
  4. Khóa phòng tạm thời qua `roomClient.reserveRoom()`.
  5. Lưu bản ghi Booking vào PostgreSQL `booking_db` với trạng thái `CONFIRMED`.
  6. Ghi Audit Log vào Amazon DynamoDB.
  7. Phát hành sự kiện `BookingCreatedEvent` lên Amazon SNS.
  8. Trả về `BookingResponse` với mã HTTP **`201 Created`**.

---

### 2.3. `notification-service` (Xử lý thông điệp bất đồng bộ & Đảm bảo Idempotency)
Dịch vụ xử lý nền, không trực tiếp nhận lưu lượng ghi từ người dùng mà tiêu thụ thông điệp từ hàng đợi SQS để gửi email xác nhận.

- **Tiêu thụ thông điệp định kỳ (`NotificationConsumerService`):**
  - Sử dụng cơ chế `@Scheduled(fixedDelay = 5000)` kết hợp short long-polling (`waitTimeSeconds = 2`) để nhận tối đa 10 tin nhắn mỗi chu kỳ từ `booking-notification-queue`.
  - Phân tích linh hoạt cấu trúc JSON qua Jackson `ObjectMapper`: Tự động nhận diện được cả hai trường hợp (SNS bọc message trong Envelope thông thường hoặc SQS nhận trực tiếp khi bật Raw Message Delivery).
- **Cơ chế chống trùng lặp thông báo (Idempotency Mechanism):**
  - Do cơ chế phân phối của AWS SQS Standard là **At-least-once Delivery** (có thể xuất hiện tin nhắn lặp lại do trễ mạng hoặc retry), em đã thiết kế bộ nhớ deduplication sử dụng cấu trúc luồng an toàn:
    ```java
    private final Set<String> processedEventIds = ConcurrentHashMap.newKeySet();
    ```
  - Khi nhận message, service trích xuất `eventId`:
    - Nếu `eventId` đã tồn tại trong `processedEventIds`: Ghi log cảnh báo `⚠️ [IDEMPOTENCY] Phát hiện sự kiện trùng lặp!`, bỏ qua việc gửi email và gọi lệnh xóa message khỏi queue ngay lập tức.
    - Nếu là sự kiện mới: Đưa `eventId` vào tập hợp và tiếp tục tiến trình xử lý gửi email.
- **Mô phỏng gửi email & Xác nhận hoàn tất (ACK):**
  - Hàm `simulateSendEmail(event)` định dạng nội dung thư xác nhận chuyên nghiệp, ghi chi tiết vào log container (sẽ được CloudWatch thu thập).
  - Sau khi gửi email thành công, thực hiện lệnh `sqsClient.deleteMessage()` (ACK - Acknowledgment) để xóa vĩnh viễn tin nhắn khỏi queue chính.
- **Cơ chế xử lý tin nhắn lỗi chuyển Dead Letter Queue (DLQ):**
  - Khi payload tin nhắn bị lỗi cú pháp JSON hoặc thiếu trường dữ liệu bắt buộc (`eventId == null`), service chủ động **ném ngoại lệ (`throw new IllegalArgumentException`)** và **không xóa message**.
  - Nhờ vậy, SQS sẽ tăng giá trị `ReceiveCount`. Khi vượt quá ngưỡng cấu hình `maxReceiveCount = 3`, AWS SQS sẽ tự động redrive tin nhắn lỗi sang `booking-notification-dlq` để bảo toàn dữ liệu phục vụ điều tra sự cố.

---

## 3. Đóng gói Container & Đẩy Image lên Amazon ECR (`v1.0.0`)

Tuân thủ kế hoạch phân công, sau khi hoàn tất kiểm tra mã nguồn, em thực hiện quy trình đóng gói và xuất bản Docker Image lên tài khoản cá nhân của nhóm (`055652951832`):

### 3.1. Biên dịch và đóng gói JAR
Sử dụng Maven Wrapper để đóng gói ứng dụng độc lập, bỏ qua unit test tạm thời trong pha dựng container:
```powershell
cd booking-service
.\mvnw.cmd package -DskipTests
```

![Terminal đóng gói JAR thành công cho booking-service với Maven Wrapper](images/01-maven-build-booking-service.png)

### 3.2. Build Docker Image với tag phiên bản `v1.0.0`
Áp dụng Dockerfile đa tầng (Multi-stage build) đã thiết kế ở tuần 1 để tạo 3 image nhẹ, chỉ chứa runtime JRE 17:
```powershell
docker build -t hotel-room-service:v1.0.0 ./room-service
docker build -t hotel-booking-service:v1.0.0 ./booking-service
docker build -t hotel-notification-service:v1.0.0 ./notification-service
```

### 3.3. Đăng nhập và đẩy image lên Amazon ECR
Đăng nhập phiên làm việc Docker vào kho ECR của tài khoản `055652951832` tại Region `us-east-1`:
```powershell
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 055652951832.dkr.ecr.us-east-1.amazonaws.com

# Gắn tag URI ECR
docker tag hotel-room-service:v1.0.0 055652951832.dkr.ecr.us-east-1.amazonaws.com/hotel-room-service:v1.0.0
docker tag hotel-booking-service:v1.0.0 055652951832.dkr.ecr.us-east-1.amazonaws.com/hotel-booking-service:v1.0.0
docker tag hotel-notification-service:v1.0.0 055652951832.dkr.ecr.us-east-1.amazonaws.com/hotel-notification-service:v1.0.0

# Đẩy image lên các repository
docker push 055652951832.dkr.ecr.us-east-1.amazonaws.com/hotel-room-service:v1.0.0
docker push 055652951832.dkr.ecr.us-east-1.amazonaws.com/hotel-booking-service:v1.0.0
docker push 055652951832.dkr.ecr.us-east-1.amazonaws.com/hotel-notification-service:v1.0.0
```

Các image phiên bản `v1.0.0` đã được đẩy lên đầy đủ và hiển thị trên Amazon ECR Console:

![Image booking-service phiên bản v1.0.0 trên ECR](images/090-ecr-booking-v100-final.jpg)

---

## 4. Kết quả kiểm thử thành phần & Minh chứng thực nghiệm (E2E Testing)

Sau khi Phúc hoàn thành việc dựng hạ tầng ECS Fargate và ALB Path-based routing, cả hai thành viên đã tiến hành kiểm thử tích hợp đầu cuối (End-to-End) trên toàn bộ hệ sinh thái dịch vụ tại tài khoản nhóm:

### 4.1. Bảng tổng hợp kịch bản kiểm thử

| STT | Kịch bản kiểm thử | Hành vi hệ thống | Mã HTTP / Kết quả | Đánh giá |
|:---:|---|---|:---:|:---:|
| 1 | **Tra cứu phòng** (`GET /api/rooms`) | ALB chuyển tiếp về `hotel-room-tg`, trả về danh sách 4 phòng mẫu. | `200 OK` | Đạt |
| 2 | **Tạo đơn đặt phòng mới** (`POST /api/bookings`) | Gọi `room-service` khóa phòng `RESERVED`, lưu đơn `booking_db`, ghi DynamoDB, bắn SNS. | `201 Created` | Đạt |
| 3 | **Trạng thái phòng sau khi đặt** | Tra cứu phòng ID vừa đặt, phòng đã chuyển từ `AVAILABLE` sang `RESERVED`. | `200 OK` | Đạt |
| 4 | **Đặt lại phòng đã bị khóa** | Yêu cầu đặt lại chính phòng đó, hệ thống chặn overbooking. | `409 Conflict` | Đạt |
| 5 | **Tính toàn vẹn dữ liệu RDS** | Dữ liệu được ghi nhận độc lập vào 2 bảng `rooms` và `bookings`. | Phân quyền cô lập | Đạt |
| 6 | **Kiểm toán trên DynamoDB** | Bảng `hotel-booking-audit-logs` ghi nhận 2 bản ghi `BOOKING_CREATED`. | Item đầy đủ thuộc tính | Đạt |
| 7 | **Xử lý sự kiện bất đồng bộ** | SNS ➡️ SQS ➡️ Notification Consumer đọc message, in email mô phỏng và gửi ACK xóa message. | Message tiêu thụ sạch | Đạt |
| 8 | **Kiểm thử chống lặp (Idempotency)** | Gửi lại sự kiện mang cùng `eventId`, Consumer phát hiện và bỏ qua lần 2. | Không gửi lặp email | Đạt |
| 9 | **Kiểm thử Dead Letter Queue (DLQ)** | Gửi bản tin sai định dạng, sau 3 lần retry bị chuyển sang DLQ. | Message lưu trong DLQ | Đạt |

---

### 4.2. Hình ảnh minh chứng kiểm thử chi tiết

#### a. Kiểm thử API Đặt phòng thành công (`201 Created`) và Phòng vệ đặt trùng (`409 Conflict`)
Kiểm thử trực tiếp qua DNS của Application Load Balancer:
- Request 1: Đặt phòng ID 1 thành công, mã đặt phòng được tạo, trạng thái `CONFIRMED`.
- Request 2: Cố gắng đặt lại phòng ID 1, service lập tức phản hồi mã lỗi `409 Conflict` kèm thông báo chi tiết phòng đã được đặt.

![Kiểm thử tạo booking 201 và đặt trùng 409](images/088-api-post201-conflict409.jpg)

#### b. Kiểm tra dữ liệu lưu trữ quan hệ trong Amazon RDS PostgreSQL
Dữ liệu đơn đặt phòng được lưu thành công vào bảng `bookings` thuộc cơ sở dữ liệu `booking_db`, đồng thời bảng `rooms` thuộc `room_db` cập nhật trạng thái phòng tương ứng sang `RESERVED`:

![Dữ liệu trong hai bảng rooms và bookings](images/089-rds-rooms-bookings-du-lieu.jpg)

#### c. Kiểm tra Audit Log trên Amazon DynamoDB
Bảng `hotel-booking-audit-logs` ghi nhận chính xác 2 mục nhật ký kiểm toán cho các giao dịch đặt phòng, lưu trữ chi tiết thời gian và email khách hàng:

![Audit log được ghi vào DynamoDB](images/075-dynamodb-audit-bookings.jpg)

#### d. Kiểm tra log xử lý Email và xác nhận (ACK) trên AWS CloudWatch
Container `notification-service` tiêu thụ thành công message từ SQS, in nội dung email mô phỏng cho khách hàng và thực hiện lệnh ACK xóa message khỏi hàng đợi:

![CloudWatch log gửi email mô phỏng và ACK message](images/071-cloudwatch-email-ack-idempotency.jpg)

#### e. Kiểm thử cơ chế Idempotency chống trùng lặp
Khi nhận bản tin có `eventId` đã từng xử lý, hệ thống kích hoạt cơ chế phòng vệ, ghi nhận log và xóa message mà không gây lặp email:

![Log bỏ qua sự kiện trùng](images/083-idempotency-final.jpg)

#### f. Kiểm thử xử lý lỗi và chuyển tiếp sang Dead Letter Queue (DLQ)
Bản tin lỗi được SQS tự động redrive sang hàng đợi `booking-notification-dlq` sau 3 lần xử lý thất bại:

![Message lỗi đã được chuyển vào DLQ](images/082-dlq-message-body.jpg)

#### g. Xác nhận trạng thái Healthy của 3 Target Groups trên ALB
Cả 3 microservices đều phản hồi tốt probe `/actuator/health` và đạt trạng thái **Healthy** trên Application Load Balancer:

| Room Service Target Group | Booking Service Target Group | Notification Service Target Group |
|:---:|:---:|:---:|
| ![Target của room-service Healthy](images/076-room-healthy-final.jpg) | ![Target của booking-service Healthy](images/077-booking-healthy-final.jpg) | ![Target của notification-service Healthy](images/069-notification-target-healthy.jpg) |

---

## 5. Sơ đồ Kiến trúc & Luồng Truyền thông Điệp Tuần 2

Hệ thống triển khai Tuần 2 phản ánh đầy đủ mô hình kết hợp giữa kiến trúc phân tán đồng bộ (Synchronous REST nội bộ) và kiến trúc hướng sự kiện bất đồng bộ (Asynchronous Event-driven):

![Sơ đồ kiến trúc hệ thống hotel booking microservices](images/so-do-kien-truc-week-2.png)

---

## 6. Phối hợp làm việc & Xử lý sự cố kỹ thuật phát sinh

Trong quá trình tích hợp mã nguồn với hạ tầng AWS của Phúc, nhóm đã phát hiện và xử lý kịp thời một số vấn đề kỹ thuật quan trọng:

1. **Lỗi tuần tự hóa ngày tháng Java 8 (Jackson JSR-310):**
   - *Hiện tượng:* Khi phát hành sự kiện `BookingCreatedEvent` có chứa trường `LocalDate` qua SNS, Jackson mặc định không hỗ trợ serialize dẫn đến lỗi runtime 500.
   - *Khắc phục:* Em đã phối hợp cùng Phúc bổ sung dependency `jackson-datatype-jsr310` và cấu hình bean `ObjectMapper` tùy biến đăng ký `JavaTimeModule()` với cấu hình `disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)` tại [`AwsConfig.java`](file:///d:/Năm%203/Cloud/project/hotel-booking-microservices/booking-service/src/main/java/com/hotel/booking/config/AwsConfig.java) của cả 2 services `booking` và `notification`.
2. **Kích hoạt cơ chế Dead Letter Queue (DLQ) trên SQS:**
   - *Hiện tượng:* Ban đầu hàm consumer trong `notification-service` bắt toàn bộ ngoại lệ và vẫn gọi `deleteMessage()`, khiến tin nhắn lỗi bị xóa ngay thay vì chuyển vào DLQ.
   - *Khắc phục:* Đã cập nhật logic để ném `IllegalArgumentException` khi payload không hợp lệ, giữ message trong queue để SQS retry đủ 3 lần (`maxReceiveCount = 3`) rồi tự động đưa vào DLQ.
3. **Giải pháp Service Discovery trong môi trường Learner Lab:**
   - *Hiện tượng:* Dịch vụ AWS Cloud Map bị hạn chế quyền tạo namespace tự động trong tài khoản AWS Academy Learner Lab.
   - *Khắc phục:* Nhóm linh hoạt chuyển sang sử dụng Route 53 Private Hosted Zone `hotel.local` liên kết với VPC để phân giải tên miền nội bộ `room-service.hotel.local` sang địa chỉ IP nội bộ của container.
4. **Bảo toàn ngân sách Lab sau thực hành:**
   - Sau khi hoàn tất toàn bộ minh chứng thực nghiệm, nhóm đã đưa cả 3 ECS service về `desired-count = 0` và tạm dừng (`Stop`) instance Amazon RDS để bảo toàn ngân sách 100 USD của tài khoản.

---

## 7. Ghi nhận sử dụng AI

Tuân thủ nghiêm ngặt quy định học phần về việc sử dụng trí tuệ nhân tạo (AI):
- **Phạm vi sử dụng AI:**
  - Tra cứu cú pháp thiết lập RestClient với cấu hình timeout `SimpleClientHttpRequestFactory` trong Spring Boot 3/4.
  - Tham khảo mẫu cấu hình AWS SDK for Java v2 (`DynamoDbClient`, `SnsClient`, `SqsClient`) tương thích với IAM `LabRole` trên AWS Academy Learner Lab.
  - Gợi ý cấu hình Bean Jackson `ObjectMapper` hỗ trợ JSR-310 JavaTimeModule.
  - Hỗ trợ rà soát cấu trúc báo cáo và kiểm tra tính nhất quán với bản đặc tả kiến trúc.
- **Tính độc lập của sinh viên:**
  - Toàn bộ mã nguồn nghiệp vụ, mô hình dữ liệu JPA, cấu trúc sự kiện JSON, quy trình xử lý lỗi và kịch bản kiểm thử thực tế trên Postman/ALB đều do sinh viên tự tay lập trình, biên dịch và kiểm chứng thực nghiệm.

---

## 8. Kế hoạch Tuần 3

1. **Hoàn tất bàn giao thông tin kết nối qua AWS Secrets Manager:**
   - Cập nhật mã nguồn và cấu hình Spring Boot để đọc chuỗi kết nối Database, SNS Topic ARN, SQS Queue URL an toàn từ biến môi trường được inject từ Secrets Manager.
2. **Triển khai Cloud Map hoặc giải pháp Service Discovery thay thế:**
   - Hoàn thiện tích hợp gọi REST nội bộ với service discovery, tự động phân giải endpoint khi task container được scale hoặc khởi động lại trong Learner Lab.
3. **Cấu hình Auto Scaling Policy cho 3 microservices:**
   - Thiết lập các ngưỡng Target Tracking Scaling trên ECS Service để thích ứng với lưu lượng tải biến động.
4. **Bổ sung kiểm thử tự động các kịch bản ngoại lệ:**
   - Bổ sung integration test mô phỏng timeout kết nối, ngắt kết nối tạm thời giữa các microservices và kiểm thử tự động đẩy thông điệp lỗi vào DLQ.
5. **Chuẩn bị nội dung báo cáo tổng kết đồ án:**
   - Tổng hợp kết quả kiểm thử toàn diện, hoàn thiện tài liệu hướng dẫn kỹ thuật và video demo hoàn chỉnh cho buổi nghiệm thu đồ án.

