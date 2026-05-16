# AWS Systems Manager – Tài liệu tóm tắt

Tài liệu này tóm tắt các thành phần chính của **AWS Systems Manager (SSM)**, ý nghĩa và cách sử dụng cơ bản, dựa trên các nội dung đã trao đổi.

---

## 1. Tổng quan AWS Systems Manager

Systems Manager giúp bạn **quản lý, vận hành và tự động hóa** hạ tầng (EC2, on‑prem, VM, container…) từ một nơi tập trung, mà **không cần SSH/RDP trực tiếp**.

Các thành phần chính:

- **Run Command** – chạy lệnh/script từ xa.
- **Session Manager** – shell vào máy mà không mở port 22/RDP.
- **Patch Manager** – quét & cài đặt bản vá OS tự động.
- **State Manager** – giữ máy ở “trạng thái mong muốn”.
- **Automation** – runbook nhiều bước, tự động hóa quy trình.
- **Change Manager** – quản lý quy trình change có approval.
- **Parameter Store** – lưu config/secret an toàn (có encryption).
- **Application Manager** – view & quản lý theo “ứng dụng”.
- **Hybrid Activations** – quản lý server on‑prem / cloud khác như EC2.

---

## 2. Hybrid Activations

Dùng để **đưa server ngoài AWS (on‑prem, VM, cloud khác)** vào Systems Manager dưới dạng **Managed Instance (`mi-xxxx`)**.

### Ý nghĩa

- Cho phép dùng:
  - Run Command
  - Patch Manager
  - State Manager
  - Automation
  - Inventory
- Với server không phải EC2, tạo **mô hình hybrid**.

### Cách dùng tóm tắt

1. **Tạo IAM Role** cho managed instance (policy `AmazonSSMManagedInstanceCore`).
2. **Tạo Activation**:
   - Chọn IAM Role.
   - Set số lượng instance, ngày hết hạn.
   - Nhận `Activation Code` + `Activation ID`.
3. **Cài SSM Agent** trên server on‑prem.
4. **Đăng ký agent** với `Activation Code/ID` + region.
5. Server xuất hiện trong **Managed instances**, dùng được như EC2.

---

## 3. Run Command

Dùng để **chạy lệnh/script từ xa** trên **1 hoặc nhiều EC2/on‑prem** mà **không cần SSH/RDP**.

### Ý nghĩa

- Quản trị hàng loạt:
  - Cài phần mềm.
  - Restart service, reboot.
  - Thu log, dọn dẹp disk.
  - Gọi `AWS-RunPatchBaseline`, …
- Logs tập trung, phân quyền theo IAM, audit đầy đủ.

### Cách dùng tóm tắt

- Vào **Systems Manager → Run Command → Run command**:
  - Chọn **Document**:
    - Linux: `AWS-RunShellScript`
    - Windows: `AWS-RunPowerShellScript`
    - Patch: `AWS-RunPatchBaseline`
    - Hoặc document custom.
  - Chọn **Targets**:
    - Theo instance ID.
    - Hoặc theo tag (best practice).
  - Nhập lệnh/script.
  - Tùy chọn:
    - Timeout, working dir, output (S3, CloudWatch Logs), IAM role.
  - Run và xem output theo từng instance.

Điều kiện: instance/managed instance phải có **SSM Agent**, **IAM role** phù hợp và **kết nối được SSM endpoint**.

---

## 4. Patch Manager

Dùng để **quét (scan) và cài đặt (install) bản vá OS** cho EC2 và managed instances (Linux/Windows).

### Thành phần chính

- **Patch Baseline**  
  - Định nghĩa **patch nào được “Approved”** để cài:
    - OS, loại patch (Security/Critical/Other).
    - Auto-approval sau X ngày.
    - Allow list / deny list.

- **Patch Group**  
  - Tag `PatchGroup=Prod`, `PatchGroup=Dev`… để **nhóm instance**.
  - Mỗi Patch Group gắn với **1 Patch Baseline**.

