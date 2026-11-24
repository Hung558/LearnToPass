# 13 — Lịch Báo Bận Nhân Sự Theo Cơ Chế Blacklist (`/api/staff-schedules`)

> **Tóm tắt nói trong 20 giây:** Hoạt động theo mô hình **Blacklist** (`Ngày × 3 Ca: MORNING | AFTERNOON | EVENING`): nhân sự mặc định rảnh, chỉ tạo bản ghi khi báo bận. Khi báo bận, hệ thống tự quét và trả kèm **`warning` + `conflictingServices`** nếu nhân sự đó đang có ca `Waiting/InProgress` trùng ca/ngày.

---

## 1. Mở nhanh code trong VS Code (Code Map)

| Tầng | File | Chức năng |
| :--- | :--- | :--- |
| **Route** | `src/routes/staffSchedule.routes.js` (`L1-22`) | `protect` + `authorize('Veterinarian', 'Groomer', 'Receptionist', 'Admin')` cho 4 API |
| **Validator** | `src/validators/staffSchedule.validator.js` (`L1-152`) | Kiểm tra ngày lịch thật (`isRealDate`), 3 ca, `reason ≤ 500`, `from ≤ to` |
| **Controller** | `src/controllers/staffSchedule.controller.js` (`L1-80`) | Điều phối 4 endpoint (`GET /me`, `GET /`, `POST /`, `DELETE /:id`) |
| **Service** | `src/services/staffSchedule.service.js` (`L1-272`) | CRUD + 3 helper dùng chung: `getShiftFromTime`, `findBusyStaffIds`, `findStaffBusyBlock` |
| **Models** | `StaffSchedule.js`, `User.js`, `AppointmentOrder.js`, `AppointmentService.js` | Unique `{ staffId: 1, date: 1, shift: 1 }`; query `{ date: 1, shift: 1 }` |
| **Test** | `tests/staffSchedule.test.js` | Test báo bận, báo hộ, cảnh báo trùng ca, hủy báo bận |

---

## 2. Quy tắc 3 Ca & Chi tiết 4 API

- **3 Ca làm việc (`staffSchedule.service.js:10-14, 36-42`):** `MORNING` (`< 12:00`), `AFTERNOON` (`12:00 – 16:59`), `EVENING` (`≥ 17:00`).

### 2.1. `POST /api/staff-schedules` — Đăng ký báo bận
- **Body:** `{ staffId? (bắt buộc với Recep/Admin, tự lấy req.user.id với Vet/Groomer), date* (YYYY-MM-DD), shift* ('MORNING'|'AFTERNOON'|'EVENING'), reason? (≤500) }`.
- **Luồng (`staffSchedule.service.js:91-193`):**
  1. **Phân quyền tự báo vs báo hộ (`L92-100`):** `Veterinarian`/`Groomer` truyền `staffId` của người khác → **`403`** `'Bạn chỉ có thể tự báo bận cho chính mình'`.
  2. Chặn báo bận cho ngày trong quá khứ (`date < todayVN()`, `L102-104`) → **`400`**.
  3. Kiểm tra nhân sự tồn tại (`404`), phải có role `Veterinarian` hoặc `Groomer` (`400`), đang `Active` (`400`) (`L106-115`).
  4. `StaffSchedule.create` → trùng `(staffId, date, shift)` `11000` → **`409`** (`L127-132`).
  5. **Quét xung đột ca đang chạy (`L137-185`):** tìm các `AppointmentService` đang `Waiting`/`InProgress` của nhân sự này trong ngày/ca đó; nếu có, trả kèm `warning` và mảng `conflictingServices` để Lễ tân biết điều phối lại (vẫn cho tạo thành công **`201`**).

### 2.2. `GET /api/staff-schedules/me` & `GET /api/staff-schedules` — Xem danh sách báo bận
- **Query:** `staffId?`, `date?`, `from?`, `to?`, `shift?`.
- **Phân quyền (`staffSchedule.service.js:200-209`):** `Veterinarian`/`Groomer` truyền `staffId` của người khác → **`403`**; mặc định chỉ thấy lịch của mình. `Receptionist`/`Admin` xem được tất cả và lọc tự do.

### 2.3. `DELETE /api/staff-schedules/:id` — Hủy lịch báo bận (xóa khỏi Blacklist)
- **Luồng (`staffSchedule.service.js:238-258`):** Check ObjectId (`400`) → tìm bản ghi (`404`) → nếu `Vet`/`Groomer` xóa lịch của người khác → **`403`** → `deleteOne` xóa hẳn để nhân sự khả dụng trở lại → **`200`**.

---

## 3. Bảng tổng hợp Case (Tốt → Fail)

| Endpoint | Case | HTTP | Message / Kết quả | Code ở đâu |
| :--- | :--- | :-: | :--- | :--- |
| Mọi API | Thiếu token / `PetOwner` gọi | **`401`** / **`403`** | `'Chưa đăng nhập'` / `'Bạn không có quyền...'` | `staffSchedule.routes.js:7` |
| `POST /` | Tạo thành công (kèm `warning` nếu trùng ca đang chạy) | **`201`** | `{ message, data, warning, conflictingServices }` | `staffSchedule.controller.js:19`, `staffSchedule.service.js:187-192` |
| `POST /` | Ngày quá khứ / Ngày ảo / Thiếu `staffId` (Recep/Admin) / Nhân sự không phải Vet-Groomer hoặc không `Active` | **`400`** | Lỗi cụ thể theo trường | `staffSchedule.validator.js:36-76`, `staffSchedule.service.js:103,111,114` |
| `POST /`, `GET /`, `DELETE /:id` | Vet/Groomer thao tác trên lịch của người khác | **`403`** | `'Bạn chỉ có thể tự báo bận/xem/hủy lịch báo bận của chính mình'` | `staffSchedule.service.js:97, 204, 250` |
| `POST /`, `DELETE /:id` | Không thấy nhân sự / Không thấy bản ghi báo bận | **`404`** | `'Không tìm thấy nhân sự'` / `'Không tìm thấy bản ghi báo bận'` | `staffSchedule.service.js:108, 245` |
| `POST /` | Trùng `(staffId, date, shift)` (`11000`) | **`409`** | `'Nhân sự "..." đã có lịch báo bận ...'` | `staffSchedule.service.js:127-132` |

---

## 4. Câu hỏi hội đồng hay xoáy (Defense Q&A)

1. **Tại sao dùng cơ chế Blacklist (chỉ lưu khi bận) thay vì Whitelist (Admin tạo lịch trực cho từng nhân viên mỗi tuần)?**
   - **Trả lời:** Giảm 90% thao tác nhập liệu lặp lại cho phòng khám nhỏ/vừa: mặc định nhân sự `Active` đều đi làm; khi nghỉ phép đột xuất chỉ cần thêm 1 dòng vào `staff_schedules`. Khi chỉ lọc theo `date` không có `shift`, `findBusyStaffIds` (`L57-74`) chỉ ẩn những người báo bận đủ cả 3 ca trong ngày đó.
2. **Nếu Bác sĩ đang có ca `Waiting` ở hàng đợi mà vẫn bấm báo bận thì hệ thống có cấm không?**
   - **Trả lời:** Không cấm (vì thực tế bác sĩ có thể bị ốm/việc gấp đột xuất), nhưng `staffSchedule.service.js:137-185` lập tức quét và trả `warning` + danh sách `conflictingServices` để Lễ tân biết ngay những ca cần điều phối sang bác sĩ khác.
