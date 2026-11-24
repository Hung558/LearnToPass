# 16 — Dịch Vụ Lịch Hẹn, Hàng Đợi Làm Việc & Ghi Chú Grooming (`/api/appointment-services`)

> **Tóm tắt nói trong 20 giây:** Quản lý 11 endpoint vòng đời dịch vụ con (`AppointmentService`): đặt lịch online trọn gói kèm tính khung giờ nối tiếp (`online-booking`), Lễ tân phân công nhân sự (`assign-staff`), hàng đợi làm việc ưu tiên cấp cứu (`my-queue`: `isEmergency DESC → queueNo ASC`), bắt đầu ca (`start` tự nhả chuồng chờ `BR-10`), Groomer hoàn tất (`complete` tự đẩy đơn sang `WaitOwner`), hủy dịch vụ lẻ (`BR-07`), và ghi chú + ảnh Before/After Grooming lên Cloudinary.

---

## 1. Mở nhanh code trong VS Code (Code Map)

| Tầng | File | Chức năng |
| :--- | :--- | :--- |
| **Route** | `src/routes/appointmentService.routes.js` (`L1-56`) | 11 API + `uploadGroomingNotePhotos` trên `POST /:id/grooming-note` (`L47-53`) |
| **Validator** | `src/validators/appointmentService.validator.js` (`L1-635`) | Validate từng endpoint (lưu ý: validator này có truy vấn DB kiểm tra quyền sở hữu, lịch bận, thứ tự `BR-12`, trạng thái) |
| **Controller** | `src/controllers/appointmentService.controller.js` (`L1-231`) | 11 handler + `handleError` đọc `err.statusCode \|\| err.status` (`L31`) |
| **Service** | `src/services/appointmentService.service.js` (`L1-673`) | Đặt lịch trọn gói (`L107-202`), phân công (`L227-270`), hàng đợi (`L283-299`), hủy lẻ (`L325-437`), đổi trạng thái (`L453-525`), Grooming Note (`L562-659`) |
| **Models (5)** | `AppointmentService`, `AppointmentOrder`, `Service`, `CageAssignment`, `User` | Index `{ orderId, sequenceNo }` unique, `{ assignedStaffId, status, enqueueAt }` |
| **Test** | `tests/services/appointmentService.service.test.js` | 1.006 dòng unit test |

---

## 2. Chi tiết 11 API — Luồng code & Toàn bộ Case

### 2.1. `POST /api/appointment-services/online` & `POST /online-booking` — Đặt dịch vụ Online (`PetOwner`)
- **`POST /online` (`service.js:51-89`):** Gắn thêm 1 dịch vụ vào `orderId` đã có của chính mình (`ownerId` check → **`403`** ở `L61-64`); tự tăng `sequenceNo` nếu không truyền; đụng unique `(orderId, sequenceNo)` → **`409`** (`L84-86`). Trả **`201`**.
- **`POST /online-booking` (`service.js:107-202`):** Đặt 1 chạm cả `AppointmentOrder` + mảng `services[]`:
  - Nếu có `slotStart` (`L117-159`): đọc `defaultDuration` của các dịch vụ (mặc định 30 phút), tính nối tiếp `scheduledStart`/`scheduledEnd` và `slotEnd` tổng; vượt `18:00` → **`400`** (`L152-157`).
  - Gọi `createOnlineAppointment` → tạo tuần tự từng `AppointmentService` (không song song để không đụng `sequenceNo`, `L169-188`).
  - **Nếu tạo dịch vụ con lỗi → tự xóa sạch `AppointmentService` + `AppointmentOrder` vừa tạo (`L189-196`)**. Trả **`201`**.

