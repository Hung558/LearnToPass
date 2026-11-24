# 02 — Middlewares, Bảo Mật (Auth/RBAC/IDOR/Upload) & Xử Lý Lỗi (VCMOS_BE)

> **Phạm vi:** Toàn bộ hàng rào bảo vệ request trước khi vào nghiệp vụ (`protect`, `authorize`, 5 luồng `multer`), ma trận phân quyền 5 vai trò, các chốt chống tấn công (`BR-03`, `BR-22`, IDOR, Mass Assignment, Magic Bytes, Timing-Safe HMAC) và kiến trúc xử lý lỗi.

---

## 1. Bản đồ Code (Code Map)

| Tầng | File mã nguồn | Vai trò |
| :--- | :--- | :--- |
| **App Middleware** | `server.js` (`L32, L34, L61-70`) | `cors`, `express.json()`, Global Error Middleware |
| **Auth & RBAC** | `src/middlewares/auth.middleware.js` (`L1-67`) | `protect` (JWT + DB status check) và `authorize(...roles)` |
| **Upload tập trung** | `src/middlewares/upload.middleware.js` (`L1-156`) | 4 middleware upload ảnh (`Grooming`, `Subtask`, `Community`, `AI`) |
| **Upload riêng Pet** | `src/routes/pet.routes.js` (`L9-37`) + `src/validators/pet.validator.js` (`L39-70`) | `uploadPhoto` inline + kiểm tra **Magic Bytes** (JPEG/PNG/WebP) |
| **Chặn brute-force** | `src/services/auth.service.js` (`L121-166`) | `BR-03`: sai mật khẩu 5 lần khóa 30 phút (`lockUntil`) |
| **Chặn xóa vật lý** | `src/controllers/admin.controller.js:75-78`, `service.controller.js:75-78`, `invoice.controller.js:294-297` | `BR-22`: trả `403` cố ý với mọi `DELETE` cứng |
| **Unit Test** | `tests/authorize.test.js` (`L1-45`) | Test 5 kịch bản của `protect` + `authorize` |

---

## 2. Trace chi tiết Middleware Xác thực & Phân quyền (`src/middlewares/auth.middleware.js`)

### 2.1. `protect` (`auth.middleware.js:7-48`)
1. Đọc header `req.headers.authorization` (`L8`). Nếu không bắt đầu bằng `'Bearer '` → **`401`** `{ message: 'Chưa đăng nhập' }` (`L9-11`).
2. Tách `token = header.split(' ')[1]` (`L13`) và gọi `jwt.verify(token, process.env.JWT_SECRET)` (`L16`).
   - Nếu token sai chữ ký, rỗng, hoặc hết hạn (`JsonWebTokenError` / `TokenExpiredError`) → rơi vào `catch` (`L45-47`) → **`401`** `{ message: 'Token không hợp lệ hoặc đã hết hạn' }`.
3. **Điểm đặc biệt — Truy vấn DB kiểm tra trạng thái tức thời (`L18-41`):**
   - Khi MongoDB đã kết nối (`readyState === 1`) và `decoded.id` là ObjectId hợp lệ, gọi `User.findById(decoded.id).select('status role').lean()` (`L23-26`).
   - Nếu `user.status !== 'Active'` (`L30-37`) → chặn ngay **`403`**:
     - `Locked`: `'Tài khoản đã bị khoá, vui lòng liên hệ quản trị viên'`
     - `Inactive`: `'Tài khoản đã ngừng hoạt động'`
   - Gán `req.user = { id: decoded.id, role: user.role || decoded.role }` (`L38`) — luôn lấy `role` mới nhất từ DB (nếu Admin vừa điều chuyển vai trò nhân sự thì có hiệu lực ngay mà không đợi hết hạn JWT).
4. **Nhánh fallback cho Unit Test không nối DB (`L43-44`):** Nếu `readyState !== 1` hoặc `decoded.id` không phải ObjectId (như trong `tests/authorize.test.js:14` với `id: 'user-id'`), vẫn gán `req.user = { id: decoded.id, role: decoded.role }` để chạy `next()`.

