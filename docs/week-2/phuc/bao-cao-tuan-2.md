# Báo cáo tuần 2 — Cao Hoàng Phúc

**Đề tài:** Hệ thống đặt phòng khách sạn với kiến trúc microservices trên ECS\
**Sinh viên:** Cao Hoàng Phúc — MSSV `24110303`\
**Phân công:** RDS, DynamoDB, SNS/SQS/DLQ; ALB, target group và ECS Fargate\
**Số giờ:** 9.0 giờ theo worklog\
**Trạng thái:** Đã triển khai và kiểm thử\
**Môi trường:** AWS Academy Learner Lab, `us-east-1`\
**Repository:** https://github.com/pttlinh32/hotel-booking-microservices\
**Ngày thực hiện:** 14–17/10/2026

Từ tuần 2, nhóm triển khai trên tài khoản AWS của Phan Thị Thùy Linh (`055652951832`) theo sự đồng ý của GVHD.

## Công việc đã làm

### Khởi tạo cơ sở dữ liệu và hệ thống truyền thông điệp

1. **Hoàn thiện Amazon RDS PostgreSQL** `hotel-db-instance`, loại `db.t3.micro`, đặt trong private subnet. Em tạo hai database `room_db`, `booking_db` và hai user `room_user`, `booking_user`. Mỗi user chỉ được kết nối tới database của mình. Mật khẩu được lưu trong Secrets Manager và sử dụng qua cấu hình ECS.

   ![Thông số RDS PostgreSQL](images/033-rds-configuration.jpg)

   ![Kiểm tra phân quyền riêng và đăng nhập hai database](images/073-rds-user-isolation-va-tables.jpg)

   Hai service đã tạo bảng `rooms`, `bookings` và ghi dữ liệu kiểm thử thành công. RDS security group chỉ nhận kết nối cổng `5432` từ ECS security group.

   ![Dữ liệu trong hai bảng rooms và bookings](images/089-rds-rooms-bookings-du-lieu.jpg)

2. **Tạo DynamoDB** `hotel-booking-audit-logs` với partition key `bookingId`, sort key `timestamp`, chế độ On-Demand. Bảng đã nhận hai bản ghi `BOOKING_CREATED` từ `booking-service` sau khi kiểm thử đặt phòng.

   ![Audit log được ghi vào DynamoDB](images/075-dynamodb-audit-bookings.jpg)

3. **Tạo SNS, SQS và DLQ** gồm topic `hotel-booking-events`, queue `booking-notification-queue` và DLQ `booking-notification-dlq`. Em kết nối SNS với SQS bằng subscription, bật Raw Message Delivery và cấu hình `maxReceiveCount = 3`. Queue chính giữ message 4 ngày; DLQ giữ 14 ngày.

   ![SNS topic và subscription đã xác nhận](images/041-sns-topic-subscription.jpg)

   ![Cấu hình queue chính và redrive sang DLQ](images/047-sqs-retention-va-redrive.jpg)

### Cấu hình ALB và target group

Em tạo ALB `hotel-booking-alb`, loại **Internet-facing**, trên hai public subnet tại `us-east-1a` và `us-east-1b`. ALB dùng security group `hotel-booking-alb-sg` và listener **HTTP:80**. Ba target group dùng loại **IP**, cổng **8080**, health check path `/actuator/health`.

| Priority | Đường dẫn | Target group |
|---|---|---|
| 10 | `/api/rooms*` | `hotel-room-tg` |
| 20 | `/api/bookings*` | `hotel-booking-tg` |
| 30 | `/api/notifications*` | `hotel-notification-tg` |
| Default | Khi không khớp rule nào | Trả về `404 Not Found` |

Em đã đồng bộ các đường dẫn với backend và gắn tag `Project=hotel-booking-microservices`, `Owner=24110303` cho tài nguyên.

![Listener với ba rule API và default response 404](images/085-alb-rules-final-co-tag.jpg)

### Build image và triển khai ECS Fargate

