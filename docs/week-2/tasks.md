# KẾ HOẠCH & PHÂN CHIA NHIỆM VỤ TUẦN 2 (WORKLOG TUẦN 2)

## 📌 THÔNG TIN ĐỒ ÁN
- **Đề tài:** Hệ thống đặt phòng khách sạn với kiến trúc microservices trên ECS (Nhóm: Capstone)
- **Học phần:** Điện toán đám mây (Cloud) - Đợt 1 (Năm học 2026–2027)
- **Giảng viên hướng dẫn (GVHD):** Thầy Huỳnh Xuân Phụng
- **Mục tiêu Tuần 2:** Triển khai các chức năng lõi trên AWS, kiểm thử từng thành phần (Mỗi sinh viên thực hiện $\ge$ 8.0 giờ/tuần).
- **Môi trường thực hành:** AWS Academy Learner Lab, Region `us-east-1`, Account ID: `0556-5295-1832` (Tài khoản nhóm của Phan Thị Thùy Linh).
- **Quy định IAM bắt buộc:** Sử dụng `LabRole` / `LabInstanceProfile`.

---

## 👥 BẢNG PHÂN CÔNG CÔNG VIỆC TUẦN 2 (WORKLOG)

| STT | Thành viên | MSSV | Vai trò chính trong Tuần 2 | Số giờ dự kiến | Trạng thái |
|:---:|---|:---:|---|:---:|:---:|
| 1 | **Cao Hoàng Phúc** | `24110303` | **Cloud Infrastructure & DevOps:** Khởi tạo RDS (2 logical databases), DynamoDB, SNS/SQS/DLQ, ALB Path-based Routing, ECS Cluster & Service. | 9.0 giờ | Hoàn thành |
| 2 | **Phan Thị Thùy Linh** | `24110271` | **Backend & Containerization:** Viết code JPA, REST internal call, SNS Producer, SQS Listener, Docker Build & Push ECR Images (`v1.0.0`). | 9.0 giờ | Hoàn thành |

---

## 🛠️ CHI TIẾT CÔNG VIỆC TỪNG THÀNH VIÊN

### 🔹 Thành viên 1: Cao Hoàng Phúc (MSSV: 24110303)
- **Vai trò:** Phụ trách toàn bộ Hạ tầng Dịch vụ AWS (Data Layer, Messaging Layer, Application Load Balancer, ECS Fargate).
- **Số giờ thực hiện:** 9.0 giờ.
- **Nhiệm vụ chi tiết:**

1. **Khởi tạo Amazon RDS (Relational Database):**
   - Tạo DB Subnet Group: Chọn 2 Private Subnets (`subnet-private-1` và `subnet-private-2`) tại `us-east-1a` và `us-east-1b`.
   - Tạo DB Instance: Loại engine PostgreSQL (hoặc MySQL), phiên bản ổn định, instance class **`db.t3.micro`** (Single-AZ để tiết kiệm ngân sách Lab).
   - Gán Security Group: `sg-hotel-rds` (Chỉ mở cổng `5432` hoặc `3306` nhận inbound từ `sg-hotel-ecs`).
   - Tạo 2 logical databases / schemas độc lập:
     - `room_db` (Tạo user `room_user` có toàn quyền trên `room_db`).
     - `booking_db` (Tạo user `booking_user` có toàn quyền trên `booking_db`).
   - Cung cấp chuỗi kết nối (Endpoint, Port, User, Password) cho Linh cấu hình vào file `application.properties`.

2. **Khởi tạo Amazon DynamoDB, SNS Topic và SQS Queue:**
   - **DynamoDB:** Tạo bảng `hotel-booking-audit-logs`, Partition Key: `bookingId` (String), Sort Key: `timestamp` (String), Billing mode: `On-Demand`.
   - **SNS Topic:** Tạo Topic Standard tên `hotel-booking-events`.
   - **SQS Dead Letter Queue:** Tạo queue `booking-notification-dlq` (Message retention: 14 ngày).
   - **SQS Main Queue:** Tạo queue `booking-notification-queue` (Message retention: 4 ngày), cấu hình Redrive policy trỏ về `booking-notification-dlq` với `maxReceiveCount = 3`.
   - **Subscription:** Đăng ký (Subscribe) `booking-notification-queue` vào topic `hotel-booking-events` (giao thức SQS, bật tính năng Raw Message Delivery nếu cần).

