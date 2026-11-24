# 04 — Xác Thực Tài Khoản: Đăng Ký, Đăng Nhập, Đăng Xuất (`/api/auth`)

> **Tóm tắt nói trong 20 giây:** Quản lý đăng ký `PetOwner` (hỗ trợ kích hoạt lại tài khoản khách Walk-in theo `BR-04`), đăng nhập bằng SĐT hoặc Email (kèm chống brute-force sai 5 lần khóa 30 phút `BR-03`) và đăng xuất JWT stateless.

---

## 1. Mở nhanh code trong VS Code (Code Map)

| Tầng | File | Chức năng |
| :--- | :--- | :--- |
| **Route** | `src/routes/auth.routes.js` (`L1-11`) | 3 API: `POST /register`, `POST /login`, `POST /logout` |
| **Middleware** | `src/middlewares/auth.middleware.js` (`L7-48`) | `protect` gắn trên `/logout` |
| **Validator** | `src/validators/auth.validator.js` (`L1-49`) | Regex SĐT `^0\d{9}$`, Email, Password `8–72` ký tự |
| **Controller** | `src/controllers/auth.controller.js` (`L1-35`) | Gọi validator → gọi service → trả JSON |
| **Service** | `src/services/auth.service.js` (`L1-171`) | Nghiệp vụ `register`, `login`, `generateUserCode`, `BR-03`, `BR-04` |
| **Model** | `src/models/User.js` (`L1-29`) | Unique `code`, `phone`, Partial Unique `email`; `passwordHash` `select: false` |
| **Test** | `tests/auth.test.js`, `tests/services/auth.service.test.js` | Test tích hợp & unit test |

---

## 2. Chi tiết từng API — Luồng code & Toàn bộ Case (Tốt → Fail)

### 2.1. `POST /api/auth/register` — Đăng ký tài khoản `PetOwner` (`UC-02`, `BR-04`)
- **Quyền:** Công khai (không cần token) — `auth.routes.js:7`
- **Body:** `{ fullName*, phone*, email?, password* }`
- **Luồng code chạy tuần tự:**
  1. **Validate** (`auth.controller.js:5-6` → `auth.validator.js:9-29`): kiểm tra `fullName` (1–100 ký tự), `phone` (`^0\d{9}$`), `email` (optional, đúng format), `password` (`8–72` ký tự, vì bcrypt chỉ băm 72 byte đầu).
  2. **Kiểm tra trùng SĐT + nhánh `BR-04` kích hoạt khách Walk-in** (`auth.service.js:55-92`):
     - Nếu `phone` đã tồn tại trong DB: kiểm tra xem có phải tài khoản khách vãng lai (`role === 'PetOwner' && status === 'Inactive' && !passwordHash`) hay không (`L62-66`).
     - Nếu **không phải** → ném **`409`** `'Số điện thoại đã được đăng ký'` (`L68-70`).
     - Nếu **đúng là Walk-in chưa kích hoạt** → kiểm tra `email` mới không trùng user khác (`L73-76`), băm mật khẩu `bcrypt.hash(password, 10)`, bật `status = 'Active'` và lưu lại (`L78-87`) để khách giữ nguyên thú cưng & lịch sử khám trước đó.
  3. **Tạo tài khoản mới** (`auth.service.js:94-118`): kiểm tra trùng `email` (`L94`), băm mật khẩu (`L96`), vòng lặp tối đa 5 lần sinh mã `USR-XXXXXX` (bỏ `0/O/1/I`, `L6, 33-39`) và `User.create` luôn ép cứng `role: 'PetOwner'` (`L106`, chống Mass Assignment).
  4. **Trả về** (`auth.controller.js:10`): **`201`** `{ message: 'Đăng ký thành công', user }` (đã lọc qua `toPublicUser`, không có `passwordHash`).

