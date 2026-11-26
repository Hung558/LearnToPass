# Checklist Sửa `Report-4_Software Design Specification.docx` (Theo Trình Tự Tuyến Tính Từ Đầu → Cuối)

> **Nguyên tắc sửa:** Đối chiếu 100% với code thật `SEP490_G38-VCMOS_BE` + chuẩn UML 2 từ 2 giáo trình (*UML Distilled* & *Software Modeling & Design*). Học xong File 02, mình và bạn cứ làm lần lượt từ Mục 0 xuống Mục IV, xong mục nào đánh `[x]` mục đó.

---

## Mục 0. Bìa, Mục lục & Change Log (Trang 1–3)
- [ ] **Cập nhật lại Mục lục (Table of Contents):** Hiện vẫn còn nguyên chữ mẫu template FPT (`3.1 table_name1`, `3.3 table_name3`, `1. <Feature/Function Name1>`, `1.2 <Sequence Diagram Name1>`, `2. <Feature/Function Name2>`). Sau khi sửa xong hết toàn bài, bấm chuột phải → *Update Field → Update entire table*.
- [ ] **Điền bảng Change Log (trang 3):** Hiện bảng 13 dòng đang **trống trơn**. Điền tối thiểu 2–3 mốc (Initial draft, Update Database Schema & Architecture, Refactor Class Diagrams per Use Case & Redraw Sequence Diagrams).

---

## Phần I. High Level Design (Trang 4–6)

### I.1. Software Architecture
- [ ] **Thống nhất cổng thanh toán & AI:** Trong bảng đang chỉ ghi `PayOS Gateway` và `Gemini API`, trong khi code BE hiện có cả `utils/payos.js`, `utils/vnpay.js` và fallback `Groq`. Chốt rõ nhóm giữ PayOS hay giữ cả 2, rồi cập nhật đồng bộ từ I.1 → I.3 → II.7 → III.
- [ ] **Kiểm tra hình `Software Architecture`:** Đảm bảo thể hiện rõ mô hình **3 tầng chặt (Strict Layered)**: `Routes/Controllers` → `Services` → `Models` (đã quét code BE: 0/22 controller gọi thẳng Model — rất chuẩn!).

### I.2. Package Diagram
- [ ] **Sửa lỗi copy-paste tiêu đề ở mục `a. Frontend`:** Tiêu đề bảng ngay dưới `a. Frontend` đang ghi nhầm thành **`Backend package descriptions`** → đổi thành `Frontend package descriptions`.
- [ ] **Cập nhật mục `b. Backend`:**
  - Bổ sung package **`jobs`** (code thật có `src/jobs/flagOverdueAppointments.job.js`).
  - Xóa ví dụ AI bịa **`FEFO inventory algorithms`** trong mô tả `services` (vì `Product.js` không có batch/hạn sử dụng, nếu thầy hỏi thuật toán FEFO viết ở file nào là kẹt cứng!).

### I.3. Database Schema (`3.1` → `3.17`)
- [ ] **Bổ sung bảng bị thiếu hoàn toàn — `3.18 ai_assessments`:** Code thật có `AiAssessment.js` và Phần II.2 có hẳn Sequence Diagram UC-14, nhưng I.3 lại quên mất bảng này.
- [ ] **Sửa lại các bảng bị lệch so với Mongoose Model thật:**
  - [ ] **`3.1 users`:** Thêm 5 cột thật đang thiếu (`isEmailVerified`, `emailOtpHash`, `emailOtpExpiresAt`, `failedLoginAttempts`, `lockUntil`).
  - [ ] **`3.2 pets`:** Sửa enum `status` từ `['Active', 'Archived']` → **`['Active', 'Deleted']`**.
  - [ ] **`3.3 staff_schedules` (SỬA NẶNG):** Hiện đang ghi cột `workDate, notes` (kiểu lịch trực cũ). Code thật là mô hình **Blacklist báo bận 3 ca**: đổi thành `staffId`, `date`, `shift` (`['MORNING', 'AFTERNOON', 'EVENING']`), `reason`, `createdBy`.
  - [ ] **`3.4 services`:** Thêm cột `requiredRole` (`['Veterinarian', 'Groomer']`).
  - [ ] **`3.6 appointment_orders`:** Thêm `slotStart`, `slotEnd`, `intake` (embedded object: `symptoms`, `initialCondition`, `recordedBy`, `recordedAt`), `reservedCageId`, `cancelReason`, `cancelledAt`, `cancelledBy`.
  - [ ] **`3.7 appointment_services`:** Thêm `scheduledStart`, `scheduledEnd`, `cancelReason`, `cancelledAt`, `cancelledBy`.
  - [ ] **`3.8 cages`:** Thêm `maxWeightKg`, `zone` (`['Holding', 'InpatientWard']`).
  - [ ] **`3.11 subtask_requests`:** Thêm `serviceId`.
  - [ ] **`3.13 invoices`:** Sửa kiểu `payosOrderCode` từ `String` → **`Number`**, xóa chữ thừa **`(vnp_TxnRef)`** ở cột Notes (lỗi sửa sót), thêm `payosPaymentLinkId` (và `vnpayTxnRef` nếu giữ VNPay).
