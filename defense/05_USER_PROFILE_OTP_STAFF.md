# 05 — Hồ Sơ Cá Nhân, Xác Thực OTP, Đổi/Quên Mật Khẩu & Danh Sách Nhân Sự (`/api/users`)

> **Tóm tắt nói trong 20 giây:** Cho phép mọi tài khoản tự xem/sửa profile, xác thực email bằng OTP 6 số (hiệu lực 5 phút, lưu dạng băm bcrypt), đổi mật khẩu / quên mật khẩu qua email đã xác thực, và tra cứu danh sách Bác sĩ/Groomer khả dụng (tự lọc theo dịch vụ & loại trừ người báo bận trên `StaffSchedule`).

---

## 1. Mở nhanh code trong VS Code (Code Map)

| Tầng | File | Chức năng |
| :--- | :--- | :--- |
| **Route** | `src/routes/user.routes.js` (`L1-28`) | 9 endpoint (`/me*`, `/forgot-password`, `/reset-password`, `/staff`) |
| **Middleware** | `src/middlewares/auth.middleware.js` | `protect` (7 route); `authorize(5 roles)` trên `GET /staff` |
| **Validator** | `src/validators/user.validator.js` (`L1-144`) | Validate profile, OTP 6 số, password, query lọc nhân sự |
| **Controller** | `src/controllers/user.controller.js` (`L1-164`) | Nhận `req.user.id` từ token (chống IDOR), trả `devToken` khi chưa có SMTP |
| **Service** | `src/services/user.service.js` (`L1-381`) | OTP + Nodemailer, cập nhật profile, đổi/reset pass, lọc nhân sự theo lịch bận |
| **Models liên quan** | `User.js`, `Service.js`, `StaffSchedule.js` | Đọc/ghi `User`; `listStaff` đọc thêm `Service` & `StaffSchedule` |
| **Test** | `tests/user.test.js`, `tests/services/user.service.test.js` | Test đầy đủ OTP, quên pass, đổi pass, lọc nhân sự |

---

## 2. Chi tiết từng API — Luồng code & Toàn bộ Case

### 2.1. `GET /api/users/me` & `PUT /api/users/me` — Xem & Cập nhật hồ sơ cá nhân
- **Quyền:** Mọi role (`protect`) — `user.routes.js:8-9`. Luôn lấy `req.user.id` từ JWT (`user.controller.js:14, 32`).
- **Luồng `GET /me` (`user.service.js:102-111`):** `User.findById(userId).select(PROFILE_FIELDS).lean()` → không thấy ném `404` → trả `200 { user }`.
- **Luồng `PUT /me` (`user.service.js:116-157`):**
  1. Validate `fullName*`, `phone*`, `email?` (`user.validator.js:29-43`).
  2. Nếu đổi sang email mới khác email cũ → reset `isEmailVerified = false` và `$unset` OTP cũ (`L133-136`).
  3. Nếu gửi `email: ""` hoặc `null` → `$unset: { email: 1, emailOtpHash: 1, emailOtpExpiresAt: 1 }` và `isEmailVerified = false` (`L137-140`) để không đụng Partial Unique Index `email`.
  4. `findByIdAndUpdate(..., { returnDocument: 'after' })`; bắt Mongo `11000` → ném `409` (`L150-154`).

| API | Case | HTTP | Message / Kết quả | Code ở đâu |
| :--- | :--- | :-: | :--- | :--- |
| `GET /me` | Tốt / User không còn trong DB | **`200`** / **`404`** | `{ user }` / `'Không tìm thấy người dùng'` | `user.service.js:106-110` |
| `PUT /me` | Tốt | **`200`** | `{ user }` (đổi email → `isEmailVerified: false`) | `user.service.js:133-147` |
| `PUT /me` | Sai format `fullName`/`phone`/`email` | **`400`** | `{ message: 'Dữ liệu không hợp lệ', errors }` | `user.controller.js:29`, `user.validator.js:32-40` |
| `PUT /me` | Không tìm thấy user | **`404`** | `'Không tìm thấy người dùng'` | `user.service.js:118-120` |
| `PUT /me` | Trùng SĐT hoặc Email với người khác (`11000`) | **`409`** | `'Số điện thoại/Email đã được sử dụng'` | `user.service.js:150-154` |

---

