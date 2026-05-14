# README – Tổng quan Elastic Beanstalk, App Runner và ECS Fargate (Express Mode)

Tài liệu này tóm tắt **các tính năng chính** và **cách chúng hoạt động** của 3 dịch vụ chạy ứng dụng trên AWS:

- **Elastic Beanstalk (EB)**
- **AWS App Runner**
- **Amazon ECS Fargate (gọi tắt: ECS express mode)**

Không đi sâu từng bước cấu hình, chỉ tập trung cơ chế và cách dùng.

---

## 1. Elastic Beanstalk (EB)

### 1.1. EB là gì?

Elastic Beanstalk là dịch vụ **PaaS “nửa managed”**:

- Bạn đưa cho EB **gói ứng dụng** (zip/war/jar/Docker image).
- EB tự dựng:
  - **EC2 instances** (VM),
  - **Auto Scaling Group**,
  - **Load Balancer** (ALB/CLB),
  - Security Group, CloudWatch metrics, log forwarding…
- Bạn vẫn có thể:
  - SSH vào EC2,
  - Tinh chỉnh cấu hình (nginx, Tomcat, env vars, scaling, health check…).

Hỗ trợ nhiều nền tảng:

- Node.js, Java (Tomcat/JAR), .NET, Python, Ruby, PHP, Go, Docker (single container)…

### 1.2. Khái niệm chính

- **Application**  
  Nhóm logic ứng dụng (ví dụ: `shopping-cart-app`).

- **Application Version**  
  Mỗi lần upload 1 bundle (zip/war/jar/Docker) → 1 version:
  - Gắn với file trong S3.
  - Dùng để deploy cho environment.

- **Environment**  
  Mỗi environment = 1 stack hạ tầng:
  - Web Server environment: EC2 + ALB + ASG.
  - Worker environment: EC2 + SQS worker.
  - Ví dụ: `shopping-cart-dev`, `shopping-cart-prod`.
  - Mỗi environment chạy **1 Application Version** tại một thời điểm.

### 1.3. Cách EB deploy và scaling

**Deploy (rollout)**

Bạn chọn version mới (v2) cho environment:

- EB áp dụng **Deployment Policy**:
  - **All at once**: update toàn bộ instance cùng lúc (downtime).
  - **Rolling**: update theo batch trong ASG (giảm downtime).
  - **Rolling with additional batch**: tạo thêm batch để giữ capacity.
  - **Immutable**: tạo ASG mới với version mới, test rồi chuyển traffic (gần giống blue‑green).
  - **Traffic Splitting** (kết hợp CodeDeploy): chia traffic giữa version cũ/mới, dùng cho canary.

**Scaling**

- EB dùng **Auto Scaling Group**:
  - Bạn đặt min/max instances (vd: 2–10).
  - Scaling trigger theo CPU, RequestCount hoặc metric khác.
- Khi tải tăng:
  - ECS/ASG thêm instance mới, EB deploy app vào đó.
- Khi tải giảm:
  - Giảm số instance, ALB deregister trước khi terminate.

**Rollback**

- EB lưu nhiều **Application Version**.
- Rollback = chọn lại version cũ cho environment:
  - EB redeploy version đó theo Deployment Policy hiện tại.
- Nếu dùng Traffic Splitting + CodeDeploy:
  - Có thể rollback **tự động** khi health/Alarm xấu trong quá trình rollout.

---

## 2. AWS App Runner

### 2.1. App Runner là gì?

App Runner là dịch vụ **chạy ứng dụng container “siêu managed”**:

- Bạn cung cấp:
  - **Source code** (GitHub/CodeCommit) hoặc
  - **Container image** (ECR).
- App Runner:
  - (Nếu cần) build source thành image,
  - Chạy container,
  - Tự lo:
    - Load balancer nội bộ,
    - TLS (HTTPS) nếu cấu hình,
    - Auto scaling,
    - Health check,
    - Logging.

Mục tiêu: **deploy 1 container app nhanh nhất có thể**, ít phải động tới hạ tầng.

### 2.2. Cách App Runner vận hành

- Bạn tạo **Service** trong App Runner:
  - Trỏ tới Git repo hoặc ECR image.
  - Cấu hình:
    - Port app (vd: 3000, 8080),
    - CPU/Mem per instance,
    - Min/Max instance,
    - Env vars, health check, conns per instance…
- App Runner:
  - Tạo service endpoint (URL public),
  - Lưu image vào hạ tầng managed riêng,
  - Phân phối traffic tới các instance container.

**Scaling**

- App Runner tự scale instance container dựa trên:
  - Concurrent requests,
  - CPU/RAM,
  - Cấu hình min/max.
- Không thấy EC2/ALB/TG; tất cả ẩn bên dưới.

**CI/CD tích hợp**

- Có thể:
  - Dùng “Auto deploy” từ GitHub/CodeCommit:
    - Push code ⇒ App Runner build & deploy.
  - Hoặc:
    - Dùng CodeBuild xây image và push ECR,
    - App Runner service tự deploy khi image mới.

---

## 3. Amazon ECS Fargate (Express Mode)

### 3.1. ECS Fargate là gì?

ECS là **orchestrator container**, Fargate là **chế độ serverless** của ECS:

- Bạn định nghĩa:
  - **Task Definition**: image, CPU, memory, port, env, log driver, IAM role.
  - **Service**: số task cần chạy, load balancer, autoscaling.
- Fargate:
  - Không cần EC2, tự lo máy chủ.
  - Mỗi task = 1 container (hoặc nhiều container) với ENI/IP riêng trong VPC.

Khi nói “ECS express mode” thường là ý:

- ECS + Fargate,
- Cấu hình bằng console/CloudFormation/Terraform/CDK,
- Không tự build EC2 cluster.

### 3.2. Khái niệm chính

- **Cluster**: nhóm logic chứa các service/task.
- **Task Definition**:
  - Bản vẽ chạy container:
    - `containerDefinitions`: name, image, portMappings, env, logConfiguration.
- **Service**:
  - Đảm bảo số task RUNNING = `desiredCount`.
  - Gắn với Load Balancer (ALB/NLB) + Target Group (TG).

### 3.3. Cách ECS Fargate hoạt động

**Chạy task**

- Bạn có TaskDef (image từ ECR, port=8070…).
- Service Fargate:
  - Lên task mới:
    - Kéo image từ ECR,
    - Gắn ENI/SG/subnet,
    - Đăng ký target với TG.
  - ALB:
    - Health check target,
    - Forward traffic tới các task healthy.

**Scaling**

- Autoscaling:
  - Application Auto Scaling gắn với ECS service:
    - Scale theo CPU, RequestCount, custom metric…
  - Khi scale out:
    - Service tăng `desiredCount`,
    - Fargate tạ
