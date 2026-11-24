# 00 — Mục Lục Tài Liệu Bảo Vệ & Kiến Trúc Cốt Lõi (VCMOS_BE)

> **Nguồn chân lý duy nhất (Single Source of Truth):** Mã nguồn thực tế trong `VCMOS_BE/server.js` và `VCMOS_BE/src/` (**115 file `.js`** + `server.js` = **116 file code**, 18 Mongoose Models, 20 Route files, 22 bộ Controller/Service/Validator, 26 Test files, 96 file đặc tả trong `api/`).
> **Mục tiêu:** Tra cứu trong **5 giây** khi hội đồng hỏi code, luồng dữ liệu, DB/Index/Transaction, bảo mật và toàn bộ HTTP status code.

---

## 1. Kế hoạch bộ tài liệu `docs/defense/` (25 file — Độ phủ 100% mã nguồn)

### A. Nhóm Nền Tảng (4 file: `00` → `03`)

| # | File tài liệu | Phạm vi phụ trách | File mã nguồn & test chịu trách nhiệm |
| :-: | :--- | :--- | :--- |
| `00` | `00_INDEX_VA_CORE_ARCHITECTURE.md` *(file này)* | Tổng mục lục, chênh lệch Code vs README, vòng đời Request, bảng HTTP Status & Error Format | `server.js`, `package.json`, `README.md`, `.env.example` |
| `01` | `01_DATABASE_MODELS_INDEXES_TRANSACTIONS.md` | Toàn bộ 18 Schema, quan hệ ERD, Unique Partial Index, Compound Index, `runWithTransaction` + fallback rollback, Seed | `src/config/db.js`, toàn bộ 18 file `src/models/*.js`, `src/utils/transaction.js`, `src/seeds/seed.js` |
| `02` | `02_MIDDLEWARES_SECURITY_ERROR_HANDLING.md` | JWT + DB status check (`protect`), RBAC (`authorize`), 5 luồng Upload (`multer`), IDOR, `BR-03` khóa login, `BR-22` chặn xóa vật lý, chuẩn hóa lỗi | `src/middlewares/auth.middleware.js`, `src/middlewares/upload.middleware.js`, `src/routes/pet.routes.js` (`L13-37`), `server.js` (`L61-70`), `tests/authorize.test.js` |
| `03` | `03_INTEGRATIONS_AND_BACKGROUND_JOBS.md` | Cloudinary (upload & dọn ảnh mồ côi), VNPay HMAC-SHA512, PayOS HMAC-SHA256, Gemini/Groq failover, Nodemailer (3 luồng), `pdfmake`, Cron Job `JOB-01` & catch-up | `src/config/cloudinary.js`, `src/utils/vnpay.js`, `src/utils/payos.js`, `src/utils/gemini.js`, `src/jobs/index.js`, `src/jobs/flagOverdueAppointments.job.js`, `tests/flagOverdueJob.test.js` |

### B. Nhóm Nghiệp Vụ / Route Độc Lập (21 file: `04` → `24`)