### 2.2. `POST /api/users/me/email/send-otp` & `POST /api/users/me/email/verify` — Xác thực Email 2 bước
- **Quyền:** Mọi role (`protect`) — `user.routes.js:10-11`.
- **Luồng bước 1 `send-otp` (`user.service.js:160-173`):** Kiểm tra user tồn tại (`404`), đã có `email` (`400`), chưa `isEmailVerified` (`400`) → gọi `createAndSendOtp` (`L78-89`): sinh OTP 6 số bằng `crypto.randomInt(100000, 1000000)`, băm `bcrypt.hash(otp, 10)`, lưu `emailOtpHash` + `emailOtpExpiresAt = now + 5 phút`, gửi Gmail (hoặc trả `devToken` nếu chưa cấu hình SMTP).
- **Luồng bước 2 `verify` (`user.service.js:176-205`):** Validate `otp` đúng 6 chữ số (`user.validator.js:45-52`) → `.select('+emailOtpHash +emailOtpExpiresAt')` → gọi `assertValidOtp` (`L92-100`: check còn hạn & `bcrypt.compare`) → bật `isEmailVerified = true` và `$unset` OTP (`L193-204`).

| Case | HTTP | Message / Kết quả | Code ở đâu |
| :--- | :-: | :--- | :--- |
| Gửi OTP thành công | **`200`** | `{ message: 'Mã xác thực đã được gửi...', devToken? }` | `user.controller.js:42-45` |
| Xác thực OTP thành công | **`200`** | `{ message: 'Xác thực email thành công', user }` | `user.controller.js:60` |
| Chưa có email / Email đã xác thực / OTP sai format / OTP sai hoặc hết 5 phút | **`400`** | `'Tài khoản chưa cập nhật...'` / `'Email này đã được xác thực'` / `'Mã xác thực không hợp lệ hoặc đã hết hạn'` | `user.service.js:94,98,166,169,185,188` |
| User không tồn tại | **`404`** | `'Không tìm thấy người dùng'` | `user.service.js:163, 182` |

---

### 2.3. `POST /api/users/forgot-password` & `POST /api/users/reset-password` — Quên mật khẩu (Public)
- **Quyền:** Công khai (không cần token) — `user.routes.js:18-19`.
- **Luồng `forgot-password` (`user.service.js:208-223`):** Validate `email` → tìm user theo email (`404` nếu không thấy) → bắt buộc `isEmailVerified === true` (`400`) → chặn nếu `status === 'Locked'` (`403`) → gửi OTP 6 số (`5 phút`).
- **Luồng `reset-password` (`user.service.js:226-246`):** Chấp nhận cả field `otp` hoặc `token` (khớp FE, `user.validator.js:66`) + `password` (`8–72` ký tự) → kiểm tra user (`404`), `isEmailVerified` (`400`), `assertValidOtp` (`400`) → lưu `passwordHash` mới và `$unset` OTP (`L241-245`).

| Case | HTTP | Message | Code ở đâu |
| :--- | :-: | :--- | :--- |
| Gửi OTP / Đặt lại mật khẩu thành công | **`200`** | `'Mã OTP khôi phục...'` / `'Đặt lại mật khẩu thành công'` | `user.controller.js:75, 94` |
| Sai format email/otp/password, email chưa verify, OTP sai/hết hạn | **`400`** | Chi tiết theo trường hoặc `'Mã xác thực không hợp lệ hoặc đã hết hạn'` | `user.validator.js:54-78`, `user.service.js:94,98,216,236` |
| Tài khoản bị `Locked` gọi quên mật khẩu | **`403`** | `'Tài khoản đã bị khoá, vui lòng liên hệ quản trị viên'` | `user.service.js:218-220` |
| Không có tài khoản ứng với email | **`404`** | `'Không tìm thấy tài khoản với email này'` | `user.service.js:213, 233` |

---

### 2.4. `POST /api/users/me/password/send-otp` & `PUT /api/users/me/password` — Đổi mật khẩu khi đang đăng nhập
- **Quyền:** Mọi role (`protect`) — `user.routes.js:14-15`.
- **Luồng (`user.service.js:249-284`):**
  - Bắt buộc tài khoản đã có email **và đã `isEmailVerified`** (`400`).
  - Ở bước `PUT /me/password`: validate `otp` (6 số), `currentPassword`, `newPassword` (`8–72`, **phải khác `currentPassword`** ở `user.validator.js:93-95`) → kiểm tra đúng mật khẩu hiện tại bằng `bcrypt.compare` (`L273-275`, sai → `400`) → `assertValidOtp` (`400`) → lưu `passwordHash` mới & xóa OTP (`L279-283`).

