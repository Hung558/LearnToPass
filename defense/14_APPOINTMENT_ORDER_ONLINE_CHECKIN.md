# 14 — Đặt Lịch Online, Check-in Quầy, Hủy Lịch & Quét Quá Hạn (`/api/appointment-orders`)

> **Tóm tắt nói trong 20 giây:** Quản lý vòng đời đơn hẹn từ lúc `PetOwner` đặt online (tự giữ chỗ chuồng dự kiến theo ngày + BCC email báo Lễ tân), Lễ tân Check-in 1 chạm trong `runWithTransaction` (ghi phiếu `intake`, phân công BS/Groomer + cấp `queueNo → Waiting`, gán chuồng `Available → Occupied`), đến Hủy lịch (`BR-07`: khách tự hủy trước ≥ 2 tiếng, Lễ tân hủy trước khi có dịch vụ chạy + tự giải phóng chuồng) và Admin kích hoạt tay `JOB-01`.

---

## 1. Mở nhanh code trong VS Code (Code Map)

| Tầng | File | Chức năng |
| :--- | :--- | :--- |
| **Route** | `src/routes/appointmentOrder.routes.js` (`L1-31`) | Mount lồng `/walk-in` (`L9`), `protect` (`L12`), 7 endpoint chính |
| **Validator** | `src/validators/appointmentOrder.validator.js` (`L1-439`) | Giờ mở cửa `08:00–18:00`, ngày lịch thật VN, `intake`, `cancelReason ≤ 200`, `serviceAssignments` |
| **Controller** | `src/controllers/appointmentOrder.controller.js` (`L1-152`) | 7 handler (gồm `triggerFlagOverdue` gọi `JOB-01` ở `L121-142`) |
| **Service** | `src/services/appointmentOrder.service.js` (`L1-1078`) | Giữ chỗ chuồng theo ngày (`L156-221`), enrich batch `$in` (`L242-346`), Check-in Transaction (`L684-947`), Hủy lịch (`L953-1069`) |
| **Models (6)** | `AppointmentOrder`, `AppointmentService`, `Pet`, `User`, `Cage`, `CageAssignment` | Compound index `{ petId, date }`, `{ date, derivedStatus }`, `{ date, reservedCageId }` |
| **Test** | `tests/appointmentOrder.test.js`, `tests/services/appointmentOrder.service.test.js`, `tests/flagOverdueJob.test.js` | Phủ đủ đặt lịch, check-in, hủy 2h, race condition |

---

## 2. Chi tiết 7 API — Luồng code & Toàn bộ Case

### 2.1. `POST /api/appointment-orders/online` — `PetOwner` đặt lịch hẹn Online
- **Quyền:** `PetOwner` — `appointmentOrder.routes.js:15`.
- **Body:** `{ petId*, date* (YYYY-MM-DD >= hôm nay), slotStart? (08:00–17:59, nếu hôm nay phải > giờ hiện tại VN), slotEnd? (<=18:00, > slotStart), note? (≤500) }`.
- **Luồng (`appointmentOrder.service.js:352-413`):**
  1. Kiểm tra `Pet` thuộc `ownerId` (`404`) và chưa `Deleted` (`400`) (`L354-360`).
  2. Chống đặt trùng 1 thú cưng 2 đơn chưa hủy trong cùng ngày (`409`, `L363-370`).
  3. **Tự động giữ chỗ chuồng dự kiến (`findAvailableReservedCageId`, `L156-221`):** tìm chuồng `isIsolated: false`, chưa bị giữ cho ngày `date` (nếu là hôm nay loại thêm chuồng đang `CageAssignment` mở), lọc tải trọng `BR-11`, ưu tiên khu `Holding` + cỡ vừa nhỏ nhất → lưu vào `reservedCageId` (**không đổi `Cage.status` sang `Occupied` để tránh khóa chuồng của ngày khác**).
  4. Sinh mã `APT-XXXXXX` (retry 5 lần, hết 5 lần `500`), tạo `AppointmentOrder` (`isOnline: true`, `derivedStatus: 'Booked'`), gửi email BCC cho toàn bộ Lễ tân `Active` đã verify email (`L123-150`, fire-and-forget không làm hỏng đơn) → trả **`201`**.

### 2.2. `GET /api/appointment-orders/me`, `GET /me/:id`, `GET /` — Xem danh sách & chi tiết
- `GET /me` (`PetOwner`, `L418-439`): lọc `status`, `from`, `to`, phân trang → gọi `enrichOrdersWithCageAndServices` (`L242-346`, gom batch `$in` lấy chuồng + dịch vụ + `currentService` tránh N+1).
- `GET /me/:id` (`PetOwner`, `L441-494`): lọc `{ _id: id, ownerId, isOnline: true }` (chống IDOR → `404`), trả kèm `services`, `cage`, `currentService`.
- `GET /` (`Admin`, `Receptionist`, `L500-544`): xem cả Online & Walk-in, lọc `status`, `isOnline`, `ownerId`, `petId`, `code` (`APT-XXXXXX`), `from`, `to`, phân trang.

