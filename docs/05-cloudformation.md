# CloudFormation – Tóm tắt các option cấu hình & ý nghĩa sử dụng

Tài liệu này dùng để nắm bức tranh tổng quan về **các “option/config” quan trọng trong CloudFormation**, mục đích và khi nào nên dùng. Không đi sâu demo code.

---

## 1. Template & Stack – khái niệm nền tảng

- **Template**  
  - File định nghĩa hạ tầng (YAML/JSON).
  - Chỉ nói “muốn có cái gì” (EC2, VPC, RDS, Lambda, v.v.).

- **Stack**  
  - Một “instance” của template khi apply lên 1 account/region.
  - Là đơn vị để **create / update / delete** hạ tầng.

**Ý nghĩa sử dụng:**  
Template là “thiết kế”, Stack là “công trình thật”. Mọi thay đổi hạ tầng đều nên thông qua update stack.

---

## 2. Các section chính trong template

### 2.1. `Parameters` – tham số hóa template

- Khai báo giá trị thay đổi theo môi trường (dev/stg/prod), vùng (region), account:
  - Ví dụ: VPC ID, subnet ID, AMI, instance type, tên DB, mật khẩu, v.v.
- Cho phép truyền giá trị qua:
  - Console, AWS CLI, file JSON tham số, CI/CD.

**Dùng khi:**
- Cùng 1 template dùng cho nhiều môi trường.
- Muốn tránh hard-code giá trị nhạy cảm hoặc phụ thuộc môi trường.

---

### 2.2. `Mappings` – bảng tra cứu (lookup table)

- Định nghĩa bảng hằng số, ví dụ:
  - AMI theo region, CIDR theo môi trường, configuration theo AZ.
- Truy xuất bằng hàm lookup.

**Dùng khi:**
- Giá trị “tĩnh” nhưng khác nhau theo key (region/env), không muốn truyền qua Parameters.

---

### 2.3. `Conditions` – điều kiện tạo/sửa resource

- Khai báo logic (true/false) dựa trên Parameters/Mapping.
- Gắn vào resource/Outputs để chỉ tạo trong một số điều kiện.

**Dùng khi:**
- Dev/stg không cần RDS Multi-AZ, nhưng prod cần.
- Một số resource chỉ có ở một vài region.

---

### 2.4. `Resources` – phần cốt lõi

- Khai báo **toàn bộ resource AWS** cần tạo:
  - EC2, VPC, Subnet, ALB, RDS, S3, Lambda, API Gateway, IAM, v.v.
- Mỗi resource:
  - Loại (`Type`)
  - Thuộc tính (`Properties`)
  - Quan hệ phụ thuộc (tự suy ra hoặc khai báo tường minh).

**Dùng khi:**
- Định nghĩa hạ tầng thực sự. Đây là phần “bắt buộc phải có” trong mọi template.

---

### 2.5. `Outputs` – xuất kết quả dùng lại

- Xuất ra thông tin sau khi stack tạo xong:
  - Ví dụ: ALB DNS name, VPC ID, DB endpoint, ARN, URL API, v.v.
- Có thể:
  - Dùng để hiển thị cho người dùng.
  - `Export` để stack khác `Import` sử dụng.

**Dùng khi:**
- Cần giá trị từ 1 stack cho stack khác (chia module).
- Cần cung cấp endpoint / ID cho team khác, CI/CD, hoặc logging.

---

### 2.6. `Metadata` – thông tin phụ trợ

- Thêm thông tin cho tool, IDE, hoặc CloudFormation Helper (cfn-init).
- Không ảnh hưởng logic resource.

**Dùng khi:**
- Gắn chú thích cho linter, tool triển khai nội bộ, hoặc script bootstrap.

---

### 2.7. `Transform` – macro & SAM

- Cho phép “biến đổi” template trước khi CloudFormation xử lý:
  - Ví dụ: `Transform: AWS::Serverless-2016-10-31` (SAM).
  - Hoặc custom macro nội bộ.

**Dùng khi:**
- Muốn dùng SAM cho serverless.
- Muốn viết macro nội bộ để rút gọn template lặp lại.

---

## 3. Các hàm & cơ chế hỗ trợ trong template

### 3.1. Intrinsic Functions – hàm nội tại

Một số hàm hay dùng:

- `Ref` – tham chiếu Parameter/Resource.
- `Sub` – thay biến trong chuỗi (string interpolation).
- `GetAtt` – lấy attribute của resource (ARN, DNSName, v.v.).
- `Join`, `Split`, `Select` – xử lý chuỗi/list.
- `FindInMap` – tra trong `Mappings`.
- `ImportValue` – lấy Output đã Export từ stack khác.
- Các hàm logic: `If`, `Equals`, `And`, `Or`, `Not`.

**Ý nghĩa:**  
Cho phép template linh hoạt, tái sử dụng, ít hard-code.

---