| # | File tài liệu | Route Prefix | File mã nguồn (`src/`) & Test (`tests/`) |
| :-: | :--- | :--- | :--- |
| `04` | `04_AUTH.md` | `/api/auth` | `auth.{routes,controller,service,validator}.js` · `tests/auth.test.js`, `tests/services/auth.service.test.js` |
| `05` | `05_USER_PROFILE_OTP_STAFF.md` | `/api/users` | `user.{routes,controller,service,validator}.js` · `tests/user.test.js`, `tests/services/user.service.test.js` |
| `06` | `06_ADMIN_USER_MANAGEMENT.md` | `/api/admin/users` | `admin.{routes,controller,service,validator}.js` · `tests/admin.test.js`, `tests/services/admin.service.test.js` |
| `07` | `07_PET_MANAGEMENT.md` | `/api/pets` *(CRUD)* | `pet.{routes,controller,service,validator}.js` · `tests/services/pet.service.test.js` |
| `08` | `08_UNIFIED_PET_TIMELINE.md` | `/api/pets/:petId/timeline` | `pet.routes.js` (`L47`), `petTimeline.{controller,service,validator}.js` · `tests/petTimeline.test.js` |
| `09` | `09_SERVICE_CATALOG.md` | `/api/services` | `service.{routes,controller,service,validator}.js` · `tests/service.test.js`, `tests/services/service.service.test.js` |
| `10` | `10_PRODUCT_CATALOG.md` | `/api/products` | `product.{routes,controller,service,validator}.js` · `tests/services/product.service.test.js` |
| `11` | `11_CAGE_MANAGEMENT.md` | `/api/cages` | `cage.{routes,controller,service,validator}.js` · `tests/services/cage.service.test.js` |
| `12` | `12_CAGE_ASSIGNMENT.md` | `/api/cage-assignments` | `cageAssignment.{routes,controller,service,validator}.js` · `tests/services/cageAssignment.service.test.js` |
| `13` | `13_STAFF_SCHEDULE_BLACKLIST.md` | `/api/staff-schedules` | `staffSchedule.{routes,controller,service,validator}.js` · `tests/staffSchedule.test.js` |
| `14` | `14_APPOINTMENT_ORDER_ONLINE_CHECKIN.md` | `/api/appointment-orders` | `appointmentOrder.{routes,controller,service,validator}.js` · `tests/appointmentOrder.test.js`, `tests/services/appointmentOrder.service.test.js` |
| `15` | `15_WALK_IN_RECEPTION.md` | `/api/appointment-orders/walk-in` | `walkIn.{routes,controller,service,validator}.js` · `tests/walkIn.test.js` |
| `16` | `16_APPOINTMENT_SERVICE_QUEUE_GROOMING.md` | `/api/appointment-services` | `appointmentService.{routes,controller,service,validator}.js` · `tests/services/appointmentService.service.test.js` |
| `17` | `17_MEDICAL_RECORD_SOAP.md` | `/api/medical-records` | `medicalRecord.{routes,controller,service,validator}.js` · `tests/medicalRecord.test.js` |
| `18` | `18_SUBTASK_REQUEST_PARACLINICAL.md` | `/api/subtask-requests` | `subtaskRequest.{routes,controller,service,validator}.js` · `tests/subtaskRequest.test.js` |
| `19` | `19_INPATIENT_RECORD_DAILY_LOGS.md` | `/api/inpatient-records` | `inpatientRecord.{routes,controller,service,validator}.js` · *(chưa có file test riêng — cần lưu ý)* |
| `20` | `20_INVOICE_PAYMENT_PDF.md` | `/api/invoices` | `invoice.{routes,controller,service,validator}.js`, `vnpay.js`, `payos.js` · `tests/invoice.test.js` |
| `21` | `21_REPORTS_AND_ANALYTICS.md` | `/api/reports` | `report.{routes,controller,service,validator}.js` · `tests/report.test.js` |
| `22` | `22_COMMUNITY_POSTS_AND_INTERACTIONS.md` | `/api/community-posts` | `communityPost.{routes,controller,service,validator}.js`, `communityInteraction.{controller,service,validator}.js` · `tests/communityPost.test.js` |
| `23` | `23_ADMIN_COMMUNITY_MODERATION.md` | `/api/admin/community` | `communityModeration.{routes,controller,service,validator}.js` · `tests/communityPost.test.js` |
| `24` | `24_AI_HEALTH_ASSESSMENT.md` | `/api/ai-assessments` | `aiAssessment.{routes,controller,service,validator}.js`, `gemini.js` · `tests/aiAssessment.test.js` |

> **Kiểm chứng độ phủ 100%:** `server.js` (1) + `src/config/` (2) + `src/models/` (18) + `src/middlewares/` (2) + `src/utils/` (4 JS) + `src/jobs/` (2 JS) + `src/seeds/` (1) + `src/routes/` (20) + `src/controllers/` (22) + `src/services/` (22) + `src/validators/` (22) = **116/116 file `.js`** và **26/26 file test**. *(2 file rỗng `src/jobs/.gitkeep`, `src/utils/.gitkeep` không chứa code).*