### 2.3. `PATCH /api/appointment-orders/:id/check-in` — Lễ tân Check-in (`UC-22`)
- **Quyền:** `Receptionist` — `appointmentOrder.routes.js:26`.
- **Body (tùy chọn tất cả):** `{ cageId?, requiresIsolation?, intake?: { symptoms*, initialCondition* }, note?, assignedStaffId? XOR serviceAssignments?: [{ appointmentServiceId*, assignedStaffId* }] }`.
- **Luồng 2 pha + Transaction (`appointmentOrder.service.js:684-947`):**
  1. **Kiểm tra trước khi ghi (`L693-746`):** đơn phải tồn tại (`404`) và đang `Booked` (`409`). Nếu có `cageId` hoặc đơn có `reservedCageId` từ lúc đặt online → `assertCageReadyForCheckIn` (`L560-582`: tồn tại `404`, `Available` `409`, đạt `BR-11` `400`). Nếu có phân công nhân sự → `prepareServiceStaffAssignments` (`L588-676`: dịch vụ thuộc đơn `404`, chưa `InProgress/Done/Cancelled/Overdue` `409`, nhân sự tồn tại `404`, là `Vet/Groomer` & `Active` `400`, khớp `requiredRole` `400`, **không báo bận ca đó trên `StaffSchedule`** `409`).
  2. **Ghi đồng bộ trong `runWithTransaction(runCheckInWrite)` (`L767-935`):**
     - Cập nhật nguyên tử `AppointmentOrder.findOneAndUpdate({ _id: id, derivedStatus: 'Booked' }, { derivedStatus: 'CheckedIn', checkedInAt, intake?, note? })` (chống 2 lễ tân check-in trùng → `409`, `L771-790`).
     - Gán `assignedStaffId`, tính `queueNo = max + 1`, đẩy `AppointmentService` sang `Waiting` + `enqueueAt` (`L830-908`).
     - Gọi `createCageAssignment` khóa chuồng thực tế `Available → Occupied` (`L911-924`).
     - **Standalone Rollback (`L794-823, 928-931`):** nếu lỗi khi `!session`, hoàn tác đơn về `Booked`, khôi phục snapshot từng `AppointmentService`.
  3. Enrich và trả **`200`** `{ message: 'Check-in thành công', data }`.

### 2.4. `PATCH /api/appointment-orders/:id/cancel` — Hủy toàn bộ đơn hẹn (`UC-18`, `BR-07`)
- **Quyền:** `PetOwner` hoặc `Receptionist` — `appointmentOrder.routes.js:29`.
- **Body:** `{ reason* (hoặc cancelReason*, 1–200 ký tự — MSG22) }`.
- **Luồng (`appointmentOrder.service.js:953-1069`):**
  1. Tìm đơn (`404`). Nếu `PetOwner` không phải `ownerId` → **`403`** (`L963-965`). Đã `Cancelled` → **`409`** (`L968-970`).
  2. **Nhánh `PetOwner` (`L972-995`):** chỉ được hủy khi `derivedStatus === 'Booked'` (`409` nếu đã Check-in); nếu có `slotStart` phải cách giờ hẹn **≥ 2 tiếng** (`CANCELLATION_WINDOW_MS`, `L982-989`), nếu không có `slotStart` thì `date >= todayVN()` (`L990-995`) — vi phạm trả **`409`**.
  3. **Nhánh `Receptionist` (`L996-1017`):** được hủy khi đơn là `Booked` hoặc `CheckedIn` nhưng **chưa có dịch vụ nào `InProgress` hoặc `Done`** (`409` nếu đã có).
  4. Cập nhật nguyên tử `AppointmentOrder` sang `Cancelled` (`L1021-1036`, đổi trạng thái giữa chừng → `409`).
  5. Hủy hàng loạt dịch vụ `Booked`/`Waiting` + xóa `queueNo`/`enqueueAt` (`L1039-1053`), và **tự động gọi `releaseCageAssignment` giải phóng chuồng về `Available`** nếu đơn đã gán chuồng (`L1056-1066`).

### 2.5. `POST /api/appointment-orders/flag-overdue` — Admin chạy tay `JOB-01`
- **Quyền:** `Admin` (`appointmentOrder.routes.js:23`) — validate `date` (`400`) → gọi `flagOverdueAppointments` → trả **`200`** (chi tiết xem `03_INTEGRATIONS_AND_BACKGROUND_JOBS.md`).