3. **Cấu hình Application Load Balancer (ALB):**
   - Tạo ALB Internet-facing đặt trên 2 Public Subnets (`subnet-public-1`, `subnet-public-2`), gán `sg-hotel-alb`.
   - Tạo 3 Target Groups (Target type: `IP addresses`, Protocol: `HTTP`, Port: `8080`, Health check path: `/actuator/health`):
     - `tg-hotel-room-service`
     - `tg-hotel-booking-service`
     - `tg-hotel-notification-service`
   - Cấu hình Listener Port 80 với các Path-based Routing Rules:
     - Rule 1: Path is `/api/rooms*` ➡️ Forward to `tg-hotel-room-service`
     - Rule 2: Path is `/api/bookings*` ➡️ Forward to `tg-hotel-booking-service`
     - Rule 3: Path is `/api/notifications*` ➡️ Forward to `tg-hotel-notification-service`
     - Default Rule: Fixed response 404 hoặc Health check chung.

4. **Thiết lập ECS Cluster & ECS Services:**
   - Tạo ECS Cluster: `hotel-cluster` (Serverless Fargate).
   - Sau khi Linh push image lên ECR, tạo 3 Task Definitions cho 3 service:
     - Chọn mạng `awsvpc`, hệ điều hành Linux/X86_64.
     - Cấu hình tài nguyên tối thiểu: `0.25 vCPU` (256) và `0.5 GB` RAM (512).
     - Chỉ định Task Execution Role và Task Role: **`LabRole`**.
     - Cấu hình Container: Port mapping `8080`, trỏ Image URI từ ECR repo tương ứng.
     - Thiết lập biến môi trường (Environment variables) kết nối RDS, DynamoDB, SNS ARN, SQS URL.
     - Bật CloudWatch logs qua `awslogs` driver.
   - Khởi chạy 3 ECS Services (Desire count = 1), đặt trong Private Subnets, gán `sg-hotel-ecs`, gắn với 3 Target Groups tương ứng của ALB.

5. **Ghi nhận minh chứng & Tuân thủ quy định:**
   - Chụp ảnh minh chứng Console (có đầy đủ Account ID `0556-5295-1832`, Region `us-east-1` và ngày giờ).
   - Gắn tag định danh: `Project=hotel-booking-microservices`, `Owner=24110303`.
   - **Sau mỗi buổi làm việc:** Scale Task về `0` và Stop RDS.

---

### 🔹 Thành viên 2: Phan Thị Thùy Linh (MSSV: 24110271)
- **Vai trò:** Phụ trách Toàn bộ Mã nguồn Backend, Mô hình dữ liệu JPA, SDK Tích hợp Cloud & Containerization.
- **Số giờ thực hiện:** 9.0 giờ.
- **Nhiệm vụ chi tiết:**

1. **Nâng cấp Dependencies trong `pom.xml` của cả 3 services:**
   - Thêm `spring-boot-starter-data-jpa` và PostgreSQL/MySQL JDBC driver vào `room-service` và `booking-service`.
   - Thêm AWS SDK for Java v2 BOM và dependencies:
     - `software.amazon.awssdk:sns`
     - `software.amazon.awssdk:sqs`
     - `software.amazon.awssdk:dynamodb`

2. **Phát triển mã nguồn `room-service`:**
   - Tạo Entity `Room` theo đúng ERD: `id`, `room_number`, `type`, `price_per_night`, `status`, `description`, `created_at`, `updated_at`.
   - Tạo `RoomRepository` kế thừa `JpaRepository`.
   - Viết các API:
     - `GET /api/rooms`: Trả về danh sách tất cả các phòng.
     - `GET /api/rooms/{id}`: Tra cứu chi tiết một phòng.
     - `PUT /api/rooms/{id}/status`: Cập nhật trạng thái phòng (khóa phòng thành `RESERVED` khi đặt, hoặc giải phóng về `AVAILABLE` khi hủy).
   - Cấu hình `application.properties`: Datasource kết nối tới `room_db`, JPA ddl-auto `update`.