### 2.2. `authorize(...allowedRoles)` (`auth.middleware.js:52-65`)
- **Luôn đặt SAU `protect`**, nhưng vẫn tự phòng thủ: nếu `!req.user` (`L55-57`, trường hợp dev quên gắn `protect` — đã test ở `tests/authorize.test.js:40-43`) → **`401`** `{ message: 'Chưa đăng nhập' }`.
- Nếu `!allowedRoles.includes(req.user.role)` (`L59-61`) → **`403`** `{ message: 'Bạn không có quyền thực hiện chức năng này' }`.
- Hợp lệ → `next()` (`L63`).

---

## 3. Ma trận Phân quyền RBAC 5 Vai Trò trên toàn bộ 20 Route Files

| Route Prefix | Public (Không token) | `PetOwner` | `Receptionist` | `Veterinarian` | `Groomer` | `Admin` |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| `/api/auth` | `POST /register`, `/login` | `/logout` | `/logout` | `/logout` | `/logout` | `/logout` |
| `/api/users` | `POST /forgot-password`, `/reset-password` | `/me*`, `GET /staff` | `/me*`, `GET /staff` | `/me*`, `GET /staff` | `/me*`, `GET /staff` | `/me*`, `GET /staff` |
| `/api/admin/users` | — | — | — | — | — | Toàn bộ (`DELETE` → `403`) |
| `/api/pets` | — | CRUD thú cưng của mình + Timeline | Timeline (all) | Timeline (ẩn `invoice` & `aiAssessments`) | Timeline (ẩn `invoice` & `aiAssessments`) | Timeline (all, ẩn `aiAssessments`) |
| `/api/services` | `GET /`, `GET /:id` | Đọc | Đọc | Đọc | Đọc | `POST`, `PUT`, `PATCH /status` (`DELETE` → `403`) |
| `/api/products` | `GET /`, `GET /:id` | Đọc | Đọc | Đọc | Đọc | `POST`, `PUT`, `PATCH /activate\|inactivate` |
| `/api/cages` | — | — | Đọc | Đọc | Đọc | Đọc + `POST`, `PATCH /:id` |
| `/api/cage-assignments` | — | — | Xem, Phân chuồng, Giải phóng | Xem, Phân chuồng, Giải phóng | Xem, Giải phóng | Xem, Phân chuồng, Giải phóng |
| `/api/staff-schedules` | — | — | Xem, Tạo/Xóa lịch bận | Xem, Tự báo bận (`/me`) | Xem, Tự báo bận (`/me`) | Toàn bộ |
| `/api/appointment-orders` | — | `/online`, `/me`, `/me/:id`, `/:id/cancel` | `GET /`, `/:id/check-in`, `/:id/cancel` | — | — | `GET /`, `POST /flag-overdue` |
| `/api/appointment-orders/walk-in` | — | — | `GET /customer-lookup`, `POST /` | — | — | `GET /customer-lookup` |
| `/api/appointment-services` | — | `/online`, `/online-booking`, `/:id/cancel` | `GET /`, `/:id/assign-staff`, `/:id/cancel` | `/my-queue`, `/:id/start` | `/my-queue`, `/:id/start`, `/:id/complete`, Grooming Note | `GET /` |
| `/api/medical-records` | — | Xem bệnh án thú cưng mình | — | Tạo/Sửa nháp, Chốt SOAP, Xem | — | Xem |
| `/api/subtask-requests` | — | — | — | Tạo, Xem, Hủy, Nhận & Trả kết quả | Xem, Nhận & Trả kết quả | Xem |
| `/api/inpatient-records` | — | `GET /pets/:petId/daily-logs` | — | Toàn bộ (Nhập, Logs, Xuất viện) | — | — |
| `/api/invoices` | `/vnpay-ipn`, `/vnpay-return`, `/payos-webhook` | `/me`, `/:id`, `/:id/pdf`, `/:id/payos-status` | Lập, Sửa retail, Thu tiền, Tạo link, PDF (`DELETE`→`403`) | — | — | `GET /`, `/:id`, PDF, Top sản phẩm (`DELETE`→`403`) |
| `/api/reports` | — | — | `/front-desk` | — | — | `/revenue`, `/operations`, `/front-desk` |
| `/api/community-posts` | `GET /public(/:id)`, `/:id/comments` | Toàn bộ CRUD, TimelineShare, Like, Comment, Report, Withdraw | — | — | — | — |
| `/api/admin/community` | — | — | — | — | — | Duyệt bài & Xử lý báo cáo |
| `/api/ai-assessments` | — | Toàn bộ (`POST`, `GET`, `GET /:id` cho thú cưng của mình) | — | — | — | — |

