# 17 — Khám Lâm Sàng, Bệnh Án Chuẩn SOAP & Kê Đơn Thuốc (`/api/medical-records`)

> **Tóm tắt nói trong 20 giây:** Toàn bộ quy trình khám của Bác sĩ thú y: Bắt đầu ca khám (kiểm tra thứ tự `BR-12`, tự nhả chuồng chờ `BR-10`, tự đổ triệu chứng tiếp nhận vào mục **S** chỉ đọc), Lưu nháp SOAP + kê đơn thuốc (`BR-16` chỉ thuốc `Active` + snapshot tên/đơn vị) + cập nhật `Pet.weight`, Hoàn tất khám (`BR-13` bắt buộc đủ O-A-P, `MSG26` chặn nếu còn Sub-task chưa xong, `BR-14` giữ `Draft` tới khi thanh toán mới khóa `Finalized`), và xem tiền sử bệnh (`BR-23`).

---

## 1. Mở nhanh code trong VS Code (Code Map)

| Tầng | File | Chức năng |
| :--- | :--- | :--- |
| **Route** | `src/routes/medicalRecord.routes.js` (`L1-45`) | 3 API ghi chỉ `Veterinarian`; 2 API đọc cho 5 roles (check IDOR `BR-23`) |
| **Validator** | `src/validators/medicalRecord.validator.js` (`L1-187`) | Validate O/A/P/notes (≤3000), `weight` (`0 < w ≤ 500`), mảng `prescriptions`, `waitingCageId` |
| **Controller** | `src/controllers/medicalRecord.controller.js` (`L1-153`) | 5 handler chuẩn hóa response lỗi |
| **Service** | `src/services/medicalRecord.service.js` (`L1-541`) | `start`, `saveDraft`, `complete`, `getByServiceId`, `getHistoryByPetId` |
| **Models (7)** | `MedicalRecord`, `AppointmentService`, `AppointmentOrder`, `SubtaskRequest`, `Product`, `Pet`, `CageAssignment` | Unique 1-1 `appointmentServiceId`, index `{ petId: 1, createdAt: -1 }` |
| **Test** | `tests/medicalRecord.test.js` | 527 dòng test đủ SOAP, `BR-12`, `BR-13`, `BR-14`, `BR-16`, `MSG26`, `BR-23` |

---

## 2. Chi tiết 5 API — Luồng code & Toàn bộ Case

### 2.1. `POST /api/medical-records/service/:appointmentServiceId/start` — Bắt đầu ca khám (`Veterinarian`)
- **Luồng (`medicalRecord.service.js:147-214`):**
  1. `loadAndVerifyAssignedService` (`L79-101`): dịch vụ phải tồn tại (`404`) và `assignedStaffId === req.user.id` (khác bác sĩ → **`403`**).
  2. Chặn nếu `status === 'Booked'` (chưa check-in → **`409`**) hoặc đã `Done/Cancelled/Overdue` (**`409`**, `L153-162`).
  3. **`BR-12` (`ensurePreviousServicesFinished`, `L106-124`):** nếu cùng đơn có dịch vụ `sequenceNo` nhỏ hơn mà chưa `Done`/`Cancelled` → **`409`**.
  4. **`BR-10` (`releaseOpenCageForOrder`, `L129-137`):** tự giải phóng chuồng chờ đang mở của đơn về `Available`.
  5. Chuyển `AppointmentService → InProgress` (`dequeueAt: now`) và `AppointmentOrder → InProgress` (`L171-188`).
  6. Tự tạo bản nháp `MedicalRecord` (`status: 'Draft'`) và **tự động copy `order.intake.symptoms || order.note` vào mục `subjectiveNote` (chữ S)** (`L22-28, 191-207`) → trả **`200`**.