---

## 2. Chênh lệch giữa Code thực tế và `README.md` (Điểm hội đồng dễ hỏi xoáy)

| # | Hạng mục | Thực tế trong mã nguồn (`file:line`) | Thiếu / Chưa nêu rõ trong `README.md` |
| :-: | :--- | :--- | :--- |
| 1 | **Upload Middleware tách làm 2 nơi** | `src/middlewares/upload.middleware.js` xử lý 4 luồng (Grooming, Subtask, Community, AI), còn `src/routes/pet.routes.js:13-37` **tự tạo `multer` riêng (`uploadPhoto`)** và kiểm tra định dạng ảnh bằng **magic bytes** trong `pet.validator.js` thay vì `mimetype`. Riêng `uploadCommunityPostPhotos` (`upload.middleware.js:116-118`) ép `multipart/form-data` (trả `415` nếu gửi JSON). | README chỉ nhắc `src/middlewares/upload.middleware.js`, không nêu nhánh `pet.routes.js` và mã `415`. |
| 2 | **Cơ chế `protect` có query DB trên mỗi request** | `src/middlewares/auth.middleware.js:18-41` sau khi `jwt.verify` sẽ gọi `User.findById(decoded.id).select('status role').lean()` để chặn ngay lập tức (`403`) nếu Admin đổi `status` thành `Locked`/`Inactive` và đồng bộ `role` mới nhất từ DB. | README mô tả ngắn là "kiểm tra JWT", dễ làm người nghe tưởng thuần stateless. |
| 3 | **22 bộ Controller/Service nhưng chỉ có 20 Route files** | `pet.routes.js` gộp cả `pet` + `petTimeline` (`L47`); `communityPost.routes.js` gộp cả `communityPost` + `communityInteraction`; `appointmentOrder.routes.js:9` mount lồng `walkIn.routes.js` tại `/walk-in` **trước** `router.use(protect)`. | README chỉ liệt kê danh sách prefix, chưa chỉ ra cấu trúc ghép 2-trong-1 và mount lồng. |
| 4 | **Hạ tầng bảo mật & 404 trong `server.js`** | `server.js` **chưa có** `helmet`, **chưa có** `express-rate-limit`, **chưa có** `express.urlencoded()`, và **chưa có 404 catch-all JSON handler** (URL sai sẽ trả HTML mặc định của Express 5). `cors` (`server.js:32`) chỉ cho phép đúng 1 `origin` duy nhất. | README không nêu các giới hạn hạ tầng này. |
| 5 | **Xử lý lỗi không đồng nhất giữa các Controller** | `server.js:61-70` có Global Error Middleware, nhưng chỉ **4/22 controller** gọi `next(err)` (`cageAssignment`, `communityPost`, `communityInteraction`, `communityModeration`); **18/22 controller** tự `try/catch` và trả JSON trực tiếp. Service cũng dùng 2 tên field lỗi: `err.statusCode` (phổ biến) và `err.status` (`appointmentOrder.service.js:29`). | README ghi chung có Global Error Middleware. |
| 6 | **Các Endpoint đặc thù / Public / Chặn xóa** | - Healthcheck: `GET /` (`server.js:36-38`).<br>- Tạo lịch online có **3 endpoint**: `POST /api/appointment-orders/online`, `POST /api/appointment-services/online`, `POST /api/appointment-services/online-booking`.<br>- Public không cần token: `GET /api/services(/:id)`, `GET /api/products(/:id)`, `POST /api/users/forgot-password` & `/reset-password`, `GET /api/community-posts/public(/:id)` & `/:id/comments`, và 3 callback thanh toán `GET /api/invoices/vnpay-ipn`, `GET /api/invoices/vnpay-return`, `POST /api/invoices/payos-webhook`.<br>- Báo cáo top sản phẩm đặt trong Invoice: `GET /api/invoices/reports/best-selling-products` (`invoice.routes.js:16-21`).<br>- 3 route cố ý trả `403` chặn xóa cứng (`BR-22`): `DELETE /api/admin/users/:id`, `DELETE /api/services/:id`, `DELETE /api/invoices/:id`. | README chỉ tóm tắt theo prefix nên không thấy rõ nhóm public/trùng vai trò/vị trí báo cáo sản phẩm. |
| 7 | **Thư viện cài thừa trong `package.json`** | `package.json:21,24` có khai báo `exceljs` và `moment`, nhưng **0 file trong `src/` import**. Ngược lại `qs` (`package.json:30`) đang được dùng thật trong `src/utils/vnpay.js`. | README đã lược bỏ nhưng `package.json` vẫn còn cài 2 package này. |
| 8 | **Thiếu test riêng cho Nội trú & Seed chưa có `AiAssessment`** | Module `inpatientRecord` chưa có file test riêng trong `tests/` (chỉ được kiểm chứng gián tiếp qua `petTimeline.test.js` và `invoice.test.js`). `src/seeds/seed.js` chỉ seed 17/18 collection (chưa seed `AiAssessment`). | README ghi tổng 26 test suites / 18 collection. |

