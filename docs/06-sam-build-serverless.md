# README – Hướng dẫn ngắn sử dụng AWS SAM cho ứng dụng serverless

Tài liệu này tóm tắt cách dùng **AWS SAM** để triển khai 1 ứng dụng serverless cơ bản (Lambda + API Gateway).

---

## 1. Khái niệm nhanh

- **SAM (Serverless Application Model)** là lớp mở rộng trên **CloudFormation**:
  - Cho phép viết resource serverless ngắn gọn (`AWS::Serverless::Function`, `AWS::Serverless::Api`, …).
  - Dùng **SAM CLI** để:
    - Build package Lambda,
    - Chạy thử local (Lambda, API),
    - Deploy lên AWS (thông qua CloudFormation).

---

## 2. Cấu trúc project mẫu

Ví dụ 1 API đơn giản:

```text
sam-hello-api/
├─ template.yaml      # SAM template
└─ src/
   └─ app.py          # Lambda handler
```
---

## 3. SAM template cơ bản (template.yaml)
