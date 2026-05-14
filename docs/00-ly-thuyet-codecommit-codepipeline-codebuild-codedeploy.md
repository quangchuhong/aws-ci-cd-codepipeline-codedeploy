# README – Tổng quan CodeCommit, CodeBuild, CodePipeline, CodeDeploy trong CI/CD

Tài liệu này tóm tắt **chức năng**, **cơ chế hoạt động** và **các kịch bản CI/CD thường dùng** với 4 dịch vụ DevOps chính của AWS:

- **CodeCommit**
- **CodeBuild**
- **CodePipeline**
- **CodeDeploy**

---

## 1. AWS CodeCommit

### 1.1. CodeCommit là gì?

CodeCommit là **Git repository managed** trong AWS:

- Thay thế cho GitHub/GitLab/Bitbucket cho repo private.
- Lưu trữ source code, IaC, manifest GitOps, v.v.
- Tích hợp trực tiếp với:
  - **AWS IAM** (quyền truy cập),
  - **CodePipeline** (Source stage),
  - **CloudWatch Events / EventBridge** (trigger pipeline khi push code).

### 1.2. Tính năng chính

- Git full tính năng:
  - Branch, tag, merge, pull request (code review cơ bản).
  - Diff, comment trên web UI.
- Quyền truy cập:
  - Sử dụng IAM user/role, policy chi tiết.
  - Hỗ trợ HTTPS (credential helper `aws codecommit`), SSH.
- Tích hợp:
  - CodePipeline: Source provider = CodeCommit.
  - CloudWatch Events: trigger mỗi khi có push/merge.

### 1.3. Vai trò trong CI/CD

Trong pipeline, CodeCommit thường đóng vai trò:

- **Source of truth**:
  - `app` repo: chứa code ứng dụng (Java, Node.js, .NET, …).
  - `infra` repo: Terraform/CloudFormation/CDK.
  - `gitops` repo: manifest K8s/ArgoCD.
- **Trigger CI/CD**:
  - Push code → EventBridge → Start CodePipeline.

---

## 2. AWS CodeBuild

### 2.1. CodeBuild là gì?

CodeBuild là **dịch vụ build/test managed**:

- Chạy **container build** theo cấu hình `buildspec.yml`.
- Không phải tự quản Jenkins agent / máy build.
- Tự scale theo số job.

### 2.2. Cơ chế hoạt động

- Bạn định nghĩa **project**:
  - Image build (vd: `aws/codebuild/standard:7.0`).
  - Role IAM (quyền S3, ECR, CodeCommit…).
  - `buildspec.yml`: mô tả các bước.

- `buildspec.yml` gồm:
  - `phases`:
    - `install` – cài tool (maven, npm, trivy, sonar-scanner…).
    - `pre_build` – run test/scan trước khi build.
    - `build` – compile, package, docker build.
    - `post_build` – scan image, push ECR, update GitOps repo, v.v.
  - `artifacts` – files trả về cho CodePipeline hoặc upload S3.

Ví dụ luồng trong 1 build:

1. `install`:
   - Cài dependency-check, trivy, sonar-scanner.
2. `pre_build`:
   - `mvn test`
   - scan dependency.
3. `build`:
   - `mvn package`
   - docker build.
4. `post_build`:
   - trivy scan image.
   - push image ECR.
   - update GitOps repo.

### 2.3. Vai trò trong CI/CD

Trong pipeline, CodeBuild thường dùng để:

- **CI**:
  - Build + Unit test.
  - Static analysis / SonarQube / lint.
  - Dependency scan (OWASP Dependency-Check…).
- **Chuẩn bị artifact cho CD**:
  - Build Docker image & push ECR.
  - Tạo `imagedefinitions.json`, `taskdef.json`, `appspec.yaml`.
  - Upload bundle (.zip) lên S3 (cho Lambda, EC2/EB, …).
  - Update GitOps repo (ArgoCD) với tag image mới.

---

## 3. AWS CodePipeline

### 3.1. CodePipeline là gì?

CodePipeline là **orchestrator CI/CD**:

- Kết nối các bước:
  - Source → Build → Test → Approve → Deploy → …