### 3.2. SSM Parameter Store & Secrets – dynamic reference

- Dùng để lấy giá trị (thường là bí mật, config) từ:
  - SSM Parameter Store,
  - Secrets Manager.
- Không lưu plaintext trong template/stack metadata.

**Dùng khi:**
- Cấu hình nhạy cảm (password, token) hoặc thay đổi thường xuyên.
- Muốn tách quyền quản lý hạ tầng và quản lý secret.

---

## 4. Tổ chức template & chia nhỏ hệ thống

### 4.1. Nested Stack

- Một stack “cha” sử dụng resource `AWS::CloudFormation::Stack` để gọi các template “con”.
- Mỗi template con có thể là:
  - `network.yaml`, `app.yaml`, `database.yaml`, `monitoring.yaml`, …

**Ý nghĩa:**

- Chia hạ tầng lớn thành module nhỏ, dễ bảo trì.
- Tái sử dụng module ở nhiều nơi (VD: module VPC chuẩn).

---

### 4.2. StackSet

- Cơ chế đặc biệt để deploy **cùng một template** tới:
  - Nhiều account,
  - Nhiều region.
- Quản lý **Stack Instances** ở mỗi account/region.

**Dùng khi:**
- Tổ chức nhiều account AWS (multi-account) trong Organizations.
- Cần rollout cấu hình chuẩn (guardrail, logging, networking, baseline) trên diện rộng.

---

## 5. Lifecycle & vận hành stack

### 5.1. Change Set

- Trước khi update stack, có thể tạo **Change Set** để xem:
  - Resource nào sẽ tạo / xóa / sửa.
  - Có thuộc tính nào gây recreate resource hay không.

**Dùng khi:**
- Môi trường production, cần kiểm soát chặt thay đổi.
- Muốn review change trước khi cho phép apply.

---

### 5.2. Rollback

- Khi create/update stack thất bại:
  - Mặc định CloudFormation rollback về trạng thái trước đó.
- Có thể:
  - Hủy update đang chạy (`cancel-update-stack`).
  - Điều chỉnh tùy chọn rollback khi create-stack/update-stack.

**Dùng khi:**
- Update gặp lỗi, cần đảm bảo hạ tầng quay về trạng thái ổn định trước đó.

---

### 5.3. Drift Detection

- So sánh trạng thái **thực tế** của resource trên AWS với trạng thái **trong stack**.
- Báo “drift” nếu resource bị chỉnh tay ngoài CloudFormation.

**Dùng khi:**
- Muốn đảm bảo “không ai sửa tay” phá vỡ IaC.
- Audit và cleanup cấu hình lệch.

---

### 5.4. Stack Policy

- Policy JSON gắn vào stack để **bảo vệ** 1 số resource khỏi bị update/xóa.
- Có thể chặn hoặc giới hạn kiểu thay đổi.

**Dùng khi:**
- Muốn bảo vệ resource quan trọng (VD: RDS prod, S3 log) khỏi update nhầm khi apply template.

---

## 6. Cách triển khai (deployment options)

### 6.1. AWS Console

- Upload template (file/local hoặc S3),
- Điền Parameters,
- Bấm create/update stack.

**Dùng khi:**
- Lab, demo nhanh, hoặc team nhỏ thao tác thủ công.

---

### 6.2. AWS CLI

- Dùng lệnh:
  - `create-stack`, `update-stack`, `delete-stack`, `describe-stacks`, `describe-stack-events`, …
- Có thể:
  - Truyền parameters từ file JSON.
  - Tích hợp script/sh.

**Dùng khi:**
- Muốn automate đơn giản,
- Viết script tự động, cron, hoặc tool nội bộ.

---

### 6.3. CI/CD Pipeline

- Sử dụng:
  - AWS CodePipeline/CodeBuild,
  - Hoặc GitHub Actions, GitLab CI, Jenkins, CircleCI, v.v.
- Pipeline thường gồm:
  - Lint template,
  - Generate/cek Change Set,
  - Review/approve,
  - Execute change set / update stack.

**Dùng khi:**
- Muốn chuẩn hóa quy trình release hạ tầng,
- Yêu cầu audit, review, approval trước khi deploy.

---

## 7. Gợi ý sử dụng trong thực tế

- **Dùng Parameters + file JSON** để tách giá trị theo môi trường.
- **Dùng Mappings + Conditions** cho logic theo region/env mà không phải copy template.
- **Tách template bằng Nested Stack** nếu hệ thống phức tạp (network/app/db/monitoring).
- **Dùng Change Set** với production để xem diff trước khi update.
- **Kết hợp SSM/Secrets** cho config & secret nhạy cảm.

CloudFormation là nền tảng IaC gốc của AWS; các công cụ cao hơn như **SAM** và **CDK** thực chất đều sinh/triển khai qua CloudFormation, nên nắm được các option trên giúp hiểu và vận hành tốt bất kỳ stack AWS nào.