- [ ] **Vẽ lại/cập nhật 2 hình ERD** đầu mục I.3 cho khớp bảng `ai_assessments` và `staff_schedules` mới.

---

## Phần II. Detailed Code Design (TRỌNG TÂM LỚN NHẤT — LÀM LẠI TOÀN BỘ DIAGRAM)

### Nguyên tắc chung cho toàn bộ `II.1` → `II.9`
1. **Tách Class Diagram theo từng Use Case (VOPC)** thay vì gộp 1 hình to cho cả Feature (đúng góp ý thầy + học ở Phần 3 & Phần 6 File 02).
2. **Sửa sạch tên Class bị AI bịa** (`BookingValidator`, `SlotCalculatorService`, `AppointmentServiceItem`, `ClinicalRecordController`, `ServiceExecutionService`, `InpatientDailyLog`, `GroomingController`, `BillingService`, `PaymentWebhookController`, `TimelineSanitizerService`, `RosterController`, `AnalyticsController`...) về **tên Class thật trong `SEP490_G38-VCMOS_BE`**.
3. **Sửa chuẩn 4 mũi tên Class Diagram** (`Dependency`, `Association`, `Aggregation`, `Composition`) + `Realization` + `Multiplicity`.
4. **Vẽ lại toàn bộ Sequence Diagram (hiện toàn bộ là AI generated)** sao cho **khớp 1–1 tuyệt đối** từng participant và từng tên hàm (operation) với Class Diagram tương ứng.

### Danh sách tuyến tính từng mục trong Phần II
- [ ] **`II.0` (Nên thêm ngay đầu Phần II hoặc cuối I.3):** **Core Domain Entity Class Diagram (`CD-00`)** — khớp Bài 0 Phần 6 File 02.
- [ ] **`II.1` Account & Authentication Management (FE-05):**
  - [ ] **SỬA GẤP LỖI DÁN TRÙNG ẢNH:** Hiện `1.2 Register (UC-02)` và `1.3 Log In (UC-03)` đang dán **cùng 1 file `image27.png`**!
  - [ ] Bổ sung đoạn text mô tả đang bị bỏ trống cho `1.1`, `1.2`, `1.3`.
  - [ ] Tách Class Diagram theo từng UC + vẽ lại `CD-01 / SD-01` (*Bài 1 Phần 6 File 02*).
- [ ] **`II.2` Pet Profile & AI Health Reference Assessment (FE-01, FE-07):**
  - [ ] Bổ sung mô tả text cho `2.1 Class Diagram`.
  - [ ] Tách Class Diagram riêng cho UC-14 + vẽ lại `CD-07 / SD-07` (*Bài 7 Phần 6 File 02*: thêm `AiAssessment` + fallback Groq + `PetSnapshot`).
- [ ] **`II.3` Online & Walk-In Appointment Scheduling Management (FE-02):**
  - [ ] Sửa tên class giả (`SlotCalculatorService`, `AppointmentServiceItem`) về tên thật (`AppointmentOrderController`, `AppointmentOrderService`, `WalkInController`, `WalkInService`, `AppointmentService`, `CageAssignment`).
  - [ ] Tách thành 2 cặp sạch: `CD-02 / SD-02` Book Online (*Bài 2*) và `CD-03 / SD-03` Walk-in / Check-in & Assign Cage (*Bài 3*).
- [ ] **`II.4` Service Execution, Clinical SOAP Examination & Sub-Task Diagnostics (FE-02, FE-03):**
  - [ ] Sửa tên class giả (`ClinicalRecordController`, `ServiceExecutionService`) về tên thật (`MedicalRecordController`, `MedicalRecordService`, `SubtaskRequestController`, `SubtaskRequestService`).
  - [ ] Vẽ lại chuẩn cụm kinh điển `MedicalRecord ◆── Prescription ─── Product` + `CD-04 / SD-04` (*Bài 4 Phần 6 File 02*).
- [ ] **`II.5` Inpatient Care, Daily Rounds & Discharge Management (FE-03):**
  - [ ] Sửa `InpatientController/InpatientService/InpatientDailyLog` → `InpatientRecordController/InpatientRecordService/DailyLog`.
  - [ ] Tách Class Diagram theo UC + vẽ lại `CD-05 / SD-05` (*Bài 5 Phần 6 File 02*).
- [ ] **`II.6` Pet Spa & Grooming Services (FE-01):**
  - [ ] Sửa `GroomingController/GroomingService` về đúng code thật (`AppointmentServiceController` & `AppointmentServiceService` với `createGroomingNote`, `GroomingNote` embedded Composition `0..1` trong `AppointmentService`).
  - [ ] Vẽ lại Class Diagram + Sequence Diagram UC-44 khớp 1–1.
