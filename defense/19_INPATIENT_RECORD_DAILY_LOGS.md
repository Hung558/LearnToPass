# 19 — Quản Lý Điều Trị Nội Trú & Nhật Ký Hằng Ngày (`/api/inpatient-records`)

> **Tóm tắt nói trong 20 giây:** Quản lý vòng đời nội trú cho Bác sĩ thú y (Nhập viện chiếm chuồng khu `InpatientWard` + tự nhả chuồng chờ `Holding` + gửi email chủ nuôi, Ghi nhật ký hằng ngày theo giờ VN, Xuất viện + gửi email chủ đón) và API riêng cho `PetOwner` xem nhật ký thú cưng mình. **Lưu ý quan trọng:** Xuất viện chỉ đổi `status → Discharged`, giữ chuồng tới khi chủ tới đón/thanh toán.

---

## 1. Mở nhanh code trong VS Code (Code Map)

| Tầng | File | Chức năng |
| :--- | :--- | :--- |
| **Route** | `src/routes/inpatientRecord.routes.js` (`L1-16`) | 6 API cho `Veterinarian` + 1 API `GET /pets/:petId/daily-logs` cho `PetOwner` |
| **Validator** | `src/validators/inpatientRecord.validator.js` (`L1-106`) | Validate ObjectId, `date` lịch thật (`YYYY-MM-DD`), `observations`/`notes` ≤ 2000 |
| **Controller** | `src/controllers/inpatientRecord.controller.js` (`L1-100`) | 7 handler + `handleError` nhận diện `ServiceError` (`err.status`) |
| **Service** | `src/services/inpatientRecord.service.js` (`L1-400`) | Nhập viện + rollback thủ công (`L156-236`), Daily Logs (`L274-360`), Xuất viện nguyên tử (`L368-389`), Gmail thông báo (`L26-142`) |
| **Models (6)** | `InpatientRecord`, `Pet`, `AppointmentOrder`, `Cage`, `CageAssignment`, `User` | Partial Unique `{ petId: 1 }` khi `status: 'Admitted'` (`InpatientRecord.js:28-31`) |
| **Test** | *(Chưa có file test riêng — mới kiểm chứng gián tiếp qua `petTimeline.test.js` & `invoice.test.js`)* | **Điểm cần nhớ nếu hội đồng hỏi độ phủ test** |

---

## 2. Chi tiết 7 API — Luồng code & Toàn bộ Case

### 2.1. `POST /api/inpatient-records` — Bác sĩ cho nhập viện (`Veterinarian`)
- **Body:** `{ petId*, orderId*, cageId*, initialDiagnosis?, treatmentPlan? }` (`attendingVetId` tự lấy từ `req.user.id`).
- **Luồng (`inpatientRecord.service.js:156-236`):**
  1. Kiểm tra song song `Pet` (`Active`, `404`) và `AppointmentOrder` (`404`, đúng `petId` `400`, chưa `Cancelled` `400`) (`L157-169`).
  2. Chống nhập viện trùng: `InpatientRecord.exists({ petId, status: 'Admitted' })` → **`409`** (`L171-172`, và có lớp chốt thứ 2 là Partial Unique Index `11000` ở `L224`).
  3. Kiểm tra chuồng (`L174-179`): phải tồn tại (`404`), bắt buộc thuộc khu **`zone === 'InpatientWard'`** (`400`), cân nặng thú cưng không vượt `cage.maxWeightKg` (`400`).
  4. **Chiếm chuồng nguyên tử (`L182-186`):** `Cage.findOneAndUpdate({ _id: cageId, status: 'Available' }, { status: 'Occupied' })` → trả `null` thì **`409`**.
  5. **Ghi đồng bộ + tự trả chuồng chờ `Holding` + Rollback thủ công (`L188-226`):**
     - Đóng `CageAssignment` khu chờ đang mở của `orderId` (nếu có), tạo `CageAssignment` mới cho chuồng nội trú, tạo `InpatientRecord`, rồi mở lại chuồng chờ cũ về `Available` (`L213-215`).
     - Nếu lỗi giữa chừng → `catch` hoàn tác ngược chuồng mới về `Available`, xóa `assignment` mới, khôi phục `releasedAt: null` cho chuồng chờ cũ (`L218-223`).
  6. Gửi email thông báo nhập viện (chỉ khi chủ có `isEmailVerified === true`, chế độ fire-and-forget không làm lỗi API, `L86-116, 231-233`) → trả **`201`**.

### 2.2. `GET /api/inpatient-records` & `GET /:id` (`Veterinarian`)
- `GET /`: lọc `status`, `petId`, `attendingVetId`, `mine=true`, phân trang; chủ động `.select('-dailyLogs')` cho nhẹ (`L247`) → **`200`**.
- `GET /:id`: populate đủ `petId`, `attendingVetId`, `cageId`, `orderId`, `dailyLogs.recordedById` (`404` nếu không thấy) → **`200`**.

### 2.3. `POST /:id/daily-logs` & `GET /:id/daily-logs` (`Veterinarian`)
- **`POST /:id/daily-logs` (`L274-310`):** Yêu cầu ít nhất `observations` hoặc `notes` (≤2000 ký tự). Hồ sơ phải tồn tại (`404`) và đang **`Admitted`** (`400` nếu đã xuất viện). Ngày `date` (mặc định hôm nay VN) **không được ở tương lai (`> hôm nay`) và không được trước ngày nhập viện (`< admittedAt`)** (`400`, `L281-285`).
  - Dùng `findOneAndUpdate({ _id: recordId, status: 'Admitted' }, { $push: { dailyLogs: { _id: logId, ... } } })` để chặn race condition với lệnh xuất viện (`L288-309`) → trả **`201`** đúng object log vừa thêm.