- **Maintenance Window**  
  - Khung giờ cho phép chạy patch (VD: CN 01:00–03:00).

- **SSM Document**  
  - Thường dùng: `AWS-RunPatchBaseline` (Scan/Install).

### Luồng hoạt động

- **Scan**:
  - Chạy `AWS-RunPatchBaseline` với mode Scan.
  - Thu thông tin patch thiếu → xem ở **Compliance / Patch Manager**.
- **Install**:
  - Chạy mode Install.
  - Cài patch đã “Approved”; có thể reboot nếu cần.
- Thường **kết hợp Maintenance Window**:
  - Tạo MW → gán Run Command `AWS-RunPatchBaseline` → target patch group.

---

## 5. SSM Documents (Command & Automation)

**SSM Document** định nghĩa **những gì SSM sẽ làm**. Có nhiều loại, ở đây tập trung:

- **Command document** (dùng với Run Command, State Manager).
- **Automation document** (runbook nhiều bước, dùng với Automation).

### Thành phần cơ bản (Command document)

- `schemaVersion`: version schema (thường `2.2`).
- `description`: mô tả.
- `parameters`: khai báo các input từ bên ngoài.
- `mainSteps`:
  - Các bước thực thi:
    - `aws:runShellScript` (Linux/Unix)
    - `aws:runPowerShellScript` (Windows)
  - Có thể dùng `precondition` để tách Linux/Windows.

### Tách Linux / Windows trong cùng document

- Dùng `precondition` với `platformType`:
  - Step Linux: `aws:runShellScript` + `StringEquals: ["platformType", "Linux"]`.
  - Step Windows: `aws:runPowerShellScript` + `StringEquals: ["platformType", "Windows"]`.

### Parameter trong document

- Khai báo ở `parameters`, dùng với cú pháp `{{ ParameterName }}`.
- Có các thuộc tính:
  - `type` (String, StringList, Boolean, Integer,…)
  - `default`
  - `allowedValues`, `description`, …

Khi chạy document (Run Command/State Manager), hệ thống sẽ hiển thị form để điền các parameter đó.

---

## 6. State Manager

Dùng để **giữ máy chủ ở “trạng thái mong muốn”** bằng cách **áp SSM Document định kỳ**.

### Ý nghĩa

- Đảm bảo:
  - Package/agent phải luôn được cài.
  - Service phải luôn chạy/luôn disabled.
  - File config phải luôn đúng template.
- Tự động sửa lại nếu có người/ứng dụng thay đổi sai.

### Khái niệm chính

- **Association**:
  - Một cấu hình State Manager cụ thể:
    - Document (AWS/custom).
    - Targets (theo tag / instance).
    - Schedule (rate/cron).
    - Parameters.

### Cách dùng tóm tắt

- Vào **State Manager → Create association**:
  - Chọn document (ví dụ cài package, đảm bảo service chạy, script cấu hình…).
  - Chọn target (theo tag `Role=WebServer`, `Env=Prod`…).
  - Đặt schedule (VD: mỗi 15 phút, hoặc cron).
  - Từ đó, State Manager **định kỳ chạy** document đó lên các target.

---

## 7. Automation (Runbook)

Dùng để **xây dựng workflow nhiều bước** (runbook) cho các tác vụ vận hành phức tạp, như:

- Tạo AMI backup + xóa AMI cũ.
- Stop → snapshot → start lại instance.
- Khắc phục sự cố (thu log, restart service, gửi thông báo).
- Tích hợp với Change Manager để chạy change chuẩn.

### Thành phần chính của Automation document

- `schemaVersion`: thường `'0.3'`.
- `description`: mô tả runbook.
- `assumeRole`:
  - IAM Role mà Automation **assume** để gọi API AWS.
  - Role cần trust `ssm.amazonaws.com` + policy phù hợp (EC2, S3,…).
- `parameters`:
  - Input cho runbook (InstanceId, list instance, retentionDays,…).
