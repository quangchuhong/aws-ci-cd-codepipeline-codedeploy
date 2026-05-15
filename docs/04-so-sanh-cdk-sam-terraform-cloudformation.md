# So sánh tóm tắt: CloudFormation / SAM / CDK / Terraform

Tài liệu này dùng nội bộ để định hướng chọn công cụ IaC khi làm việc với AWS (đặc biệt là serverless).

---

## 1. CloudFormation

### Ý nghĩa

- Là **engine IaC gốc của AWS**.
- Đọc template (YAML/JSON) → tạo **Stack** → tạo/cập nhật/xóa resource AWS.
- Là nền tảng mà **SAM** và **CDK** đều dựa lên.

### Cách sử dụng

- Viết template YAML/JSON:
  - Ví dụ: `AWS::EC2::Instance`, `AWS::S3::Bucket`, `AWS::Lambda::Function`, `AWS::DynamoDB::Table`, …
- Triển khai:
  - AWS Console (CloudFormation).
  - Hoặc AWS CLI:
    - `aws cloudformation create-stack`
    - `aws cloudformation update-stack`
    - `aws cloudformation delete-stack`
- Trạng thái (state) stack được AWS quản lý, không cần tự quản lý file state.

### Khi nên dùng

- Hệ thống **chỉ chạy trên AWS**.
- Muốn dùng giải pháp **native** của AWS, không phụ thuộc tool bên ngoài.
- Hạ tầng từ đơn giản đến phức tạp (VPC, EC2, RDS, IAM, ECS/EKS, …), chấp nhận viết YAML.

---

## 2. SAM (AWS Serverless Application Model)

### Ý nghĩa

- **Lớp mở rộng trên CloudFormation**, chuyên cho **serverless** trên AWS:
  - Lambda, API Gateway (REST/HTTP), EventBridge, Step Functions, DynamoDB, SQS, SNS, …
- Cung cấp:
  - **Cú pháp rút gọn**: `AWS::Serverless::Function`, `AWS::Serverless::Api`, `AWS::Serverless::StateMachine`, …
  - **SAM CLI** để build, test local, deploy tự động.

> Về bản chất: SAM template + SAM CLI → **CloudFormation template + CloudFormation stack**.

### Cách sử dụng

- Viết file `template.yaml`:

  ```yaml
  AWSTemplateFormatVersion: '2010-09-09'
  Transform: AWS::Serverless-2016-10-31

  Resources:
    HelloFunction:
      Type: AWS::Serverless::Function
      Properties:
        CodeUri: src/
        Handler: app.lambda_handler
        Runtime: python3.11
        Events:
          ApiEvent:
            Type: Api
            Properties:
              Path: /hello
              Method: GET
```

- Dùng SAM CLI:

  - Build:
```bash
sam build

```

  - Test Lambda local:
```bash
sam local invoke HelloFunction -e event.json

```

  - Test API Gateway local:
```bash
sam local start-api
# gọi: http://127.0.0.1:3000/hello?name=local

```

  - Deploy (lần đầu):
```bash
sam deploy --guided

```

  - Deploy (các lần sau):
```bash
sam build
sam deploy

```

- SAM sẽ:

  - Build package cho Lambda (và các resource liên quan).
  - Upload code lên S3.
  - Sinh CloudFormation template đã “expand”.
  - Gọi CloudFormation create/update stack.


## Điểm mạnh chính của SAM

**1. Cú pháp serverless rút gọn**

  - Một resource AWS::Serverless::Function có thể tự tạo:
    - Lambda function,
    - Permission,
    - API Gateway event / SQS event / DynamoDB stream event, …
  - So với CloudFormation thuần hoặc Terraform, template ngắn và dễ đọc hơn cho use case serverless.


**2. Hỗ trợ kiểm thử local**

  - sam local invoke: chạy 1 Lambda với event JSON trên máy dev.
  - sam local start-api: giả lập API Gateway + Lambda, gọi qua HTTP như thật.
  - Có thể gắn debugger (VS Code, PyCharm, …) để debug như app bình thường.

**3. Tự động hóa build + deploy**

  - Gói bundling/build cho Lambda (npm/pip, …).
  - Package artifact lên S3, generate template CloudFormation.
  - Gọi CloudFormation deploy với đủ capabilities (IAM, AUTO_EXPAND, …).
  - Dễ đưa vào CI/CD (GitHub Actions, GitLab CI, Jenkins, CodePipeline).