### 2.2. `PUT /api/medical-records/service/:appointmentServiceId` — Lưu nháp SOAP & kê đơn (`Veterinarian`)
- **Body:** `{ objectiveNote?, assessment?, plan?, notes?, weight?, prescriptions?: [{ productId*, dosage*, frequency*, instructions?, durationDays* (1–365) }] }` *(không nhận `subjectiveNote` vì chữ S bị khóa chỉ đọc)*.
- **Luồng (`medicalRecord.service.js:222-297`):**
  1. Verify bác sĩ phụ trách (`403`), chặn `Booked`/`Cancelled`/`Overdue` (`409`), check `BR-12` nếu đi thẳng từ `Waiting` (`L228-242`).
  2. **`BR-14` (`L244-247`):** Nếu `MedicalRecord.status === 'Finalized'` → chặn **`409`** `'Bệnh án này đã được khóa (Finalized), không thể chỉnh sửa'`.
  3. **`BR-16` (`buildPrescriptionSnapshots`, `L35-74`):** Đọc batch `Product` theo `$in`, không thấy → **`404`**, `status !== 'Active'` (hết hàng/ngừng bán) → **`400`**; lưu snapshot `productNameSnapshot`, `unitSnapshot`.
  4. Nếu có `weight` → cập nhật thẳng vào `Pet.weight` (`L256-260`). Nếu đang `Waiting` → tự nhả chuồng chờ & chuyển sang `InProgress` (`L263-271`).
  5. Upsert `MedicalRecord` (`status: 'Draft'`, chữ S chỉ điền lúc tạo lần đầu không bao giờ ghi đè, `L279-296`) → trả **`200`**.

### 2.3. `POST /api/medical-records/service/:appointmentServiceId/complete` — Hoàn tất khám (`UC-37`, `BR-13`, `BR-14`, `MSG26`)
- **Body:** Giống lưu nháp + tùy chọn `waitingCageId` (chuồng chờ chủ đón).
- **Luồng (`medicalRecord.service.js:308-467`):**
  1. Verify bác sĩ phụ trách (`403`), chặn `Done` hoặc không phải `Waiting/InProgress` (`409`), check `BR-12` (`409`).
  2. **`MSG26` (`L331-340`):** Kiểm tra `SubtaskRequest.exists({ appointmentServiceId, status: { $in: ['Pending', 'InProgress'] } })` → nếu còn chỉ định cận lâm sàng chưa xong → chặn **`409`**.
  3. Chặn nếu bệnh án đã `Finalized` (`409`, `L343-345`).
  4. **`BR-13` (`L348-378`):** Gộp payload gửi lên với bản nháp đã lưu (chữ S giữ nguyên); nếu thiếu bất kỳ 1 trong 3 mục **O (`objectiveNote`), A (`assessment`), P (`plan`)** → trả **`400`** kèm `errors` từng trường.
  5. Kiểm tra đơn thuốc `BR-16` (`L382-384`), cập nhật `Pet.weight` (`L388-395`), nhả chuồng chờ cũ nếu từ `Waiting` (`L398-400`), và nếu có `waitingCageId` thì gọi `createCageAssignment` (kiểm tra tải trọng `BR-11` & khóa chuồng chờ đón `Occupied`, `L403-411`).
  6. **Chú ý `BR-14` (`L413-431`):** Lưu `MedicalRecord` với **`status: 'Draft'`** (chỉ khi Hóa đơn được thanh toán `Paid` trong `invoice.service.js` thì bệnh án mới tự động khóa thành `Finalized` để bác sĩ còn kịp sửa đơn thuốc nếu cần trước lúc thu tiền!).
  7. Chuyển `AppointmentService → Done` (`L434-446`); đếm số dịch vụ chưa `Done`/`Cancelled` còn lại trong đơn: nếu `=== 0` thì đẩy `AppointmentOrder → WaitOwner`, ngược lại giữ `InProgress` (`L449-459`, `BR-09`) → trả **`200`**.

### 2.4. `GET /api/medical-records/service/:appointmentServiceId` & `GET /pet/:petId` — Xem bệnh án & Tiền sử (`UC-32`, `BR-23`)
- **Quyền:** Cả 5 vai trò (`medicalRecord.routes.js:31-42`).
- **Chống IDOR `BR-23` (`L491-496, L512-514`):** Nếu `PetOwner` truy cập bệnh án/tiền sử thú cưng của chủ khác → **`403`**. Nhân sự phòng khám xem được mọi thú cưng. Không thấy trả **`404`**.