3. **Phát triển mã nguồn `booking-service`:**
   - Tạo Entity `Booking` theo ERD: `id`, `booking_code`, `customer_id`, `customer_name`, `customer_email`, `customer_phone`, `room_id`, `check_in_date`, `check_out_date`, `total_price`, `status`, `created_at`, `updated_at`.
   - Tạo `BookingRepository` kế thừa `JpaRepository`.
   - Viết Service & Controller xử lý đặt phòng:
     - `POST /api/bookings`: Nhận thông tin đặt phòng từ Client.
     - Sử dụng `RestClient` / `WebClient` gọi đồng bộ nội bộ sang `room-service` (`PUT /api/rooms/{id}/status`) để khóa phòng.
     - Nếu khóa phòng thành công: Lưu bản ghi đơn đặt phòng vào `booking_db` với trạng thái `CONFIRMED`.
     - Sử dụng DynamoDB SDK: Ghi nhận vết kiểm toán (Audit Trail) vào bảng `hotel-booking-audit-logs`.
     - Sử dụng SNS SDK: Đẩy sự kiện `BookingCreatedEvent` (định dạng JSON đã đặc tả) vào Amazon SNS Topic.
   - Cấu hình `application.properties`: Datasource trỏ tới `booking_db`, SNS Topic ARN, DynamoDB table name.

4. **Phát triển mã nguồn `notification-service`:**
   - Sử dụng SQS Client: Thiết lập Scheduled task hoặc SQS Listener định kỳ poll tin nhắn từ SQS Queue `booking-notification-queue`.
   - Phân tích payload JSON của `BookingCreatedEvent`.
   - Triển khai cơ chế **Idempotency**: Kiểm tra `eventId` đã xử lý chưa để tránh gửi lặp thông báo.
   - In log ra console (chuẩn bị đẩy sang CloudWatch): Mô phỏng gửi email xác nhận thành công tới địa chỉ `customerEmail`.
   - Xóa message khỏi queue (ACK) sau khi xử lý thành công.

5. **Đóng gói Docker & Đẩy Image lên Amazon ECR:**
   - Đóng gói ứng dụng: `mvn clean package -DskipTests`.
   - Build Docker Image trên máy với tag phiên bản `v1.0.0`:
     - `docker build -t hotel-room-service:v1.0.0 ./room-service`
     - `docker build -t hotel-booking-service:v1.0.0 ./booking-service`
     - `docker build -t hotel-notification-service:v1.0.0 ./notification-service`
   - Đăng nhập Amazon ECR bằng AWS CLI và push cả 3 images lên 3 kho chứa tương ứng trên tài khoản của Linh.
   - Chụp ảnh minh chứng terminal quá trình push và danh sách image trên ECR Console.

---

## 🔄 QUY TRÌNH PHỐI HỢP LIÊN TỤC GIỮA 2 THÀNH VIÊN

```
[BƯỚC 1 - Phúc]: Tạo RDS (2 logical databases), DynamoDB, SNS Topic, SQS Queue & DLQ trên AWS
       │
       ▼ (Bàn giao: Database Host Endpoint, Credentials, SNS ARN, SQS URL)
[BƯỚC 2 - Linh]: Viết code JPA, gọi REST nội bộ, cấu hình AWS SDK producer/consumer
       │
       ▼ (Đóng gói JAR & Docker Build)
[BƯỚC 3 - Linh]: Push 3 Container Images (tag v1.0.0) lên Amazon ECR
       │
       ▼ (Bàn giao: ECR Image URIs hoàn chỉnh)
[BƯỚC 4 - Phúc]: Tạo ECS Task Definitions, khởi chạy Services, tạo ALB & cấu hình Path Routing
       │
       ▼ (Cả 2 cùng kiểm thử)
[BƯỚC 5 - Cả hai]: Kiểm thử từng thành phần: Tra cứu phòng, Đặt phòng, Message fanout qua SQS, Health check
       │
       ▼ (HẾT BUỔI THỰC HÀNH)
[BƯỚC 6 - BẮT BUỘC]: Scale ECS Tasks về 0 và bấm Dừng (Stop) RDS Instance
```

---

## 📋 DANH MỤC MINH CHỨNG NGHIỆM THU TUẦN 2 (CHECKLIST ẢNH)

