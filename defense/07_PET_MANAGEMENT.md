# 07 — Quản Lý Hồ Sơ Thú Cưng (`/api/pets` — CRUD)

> **Tóm tắt nói trong 20 giây:** Cho phép chủ nuôi (`PetOwner`) tạo/xem/sửa/xóa mềm hồ sơ thú cưng kèm ảnh Cloudinary (kiểm tra Magic Bytes chống giả mạo định dạng + tự dọn ảnh cũ/ảnh mồ côi). Nhân sự phòng khám xem được hồ sơ mọi thú cưng kèm lịch sử ghi chú đặt lịch (`appointmentNotes`). Không xóa cứng nếu thú cưng còn đơn đang hoạt động (`409`).

---

## 1. Mở nhanh code trong VS Code (Code Map)

| Tầng | File | Chức năng |
| :--- | :--- | :--- |
| **Route + Multer** | `src/routes/pet.routes.js` (`L1-55`) | `router.use(protect)` (`L40`), inline `uploadPhoto` (`L13-37`), 5 API CRUD + `/:petId/timeline` |
| **Validator** | `src/validators/pet.validator.js` (`L1-249`) | Magic Bytes (JPEG/PNG/WebP), data URI chống SSRF, validate `dob`/`weight`/`page`/`limit` |
| **Controller** | `src/controllers/pet.controller.js` (`L1-91`) | Điều phối 5 API + `handleError` ẩn chi tiết lỗi nội bộ về `500` |
| **Service** | `src/services/pet.service.js` (`L1-221`) | CRUD, sinh mã `PET-XXXXXX`, Cloudinary `uploadPhoto`/`removePhoto`, kiểm tra đơn đang mở trước khi xóa |
| **Models** | `Pet.js`, `AppointmentOrder.js` | `Pet` (index `{ ownerId: 1 }`, unique `code`); `AppointmentOrder` (đọc ghi chú & chặn xóa) |
| **Test** | `tests/services/pet.service.test.js` | Unit test service |

---

## 2. Chi tiết 5 API CRUD — Luồng code & Toàn bộ Case

> **Lưu ý:** Toàn bộ route đều yêu cầu `protect` (`pet.routes.js:40`). Không gắn `authorize` ở route vì quyền sở hữu (`ownerId = req.user.id`) được kiểm soát ở Service.

### 2.1. `POST /api/pets` — Tạo hồ sơ thú cưng
- **Middleware:** `protect` → `uploadPhoto` (`pet.routes.js:23-37`, field `photo`, tối đa 1 ảnh `5MB`, lỗi Multer trả `400`).
- **Input:** `name*` (≤100), `species?`, `breed?`, `coatColor?` (≤50), `notes?` (≤1000), `dob?` (không ở tương lai), `weight?` (`0–500` kg), `photo?` (file Buffer hoặc base64 data URI).
- **Luồng (`pet.controller.js:27-36` → `pet.service.js:80-107`):**
  1. `validateCreatePet` (`pet.validator.js:169-180`): kiểm tra tất cả field + kiểm tra **Magic Bytes** (`L39-63`: JPEG `FF D8 FF`, PNG 8-byte, WebP `RIFF..WEBP`). Từ chối plain URL / file path để **chống SSRF / Local File Read** (`L65-75`).
  2. Chỉ lấy field trong `UPDATABLE_FIELDS` (`pet.service.js:12, 81-84`, chặn Mass Assignment `code/ownerId/status`).
  3. Upload ảnh lên Cloudinary thư mục `pets/` (`L86`).
  4. Vòng lặp 5 lần sinh mã `PET-XXXXXX` và `Pet.create` (`L90-101`). **Nếu DB lỗi → `catch` gọi `removePhoto(photoUrl)` dọn ngay ảnh mồ côi (`L103-105`)**.
  5. Trả **`201`** `{ pet }`.

### 2.2. `GET /api/pets` — Danh sách thú cưng của tôi
- **Query:** `status?` (`Active` mặc định | `Deleted` | `All`), `page?` (≥1), `limit?` (`1–100`, mặc định 20).
- **Luồng (`pet.service.js:143-156`):** Luôn lọc `ownerId = req.user.id`, chạy song song `Pet.find` + `countDocuments` → trả **`200`** `{ items, total, page, limit, totalPages }`.

### 2.3. `GET /api/pets/:petId` — Xem chi tiết thú cưng + ghi chú đặt lịch
- **Luồng (`pet.controller.js:50-59` → `pet.service.js:114-133`):**
  1. `validatePetId` (`pet.validator.js:211-216`) kiểm tra 24 ký tự hex (`400` nếu sai).
  2. **Phân quyền đọc (`L115`):** `PetOwner` chỉ lọc `{ _id: petId, ownerId }` (chống IDOR → không phải của mình trả `404`); 4 vai trò nhân sự phòng khám lọc `{ _id: petId }` (xem được mọi pet).
  3. Đọc thêm `getPetAppointmentNotes(pet._id)` (`L125-133`): các đơn chưa `Cancelled` kèm `note` & `intake` → trả **`200`** `{ pet, appointmentNotes }`.

