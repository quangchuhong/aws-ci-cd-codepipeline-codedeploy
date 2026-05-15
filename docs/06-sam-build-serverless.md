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

Ví dụ tối giản 1 Lambda + 1 API Gateway (REST):
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Simple SAM API demo

Globals:
  Function:
    Runtime: python3.11
    Timeout: 5
    MemorySize: 128

Resources:
  HelloFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: app.lambda_handler
      Environment:
        Variables:
          STAGE: dev
      Events:
        ApiEvent:
          Type: Api
          Properties:
            Path: /hello
            Method: GET

Outputs:
  ApiUrl:
    Description: "API base URL"
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/hello"

```
Ý nghĩa chính:

  - Transform: AWS::Serverless-2016-10-31 → dùng SAM.
  - AWS::Serverless::Function + Events.ApiEvent:
    - Tự tạo Lambda + API Gateway route GET /hello.

---

## 4. Lambda handler (src/app.py)

Ví dụ Python:
```python
import json
import os

def lambda_handler(event, context):
    print("Event:", json.dumps(event))

    name = (event.get("queryStringParameters") or {}).get("name", "world")
    stage = os.environ.get("STAGE", "unknown")

    body = {
        "message": f"Hello {name} from Lambda (stage={stage})"
    }

    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps(body),
    }

```

## 5. Quy trình sử dụng SAM CLI

### 5.1. Build

Chuẩn bị package cho Lambda:
```bash
sam build

```
SAM sẽ tạo thư mục .aws-sam/ chứa code đã build.