1. **Minh chứng Hạ tầng (Phúc):**
   - [x] Ảnh 1: Amazon RDS Instance đang chạy (loại `db.t3.micro`, Private Subnet, tag đầy đủ).
   - [x] Ảnh 2: Hai logical database `room_db` và `booking_db` trên cùng RDS PostgreSQL instance.
   - [x] Ảnh 3: Bảng Amazon DynamoDB `hotel-booking-audit-logs` (Partition Key `bookingId`, Sort Key `timestamp`).
   - [x] Ảnh 4: Amazon SNS Topic `hotel-booking-events` và Subscription liên kết tới SQS.
   - [x] Ảnh 5: Amazon SQS Main Queue (`booking-notification-queue`) và Dead Letter Queue (`booking-notification-dlq`).
   - [x] Ảnh 6: Application Load Balancer (ALB) và các Path-based Routing Rules (`/api/rooms*`, `/api/bookings*`, `/api/notifications*`).
   - [x] Ảnh 7: Danh sách 3 Target Groups của ALB đều báo trạng thái **Healthy**.
   - [x] Ảnh 8: ECS Cluster `hotel-cluster` với 3 Task Definitions và 3 Services chạy trên Fargate.

2. **Minh chứng Ứng dụng & Kiểm thử (Linh):**
   - [x] Ảnh 9: Terminal đóng gói Maven và Docker build image `v1.0.0` thành công.
   - [x] Ảnh 10: Terminal thực hiện lệnh `docker push` đẩy image lên Amazon ECR.
   - [x] Ảnh 11: Giao diện Amazon ECR Repositories hiển thị image tag `v1.0.0`.
   - [x] Ảnh 12: Kiểm thử gọi API `GET /api/rooms` qua DNS của ALB trên Postman/Browser (trả về JSON danh sách phòng).
   - [x] Ảnh 13: Kiểm thử gọi API `POST /api/bookings` tạo đơn thành công (mã `201 Created`).
   - [x] Ảnh 14: Dữ liệu được ghi thành công vào RDS bảng `rooms` và `bookings`.
   - [x] Ảnh 15: Dữ liệu Audit Trail được ghi vào DynamoDB.
   - [x] Ảnh 16: Log của `notification-service` tiêu thụ message từ SQS và in ra log xác nhận gửi email.

---

## 🚀 KẾ HOẠCH DỰ KIẾN CHO TUẦN 3

Dựa trên kết quả triển khai hạ tầng và backend cốt lõi ở Tuần 2, nhóm đề ra 5 mục tiêu trọng tâm cho Tuần 3:

1. **Hoàn tất bàn giao thông tin kết nối qua AWS Secrets Manager:**
   - Đưa toàn bộ chuỗi kết nối database (RDS Endpoint, credentials của `room_user` và `booking_user`), SNS Topic ARN, SQS Queue URL vào AWS Secrets Manager.
   - Cấu hình ECS Task Definitions tự động nạp biến môi trường từ Secrets Manager (`secrets` block) thay vì truyền plaintext qua `environment`.

2. **Triển khai giải pháp Service Discovery / Cloud Map trong giới hạn Learner Lab:**
   - Đánh giá khả năng áp dụng AWS Cloud Map kết hợp ECS Service Connect hoặc hoàn thiện giải pháp Route 53 Private Hosted Zone (`hotel.local`) tự động cập nhật bản ghi khi container tái khởi động hoặc đổi IP nội bộ.

3. **Cấu hình chính sách tự động mở rộng (Auto Scaling Policy):**
   - Thiết lập Target Tracking Scaling Policy cho 3 ECS Services dựa trên CPU Utilization (> 70%) và Request Count per Target của ALB.
   - Thử nghiệm scale out và scale in để đảm bảo tính sẵn sàng cao và tối ưu chi phí.

4. **Bổ sung kiểm thử tự động cho các luồng xử lý lỗi và ngoại lệ:**
   - Viết kịch bản kiểm thử lỗi timeout kết nối nội bộ giữa `booking-service` và `room-service` (mô phỏng Circuit Breaker / Retry).
   - Kiểm thử tự động cơ chế đẩy message lỗi vào SQS Dead Letter Queue (DLQ) khi dữ liệu sự kiện bị sai định dạng.

5. **Chuẩn bị và hoàn thiện nội dung báo cáo tổng kết đồ án:**
   - Tổng hợp số liệu đo lường hiệu năng, chi phí vận hành trên Learner Lab.
   - Soạn thảo tài liệu kiến trúc, hướng dẫn triển khai (Deployment Guide) và slide/video demo hoàn chỉnh cho buổi báo cáo tổng kết.