### 2.2. `GET /api/appointment-services?orderId=...` & `PATCH /:id/assign-staff` — Điều phối (`Receptionist` / `Admin`)
- `GET /` (`Receptionist`, `Admin`, `L207-219`): lấy danh sách dịch vụ của 1 đơn sắp theo `sequenceNo: 1`.
- `PATCH /:id/assign-staff` (`Receptionist`, `L227-270`):
  - Validator kiểm tra nhân sự `Active`, đúng `requiredRole`, không báo bận trên `StaffSchedule`.
  - Tính `nextQueueNo = max(queueNo) + 1` của nhân sự (`L234-242`). Nếu đơn đã `CheckedIn` hoặc `InProgress` → tự chuyển dịch vụ sang **`Waiting` + `enqueueAt: now`** (`L251-254`). Trả **`200`**.

### 2.3. `GET /my-queue`, `PATCH /:id/start`, `PATCH /:id/complete` — Hàng đợi & Thực hiện
- **`GET /my-queue?status=...`** (`Veterinarian`, `Groomer`, `L283-299`): lấy ca được giao cho `req.user.id`, sắp **`{ isEmergency: -1, queueNo: 1, enqueueAt: 1 }`** (`L292` — **cấp cứu luôn nhảy lên đầu hàng đợi**).
- **`PATCH /:id/start`** (`Veterinarian`, `Groomer`, `L515-525`):
  - Chuyển `Waiting → InProgress`, ghi `dequeueAt: now`, đẩy đơn cha `CheckedIn → InProgress` (`L474, 487-489`).
  - **Tự động gọi `releaseOpenCageForOrder(orderId)` nhả chuồng chờ về `Available` (`BR-10`, `L522`)**.
- **`PATCH /:id/complete`** (`Groomer`, `L453-510`): chuyển `InProgress → Done`. Đếm số dịch vụ chưa `Done`/`Cancelled` còn lại trong đơn (`L491-494`): nếu `=== 0` thì cập nhật nguyên tử `AppointmentOrder` sang **`WaitOwner`** (`L498-503`, `BR-09`). *(Lưu ý: Bác sĩ hoàn tất khám qua `/api/medical-records/service/:id/complete` chứ không qua route này).*

### 2.4. `PATCH /api/appointment-services/:id/cancel` — Hủy 1 dịch vụ lẻ (`UC-18`, `BR-07`)
- **Quyền:** `PetOwner` hoặc `Receptionist` (`L325-437`).
- **Luồng:** Kiểm tra tồn tại (`404`), `PetOwner` check IDOR (`403`, `L339-341`), đã `Cancelled` (`409`). Áp cùng luật `BR-07` (khách hủy trước Check-in & trước ≥ 2 tiếng; Lễ tân hủy dịch vụ `Booked`/`Waiting` của đơn chưa kết thúc). Sau khi hủy + xóa `queueNo`:
  - Nếu **tất cả dịch vụ trong đơn đều `Cancelled`** → đẩy đơn cha sang `Cancelled` và **tự giải phóng chuồng chờ** (`L413-423`).
  - Nếu các dịch vụ còn lại đều đã `Done` (và đơn đang `CheckedIn`/`InProgress`) → đẩy đơn cha sang **`WaitOwner`** (`L424-431`).

### 2.5. `POST /:id/grooming-note`, `GET /:id/grooming-note`, `GET /my-grooming-notes` (`Groomer`)
- **`POST /:id/grooming-note`** (`uploadGroomingNotePhotos` `beforePhoto`/`afterPhoto` ≤ 5MB + text `beforeNote`, `afterNote`, `recommendations`, `service.js:562-615`):
  1. Upload song song 1–2 ảnh lên Cloudinary `grooming-notes/<id>` bằng `Promise.allSettled` (`L572-590`). Nếu bất kỳ ảnh nào lỗi → **xóa sạch các ảnh vừa tải thành công** và ném **`502`** (`L582-585`).
  2. Cập nhật nguyên tử `AppointmentService.findOneAndUpdate({ _id, assignedStaffId: groomerId, groomingNote: null }, ...)` (`L595-612`). Nếu DB lỗi hoặc đã có ghi chú (trả `null`) → **xóa sạch ảnh vừa tải trên Cloudinary** và ném **`409`**. Trả **`201`**.