---

## 4. Trace chi tiết 5 Luồng Upload Ảnh (`multer` Memory Storage)

Mọi luồng upload đều dùng `multer.memoryStorage()` (không ghi file tạm ra ổ đĩa server), giới hạn **`5MB/ảnh`**, và luôn đặt **SAU `protect` + `authorize`**:

| # | Middleware (`file:line`) | Tên field & Số ảnh | Định dạng chấp nhận | Cho phép JSON không file? | Kiểm tra Magic Bytes? |
| :-: | :--- | :--- | :--- | :---: | :---: |
| 1 | `uploadGroomingNotePhotos` (`upload.middleware.js:30-33,85-89`) | `beforePhoto` (1), `afterPhoto` (1) | JPEG, PNG, WebP (check `mimetype`) | Có | Không |
| 2 | `uploadSubtaskResultPhoto` (`upload.middleware.js:35,95-99`) | `resultImage` (1) | JPEG, PNG, WebP (check `mimetype`) | Có | Không |
| 3 | `uploadCommunityPostPhotos` (`upload.middleware.js:38-43,115-120`) | `image` (1), `timelineImages` (≤10) | JPEG, PNG, WebP (check `mimetype`) | **Không** (trả **`415`** tại `L117`) | Không |
| 4 | `uploadAiAssessmentPhotos` (`upload.middleware.js:46-55,132-147`) | `photos` hoặc `photo` (tổng ≤3, check gộp tại `L138-144`) | **Chỉ JPG/PNG** (`MSG35`) | Có (0 ảnh gửi JSON) | **Có** (`aiAssessment.validator.js:18-26,137`) |
| 5 | `uploadPhoto` inline (`pet.routes.js:13-37`) | `photo` (1) | JPEG, PNG, WebP | Có (hoặc data URI base64) | **Có** (`pet.validator.js:39-60`: JPEG `FF D8 FF`, PNG 8-byte, WebP `RIFF..WEBP`) |

---

## 5. Các cơ chế Bảo mật & Kiến trúc Xử lý Lỗi

### 5.1. Danh sách chốt bảo mật thực tế trong mã nguồn
1. **Băm mật khẩu & OTP bằng `bcryptjs` + Ẩn khỏi query mặc định:** `User.js:10-12` gắn `select: false` cho `passwordHash`, `emailOtpHash`, `emailOtpExpiresAt`. Login phải chủ động `.select('+passwordHash')` (`auth.service.js:129`).
2. **Chống Brute-Force Login (`BR-03` / `MSG12`):** `auth.service.js:121-166` đếm `failedLoginAttempts`; sai lần 5 → đặt `lockUntil = now + 30 phút` và trả `403`. Khi chưa hết 30 phút, chặn ngay từ đầu ở `L132-137`. Đăng nhập thành công reset về `0` và `null` (`L162-166`). Ngoài ra `L140` gộp chung thông báo lỗi sai user/sai pass để tránh dò tài khoản (Account Enumeration).
3. **Chống IDOR (Ownership Enforcement):** Không bao giờ tin `ownerId`/`role`/`status` từ `req.body`; luôn lấy `req.user.id` từ token và so sánh `String(entity.ownerId) === String(req.user.id)` trong Service (`pet.service.js`, `petTimeline.service.js`, `appointmentOrder.service.js`, `invoice.service.js`, `medicalRecord.service.js`, `inpatientRecord.service.js`, `communityPost.service.js`, `aiAssessment.service.js`).
4. **Chống Mass Assignment:** Mọi Validator/Service chỉ whitelist từng field được phép ghi (vd `UPDATABLE_FIELDS` ở `pet.service.js:82`, không bao giờ chạy `Model.create(req.body)` hay `findByIdAndUpdate(id, req.body)`).
5. **Chống xóa vật lý hồ sơ (`BR-22`):** 3 endpoint `DELETE /api/admin/users/:id` (`admin.controller.js:75-78`), `DELETE /api/services/:id` (`service.controller.js:75-78`), `DELETE /api/invoices/:id` (`invoice.controller.js:294-297`) trả thẳng `403` yêu cầu khóa / chuyển `Inactive`.
6. **Chống giả mạo Webhook & Timing Attack:** `vnpay.js:120` và `payos.js:75` dùng **`crypto.timingSafeEqual`** sau khi so chiều dài chuỗi hex để xác thực chữ ký HMAC-SHA512 / HMAC-SHA256.
7. **Chống XSS trong Email HTML:** `appointmentOrder.service.js:44-50` và `inpatientRecord.service.js:17-18` dùng `escapeHtml()` mã hóa 5 ký tự `& < > " '` trước khi chèn dữ liệu người dùng vào template HTML email.

