# 06 — Quản Trị Tài Khoản Người Dùng & Nhân Sự (`/api/admin/users` — `SCR-32`)

> **Tóm tắt nói trong 20 giây:** Toàn bộ cụm API chỉ dành riêng cho `Admin`: xem/tìm/lọc tất cả tài khoản (`UC-60`), xem chi tiết kèm thú cưng nếu là `PetOwner`, tạo tài khoản nhân sự `Receptionist/Veterinarian/Groomer` (`UC-61`), sửa tên/điều chuyển vai trò nhân sự, khóa/mở khóa (`UC-62`) và chặn xóa vật lý (`BR-22`).

---

## 1. Mở nhanh code trong VS Code (Code Map)

| Tầng | File | Chức năng |
| :--- | :--- | :--- |
| **Route** | `src/routes/admin.routes.js` (`L1-18`) | Gắn chung `router.use(protect, authorize('Admin'))` ở `L8` cho cả 6 API |
| **Validator** | `src/validators/admin.validator.js` (`L1-88`) | Validate tạo nhân sự, cập nhật tên/role, trạng thái `Active \| Locked` |
| **Controller** | `src/controllers/admin.controller.js` (`L1-89`) | Điều phối 6 handler + chặn cứng `deleteUser` trả `403` (`L75-79`) |
| **Service** | `src/services/admin.service.js` (`L1-183`) | 5 hàm nghiệp vụ + dùng lại `generateUserCode` từ `auth.service.js` |
| **Models** | `User.js`, `Pet.js` | Đọc/ghi `User`; `getUserById` đọc thêm `Pet` nếu user là `PetOwner` |
| **Test** | `tests/admin.test.js`, `tests/services/admin.service.test.js` | Test toàn bộ RBAC, tạo nhân sự, tự khóa, `BR-22` |

---

## 2. Chi tiết 6 API — Luồng code & Toàn bộ Case

> **Lưu ý chung:** Mọi endpoint dưới đây đều đi qua `protect` + `authorize('Admin')` (`admin.routes.js:8`), nên thiếu token → **`401`**, role khác `Admin` → **`403`**.

### 2.1. `GET /api/admin/users` — Danh sách & tìm kiếm/lọc tài khoản (`UC-60`)
- **Query:** `role?` (5 roles), `status?` (`Active|Inactive|Locked`), `search?` (tìm theo tên, SĐT, email, mã `USR-*`).
- **Luồng (`admin.service.js:21-51`):** Kiểm tra `role`/`status` hợp lệ (`400` nếu sai), escape regex từ khóa chống ReDoS (`L40`), tìm `$or` trên 4 trường `fullName, phone, email, code`, chỉ `.select(PROFILE_FIELDS)` và sắp mới nhất trước (`createdAt: -1`).

### 2.2. `GET /api/admin/users/:id` — Xem chi tiết 1 tài khoản (`SCR-32`)
- **Luồng (`admin.service.js:54-69`):** Kiểm tra ObjectId (`400`) → tìm `User` (`404` nếu không thấy) → nếu `user.role === 'PetOwner'`, query thêm `Pet.find({ ownerId: user._id })` gắn vào `user.pets` (`L64-66`) → trả `200 { user }`.

### 2.3. `POST /api/admin/users` — Tạo tài khoản nhân sự (`UC-61`, `BR-24`)
- **Body:** `{ fullName*, phone*, email?, password* (8–72), role* ('Receptionist' | 'Veterinarian' | 'Groomer') }`.
- **Luồng (`admin.service.js:72-113`):**
  1. Validator chặn không cho tạo `Admin` hoặc `PetOwner` qua API này (`admin.validator.js:10, 44-46`).
  2. Check trùng `phone` (`409`), trùng `email` (`409`), băm mật khẩu `bcrypt.hash(password, 10)` (`L76-83`).
  3. Vòng lặp 5 lần sinh mã `USR-XXXXXX`, tạo `status: 'Active'`, bắt `11000` (nếu trùng `code` thì retry, trùng `phone`/`email` do race condition thì trả `409`, hết 5 lần trả `500`), trả `201 { user }`.

### 2.4. `PUT /api/admin/users/:id` — Cập nhật họ tên & điều chuyển vai trò nhân sự
- **Body:** `{ fullName*, role? ('Receptionist' | 'Veterinarian' | 'Groomer') }`.
- **Luồng (`admin.service.js:117-148`):**
  - Không cho Admin sửa `phone`/`email` (đó là định danh đăng nhập gắn với OTP chính chủ ở `/api/users/me`).
  - Kiểm tra ObjectId (`400`), tìm user (`404`).
  - **Chặn đổi role của `PetOwner` hoặc `Admin` (`L127-131`):** nếu gửi `role` mới khác role hiện tại mà user không nằm trong `STAFF_ROLES` → ném **`400`** `'Chỉ được phép thay đổi vai trò của tài khoản nhân viên'`.
  - Cập nhật và trả `200 { user }`.