---

### 2.5. `GET /api/users/staff` — Tra cứu danh sách Bác sĩ / Groomer khả dụng
- **Quyền:** Tất cả 5 role đã đăng nhập — `user.routes.js:21-26`.
- **Query:** `role?` (`Veterinarian|Groomer`), `status?` (mặc định `Active`), `search?`, `date?`/`workDate?` (`YYYY-MM-DD`), `shift?` (`MORNING|AFTERNOON|EVENING`), `time?` (`HH:mm`), `serviceId?`, `page?`, `limit?` (`1–100`).
- **Luồng thông minh (`user.service.js:290-369`):**
  1. Nếu truyền `serviceId` (`L304-321`): đọc `Service` (`404` nếu không thấy) → suy ra `requiredRole` (Grooming → `Groomer`, khác → `Veterinarian`). Nếu client lọc `role` mâu thuẫn với `requiredRole` → trả mảng rỗng ngay (`L314-319`).
  2. Nếu truyền `date`/`workDate` hoặc `shift` (`L328-350`): tự tính `effectiveShift` (từ `shift`, hoặc suy từ `time`, hoặc giờ hiện tại VN nếu là hôm nay) → gọi `findBusyStaffIds` từ `staffSchedule.service.js` → loại trừ bằng `filter._id = { $nin: busyIds }` (**mô hình Blacklist**).
  3. Tìm kiếm regex đã escape chống ReDoS (`L22, L352-356`), chỉ `.select('code fullName role status')` (`L20`, không lộ SĐT/email/password), chạy song song `find` + `countDocuments` (`L360-363`).

| Case | HTTP | Kết quả / Message | Code ở đâu |
| :--- | :-: | :--- | :--- |
| Thành công | **`200`** | `{ items: [{ _id, code, fullName, role, status }], pagination }` | `user.service.js:365-368` |
| Sai query (`role`, `date`, `shift`, `time`, `serviceId`, `page`, `limit`) | **`400`** | `{ message: 'Dữ liệu không hợp lệ', errors }` | `user.controller.js:132`, `user.validator.js:100-135` |
| `serviceId` không tồn tại | **`404`** | `'Không tìm thấy dịch vụ'` | `user.service.js:308-312` |

---

## 3. Câu hỏi hội đồng hay xoáy (Defense Q&A)

1. **Mã OTP lưu trong MongoDB dạng rõ hay dạng băm? Nếu lộ DB thì kẻ gian có đọc được OTP không?**
   - **Trả lời:** Lưu dạng băm `bcrypt.hash(otp, 10)` vào `emailOtpHash` (`user.service.js:80`), đồng thời `select: false` trên Model `User.js:10-11`, có hạn 5 phút (`OTP_TTL_MS`) và bị `$unset` xóa sạch ngay sau khi xác thực thành công (`L197, L244, L282`).
2. **Nếu người dùng đổi sang email mới trong `PUT /api/users/me` thì trạng thái xác thực cũ thế nào?**
   - **Trả lời:** `user.service.js:133-136` so sánh `trimmedEmail !== currentUser.email`; nếu khác lập tức reset `isEmailVerified = false` và xóa OTP cũ để bắt xác thực lại email mới.
3. **Ở `GET /api/users/staff`, làm sao biết bác sĩ nào rảnh mà không cần Admin phải tạo lịch làm việc mỗi ngày?**
   - **Trả lời:** Dùng mô hình **Blacklist**: mặc định bác sĩ `Active` đều rảnh, chỉ khi họ báo bận mới có bản ghi trong `StaffSchedule`. `user.service.js:346-349` gọi `findBusyStaffIds` rồi loại trừ bằng `$nin: busyIds`.
4. **Một điểm nhỏ hội đồng có thể góp ý ở `resetPassword`?**
   - **Trả lời:** `user.service.js:228-237` chưa kiểm tra `user.status === 'Locked'` ở bước `resetPassword` (mới chặn ở bước `forgotPassword:218-220`). Nếu tài khoản bị khóa giữa bước 1 và bước 2, vẫn nên chặn thêm ở `resetPassword` (nhưng dù có đặt lại được pass thì `login` và `protect` vẫn chặn `403`).