### 5.2. Kiến trúc Xử lý Lỗi (Global vs Local Controller)
- **Global Error Middleware (`server.js:61-70`):**
  ```js
  const statusCode = err.statusCode || err.status || 500;
  const response = { message: err.message || 'Lỗi hệ thống' };
  if (err.errors) response.errors = err.errors;
  return res.status(statusCode).json(response);
  ```
- **Thực tế 2 nhánh xử lý trong 22 Controller:**
  - **4 controller đẩy về Global Error Handler bằng `next(err)`:** `cageAssignment`, `communityPost`, `communityInteraction`, `communityModeration`.
  - **18 controller tự bắt trong `try/catch` (hàm `handleError` / `handleControllerError`):** đọc `err.statusCode || err.status`, nếu lỗi chưa định nghĩa thì log `console.error` và trả `500`.
  - **Lưu ý nhỏ:** `appointmentOrder.service.js:29` và `inpatientRecord.service.js:13` dùng `err.status`, còn các service khác dùng `err.statusCode` — cả `server.js:62` lẫn các controller đều đọc cả 2 thuộc tính nên hoạt động ổn định.

---

## 6. Ma trận toàn bộ Case Lỗi tầng Middleware & Bảo mật

| Nhóm | Kịch bản kích hoạt cụ thể | HTTP Status | JSON Error Message thực tế | Tọa độ code (`file:line`) |
| :--- | :--- | :-: | :--- | :--- |
| **Auth** | Thiếu header `Authorization` hoặc không có tiền tố `Bearer ` | **`401`** | `{ message: "Chưa đăng nhập" }` | `auth.middleware.js:9-11` |
| **Auth** | Token sai chữ ký, rỗng, bị sửa payload hoặc hết hạn | **`401`** | `{ message: "Token không hợp lệ hoặc đã hết hạn" }` | `auth.middleware.js:16,45-47` |
| **Auth** | Route quên gắn `protect` mà gọi thẳng `authorize` | **`401`** | `{ message: "Chưa đăng nhập" }` | `auth.middleware.js:55-57` |
| **Account** | Token còn hạn nhưng Admin đã đổi `user.status = 'Locked'` | **`403`** | `{ message: "Tài khoản đã bị khoá, vui lòng liên hệ quản trị viên" }` | `auth.middleware.js:30-36` |
| **Account** | Token còn hạn nhưng `user.status = 'Inactive'` | **`403`** | `{ message: "Tài khoản đã ngừng hoạt động" }` | `auth.middleware.js:30-36` |
| **RBAC** | `req.user.role` không thuộc danh sách `allowedRoles` | **`403`** | `{ message: "Bạn không có quyền thực hiện chức năng này" }` | `auth.middleware.js:59-61` |
| **BR-03** | Nhập sai mật khẩu lần thứ 5 liên tiếp | **`403`** | `{ message: "Bạn đã nhập sai mật khẩu 5 lần liên tiếp. Tài khoản bị tạm khóa trong 30 phút" }` | `auth.service.js:145-152` |
| **BR-03** | Đăng nhập lại khi `lockUntil > now` (dù đúng hay sai mật khẩu) | **`403`** | `{ message: "Bạn đã nhập sai mật khẩu quá 5 lần liên tiếp. Tài khoản đang bị tạm khóa trong 30 phút" }` | `auth.service.js:132-137` |
| **BR-22** | Gọi `DELETE` lên Admin User / Service / Invoice | **`403`** | `{ message: "Không được phép xóa vật lý..." }` | `admin.controller.js:76`, `service.controller.js:76`, `invoice.controller.js:295` |
| **Upload** | Gửi JSON vào `POST/PUT /api/community-posts` (yêu cầu multipart) | **`415`** | `{ message: "Chỉ chấp nhận multipart/form-data" }` | `upload.middleware.js:116-118` |
| **Upload** | Ảnh vượt quá `5MB` (`LIMIT_FILE_SIZE`) | **`400`** | `{ message: "Mỗi ảnh tối đa 5MB" }` (hoặc `"Ảnh không được vượt quá 5MB."` ở Pet) | `upload.middleware.js:66,70`, `pet.routes.js:29,32` |
| **Upload** | Sai tên trường file (`LIMIT_UNEXPECTED_FILE`) | **`400`** | `{ message: "Chỉ nhận ảnh qua..." }` | `upload.middleware.js:67,70`, `pet.routes.js:30,32` |
| **Upload** | Gửi quá số ảnh tối đa (`LIMIT_FILE_COUNT` hoặc tổng `photos`+`photo` > 3 ở AI) | **`400`** | `{ message: "Chỉ được tải lên tối đa N ảnh..." }` | `upload.middleware.js:68,140-143` |
| **Upload** | Sai MIME type (vd gửi `.pdf` vào field ảnh) | **`400`** | `{ message: "Ảnh phải có định dạng JPEG, PNG hoặc WebP" }` / `"...JPG hoặc PNG (MSG35)"` | `upload.middleware.js:8-11,72-74` |
| **Upload** | Giả mạo đuôi `.jpg`/mimetype nhưng file thực không đúng Magic Bytes | **`400`** | `{ message: "Dữ liệu không hợp lệ", errors: { photo/photos: "..." } }` | `pet.validator.js:58-60`, `aiAssessment.validator.js:137-138` |
| **Parser** | Body JSON bị lỗi cú pháp (vd thiếu dấu `}`) | **`400`** | `express.json()` ném `SyntaxError` (`err.status = 400`) → Global Error Handler | `server.js:34,61-70` |
| **Routing** | Gọi sai URL / HTTP method không tồn tại | **`404`** | HTML mặc định Express 5 (`Cannot GET /api/...`) — chưa có 404 JSON catch-all | `server.js` (thiếu 404 middleware) |