---

## 3. Bảng tổng hợp Case (Tốt → Fail)

| Endpoint | Case | HTTP | Message / Kết quả | Code ở đâu |
| :--- | :--- | :-: | :--- | :--- |
| Cả 5 API | Thành công | **`200`** | `{ appointmentService, medicalRecord, ... }` / `{ pet, records }` | `medicalRecord.controller.js:32,65,98,119,140` |
| 3 API ghi | Role không phải `Veterinarian` / Không phải bác sĩ được gán | **`403`** | `'Bạn không có quyền...'` / `'Bạn không phải là bác sĩ được phân công...'` | `routes.js:12,19,26`, `service.js:92` |
| 2 API đọc | `PetOwner` xem thú cưng của chủ khác (`BR-23`) | **`403`** | `'Bạn không có quyền truy cập/xem lịch sử bệnh án của thú cưng này'` | `service.js:495, 513` |
| `PUT`, `POST /complete` | Validate sai / Thiếu O-A-P (`BR-13`) / Thuốc `Inactive` (`BR-16`) / Chuồng `waitingCageId` sai tải trọng (`BR-11`) | **`400`** | `{ message, errors }` | `validator.js`, `service.js:56, 373` |
| Mọi API | Không thấy dịch vụ / đơn / thuốc / bệnh án / thú cưng | **`404`** | Lỗi tương ứng | `service.js:51,85,97,487,508` |
| `start`, `saveDraft`, `complete` | Chưa Check-in (`Booked`) / Dịch vụ trước chưa xong (`BR-12`) / Còn Sub-task chưa `Completed` (`MSG26`) / Bệnh án đã `Finalized` (`BR-14`) / Ca đã `Done` | **`409`** | Thông báo chi tiết từng quy tắc | `service.js:119,154,158,229,233,246,315,319,336,344` |

---

## 4. Câu hỏi hội đồng hay xoáy (Defense Q&A)

1. **Tại sao mục S (`subjectiveNote`) trong SOAP lại bị khóa chỉ đọc, không cho Bác sĩ sửa?**
   - **Trả lời:** Mục S là lời khai triệu chứng ban đầu của chủ nuôi / ghi nhận tiếp nhận tại quầy (`order.intake.symptoms || order.note`). `medicalRecord.service.js:22-28, 279-281` tự copy nguyên văn vào lúc bắt đầu khám và validator không nhận trường `subjectiveNote` từ client để bảo toàn tính khách quan pháp lý của bệnh án; bác sĩ chỉ ghi nhận O, A, P và `notes`.
2. **Tại sao sau khi Bác sĩ bấm "Hoàn tất ca khám" (`/complete`) mà `MedicalRecord.status` vẫn là `Draft` chứ chưa chuyển ngay sang `Finalized`?**
   - **Trả lời:** Theo đúng `BR-14` (`medicalRecord.service.js:413-425`), giữa lúc bác sĩ khám xong và lúc khách thanh toán tại quầy, bác sĩ vẫn có thể cần bổ sung/điều chỉnh đơn thuốc hoặc ghi chú. Chỉ khi Lễ tân thu tiền thành công (`Invoice.paymentStatus = 'Paid'`), `invoice.service.js` mới tự động chốt `MedicalRecord.status = 'Finalized'` và từ đó `L245-247` khóa cứng không cho sửa nữa.
3. **Nếu Bác sĩ đã chỉ định Xét nghiệm máu (Sub-task) nhưng nhân sự xét nghiệm chưa trả kết quả, Bác sĩ cố bấm hoàn tất khám thì sao?**
   - **Trả lời:** Bị chặn ngay bằng **`409 Conflict`** ở `medicalRecord.service.js:331-340` (`MSG26`) vì còn `SubtaskRequest` đang `Pending` hoặc `InProgress`.