- `mainSteps`:
  - Các bước workflow:
    - `aws:executeAwsApi` – gọi API AWS trực tiếp.
    - `aws:runCommand` – gọi Run Command.
    - `aws:executeScript` – chạy code Python/PowerShell inline.
    - `aws:approve` / `aws:reject` / `aws:pause` – phê duyệt, tạm dừng.
    - Các action chuyên biệt khác (`aws:createImage`, `aws:createStack`,…).
  - Có thể điều khiển flow:
    - `nextStep`, `isEnd`, `onFailure`, `timeoutSeconds`…
- `outputs`:
  - Định nghĩa output cuối cùng của runbook (JSONPath tới step nào đó).

### Cách sử dụng

- Vào **Automation → Execute automation**:
  - Chọn runbook (AWS cung cấp sẵn hoặc custom).
  - Điền parameter.
  - Thực thi và theo dõi từng step.
- Có thể trigger từ:
  - **Change Manager** (sau khi approve).
  - **EventBridge / CloudWatch Events**.
  - **Maintenance Window**.

---

## 8. Change Manager

Dùng để **quản lý & kiểm soát change** theo quy trình, có **approval**, log, audit.

### Ý nghĩa

- Chuẩn hóa **quy trình change**:
  - Deploy, patch prod, đổi cấu hình quan trọng…
- Đảm bảo:
  - Có người/nhóm approve trước khi chạy.
  - Tách vai trò: người tạo ≠ người approve ≠ runbook.
- Audit đầy đủ: ai đề xuất, ai approve, chạy runbook nào, kết quả ra sao.

### Khái niệm chính

- **Change Template**:
  - Định nghĩa loại change:
    - Form thông tin cần điền.
    - Workflow approval (user/role/group).
    - Runbook Automation sẽ chạy.
- **Change Request**:
  - Người dùng tạo request từ template:
    - Điền thông tin change + parameter cho runbook.
  - Gửi duyệt → được approve → Change Manager gọi Automation theo lịch.

---

## 9. Application Manager

Dùng để **xem và quản lý hệ thống theo “ứng dụng”** thay vì từng resource riêng lẻ.

### Ý nghĩa

- Gom các resource thành **application**:
  - EC2, ECS, Lambda, RDS, ELB, S3, …
- Dựa trên:
  - Tag (thông qua **Resource Groups**).
  - **CloudFormation Stack**.
  - **AppRegistry**.
- Cho phép:
  - Xem toàn bộ tài nguyên của 1 ứng dụng từ 1 màn hình.
  - Thao tác SSM (Run Command, Automation, Patch…) dựa trên app đó.

### Cách dùng tóm tắt

1. Chuẩn hóa **tag/stack** cho application (VD: `Application=PaymentService`).
2. Tạo **Resource Group/Application** dựa trên tag/stack.
3. Vào **Application Manager**, chọn app để xem resource và thao tác.

---

## 10. Parameter Store

Dùng để **lưu trữ cấu hình & secret tập trung**, truy cập được từ EC2, ECS, Lambda, script, SSM Document,…

### Loại parameter

- `String`      – plain text.
- `StringList`  – list string, phân tách bằng dấu phẩy.
- `SecureString` – **mã hóa bằng KMS**, dùng cho secret (password, API key,…).

### Ý nghĩa

- Tách config/secret khỏi code.
- Dễ quản lý theo path:
  - `/app/dev/...`
  - `/app/prod/...`
- Kết hợp IAM + KMS để hạn chế ai/ứng dụng nào được đọc.

### Encryption (SecureString)

- Khi tạo parameter:
  - Chọn type **SecureString**.
  - Chọn KMS key:
    - AWS managed.
    - Hoặc customer managed key.
- Khi đọc:
  - Gọi API/CLI với `WithDecryption=true`.
  - Ứng dụng (hoặc SSM document) phải có quyền KMS + SSM phù hợp.

---

## 11. Gợi ý sử dụng tổng thể