---

## 3. Bảng tổng hợp Case (Tốt → Fail)

| Endpoint | Case | HTTP | Message / Kết quả | Code ở đâu |
| :--- | :--- | :-: | :--- | :--- |
| `POST /online` | Đặt thành công (tự giữ chuồng dự kiến + email BCC lễ tân) | **`201`** | `{ message: 'Đặt lịch thành công', data }` | `appointmentOrder.controller.js:34` |
| `POST /online` | Sai format / ngày quá khứ / ngoài `08:00–18:00` / pet đã `Deleted` | **`400`** | `{ message, errors }` / `'Không thể đặt lịch cho thú cưng đã bị xóa...'` | `appointmentOrder.validator.js:101-166`, `service.js:359` |
| `POST /online` | Pet không thuộc chủ / Trùng lịch trong ngày | **`404`** / **`409`** | `'Không tìm thấy thú cưng'` / `'Thú cưng này đã có lịch hẹn trong ngày đã chọn'` | `appointmentOrder.service.js:356, 369` |
| `PATCH /:id/check-in` | Check-in thành công (đơn + dịch vụ + chuồng) | **`200`** | `{ message: 'Check-in thành công', data }` | `appointmentOrder.controller.js:94` |
| `PATCH /:id/check-in` | Sai `BR-11` / nhân sự không Active hoặc sai `requiredRole` / gửi cả `assignedStaffId` & `serviceAssignments` | **`400`** | Chi tiết lỗi | `validator.js:290-429`, `service.js:579,640,643,651` |
| `PATCH /:id/check-in` | Không thấy đơn / dịch vụ / nhân sự / chuồng | **`404`** | Lỗi tương ứng | `service.js:564,605,637,698,784` |
| `PATCH /:id/check-in` | Đơn không phải `Booked` / chuồng `Occupied` / nhân sự báo bận ca đó / Check-in trùng | **`409`** | Lỗi tương ứng | `service.js:567,596,608,667,701,786` |
| `PATCH /:id/cancel` | Hủy thành công (kèm hủy dịch vụ & giải phóng chuồng) | **`200`** | `{ message: 'Hủy lịch hẹn thành công', data }` | `appointmentOrder.controller.js:114` |
| `PATCH /:id/cancel` | Thiếu `reason` hoặc `> 200` ký tự | **`400`** | `'Vui lòng nhập lý do hủy'` | `appointmentOrder.validator.js:223-245` |
| `PATCH /:id/cancel` | `PetOwner` hủy đơn người khác | **`403`** | `'Bạn chỉ có thể hủy lịch hẹn của chính mình'` | `appointmentOrder.service.js:964` |
| `PATCH /:id/cancel` | Đã hủy rồi / `PetOwner` hủy sau Check-in hoặc `< 2h` / Lễ tân hủy khi đã có dịch vụ `InProgress`/`Done` | **`409`** | Thông báo chi tiết theo `BR-07` | `appointmentOrder.service.js:969,976,986,992,999,1012,1035` |

---

## 4. Câu hỏi hội đồng hay xoáy (Defense Q&A)

1. **Khi khách đặt lịch online cho tuần sau, hệ thống có khóa `Cage.status = 'Occupied'` ngay không?**
   - **Trả lời:** Không! Nếu đổi `Cage.status = 'Occupied'` cho tuần sau thì hôm nay chuồng trống cũng không dùng được. Do đó `findAvailableReservedCageId` (`L156-221`) chỉ lưu `reservedCageId` gắn theo ngày `date` trên `AppointmentOrder`; đến đúng hôm khách tới quầy Check-in (`L911-924`) mới khóa chuồng vật lý sang `Occupied` bằng `CageAssignment`.
2. **Làm sao chống 2 Lễ tân cùng bấm Check-in 1 đơn tại cùng thời điểm?**
   - **Trả lời:** `L771-775` cập nhật nguyên tử `AppointmentOrder.findOneAndUpdate({ _id: id, derivedStatus: 'Booked' }, { derivedStatus: 'CheckedIn' })` bên trong `runWithTransaction`; người thứ 2 nhận `null` và lập tức bị đẩy ra `409 Conflict`.
3. **Nếu Lễ tân hủy đơn đã Check-in (chưa có dịch vụ chạy) nhưng trước đó đã gán chuồng thì chuồng có bị kẹt không?**
   - **Trả lời:** Không. Bước 6 của `cancelAppointment` (`L1056-1066`) tự tìm `CageAssignment` đang mở (`releasedAt: null`) của đơn và gọi `releaseCageAssignment` trả chuồng về `Available`.