---

## 7. Điểm nóng kỹ thuật & Bộ câu hỏi phản biện (Defense Q&A)

### Q1: JWT sinh ra là để stateless, tại sao `protect` lại query ngược MongoDB trên MỌI request? Có chậm không?
- **Trả lời (30s):** Theo `UC-62`, khi Admin khóa tài khoản nhân viên nghỉ việc hoặc khóa người dùng vi phạm, hệ thống phải thu hồi quyền truy cập **ngay lập tức**, không thể chờ JWT sống thêm tới 24 giờ (`JWT_EXPIRES_IN=1d`). Em chỉ query theo `_id` (Primary Key có B-Tree index sẵn), dùng `.select('status role').lean()` nên thời gian đọc dưới 1–2ms, hoàn toàn đánh đổi xứng đáng cho tính an toàn.
- **Code chứng minh:** `src/middlewares/auth.middleware.js:18-41`.
- **Hạn chế & cải thiện:** Khi scale lớn hàng chục nghìn req/s, có thể lưu danh sách user bị khóa / blacklist token trên **Redis** thay vì gọi MongoDB mỗi request.

### Q2: Tại sao trong `pet.validator.js` và `aiAssessment.validator.js` phải đọc Magic Bytes của buffer thay vì tin `file.mimetype` của `multer`?
- **Trả lời (30s):** `file.mimetype` chỉ là header `Content-Type` do client gửi lên — kẻ tấn công chỉ cần đổi đuôi file `.exe`/`.html` thành `.jpg` là qua mặt được `fileFilter`. Đọc trực tiếp các byte đầu của binary buffer (`FF D8 FF` cho JPEG, `89 50 4E 47 0D 0A 1A 0A` cho PNG, `RIFF...WEBP` cho WebP) đảm bảo file thực sự là ảnh hợp lệ trước khi gửi lên Cloudinary hoặc AI.
- **Code chứng minh:** `src/validators/pet.validator.js:39-60`, `src/validators/aiAssessment.validator.js:18-26, 137-138`.
- **Hạn chế & cải thiện:** Các luồng upload còn lại (`Grooming`, `Subtask`, `Community`) hiện mới kiểm tra `mimetype` ở `upload.middleware.js:8`, có thể tái sử dụng chung hàm kiểm tra Magic Bytes cho đồng bộ 100%.