Em lấy mã nguồn từ nhánh `linh-week-2-backend`, build và push ba image phiên bản `v1.0.0` lên ECR. Trong quá trình chạy thử, em bổ sung cấu hình `ObjectMapper` cho booking/notification và sửa xử lý message lỗi để SQS có thể retry, chuyển sang DLQ. Bản sửa đã push lên nhánh `linh-week-2-backend`, commit `e72130b`; tệp `backend-week2-fixes.patch` được lưu kèm bộ minh chứng.

| Service | ECR image | Task definition |
|---|---|---|
| `room-service` | `hotel-room-service:v1.0.0` | `hotel-room-service:1` |
| `booking-service` | `hotel-booking-service:v1.0.0` | `hotel-booking-service:1` |
| `notification-service` | `hotel-notification-service:v1.0.0` | `hotel-notification-service:1` |

![Image booking-service phiên bản v1.0.0 trên ECR](images/090-ecr-booking-v100-final.jpg)

Em tạo ba task definition sử dụng **Fargate**, mạng `awsvpc`, Linux/X86_64, **0.25 vCPU**, **512 MiB RAM**, cổng container **8080**. Task role và task execution role đều dùng `LabRole`. Log của các container được ghi vào CloudWatch, giữ trong 3 ngày.

![Ba task definition đã tạo](images/048-ecs-ba-task-definitions.jpg)

![Thông số task definition của room-service](images/050-room-task-definition-day-du.jpg)

Em triển khai ba ECS service trong cluster `hotel-cluster`, mỗi service chạy một task trong private subnet và liên kết với target group tương ứng. NAT Gateway được dùng trong đợt kiểm thử để các task tải image và gọi dịch vụ AWS. Booking gọi room qua private DNS `room-service.hotel.local` trong Route 53.

![Ba ECS service chạy trên Fargate](images/055-ecs-ba-service-running.jpg)

Cả ba target group đã đạt trạng thái **Healthy**.

![Target của room-service Healthy](images/076-room-healthy-final.jpg)

![Target của booking-service Healthy](images/077-booking-healthy-final.jpg)

![Target của notification-service Healthy](images/069-notification-target-healthy.jpg)

### Kiểm thử các thành phần

Em kiểm thử bằng dữ liệu giả có mã khách hàng `WEEK2-TEST` và `WEEK2-E2E`.

| Nội dung kiểm thử | Kết quả |
|---|---|
| Tra cứu phòng qua ALB | `200 OK` |
| Tạo booking | `201 Created`, trạng thái `CONFIRMED` |
| Trạng thái phòng sau khi đặt | Chuyển sang `RESERVED` |
| Đặt lại phòng đã được giữ | `409 Conflict` |
| Ghi dữ liệu RDS | Có dữ liệu trong `rooms` và `bookings` |
| Ghi audit log DynamoDB | Có hai bản ghi `BOOKING_CREATED` |
| SNS → SQS → notification | Nhận sự kiện, ghi log email mô phỏng và ACK message |
| Gửi lại cùng `eventId` | Bỏ qua thông báo trùng |
| Message lỗi | Sau ba lần xử lý lỗi, chuyển sang DLQ |

![Kiểm thử tạo booking 201 và đặt trùng 409](images/088-api-post201-conflict409.jpg)

![CloudWatch log gửi email mô phỏng và ACK message](images/071-cloudwatch-email-ack-idempotency.jpg)

![Log bỏ qua sự kiện trùng](images/083-idempotency-final.jpg)

![Message lỗi đã được chuyển vào DLQ](images/082-dlq-message-body.jpg)

Khi kiểm thử DLQ, em tạm giảm visibility timeout xuống 10 giây để rút ngắn thời gian chờ; sau đó khôi phục về 60 giây.

### Kết thúc buổi thực hành