### 2.4. `PATCH /api/pets/:petId` — Cập nhật hồ sơ thú cưng
- **Middleware:** `protect` → `uploadPhoto`.
- **Luồng (`pet.service.js:162-187`):**
  1. `validateUpdatePet` (`pet.validator.js:190-205`): gửi rỗng không đổi gì → **`400`** `'Cần ít nhất một thông tin để cập nhật.'`; field optional gửi chuỗi rỗng sẽ được set `null` để xóa nội dung cũ (`L111`).
  2. Tìm `{ _id: petId, ownerId }` (`404` nếu không tồn tại/không phải chủ).
  3. Nếu `pet.status === 'Deleted'` → **`409`** `'Không thể cập nhật các thú cưng đã bị xóa'` (`L165`).
  4. Nếu có ảnh mới: upload ảnh mới → `pet.save()`. Lưu DB lỗi → xóa ảnh mới (`L181`); lưu thành công → xóa ảnh cũ trên Cloudinary (`L185`).

### 2.5. `DELETE /api/pets/:petId` — Xóa mềm thú cưng (`status → 'Deleted'`)
- **Luồng (`pet.service.js:193-212`):**
  1. Kiểm tra `AppointmentOrder.exists({ petId, ownerId, derivedStatus: { $in: ['Booked', 'CheckedIn', 'InProgress', 'WaitOwner'] } })` (`L195-202`). Nếu còn đơn đang hoạt động → **`409`** `'Không thể xóa thú cưng đang có lịch hẹn hoặc ca khám chưa hoàn tất'`.
  2. `Pet.findOneAndUpdate({ _id: petId, ownerId }, { $set: { status: 'Deleted' } }, { new: true })` (giữ nguyên dữ liệu & ảnh cũ, idempotent) → không thấy ném `404` → trả **`200`** `{ message: 'Đã xóa thú cưng', pet }`.

---

## 3. Bảng tổng hợp Case (Tốt → Fail)

| Endpoint | Case | HTTP | Message / Kết quả | Code ở đâu |
| :--- | :--- | :-: | :--- | :--- |
| `POST /` | Tạo thành công | **`201`** | `{ pet }` | `pet.controller.js:32` |
| `POST /`, `PATCH /:petId` | Multer quá 5MB / sai tên field `photo` | **`400`** | `'Ảnh không được vượt quá 5MB.'` / `'Tệp ảnh phải được gửi với tên trường "photo".'` | `pet.routes.js:28-32` |
| `POST /`, `PATCH /:petId` | Sai validate (`name`, `dob` tương lai, `weight`, Magic Bytes sai, gửi URL ngoài, body rỗng) | **`400`** | `{ message, errors: [{ field, message }] }` | `pet.validator.js:54-77, 177, 202` |
| `GET /:petId`, `PATCH`, `DELETE` | `petId` sai định dạng 24 hex | **`400`** | `'Mã thú cưng không hợp lệ.'` | `pet.validator.js:212-214` |
| `GET /` | `status`/`page`/`limit` sai | **`400`** | `'Tham số truy vấn không hợp lệ.'` | `pet.validator.js:230-240` |
| `GET/PATCH/DELETE /:petId` | Không tồn tại hoặc không thuộc `ownerId` | **`404`** | `'Không tìm thấy thú cưng'` | `pet.service.js:117, 164, 210` |
| `PATCH /:petId` | Thú cưng đã `Deleted` | **`409`** | `'Không thể cập nhật các thú cưng đã bị xóa'` | `pet.service.js:165` |
| `DELETE /:petId` | Còn lịch hẹn/ca khám chưa hoàn tất | **`409`** | `'Không thể xóa thú cưng đang có lịch hẹn hoặc ca khám chưa hoàn tất'` | `pet.service.js:200-202` |
| `POST /` | 5 lần sinh mã `PET-XXXXXX` đều trùng | **`500`** | `'Không thể tạo mã thú cưng. Vui lòng thử lại.'` (đã xóa ảnh Cloudinary) | `pet.service.js:102-105` |

---

## 4. Câu hỏi hội đồng hay xoáy (Defense Q&A)

1. **Nếu upload ảnh lên Cloudinary xong nhưng ghi MongoDB bị lỗi, ảnh trên Cloudinary có thành rác không?**
   - **Trả lời:** Không. `pet.service.js:103-105` (create) và `L180-182` (update) bắt lỗi và gọi `removePhoto()` xóa ngay ảnh mới đẩy lên; còn khi cập nhật ảnh mới thành công ở `L185` sẽ xóa ảnh cũ đi.
2. **Tại sao `pet.validator.js:65-75` lại chặn client truyền chuỗi URL ảnh (`https://...`) mà chỉ nhận file upload hoặc base64 data URI?**
   - **Trả lời:** Vì `cloudinary.uploader.upload(string)` sẽ tự fetch chuỗi đó từ phía server — nếu cho client truyền URL hoặc đường dẫn tùy ý sẽ dính lỗ hổng **SSRF** hoặc **Local File Read**.
3. **Chủ nuôi cố xóa thú cưng đang nằm viện hoặc đang có lịch hẹn chưa khám thì sao?**
   - **Trả lời:** `pet.service.js:195-202` kiểm tra `AppointmentOrder.exists` với 4 trạng thái hoạt động (`Booked`, `CheckedIn`, `InProgress`, `WaitOwner`) và chặn bằng **`409 Conflict`**.