### Q3: Chuyện gì xảy ra nếu user đã bị Admin đổi role từ `Receptionist` xuống `PetOwner`, nhưng vẫn cầm token cũ?
- **Trả lời (30s):** `protect` không tin mù quáng `decoded.role` trong token cũ. Ở dòng 38 `auth.middleware.js`, nó ghi đè `req.user.role = user.role || decoded.role` lấy trực tiếp từ MongoDB, nên ở middleware `authorize('Receptionist')` tiếp theo, request bị chặn `403` ngay lập tức.
- **Code chứng minh:** `src/middlewares/auth.middleware.js:26, 38`.

### Q4: Nếu lập trình viên quên gắn `protect` trước `authorize('Admin')` trên một route mới thì có bị bypass không?
- **Trả lời (30s):** Không bị bypass và cũng không crash `Cannot read property 'role' of undefined`. Dòng 55 `auth.middleware.js` kiểm tra phòng thủ `if (!req.user) return res.status(401)...`, và em có hẳn unit test `missing-protect` chứng minh trường hợp này.
- **Code chứng minh:** `src/middlewares/auth.middleware.js:54-57`, `tests/authorize.test.js:12, 40-43`.

### Q5: Hệ thống chống tấn công IDOR (đổi `:id` trên URL để xem dữ liệu người khác) ở đâu?
- **Trả lời (30s):** `authorize()` chỉ lọc theo vai trò (Role), còn chống IDOR nằm ở tầng **Service**: mọi API đọc/sửa/xóa của `PetOwner` (`pets`, `timeline`, `appointment-orders`, `invoices`, `medical-records`, `ai-assessments`, `community-posts`) đều so sánh `ownerId` của bản ghi trong DB với `req.user.id` trích từ JWT, sai là ném `403` (hoặc `404` ở `communityPost` để không lộ sự tồn tại của bài viết).
- **Code chứng minh:** `src/services/aiAssessment.service.js:194-196`, `src/services/communityPost.service.js:51-54`.

### Q6: `server.js` chưa có `helmet` và `express-rate-limit` — nếu bị DDoS hoặc spam OTP/AI thì sao?
- **Trả lời (30s):** Hiện tại hệ thống đã có chốt chống brute-force login 5 lần khóa 30 phút ở tầng DB (`BR-03`), giới hạn payload ảnh 5MB bằng Multer và timeout AI 30s. Em thừa nhận `server.js` chưa gắn `helmet` (security headers) và `express-rate-limit` (chặn spam gửi OTP / spam gọi AI tốn quota). Hướng cải thiện chỉ cần gắn `app.use(helmet())` và `rateLimit` riêng cho `/api/auth`, `/api/users/*/send-otp`, `/api/ai-assessments` mà không ảnh hưởng logic nghiệp vụ.
- **Code chứng minh:** `server.js:29-35`, `src/services/auth.service.js:121-155`.

### Q7: Nếu client gọi `GET /api/khong-ton-tai`, server trả về gì?
- **Trả lời (30s):** Hiện tại `server.js` chưa gắn 404 catch-all middleware trước Global Error Handler nên Express 5 sẽ trả mã `404` kèm trang HTML mặc định `Cannot GET /api/khong-ton-tai`. Có thể bổ sung 3 dòng `app.use((req, res) => res.status(404).json({ message: 'Không tìm thấy đường dẫn API' }))` trước dòng 61 của `server.js` để trả JSON đồng nhất.
- **Code chứng minh:** `server.js:58-70`.

### Q8: Tại sao các endpoint `DELETE` bị cấm (`admin/users`, `services`, `invoices`) lại trả `403 Forbidden` thay vì `405 Method Not Allowed`?
- **Trả lời (30s):** Theo quy chuẩn của nhóm trong SRS (`BR-22`), hệ thống cố ý định nghĩa route `DELETE` để bắt và giải thích rõ bằng tiếng Việt lý do chính sách cấm xóa vật lý (nhằm bảo toàn lịch sử bệnh án và chứng từ tài chính), kèm hướng dẫn dùng thao tác khóa/ngừng hoạt động, và thống nhất dùng `403` cho mọi hành vi bị cấm theo chính sách hệ thống.
- **Code chứng minh:** `src/controllers/admin.controller.js:75-78`, `src/controllers/service.controller.js:75-78`, `src/controllers/invoice.controller.js:294-297`.