- **Run Command**: thao tác ad‑hoc, vận hành hàng loạt.
- **Patch Manager + Maintenance Window**: patch OS tự động, theo lịch.
- **State Manager**: đảm bảo “trạng thái chuẩn” (service, package, config).
- **Automation Runbook**: quy trình phức tạp nhiều bước, có logic & tích hợp dịch vụ AWS.
- **Change Manager**: thêm lớp **quy trình & approval** cho Automation, đặc biệt môi trường Prod.
- **Parameter Store**: quản lý config/secret tập trung, an toàn.
- **Hybrid Activation**: quản lý server on‑prem/cloud khác bằng cùng công cụ.
- **Application Manager**: vận hành theo khái niệm “ứng dụng” thay vì tài nguyên rời rạc.

---

## 12. Cấu trúc & ý nghĩa các thành phần trong Automation Runbook (tóm tắt)

Phần này tóm tắt cấu trúc của **Automation document** (runbook) và ý nghĩa các thành phần chính.

### 12.1 Cấu trúc cơ bản

Một Automation document (YAML) thường gồm:

- `schemaVersion`
- `description`
- `assumeRole`
- `parameters`
- `mainSteps`
- `outputs`

### 12.2 `schemaVersion` & `description`

- `schemaVersion: '0.3'`
  - Version schema cho Automation (chọn `'0.3'` cho runbook mới).
- `description`
  - Mô tả mục đích runbook (hiển thị trong console, Change Manager).

### 12.3 `assumeRole`

- IAM Role mà Automation **assume** để gọi API AWS.
- Role phải:
  - Trust `ssm.amazonaws.com`.
  - Có policy phù hợp (StopInstances, CreateImage, SNS:Publish,…).
- Giúp tách **quyền runbook** khỏi quyền người chạy.

### 12.4 `parameters`

- Khai báo input cho runbook (vd: `InstanceId`, `InstanceIds`, `RetentionDays`, …).
- Thuộc tính chính:
  - `type` – `String`, `StringList`, `Integer`, `Boolean`, …
  - `default` – giá trị mặc định.
  - `description` – mô tả.
  - `allowedValues` / `allowedPattern` – validate.
- Dùng trong step bằng cú pháp: `{{ ParameterName }}`.

### 12.5 `mainSteps`

Tập hợp các **bước xử lý** theo thứ tự:

Mỗi step có:

- `name` – tên duy nhất (để tham chiếu & lấy output).
- `action` – loại hành động:
  - `aws:executeAwsApi` – gọi API AWS (EC2, S3, RDS, IAM,…).
  - `aws:runCommand` – chạy Command document trên instance.
  - `aws:executeScript` – chạy logic Python/PowerShell inline.
  - `aws:approve` / `aws:reject` / `aws:pause` – approval / dừng chờ.
- `inputs` – tham số cho `action` (Service, Api, InstanceIds, DocumentName, Parameters,…).
- Control flow (tùy chọn):
  - `nextStep` – step tiếp theo nếu thành công.
  - `isEnd: true` – đánh dấu kết thúc runbook.
  - `onFailure` – hành vi khi lỗi:
    - `Abort` / `Continue` / `step:<TênStepKhác>`.
  - `timeoutSeconds` – timeout cho step.

→ `mainSteps` cho phép bạn **xâu chuỗi nhiều hành động** (dừng EC2, snapshot, start lại, gửi SNS,…) trong một runbook có luồng xử lý & lỗi rõ ràng.

### 12.6 `outputs`

- Định nghĩa **output cuối cùng** của runbook:
  - `Name` – tên biến output.
  - `Selector` – JSONPath tới output của một step (vd: `$.StepName.Field`).
  - `Type` – `String`, `StringList`, `Map`, …
- Output dùng để:
  - Xem kết quả trong console.
  - Cho Change Manager / hệ thống khác đọc lại kết quả qua API.

### 12.7 Liên kết với thành phần khác

- **Change Manager**: sử dụng runbook làm “engine” thực thi change sau khi approve.
- **EventBridge / CloudWatch**: trigger runbook theo sự kiện (alarm, EC2 state change,…).
- **Maintenance Window**: chạy runbook theo lịch (backup, patch, cleanup,…).

---