- Cho phép tự động hoá từ commit đến deploy.
- Hoạt động dạng **stage** và **action**:
  - Mỗi stage có 1 hoặc nhiều action (song song).

### 3.2. Cơ chế hoạt động

- Bạn khai báo pipeline với các stage:

  - **Source**:
    - Lấy code/artifact từ CodeCommit, GitHub, S3, ECR…
    - Trigger khi có thay đổi.

  - **Build/Test**:
    - Gọi CodeBuild (hoặc Lambda, CloudFormation, v.v.).

  - **Approval**:
    - Manual approval (người phải bấm OK trước khi vào stage tiếp theo).

  - **Deploy**:
    - ECS (Blue/Green),
    - CodeDeploy,
    - CloudFormation,
    - Lambda,
    - Elastic Beanstalk,
    - v.v.

- Mỗi action nhận **input artifact** và trả về **output artifact** (bundle S3 hoặc metadata).

### 3.3. Vai trò trong CI/CD

CodePipeline là “xương sống” pipeline:

- **CI**:
  - Stage Source (CodeCommit) → Build (CodeBuild).
- **CD**:
  - Build → Deploy (CodeDeploy ECS/EC2/Lambda),
  - Hoặc Build → update GitOps → ArgoCD self-sync.
- Tự động trigger theo source:
  - Ví dụ: chỉ cho branch `release/*` hoặc `main` chạy stage Deploy.

---

## 4. AWS CodeDeploy

### 4.1. CodeDeploy là gì?

CodeDeploy là **dịch vụ deploy orchestrator**:

- Hỗ trợ:
  - **EC2 / On-Prem**: deploy app lên máy ảo/vật lý (agent + AppSpec).
  - **ECS**: Blue‑Green deploy cho container (Fargate/EC2).
  - **Lambda**: Canary/Linear/AllAtOnce rollout Lambda version (alias).

### 4.2. Cơ chế hoạt động cho từng loại

**EC2 / On‑Prem (in-place hoặc blue‑green)**:

- Sử dụng `appspec.yml` + script hooks:
  - `BeforeInstall`, `AfterInstall`, `ApplicationStart`, `ValidateService`.
- CodeDeploy agent trên từng instance:
  - Nhận lệnh, copy files, chạy script theo AppSpec.
- Hỗ trợ:
  - Rolling update,
  - Blue‑Green với Auto Scaling Group + Load Balancer,
  - Rollback khi hook fail hoặc Alarm ALARM.

**ECS (Blue‑Green)**:

- Dùng với ECS service + ALB:
  - Blue task set: version cũ (TG‑Blue),
  - Green task set: version mới (TG‑Green).
- AppSpec (ECS):

  ```yaml
  version: 0.0
  Resources:
    - TargetService:
        Type: AWS::ECS::Service
        Properties:
          TaskDefinition: taskdef.json / <TASK_DEFINITION>
          LoadBalancerInfo:
            ContainerName: app
            ContainerPort: 80
  ```

- **CodeDeploy:**
  - Tạo TaskDef mới (image mới từ imagedefinitions.json).
  - Tạo Green task set.
  - Health check Green.
  - Chia traffic Blue↔Green (AllAtOnce/Canary/Linear).
  - Rollback nếu Alarm/health fail.

- **Lambda (canary/linear):**

  - Tích hợp với Lambda version + alias:
    - Blue = alias trỏ version cũ,
    - Green = version mới.
  - CodeDeploy điều chỉnh traffic alias:
    - Canary: 10% → đợi → 100%.
    - Linear: 10% → 20% → 30%… → 100%.
  - Theo dõi Alarm và rollback alias nếu lỗi.


### 4.3. Vai trò trong CI/CD

Trong pipeline:

  - Deploy stage = CodeDeploy:
    - Deploy EC2 app (script-based).
    - Rollout ECS Blue‑Green.
    - Rollout Lambda canary/linear.
      
Thường được dùng kết hợp với CodePipeline & CodeBuild:

  - CodeBuild build/test/scan, tạo artifact (zip/image + AppSpec + TaskDef/Imagedef).
  - CodePipeline gửi artifact vào CodeDeploy.
  - CodeDeploy lo phần rollout + rollback.