- **`GET /:id/grooming-note` (`L623-641`) & `GET /my-grooming-notes` (`L648-659`):** Groomer chỉ xem ghi chú của ca được gán cho chính mình (`404` nếu chưa có).

---

## 3. Bảng tổng hợp Case (Tốt → Fail)

| Endpoint | Case | HTTP | Message / Kết quả | Code ở đâu |
| :--- | :--- | :-: | :--- | :--- |
| `POST /online`, `/online-booking`, `/:id/grooming-note` | Tạo thành công | **`201`** | `{ message, data }` | `appointmentService.controller.js:48, 65, 185` |
| `GET /`, `/my-queue`, `/my-grooming-notes`, `PATCH ...` | Thành công | **`200`** | `{ message, data }` | `controller.js:79,94,112,131,147,162,199,213` |
| Mọi API | Sai role / Thiếu token | **`401`** / **`403`** | `'Chưa đăng nhập'` / `'Bạn không có quyền...'` | `appointmentService.routes.js:21-54` |
| `POST /online-booking` | Mảng dịch vụ rỗng / `slotStart` ngoài `08:00–18:00` / tổng thời lượng vượt `18:00` | **`400`** | Lỗi thời gian / validate | `service.js:109, 119, 123, 153` |
| `POST /online`, `PATCH /:id/cancel` | `PetOwner` thao tác trên đơn của người khác | **`403`** | `'Bạn chỉ có thể tạo/hủy dịch vụ cho lịch hẹn của mình'` | `service.js:63, 340` |
| Các API `/:id/*` | Không thấy dịch vụ / đơn cha / chưa có Grooming Note | **`404`** | Lỗi tương ứng | `service.js:230,329,335,457,463,634,637` |
| `POST /online`, `PATCH cancel/start/complete`, `POST grooming-note` | Trùng `sequenceNo` (`11000`) / vi phạm `BR-07` hủy `<2h` / đơn đã kết thúc / dịch vụ đã có Grooming Note | **`409`** | Chi tiết lỗi xung đột | `service.js:85,345-384,466,611` |
| `POST /:id/grooming-note` | Lỗi upload Cloudinary (đã tự xóa ảnh còn lại) | **`502`** | `'Không thể tải ảnh lên, vui lòng thử lại sau'` | `service.js:582-585` |

---

## 4. Câu hỏi hội đồng hay xoáy (Defense Q&A)

1. **Hàng đợi `my-queue` sắp xếp ưu tiên như thế nào nếu có ca cấp cứu xen vào?**
   - **Trả lời:** `appointmentService.service.js:292` sort theo `{ isEmergency: -1, queueNo: 1, enqueueAt: 1 }` nên mọi ca có `isEmergency: true` luôn được đẩy lên đầu danh sách trước các ca thường.
2. **Tại sao `createOnlineBooking` phải dùng vòng lặp `for` tạo từng dịch vụ tuần tự thay vì `Promise.all`?**
   - **Trả lời:** Vì mỗi dịch vụ tự tính `sequenceNo = last + 1` (`L68-73`). Nếu chạy `Promise.all` song song, cả 2 lời gọi sẽ đọc cùng 1 `last` và đụng Unique Index `{ orderId: 1, sequenceNo: 1 }` (`11000`). Nếu tạo dịch vụ thứ 2 lỗi, khối `catch` (`L190-195`) tự xóa sạch đơn & dịch vụ đã tạo.
3. **Nếu 2 request Groomer cùng bấm tạo Grooming Note 1 lúc thì sao? Ảnh Cloudinary có bị rác không?**
   - **Trả lời:** `L595-612` dùng điều kiện nguyên tử `{ _id, assignedStaffId, groomingNote: null }`, nên chỉ 1 request ghi thành công; request thứ 2 nhận `null`, tự gọi `deleteCloudinaryImages(uploadedPublicIds)` xóa ảnh vừa upload rồi trả `409`.