- [ ] **`II.7` Consolidated Billing & Cash / PayOS QR Payment (FE-04):**
  - [ ] Sửa `BillingService`, `PaymentWebhookController` về tên thật (`InvoiceController`, `InvoiceService`, `«interface» IPaymentGateway` + `PayOSGateway`/`VNPayGateway`).
  - [ ] Vẽ lại `CD-06 / SD-06` (*Bài 6 Phần 6 File 02*: chuẩn mũi tên Sync vs Async Webhook IPN + `alt` + `ref`).
- [ ] **`II.8` Pet Care Community Hub & Content Moderation (FE-08):**
  - [ ] Sửa `CommunityController`, `TimelineSanitizerService` về tên thật (`CommunityPostController`, `CommunityInteractionController`, `CommunityModerationController` & 3 Services tương ứng).
  - [ ] Vẽ lại `CD-08 / SD-08` (*Bài 8 Phần 6 File 02*: có Self-Association ở `PostComment.parentCommentId` và Composition `CommunityPost ◆── TimelineItem`).
- [ ] **`II.9` Administration, Staff Shift Roster & Operational Analytics (FE-02, FE-10):**
  - [ ] Sửa `RosterController/RosterService/AnalyticsController/AnalyticsAggregationService` về tên thật (`StaffScheduleController/StaffScheduleService`, `ReportController/ReportService`).
  - [ ] Sửa mô tả & Sequence Diagram `StaffSchedule` theo đúng **mô hình Blacklist 3 ca** (không phải tạo ca làm việc hàng tuần).

---

## Phần III. Design for Non-Functional Requirements (Trang cuối — RÀ LỖI AI CHÉM GIÓ LỆCH CODE)

- [ ] **`III.1.1` Authentication Flow & CORS:**
  - Đối chiếu lại với `auth.controller.js` / `auth.service.js`: hiện SDS đang ghi `rotating refresh tokens`, `POST /api/v1/auth/refresh`, `HttpOnly cookie`, `user.token_version`, `JOB-03`, `JOB-05` — **code BE thật hiện dùng `/api/auth/...` và JWT Bearer đơn giản, không có `refresh` hay `token_version`**. Sửa lại cho đúng thực tế code để thầy mở code soi không bị vặn!
- [ ] **`III.1.2` Authorization & RBAC:**
  - Sửa đường dẫn `/api/v1/**` → `/api/**` cho khớp `BE_api.md` và `server.js`.
- [ ] **`III.1.3` Data Protection:**
  - Sửa lỗi dính chữ ở dòng External API Keys: `GROQ_API_KEY_, / PAYOS_CLIENT_ID, PAYOS_API_KEY, PAYOS_CHECKSUM_KEYCLOUDINARY_API_SECRET`.
  - Sửa `.select('-password_hash')` → `passwordHash` (`select: false` trong Mongoose schema).
- [ ] **`III.2.1` Caching Strategy:**
  - **Sửa lỗi tự vả:** Đoạn trên ghi *không dùng Redis (chỉ dùng `node-cache` + `Socket.IO`)* nhưng tiêu đề cột 2 của bảng ngay bên dưới lại ghi **`Redis Key`**!
  - Kiểm tra lại `package.json` BE xem nhóm có thực sự cài `node-cache` / `socket.io` không — nếu chưa cài thì viết trung thực theo hiện trạng (hoặc mô tả là chiến lược dự kiến/đã áp dụng Mongoose index & pagination).
- [ ] **`III.2.2` Database Query Optimization:**
  - Đổi toàn bộ tên cột từ `snake_case` (`pet_code`, `order_code`, `work_date`, `visit_date`, `derived_status`, `issued_at`, `payment_status`) sang **`camelCase`** cho khớp I.3 và Mongoose thật.
  - Xóa tên collection ảo **`invoice_items`** (vì `items[]` nhúng trong `invoices`).
  - Sửa index `staff_schedules` thành `{ staffId: 1, date: 1, shift: 1 }`.
- [ ] **`III.3.1` External Service Resilience:**
  - Sửa đường dẫn `/src/integrations/` → **`/src/utils/`**.
  - Rà lại danh sách `JOB-01` → `JOB-05` (code thật `src/jobs/` hiện chỉ có `flagOverdueAppointments.job.js` — tránh chém gió mấy job không tồn tại).

---

## Phần IV. Other Design Specifications
- [ ] **`IV.1` State Machine Diagrams (`1.1 Appointment Order`, `1.2 Appointment Service`):** Kiểm tra nhanh tên các state trên 2 hình đảm bảo khớp 100% enum trong code thật (`Booked`, `CheckedIn`, `InProgress`, `WaitOwner`, `Done`, `Cancelled`, `Overdue` cho Order; `Booked`, `Waiting`, `InProgress`, `Done`, `Cancelled`, `Overdue` cho Service).
