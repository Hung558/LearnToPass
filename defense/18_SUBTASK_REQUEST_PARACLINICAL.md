# 18 — Chỉ Định Cận Lâm Sàng & Dịch Vụ Phụ (`/api/subtask-requests` — `UC-33`, `UC-34`, `MSG26`)

> **Tóm tắt nói trong 20 giây:** Khi đang khám chính (`InProgress`), Bác sĩ có thể chỉ định thêm xét nghiệm/siêu âm/cắt móng (`SubtaskRequest`) cho đồng nghiệp (`Veterinarian`/`Groomer`), có kiểm tra `requiredRole` & lịch báo bận `StaffSchedule`. Nhân sự nhận ca (`Pending → InProgress`), trả kết quả chữ/ảnh Cloudinary (`Completed`, có dọn ảnh nếu DB lỗi). Chốt chặn **`MSG26`** ngăn Bác sĩ chính hoàn tất bệnh án khi còn Sub-task chưa xong, nhưng cho phép xóa Sub-task khi còn `Pending`.

---

## 1. Mở nhanh code trong VS Code (Code Map)

| Tầng | File | Chức năng |
| :--- | :--- | :--- |
| **Route** | `src/routes/subtaskRequest.routes.js` (`L1-61`) | `protect` toàn bộ; 7 API + `uploadSubtaskResultPhoto` (`resultImage`, ≤ 5MB) ở `L49` |
| **Validator** | `src/validators/subtaskRequest.validator.js` (`L1-216`) | Chặn trường lạ (Mass Assignment `L50-53, 167-172`), yêu cầu ít nhất `resultText` hoặc `resultImage` |
| **Controller** | `src/controllers/subtaskRequest.controller.js` (`L1-210`) | 7 handler chuẩn hóa response |
| **Service** | `src/services/subtaskRequest.service.js` (`L1-374`) | Tạo chỉ định, hàng đợi `my-queue`, xem theo ca (`BR-23`), atomic `start`/`complete`/`delete` |
| **Models (6)** | `SubtaskRequest`, `AppointmentService`, `AppointmentOrder`, `MedicalRecord`, `Service`, `User` | Index `{ assignedStaffId, status }` & `{ appointmentServiceId }` |
| **Test** | `tests/subtaskRequest.test.js` | 520 dòng test đủ 7 API, `MSG26`, `BR-23`, upload ảnh |

---

## 2. Chi tiết 7 API — Luồng code & Toàn bộ Case

### 2.1. `POST /api/subtask-requests` — Bác sĩ chính tạo chỉ định (`UC-33`, `Veterinarian`)
- **Body (chỉ nhận đúng 5 field — gửi thừa trường bị `400`):** `{ appointmentServiceId*, serviceId?, assignedStaffId*, taskType* (≤200), instructions? (≤2000) }`.
- **Luồng (`subtaskRequest.service.js:72-170`):**
  1. Ca khám chính phải tồn tại (`404`), đúng bác sĩ được gán (`403`, `L83-88`), đang **`InProgress`** (`409`, `L90-95`) và bệnh án chưa `Finalized` (`409`, `L97-102`).
  2. Nhân sự được giao phải tồn tại (`404`), role `Veterinarian|Groomer` (`400`), `Active` (`400`).
  3. Nếu có `serviceId`: dịch vụ phải tồn tại (`404`), `Active` (`400`), nhân sự khớp `requiredRole` (`400`, `L118-135`).
  4. Kiểm tra lịch báo bận `findStaffBusyBlock` trên `StaffSchedule` → đang bận trả **`409`** (`L138-156`).
  5. Tạo `SubtaskRequest` (`status: 'Pending'`) → trả **`201`**.

### 2.2. `GET /my-queue`, `GET /service/:appointmentServiceId`, `GET /:id` — Tra cứu
- `GET /my-queue?status=` (`Veterinarian`, `Groomer`, `L175-185`): hàng đợi `Pending`/`InProgress` xếp **FIFO** (`createdAt: 1`), lịch sử `Completed` xếp **mới nhất trước** (`completedAt: -1`).
- `GET /service/:appointmentServiceId` & `GET /:id` (5 roles, `L190-233`): chống IDOR `BR-23` — `PetOwner` chỉ xem được Sub-task của thú cưng mình (`403` nếu khác chủ).