---

## 3. Vòng đời 1 Request từ đầu đến cuối (`server.js`)

### 3.1. Quá trình khởi động ứng dụng (Bootstrap)

1. **`server.js:2`** — `require('dotenv').config()` nạp `.env` đầu tiên.
2. **`server.js:3`** — `require('./src/config/cloudinary')` kiểm tra ngay 3 biến `CLOUDINARY_*`; **thiếu là crash ngay lúc startup (fail-fast)** tại `src/config/cloudinary.js:6-8`.
3. **`server.js:73-79`** — Kiểm tra `if (require.main === module)`:
   - Khi chạy `npm start` / `npm run dev`: gọi `connectDB()` (`src/config/db.js:3-11`, lỗi kết nối gọi `process.exit(1)`) → `.then(() => startBackgroundJobs())` (`src/jobs/index.js:13-27`: lên lịch cron `JOB-01` lúc `23:59` VN + chạy quét bù bất đồng bộ các ngày `< hôm nay` với `maxRetries: 2`), đồng thời mở cổng `app.listen(process.env.PORT || 9999)`.
   - Khi chạy `npm test` (Jest + Supertest): chỉ `require('../server')` lấy `app` (`server.js:81`), **không** mở port và **không** tự khởi chạy cron job.

### 3.2. Chuỗi xử lý mỗi HTTP Request

