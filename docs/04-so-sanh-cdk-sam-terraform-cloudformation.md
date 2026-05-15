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