### 2.5. `PATCH /api/admin/users/:id/status` — Khóa / Mở khóa tài khoản (`UC-62`)
- **Body:** `{ status*: 'Active' | 'Locked' }` (không cho set `Inactive` ở đây, `admin.validator.js:13, 72-74`).
- **Luồng (`admin.service.js:151-174`):**
  - Kiểm tra ObjectId (`400`).
  - **Chống Admin tự khóa chính mình (`L157-159`):** `String(adminId) === String(targetUserId) && status === 'Locked'` → ném **`400`** `'Không thể tự khóa tài khoản của chính mình'`.
  - `findByIdAndUpdate` → không thấy ném `404` → trả `200 { user }`.

### 2.6. `DELETE /api/admin/users/:id` — Chặn xóa vật lý (`BR-22`)
- **Luồng (`admin.controller.js:75-79`):** Trả thẳng **`403`** `'Không được phép xóa vật lý tài khoản người dùng (BR-22). Vui lòng sử dụng chức năng khóa tài khoản'`.

---

## 3. Bảng tổng hợp tất cả Case (Tốt → Fail)

| Endpoint | Case | HTTP | Message / Kết quả | Code ở đâu |
| :--- | :--- | :-: | :--- | :--- |
| Mọi API | Thiếu/sai token | **`401`** | `'Chưa đăng nhập'` / `'Token không hợp lệ...'` | `auth.middleware.js:10,46` |
| Mọi API | Không phải `Admin` | **`403`** | `'Bạn không có quyền thực hiện chức năng này'` | `admin.routes.js:8`, `auth.middleware.js:60` |
| `GET /` | Tốt / Lọc `role` hoặc `status` sai | **`200`** / **`400`** | `{ users }` / `'Vai trò/Trạng thái lọc không hợp lệ'` | `admin.service.js:26,33,50` |
| `GET /:id` | Tốt / ID sai / Không thấy | **`200`** / **`400`** / **`404`** | `{ user }` (kèm `pets` nếu `PetOwner`) | `admin.service.js:56,61,68` |
| `POST /` | Tốt | **`201`** | `{ user }` (`USR-XXXXXX`, `Active`) | `admin.controller.js:36` |
| `POST /` | Validate sai (`role` là `Admin`/`PetOwner`, pass `<8`...) | **`400`** | `{ message: firstErrorMessage, errors }` | `admin.controller.js:31-32`, `admin.validator.js:26-48` |
| `POST /` | Trùng SĐT hoặc Email (hoặc `11000`) | **`409`** | `'Số điện thoại/Email đã được đăng ký'` | `admin.service.js:77,80,106` |
| `POST /` | 5 lần sinh mã đều trùng | **`500`** | `'Không thể tạo mã nhân viên, vui lòng thử lại'` | `admin.service.js:112` |
| `PUT /:id` | Tốt / Sai validate / Đổi role `PetOwner`/`Admin` / Không thấy | **`200`** / **`400`** / **`404`** | `{ user }` / `'Chỉ được phép thay đổi vai trò của tài khoản nhân viên'` | `admin.service.js:119,124,129,141` |
| `PATCH /:id/status` | Tốt / Sai `status` / Admin tự khóa mình / Không thấy | **`200`** / **`400`** / **`404`** | `{ user }` / `'Không thể tự khóa tài khoản của chính mình'` | `admin.service.js:153,158,170` |
| `DELETE /:id` | Mọi request xóa cứng (`BR-22`) | **`403`** | `'Không được phép xóa vật lý tài khoản người dùng (BR-22)...'` | `admin.controller.js:76-78` |

---

## 4. Câu hỏi hội đồng hay xoáy (Defense Q&A)

1. **Khi Admin khóa (`Locked`) một nhân viên đang đăng nhập, nhân viên đó có còn thao tác tiếp được không?**
   - **Trả lời:** Bị đá ra ngay ở request kế tiếp với mã `403`, vì middleware `protect` (`auth.middleware.js:23-37`) kiểm tra trực tiếp `user.status` trong MongoDB trên mỗi request.
2. **Tại sao Admin không sửa được SĐT và Email của nhân viên/khách hàng trong `PUT /api/admin/users/:id`?**
   - **Trả lời:** SĐT là định danh đăng nhập chính, còn Email gắn với cơ chế xác thực OTP cá nhân. Tách quyền chỉ cho chính chủ sửa ở `/api/users/me` (`admin.validator.js:51-53`) tránh rủi ro Admin chiếm đoạt tài khoản hoặc làm lệch trạng thái `isEmailVerified`.
3. **Tại sao không cho Admin xóa tài khoản khi nhân viên nghỉ việc?**
   - **Trả lời:** Theo `BR-22`, các bảng `AppointmentOrder`, `MedicalRecord`, `InpatientRecord`, `Invoice` đều tham chiếu `ObjectId` đến `User`. Xóa cứng sẽ làm gãy toàn bộ lịch sử bệnh án và hóa đơn, nên chỉ khóa `Locked` để bảo toàn dữ liệu kiểm toán.