- **`GET /:id/daily-logs` (`L321-327`):** Lọc `date` hoặc `from`–`to`, sắp mới nhất lên đầu (`L314-319`) → **`200`**.

### 2.4. `GET /api/inpatient-records/pets/:petId/daily-logs` — Chủ nuôi xem nhật ký (`PetOwner`)
- **Luồng (`L333-360`):** Kiểm tra `Pet.findOne({ _id: petId, ownerId, status: 'Active' })` → không phải thú cưng của mình trả **`404`** (giấu luôn sự tồn tại để chống dò IDOR, `L331-335`). Gộp `dailyLogs` qua mọi đợt nội trú của bé (hoặc lọc theo `recordId`), chỉ populate `fullName` của bác sĩ (`L342`), lọc `date`/`from`/`to` & phân trang → **`200`**.

### 2.5. `PATCH /api/inpatient-records/:id/discharge` — Xuất viện (`Veterinarian`)
- **Luồng (`L368-389`):** Cập nhật nguyên tử `findOneAndUpdate({ _id: recordId, status: 'Admitted' }, { status: 'Discharged', dischargedAt: new Date() })` (không thấy → `404`, đã xuất viện trước đó → **`409`**).
- **Không giải phóng chuồng ở đây** vì thú cưng vẫn ở trong lồng chờ chủ đến đón & thanh toán hóa đơn (`L364-365`). Gửi email báo chủ đón (fire-and-forget) → **`200`**.

---

## 3. Bảng tổng hợp Case (Tốt → Fail)

| Endpoint | Case | HTTP | Message / Kết quả | Code ở đâu |
| :--- | :--- | :-: | :--- | :--- |
| `POST /`, `POST /:id/daily-logs` | Tạo thành công | **`201`** | `record` / `log` | `inpatientRecord.controller.js:25, 59` |
| `GET`, `PATCH /:id/discharge` | Thành công | **`200`** | Dữ liệu tương ứng | `controller.js:36,46,71,84,94` |
| Mọi API | Sai role (`PetOwner` gọi route BS hoặc ngược lại) | **`403`** | `'Bạn không có quyền...'` | `inpatientRecord.routes.js:6-14` |
| `POST /` | Sai ID, đơn không thuộc pet, đơn đã `Cancelled`, chuồng không phải `InpatientWard`, quá `maxWeightKg` | **`400`** | Thông báo cụ thể | `service.js:165,168,176,178` |
| `POST /:id/daily-logs` | Hồ sơ đã `Discharged`, ngày tương lai, ngày trước ngày nhập viện, thiếu cả `observations` & `notes` | **`400`** | Thông báo cụ thể | `validator.js:71`, `service.js:278,282,284,308` |
| Mọi API | Không thấy pet (hoặc pet chủ khác ở `listMyPetLogs`) / order / cage / record | **`404`** | Lỗi tương ứng | `service.js:162,163,175,261,276,323,335,345,379` |
| `POST /`, `PATCH /discharge` | Thú cưng đang `Admitted` đợt khác (hoặc `11000`) / chuồng đã `Occupied` / hồ sơ đã xuất viện trước đó | **`409`** | Lỗi xung đột | `service.js:172, 186, 224, 380` |

---

## 4. Câu hỏi hội đồng hay xoáy (Defense Q&A)

1. **Khi thú cưng đang nằm chuồng chờ (`Holding`) rồi bác sĩ cho nhập viện nội trú (`InpatientWard`), chuồng chờ cũ xử lý sao?**
   - **Trả lời:** `inpatientRecord.service.js:193-215` tự động đóng `CageAssignment` của chuồng chờ cũ (`releasedAt = now`) và mở lại `Cage.status = 'Available'` ngay sau khi tạo xong hồ sơ nội trú. Nếu tạo hồ sơ lỗi giữa chừng, khối `catch` (`L218-223`) tự khôi phục lại chuồng chờ cũ.
2. **Tại sao khi bấm Xuất viện (`PATCH /:id/discharge`) hệ thống không tự mở chuồng về `Available` ngay?**
   - **Trả lời:** Theo thực tế phòng khám (`L364-365`), bác sĩ xác nhận "đủ điều kiện xuất viện" xong thì gửi email báo chủ, thú cưng vẫn nằm chờ trong chuồng đó cho tới khi chủ tới quầy thanh toán hóa đơn — lúc đó `invoice.service.js` (hoặc lễ tân giải phóng chuồng) mới trả chuồng về `Available`.
3. **Làm sao chống race condition nếu 1 bác sĩ đang thêm Daily Log cùng lúc 1 bác sĩ khác bấm Xuất viện?**
   - **Trả lời:** Cả `createLog` (`L289-290`) và `discharge` (`L369-370`) đều dùng `findOneAndUpdate` với điều kiện lọc nguyên tử `{ _id, status: 'Admitted' }`. Nếu xuất viện chạy trước, `createLog` nhận `null` và báo lỗi `400`.