| Case | Điều kiện kích hoạt | HTTP | Message / Kết quả | Code ở đâu |
| :--- | :--- | :-: | :--- | :--- |
| **Tốt (mới)** | Dữ liệu hợp lệ, SĐT/email chưa tồn tại | **`201`** | `{ message: 'Đăng ký thành công', user }` | `auth.controller.js:10`, `auth.service.js:100-108` |
| **Tốt (`BR-04`)** | SĐT thuộc khách Walk-in (`Inactive`, `passwordHash: null`) | **`201`** | Kích hoạt lên `Active`, giữ nguyên hồ sơ cũ | `auth.service.js:62-87` |
| **Fail Validate** | Thiếu/sai `fullName`, `phone`, `email`, `password` (`<8` hoặc `>72`) | **`400`** | `{ message: 'Dữ liệu không hợp lệ', errors }` | `auth.controller.js:6`, `auth.validator.js:12-26` |
| **Fail Trùng SĐT** | SĐT đã thuộc tài khoản `Active`/`Locked` hoặc có mật khẩu | **`409`** | `'Số điện thoại đã được đăng ký'` | `auth.service.js:68-70` |
| **Fail Trùng Email** | Email đã được tài khoản khác dùng (hoặc race condition `11000`) | **`409`** | `'Email đã được đăng ký'` / `'Số điện thoại hoặc email đã được đăng ký'` | `auth.service.js:75, 89, 94, 113` |
| **Fail Hệ thống** | 5 lần sinh mã `USR-XXXXXX` đều trùng `11000` | **`500`** | `'Không thể tạo mã người dùng, vui lòng thử lại'` | `auth.service.js:118` |

---

### 2.2. `POST /api/auth/login` — Đăng nhập bằng SĐT hoặc Email (`UC-01`, `BR-03`)
- **Quyền:** Công khai — `auth.routes.js:8`
- **Body:** `{ identifier* (SĐT hoặc Email), password* }`
- **Luồng code chạy tuần tự:**
  1. **Validate** (`auth.controller.js:17-18` → `auth.validator.js:31-38`): bắt buộc `identifier` và `password`.
  2. **Tìm User kèm `passwordHash`** (`auth.service.js:126-129`): nếu `identifier` khớp `PHONE_REGEX` thì tìm theo `{ phone }`, ngược lại tìm theo `{ email: lower }`, bắt buộc gọi `.select('+passwordHash')`.
  3. **Chặn đang bị tạm khóa `BR-03`** (`auth.service.js:132-137`): nếu `user.lockUntil > now` → ném ngay **`403`** (kể cả gõ đúng mật khẩu).
  4. **So khớp mật khẩu + đếm số lần sai** (`auth.service.js:140-157`):
     - Nếu không thấy user, hoặc `passwordHash` null (khách Walk-in chưa đăng ký), hoặc `bcrypt.compare` sai:
       - Nếu user có `passwordHash`: tăng `failedLoginAttempts`. Đạt **5 lần** → set `lockUntil = now + 30 phút`, lưu DB, ném **`403`** (`L145-152`).
       - Dưới 5 lần → lưu DB, ném **`401`** `'Thông tin đăng nhập không chính xác'` (`L156`) — gộp chung sai user & sai pass để tránh lộ tài khoản tồn tại.
  5. **Kiểm tra trạng thái tài khoản** (`auth.service.js:158-159`): `Locked` → **`403`**, `Inactive` → **`403`**.
  6. **Reset bộ đếm & cấp JWT** (`auth.service.js:162-168`): reset `failedLoginAttempts = 0`, `lockUntil = null`, ký JWT payload `{ id, role }` (`L28-31`, hạn `JWT_EXPIRES_IN || '1d'`), trả **`200`** `{ message, token, user }`.