```text
Client Request
  │
  ├─► [1] CORS: server.js:32 → cors({ origin: process.env.FRONTEND_URL || 'http://localhost:3000' })
  ├─► [2] Body Parser: server.js:34 → express.json() (JSON sai cú pháp ném SyntaxError err.status=400 → nhảy thẳng bước [8])
  ├─► [3] Healthcheck: server.js:36-38 → GET / trả 200 { message: 'VCMOS API is running' }
  ├─► [4] Router Mounting: server.js:40-58 (19 cụm mount theo thứ tự):
  │         L40 /api/auth                  → auth.routes.js
  │         L41 /api/users                 → user.routes.js
  │         L42 /api/pets                  → pet.routes.js (+ /:petId/timeline)
  │         L43 /api/admin/users           → admin.routes.js
  │         L44 /api/services              → service.routes.js
  │         L45 /api/products              → product.routes.js
  │         L46 /api/appointment-orders    → appointmentOrder.routes.js (L9 lồng /walk-in → walkIn.routes.js)
  │         L47 /api/cages                 → cage.routes.js
  │         L48 /api/cage-assignments      → cageAssignment.routes.js
  │         L49 /api/appointment-services  → appointmentService.routes.js
  │         L50 /api/medical-records       → medicalRecord.routes.js
  │         L51 /api/subtask-requests      → subtaskRequest.routes.js
  │         L52 /api/staff-schedules       → staffSchedule.routes.js
  │         L53 /api/inpatient-records     → inpatientRecord.routes.js
  │         L54 /api/invoices              → invoice.routes.js
  │         L55 /api/community-posts       → communityPost.routes.js
  │         L56 /api/admin/community       → communityModeration.routes.js
  │         L57 /api/reports               → report.routes.js
  │         L58 /api/ai-assessments        → aiAssessment.routes.js
  │
  ├─► [5] Route-level Middlewares (đặt đúng thứ tự bảo mật):
  │         (a) protect (src/middlewares/auth.middleware.js:7-48):
  │             - Không có "Bearer " → 401
  │             - jwt.verify fail → 401
  │             - Query DB User.findById(id).select('status role').lean():
  │               status !== 'Active' → 403 (Locked / Inactive)
  │             - Gán req.user = { id, role } → next()
  │         (b) authorize(...roles) (src/middlewares/auth.middleware.js:52-65):
  │             - Thiếu req.user → 401; role không nằm trong allowedRoles → 403
  │         (c) Upload middleware (chỉ route có ảnh, luôn đặt SAU protect/authorize):
  │             - upload.middleware.js (memoryStorage, <= 5MB/ảnh, check mimetype, lỗi Multer → 400; community không multipart → 415)
  │             - hoặc pet.routes.js:13-37 (uploadPhoto, memoryStorage, <= 5MB, check magic bytes ở pet.validator.js)
  │
  ├─► [6] Controller + Validator (src/controllers/* & src/validators/*):
  │         - Kiểm tra ObjectId params, query filter, body fields
  │         - Fail → trả ngay 400 { message, errors }
  │
  ├─► [7] Service + Model / Transaction / 3rd-party (src/services/*):
  │         - Kiểm tra quyền sở hữu (IDOR), tồn tại (404), trạng thái hợp lệ (409), unique index 11000 → 409
  │         - Thao tác đa bảng đi qua runWithTransaction (src/utils/transaction.js:19-34)
  │         - Trả dữ liệu sạch (không bao giờ trả passwordHash) → Controller gửi 200 / 201 / PDF stream
  │
  └─► [8] Error Handling:
            - Nhánh A (18/22 controller): catch (err) tự đọc err.statusCode || err.status → res.status(...).json({ message, errors? })
            - Nhánh B (4/22 controller + lỗi Express/multer/JSON SyntaxError): next(err) → Global Error Middleware (server.js:61-70):
              statusCode = err.statusCode || err.status || 500
              res.status(statusCode).json({ message: err.message || 'Lỗi hệ thống', ...(err.errors && { errors }) })
            - Route không khớp bất kỳ path nào: Express 5 trả mặc định 404 HTML ("Cannot GET ...") vì chưa gắn 404 JSON middleware.
```

---

## 4. Quy chuẩn HTTP Status Code & Error Format toàn hệ thống

### 4.1. Cấu trúc Response lỗi chuẩn

- **Lỗi chung (`401`, `403`, `404`, `409`, `415`, `500`, `502`, `503`, `504`):**
  ```json
  { "message": "Thông báo lỗi tiếng Việt rõ ràng" }
  ```
- **Lỗi validate dữ liệu đầu vào (`400 Bad Request`):**
  ```json
  {
    "message": "Dữ liệu không hợp lệ",
    "errors": {
      "tenTruong": "Lý do sai chi tiết"
    }
  }
  ```
- **Ngoại lệ đặc biệt không theo JSON `{ message }`:**
  - `GET /api/invoices/vnpay-ipn` (`invoice.controller.js:117-125`): **luôn trả HTTP `200`** theo giao thức VNPay với body `{ "RspCode": "00" | "01" | "02" | "04" | "97" | "99", "Message": "..." }`.
  - `GET /api/invoices/:id/pdf` (`invoice.controller.js:261-263`): trả HTTP `200` dạng binary PDF (`Content-Type: application/pdf`, `Content-Disposition: inline; filename="..."`).

