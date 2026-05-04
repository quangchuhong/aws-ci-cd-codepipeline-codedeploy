# LAB: CI/CD Maven Spring Boot lên AWS (CodePipeline + CodeBuild + ECR + ECS Fargate + CodeDeploy Blue‑Green)

## 1. Mục tiêu

- Build & test ứng dụng Spring Boot (Maven) trên **CodeBuild**
- Build Docker image, push lên **Amazon ECR**
- Deploy ứng dụng lên **ECS Fargate** với **blue‑green deployment** qua **CodeDeploy**
- Orchestrate toàn bộ luồng bằng **CodePipeline**

---

## 2. Kiến trúc & Dịch vụ

**Dịch vụ sử dụng:**

- **CodeCommit** (hoặc GitHub): lưu source code
- **CodeBuild**: `mvn test`, `mvn package`, build & push Docker image
- **ECR**: lưu Docker image
- **ECS Fargate**: chạy container (service `shopping-cart-service`)
- **ALB** + 2 Target Group: phục vụ blue‑green
- **CodeDeploy**: triển khai blue‑green cho ECS
- **CodePipeline**: pipeline 3 stage (Source → Build → Deploy)

**Giả sử đặt tên:**

- ECS Cluster: `shopping-cart-cluster`
- ECS Service: `shopping-cart-service`
- Container name: `shopping-cart-container`
- ECR repo: `shopping-cart-repo`
- App port: `8070`
- Region: `ap-southeast-1` (thay theo region của bạn)

---

## 3. Workflow CI/CD (Text Diagram)

### 3.1. Chuỗi workflow tổng thể

```text
┌────────────┐        ┌────────────────┐        ┌────────────────┐
│ Developer  │        │   CodeCommit   │        │  CodePipeline  │
│  (local)   │        │   / GitHub     │        │ (Orchestrator) │
└────┬───────┘        └───────┬────────┘        └───────┬────────┘
     │   git push              │    Source change event          │
     └────────────────────────>│                                │
                               └───────────────────────────────>│
                                                   Start pipeline
                                                           │
                                                           v
                                            ┌──────────────┴─────────────┐
                                            │     Stage 1: SOURCE        │
                                            │  Lấy code từ CodeCommit    │
                                            └──────────────┬─────────────┘
                                                           │ SourceArtifact
                                                           v
                                            ┌──────────────┴─────────────┐
                                            │     Stage 2: BUILD         │
                                            │   AWS CodeBuild            │
                                            │ - mvn test                 │
                                            │ - mvn package              │
                                            │ - docker build + push ECR  │
                                            │ - tạo imagedefinitions.json│
                                            └──────────────┬─────────────┘
                                                     BuildArtifact
                                                           │
                                                           v
                                            ┌──────────────┴─────────────┐
                                            │     Stage 3: DEPLOY        │
                                            │    AWS CodeDeploy (ECS)    │
                                            │ - Tạo TaskDef mới          │
                                            │ - Tạo Green task set       │
                                            │ - Health check             │
                                            │ - Chuyển traffic Blue→Green│
                                            │ - Rollback nếu lỗi         │
                                            └──────────────┬─────────────┘
                                                           │
                                                           v
                                       ┌────────────────────────────┐
                                       │     ECS Fargate Service    │
                                       │  Blue/Green + ALB + TGs    │
                                       └────────────────────────────┘
```
