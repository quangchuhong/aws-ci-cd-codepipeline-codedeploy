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
  - Khung giờ cho