### 2.3. `PATCH /:id/start` & `PATCH /:id/complete` — Nhận ca & Trả kết quả (`UC-34`, `Vet`/`Groomer`)
- **`PATCH /:id/start` (`L238-267`):** Đúng nhân sự được giao (`403`), phải đang `Pending` (`409`) → cập nhật nguyên tử `findOneAndUpdate({ _id, assignedStaffId, status: 'Pending' }, { status: 'InProgress' })` → **`200`**.
- **`PATCH /:id/complete` (`L273-330`):** Middleware `uploadSubtaskResultPhoto` (nhận `resultImage` ≤ 5MB hoặc JSON chỉ có `resultText`). Validator chặn truyền URL ảnh tự gõ (`COMPLETE_ALLOWED_FIELDS = ['resultText']`, `L9, 167-172`) và bắt buộc có ít nhất `resultText` hoặc `resultImage`.
  - Đúng nhân sự (`403`), chưa `Completed` (`409`, cho phép hoàn tất từ `Pending` hoặc `InProgress`).
  - Upload ảnh lên Cloudinary `subtask-results/<id>` (lỗi → **`502`**).
  - Cập nhật nguyên tử `findOneAndUpdate({ _id, assignedStaffId, status: { $in: ['Pending', 'InProgress'] } }, { status: 'Completed', completedAt, resultText?, resultImageUrl? })`. Nếu ghi DB lỗi hoặc đã bị hoàn tất trước đó → **gọi `deleteCloudinaryImage` dọn sạch ảnh mồ côi (`L320, 325`)** → **`200`**.

### 2.4. `DELETE /api/subtask-requests/:id` — Hủy Sub-task còn `Pending` (`Veterinarian`)
- **Luồng (`L335-363`):** Chỉ đúng bác sĩ đã tạo (`requestedById === req.user.id`, `403`) và còn `Pending` (`409` nếu đã `InProgress`/`Completed`) mới được xóa nguyên tử (`findOneAndDelete`) — giúp tránh kẹt chốt chặn `MSG26` nếu chỉ định nhầm → **`200`**.

---

## 3. Bảng tổng hợp Case (Tốt → Fail)

| Endpoint | Case | HTTP | Message / Kết quả | Code ở đâu |
| :--- | :--- | :-: | :--- | :--- |
| `POST /` | Tạo thành công | **`201`** | `{ message, data }` | `subtaskRequest.controller.js:34` |
| 6 API còn lại | Thành công | **`200`** | `{ message, data? }` | `controller.js:58,85,109,133,169,193` |
| `POST /`, `PATCH complete` | Truyền trường lạ (`status`/`resultImageUrl`...) / thiếu `taskType` / cả `resultText` & `resultImage` đều trống / nhân sự hoặc dịch vụ sai role/không Active | **`400`** | `{ message, errors }` | `validator.js:50-98,167-194`, `service.js:109,115,126,130` |
| `POST`, `GET`, `PATCH`, `DELETE` | Không phải BS phụ trách / không phải nhân sự được giao / không phải người tạo / `PetOwner` xem pet khác (`BR-23`) | **`403`** | Chi tiết quyền bị chặn | `service.js:87,205,228,245,280,342` |
| Các API `/:id`, `/service/:id` | Không thấy ca khám / nhân sự / dịch vụ / subtask | **`404`** | Lỗi tương ứng | `service.js:80,106,123,196,221,241,276,338` |
| `POST`, `start`, `complete`, `DELETE` | Ca khám không `InProgress` / bệnh án `Finalized` / nhân sự báo bận / sai trạng thái / hoàn tất đúp | **`409`** | Chi tiết xung đột | `service.js:91,101,151,249,263,284,326,346,359` |
| `PATCH /:id/complete` | Lỗi upload Cloudinary | **`502`** | `'Không thể tải ảnh kết quả lên, vui lòng thử lại sau'` | `service.js:297` |

---

## 4. Câu hỏi hội đồng hay xoáy (Defense Q&A)

1. **Tại sao `subtaskRequest.validator.js:166-172` lại từ chối nếu client gửi `resultImageUrl` trong JSON body?**
   - **Trả lời:** Để bắt buộc ảnh kết quả X-quang/siêu âm phải đi qua `multer` upload lên Cloudinary do BE kiểm soát, chống chèn link độc hại hoặc giả mạo URL bên ngoài.
2. **Nếu Bác sĩ lỡ chỉ định nhầm Sub-task cho đồng nghiệp thì có bị kẹt mãi không hoàn tất được bệnh án (`MSG26`) không?**
   - **Trả lời:** Không. Nếu đồng nghiệp chưa bấm nhận (`status === 'Pending'`), chính bác sĩ tạo có quyền gọi `DELETE /api/subtask-requests/:id` (`L335-363`) để hủy chỉ định đó.
3. **Sub-task có thể dùng dịch vụ có tính phí (`serviceId`) để cộng vào hóa đơn không?**
   - **Trả lời:** Có. Khi `SubtaskRequest` có `serviceId` và đã `Completed`, `invoice.service.js` tự động đọc và gom vào dòng `Service` của hóa đơn tổng hợp.
