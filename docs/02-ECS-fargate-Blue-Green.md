# ECS Fargate & Blue-Green Deployment – Overview

Tài liệu này tóm tắt cơ chế hoạt động của **Amazon ECS Fargate** và **triển khai Blue‑Green trên ECS**, tập trung vào cách hệ thống vận hành ở runtime, **không đi vào chi tiết từng bước cấu hình**.

---

## 1. ECS Fargate – Cơ chế hoạt động

### 1.1. Fargate là gì?

**AWS Fargate** là chế độ chạy container “serverless” cho ECS:

- Bạn định nghĩa **Task Definition** (image, CPU, memory, port, env, log driver, IAM role…).
- Bạn tạo **ECS Service** để giữ một số lượng task nhất định luôn chạy.
- AWS tự lo:
  - Provision & quản lý EC2 host bên dưới.
  - Scheduling task lên hạ tầng phù hợp.
  - Scale capacity vật lý tương ứng.

Bạn không quản lý EC2, không cần nghĩ đến cluster capacity ở mức node, chỉ làm việc với **task** và **service**.

### 1.2. Task, Service, và ALB

- **Task Definition**:
  - Định nghĩa 1 “đơn vị chạy” (task) gồm 1 hoặc nhiều container.
  - Ví dụ: 1 task chạy 1 container Nginx/Spring Boot.

- **Task**:
  - Một instance đang chạy của Task Definition.
  - Mỗi task Fargate có ENI/IP riêng, gắn vào subnet và security group.

- **Service**:
  - Đảm bảo có `desiredCount` task luôn RUNNING.
  - Nếu 1 task chết, ECS service tự tạo task mới.

- **Application Load Balancer (ALB)**:
  - Nhận HTTP/HTTPS traffic từ user.
  - Forward tới **Target Group** chứa danh sách IP:port của các task.
  - Sử dụng **health check** (path, port) để đánh dấu target healthy/unhealthy và chỉ gửi traffic tới target healthy.

### 1.3. Scaling (tăng/giảm số task)

Fargate scale bằng **số lượng task**:

- `desiredCount = 3` → 3 task → 3 container app giống nhau đang phục vụ.
- Autoscaling policy (Application Auto Scaling) có thể tự tăng/giảm `desiredCount` theo:
  - CPU/Memory sử dụng.
  - RequestCount từ ALB.
  - Hoặc metric tùy chỉnh.

Khi scale out:

- ECS tạo thêm task mới (kéo image từ ECR, gắn ENI/SG).
- Đăng ký vào Target Group → health check OK → bắt đầu nhận traffic.

Khi scale in:

- ECS dừng bớt task.
- ALB deregister target, không gửi traffic vào task đã bị stop.

---

## 2. Blue‑Green trên ECS – Cơ chế tổng thể

**Mục tiêu**: cập nhật phiên bản ứng dụng **không downtime** và **rollback nhanh**, bằng cách chạy song song:

- **Blue**: phiên bản hiện tại (đang phục vụ traffic).
- **Green**: phiên bản mới (được dựng và kiểm tra trước khi chuyển traffic).

Các thành phần chính:

- **Amazon ECS (Fargate)**: chạy task & service.
- **AWS CodeDeploy**: tự động hóa triển khai Blue‑Green.
- **ALB + 2 Target Groups**:
  - TG‑Blue: trỏ vào task set Blue.
  - TG‑Green: trỏ vào task set Green.

---

## 3. Luồng hoạt động Blue‑Green ECS (theo CodeDeploy)

### 3.1. Trạng thái ban đầu (Blue)

- ECS service có **1 task set “primary”** (Blue), chạy version hiện tại.
- ALB listener production (vd port 80) forward 100% traffic đến **TG‑Blue**.
- Người dùng chỉ tương tác với Blue.

### 3.2. Khởi tạo deployment (tạo Green)

Khi bạn tạo 1 deployment mới (thường qua CodePipeline/CLI/console):

1. CodeDeploy đọc:
   - Task Definition mới (image mới, cấu hình mới).
   - AppSpec mô tả service, container name, port, TG, listener.
2. CodeDeploy đăng ký **Task Definition revision mới**.
3. CodeDeploy yêu cầu ECS tạo **Green task set**:
   - Task set này dùng TaskDef mới.
   - Được gắn vào **TG‑Green** của ALB.
   - Số lượng task Green được tính từ cấu hình service/deployment.

Kết quả: Blue và Green cùng chạy song song, nhưng **traffic vẫn đang ở Blue**.

### 3.3. Health check và kiểm tra Green

- ALB health check TG‑Green (ví dụ path `/`, `/health`, `/home`):
  - Nếu tất cả target Green healthy → Green được coi là “sẵn sàng”.
  - Nếu health check Green fail:
    - CodeDeploy đánh dấu deployment FAILED.
    - **Không** chuyển traffic khỏi Blue.

Ở bước này, có thể:

- Chạy test nội bộ, kiểm tra log Green.
- Kết hợp CloudWatch Alarms (latency, 5xx, CPU,…) để quyết định rollback nếu có vấn đề trong quá trình shift traffic.

### 3.4. Chuyển traffic (Blue → Green)

Khi Green OK, CodeDeploy dùng **ALB listener** để điều khiển traffic:

- Tuỳ `deploymentConfig`:
  - **AllAtOnce**:
    - 100% traffic → TG‑Green.
    - 0% traffic → TG‑Blue.
  - **Canary/Linear**:
    - Tăng dần traffic cho Green (ví dụ 10% → 50% → 100%), giúp quan sát hành vi thật trước khi full cutover.

Trong suốt quá trình này, CodeDeploy theo dõi health check + alarm:

- Nếu mọi thứ ổn → tiến đến 100% Green.
- Nếu alarm/health xấu → rollback.

### 3.5. Hoàn tất và dọn dẹp

Khi deploy thành công:

- **Green trở thành phiên bản “primary”** của service.
- **Blue**:
  - Có thể bị giữ 