### 4.2. Bảng đầy đủ toàn bộ HTTP Status Code xuất hiện trong mã nguồn

| HTTP Code | Ý nghĩa chuẩn trong VCMOS_BE | Nơi phát sinh thực tế (`file:line` tiêu biểu) |
| :-: | :--- | :--- |
| **`200`** | Thành công (GET, PUT, PATCH, DELETE mềm, hành động thanh toán, VNPay IPN/Return, xuất PDF) | Toàn bộ 22 controller + `server.js:37` |
| **`201`** | Tạo mới tài nguyên thành công (User, Pet, Service, Product, Cage, Order, Walk-in, Subtask, Inpatient, Invoice, Post, Comment, Report, AI Assessment...) | 17 controller có thao tác `POST` tạo bản ghi |
| **`400`** | Lỗi dữ liệu đầu vào (validate body/query/ObjectId, JSON body hỏng cú pháp, lỗi Multer quá 5MB/sai tên trường/sai định dạng ảnh, vi phạm điều kiện tham số nghiệp vụ) | Toàn bộ validators/controllers, `upload.middleware.js:10,70,141`, `pet.routes.js:32` |
| **`401`** | Chưa đăng nhập (thiếu header `Bearer`), JWT hết hạn/sai chữ ký, hoặc sai SĐT/Email/mật khẩu khi login | `auth.middleware.js:10,46,56`, `auth.service.js:156` |
| **`403`** | Không có quyền thực thi: sai `role` (`authorize`), tài khoản `Locked`/`Inactive` (`protect` & login), tạm khóa 30 phút do sai mật khẩu 5 lần (`BR-03`), vi phạm quyền sở hữu (IDOR), hoặc gọi `DELETE` vật lý bị cấm (`BR-22`) | `auth.middleware.js:31,60`, `auth.service.js`, `admin.controller.js:76`, `service.controller.js:76`, `invoice.controller.js:295` và 11 services kiểm tra ownership |
| **`404`** | Không tìm thấy bản ghi theo ID / mã hoặc bản ghi đã bị xóa mềm (`Deleted` / `Removed`) | 18 services (`admin`, `pet`, `appointmentOrder`, `invoice`, `aiAssessment`, ...) |
| **`409`** | Xung đột dữ liệu / trạng thái: trùng Unique Index Mongo `11000` (SĐT, email, mã, tên dịch vụ, like/report trùng, lịch đã lập hóa đơn), sai máy trạng thái (vd: check-in đơn không phải `Booked`, xuất viện ca đã xuất, sửa bài không còn `PendingReview`), chuồng đang có thú cưng, nhân viên trùng lịch bận | 17 services |
| **`415`** | Sai `Content-Type`: tạo/cập nhật bài viết cộng đồng nhưng không gửi `multipart/form-data` | `upload.middleware.js:117` |
| **`500`** | Lỗi nội bộ không lường trước (DB disconnect, cạn 5 lần retry sinh mã trùng, exception chưa gán `statusCode`) | `server.js:62`, 12 controllers fallback, các vòng retry sinh mã trong services |
| **`502`** | Lỗi dịch vụ bên thứ 3 (upstream): upload ảnh lên Cloudinary thất bại hoặc PayOS API trả lỗi/không phản hồi hợp lệ | `aiAssessment.service.js:58`, `communityPost.service.js:31`, `subtaskRequest.service.js:297`, `appointmentService.service.js`, `utils/payos.js:99` |
| **`503`** | Dịch vụ AI tạm quá tải / chưa cấu hình API key / tất cả key Gemini & Groq đều lỗi hoặc hết quota (`MSG34`) | `src/utils/gemini.js:573,657` |
| **`504`** | Quá thời gian chờ (Timeout) khi gọi cổng ngoài: Google Gemini / Groq vượt ngân sách timeout `<= 30s`, hoặc gọi PayOS API timeout | `src/utils/gemini.js:435,512,657`, `src/utils/payos.js:107` |