Sau khi chụp minh chứng, em đưa ba ECS service về **desired count = 0**, dừng RDS và EC2 quản trị tạm. Rule truy cập tạm của EC2 được gỡ bằng cách khôi phục security group ban đầu. NAT Gateway và Elastic IP tạo cho đợt kiểm thử được gỡ để tiết kiệm ngân sách.

![ECS đã về 0 running và 0 pending](images/091-ecs-scale-zero.jpg)

![RDS đã dừng sau kiểm thử](images/092-rds-stopped.jpg)

**Kết quả:** Đã triển khai ba microservice lên ECS Fargate, kiểm tra target Healthy và thực hiện thành công luồng đặt phòng, lưu dữ liệu, ghi audit log và xử lý thông báo. Tài nguyên chạy theo giờ đã được dừng hoặc thu hồi sau buổi thực hành.

## Nghiên cứu dịch vụ

1. **RDS và Secrets Manager:** Tách database, user và quyền truy cập theo từng service; đưa thông tin đăng nhập vào ECS qua Secrets Manager.
2. **ALB và ECS Fargate:** Phân luồng request theo đường dẫn, chạy container độc lập và kiểm tra sức khỏe qua `/actuator/health`.
3. **SNS, SQS và DLQ:** Truyền sự kiện giữa các service, xử lý thông báo bất đồng bộ và giữ lại message lỗi để kiểm tra.
4. **CloudWatch Logs:** Theo dõi khởi động ứng dụng, kết nối database, xử lý sự kiện và kiểm thử chống trùng.

## Sơ đồ kiến trúc

![Sơ đồ kiến trúc hệ thống hotel booking microservices](images/so-do-kien-truc-week-2.png)

Sơ đồ thể hiện cấu hình đã triển khai và kiểm thử tuần 2: ALB HTTP cổng 80, private DNS Route 53, NAT Gateway tạm và notification mô phỏng email bằng log.

**Sử dụng AI:** Em dùng AI hỗ trợ cấu hình AWS, build và kiểm tra backend, chọn ảnh minh chứng và biên tập báo cáo.

## Công việc chưa hoàn thành so với tasks.md tuần 2

- Chưa bàn giao trực tiếp thông tin kết nối cho Linh. Thông tin database đã được chuẩn bị trong Secrets Manager; image và các cấu hình triển khai đã có trên tài khoản nhóm.
- Cloud Map chưa triển khai do giới hạn quyền Learner Lab. Phần gọi REST nội bộ đã kiểm thử bằng private DNS Route 53; việc tự động cập nhật DNS khi task đổi IP chưa hoàn thiện. Tệp `resume-lab.py` kèm minh chứng cập nhật IP khi khởi động lại môi trường.

## Kế hoạch tuần 3

1. **Hoàn tất bàn giao thông tin kết nối qua Secrets Manager:** Tích hợp đọc biến môi trường cấu hình kết nối database, SNS topic và SQS queue từ AWS Secrets Manager vào ECS Task Definition để tăng cường bảo mật.
2. **Triển khai Cloud Map hoặc giải pháp Service Discovery thay thế:** Tối ưu hóa cơ chế định tuyến nội bộ trong môi trường AWS Academy Learner Lab (hoàn thiện kịch bản cập nhật Route 53 Private Hosted Zone hoặc ECS Service Connect).
3. **Thiết lập chính sách Auto Scaling cho các dịch vụ:** Cấu hình Target Tracking Scaling Policy cho cả 3 microservices trên ECS Fargate dựa theo CPU Utilization và Request Count của ALB.
4. **Bổ sung kiểm thử tự động các luồng lỗi:** Kiểm thử kịch bản timeout khi gọi REST nội bộ, mô phỏng cơ chế retry/chặn lỗi và kiểm thử việc đẩy tin nhắn lỗi sang Dead Letter Queue (DLQ).
5. **Chuẩn bị nội dung báo cáo tổng kết:** Thu thập số liệu vận hành, tổng kết kiến trúc hệ thống và chuẩn bị slide, video demo phục vụ nghiệm thu đồ án.