| Case | Điều kiện kích hoạt | HTTP | Message / Kết quả | Code ở đâu |
| :--- | :--- | :-: | :--- | :--- |
| **Tốt** | Đúng SĐT/Email + mật khẩu, `status === 'Active'` | **`200`** | `{ message: 'Đăng nhập thành công', token, user }` | `auth.controller.js:22`, `auth.service.js:168` |
| **Fail Validate** | Thiếu `identifier` hoặc `password` | **`400`** | `{ message: 'Dữ liệu không hợp lệ', errors }` | `auth.controller.js:18`, `auth.validator.js:34-35` |
| **Fail Sai thông tin** | Sai SĐT/Email, sai mật khẩu (lần 1–4), hoặc Walk-in chưa có mật khẩu | **`401`** | `'Thông tin đăng nhập không chính xác'` | `auth.service.js:140-156` |
| **Fail Khóa 30p (`BR-03`)** | Sai mật khẩu lần 5 hoặc đang trong thời gian `lockUntil > now` | **`403`** | `'Bạn đã nhập sai mật khẩu ... tạm khóa ... 30 phút'` | `auth.service.js:132-137, 145-152` |
| **Fail Bị khóa/Ngừng** | `user.status === 'Locked'` hoặc `'Inactive'` | **`403`** | `'Tài khoản đã bị khoá...'` / `'Tài khoản đã ngừng hoạt động'` | `auth.service.js:158-159` |

---

### 2.3. `POST /api/auth/logout` — Đăng xuất
- **Quyền:** Mọi role đã đăng nhập (`protect`) — `auth.routes.js:9`
- **Luồng code:** `protect` kiểm tra Bearer token (`auth.middleware.js:7-48`) → `auth.controller.js:31-33` trả **`200`** `{ message: 'Đăng xuất thành công' }`. FE tự xóa token khỏi bộ nhớ (JWT stateless).

| Case | Điều kiện kích hoạt | HTTP | Message | Code ở đâu |
| :--- | :--- | :-: | :--- | :--- |
| **Tốt** | Gửi Bearer token hợp lệ, user `Active` | **`200`** | `'Đăng xuất thành công'` | `auth.controller.js:32` |
| **Fail Auth** | Thiếu token / token hỏng / hết hạn | **`401`** | `'Chưa đăng nhập'` / `'Token không hợp lệ hoặc đã hết hạn'` | `auth.middleware.js:10, 46` |
| **Fail Bị khóa** | Tài khoản đã bị Admin khóa | **`403`** | `'Tài khoản đã bị khoá...'` | `auth.middleware.js:31-36` |

---

## 3. Câu hỏi hội đồng hay xoáy (Defense Q&A)

1. **Khách hàng từng đến khám Walk-in tại quầy (chưa có mật khẩu), nay về nhà tự đăng ký online cùng SĐT đó thì có bị báo trùng không?**
   - **Trả lời:** Không. Theo `BR-04` (`auth.service.js:62-87`), hệ thống nhận diện tài khoản Walk-in có `role: 'PetOwner'`, `status: 'Inactive'`, `passwordHash: null`, cho phép đặt mật khẩu và nâng lên `Active` để giữ trọn hồ sơ thú cưng & lịch sử khám trước đó.
2. **Nếu hacker gửi `"role": "Admin"` trong body `POST /api/auth/register` thì có tạo được Admin không?**
   - **Trả lời:** Không. `auth.service.js:100-107` chỉ lấy 4 field `fullName, phone, email, password` và luôn gán cứng `role: 'PetOwner'`.
3. **Tại sao `MAX_PASSWORD = 72` trong `auth.validator.js:5`?**
   - **Trả lời:** Thuật toán `bcrypt` chỉ băm 72 byte đầu tiên và cắt bỏ phần dư, nên chặn trên 72 ký tự để tránh ảo giác bảo mật khi người dùng đặt mật khẩu dài hơn.
4. **Sau khi gọi `POST /api/auth/logout`, lấy lại token cũ gọi API có chạy được không?**
   - **Trả lời:** Vì dùng JWT stateless (`auth.controller.js:28-33`), token cũ vẫn còn hiệu lực đến khi hết hạn (`JWT_EXPIRES_IN=1d`) trừ khi Admin khóa tài khoản (lúc đó `protect` chặn ngay qua DB). Nếu mở rộng production cần thu hồi tức thời, sẽ lưu `jti` vào Redis blacklist tới lúc token hết hạn.
