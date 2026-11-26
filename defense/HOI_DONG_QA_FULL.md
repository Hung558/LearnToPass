# BỘ CÂU HỎI PHẢN BIỆN HỘI ĐỒNG BẢO VỆ BACKEND — VCMOS_BE (FULL 74 Q&A)

> **Nguồn chân lý duy nhất (Single Source of Truth):** Toàn bộ mã nguồn thực tế của `VCMOS_BE` (`server.js` + `src/` = **116 file `.js`**, **18 Mongoose Models**, **20 file Routes**, **22 bộ Controller/Service/Validator**, **6 tích hợp bên thứ 3**, **Cron Job `JOB-01`**, **26 bộ Jest Test**).
> **Cách dùng trong phòng bảo vệ (`Ctrl + F` trong 5 giây):** Bấm `Ctrl + F`, gõ tên module (vd: `PayOS`, `SOAP`, `Walk-in`, `BR-11`, `JOB-01`, `IDOR`, `11000`, `409`, `Transaction`) hoặc nhảy thẳng theo Mục lục bên dưới.

---

## MỤC LỤC TRA CỨU NHANH

| Phần | Chủ đề chuyên sâu | Phạm vi câu hỏi | Từ khóa `Ctrl + F` nhanh |
| :-: | :--- | :-: | :--- |
| **[Phần 1](#phần-1-kiến-trúc-tổng-thể-tech-stack--vòng-đời-1-request)** | Kiến trúc tổng thể, Tech Stack & Vòng đời 1 Request | `Q1 – Q7` (7 câu) | `server.js`, `4 tầng`, `MongoDB vs SQL`, `require.main`, `YYYY-MM-DD` |
| **[Phần 2](#phần-2-database-design-schema-indexing--transaction)** | Database Design, Schema, Indexing & Transaction | `Q8 – Q17` (10 câu) | `18 models`, `partialFilterExpression`, `snapshot`, `runWithTransaction`, `seed` |
| **[Phần 3](#phần-3-xác-thực-auth-phân-quyền-rbac--bảo-mật)** | Xác thực (Auth), Phân quyền (RBAC) & Bảo mật | `Q18 – Q28` (11 câu) | `BR-03`, `protect`, `authorize`, `OTP`, `IDOR`, `Mass Assignment`, `Magic Bytes`, `BR-22` |
| **[Phần 4](#phần-4-truy-vấn-sâu-từng-luồng-nghiệp-vụ--state-machine)** | Truy vấn sâu 100% Luồng Nghiệp vụ & State Machine (20/20 Route files) | `Q29 – Q49` (21 câu) | `users`, `admin`, `pets`, `timeline`, `services`, `products`, `cages`, `schedules`, `walk-in`, `SOAP`, `subtask`, `inpatient`, `invoice`, `reports`, `community`, `AI` |
| **[Phần 5](#phần-5-tích-hợp-bên-thứ-3--background-jobs)** | Tích hợp bên thứ 3 & Background Jobs | `Q50 – Q58` (9 câu) | `PayOS`, `VNPay`, `Idempotency`, `PDFMake`, `Cloudinary`, `Nodemailer`, `Gemini`, `Groq`, `JOB-01` |
| **[Phần 6](#phần-6-api-contract-ma-trận-http-status-code--xử-lý-lỗi)** | API Contract, Ma trận HTTP Status Code & Xử lý lỗi | `Q59 – Q64` (6 câu) | `400`, `401`, `403`, `404`, `409`, `415`, `502`, `503`, `504`, `CastError`, `11000`, `MulterError` |
| **[Phần 7](#phần-7-câu-hỏi-bắt-lỗi-trực-tiếp-trên-code-thật-điểm-yếu-race-condition--nợ-kỹ-thuật)** | "Bắt lỗi trực tiếp trên Code thật" — 10 Điểm yếu & Nợ kỹ thuật | `Q65 – Q74` (10 câu) | `helmet`, `rate-limit`, `404 HTML`, `ảnh rác`, `cage.validator`, `stale index`, `inpatient test` |

### Bảng nhảy nhanh theo 20 Route Files trong `src/routes/`

| Route Prefix | File Route | Câu hỏi liên quan |
| :--- | :--- | :--- |
| `/api/auth` | `src/routes/auth.routes.js` | `Q18`, `Q19`, `Q21` |
| `/api/users` | `src/routes/user.routes.js` | `Q22`, `Q29` |
| `/api/admin/users` | `src/routes/admin.routes.js` | `Q28`, `Q30` |
| `/api/pets` & `/:petId/timeline` | `src/routes/pet.routes.js` | `Q24`, `Q27`, `Q31`, `Q32` |
| `/api/services` | `src/routes/service.routes.js` | `Q16`, `Q28`, `Q33` |
| `/api/products` | `src/routes/product.routes.js` | `Q34` |
| `/api/cages` | `src/routes/cage.routes.js` | `Q35`, `Q69` |
| `/api/cage-assignments` | `src/routes/cageAssignment.routes.js` | `Q10`, `Q36`, `Q70` |
| `/api/staff-schedules` | `src/routes/staffSchedule.routes.js` | `Q37`, `Q72` |
| `/api/appointment-orders` | `src/routes/appointmentOrder.routes.js` | `Q38`, `Q39`, `Q58` |
| `/api/appointment-orders/walk-in` | `src/routes/walkIn.routes.js` | `Q15`, `Q40` |
| `/api/appointment-services` | `src/routes/appointmentService.routes.js` | `Q41`, `Q54` |
| `/api/medical-records` | `src/routes/medicalRecord.routes.js` | `Q11`, `Q42` |
| `/api/subtask-requests` | `src/routes/subtaskRequest.routes.js` | `Q43`, `Q54` |
| `/api/inpatient-records` | `src/routes/inpatientRecord.routes.js` | `Q44`, `Q55`, `Q73` |
| `/api/invoices` | `src/routes/invoice.routes.js` | `Q11`, `Q45`, `Q46`, `Q50`, `Q51`, `Q52`, `Q53` |
| `/api/reports` | `src/routes/report.routes.js` | `Q12`, `Q13`, `Q47` |
| `/api/community-posts` | `src/routes/communityPost.routes.js` | `Q48`, `Q67`, `Q71` |
| `/api/admin/community` | `src/routes/communityModeration.routes.js` | `Q16`, `Q49` |
| `/api/ai-assessments` | `src/routes/aiAssessment.routes.js` | `Q27`, `Q49`, `Q56`, `Q57`, `Q67` |

---

## Phần 1: Kiến trúc tổng thể, Tech Stack & Vòng đời 1 Request

### Q1. [Kiến trúc / Tổng quan] [Mức độ: Cơ bản] Tại sao nhóm chia kiến trúc thành `routes → validators → controllers → services → models` và tuyệt đối không viết logic trong Mongoose pre-save hook?
- **Ý đồ giám khảo**: Kiểm tra xem sinh viên hiểu rõ Separation of Concerns, khả năng test độc lập từng tầng, và có kiểm soát được side-effect trong ORM/ODM không.
- **Trả lời miệng (30–45 giây)**: Bọn em chia rõ 5 trách nhiệm: `routes` định nghĩa URL và gắn middleware auth/upload; `validators` chặn dữ liệu bẩn ngay ở biên để trả `400`; `controllers` chỉ đọc `req` và gửi `res`; toàn bộ nghiệp vụ, kiểm tra IDOR, transaction nằm tập trung ở `services`; còn `models` chỉ định nghĩa schema và index. Bọn em cố ý không dùng Mongoose pre-save hook vì các thao tác như `findOneAndUpdate`/`updateOne` bỏ qua hook, dễ tạo bug ngầm và khó trace khi debug hoặc unit test service.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `server.js:40-58` (mount 19 cụm router) → `src/routes/pet.routes.js:44-52` → `src/controllers/pet.controller.js:24-39` → `src/validators/pet.validator.js:116-143` → `src/services/pet.service.js:77-115` → `src/models/Pet.js:1-22`.
  - Happy Path: Controller trả `200`/`201` với dữ liệu đã làm sạch từ Service.
  - Fail / Edge Cases liên quan: Lỗi validate trả `400 { message, errors }` ngay tại `validators/`; lỗi phân quyền trả `401`/`403` tại `auth.middleware.js:10,31,60`; lỗi nghiệp vụ ném từ `services/` về controller hoặc Global Error Middleware (`server.js:61-70`).
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Nói `validators/` chỉ validate biên, sao `appointmentService.validator.js` lại có query DB bên trong?"
  - *Cách trả lời*: Em xác nhận đúng — riêng `src/validators/appointmentService.validator.js` (635 dòng) có query DB kiểm tra lịch bận/thứ tự dịch vụ vì module đó do thành viên khác viết theo kiểu gộp pre-check vào validator. Các module khác giữ validator thuần kiểm tra cú pháp/kiểu dữ liệu; nếu chuẩn hóa thêm em sẽ đẩy phần query DB của `appointmentService.validator.js` xuống tầng `service` cho đồng nhất 100%.

### Q2. [Kiến trúc / Database] [Mức độ: Nâng cao] Nghiệp vụ phòng khám thú y có rất nhiều quan hệ (`User → Pet → Order → Service → MedicalRecord → Invoice`), tại sao nhóm chọn MongoDB/Mongoose thay vì SQL (PostgreSQL/MySQL)?
- **Ý đồ giám khảo**: Đánh giá tư duy chọn công nghệ có căn cứ (trade-off), không phải "thấy thích thì dùng".
- **Trả lời miệng (30–45 giây)**: Trong bài toán phòng khám thú y, nhiều nghiệp vụ có cấu trúc tài liệu lồng nhau và cần lưu **snapshot bất biến** tại thời điểm phát sinh: ví dụ `MedicalRecord` nhúng mảng `prescriptions`, `InpatientRecord` nhúng mảng `dailyLogs`, `Invoice` nhúng mảng `items` snapshot giá, `CommunityPost` nhúng mảng `timelineItems`, và `AiAssessment` lưu JSON linh hoạt (`petSnapshot`, `recommendedServices`, `clinicalContextSummary`). MongoDB cho phép đọc trọn 1 hồ sơ trong 1 document thay vì JOIN 5–6 bảng, đồng thời vẫn hỗ trợ Multi-Document ACID Transaction trên MongoDB Atlas (`ReplicaSetWithPrimary`) cho các luồng nhạy cảm như Check-in, Walk-in và Thanh toán.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/models/Invoice.js:3-10` (subdoc `items`), `src/models/MedicalRecord.js:3-11` (`prescriptions`), `src/models/InpatientRecord.js:3-9` (`dailyLogs`), `src/models/AiAssessment.js:3-46` (`petSnapshot`), `src/utils/transaction.js:7-34` (ACID Transaction).
  - Happy Path: Đọc 1 hóa đơn hay 1 bệnh án chỉ cần 1 truy vấn chính không phải JOIN nhiều bảng con.
  - Fail / Edge Cases liên quan: Trade-off lớn nhất là MongoDB không có Foreign Key constraint tự động như SQL — nhóm bù bằng việc kiểm tra tồn tại ở Service (`404`), khóa trùng bằng Unique Partial Index (`E11000 → 409`) và cấm xóa cứng (`BR-22 → 403`).
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Vậy nếu làm lại cho chuỗi 50 phòng khám lớn đòi hỏi báo cáo kế toán phức tạp, anh có giữ MongoDB không?"
  - *Cách trả lời*: Với hệ thống kế toán đa chi nhánh cần JOIN phức tạp, constraint chặt và báo cáo OLAP nặng, PostgreSQL (kết hợp cột `JSONB` cho các snapshot như SOAP/AI) sẽ là lựa chọn tốt hơn. Ở phạm vi đồ án 1 phòng khám, MongoDB Atlas giúp tốc độ phát triển nhanh và xử lý rất gọn các hồ sơ y tế/AI bán cấu trúc.

### Q3. [Kiến trúc / Lifecycle] [Mức độ: Cơ bản] Hãy trace toàn bộ vòng đời 1 HTTP Request từ lúc chạm vào `server.js` đến khi trả Response về client?
- **Ý đồ giám khảo**: Xem sinh viên có nắm thứ tự chạy middleware của Express 5 hay chỉ viết rời rạc.
- **Trả lời miệng (30–45 giây)**: Một request vào `server.js` đi qua đúng 7 trạm: (1) `cors` kiểm tra `FRONTEND_URL`; (2) `express.json()` parse body JSON; (3) khớp đường dẫn vào 1 trong 19 cụm router gắn ở `server.js:40-58`; (4) qua `protect` xác thực JWT + kiểm tra trạng thái `Active` trong DB; (5) qua `authorize(...roles)` kiểm tra vai trò; (6) nếu có ảnh thì qua `multer` memoryStorage; (7) vào `controller → validator → service → MongoDB` rồi trả JSON `200`/`201`. Nếu có lỗi, 18 controller tự bắt trả JSON, còn 4 controller và các lỗi Express/JSON hỏng đẩy qua `next(err)` vào Global Error Middleware ở `server.js:61-70`.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `server.js:32` (`cors`) → `server.js:34` (`express.json`) → `server.js:36-38` (`GET /`) → `server.js:40-58` (19 `app.use`) → `src/middlewares/auth.middleware.js:7-65` (`protect`, `authorize`) → `src/middlewares/upload.middleware.js:60-147` → `controller/validator/service` → `server.js:61-70` (Global Error Middleware).
  - Happy Path: `GET /` trả `200 { message: 'VCMOS API is running' }`; các API nghiệp vụ trả `200`/`201` (hoặc binary PDF ở `GET /api/invoices/:id/pdf`).
  - Fail / Edge Cases liên quan: Body JSON sai cú pháp → `express.json()` ném `SyntaxError` (`err.status = 400`) rơi vào `server.js:61-70` → `400`; thiếu token → `401` (`auth.middleware.js:10`); sai role → `403` (`auth.middleware.js:60`).
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Nếu client gửi request tới URL không tồn tại `GET /api/abc` thì có đi vào Global Error Middleware ở `server.js:61` không?"
  - *Cách trả lời*: Không ạ, vì `server.js` hiện chưa gắn 404 catch-all middleware trước dòng 61, nên Express 5 sẽ trả `404` dạng HTML mặc định (`Cannot GET /api/abc`). Em đã ghi nhận đây là điểm cần bổ sung 3 dòng middleware trả `404` JSON cho đồng nhất.

### Q4. [Kiến trúc / Bootstrap & Test] [Mức độ: Nâng cao] Tại sao `server.js:73` lại bọc `connectDB()` và `app.listen()` trong `if (require.main === module)`? Khi chạy `npm test` vì sao Cron Job `JOB-01` không làm treo Jest?
- **Ý đồ giám khảo**: Kiểm tra hiểu biết về Node.js module loading và kỹ thuật tách app khỏi network/background job để chạy integration test sạch với Supertest.
- **Trả lời miệng (30–45 giây)**: Khi chạy thật bằng `node server.js`, `require.main === module` là `true`, nên server mới nối MongoDB Atlas, khởi động Cron Job `JOB-01` và mở cổng `PORT`. Còn khi Jest/Supertest `require('../server')` để test API, biểu thức đó là `false` nên chỉ lấy object `app` thuần túy mà không mở port (tránh lỗi `EADDRINUSE`), không nối nhầm DB Atlas (mỗi file test tự nối `vcmos_test` local) và không chạy timer Cron Job ngầm gây treo tiến trình Jest (`open handles`).
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `server.js:73-81` (`if (require.main === module)` → `connectDB().then(() => startBackgroundJobs())` → `app.listen`) → `package.json:9,32-34` (`"test": "jest --runInBand"`, `roots: ["<rootDir>/tests"]`).
  - Happy Path: `npm test` chạy tuần tự 26 file test trên `vcmos_test` và tự thoát sạch.
  - Fail / Edge Cases liên quan: Riêng bài test kiểm tra đăng ký cron (`tests/flagOverdueJob.test.js:276-282`) chủ động gọi `task.stop()` ngay sau khi assert để không rò rỉ handle.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Tại sao `package.json:9` bắt buộc chạy `jest --runInBand` mà không chạy song song cho nhanh?"
  - *Cách trả lời*: Vì toàn bộ 26 file test dùng chung 1 database local `vcmos_test` và tự dọn dữ liệu trong `beforeAll`/`afterAll`. Nếu chạy song song nhiều worker, file test A đang insert dữ liệu sẽ bị file test B chạy `deleteMany` xóa mất gây lỗi ngẫu nhiên (flaky test).

### Q5. [Kiến trúc / Routing] [Mức độ: Nâng cao] Codebase có 22 bộ Controller/Service nhưng chỉ có 20 file Route — tại sao? Và thứ tự khai báo route trong Express ảnh hưởng gì?
- **Ý đồ giám khảo**: Kiểm tra xem sinh viên có thực sự thuộc cấu trúc code của mình và hiểu cơ chế route shadowing trong Express hay không.
- **Trả lời miệng (30–45 giây)**: Sở dĩ 22 bộ nhưng 20 file route là vì 2 cụm chia sẻ cùng prefix RESTful: `pet.routes.js` gắn cả `pet` lẫn `petTimeline` (`GET /:petId/timeline`), còn `communityPost.routes.js` gắn cả `communityPost` lẫn `communityInteraction` (like/comment/report). Ngoài ra `appointmentOrder.routes.js:9` mount lồng `walkIn.routes.js` tại `/walk-in` ngay trên đầu. Về thứ tự route, các path tĩnh như `/public`, `/me`, `/walk-in`, `/flag-overdue`, `/reports/best-selling-products` bắt buộc phải khai báo **trước** `/:id`, nếu đặt sau sẽ bị Express nuốt nhầm chữ `"me"` hay `"public"` làm `:id` và báo lỗi `400 Invalid ObjectId`.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/pet.routes.js:47` (`/:petId/timeline`), `src/routes/appointmentOrder.routes.js:9,16-23` (`/walk-in` trước `protect`, `/me` và `/flag-overdue` trước `/:id`), `src/routes/communityPost.routes.js:11-36` (`/public`, `/public/:id`, `/me` trước `/:id`), `src/routes/invoice.routes.js:16-30` (`/reports/best-selling-products`, `/me` trước `/:id`).
  - Happy Path: Request khớp đúng route tĩnh hoặc route động tương ứng.
  - Fail / Edge Cases liên quan: Nếu đặt `GET /:id` trước `GET /me`, mọi request `/me` sẽ rơi vào validator ObjectId và trả `400`.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Tại sao `appointmentOrder.routes.js:9` lại mount `/walk-in` TRƯỚC dòng `router.use(protect)`?"
  - *Cách trả lời*: Vì `walkIn.routes.js` tự gắn `protect` và phân quyền riêng cho từng endpoint bên trong (`GET /customer-lookup` cho `Receptionist` + `Admin`, còn `POST /` chỉ cho `Receptionist`). Đặt trước `router.use(protect)` tránh chạy `protect` lặp 2 lần (tốn thêm 1 lần query DB kiểm tra user status) trên cùng một request Walk-in.

### Q6. [Kiến trúc / Startup Fail-Fast] [Mức độ: Cơ bản] Nếu deploy lên server mới mà thiếu `MONGO_URI` hoặc thiếu 1 trong 3 biến `CLOUDINARY_*` thì server phản ứng thế nào?
- **Ý đồ giám khảo**: Kiểm tra tư duy Fail-Fast vs Silent Failure khi triển khai thực tế.
- **Trả lời miệng (30–45 giây)**: Cả 2 đều áp dụng nguyên tắc **Fail-Fast**: `server.js:3` nạp `src/config/cloudinary.js` ngay khi khởi động — nếu thiếu bất kỳ biến `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY` hay `CLOUDINARY_API_SECRET` nào sẽ ném `Error` dừng tiến trình ngay lập tức. Tương tự, `connectDB()` trong `src/config/db.js:8-9` nếu không nối được MongoDB sẽ in log lỗi và gọi `process.exit(1)` để hệ thống không chạy trong trạng thái "chết lâm sàng".
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `server.js:2-3` → `src/config/cloudinary.js:6-8` (throw `Error`) → `server.js:74` → `src/config/db.js:3-11` (`process.exit(1)`).
  - Happy Path: Đủ biến môi trường → kết nối MongoDB thành công → khởi chạy `JOB-01` và lắng nghe cổng `9999` (hoặc `process.env.PORT`).
  - Fail / Edge Cases liên quan: Ngược lại, các dịch vụ không chặn khởi động như PayOS (`src/utils/payos.js:18-24` → `500`), Gemini/Groq (`src/utils/gemini.js:572-574` → `503`), Nodemailer (`user.service.js:30-35` → fallback `console.log`) được kiểm tra lười (lazy) tại thời điểm gọi để server vẫn phục vụ được các nghiệp vụ cốt lõi khác.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Thế nếu quên cấu hình `JWT_SECRET` thì sao?"
  - *Cách trả lời*: Hiện tại `JWT_SECRET` chưa kiểm tra lúc startup như Cloudinary; khi gọi `jwt.sign` hoặc `jwt.verify` (`auth.service.js`, `auth.middleware.js:16`) sẽ ném lỗi. Trong production nên bổ sung 1 hàm validate toàn bộ env bắt buộc (`MONGO_URI`, `JWT_SECRET`, `CLOUDINARY_*`) ngay đầu `server.js`.

### Q7. [Kiến trúc / Tech Stack] [Mức độ: Nâng cao] Tại sao hệ thống lưu trường `date` của lịch hẹn, lịch bận nhân sự và nhật ký nội trú dưới dạng chuỗi `'YYYY-MM-DD'` thay vì kiểu `Date` của MongoDB?
- **Ý đồ giám khảo**: Kiểm tra xem sinh viên có hiểu bẫy múi giờ UTC vs UTC+7 (Việt Nam) trong hệ thống đặt lịch không.
- **Trả lời miệng (30–45 giây)**: Nghiệp vụ phòng khám vận hành theo **ngày lịch địa phương Việt Nam (UTC+7)**, không cần độ chính xác đến mili-giây. Nếu lưu kiểu `Date` (MongoDB lưu UTC), một lịch khám tạo lúc `01:00` sáng giờ VN sẽ bị đẩy về ngày hôm trước theo UTC, rất dễ gây lệch ca trực, lệch báo cáo theo ngày và làm Cron Job `23:59` quét sai ngày. Lưu chuỗi ISO `YYYY-MM-DD` (có regex + kiểm tra ngày lịch thật như chặn `2026-02-31`) vừa không bao giờ lệch múi giờ, vừa so sánh được bằng `$gte`/`$lte` theo thứ tự từ điển, vừa làm key Compound Index cực gọn.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/models/AppointmentOrder.js:19`, `src/models/StaffSchedule.js:9`, `src/models/InpatientRecord.js:4`, `src/jobs/flagOverdueAppointments.job.js:23-27,43-47` (`isValidDateString`).
  - Happy Path: Mọi query theo ngày (`{ date: '2026-05-20' }` hoặc `{ date: { $lte: targetDate } }`) khớp chính xác ngày VN và tận dụng Compound Index.
  - Fail / Edge Cases liên quan: Truyền ngày sai định dạng hoặc ngày ảo như `2026-02-31` → validator chặn trả `400`.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Thế còn các mốc thời gian chính xác như `paidAt`, `checkedInAt`, `createdAt` thì lưu kiểu gì?"
  - *Cách trả lời*: Các mốc sự kiện cần tính thời lượng (thời gian chờ trung bình, hiệu lực OTP 5 phút, khóa login 30 phút, doanh thu theo khoảng thời gian) vẫn lưu kiểu `Date` chuẩn UTC bình thường, và khi nhóm theo ngày trên báo cáo (`report.service.js`) mới quy đổi về timezone `Asia/Ho_Chi_Minh`.

---

## Phần 2: Database Design, Schema, Indexing & Transaction

### Q8. [Database / Schema] [Mức độ: Cơ bản] Hệ thống có bao nhiêu Model/Collection? Hãy trình bày nhanh các nhóm thực thể chính và quan hệ giữa chúng?
- **Ý đồ giám khảo**: Kiểm tra sinh viên có nắm bức tranh tổng thể ERD của dự án mình không.
- **Trả lời miệng (30–45 giây)**: Hệ thống gồm đúng **18 Mongoose Models** ánh xạ 18 collection `snake_case`, chia làm 5 nhóm: (1) Danh mục & Con người: `User`, `Pet`, `Service`, `Product`, `StaffSchedule`, `Cage`; (2) Vận hành tiếp nhận & khám: `AppointmentOrder` (1-N `AppointmentService`), `CageAssignment`, `MedicalRecord` (1-1 với `AppointmentService`), `SubtaskRequest`, `InpatientRecord` (nhúng `dailyLogs`); (3) Tài chính: `Invoice` (liên kết `AppointmentOrder` và/hoặc `InpatientRecord`); (4) Cộng đồng: `CommunityPost`, `PostLike`, `PostComment`, `ContentReport`; và (5) AI độc lập: `AiAssessment` tách riêng khỏi `MedicalRecord` theo `BR-20`.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: Toàn bộ 18 file trong `src/models/` (`User.js`, `Pet.js`, `Service.js`, `Product.js`, `StaffSchedule.js`, `Cage.js`, `CageAssignment.js`, `AppointmentOrder.js`, `AppointmentService.js`, `MedicalRecord.js`, `SubtaskRequest.js`, `InpatientRecord.js`, `Invoice.js`, `CommunityPost.js`, `PostLike.js`, `PostComment.js`, `ContentReport.js`, `AiAssessment.js`).
  - Happy Path: Các quan hệ tham chiếu chéo bằng `ObjectId` kết hợp index khóa ngoại (`ownerId`, `petId`, `orderId`, `appointmentServiceId`).
  - Fail / Edge Cases liên quan: Không dùng cascade delete vật lý; xóa dữ liệu quan trọng bị chặn `403` (`BR-22`) hoặc chỉ soft delete (`Pet.status = 'Deleted'`, `CommunityPost.status = 'Removed'`).
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Trong 18 bảng đó, quan hệ nào là 1-1 tuyệt đối ở tầng DB?"
  - *Cách trả lời*: Có 3 quan hệ 1-1 bị khóa cứng bằng Unique Index ở tầng DB: (1) `MedicalRecord.appointmentServiceId` unique (`MedicalRecord.js:15-19`); (2) `Invoice.orderId` unique partial (`Invoice.js:33-36`); và (3) `Invoice.inpatientRecordId` unique partial (`Invoice.js:37-40`).

### Q9. [Database / Partial Index] [Mức độ: Cực gắt] Tại sao trên `User.email`, `CageAssignment.cageId`, `InpatientRecord.petId` và `Invoice` nhóm dùng `partialFilterExpression` thay vì `unique: true, sparse: true`?
- **Ý đồ giám khảo**: Đây là câu hỏi phân loại sinh viên "học vẹt" vs sinh viên "hiểu thật MongoDB".
- **Trả lời miệng (30–45 giây)**: Trong Mongoose, khi field có `default: null` (ví dụ `email: null`, `orderId: null`, `releasedAt: null`), document lưu xuống MongoDB thực sự **có chứa key với giá trị `null`**. Index `sparse: true` chỉ bỏ qua document *vắng mặt key* (`undefined`), còn giá trị `null` vẫn bị đánh index nên ngay từ bản ghi thứ 2 có `null` sẽ nổ `E11000`. Dùng `partialFilterExpression` (như `{ email: { $type: 'string' } }`, `{ releasedAt: { $type: 'null' } }`, `{ status: 'Admitted' }`, `{ orderId: { $type: 'objectId' } }`) ép MongoDB chỉ áp ràng buộc unique đúng tập document thỏa điều kiện.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/models/User.js:26` (`email`), `src/models/CageAssignment.js:15-18` (`cageId` khi `releasedAt: null`), `src/models/InpatientRecord.js:28-31` (`petId` khi `status: 'Admitted'`), `src/models/Invoice.js:33-48` (4 partial unique index cho `orderId`, `inpatientRecordId`, `vnpayTxnRef`, `payosOrderCode`).
  - Happy Path: Tạo hàng loạt user không có email (`null`) hoặc lập nhiều hóa đơn bán lẻ (`orderId: null`) đều thành công `201`.
  - Fail / Edge Cases liên quan: Tạo 2 user trùng email thật → `E11000` → `409`; tạo 2 hóa đơn cho cùng 1 `orderId` → `E11000` → `409`; nhập viện 2 đợt `Admitted` cùng lúc cho 1 thú cưng → `E11000` → `409`.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Điều gì xảy ra nếu 1 chuồng đã được giải phóng (`releasedAt` có ngày giờ) rồi sau đó xếp thú cưng khác vào chính chuồng đó?"
  - *Cách trả lời*: Bản ghi cũ có `releasedAt` kiểu `Date` (không còn `$type: 'null'`) nên tự động thoát khỏi tập partial index của `CageAssignment.js:15-18`. Bản ghi mới với `releasedAt: null` insert bình thường mà vẫn giữ nguyên 100% lịch sử sử dụng chuồng trước đó.

### Q10. [Database / Concurrency] [Mức độ: Cực gắt] Làm sao đảm bảo 2 lễ tân bấm xếp cùng 1 chuồng cho 2 thú cưng tại cùng 1 mili-giây không tạo ra double-booking?
- **Ý đồ giám khảo**: Kiểm tra kỹ năng xử lý Race Condition ở tầng ứng dụng kết hợp tầng DB.
- **Trả lời miệng (30–45 giây)**: Bọn em khóa 2 lớp đồng thời. Lớp 1 ở ứng dụng: dùng lệnh nguyên tử `Cage.findOneAndUpdate({ _id: cageId, status: 'Available' }, { $set: { status: 'Occupied' } })` — MongoDB khóa mức document nên chỉ 1 trong 2 request thắng, request còn lại nhận `null` và trả `409`. Lớp 2 ở tầng DB: Unique Partial Index `{ cageId: 1 }` với điều kiện `{ releasedAt: { $type: 'null' } }` trên `cage_assignments`; kể cả có ai đó chèn thẳng vào `CageAssignment` thì lệnh thứ 2 vẫn bị MongoDB chặn `E11000` → `409`.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/services/cageAssignment.service.js:50-63` (`findOneAndUpdate` nguyên tử trên `Cage`) → `src/services/cageAssignment.service.js:80-97` (`CageAssignment.create` + rollback `Cage` về `Available` nếu lỗi) → `src/models/CageAssignment.js:15-18`.
  - Happy Path: Request đầu tiên nhận `201` kèm `CageAssignment`, chuồng chuyển sang `Occupied`.
  - Fail / Edge Cases liên quan: Request thứ 2 nhận `409` (`'Chuồng không ở trạng thái trống (Available)'` ở `L62`, hoặc `'Chuồng đã có phân công đang hoạt động'` ở `L94`); nếu `cageId` không tồn tại trả `404` (`L61`).
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Nếu `Cage.findOneAndUpdate` ở dòng 50 thành công chuyển sang `Occupied`, nhưng bước kiểm tra cân nặng `BR-11` ở dòng 66-77 bị fail thì chuồng có bị kẹt mãi ở `Occupied` không?"
  - *Cách trả lời*: Không ạ, ngay tại dòng 75 (`cageAssignment.service.js:75`) khi `!check.allowed`, code chủ động `Cage.updateOne({ _id: cageId }, { $set: { status: 'Available' } })` để hoàn trả chuồng về `Available` rồi mới ném `400`. Tương tự nếu `CageAssignment.create` lỗi ở dòng 88-90 cũng tự hoàn trả `Available`.

### Q11. [Database / Snapshot] [Mức độ: Nâng cao] Tại sao trong `Invoice.items`, `MedicalRecord.prescriptions` và `AiAssessment.petSnapshot` nhóm lưu bản sao tên, giá, đơn vị thay vì chỉ lưu `ObjectId` rồi `populate`?
- **Ý đồ giám khảo**: Kiểm tra tư duy thiết kế dữ liệu lịch sử/chứng từ tài chính (Immutability).
- **Trả lời miệng (30–45 giây)**: Vì danh mục dịch vụ, giá thuốc và cân nặng thú cưng thay đổi liên tục theo thời gian. Nếu hóa đơn chỉ lưu `serviceId` rồi `populate` lấy `Service.price`, thì tháng sau Admin tăng giá khám từ 200k lên 250k sẽ làm toàn bộ hóa đơn tháng trước tự động đổi số tiền, phá hỏng báo cáo doanh thu và biên lai PDF. Lưu snapshot (`name`, `unitPrice`, `productNameSnapshot`, `unitSnapshot`, `petSnapshot`) ngay tại thời điểm phát sinh đảm bảo tính bất biến của chứng từ kế toán và hồ sơ y tế.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/models/Invoice.js:3-10` (`invoiceItemSchema`), `src/models/MedicalRecord.js:3-11` (`prescriptionSchema`), `src/models/AiAssessment.js:3-16` (`petSnapshot`), `src/services/invoice.service.js:1058-1174` (PDF đọc từ `invoice.items`).
  - Happy Path: Đổi giá dịch vụ hoặc ngừng bán sản phẩm không làm thay đổi lịch sử hóa đơn hay đơn thuốc cũ.
  - Fail / Edge Cases liên quan: Khi lập hóa đơn mới vẫn phải lấy giá trực tiếp từ DB (`Service`/`Product`), tuyệt đối không lấy `unitPrice` do client gửi lên để chống sửa giá.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Vậy khi cần thống kê Top sản phẩm bán chạy, snapshot có gây khó khăn gì không?"
  - *Cách trả lời*: Không ạ, vì mỗi item trong `Invoice.items` vẫn lưu kèm `referenceId` (`ObjectId` trỏ tới `Product`) bên cạnh `name` snapshot. Báo cáo Top sản phẩm (`invoice.service.js:1183-1262`) gom nhóm theo `referenceId` để cộng dồn chính xác kể cả khi tên sản phẩm từng được chỉnh sửa nhẹ.

### Q12. [Database / Compound Index] [Mức độ: Nâng cao] Khi dữ liệu chạy 12 tháng lên hàng chục nghìn bản ghi, làm sao chứng minh các truy vấn nặng nhất (Báo cáo doanh thu, Hàng đợi bác sĩ, Cron quá hạn) không bị Full Collection Scan?
- **Ý đồ giám khảo**: Kiểm tra khả năng thiết kế Compound Index theo đúng quy tắc ESR (Equality – Sort – Range).
- **Trả lời miệng (30–45 giây)**: Bọn em thiết kế Compound Index bám sát điều kiện `$match` và `sort` của từng luồng đọc nặng: (1) Báo cáo doanh thu & Top sản phẩm dùng `{ paymentStatus: 1, paidAt: -1 }` trên `Invoice.js:50` để lọc thẳng nhánh `Paid` và Index Range Scan theo dải ngày; (2) Hàng đợi `my-queue` dùng `{ assignedStaffId: 1, status: 1, enqueueAt: 1 }` trên `AppointmentService.js:42`; (3) `JOB-01` quét quá hạn dùng `{ date: 1, derivedStatus: 1 }` trên `AppointmentOrder.js:42`; (4) Timeline & lịch sử AI dùng `{ petId: 1, createdAt: -1 }` trên `MedicalRecord.js:33` và `AiAssessment.js:83`.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/models/Invoice.js:50-51`, `src/models/AppointmentService.js:42-43`, `src/models/AppointmentOrder.js:40-43`, `src/models/MedicalRecord.js:33`, `src/models/AiAssessment.js:83-84`, `src/models/StaffSchedule.js:18`, `src/models/CommunityPost.js:29-30`, `src/models/ContentReport.js:22`.
  - Happy Path: Các truy vấn báo cáo 12 tháng và hàng đợi chạy bằng `IXSCAN` thay vì `COLLSCAN`, đáp ứng mục tiêu `<= 3s` (`T-52`).
  - Fail / Edge Cases liên quan: Báo cáo còn giới hạn cứng dải ngày `from..to <= 366 ngày` (`report.validator.js`, `invoice.validator.js`) để tránh client query 10 năm cùng lúc gây nghẽn RAM.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Index `{ paymentStatus: 1, paidAt: -1 }` có phục vụ tốt cho câu query lọc theo `issuedAt` không?"
  - *Cách trả lời*: Không, vì `issuedAt` không phải prefix của index đó — đó là lý do ngay dòng dưới (`Invoice.js:51`) bọn em tạo thêm 1 index riêng `{ issuedAt: -1 }` chuyên phục vụ danh sách hóa đơn sắp xếp mới nhất cho quầy Lễ tân và Admin.

### Q13. [Database / Populate vs Aggregate] [Mức độ: Nâng cao] Trong codebase, khi nào nhóm dùng `populate()`, khi nào dùng gom lô `$in`, và khi nào dùng MongoDB Aggregation Pipeline?
- **Ý đồ giám khảo**: Xem sinh viên có biết tránh lỗi kinh điển N+1 query và biết chọn công cụ truy vấn phù hợp không.
- **Trả lời miệng (30–45 giây)**: (1) Dùng `populate()` kết hợp `.select()` + `.lean()` cho các màn hình CRUD chi tiết 1 bản ghi hoặc danh sách ngắn có khóa ngoại cố định (như chi tiết đơn hẹn, chi tiết chuồng); (2) Dùng gom lô `$in` + `Map` trong `Promise.all` khi cần ghép dữ liệu đa hình (polymorphic) như hàng đợi báo cáo vi phạm (`communityModeration.service.js:127-205` tách `postTargetIds` & `commentTargetIds` thay vì query trong vòng `for`) hoặc gắn thông tin thú cưng/hóa đơn vào Timeline; (3) Dùng `aggregate()` cho mọi phép cộng dồn, đếm nhóm, tính tăng trưởng và xếp hạng (như `report.service.js` và Top sản phẩm trong `invoice.service.js`) để đẩy việc tính toán xuống DB engine thay vì kéo hàng vạn bản ghi lên RAM Node.js.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/services/report.service.js:135-490` (Aggregation), `src/services/communityModeration.service.js:127-205` (batch `$in` chống N+1), `src/services/invoice.service.js:1183-1262` (`$unwind` + `$group` Top sản phẩm).
  - Happy Path: Trả `200` với số lượng round-trip DB cố định `O(1)` bất kể có 10 hay 100 bản ghi trong trang.
  - Fail / Edge Cases liên quan: Khi không có dữ liệu trong kỳ, `report.service.js` tự điền 0 (zero-filling) để biểu đồ FE không bị thiếu mốc ngày/tháng.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Có chỗ nào trong codebase vẫn bị query trong vòng lặp không?"
  - *Cách trả lời*: Có 2 chỗ chủ đích chạy tuần tự trong vòng lặp ngắn: (1) `createOnlineBooking` (`appointmentService.service.js`) tạo từng dịch vụ theo `sequenceNo` để bảo đảm thứ tự; (2) `flagOverdueAppointments` (`flagOverdueAppointments.job.js:64-91`) chạy `findOneAndUpdate` từng đơn để chống race condition với Lễ tân Check-in và chỉ đánh dấu dịch vụ #1 theo `BR-08`.

### Q14. [Database / Transaction] [Mức độ: Cực gắt] Cơ chế `runWithTransaction` trong `src/utils/transaction.js` hoạt động ra sao khi chạy trên MongoDB Atlas so với khi chạy test local?
- **Ý đồ giám khảo**: Kiểm tra sinh viên có hiểu thực tế MongoDB Standalone không hỗ trợ Transaction và cách xử lý đa môi trường.
- **Trả lời miệng (30–45 giây)**: MongoDB chỉ hỗ trợ Multi-Document Transaction trên Replica Set hoặc Sharded Cluster (như MongoDB Atlas), còn MongoDB Standalone cài local để chạy Jest sẽ ném lỗi ngay nếu gọi `startTransaction`. Vì vậy em viết helper `runWithTransaction` trong `src/utils/transaction.js`: nó kiểm tra `mongoose.connection.client.topology.description.type`. Trên Atlas (`ReplicaSetWithPrimary`), nó mở `session` và chạy `session.withTransaction(workFn)` đảm bảo ACID toàn phần. Trên Standalone local, nó truyền `session = null` vào callback để service kích hoạt nhánh compensating rollback thủ công.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/utils/transaction.js:7-13` (`supportsMongoTransaction`) → `src/utils/transaction.js:19-34` (`runWithTransaction`) → 3 điểm gọi chính: `src/services/appointmentOrder.service.js:935` (Check-in), `src/services/walkIn.service.js:345` (Walk-in), `src/services/invoice.service.js:592` (Finalize sau thanh toán).
  - Happy Path: Trên Atlas, toàn bộ ghi nhiều bảng commit cùng lúc hoặc abort sạch cùng lúc; luôn đóng session trong `finally` (`L31-33`) tránh rò rỉ tài nguyên.
  - Fail / Edge Cases liên quan: Nếu lỗi xảy ra giữa chừng trên Atlas, `session.withTransaction` tự abort và ném lại đúng lỗi gốc (`400`/`409`/`500`) về service/controller.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Chạy test trên Standalone thì làm sao chắc chắn code truyền `{ session }` vào mọi câu query bên trong `runWithTransaction` không bị sót khi lên Atlas?"
  - *Cách trả lời*: Em đồng ý đó là hạn chế của môi trường test Standalone: nếu 1 câu query bên trong quên truyền `{ session }` thì Standalone vẫn pass nhưng lên Atlas sẽ nằm ngoài transaction. Để khắc phục triệt để trong CI/CD, nên chạy MongoDB local 1-node Replica Set (`--replSet rs0`) hoặc `mongodb-memory-server` mode ReplSet để test thật cả commit/abort của session.

### Q15. [Database / Multi-collection Rollback] [Mức độ: Cực gắt] Trong luồng Walk-in ghi tới 6 collection (`User`, `Pet`, `AppointmentOrder`, `AppointmentService`, `CageAssignment`, `Cage`), nếu bước cuối gán chuồng bị lỗi `409` trên Standalone thì có bị sinh tài khoản/thú cưng rác không?
- **Ý đồ giám khảo**: Kiểm tra khả năng bảo toàn tính nhất quán dữ liệu khi không có transaction native.
- **Trả lời miệng (30–45 giây)**: Không sinh rác ạ. Khi chạy trên Atlas (`session !== null`), MongoDB tự abort toàn bộ 6 collection. Khi chạy trên Standalone (`session === null`), `walkIn.service.js` lưu sẵn các biến cờ `createdCustomerId`, `createdPetId`, `createdOrderId`, `createdServiceIds`, `createdAssignmentId`. Nếu bất kỳ bước nào ném lỗi (vd chuồng vừa bị lễ tân khác chiếm), khối `catch` sẽ thực hiện **compensating rollback** xóa ngược đúng những bản ghi mới tạo trong phiên đó và trả chuồng về `Available`, tuyệt đối không đụng tới khách hàng/thú cưng cũ đã có trước đó.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/services/walkIn.service.js:345-452` (bọc `runWithTransaction` + lưu ID các thực thể vừa tạo + khối hoàn tác khi `!session`). Tương tự cho Check-in tại `src/services/appointmentOrder.service.js:767-935` (lưu `previousOrder`, `previousServiceSnapshots`).
  - Happy Path: Tạo trọn gói thành công trả `201 { message, data: { isNewCustomer, customer, isNewPet, pet, appointment, services, cageAssignment } }`.
  - Fail / Edge Cases liên quan: Chuồng vừa bị chiếm → `409` (`walkIn.service.js:274` / `cageAssignment.service.js:62`); hết 5 lần sinh mã `USR`/`PET`/`APT` → `500` (`walkIn.service.js:75,99,142,441-448`), toàn bộ bản ghi tạo dở được dọn sạch.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Điều gì xảy ra nếu server bị mất điện đột ngột đúng giữa 2 câu lệnh xóa ngược trên Standalone?"
  - *Cách trả lời*: Compensating rollback chỉ cứu được lỗi ứng dụng/DB logic, không chống được crash tiến trình giữa chừng. Chính vì thế trên môi trường Production (MongoDB Atlas), hệ thống bắt buộc chạy nhánh Replica Set Transaction thật (`supportsMongoTransaction() === true`) để đảm bảo tính nguyên tử mức cơ sở dữ liệu.

### Q16. [Database / Dynamic Schema] [Mức độ: Nâng cao] Trong 18 model, `Service.requiredRole` và `ContentReport.targetId` có cách thiết kế schema gì đặc biệt?
- **Ý đồ giám khảo**: Kiểm tra sinh viên có đọc kỹ và hiểu các cấu trúc đặc thù trong `src/models/` hay không.
- **Trả lời miệng (30–45 giây)**: Có 2 điểm thiết kế rất gọn trong 18 model: (1) Ở `Service.js:16-18`, `requiredRole` dùng **dynamic default function**: nếu `category === 'Grooming'` thì mặc định `'Groomer'`, còn lại (`Consultation`, `Vaccination`, `LabTest`, `Boarding`...) mặc định `'Veterinarian'`, giảm lỗi quên gán role khi tạo dịch vụ; (2) Ở `ContentReport.js:13`, `targetId` dùng **dynamic `ref` function** dựa vào `doc.targetType` (`'Post' ? 'CommunityPost' : 'PostComment'`), cho phép 1 bảng `content_reports` duy nhất báo cáo được cả Bài viết lẫn Bình luận.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/models/Service.js:16-18` (`defaultRequiredRole`), `src/models/ContentReport.js:10-16` (`ref: (doc) => ...`).
  - Happy Path: Tạo dịch vụ Grooming tự động nhận `requiredRole: 'Groomer'`; báo cáo vi phạm lưu gọn trong 1 collection với Unique Index `{ reporterId: 1, targetType: 1, targetId: 1 }` (`ContentReport.js:23`).
  - Fail / Edge Cases liên quan: Khi Admin xem danh sách báo cáo (`communityModeration.service.js:127-205`), thay vì dựa vào `populate` đa hình chậm, service tách `postTargetIds` và `commentTargetIds` để query song song bằng `$in` chống N+1.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Nếu Admin cố tạo dịch vụ `Grooming` nhưng truyền `requiredRole: 'Veterinarian'` thì có bị chặn không?"
  - *Cách trả lời*: Trong `service.service.js`, hệ thống chuẩn hóa hoặc kiểm tra tính hợp lệ của `requiredRole` (`Veterinarian | Groomer`), và khi phân công nhân sự (`appointmentService.service.js`, `walkIn.service.js`), hệ thống luôn đối chiếu `staff.role === service.requiredRole`, sai role trả `400` để đảm bảo Groomer không bao giờ bị gán đi khám bệnh.

### Q17. [Database / Seed Safety] [Mức độ: Cơ bản] Nếu thành viên trong nhóm vô tình gõ `npm run seed` khi đang kết nối tới DB MongoDB Atlas dùng chung thì dữ liệu có bị xóa sạch không?
- **Ý đồ giám khảo**: Kiểm tra ý thức an toàn vận hành (operational safety) trên DB dùng chung.
- **Trả lời miệng (30–45 giây)**: Không bị xóa ạ. Trong `src/seeds/seed.js:28-35`, ngay trước khi làm bất cứ thao tác ghi/xóa nào, script sẽ đếm bản ghi trên toàn bộ model đã nạp. Nếu bất kỳ collection nào đã có `count > 0` và câu lệnh **không truyền rõ cờ `--force`**, script lập tức in cảnh báo và thoát mà không xóa bất kỳ document nào. Chỉ khi chạy tường minh `npm run seed -- --force` nó mới `deleteMany({})`, gọi `syncIndexes()` rồi tạo lại dữ liệu mẫu.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `package.json:10` → `src/seeds/seed.js:11-48` (nạp toàn bộ model trong `src/models/` → check `count > 0` & `--force` tại `L28-35` → `deleteMany` + `syncIndexes` tại `L37-40`).
  - Happy Path: Chạy `npm run seed` trên DB trống → seed đủ dữ liệu mẫu và đồng bộ toàn bộ index.
  - Fail / Edge Cases liên quan: Chạy `npm run seed` trên DB đã có dữ liệu → dừng an toàn, bảo toàn 100% dữ liệu team trên Atlas.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Seed script hiện đã seed đủ cả 18 collection chưa?"
  - *Cách trả lời*: Em báo cáo thẳng: `seed.js:11-13` tự động đọc toàn bộ 18 model để chạy `syncIndexes()`, nhưng ở dòng `L14-18` mới tạo dữ liệu mẫu cho 17/18 collection, chưa tạo document mẫu cho `AiAssessment` (vì `AiAssessment` được bổ sung sau cùng).

---

## Phần 3: Xác thực (Auth), Phân quyền (RBAC) & Bảo mật

### Q18. [Auth / Register] [Mức độ: Cơ bản] Luồng `POST /api/auth/register` bảo vệ thế nào nếu client cố truyền `role: "Admin"` hoặc đăng ký trùng SĐT của một khách vãng lai cũ (`passwordHash: null`)?
- **Ý đồ giám khảo**: Kiểm tra chống leo thang đặc quyền (Privilege Escalation) và xử lý edge case khách Walk-in.
- **Trả lời miệng (30–45 giây)**: (1) `auth.service.js` tuyệt đối bỏ qua mọi `role` hay `status` từ `req.body`, luôn gán cứng `role: 'PetOwner'` và `status: 'Active'` ở server; (2) Mật khẩu được băm bằng `bcryptjs` (`salt = 10`) và mã `USR-XXXXXX` sinh ngẫu nhiên có retry tối đa 5 lần khi trùng; (3) Với trường hợp SĐT đã tồn tại — kể cả khách Walk-in cũ đang có `passwordHash: null` — hệ thống vẫn chặn và trả **`409 Conflict`** vì SĐT là định danh unique ở tầng DB, tránh việc người lạ chiếm đoạt lịch sử khám/thú cưng của khách Walk-in chỉ bằng cách biết SĐT mà không xác minh OTP.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/auth.routes.js` → `src/controllers/auth.controller.js` → `src/validators/auth.validator.js` → `src/services/auth.service.js` (`register`) → `src/models/User.js:5-7,26`.
  - Happy Path: `201 Created` trả `{ message, user }` (tuyệt đối không có `passwordHash`).
  - Fail / Edge Cases liên quan: Sai format SĐT VN (`0[3|5|7|8|9]...`), mật khẩu < 6 ký tự, email sai → `400 { message, errors }`; trùng SĐT hoặc email (kể cả `E11000` đua lệnh) → `409`; cạn 5 lần sinh mã `USR-XXXXXX` → `500` (`auth.service.js:118`).
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Vậy khách Walk-in về nhà muốn kích hoạt tài khoản để xem bệnh án online thì làm sao?"
  - *Cách trả lời*: Hiện tại đây là giả định an toàn nhóm chọn để tránh chiếm tài khoản qua SĐT chưa xác thực (nếu Walk-in có lưu email thì khách có thể dùng luồng Quên mật khẩu qua OTP email hoặc nhờ lễ tân/admin hỗ trợ). Hướng mở rộng chuẩn thực tế là thêm luồng **Claim Account qua SMS OTP** để xác minh đúng chủ SĐT rồi mới cho đặt mật khẩu và chuyển `Inactive → Active`.

### Q19. [Auth / Login & BR-03] [Mức độ: Cực gắt] Hãy trace cơ chế đăng nhập (`POST /api/auth/login`), chống Brute-Force (`BR-03`) và chống Account Enumeration?
- **Ý đồ giám khảo**: Kiểm tra bảo mật xác thực thực chiến: khóa tạm thời, reset counter, và thông báo lỗi không tiết lộ thông tin.
- **Trả lời miệng (30–45 giây)**: `login` nhận `identifier` (SĐT hoặc email) và phải chủ động gọi `.select('+passwordHash')` vì schema mặc định ẩn field này. Thứ tự kiểm tra: (1) Nếu không tìm thấy user hoặc `passwordHash == null`, trả chung 1 thông báo `401` `'Số điện thoại/email hoặc mật khẩu không đúng'` để kẻ tấn công không dò được SĐT nào đã đăng ký; (2) Nếu `lockUntil > now` → chặn ngay **`403`** dù nhập đúng hay sai mật khẩu; (3) Nếu `status` là `Locked`/`Inactive` → trả **`403`**; (4) So sánh mật khẩu bằng `bcrypt.compare`: nếu sai thì tăng `failedLoginAttempts`, đạt lần 5 sẽ đặt `lockUntil = now + 30 phút` và trả **`403`** (`MSG12`); nếu đúng thì reset `failedLoginAttempts = 0`, `lockUntil = null` và ký JWT chỉ chứa `{ id, role }`.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/auth.routes.js` → `src/controllers/auth.controller.js` → `src/services/auth.service.js:121-166` (`login`, `.select('+passwordHash')` ở `L129`, check `lockUntil` ở `L132-137`, lỗi gộp ở `L140`, đếm 5 lần ở `L145-152`, reset ở `L162-166`).
  - Happy Path: `200 OK` trả `{ token, user }` (không chứa `passwordHash`).
  - Fail / Edge Cases liên quan: Thiếu `identifier`/`password` → `400`; sai tài khoản hoặc sai mật khẩu (lần 1–4) → `401` (`auth.service.js:156`); sai lần 5 hoặc đang trong 30 phút tạm khóa → `403` (`L132-137, 145-152`); tài khoản `Locked`/`Inactive` → `403`.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Sau khi hết 30 phút tạm khóa, lần đăng nhập tiếp theo nhập sai tiếp 1 lần thì bị khóa lại ngay hay được đếm lại từ 1?"
  - *Cách trả lời*: Khi `lockUntil` đã quá hạn (`lockUntil <= now`), hệ thống kiểm tra và reset bộ đếm hợp lý trước khi cộng dồn lại để tránh việc vừa hết khóa đã bị khóa tiếp ngay ở lần thử đầu tiên. Tuy nhiên như em cũng nhìn nhận ở Phần 7, hệ thống nên bổ sung thêm `express-rate-limit` theo IP để chặn kẻ tấn công thử 1 mật khẩu phổ biến trên hàng nghìn SĐT khác nhau (Password Spraying).

### Q20. [Auth / JWT & Protect] [Mức độ: Cực gắt] JWT bản chất là stateless, tại sao middleware `protect` lại query ngược MongoDB trên MỌI request? Có đánh đổi hiệu năng không?
- **Ý đồ giám khảo**: Kiểm tra hiểu biết sâu về JWT revocation (thu hồi token ngay lập tức) và trade-off giữa bảo mật với hiệu năng.
- **Trả lời miệng (30–45 giây)**: Theo nghiệp vụ `UC-62`, khi Admin khóa tài khoản một nhân viên nghỉ việc hoặc người dùng vi phạm, quyền truy cập phải bị cắt **ngay lập tức**, không thể đợi JWT sống thêm tối đa 24 giờ (`JWT_EXPIRES_IN=1d`). Ngoài ra nếu Admin vừa điều chuyển vai trò nhân sự, quyền mới cũng phải có hiệu lực ngay. Vì vậy sau `jwt.verify()`, `protect` chạy `User.findById(decoded.id).select('status role').lean()`: tìm theo `_id` (Primary Key có B-Tree index sẵn) chỉ lấy 2 trường nhỏ nên tốn dưới 1–2ms, hoàn toàn xứng đáng để đổi lấy bảo mật tức thời.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/middlewares/auth.middleware.js:7-48` (`Bearer ` check `L8-11` → `jwt.verify` `L16` → `User.findById(decoded.id).select('status role').lean()` `L18-27` → chặn `status !== 'Active'` `L30-37` → ghi đè `req.user.role = user.role || decoded.role` `L38`).
  - Happy Path: Gán `req.user = { id, role }` đồng bộ từ DB rồi gọi `next()`.
  - Fail / Edge Cases liên quan: Thiếu header `Bearer ` → `401` (`L10`); token giả/hết hạn → `401` (`L46`); token còn hạn nhưng DB là `Locked` hoặc `Inactive` → `403` (`L31-36`). Nếu chạy unit test không nối DB (`readyState !== 1` hoặc ID giả như `tests/authorize.test.js:14`), nhánh fallback `L43-44` dùng payload JWT.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Nếu hệ thống có 20.000 request/giây thì query MongoDB mỗi request sẽ gây áp lực lớn — anh giải quyết thế nào?"
  - *Cách trả lời*: Khi scale lớn, em sẽ chuyển sang mô hình **Access Token ngắn hạn (15 phút) + Refresh Token**, kết hợp lưu danh sách user bị khóa/đổi role vào **Redis** (đọc trong RAM < 0.2ms, chỉ ghi Redis khi Admin thực sự khóa/đổi role) thay vì query MongoDB trên mọi request.

### Q21. [Auth / Logout] [Mức độ: Cơ bản] `POST /api/auth/logout` có thu hồi token ở phía server không? Nếu JWT stateless mà không có blacklist thì tạo endpoint `/logout` để làm gì?
- **Ý đồ giám khảo**: Kiểm tra sự trung thực kỹ thuật, không "chém gió" về JWT logout.
- **Trả lời miệng (30–45 giây)**: Em trình bày thẳng: hiện tại hệ thống dùng JWT stateless, chưa có bảng Token Blacklist trên Redis nên `POST /api/auth/logout` xác thực token hợp lệ qua `protect` rồi trả `200` để FE xóa token khỏi bộ nhớ trình duyệt. Việc vẫn duy trì endpoint này giúp giữ chuẩn REST contract cho FE và sẵn sàng cắm logic lưu `jti` vào Redis Blacklist ở server về sau mà không phải sửa API của FE.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/auth.routes.js` (`POST /logout`, gắn `protect`) → `src/controllers/auth.controller.js` (`logout`).
  - Happy Path: `200 OK` `{ message: 'Đăng xuất thành công' }`.
  - Fail / Edge Cases liên quan: Không gửi Bearer token hoặc token hết hạn → `401` (`auth.middleware.js:10,46`); tài khoản đã bị khóa → `403` (`auth.middleware.js:31`).
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Nếu ai đó đánh cắp JWT trước khi user bấm Logout thì họ vẫn dùng tiếp được token đó?"
  - *Cách trả lời*: Đúng ạ, trong phạm vi đồ án token còn sống cho đến khi hết `exp` (trừ khi Admin khóa tài khoản thì `protect` cắt ngay). Để khắc phục hoàn toàn trong production cần thêm cột `tokenVersion` trên `User` (tăng lên +1 khi logout/đổi mật khẩu, đối chiếu trong `protect`) hoặc dùng Redis blacklist theo `jti` với TTL đúng bằng thời gian sống còn lại của token.

### Q22. [Auth / OTP & Forgot Password] [Mức độ: Cực gắt] Toàn bộ luồng OTP (xác thực email, đổi mật khẩu, quên mật khẩu) lưu và kiểm tra mã OTP như thế nào? Có rủi ro lộ OTP trong DB không?
- **Ý đồ giám khảo**: Soi kỹ luồng nhạy cảm nhất nhì hệ thống (chiếm đoạt tài khoản qua Quên mật khẩu).
- **Trả lời miệng (30–45 giây)**: Mã OTP 6 chữ số không bao giờ lưu plain-text trong MongoDB mà được **băm bằng `bcryptjs`** lưu vào `emailOtpHash` (có `select: false` ở `User.js:11`) cùng hạn dùng 5 phút ở `emailOtpExpiresAt` (`User.js:12`). Khi xác thực hoặc đặt lại mật khẩu, service bắt buộc `.select('+emailOtpHash +emailOtpExpiresAt')`, kiểm tra hết hạn trước, đối chiếu `bcrypt.compare`, và dùng xong thì **xóa sạch `emailOtpHash = null`, `emailOtpExpiresAt = null`** ngay lập tức để chống dùng lại OTP (replay attack). Ở luồng đổi/quên mật khẩu thành công, mật khẩu mới được băm mới và reset cả `failedLoginAttempts`/`lockUntil`.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/user.routes.js` (`POST /me/email/send-otp`, `POST /me/email/verify`, `POST /me/password/send-otp`, `PUT /me/password`, `POST /forgot-password`, `POST /reset-password`) → `src/services/user.service.js:26-70` (gửi email Nodemailer) & các hàm OTP → `src/models/User.js:10-12`.
  - Happy Path: `200 OK` (ở môi trường dev chưa cấu hình SMTP, trả thêm thông tin hỗ trợ dev/log console để test).
  - Fail / Edge Cases liên quan: Email/OTP sai định dạng → `400`; OTP sai hoặc hết hạn 5 phút → `400`; chưa xác thực email mà yêu cầu đổi mật khẩu qua OTP → `400`; cố đăng ký email trùng người khác → `409`.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Nếu kẻ tấn công thử gửi 1000 request `POST /api/users/reset-password` đoán mã OTP 6 chữ số trong 5 phút thì sao?"
  - *Cách trả lời*: Hiện tại OTP có TTL 5 phút và dùng một lần, nhưng chưa có bộ đếm `failedOtpAttempts` (ví dụ sai 5 lần hủy OTP) và chưa có `express-rate-limit` trên endpoint `/reset-password` và `/forgot-password`. Trong production bắt buộc phải giới hạn số lần nhập sai OTP (tối đa 5 lần hủy mã) + cooldown 60 giây giữa 2 lần gửi OTP.

### Q23. [RBAC / Authorize] [Mức độ: Nâng cao] Cơ chế phân quyền 5 vai trò (`PetOwner`, `Receptionist`, `Veterinarian`, `Groomer`, `Admin`) hoạt động thế nào? Nếu dev quên gắn `protect` trước `authorize` thì sao?
- **Ý đồ giám khảo**: Kiểm tra tính chặt chẽ và lập trình phòng thủ của tầng RBAC.
- **Trả lời miệng (30–45 giây)**: `authorize(...allowedRoles)` là một higher-order middleware nhận danh sách vai trò được phép. Nó đặt ngay sau `protect`, đọc `req.user.role` (vốn đã được `protect` đồng bộ mới nhất từ MongoDB) và kiểm tra `allowedRoles.includes(req.user.role)`, sai trả ngay `403`. Đặc biệt, ở dòng 55 `auth.middleware.js` có kiểm tra phòng thủ `if (!req.user) return res.status(401)` — nên kể cả dev có quên gắn `protect` phía trước thì route vẫn chặn `401` an toàn, không bị bypass và không crash `TypeError`.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/middlewares/auth.middleware.js:52-65` (`authorize`) → kiểm chứng bởi `tests/authorize.test.js:1-45` (gồm cả test case `missing-protect` ở `L12, 40-43`).
  - Happy Path: Role hợp lệ → gọi `next()` vào controller.
  - Fail / Edge Cases liên quan: Thiếu `req.user` → `401` (`L56`); role không nằm trong `allowedRoles` → `403` (`L60`).
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Trong ma trận phân quyền, có API nào `Admin` KHÔNG được gọi nhưng `PetOwner` hoặc `Veterinarian` lại được gọi không?"
  - *Cách trả lời*: Có ạ, nhóm phân tách rõ theo đúng trách nhiệm nghiệp vụ chứ không cho `Admin` làm "siêu nhân" đụng vào hồ sơ chuyên môn: ví dụ `POST /api/medical-records/service/:id/start|complete` và `/api/inpatient-records` chỉ dành cho `Veterinarian`; `POST /api/ai-assessments` và toàn bộ tạo/tương tác bài viết ở `/api/community-posts` chỉ dành cho `PetOwner`; còn `POST /api/invoices` chỉ cho `Receptionist`.

### Q24. [Security / IDOR] [Mức độ: Cực gắt] Nếu một `PetOwner` A đăng nhập hợp lệ rồi đổi `:id` trên URL sang hồ sơ thú cưng, bệnh án, hóa đơn PDF hoặc đánh giá AI của `PetOwner` B thì hệ thống chặn ở đâu và trả mã gì?
- **Ý đồ giám khảo**: IDOR (Insecure Direct Object Reference) là lỗ hổng số 1 hay bị bắt lỗi nhất trong đồ án sinh viên.
- **Trả lời miệng (30–45 giây)**: Middleware `authorize('PetOwner')` chỉ lọc vai trò, nên chống IDOR được thực thi chặt ở tầng **Service** trên toàn bộ tài nguyên cá nhân: mọi hàm đọc/sửa/xóa/tải PDF đều so sánh `String(record.ownerId) === String(req.user.id)` (lấy từ JWT, tuyệt đối không lấy từ `req.body` hay `req.query`). Riêng một số endpoint nhạy cảm như `inpatientRecord.service.js:331-335` (`GET /pets/:petId/daily-logs`) hay `communityPost.service.js:51-54` chủ động trả **`404`** để giấu luôn sự tồn tại của bản ghi, còn các luồng `pets`, `petTimeline`, `appointmentOrder`, `medicalRecord`, `invoice`, `aiAssessment` trả **`403`** (`BR-23`).
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/services/pet.service.js`, `src/services/petTimeline.service.js`, `src/services/appointmentOrder.service.js`, `src/services/medicalRecord.service.js`, `src/services/invoice.service.js` (`assertCanAccessInvoice` dùng cho cả `GET /:id`, `GET /:id/pdf`, `GET /:id/payos-status`), `src/services/inpatientRecord.service.js:331-335`, `src/services/communityPost.service.js:51-54`, `src/services/aiAssessment.service.js:194-196`.
  - Happy Path: Đúng chủ sở hữu (hoặc nhân sự có thẩm quyền) → `200`.
  - Fail / Edge Cases liên quan: Khác `ownerId` → `403` (hoặc `404` tại `inpatientRecord`/`communityPost`); kiểm chứng đầy đủ trong `tests/invoice.test.js`, `tests/medicalRecord.test.js`, `tests/petTimeline.test.js`, `tests/aiAssessment.test.js`, `tests/communityPost.test.js`.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Thế còn trường hợp `PetOwner` A truyền `petId` của `PetOwner` B vào body lúc tạo lịch hẹn online hoặc tạo bài `TimelineShare` thì sao?"
  - *Cách trả lời*: Cũng bị chặn hết ở tầng Service ạ — cả `appointmentOrder.service.js`, `communityPost.service.js` (`createTimelineShare`), và `aiAssessment.service.js:194-196` đều truy vấn `Pet` và kiểm tra `String(pet.ownerId) === String(req.user.id)` trước khi xử lý, sai là chặn ngay `403`/`400`.

### Q25. [Security / Mass Assignment] [Mức độ: Nâng cao] Nếu client cố tình chèn thêm `status: "Paid"`, `totalAmount: 0`, `role: "Admin"` hoặc `resultImageUrl: "https://evil.com/x.jpg"` vào JSON body thì có bị ghi đè vào DB không?
- **Ý đồ giám khảo**: Kiểm tra xem dev có viết kiểu ẩu `Model.create(req.body)` hay `findByIdAndUpdate(id, req.body)` không.
- **Trả lời miệng (30–45 giây)**: Không bị ghi đè ạ. Trong toàn bộ 22 module, bọn em không bao giờ truyền thẳng `req.body` vào Mongoose. Mọi tầng Validator và Service đều **whitelist rõ ràng từng trường được phép ghi** (ví dụ `UPDATABLE_FIELDS` ở `pet.service.js:82`, `pick(...)` ở `cage.validator.js:109,127`), thậm chí một số validator còn chủ động **reject `400`** ngay nếu phát hiện trường lạ nhạy cảm (ví dụ `subtaskRequest.validator.js:50-53, 167-172` chặn thẳng nếu client gửi `status` hay `resultImageUrl` trong JSON, bắt buộc ảnh phải đi qua luồng upload Cloudinary của server).
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/validators/subtaskRequest.validator.js:50-53,167-172`, `src/services/pet.service.js:82`, `src/validators/cage.validator.js:19-23,109,127`, `src/services/invoice.service.js:179-412` (tự tính giá từ DB).
  - Happy Path: Chỉ các trường hợp lệ trong whitelist được ghi vào MongoDB; các trường tính toán (`totalAmount`, `status`, `ownerId`, `issuedById`) luôn do server tự thiết lập.
  - Fail / Edge Cases liên quan: Truyền trường bị cấm ở `subtaskRequest` → `400`; truyền URL ảnh ngoài ở `communityPost` → `400`.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Việc whitelist quá chặt có gây ra bug nào thật trong codebase hiện tại không?"
  - *Cách trả lời*: Em thừa nhận thật là có 1 chỗ bị whitelist thiếu: trong `src/validators/cage.validator.js:109,127`, hàm `pick` chỉ lấy 4 trường `['label', 'size', 'isIsolated', 'status']`, vô tình bỏ sót 2 trường mới của schema `Cage.js:7-8` là `zone` và `maxWeightKg`. Em đã liệt kê rõ ở Phần 7 (`Q69`) để mở rộng whitelist ngay khi cần.

### Q26. [Security / Sensitive Fields] [Mức độ: Cơ bản] Làm sao đảm bảo `passwordHash`, `emailOtpHash` không bao giờ lọt ra API response kể cả khi các bảng khác `populate('ownerId')` hoặc `populate('assignedById')`?
- **Ý đồ giám khảo**: Kiểm tra lớp bảo vệ mặc định ở tầng Schema (Defense in Depth).
- **Trả lời miệng (30–45 giây)**: Bọn em khóa ngay từ gốc tại Mongoose Schema (`src/models/User.js:10-12`): cả 3 trường nhạy cảm `passwordHash`, `emailOtpHash`, `emailOtpExpiresAt` đều gắn **`select: false`**. Nhờ đó mọi lệnh `find`, `findById`, hay `populate` từ bất kỳ bảng nào trong toàn hệ thống đều tự động loại bỏ 3 trường này mà không cần nhớ viết `-passwordHash` ở từng chỗ. Ngoài ra khi `User.create()` trả về document vừa tạo (Mongoose vẫn giữ field vừa gán trong memory), service luôn trích xuất object sạch (`formatUser` / `toSafeUser`) trước khi trả về controller.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/models/User.js:10-12` (`select: false`) → chỉ 2 nơi chủ động bật lại là `auth.service.js:129` (`.select('+passwordHash')`) và `user.service.js` (`.select('+emailOtpHash +emailOtpExpiresAt')`).
  - Happy Path: Toàn bộ response API trả thông tin user hoàn toàn sạch, không có hash hay OTP.
  - Fail / Edge Cases liên quan: Lưu ý nhỏ trong `cageAssignment.service.js:21` viết `select: '-password'` (tên field thực là `passwordHash`), nhưng nhờ `select: false` ở `User.js:10` nên `passwordHash` vẫn được bảo vệ an toàn tuyệt đối.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Còn Payload của JWT chứa những trường gì? Có chứa SĐT hay email không?"
  - *Cách trả lời*: Payload JWT chỉ chứa đúng 2 trường tối giản `{ id, role }` (cùng `iat`, `exp`). Không nhét SĐT, email hay thông tin cá nhân vào JWT vì phần payload của JWT chỉ là Base64URL có thể decode dễ dàng ở phía client.

### Q27. [Security / File Upload Magic Bytes] [Mức độ: Cực gắt] Nếu kẻ tấn công đổi tên một file `.exe` hoặc `.html` thành `cat.jpg` rồi gửi lên `POST /api/pets` hoặc `POST /api/ai-assessments` thì hệ thống phát hiện bằng cách nào?
- **Ý đồ giám khảo**: Kiểm tra hiểu biết về lỗ hổng MIME-type spoofing khi upload file.
- **Trả lời miệng (30–45 giây)**: Kiểm tra `file.mimetype` trong Multer chỉ là đọc header do client khai báo nên dễ bị giả mạo. Ở 2 luồng `pets` và `ai-assessments`, bọn em đọc trực tiếp **Magic Bytes** (những byte đầu tiên của binary `Buffer` trong RAM): JPEG phải bắt đầu bằng `FF D8 FF`, PNG phải khớp đúng 8 byte `89 50 4E 47 0D 0A 1A 0A`, WebP (chỉ ở Pet) khớp `RIFF....WEBP`. File `.exe`/`.html` đổi đuôi sẽ sai chữ ký nhị phân và bị chặn trả **`400`** ngay trước khi chạm tới Cloudinary hoặc AI.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/pet.routes.js:13-37` (`multer.memoryStorage()`, ≤5MB) → `src/validators/pet.validator.js:39-60` (check Magic Bytes JPEG/PNG/WebP); và `src/middlewares/upload.middleware.js:46-55,132-147` → `src/validators/aiAssessment.validator.js:18-26,137-138` (check Magic Bytes JPG/PNG `MSG35`).
  - Happy Path: Ảnh thật hợp lệ đi tiếp qua Cloudinary / gửi base64 cho AI.
  - Fail / Edge Cases liên quan: File giả mạo đuôi/MIME → `400 { message: 'Dữ liệu không hợp lệ', errors: { photo/photos: '...' } }`; file > 5MB → `400` (`upload.middleware.js:66`, `pet.routes.js:29`).
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Thế còn 3 luồng upload khác (Grooming, Subtask, Community) có check Magic Bytes không?"
  - *Cách trả lời*: Em thừa nhận trung thực là 3 luồng đó ở `upload.middleware.js:7-14` hiện mới kiểm tra `file.mimetype` (cộng với việc Cloudinary phía sau sẽ từ chối nếu không phải ảnh hợp lệ). Em đã ghi nhận vào Phần 7 (`Q68`) sẽ tách helper check Magic Bytes dùng chung vào `upload.middleware.js` để đồng bộ cả 5 luồng.

### Q28. [Security / BR-22 & XSS Email] [Mức độ: Nâng cao] Tại sao 3 API `DELETE` (`/api/admin/users/:id`, `/api/services/:id`, `/api/invoices/:id`) lại tồn tại chỉ để trả `403`? Và khi gửi email HTML, nhóm chống HTML Injection thế nào?
- **Ý đồ giám khảo**: Kiểm tra việc hiện thực đúng Business Rule kiểm toán (`BR-22`) và an toàn nội dung HTML trong email.
- **Trả lời miệng (30–45 giây)**: (1) Theo `BR-22`, hồ sơ tài khoản, dịch vụ và hóa đơn không bao giờ được xóa vật lý nhằm bảo toàn dấu vết kiểm toán (audit trail) và lịch sử bệnh án/chứng từ tài chính. Thay vì bỏ route để Express trả `404` khó hiểu, bọn em cố ý định nghĩa route `DELETE` trả **`403 Forbidden`** kèm thông báo tiếng Việt hướng dẫn chuyển sang trạng thái `Locked` hoặc `Inactive`. (2) Với email HTML (thông báo đặt lịch cho lễ tân và nhập/xuất viện cho chủ nuôi), mọi dữ liệu động do người dùng nhập (tên khách, tên thú cưng, ghi chú) đều đi qua hàm `escapeHtml()` mã hóa 5 ký tự `& < > " '` để chống chèn thẻ HTML/script độc hại vào hộp thư.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/controllers/admin.controller.js:75-78`, `src/controllers/service.controller.js:75-78`, `src/controllers/invoice.controller.js:294-297` (`403`); `src/services/appointmentOrder.service.js:44-50` & `src/services/inpatientRecord.service.js:17-18` (`escapeHtml`).
  - Happy Path: Muốn ngừng sử dụng thì gọi `PATCH /status` hoặc `PATCH /lock` trả `200`.
  - Fail / Edge Cases liên quan: Mọi lời gọi `DELETE` lên 3 tài nguyên trên (kể cả bằng token `Admin`) đều trả **`403`**.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Tại sao gửi mail thông báo đơn mới cho nhiều Lễ tân lại đặt danh sách email vào `bcc` thay vì `to`?"
  - *Cách trả lời*: Ở `appointmentOrder.service.js:79-80`, đặt `to: emailUser` và `bcc: recipients` giúp bảo vệ quyền riêng tư giữa các nhân viên (không lộ địa chỉ email cá nhân của nhau) và tránh tình trạng lễ tân bấm "Reply All" làm spam hộp thư đồng nghiệp.

---

## Phần 4: Truy vấn sâu từng Luồng Nghiệp vụ & State Machine

> **Bao phủ 100% (20/20) file route trong `src/routes/`** — từ quản lý người dùng, danh mục, chuồng, lịch bận, đặt lịch Online/Walk-in, hàng đợi, khám SOAP, cận lâm sàng, nội trú, hóa đơn, báo cáo, cộng đồng, kiểm duyệt đến AI sức khỏe.

### Q29. [User Profile & Staff / `/api/users`] [Mức độ: Cơ bản] Hãy trace luồng xem/sửa hồ sơ cá nhân (`GET/PUT /api/users/me`) và lấy danh sách nhân sự (`GET /api/users/staff`). Khi user đổi SĐT cần chặn những lỗi gì?
- **Ý đồ giám khảo**: Kiểm tra luồng hồ sơ cá nhân có tách sạch khỏi luồng Admin và bảo vệ định danh SĐT/Email không.
- **Trả lời miệng (30–45 giây)**: `user.routes.js` phục vụ luồng cá nhân cho mọi user đã đăng nhập (cộng 2 API public quên/đặt lại mật khẩu). `GET/PUT /api/users/me` luôn lấy `req.user.id` từ token (không có `:id` trên URL nên miễn nhiễm IDOR), chỉ whitelist các field hồ sơ cơ bản (`fullName`, `phone`, `address`...), tuyệt đối không cho tự đổi `role`/`status`/`emailVerified`. Nếu user đổi `phone` trùng người khác sẽ trả `409`. Còn `GET /api/users/staff` cho phép mọi user đã đăng nhập lọc danh sách nhân sự `Active` (thường theo `role=Veterinarian|Groomer`) để chọn bác sĩ mong muốn khi đặt lịch hoặc phân công.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/user.routes.js` → `src/controllers/user.controller.js` → `src/validators/user.validator.js` → `src/services/user.service.js` → `src/models/User.js:5-26`.
  - Happy Path: `200 OK` trả `{ user }` (hoặc `{ items }` cho `/staff`, chỉ gồm thông tin công khai cần thiết, không có `passwordHash`).
  - Fail / Edge Cases liên quan: Thiếu token → `401`; sai định dạng SĐT VN / tên rỗng → `400`; đổi SĐT trùng user khác (`E11000`) → `409`; user không tồn tại → `404`.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Tại sao tách `user.*` riêng với `admin.*` dù cùng thao tác trên collection `users`?"
  - *Cách trả lời*: Để mỗi file ngắn gọn, dễ phân công git không conflict và tách biệt hoàn toàn ranh giới bảo mật: `user.*` chỉ xoay quanh `req.user.id` + OTP cá nhân, còn `admin.*` gắn cứng `authorize('Admin')` toàn bộ router cho các nghiệp vụ quản trị nhân sự (`UC-60..62`).

### Q30. [Admin Users / `/api/admin/users`] [Mức độ: Nâng cao] Hãy trace luồng Admin quản lý tài khoản (`SCR-32`): tạo nhân sự (`UC-61`), điều chuyển vai trò và khóa/mở khóa (`UC-62`). Admin có thể tự khóa chính mình hoặc tạo thêm Admin qua API không?
- **Ý đồ giám khảo**: Kiểm tra các quy tắc an toàn quản trị (`BR-24`, chống tự khóa mất quyền điều hành hệ thống).
- **Trả lời miệng (30–45 giây)**: Toàn bộ `admin.routes.js` gắn `protect, authorize('Admin')`. Khi Admin tạo tài khoản mới (`POST /api/admin/users`), validator chỉ cho phép 3 role nhân sự: `Receptionist | Veterinarian | Groomer` (không cho tạo `Admin` mới hay `PetOwner` qua luồng này theo `BR-24`), tự sinh mã `USR-XXXXXX` và băm mật khẩu. Khi xem chi tiết `GET /:id`, nếu là `PetOwner` service truy vấn kèm danh sách `pets`. Khi khóa/mở khóa (`UC-62`), service **chặn không cho Admin tự khóa chính tài khoản đang đăng nhập** (`400`/`403`) và chặn đổi vai trò sang `Admin` hoặc đổi vai trò của `PetOwner`. Mọi lệnh `DELETE /:id` trả `403` (`BR-22`).
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/admin.routes.js` → `src/controllers/admin.controller.js:75-78` (`DELETE` → `403`) → `src/validators/admin.validator.js` → `src/services/admin.service.js` → `src/models/User.js`, `src/models/Pet.js`.
  - Happy Path: `POST /` trả `201` kèm user mới; `GET /`, `GET /:id`, `PUT /:id`, `PATCH /:id/status` trả `200`.
  - Fail / Edge Cases liên quan: Không phải `Admin` → `403`; truyền `role: 'Admin'` khi tạo → `400`; trùng SĐT/email → `409`; Admin tự khóa mình → `400`/`403`; gọi `DELETE /:id` → `403` (`admin.controller.js:76`); không tìm thấy user → `404`.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Sau khi Admin khóa tài khoản một Lễ tân, mất bao lâu thì phiên làm việc của Lễ tân đó bị đá văng?"
  - *Cách trả lời*: Ngay lập tức ở request tiếp theo ạ, vì `protect` (`auth.middleware.js:23-37`) đọc `status` trực tiếp từ MongoDB và lập tức trả `403 'Tài khoản đã bị khoá...'`.

### Q31. [Pets / `/api/pets`] [Mức độ: Nâng cao] Hãy trace luồng CRUD thú cưng (`POST/GET/PUT/DELETE /api/pets`), upload ảnh đại diện và điều kiện xóa mềm (`status: 'Deleted'`)?
- **Ý đồ giám khảo**: Kiểm tra bảo vệ toàn vẹn lịch sử khám và dọn ảnh Cloudinary khi tạo/sửa thú cưng.
- **Trả lời miệng (30–45 giây)**: Mọi thao tác CRUD thú cưng chỉ dành cho `PetOwner` trên thú cưng của chính mình (`ownerId === req.user.id`). Ảnh đại diện đi qua `uploadPhoto` inline trong `pet.routes.js:13-37` (`multer.memoryStorage()`, ≤5MB), sau đó `pet.validator.js:39-60` kiểm tra **Magic Bytes** (JPEG/PNG/WebP) rồi `pet.service.js:37-70` mới đẩy lên Cloudinary. Nếu `Pet.create` lỗi DB, service gọi `removePhoto()` xóa ngược ảnh vừa đẩy tránh rác. Khi `DELETE /api/pets/:id`, hệ thống kiểm tra thú cưng có đang dính lịch hẹn chưa kết thúc (`Booked`/`CheckedIn`/`InProgress`/`WaitOwner`) hoặc đang nhập viện `Admitted` không — nếu có chặn **`409`**, không có mới soft delete `status: 'Deleted'` để giữ nguyên lịch sử bệnh án cũ.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/pet.routes.js:13-52` (`uploadPhoto` + routes) → `src/validators/pet.validator.js:39-60` (Magic Bytes) → `src/controllers/pet.controller.js` → `src/services/pet.service.js:37-115` (`removePhoto`, `UPDATABLE_FIELDS` `L82`, retry mã `PET-XXXXXX` `L102`).
  - Happy Path: `POST /` → `201`; `GET /`, `GET /:id`, `PUT /:id`, `DELETE /:id` (soft delete) → `200`.
  - Fail / Edge Cases liên quan: Ảnh > 5MB hoặc sai Magic Bytes → `400`; `PetOwner` xem/sửa/xóa pet người khác → `403`/`404`; thú cưng đang có đơn khám hoạt động mà bấm xóa → `409`; hết 5 lần sinh mã `PET-XXXXXX` → `500` (`pet.service.js:102`).
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Khi chủ nuôi cập nhật ảnh mới cho thú cưng ở `PUT /api/pets/:id`, ảnh cũ trên Cloudinary có bị bỏ lại làm rác không?"
  - *Cách trả lời*: Không bị bỏ lại ạ, `pet.service.js:55-70` tách `publicId` từ URL ảnh cũ và gọi `removePhoto(oldPhotoUrl)` (best-effort `cloudinary.uploader.destroy`) sau khi cập nhật DB thành công.

### Q32. [Pet Timeline / `/api/pets/:petId/timeline`] [Mức độ: Cực gắt] API `GET /api/pets/:petId/timeline` tổng hợp dữ liệu từ những bảng nào? Tại sao cùng 1 API nhưng các vai trò khác nhau lại thấy dữ liệu khác nhau?
- **Ý đồ giám khảo**: Kiểm tra khả năng thiết kế API tổng hợp đa nguồn (Unified Timeline) kèm lọc quyền hiển thị theo vai trò (`Field-level RBAC` & `BR-20`).
- **Trả lời miệng (30–45 giây)**: Endpoint này đặt trong `pet.routes.js:47` và giao cho `petTimeline.service.js` gom lịch sử thống nhất của 1 thú cưng từ nhiều nguồn: `AppointmentOrder` + `AppointmentService` (kèm Grooming Note), `MedicalRecord` (kèm đơn thuốc), `SubtaskRequest`, `InpatientRecord` (kèm `dailyLogs`), `Invoice` và `AiAssessment`. Tuy nhiên dữ liệu trả về được **lọc chặt theo vai trò**: (1) `PetOwner` chỉ xem được pet của mình (`403` nếu khác chủ) và là **vai trò duy nhất** thấy nhánh `aiAssessments` (`BR-20`); (2) `Veterinarian` và `Groomer` xem được tiền sử y khoa nhưng bị **ẩn thông tin tài chính (`invoice`) và ẩn `aiAssessments`**; (3) `Receptionist` và `Admin` xem được cả lịch hẹn lẫn hóa đơn nhưng vẫn ẩn `aiAssessments`.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/pet.routes.js:47` → `src/controllers/petTimeline.controller.js` → `src/validators/petTimeline.validator.js` → `src/services/petTimeline.service.js` → kiểm chứng trong `tests/petTimeline.test.js`.
  - Happy Path: `200 OK` trả dòng thời gian hợp nhất sắp xếp theo thời gian kèm thông tin thú cưng.
  - Fail / Edge Cases liên quan: Sai `petId` hoặc query lọc sai → `400`; `PetOwner` xem timeline thú cưng người khác → `403`; không thấy thú cưng → `404`.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Tại sao bác sĩ thú y lại không được xem `aiAssessments` trên Timeline?"
  - *Cách trả lời*: Để thực thi triệt để `BR-20` trong SRS: đánh giá AI chỉ là sàng lọc tham khảo riêng cho chủ nuôi (trước khi đến khám), tránh gây thiên kiến chẩn đoán (anchoring bias) cho bác sĩ; khi chủ nuôi đặt lịch có thể tự copy đoạn `bookingNoteDraft` vào ghi chú đặt hẹn nếu muốn chia sẻ triệu chứng.

### Q33. [Service Catalog / `/api/services`] [Mức độ: Cơ bản] Hãy trace luồng quản lý danh mục dịch vụ (`SCR-29`): quyền đọc/ghi, chống trùng tên và chuyển trạng thái `Active`/`Inactive`?
- **Ý đồ giám khảo**: Kiểm tra chuẩn REST cho danh mục master data và liên kết với các nghiệp vụ đặt lịch/hóa đơn.
- **Trả lời miệng (30–45 giây)**: `GET /api/services` và `GET /api/services/:id` mở **Public** (không cần token) để khách vãng lai xem bảng giá phòng khám. Các thao tác `POST`, `PUT`, `PATCH /:id/status` chỉ cho `Admin`. Khi tạo mới, mã `SRV-XXXXXX` được tự sinh, `requiredRole` tự mặc định theo `category` (`Service.js:16-18`), và tổ hợp `{ category, name }` bị khóa Unique Index (`Service.js:28`) để chống trùng tên dịch vụ trong cùng danh mục (`E11000 → 409`). Khi muốn ngừng cung cấp, Admin chuyển sang `Inactive` (không xóa cứng — `DELETE /:id` trả `403` theo `BR-22`), nhờ đó các hóa đơn và lịch hẹn cũ vẫn giữ nguyên tham chiếu.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/service.routes.js` → `src/controllers/service.controller.js:75-78` (`DELETE` → `403`) → `src/validators/service.validator.js` → `src/services/service.service.js` → `src/models/Service.js:5,16-18,28`.
  - Happy Path: `POST /` → `201`; `GET /`, `GET /:id`, `PUT /:id`, `PATCH /:id/status` → `200`.
  - Fail / Edge Cases liên quan: Giá âm, thời lượng ≤ 0, sai `category` → `400`; không phải `Admin` gọi ghi → `403`; trùng `(category, name)` → `409`; gọi `DELETE /:id` → `403`; không tìm thấy dịch vụ → `404`.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Nếu Admin chuyển 1 dịch vụ sang `Inactive`, các đơn hẹn cũ đã `Booked` có bị hỏng không?"
  - *Cách trả lời*: Đơn đã đặt trước đó không bị xóa hay hỏng dữ liệu, còn các lượt đặt lịch Online mới (`appointmentOrder.service.js`) hay Walk-in (`walkIn.service.js`) sẽ kiểm tra `service.status === 'Active'`, nếu `Inactive` sẽ chặn `400`.

### Q34. [Product Catalog / `/api/products`] [Mức độ: Cơ bản] Hãy trace luồng danh mục sản phẩm/thuốc (`SCR-31`): vì sao Unique Index đặt trên `(name, unit)` và việc đổi giá ảnh hưởng gì tới đơn thuốc cũ?
- **Ý đồ giám khảo**: Kiểm tra hiểu biết về nghiệp vụ dược/bán lẻ thú y và xử lý lỗi Mongoose.
- **Trả lời miệng (30–45 giây)**: Danh mục `Product` gồm 3 loại `Medicine | Vaccine | Retail`, đọc public (`GET /`, `GET /:id`), ghi chỉ `Admin` (`POST /`, `PUT /:id`, `PATCH /:id/activate`, `PATCH /:id/inactivate`). Nhóm đặt Compound Unique Index `{ name: 1, unit: 1 }` (`Product.js:15`) vì thực tế cùng 1 loại thuốc hoặc thức ăn có thể bán theo nhiều quy cách khác nhau (vd "Viên" vs "Hộp", "Gói 500g" vs "Túi 2kg"). Khi Admin ngừng bán (`Inactive`), bác sĩ sẽ bị chặn kê loại thuốc đó ở `BR-16` (`medicalRecord.service.js` trả `400`), nhưng các đơn thuốc và hóa đơn cũ vẫn nguyên vẹn nhờ lưu snapshot.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/product.routes.js` → `src/controllers/product.controller.js` (có bắt thêm `CastError` và `ValidationError` → `400`) → `src/validators/product.validator.js` → `src/services/product.service.js` → `src/models/Product.js:5,15`.
  - Happy Path: `POST /` → `201`; `GET`, `PUT`, `PATCH /activate|inactivate` → `200`.
  - Fail / Edge Cases liên quan: Giá âm/sai `category`/sai ID → `400`; trùng `(name, unit)` (`11000`) → `409`; không tìm thấy → `404`; kích hoạt sản phẩm đang `Active` hoặc vô hiệu hóa sản phẩm đã `Inactive` được xử lý đúng theo state hiện tại.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Hệ thống có quản lý tồn kho (`stockQuantity`) và trừ kho tự động khi lập hóa đơn bán lẻ không?"
  - *Cách trả lời*: Em báo cáo rõ theo đúng SRS và Schema `Product.js`: phạm vi đồ án VCMOS chỉ quản lý danh mục sản phẩm/giá bán (`code`, `name`, `category`, `unit`, `price`, `status`), không bao gồm phân hệ kho vận (Inventory/Stock).

### Q35. [Cage Management / `/api/cages`] [Mức độ: Nâng cao] Hãy trace luồng quản lý chuồng (`/api/cages`) và quy tắc tải trọng/cách ly `BR-11` (`canCageAccommodatePet`). Tại sao Admin không được chuyển tay chuồng đang `Occupied` về `Available`?
- **Ý đồ giám khảo**: Kiểm tra bảo vệ tính nhất quán giữa `Cage.status` và `CageAssignment`.
- **Trả lời miệng (30–45 giây)**: Nhân sự nội bộ (`Receptionist`, `Veterinarian`, `Groomer`, `Admin`) được xem danh sách/chi tiết chuồng, còn chỉ `Admin` được tạo mới (`POST /`) và cập nhật (`PATCH /:id`). Trong `cage.service.js`, hàm dùng chung `canCageAccommodatePet(size, petWeight, isIsolated, requiresIsolation, maxWeightKg)` thực thi `BR-11`: đối chiếu cân nặng thú cưng với ngưỡng theo size (`S/M/L/XL` hoặc `maxWeightKg` tùy chỉnh) và bắt buộc chuồng `isIsolated: true` nếu thú cưng cần cách ly (`requiresIsolation: true`). Nếu Admin sửa `PATCH /:id` cố đổi trạng thái chuồng đang có thú cưng (`Occupied` hoặc đang có `CageAssignment` chưa `releasedAt`), hệ thống chặn để tránh làm lệch dữ liệu giữa `Cage` và `CageAssignment` — muốn nhả chuồng bắt buộc đi qua API `PATCH /api/cage-assignments/:id/release`.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/cage.routes.js` → `src/controllers/cage.controller.js` → `src/validators/cage.validator.js:42-203` → `src/services/cage.service.js` (`canCageAccommodatePet`) → `src/models/Cage.js:1-15`.
  - Happy Path: `POST /` → `201`; `GET /`, `GET /:id`, `PATCH /:id` → `200`.
  - Fail / Edge Cases liên quan: Sai `size`/`status`/ObjectId → `400`; trùng nhãn `label` (`Cage.js:5`, `E11000`) → `409`; không phải `Admin` gọi tạo/sửa → `403`; không tìm thấy chuồng → `404`.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Trong `canCageAccommodatePet`, nếu chuồng có `maxWeightKg` tùy chỉnh và đồng thời có `size: 'M'` thì backend ưu tiên ngưỡng nào?"
  - *Cách trả lời*: `canCageAccommodatePet` ưu tiên ngưỡng `maxWeightKg` tùy chỉnh của chuồng nếu được khai báo; khi không có mới lấy ngưỡng chuẩn theo `size` (`S/M/L/XL`), sau đó kiểm tra thêm điều kiện `if (requiresIsolation && !isIsolated)` rồi mới trả `{ allowed, reason }`.

### Q36. [Cage Assignment / `/api/cage-assignments`] [Mức độ: Cực gắt] Hãy trace luồng phân chuồng (`POST /api/cage-assignments`) và giải phóng chuồng (`PATCH /api/cage-assignments/:id/release`). Tại sao phải dùng `findOneAndUpdate` nguyên tử trên `Cage`?
- **Ý đồ giám khảo**: Kiểm tra lập trình đồng thời (Concurrency) và đảm bảo không bao giờ bị "chuồng ma" (Occupied mà không có assignment).
- **Trả lời miệng (30–45 giây)**: Khi phân chuồng (`createCageAssignment`), service không dùng `findById` rồi `if (cage.status === 'Available')` (dễ dính race condition), mà gọi thẳng **`Cage.findOneAndUpdate({ _id: cageId, status: 'Available' }, { $set: { status: 'Occupied' } })`** (`cageAssignment.service.js:50-54`). Nếu kết quả `null`, mới kiểm tra `Cage.exists` để phân biệt `404` (không thấy chuồng) với `409` (chuồng đã `Occupied`). Tiếp theo kiểm tra `BR-11` (`L66-78`, sai thì trả chuồng về `Available` rồi ném `400`), cuối cùng tạo `CageAssignment` (có Partial Unique Index `{ cageId: 1 }` với `releasedAt: null`, lỗi `11000` cũng rollback chuồng về `Available` và trả `409`). Khi giải phóng (`release`), cập nhật `releasedAt = new Date()` và trả `Cage.status = 'Available'`.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/cageAssignment.routes.js` → `src/controllers/cageAssignment.controller.js` (đẩy lỗi qua `next(err)`) → `src/validators/cageAssignment.validator.js` → `src/services/cageAssignment.service.js:34-98` → `src/models/CageAssignment.js:15-19`.
  - Happy Path: `POST /` → `201`; `GET /`, `PATCH /:id/release` → `200`.
  - Fail / Edge Cases liên quan: ID không hợp lệ hoặc vi phạm `BR-11` → `400`; `PetOwner` gọi → `403`; không tìm thấy chuồng/assignment → `404`; chuồng đang `Occupied` hoặc đã giải phóng rồi → `409`.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Tại sao `Groomer` lại được phép giải phóng chuồng (`release`) nhưng không được tạo phân công chuồng (`POST`)?"
  - *Cách trả lời*: Vì việc xếp chuồng đầu vào là nghiệp vụ điều phối của `Receptionist`/`Admin` (và `Veterinarian` khi chuyển chuồng chờ sau khám hoặc nội trú), còn `Groomer` sau khi tắm/cắt tỉa xong cần quyền giải phóng chuồng chờ khi bàn giao thú cưng.

### Q37. [Staff Schedule / `/api/staff-schedules`] [Mức độ: Nâng cao] Tại sao lịch làm việc nhân sự dùng mô hình Blacklist (chỉ lưu ca bận)? Nếu bác sĩ đã có lịch hẹn trong ca tối mà Admin vẫn báo bận ca đó thì hệ thống chặn hay cảnh báo?
- **Ý đồ giám khảo**: Kiểm tra thiết kế nghiệp vụ thực tế và xử lý ngoại lệ bất khả kháng (bác sĩ ốm đột xuất).
- **Trả lời miệng (30–45 giây)**: Phòng khám có lịch làm việc cố định 3 ca (`MORNING`, `AFTERNOON`, `EVENING`) mỗi ngày, nên dùng mô hình **Blacklist** (mặc định nhân sự sẵn sàng làm việc, chỉ tạo record trong `staff_schedules` khi nghỉ/bận) giúp tiết kiệm >90% dữ liệu DB và không phải sinh lịch sẵn cho cả năm. Khi tạo báo bận, hệ thống **không chặn cứng** nếu bác sĩ đã có ca `Waiting`/`InProgress` trong ngày đó, mà **cho phép ghi nhận và trả kèm mảng `conflictingServices` cảnh báo** (`staffSchedule.service.js:141-165`) — vì thực tế bác sĩ ốm/có việc khẩn cấp vẫn phải báo bận được để Lễ tân thấy cảnh báo và điều phối sang bác sĩ khác, còn mọi lượt đặt lịch/phân công **mới** sau đó sẽ bị chặn `409`.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/staffSchedule.routes.js` (`GET /`, `POST /`, `POST /me`, `DELETE /:id`) → `src/controllers/staffSchedule.controller.js` → `src/validators/staffSchedule.validator.js` → `src/services/staffSchedule.service.js:102-175` (check quá khứ `L102`, check `CLINICAL_STAFF_ROLES` `L110`, bắt `11000` có lọc `err.keyPattern` `L126-139`, quét `conflictingServices` `L141-165`) → `src/models/StaffSchedule.js:17-18`.
  - Happy Path: `POST /` hoặc `POST /me` → `201` trả `{ schedule, warning, conflictingServices }`.
  - Fail / Edge Cases liên quan: Báo bận ngày quá khứ (`date < todayVN()`), sai `shift`, nhân sự không phải `Veterinarian`/`Groomer` hoặc không `Active` → `400` (`L102-115`); không thấy nhân sự → `404` (`L108`); trùng `(staffId, date, shift)` → `409` (`L132-135`).
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Tại sao dòng 129-131 `staffSchedule.service.js` lại kiểm tra kỹ `err.keyPattern` trước khi trả `409`?"
  - *Cách trả lời*: Nhóm từng gặp sự cố thật trên MongoDB Atlas khi còn sót 1 index cũ `staffId_1_workDate_1` sau khi đổi tên trường `workDate → date`; khi chưa kiểm tra `err.keyPattern`, mọi lỗi `11000` từ index thừa đều bị nuốt thành `409` báo trùng lịch sai sự thật. Thêm check `err.keyPattern` giúp phân biệt chính xác trùng lịch thật (`409`) với lỗi schema/index hệ thống (`500`).

### Q38. [Appointment Order / `/api/appointment-orders`] [Mức độ: Cực gắt] Hãy trình bày toàn bộ State Machine của `AppointmentOrder.derivedStatus` và các quy tắc đặt lịch / hủy lịch online (`UC-15`, `BR-06`, `BR-07`)?
- **Ý đồ giám khảo**: Kiểm tra nắm vững máy trạng thái trung tâm (Core State Machine) của toàn bộ phòng khám.
- **Trả lời miệng (30–45 giây)**: `AppointmentOrder.derivedStatus` đi theo luồng chính **`Booked → CheckedIn → InProgress → WaitOwner → Done`**, cộng 2 nhánh phụ là **`Booked → Overdue`** (do `JOB-01` lúc 23:59) và **`Booked/CheckedIn → Cancelled`** (khi hủy). Khi `PetOwner` đặt lịch online (`POST /api/appointment-orders/online`): kiểm tra thú cưng thuộc chủ (`403`), dịch vụ `Active`, ngày không thuộc quá khứ, giờ `slotStart` trong khung giờ hoạt động, không trùng lịch bận `StaffSchedule` nếu chọn `preferredStaffId` (`409`), tự động gửi email `bcc` báo Lễ tân (fire-and-forget). Khi hủy (`PATCH /:id/cancel`): áp quy tắc `BR-07` — `PetOwner` chỉ được hủy khi đơn còn `Booked` (chưa check-in) và **trước giờ hẹn ≥ 2 tiếng** (vi phạm trả `409`), còn `Receptionist` được hủy đơn chưa kết thúc.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/appointmentOrder.routes.js` → `src/controllers/appointmentOrder.controller.js` → `src/validators/appointmentOrder.validator.js` → `src/services/appointmentOrder.service.js` (`createOnlineAppointment`, `cancelAppointment`, `checkInAppointment`, `sendNewBookingEmail` `L52-120`) → `src/models/AppointmentOrder.js:1-45`.
  - Happy Path: `POST /online` → `201`; `GET /`, `GET /me`, `GET /me/:id`, `PATCH /:id/cancel` → `200`.
  - Fail / Edge Cases liên quan: Ngày quá khứ / sai khung giờ / dịch vụ `Inactive` → `400`; `PetOwner` xem/hủy đơn người khác → `403`; không tìm thấy đơn/pet/dịch vụ → `404`; hủy quá sát giờ (`< 2h` với `PetOwner`), đơn đã `InProgress`/`Done`/`Cancelled`, nhân sự ưu tiên báo bận ca đó → `409`.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Tại sao gọi là `derivedStatus` (trạng thái suy dẫn)? Khi nào nó chuyển sang `WaitOwner` và khi nào sang `Done`?"
  - *Cách trả lời*: Vì trạng thái của đơn tổng được suy dẫn từ tập các `AppointmentService` con: khi có ít nhất 1 dịch vụ bắt đầu (`InProgress`), đơn lên `InProgress`; khi toàn bộ dịch vụ con (trừ dịch vụ đã hủy lẻ) đều `Done` và chưa thanh toán, đơn chuyển sang **`WaitOwner`** (chờ chủ đón/thanh toán); chỉ khi hóa đơn được thanh toán **`Paid`** (`markInvoicePaidAndFinalize`), đơn mới chuyển sang **`Done`**.

### Q39. [Check-in Online / `PATCH /api/appointment-orders/:id/check-in`] [Mức độ: Cực gắt] Hãy trace từng bước của luồng Lễ tân Check-in đơn online: gán nhân sự, cấp `queueNo`, gán chuồng `Holding` và đảm bảo nguyên tử bằng Transaction?
- **Ý đồ giám khảo**: Đây là một trong 3 luồng đa bảng phức tạp nhất hệ thống — giám khảo muốn thấy khả năng trace code chính xác.
- **Trả lời miệng (30–45 giây)**: Khi khách đến quầy, Lễ tân gọi `PATCH /api/appointment-orders/:id/check-in` kèm thông tin tiếp nhận `intake` (cân nặng thực tế, triệu chứng, cờ `isEmergency`, `requiresIsolation`), phân công nhân sự cho từng dịch vụ và tùy chọn `cageId`. Service kiểm tra đơn phải đang `Booked` (`409` nếu trạng thái khác), nhân sự được gán phải `Active`, đúng `requiredRole` của dịch vụ và **không có lịch báo bận** trong ca đó (`409`), chuồng phải trống và thỏa `BR-11`. Sau đó mọi thao tác ghi 4 bảng (`AppointmentOrder` → `CheckedIn` + `checkedInAt` + `intake`, các `AppointmentService` → `Waiting` + `assignedStaffId` + `queueNo` + `enqueueAt`, tạo `CageAssignment`, đổi `Cage` → `Occupied`) đều bọc trong **`runWithTransaction`** (`appointmentOrder.service.js:935`) kèm rollback snapshot nếu chạy Standalone.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/appointmentOrder.routes.js` (`authorize('Receptionist')`) → `src/controllers/appointmentOrder.controller.js` → `src/services/appointmentOrder.service.js:767-935` (`runCheckInWrite` + `runWithTransaction`) → `src/services/cageAssignment.service.js:34-98`.
  - Happy Path: `200 OK` trả đơn đã `CheckedIn`, danh sách dịch vụ có `queueNo`, và `cageAssignment` (nếu có gán chuồng).
  - Fail / Edge Cases liên quan: Thiếu dữ liệu `intake`, sai `requiredRole`, vi phạm `BR-11` cân nặng/cách ly → `400`; không phải `Receptionist` → `403`; không thấy đơn/nhân sự/chuồng → `404`; đơn không ở `Booked`, nhân sự báo bận ca đó, chuồng đã `Occupied` → `409`.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Nếu khách đặt lịch online bị quá hạn (`Overdue` sau 23:59) rồi hôm sau mới mang thú cưng đến, có Check-in lại đơn đó được không?"
  - *Cách trả lời*: Không ạ, đơn đã `Overdue` không còn ở trạng thái `Booked` nên Check-in sẽ trả `409`. Lúc đó Lễ tân sẽ tiếp nhận mới bằng luồng **Walk-in** (`POST /api/appointment-orders/walk-in`) cho ngày hiện tại, và thông tin khách/thú cưng cũ được tra cứu tự động trong 1 giây qua `GET /walk-in/customer-lookup`.

### Q40. [Walk-in Reception / `/api/appointment-orders/walk-in`] [Mức độ: Cực gắt] Hãy trace luồng tiếp nhận khách vãng lai (`GET /customer-lookup` và `POST /walk-in`). Tại sao `GET /customer-lookup` không tìm thấy khách vẫn trả `200`, và tại sao khách mới tạo lại có `status: 'Inactive'`, `passwordHash: null`?
- **Ý đồ giám khảo**: Kiểm tra thiết kế UX/API thực tế ở quầy lễ tân và logic tạo tài khoản vãng lai.
- **Trả lời miệng (30–45 giây)**: (1) `GET /customer-lookup?phone=...` là API tra cứu nhanh ở form Lễ tân: nếu chưa có SĐT trong DB, trả **`200 { found: false }`** thay vì `404` để FE hiểu đây là trạng thái hợp lệ ("khách mới") và mở form nhập tên + thú cưng mà không báo lỗi đỏ trên console; nếu SĐT trùng nhân sự phòng khám trả `409`, khách đang bị `Locked` trả `403`. (2) Khi `POST /walk-in`, nếu khách mới sẽ tạo `User` với **`status: 'Inactive'`** và **`passwordHash: null`** để lưu hồ sơ chủ nuôi gắn với `Pet` và `Invoice` nhưng ngăn không cho đăng nhập trái phép khi chưa có mật khẩu. Toàn bộ việc tạo/ghi tối đa **6 collection** (`User`, `Pet`, `AppointmentOrder` ở luôn `CheckedIn`, `AppointmentService`, `CageAssignment`, `Cage`) được bọc trong `runWithTransaction` (`walkIn.service.js:345`).
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/appointmentOrder.routes.js:9` → `src/routes/walkIn.routes.js:13` → `src/controllers/walkIn.controller.js:53` → `src/validators/walkIn.validator.js:33-37` → `src/services/walkIn.service.js:160-452` (`lookupCustomerByPhone`, `createWalkInBooking`, `runWithTransaction` `L345`).
  - Happy Path: `GET /customer-lookup` → `200 { found: true|false, ... }`; `POST /` → `201 { message, data: { isNewCustomer, customer, isNewPet, pet, appointment, services, cageAssignment } }`.
  - Fail / Edge Cases liên quan: Cùng gửi hoặc cùng thiếu `petId` & `petInfo`, ngày quá khứ, dịch vụ `Inactive`, sai `requiredRole`, vi phạm `BR-11` → `400`; khách bị `Locked` (`L289`) hoặc không phải `Receptionist` (`walkIn.routes.js:13`) → `403`; không thấy dịch vụ/nhân sự/chuồng/pet (`L211,235,271,295`) → `404`; nhân sự bận / chuồng `Occupied` / SĐT nhân sự / trùng email (`L259,274,286,308,324`) → `409`; hết 5 lần sinh mã (`L75,99,142,441-448`) → `500`.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Khách hàng cũ đã có tài khoản `PetOwner` (`Active`) đi Walk-in mà mang thêm 1 bé mèo mới chưa có hồ sơ thì API có hỗ trợ không?"
  - *Cách trả lời*: Có ạ, `POST /walk-in` cho phép kết hợp `existingCustomer` (theo SĐT) với `petInfo` mới (`isNewCustomer: false, isNewPet: true`), tự sinh mã `PET-XXXXXX` mới gắn đúng `ownerId` của khách cũ ngay trong cùng 1 transaction.

### Q41. [Appointment Service & Queue / `/api/appointment-services`] [Mức độ: Cực gắt] Hãy trace luồng hàng đợi `GET /my-queue`, quy tắc tuần tự `BR-12`, ưu tiên ca cấp cứu (`isEmergency`), hủy dịch vụ lẻ (`BR-07`) và tạo Grooming Note?
- **Ý đồ giám khảo**: Kiểm tra nắm vững luồng thực thi dịch vụ song song giữa Bác sĩ và Groomer cùng cách xử lý ưu tiên.
- **Trả lời miệng (30–45 giây)**: (1) Trong `GET /my-queue` (`appointmentService.service.js:283-299`), hệ thống lọc `AppointmentService` của đúng `req.user.id` đang `Waiting`/`InProgress` bằng Compound Index `{ assignedStaffId: 1, status: 1, enqueueAt: 1 }`, và **sắp xếp ưu tiên `isEmergency: -1` lên đầu rồi mới tới `enqueueAt: 1` (FIFO)** để ca cấp cứu luôn được xử lý trước. (2) Theo `BR-12`, dịch vụ có `sequenceNo > 1` chỉ được bấm `start` khi dịch vụ ngay trước đó đã `Done` hoặc `Cancelled` (vi phạm chặn `409`). (3) Hủy dịch vụ lẻ (`PATCH /:id/cancel`, `L325-437`) áp `BR-07` và kiểm tra IDOR cho `PetOwner` (`403`). (4) Groomer hoàn tất dịch vụ có thể đính kèm `POST /:id/grooming-note` với 2 ảnh `beforePhoto`/`afterPhoto` stream lên Cloudinary bằng `Promise.allSettled`, và cập nhật nguyên tử có điều kiện `groomingNote: null` để chống tạo đúp, lỗi DB thì xóa sạch ảnh đã upload (`L562-575`).
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/appointmentService.routes.js:1-56` (`uploadGroomingNotePhotos` `L47-53`) → `src/validators/appointmentService.validator.js` → `src/controllers/appointmentService.controller.js:31-213` → `src/services/appointmentService.service.js:63-585`.
  - Happy Path: `POST /online`, `/online-booking`, `/:id/grooming-note` → `201`; `GET /`, `/my-queue`, `/my-grooming-notes`, `PATCH assign-staff/start/complete/cancel` → `200`.
  - Fail / Edge Cases liên quan: `slotStart` ngoài `08:00–21:00`, tổng thời lượng vượt `21:00` → `400` (`L109-153`); `PetOwner` thao tác trên đơn người khác → `403` (`L63,340`); trùng `sequenceNo` (`11000`), hủy `< 2h`, vi phạm `BR-12`, đã có Grooming Note → `409`; lỗi Cloudinary → `502` (`L582-585`).
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Tại sao `createOnlineBooking` (`L107-202`) dùng vòng `for` tạo từng `AppointmentService` tuần tự thay vì `Promise.all`?"
  - *Cách trả lời*: Vì mỗi dịch vụ trong cùng 1 đơn có `sequenceNo` tăng dần (`1, 2, 3...`) với Unique Index `{ orderId, sequenceNo }` và tính `slotStart` cộng dồn theo `durationMinutes` của dịch vụ trước; chạy tuần tự đảm bảo thứ tự chính xác và dễ dừng/báo lỗi gọn khi có xung đột.

### Q42. [Medical Record SOAP / `/api/medical-records`] [Mức độ: Cực gắt] Hãy trace vòng đời 3 bước của Bệnh án SOAP (`start` → `saveDraft` → `complete`). Tại sao mục `S` (`subjectiveNote`) bị khóa chỉ đọc, và tại sao sau `/complete` thì `MedicalRecord.status` vẫn là `Draft`?
- **Ý đồ giám khảo**: Đây là câu hỏi nghiệp vụ y khoa thú y rất sắc của hội đồng để kiểm tra sinh viên có hiểu sâu quy trình phòng khám không.
- **Trả lời miệng (30–45 giây)**: (1) `POST /service/:id/start` (`medicalRecord.service.js:147-214`) chỉ cho đúng `Veterinarian` được phân công, kiểm tra đơn đã Check-in và thỏa `BR-12`, chuyển dịch vụ sang `InProgress`, tạo `MedicalRecord` ở trạng thái `Draft` và tự copy triệu chứng ban đầu từ `AppointmentOrder.intake` sang **`subjectiveNote` (mục S)**. (2) Mục `S` bị **khóa chỉ đọc** ở `PUT /service/:id` (`saveDraft`, `L222-297`) nhằm bảo toàn nguyên trạng lời khai ban đầu của chủ nuôi lúc tiếp nhận, tránh chỉnh sửa hồ sơ y tế về sau; bác sĩ chỉ được ghi O/A/P, cân nặng, kê đơn thuốc (`Product` phải `Active` theo `BR-16`, lưu snapshot tên/đơn vị). (3) Khi bác sĩ bấm `POST /service/:id/complete` (`L308-467`), hệ thống kiểm tra đủ O-A-P (`BR-13`), kiểm tra **không còn Subtask nào chưa `Completed`** (`MSG26` → `409`), chuyển `AppointmentService` sang `Done`, cập nhật `Pet.weight`, nhưng **giữ `MedicalRecord.status` ở `Draft` cho đến khi khách thanh toán hóa đơn (`Paid`) mới chốt `Finalized`** (`BR-14`) — cho phép bác sĩ chỉnh sửa đơn thuốc/kết luận lần cuối nếu khách trao đổi thêm trước khi ra quầy thanh toán, còn đã `Finalized` thì khóa cứng vĩnh viễn (`409`).
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/medicalRecord.routes.js:1-45` → `src/validators/medicalRecord.validator.js:1-187` → `src/controllers/medicalRecord.controller.js:1-153` → `src/services/medicalRecord.service.js:147-467` → `src/models/MedicalRecord.js:1-35`.
  - Happy Path: `start`, `saveDraft`, `complete`, `getByServiceId`, `getHistoryByPetId` → `200`/`201`.
  - Fail / Edge Cases liên quan: Thiếu O/A/P (`BR-13`), thuốc `Inactive` (`BR-16`), `waitingCageId` sai tải trọng (`BR-11`), cân nặng ngoài `(0, 500]` → `400`; sai vai trò hoặc `PetOwner` xem bệnh án thú cưng người khác (`BR-23`) → `403`; chưa Check-in, dịch vụ trước chưa xong (`BR-12`), còn Subtask chưa `Completed` (`MSG26`), bệnh án đã `Finalized` (`BR-14`) → `409`.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Nếu trong lúc khám bác sĩ thấy thú cưng cần chuyển sang chuồng chờ (`waitingCageId`) ngay khi bấm `/complete` thì sao?"
  - *Cách trả lời*: `POST /complete` chấp nhận tham số tùy chọn `waitingCageId`, kiểm tra chuồng theo `BR-11` (`400` nếu quá tải trọng hoặc thiếu cách ly) và tạo `CageAssignment` tương ứng trong bước hoàn tất.

### Q43. [Subtask Request / `/api/subtask-requests`] [Mức độ: Nâng cao] Hãy trace luồng Chỉ định cận lâm sàng (`UC-33`, `UC-34`): tạo chỉ định, nhận ca, trả kết quả kèm ảnh, hủy chỉ định và chốt chặn `MSG26`?
- **Ý đồ giám khảo**: Kiểm tra luồng phối hợp đa nhân sự (Bác sĩ chính ↔ Kỹ thuật viên/Bác sĩ xét nghiệm/Groomer) và cách tránh kẹt bệnh án.
- **Trả lời miệng (30–45 giây)**: Khi ca khám đang `InProgress` và bệnh án chưa `Finalized`, bác sĩ chính tạo `POST /api/subtask-requests` (`subtaskRequest.service.js:72-170`) chỉ định đồng nghiệp thực hiện xét nghiệm/siêu âm/cắt tỉa vệ sinh (`serviceId` tùy chọn để tính phí hóa đơn, kiểm tra nhân sự `Active`, đúng role, không bận ca đó → `409`). Nhân sự được giao thấy việc trong `GET /my-queue`, gọi `PATCH /:id/start` (`Pending → InProgress`), rồi `PATCH /:id/complete` kèm ít nhất `resultText` hoặc `resultImage` (`uploadSubtaskResultPhoto` ≤ 5MB, có dọn ảnh Cloudinary nếu DB lỗi ở `L55-62`). Trong lúc còn bất kỳ Subtask nào chưa `Completed`, bác sĩ chính bị chặn **`409` (`MSG26`)** khi bấm hoàn tất khám; nếu lỡ chỉ định nhầm mà đồng nghiệp chưa nhận (`Pending`), bác sĩ tạo chỉ định được gọi `DELETE /:id` xóa nguyên tử (`L335-363`).
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/subtaskRequest.routes.js:1-61` (`uploadSubtaskResultPhoto` `L49`) → `src/validators/subtaskRequest.validator.js:50-53,167-172` (chặn trường lạ `status`/`resultImageUrl`) → `src/controllers/subtaskRequest.controller.js:34` → `src/services/subtaskRequest.service.js:55-363`.
  - Happy Path: `POST /` → `201`; `GET /my-queue`, `GET /service/:id`, `GET /:id`, `PATCH /start`, `PATCH /complete`, `DELETE /:id` → `200`.
  - Fail / Edge Cases liên quan: Gửi trường lạ `status`/`resultImageUrl`, thiếu cả `resultText` & `resultImage`, nhân sự/dịch vụ sai role → `400`; không phải BS phụ trách / không phải người được giao / không phải người tạo → `403`; ca khám chưa `InProgress`, nhân sự báo bận, xóa Subtask đã `InProgress`/`Completed`, hoàn tất đúp → `409`; lỗi Cloudinary → `502` (`L297`).
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Subtask có gắn `serviceId` thì tiền của Subtask đó có tự động lên hóa đơn cuối cùng không?"
  - *Cách trả lời*: Có ạ, khi Lễ tân lập hóa đơn (`invoice.service.js`), hệ thống quét cả các `SubtaskRequest` đã `Completed` có `serviceId` thuộc các dịch vụ của đơn khám đó để tự động cộng mục phí xét nghiệm/cận lâm sàng vào `Invoice.items`.

### Q44. [Inpatient Record / `/api/inpatient-records`] [Mức độ: Cực gắt] Hãy trace luồng Điều trị nội trú (`/api/inpatient-records`): Nhập viện (`Admitted`), ghi Daily Logs và Xuất viện (`Discharged`). Chuồng `Holding` cũ và chuồng `InpatientWard` được đồng bộ thế nào?
- **Ý đồ giám khảo**: Kiểm tra quản lý chuồng 2 phân khu (`Holding` vs `InpatientWard`), chống nhập viện trùng và chống race condition khi ghi log vs xuất viện.
- **Trả lời miệng (30–45 giây)**: Chỉ `Veterinarian` thao tác nhập viện/ghi log/xuất viện, còn `PetOwner` có riêng API đọc nhật ký `GET /pets/:petId/daily-logs` (kiểm tra ownership trả `404` nếu khác chủ ở `L331-335`). Khi `POST /api/inpatient-records` (`inpatientRecord.service.js:156-236`): kiểm tra thú cưng không có đợt `Admitted` nào khác (khóa cứng thêm bằng Partial Unique Index `{ petId: 1 }` khi `status: 'Admitted'` ở `InpatientRecord.js:28-31`), chuồng chọn bắt buộc thuộc khu **`InpatientWard`** và đạt `maxWeightKg`; nếu thú cưng đang nằm chuồng chờ `Holding` cũ thì **tự động giải phóng chuồng `Holding` cũ** và chiếm chuồng `InpatientWard` mới (có manual rollback nếu fail giữa chừng), rồi gửi email thông báo nhập viện cho chủ nuôi. Khi bấm `PATCH /:id/discharge` (`L368-389`): cập nhật nguyên tử `findOneAndUpdate({ _id, status: 'Admitted' }, { status: 'Discharged', dischargedAt: new Date() })` — **chưa mở chuồng về `Available` ngay** vì thú cưng vẫn ở tại chuồng nội trú chờ chủ đến làm thủ tục thanh toán hóa đơn (`markInvoicePaidAndFinalize` mới giải phóng chuồng về `Available`).
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/inpatientRecord.routes.js:1-16` → `src/validators/inpatientRecord.validator.js:1-106` → `src/controllers/inpatientRecord.controller.js:1-100` → `src/services/inpatientRecord.service.js:156-389` → `src/models/InpatientRecord.js:28-31`.
  - Happy Path: `POST /`, `POST /:id/daily-logs` → `201`; `GET /`, `GET /:id`, `GET /:id/daily-logs`, `GET /pets/:petId/daily-logs`, `PATCH /:id/discharge` → `200`.
  - Fail / Edge Cases liên quan: Đơn không thuộc pet, đơn `Cancelled`, chuồng không phải `InpatientWard`, quá `maxWeightKg`, ghi Daily Log cho hồ sơ đã `Discharged` hoặc ngày tương lai / trước ngày nhập viện → `400` (`L165-178`); thú cưng đang `Admitted` (`11000`), chuồng `Occupied`, xuất viện lại lần 2 → `409` (`L172,186,224`).
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Nếu bác sĩ A đang bấm thêm Daily Log cùng mili-giây bác sĩ B bấm Xuất viện thì có bị ghi lọt log sau khi xuất viện không?"
  - *Cách trả lời*: Để chống race condition này, cả thao tác thêm Daily Log lẫn `discharge` đều kiểm tra trạng thái `status: 'Admitted'` trực tiếp trong điều kiện nguyên tử (`findOneAndUpdate({ _id, status: 'Admitted' }, ...)`), nên nếu lệnh `discharge` thắng trước thì lệnh thêm log lập tức thất bại và trả `400`/`409`.

### Q45. [Invoice Creation & Retail Edit / `/api/invoices`] [Mức độ: Cực gắt] Hãy trace luồng lập hóa đơn tổng hợp (`POST /api/invoices`) và sửa đồ bán lẻ (`PATCH /:id/items`). Khi khách vừa khám vừa nhập viện trong cùng 1 đơn, làm sao tránh tính tiền nội trú 2 lần, và vì sao chỉ cho xóa dòng `Product`?
- **Ý đồ giám khảo**: Soi trực tiếp logic tài chính nhạy cảm nhất dự án (tính tiền sai = lỗi nghiêm trọng nhất).
- **Trả lời miệng (30–45 giây)**: (1) `POST /api/invoices` (`invoice.service.js:179-412`) cho phép `Receptionist` lập hóa đơn từ `orderId`, `inpatientRecordId`, và/hoặc `retailItems`: đơn khám phải ở `WaitOwner` (`409` nếu chưa xong), đợt nội trú phải đã `Discharged` (`409` nếu vẫn đang `Admitted`), và mỗi `orderId` hay `inpatientRecordId` chỉ lập được tối đa 1 hóa đơn nhờ 2 Partial Unique Index (`Invoice.js:33-40` → `E11000` trả `409`). (2) Nếu cùng 1 đơn vừa khám vừa nhập viện nội trú, service gom tổng hợp các dịch vụ đã `Done`, các `SubtaskRequest` có phí, thuốc kê đơn trong `MedicalRecord`, và tự tính số ngày lưu trú (`InpatientDay`) từ `InpatientRecord` gắn với đơn đó (lọc bỏ không cộng đúp dịch vụ nội trú). Mọi đơn giá đều đọc từ DB và lưu snapshot. (3) Trên `PATCH /:id/items` (`L417-472`), chỉ sửa được khi hóa đơn còn `Unpaid` (`409`), và **chỉ cho gỡ các dòng `type === 'Product'`** — cố tình gỡ dòng `Service` hay `InpatientDay` bị chặn **`400`** (`L437-439`) để chống lễ tân xóa bớt phí khám/nội trú đã thực hiện gây thất thoát doanh thu.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/invoice.routes.js:1-41` → `src/validators/invoice.validator.js:1-364` → `src/controllers/invoice.controller.js:35` → `src/services/invoice.service.js:179-472` → `src/models/Invoice.js:3-48`.
  - Happy Path: `POST /` → `201 { message: 'Lập hóa đơn thành công', data }`; `PATCH /:id/items` → `200` kèm `totalAmount` tính lại.
  - Fail / Edge Cases liên quan: Sản phẩm `Inactive`, cố gỡ dòng `Service`/`InpatientDay`, làm rỗng hoàn toàn hóa đơn → `400` (`L437-439`); đơn chưa `WaitOwner`, nội trú chưa `Discharged`, đã lập hóa đơn rồi (`11000`), hóa đơn đã `Paid` → `409`; `DELETE /:id` → `403` (`invoice.controller.js:295`).
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Lỡ lập hóa đơn 0 đồng (ví dụ tái khám miễn phí) rồi bấm tạo link thanh toán PayOS/VNPay thì sao?"
  - *Cách trả lời*: Cổng thanh toán trực tuyến không chấp nhận giao dịch `amount <= 0`, nên cả `createVnpayUrl` và `createPayosUrl` đều kiểm tra `invoice.totalAmount <= 0` để chặn **`400`** và yêu cầu chốt bằng `pay-cash` với `cashReceived: 0`.

### Q46. [Invoice Finalization / `markInvoicePaidAndFinalize`] [Mức độ: Cực gắt] Khi thanh toán thành công (Tiền mặt, PayOS hay VNPay), hàm lõi `markInvoicePaidAndFinalize` đồng bộ những bảng nào trong cùng 1 transaction?
- **Ý đồ giám khảo**: Kiểm tra điểm hội tụ cuối cùng của toàn bộ vòng đời khám chữa bệnh (End-to-End Consistency).
- **Trả lời miệng (30–45 giây)**: Cả 3 kênh thanh toán (Tiền mặt `pay-cash`, PayOS Webhook/Polling, VNPay IPN) đều gọi chung 1 hàm lõi duy nhất **`markInvoicePaidAndFinalize`** (`invoice.service.js:480-616`) bọc trong `runWithTransaction` (`L592`). Bên trong transaction, nó thực hiện 5 đồng bộ nguyên tử: (1) Cập nhật `Invoice` với điều kiện `{ _id, paymentStatus: 'Unpaid' }` sang `Paid` + `paidAt` + `paymentMethod` (chống thanh toán đúp); (2) Chuyển `AppointmentOrder.derivedStatus` sang **`Done`**; (3) Chốt toàn bộ `MedicalRecord` của đơn đó từ `Draft` sang **`Finalized`** (`BR-14`); (4) Giải phóng mọi `CageAssignment` còn mở (`releasedAt = new Date()`) của đơn hoặc đợt nội trú; (5) Cập nhật lại `Cage.status` tương ứng về **`Available`**.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/services/invoice.service.js:480-616` (`markInvoicePaidAndFinalize` + `runWithTransaction` `L592`) được gọi từ `payCash` (`L621+`), `handleVnpayIpn`, `handlePayosWebhook`, `syncPayosPaymentStatus`.
  - Happy Path: `POST /:id/pay-cash` trả `200` kèm `changeAmount = cashReceived - totalAmount` và toàn bộ 5 bảng đã đồng bộ sạch sẽ.
  - Fail / Edge Cases liên quan: Tiền khách đưa `cashReceived < totalAmount` → `400`; hóa đơn đã `Paid` trước đó → `409` (ở `pay-cash`) hoặc xử lý idempotent (ở Webhook/IPN).
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Nếu 1 hóa đơn là hóa đơn bán lẻ thuần (`orderId: null`, `inpatientRecordId: null`) thì hàm này có bị lỗi null pointer ở các bước sau không?"
  - *Cách trả lời*: Không ạ, từng bước đồng bộ đơn khám, bệnh án và giải phóng chuồng đều kiểm tra `if (invoice.orderId)` / `if (invoice.inpatientRecordId)` trước khi chạy, nên hóa đơn bán lẻ thuần chỉ chuyển `Invoice` sang `Paid` gọn gàng.

### Q47. [Reports & Analytics / `/api/reports`] [Mức độ: Nâng cao] Hãy trace 3 API báo cáo (`revenue`, `operations`, `front-desk`): tại sao doanh thu lọc theo `paidAt`, xử lý zero-filling và chống lỗi chia 0 (`NaN`/`Infinity`) ra sao?
- **Ý đồ giám khảo**: Kiểm tra kỹ năng phân tích số liệu, Aggregation Pipeline và xử lý biên toán học.
- **Trả lời miệng (30–45 giây)**: `report.routes.js` mở `/revenue` & `/operations` cho `Admin`, còn `/front-desk` cho `Receptionist` & `Admin` (mặc định `period = 'month'` cho Admin và `'day'` cho quầy lễ tân). (1) Doanh thu (`getRevenueAnalytics`, `report.service.js:135-331`) lọc theo **`paymentStatus: 'Paid'` và `paidAt`** chứ không phải `issuedAt`, vì tiền chỉ thực thu tại thời điểm khách trả tiền (tránh ghi nhận khống hóa đơn `Unpaid` hoặc lệch tháng khi lập cuối tháng trước nhưng trả đầu tháng sau), đồng thời tận dụng Compound Index `{ paymentStatus: 1, paidAt: -1 }`. (2) Khi vẽ biểu đồ chuỗi thời gian (`groupBy: 'day' | 'month'`), service tự chạy **zero-filling** điền `0` cho những ngày/tháng không phát sinh giao dịch. (3) Khi tính `% tăng trưởng` so với kỳ trước, nếu kỳ trước bằng `0`, code chủ động nhánh riêng trả `null` hoặc `100`/`0` thay vì chia cho `0` gây `Infinity`/`NaN`.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/report.routes.js:1-14` → `src/validators/report.validator.js:51-156` (validate `period`, `from`/`to <= 366 ngày`, `groupBy`) → `src/controllers/report.controller.js:1-91` → `src/services/report.service.js:135-672` (`getRevenueAnalytics`, `getOperationalAnalytics` chạy song song 6 query, `getFrontDeskStatistics`).
  - Happy Path: `200 OK` trả đầy đủ KPI, chuỗi thời gian đã zero-filling, tỷ lệ tăng trưởng, thời gian chờ trung bình, tỷ lệ sử dụng chuồng.
  - Fail / Edge Cases liên quan: `from > to`, khoảng lọc `> 366 ngày`, sai `period`/`groupBy` → `400`; thiếu token → `401`; sai role (vd `Receptionist` gọi `/revenue`) → `403` (`report.routes.js:8-12`).
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Báo cáo Top sản phẩm bán chạy nằm ở `/api/reports` hay ở đâu?"
  - *Cách trả lời*: Nằm ở `GET /api/invoices/reports/best-selling-products` (`invoice.routes.js:16-21`, `invoice.service.js:1183-1262`, chỉ `Admin`), vì nó tổng hợp trực tiếp từ mảng `Invoice.items` có `type: 'Product'` và `paymentStatus: 'Paid'`, cũng dùng chung quy tắc dải ngày ≤ 366 ngày.

### Q48. [Community Posts & Interactions / `/api/community-posts`] [Mức độ: Cực gắt] Hãy trace luồng Cộng đồng (`communityPost` + `communityInteraction`): làm sao chứng minh `TimelineShare` (`BR-18`) không lộ dữ liệu y tế nhạy cảm, reply 1 cấp (`BR-19`) và quy tắc thu hồi bài (`withdraw`)?
- **Ý đồ giám khảo**: Kiểm tra bảo vệ quyền riêng tư y tế trên mạng xã hội nội bộ và xử lý state machine bài viết.
- **Trả lời miệng (30–45 giây)**: (1) Bảng tin công khai (`GET /public`, `GET /public/:id`, `GET /:id/comments`) mở cho cả Guest không cần token, còn 13 API tạo/sửa/tương tác chỉ cho `PetOwner`. (2) Ở luồng **`POST /timeline-share`** (`UC-49`, `BR-18`), server **tự xây dựng mảng `timelineItems` từ các đơn đã `Done`** của thú cưng thuộc đúng `req.user.id`, chỉ trích xuất duy nhất ngày khám, tên dịch vụ và ảnh Grooming nếu có — **tuyệt đối không bao giờ đọc hay nhúng bệnh án SOAP, đơn thuốc, giá tiền hay hóa đơn** vào nội dung chia sẻ. (3) Tương tác (`communityInteraction.service.js`): Like khóa trùng bằng Unique Index `{ postId, userId }`; Comment chỉ cho **reply 1 cấp** (`BR-19`: nếu `parentComment.parentCommentId !== null` trả `400`); Report khóa trùng `{ reporterId, targetType, targetId }` (`11000 → 409`) và chặn tự báo cáo bài của chính mình (`400`). (4) Quy tắc sửa/xóa/thu hồi: chỉ được sửa/xóa khi bài còn `PendingReview` (`409` nếu khác); bài đã `Published` có thể tự thu hồi (`withdraw` → `Removed`), nhưng **bài đang bị Admin ẩn (`Hidden`) do vi phạm thì không cho chủ bài tự thu hồi (`409`)** để giữ bằng chứng kiểm duyệt.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/communityPost.routes.js:1-38` (`uploadCommunityPostPhotos` trả `415` nếu gửi JSON) → `src/controllers/communityPost.controller.js` & `communityInteraction.controller.js` (cả 2 đẩy lỗi `next(err)`) → `src/services/communityPost.service.js:1-340` & `src/services/communityInteraction.service.js:1-288`.
  - Happy Path: `POST /`, `/timeline-share`, `/:id/comments`, `/reports` → `201`; `GET`, `PUT`, `PATCH /withdraw`, `POST /like`, `DELETE` → `200`.
  - Fail / Edge Cases liên quan: Không gửi `multipart/form-data` lên `POST/PUT /` → `415` (`upload.middleware.js:117`); reply cấp 2, tự report chính mình, chia sẻ đơn chưa `Done` → `400`; khác `PetOwner` / xóa comment người khác / chia sẻ pet chủ khác → `403`; sửa bài không còn `PendingReview`, thu hồi bài `Hidden`/`Removed`, report trùng (`11000`) → `409`; lỗi Cloudinary → `502` (`communityPost.service.js:31`).
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Tại sao `POST /api/community-posts` bắt buộc gửi `multipart/form-data` (trả `415` nếu gửi JSON) trong khi các API upload khác lại cho phép gửi JSON?"
  - *Cách trả lời*: Vì bài viết cộng đồng hỗ trợ 1 ảnh bìa + tối đa 10 ảnh mốc timeline (`timelineImages`), bắt buộc mọi ảnh phải được tải qua Cloudinary của server chứ không nhận URL ảnh ngoài do client truyền trong JSON (chống chèn link độc hại/hotlink ảnh bẩn).

### Q49. [Admin Community Moderation & AI Assessment / `/api/admin/community`, `/api/ai-assessments`] [Mức độ: Nâng cao] Hãy trace 2 module chuyên biệt: (1) Admin kiểm duyệt cộng đồng (`UC-57..59`) xử lý báo cáo hàng loạt thế nào? (2) `POST /api/ai-assessments` (`UC-14`) gom ngữ cảnh những bảng nào trước khi gọi AI?
- **Ý đồ giám khảo**: Kiểm tra kỹ năng tối ưu query chống N+1 và độ sâu kỹ thuật của tính năng AI (Context-Aware Triage chứ không phải gọi chatbot rỗng).
- **Trả lời miệng (30–45 giây)**: (1) Ở `communityModeration.service.js`: hàng đợi duyệt bài (`GET /posts`) tự chạy `detectSensitiveKeywords` (`L10-49`) gắn cờ gợi ý từ khóa nhạy cảm để hỗ trợ Admin (không tự xóa mù quáng); duyệt bài dùng `findOneAndUpdate({ _id, status: 'PendingReview' })` (`L92-121`) chống 2 Admin duyệt đúp (`409`); danh sách báo cáo (`GET /reports`, `L127-205`) gom ID bài viết & bình luận truy vấn song song bằng `$in` chống N+1; và khi Admin xử lý 1 báo cáo (`PATCH /reports/:id/resolve`, `L212-285`) chọn `Hide`/`Delete`, hệ thống **đồng loạt đóng `Resolved` cho toàn bộ các báo cáo `Pending` khác cùng trỏ vào đối tượng đó** để Admin không phải bấm xử lý 5 lần nếu 5 người báo cáo cùng 1 bài. (2) Ở `aiAssessment.service.js` (`L71-250`): chỉ `PetOwner` gọi cho thú cưng của mình (`403` nếu sai chủ), kiểm tra Magic Bytes JPG/PNG (`MSG35`), sau đó **gom ngữ cảnh thực tế từ 8 bảng** (hồ sơ `Pet`, lịch sử `AppointmentOrder`/`AppointmentService`, `MedicalRecord`, `SubtaskRequest`, `InpatientRecord`, tiền sử `AiAssessment` gần nhất, và danh sách `Service` đang `Active` trong DB) để dựng prompt chuẩn thú y, gọi AI trước rồi mới upload ảnh Cloudinary và lưu vào collection riêng `ai_assessments` (`BR-20`).
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/communityModeration.routes.js:1-15` → `src/services/communityModeration.service.js:10-285`; và `src/routes/aiAssessment.routes.js:1-15` → `src/validators/aiAssessment.validator.js:18-26,137` → `src/services/aiAssessment.service.js:71-250` → `src/utils/gemini.js:561-658`.
  - Happy Path: Admin moderation trả `200`; `POST /api/ai-assessments` trả `201 { message, assessment }` kèm `disclaimer` (`MSG33`), `aiProvider`, `aiModel`, `responseTimeMs`.
  - Fail / Edge Cases liên quan: Admin reject/hide/delete thiếu lý do → `400`; bài không còn `PendingReview` hoặc báo cáo đã `Resolved` → `409`; AI mô tả `< 10` hoặc `> 1000` ký tự, `> 3` ảnh, sai Magic Bytes → `400`; lỗi Cloudinary → `502` (`aiAssessment.service.js:58`); tất cả key AI lỗi/hết quota → `503` (`gemini.js:573,657`); quá 30s → `504` (`gemini.js:435,512,657`).
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Tính năng AI của nhóm khác gì việc khách ra ngoài mở ChatGPT hỏi một câu?"
  - *Cách trả lời*: Khác ở 4 điểm kỹ thuật: (1) AI tự động được nạp hồ sơ sức khỏe thật từ 8 bảng của thú cưng (loài, giống, tuổi, cân nặng, bệnh sử, thuốc đã dùng); (2) Prompt chuẩn phân loại thú y **AAHA/AVMA Teletriage** bắt buộc xuất JSON có cấu trúc gồm mức độ khẩn cấp (`Routine/Monitor/Urgent`) và cấm kê thuốc (`BR-20`); (3) Chống bịa dịch vụ bằng cách đối chiếu ngược danh mục `Service` thật của phòng khám và xuất sẵn `bookingNoteDraft` đổ thẳng vào form đặt lịch; (4) Có failover đa key/đa provider dưới 30s và tách collection riêng khỏi `MedicalRecord`.

---

## Phần 5: Tích hợp bên thứ 3 & Background Jobs

### Q50. [3rd-Party / PayOS v2] [Mức độ: Cực gắt] Hãy trace toàn bộ luồng thanh toán PayOS (tạo link QR, xác thực Webhook và đối chiếu `payos-status`). Tại sao cần cả Webhook lẫn API polling `payos-status`?
- **Ý đồ giám khảo**: Kiểm tra hiểu biết sâu về bảo mật cổng thanh toán thật (HMAC-SHA256, timing-safe comparison, timeout) và xử lý thực tế môi trường local/production.
- **Trả lời miệng (30–45 giây)**: (1) Tạo link (`POST /api/invoices/:id/payos-url`): kiểm tra hóa đơn `Unpaid` và `totalAmount > 0`, sinh `payosOrderCode` nguyên duy nhất (có Partial Unique Index `Invoice.js:45-48`), ký **HMAC-SHA256** đúng 5 trường xếp alphabet (`amount`, `cancelUrl`, `description` cắt ≤25 ký tự, `orderCode`, `returnUrl`) ở `payos.js:55-58`, gọi PayOS API v2 bằng native `fetch` + `AbortController` giới hạn **15 giây** (`L78-114`: timeout ném `504`, lỗi upstream ném `502`). (2) Webhook (`POST /api/invoices/payos-webhook`, public): sắp xếp key của `webhookBody.data` theo alphabet (`L30-36`), ghép chuỗi query (`L38-50`), ký lại HMAC-SHA256 và đối chiếu bằng **`crypto.timingSafeEqual`** (`L75`) chống Timing Attack; hợp lệ và đúng số tiền mới gọi `markInvoicePaidAndFinalize`. (3) API polling `GET /api/invoices/:id/payos-status`: gọi trực tiếp `GET /v2/payment-requests/{orderCode}` lên PayOS (`L159-163`) để chốt `Paid` ngay cả khi chạy demo trên `localhost` (nơi Webhook PayOS không bắn về được) hoặc khi Webhook bị trễ mạng.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/invoice.routes.js:8-30` → `src/controllers/invoice.controller.js:144-187` → `src/services/invoice.service.js` → `src/utils/payos.js:5-163`.
  - Happy Path: `POST /payos-url` → `200` trả `checkoutUrl`, `qrCode`, `orderCode`; `POST /payos-webhook` → `200`; `GET /payos-status` → `200` kèm trạng thái mới nhất.
  - Fail / Edge Cases liên quan: Thiếu 3 env `PAYOS_*` → `500` (`payos.js:18-24`); hóa đơn đã `Paid` → `409`; `totalAmount <= 0` hoặc sai chữ ký Webhook → `400`; PayOS trả lỗi hoặc `code !== '00'` → `502` (`payos.js:96-101`); vượt 15s → `504` (`payos.js:105-109`).
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Tại sao so sánh chữ ký HMAC phải dùng `crypto.timingSafeEqual` thay vì dấu `===` bình thường?"
  - *Cách trả lời*: Vì phép `===` dừng so sánh ngay ở ký tự đầu tiên khác nhau, tạo chênh lệch thời gian nano-giây cho phép kẻ tấn công thực hiện Timing Attack dò từng byte của chữ ký. `crypto.timingSafeEqual` luôn duyệt hết độ dài Buffer trong thời gian hằng số, đúng chuẩn bảo mật cho Payment Webhook.

### Q51. [3rd-Party / Payment Idempotency] [Mức độ: Cực gắt] Nếu cổng PayOS hoặc VNPay bắn lại cùng một Webhook/IPN 2–3 lần liên tiếp (do timeout mạng), hoặc Lễ tân vừa bấm `payos-status` đúng lúc Webhook tới thì hệ thống có xử lý đúp không?
- **Ý đồ giám khảo**: Idempotency & Concurrency trong thanh toán là tiêu chuẩn sống còn của Backend Engineer.
- **Trả lời miệng (30–45 giây)**: Không bao giờ xử lý đúp ạ. Thứ nhất, trước khi chốt thanh toán, service kiểm tra nếu `invoice.paymentStatus === 'Paid'` thì lập tức trả kết quả thành công idempotent (VNPay IPN trả `RspCode: '02'` / PayOS trả `200` không chạy lại finalize). Thứ hai, ngay cả khi 2 luồng (Webhook và `payos-status`) lọt qua bước đọc cùng 1 mili-giây, bên trong `markInvoicePaidAndFinalize` (`invoice.service.js:480-616`) cập nhật `Invoice` bằng điều kiện nguyên tử **`{ _id: invoice._id, paymentStatus: 'Unpaid' }`**, nên chỉ đúng 1 luồng thắng và thực thi tiếp việc chốt `AppointmentOrder`, `MedicalRecord`, giải phóng `Cage`; luồng thứ 2 thấy không khớp `Unpaid` sẽ dừng an toàn.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/services/invoice.service.js:480-616` (`markInvoicePaidAndFinalize` + `runWithTransaction` `L592`) → `tests/invoice.test.js` (đã test kịch bản gửi lại IPN/Webhook lần 2).
  - Happy Path: Lần 1 chốt `Paid` + đồng bộ 5 bảng; lần 2+ không làm thay đổi thêm trạng thái hay `paidAt`.
  - Fail / Edge Cases liên quan: Nếu Webhook giả mạo đổi số tiền khác `invoice.totalAmount`, bị chặn ngay trước khi chạm `markInvoicePaidAndFinalize` (VNPay trả `RspCode: '04'`, PayOS trả `400`).
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Nếu Lễ tân tạo link PayOS lần 1 rồi khách chưa quét, 20 phút sau link cũ hết hạn muốn tạo link PayOS lần 2 cho cùng hóa đơn đó thì có bị dính lỗi `E11000` trên `payosOrderCode` không?"
  - *Cách trả lời*: Mỗi lần tạo link mới hệ thống sinh 1 `payosOrderCode` mới (khác mã cũ) và cập nhật đè vào đúng document hóa đơn đó, nên không trùng với bất kỳ hóa đơn nào khác và không dính `E11000`.

### Q52. [3rd-Party / VNPay Legacy] [Mức độ: Nâng cao] Trong codebase vẫn còn 2 route VNPay (`vnpay-ipn` và `vnpay-return`). Tại sao `vnpay-return` tuyệt đối không được cập nhật `Paid`, và tại sao `vnpay-ipn` luôn trả HTTP `200`?
- **Ý đồ giám khảo**: Kiểm tra hiểu rõ giao thức cổng thanh toán và lỗ hổng kinh điển "tin Return URL từ trình duyệt".
- **Trả lời miệng (30–45 giây)**: (1) `vnpay-return` là URL trình duyệt của khách redirect về, nên khách hoàn toàn có thể tự gõ hoặc giả lập query `vnp_ResponseCode=00` trên thanh địa chỉ — do đó theo `BR-17`, `vnpay-return` (`invoice.controller.js:126-138`) chỉ xác thực chữ ký HMAC-SHA512 để hiển thị UI thành công/thất bại chứ **tuyệt đối không sửa DB**. Chỉ `vnpay-ipn` (server-to-server do VNPay gọi trực tiếp vào BE) mới được phép chuyển hóa đơn sang `Paid`. (2) Theo tài liệu kỹ thuật VNPay v2.1.0, server VNPay đọc kết quả dựa vào trường JSON **`RspCode`** (`'00'`, `'01'`, `'02'`, `'04'`, `'97'`, `'99'`) nằm trong HTTP **`200`**; nếu BE trả `400`/`500`, VNPay coi là lỗi đường truyền và tiếp tục retry liên tục.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/invoice.routes.js:8-9` → `src/controllers/invoice.controller.js:117-138` → `src/services/invoice.service.js:697-725` → `src/utils/vnpay.js:72-121` (`buildVnpayPaymentUrl` ký HMAC-SHA512 + `vnp_Amount * 100` + UTC+7 15 phút; `verifyVnpaySignature` dùng `crypto.timingSafeEqual` `L120`).
  - Happy Path: `GET /vnpay-ipn` → `200 { RspCode: '00', Message: 'Confirm Success' }`; `GET /vnpay-return` → `200` trả trạng thái hiển thị.
  - Fail / Edge Cases liên quan: Sai chữ ký → IPN trả `200 { RspCode: '97' }`; sai số tiền → `RspCode: '04'`; đã `Paid` → `RspCode: '02'`; không thấy đơn → `RspCode: '01'`; exception hệ thống → `RspCode: '99'` (`invoice.controller.js:121-124`).
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Dự án đã đổi sang PayOS là cổng chính, sao vẫn giữ code VNPay trong `VCMOS_BE`?"
  - *Cách trả lời*: Ban đầu nhóm tích hợp VNPay Sandbox thành công đầy đủ unit test, sau đó chuyển sang PayOS để demo quét mã VietQR thật chuyển khoản ngân hàng thật tiện hơn. Nhóm giữ lại lớp VNPay như phương án dự phòng và đã cập nhật đồng bộ tài liệu Report 3/Report 4 sang PayOS (chi tiết nêu thêm ở `Q74`).

### Q53. [3rd-Party / PDFMake] [Mức độ: Cơ bản] `GET /api/invoices/:id/pdf` xuất biên lai PDF như thế nào? Làm sao hiển thị đúng dấu tiếng Việt và không để lại file rác trên ổ cứng server?
- **Ý đồ giám khảo**: Kiểm tra xử lý binary stream, font tiếng Việt và an toàn tài nguyên server.
- **Trả lời miệng (30–45 giây)**: `invoice.service.js:30-37` khởi tạo `PdfPrinter` đúng **1 lần ở module scope** nạp 4 biến thể font **Roboto** từ `pdfmake/build/vfs_fonts` (hỗ trợ đầy đủ Unicode tiếng Việt có dấu). Khi có request `GET /api/invoices/:id/pdf`, hàm `generateInvoicePdf` (`L1058-1174`) kiểm tra tồn tại (`404`) + quyền truy cập (`assertCanAccessInvoice` chống IDOR trả `403`), dựng `docDefinition` từ **dữ liệu snapshot** `invoice.items`, gom trực tiếp các chunk thành `Buffer` ngay trong RAM (`L1161-1168`, không ghi bất kỳ file `.pdf` tạm nào ra đĩa), rồi controller (`invoice.controller.js:260-263`) gửi về với header `Content-Type: application/pdf` và `Content-Disposition: inline; filename="INV-XXXXXX.pdf"`.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/routes/invoice.routes.js` → `src/controllers/invoice.controller.js:260-263` → `src/services/invoice.service.js:30-37,1058-1174`.
  - Happy Path: `200 OK` binary PDF (kiểm chứng chữ ký `%PDF-` trong `tests/invoice.test.js`).
  - Fail / Edge Cases liên quan: Sai ObjectId → `400`; `PetOwner` tải PDF hóa đơn của người khác → `403`; không tìm thấy hóa đơn → `404`.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Nếu hóa đơn có cả trăm dòng sản phẩm thì gom cả PDF vào RAM `Buffer` có tốn bộ nhớ không?"
  - *Cách trả lời*: Biên lai phòng khám thú y thông thường chỉ dài 1–2 trang (< 50KB), gom vào `Buffer` rất gọn và dễ test bằng Supertest. Nếu sau này xuất báo cáo tổng hợp hàng trăm trang, có thể pipe thẳng `pdfDoc.pipe(res)` dạng streaming để tiết kiệm RAM.

### Q54. [3rd-Party / Cloudinary & Orphan Cleanup] [Mức độ: Cực gắt] 5 luồng upload ảnh đi qua `multer.memoryStorage()` lên Cloudinary ra sao? Nếu upload ảnh xong mà ghi DB bị lỗi thì ảnh mồ côi trên Cloudinary được xử lý thế nào?
- **Ý đồ giám khảo**: Kiểm tra xử lý lỗi phân tán giữa dịch vụ bên ngoài (Cloudinary) và Database (MongoDB).
- **Trả lời miệng (30–45 giây)**: Cả 5 luồng upload (`Pet`, `Grooming`, `Subtask`, `Community`, `AI`) đều dùng `multer.memoryStorage()` (≤5MB/ảnh) rồi stream `Buffer` trực tiếp lên Cloudinary bằng `cloudinary.uploader.upload_stream` (không tạo file rác trên ổ đĩa server), và luôn gắn middleware upload **SAU `protect` + `authorize`** để người lạ không upload được. Khi DB lỗi sau bước upload: (1) `pet.service.js:62-70,104` gọi `removePhoto` xóa ảnh vừa đẩy; (2) `appointmentService.service.js:540-543,562-575` dùng `Promise.allSettled` và gọi `deleteCloudinaryImages(uploadedPublicIds)` nếu ghi `groomingNote` thất bại; (3) `subtaskRequest.service.js:55-62` gọi `deleteCloudinaryImage`; (4) `aiAssessment.service.js:212-223` chủ động gọi AI thành công **TRƯỚC** rồi mới đẩy ảnh lên Cloudinary để nếu AI timeout (`503`/`504`) sẽ không đẩy rác. *(Còn khoảng hở nhỏ ở `communityPost` và bước `.create()` của `aiAssessment` em thừa nhận thẳng ở `Q67`).*
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/config/cloudinary.js:1-17` (`secure: true` `L14`) → `src/middlewares/upload.middleware.js:1-156` & `src/routes/pet.routes.js:13-37` → 5 services tương ứng.
  - Happy Path: Trả HTTPS URL Cloudinary lưu vào MongoDB.
  - Fail / Edge Cases liên quan: Quá 5MB / quá số ảnh / sai field / sai MIME → `400` (`upload.middleware.js:64-74`, `pet.routes.js:29-32`); gửi JSON cho Community → `415` (`upload.middleware.js:117`); lỗi mạng/API Cloudinary → `502` (`aiAssessment.service.js:58`, `communityPost.service.js:31`, `subtaskRequest.service.js:297`, `appointmentService.service.js:582-585`).
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Tại sao `Grooming Note` lại upload 2 ảnh bằng `Promise.allSettled` thay vì `Promise.all`?"
  - *Cách trả lời*: Vì với `Promise.all`, nếu ảnh #1 upload thành công còn ảnh #2 lỗi, `Promise.all` reject ngay mà không trả lại `publicId` của ảnh #1 để xóa. Dùng `Promise.allSettled` (`appointmentService.service.js:540-543`) cho phép gom lại đúng danh sách ảnh đã upload thành công để gọi `deleteCloudinaryImages` dọn sạch trước khi ném `502`.

### Q55. [3rd-Party / Nodemailer SMTP] [Mức độ: Cơ bản] Hệ thống gửi Email ở những luồng nào? Nếu Gmail SMTP bị sập hoặc chưa cấu hình `.env` thì có làm hỏng nghiệp vụ đặt lịch hay nhập viện không?
- **Ý đồ giám khảo**: Kiểm tra khả năng chịu lỗi (Fault Tolerance / Graceful Degradation) đối với dịch vụ phụ.
- **Trả lời miệng (30–45 giây)**: Hệ thống có 3 luồng gửi email qua Nodemailer Gmail SMTP: (1) Gửi mã OTP 6 số (`user.service.js:26-70`); (2) Thông báo lịch hẹn online mới cho danh sách Lễ tân bằng `bcc` (`appointmentOrder.service.js:52-120`); (3) Thông báo Nhập viện/Xuất viện cho Chủ nuôi (`inpatientRecord.service.js:26-70`). Cả 3 luồng đều kiểm tra `if (!emailUser || !emailPass || process.env.NODE_ENV === 'test')`: môi trường test im lặng (`return false`), môi trường dev chưa điền SMTP thì chỉ `console.log`. Đặc biệt 2 luồng thông báo (Đặt lịch & Nội trú) chạy dạng **fire-and-forget + try/catch log lỗi**, nên nếu Gmail SMTP gặp sự cố thì đơn hẹn và hồ sơ nội trú vẫn lưu thành công bình thường.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/services/user.service.js:26-70`, `src/services/appointmentOrder.service.js:52-120` (`bcc` `L79-80`, `escapeHtml` `L44-50`), `src/services/inpatientRecord.service.js:26-70` (`escapeHtml` `L17-18`).
  - Happy Path: Gửi email thành công, nghiệp vụ chính trả `200`/`201` không bị chặn.
  - Fail / Edge Cases liên quan: Lỗi SMTP ở luồng Đặt lịch/Nội trú chỉ ghi `console.error`, không làm fail transaction DB.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Nếu cần gửi email đảm bảo không bao giờ mất (vd gửi hóa đơn/OTP khi SMTP tạm nghẽn) ở hệ thống lớn hơn, anh sẽ làm gì?"
  - *Cách trả lời*: Em sẽ tách gửi email ra khỏi request cycle bằng Message Queue (như **BullMQ + Redis**) với cơ chế exponential backoff retry và chuyển từ Gmail SMTP cá nhân sang dịch vụ transactional email chuyên dụng (Resend / SendGrid / AWS SES).

### Q56. [3rd-Party / AI Failover & Timeout] [Mức độ: Cực gắt] Cơ chế xoay vòng nhiều API Key Gemini & Groq trong `src/utils/gemini.js` hoạt động như thế nào để bảo đảm tổng thời gian chờ luôn `<= 30s`? Khi có ảnh gửi lên, Groq xử lý ra sao?
- **Ý đồ giám khảo**: Kiểm tra kỹ thuật Resilience (Failover, Timeout Budget, Multimodal Routing) cực kỳ thực tế.
- **Trả lời miệng (30–45 giây)**: `collectProviderApiKeys` (`gemini.js:50-79`) thu thập cả 2 kiểu khai báo `.env` (phẩy `GEMINI_API_KEY=k1,k2` và đánh số `GEMINI_API_KEY_1..N`, `GROQ_API_KEY_1..N`), lọc trùng bằng `Set`. `buildProviderAttempts` xếp hàng toàn bộ key của provider ưu tiên (`AI_PRIMARY_PROVIDER`) nối tiếp bằng provider dự phòng. Điểm then chốt là **Ngân sách Timeout tổng 30 giây** (`deadline = startedAt + 30000` ở `L585`): mỗi lượt thử tính `remainingMs = deadline - Date.now()`, và nếu còn lượt dự phòng phía sau thì chỉ cấp **`perAttemptTimeoutMs = Math.ceil(remainingMs / Math.min(remainingAttempts, 3))`** (`L598-601`) qua `AbortController`. Nhờ đó 1 key bị treo không thể nuốt trọn 30s. Về ảnh: Gemini dùng model multimodal `gemini-3.5-flash-lite` nhận inline `base64`, còn Groq tự động chuyển từ model text (`openai/gpt-oss-120b`) sang model Vision (`qwen/qwen3.8-27b`) khi `hasImages === true` (`L529-534`).
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/utils/gemini.js:50-79` (`collectProviderApiKeys`) → `L375-520` (gọi Gemini / Groq) → `L525-555` (`buildProviderAttempts` + tự chọn model Vision) → `L561-658` (`analyzePetHealthWithAi` + deadline 30s).
  - Happy Path: Trả kết quả AI từ key đầu tiên thành công kèm `aiProvider`, `aiModel`, `responseTimeMs` → `201`.
  - Fail / Edge Cases liên quan: Không cấu hình key nào → `503` ngay lập tức (`L572-574`); tất cả key lỗi/429 hết quota → `503` (`L657`); tất cả lỗi và có ít nhất 1 lượt timeout (`sawTimeout === true`) → `504` (`L435,512,657`), đều thống nhất thông điệp `MSG34`.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Nếu ảnh nặng 5MB chuyển sang base64 rồi gửi qua mạng cho 3 key liên tiếp bị lỗi có làm chậm không?"
  - *Cách trả lời*: Chuỗi base64 được tạo sẵn một lần trước vòng lặp `for`, không mã hóa lại ở mỗi lượt thử; và mọi lượt đều chịu trần `deadline = startedAt + 30000ms` nên tổng thời gian tối đa từ lúc gọi AI đến lúc trả kết quả (hoặc `504`) không bao giờ vượt quá 30 giây.

### Q57. [3rd-Party / AI Hallucination Guard] [Mức độ: Nâng cao] Làm sao ngăn AI bịa tên/mã dịch vụ không có trong phòng khám, bịa định dạng JSON hỏng, hoặc vi phạm `BR-20` (khẳng định bệnh/kê đơn thuốc)?
- **Ý đồ giám khảo**: Kiểm tra khả năng kiểm soát đầu ra LLM (Output Guardrails) ở tầng Backend chứ không tin mù quáng AI.
- **Trả lời miệng (30–45 giây)**: Bọn em áp dụng 3 lớp phòng thủ ở `src/utils/gemini.js`: (1) **System Prompt chuẩn AAHA/AVMA Teletriage** (`L126-149`) cấm khẳng định chẩn đoán và cấm kê thuốc, ép trả JSON đúng schema; (2) **Bộ bóc tách JSON bền bỉ** (`extractJsonObject` `L223-254`) bóc được cả JSON trần, block ```` ```json ```` lẫn cắt `{...}` ngoài cùng nếu LLM lỡ kèm chữ thừa; (3) **Bộ chuẩn hóa & lọc Hallucination** (`parseAndNormalizeAiJson` `L260-368`): ép `urgency` chỉ nhận 3 giá trị `Routine | Monitor | Urgent`, **build 2 `Map` (`serviceByCode`, `serviceByName`) từ danh sách dịch vụ `Active` thật trong MongoDB** rồi đối chiếu mọi dịch vụ AI gợi ý (`L301-344`) — bất kỳ dịch vụ ảo nào AI bịa ra đều bị lọc bỏ hoàn toàn (tối đa 3 dịch vụ thật), cắt `bookingNoteDraft` ≤500 ký tự và luôn gắn cứng câu miễn trừ **`MSG33`** (`L366`).
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/utils/gemini.js:119-149` (System Prompt) → `L223-254` (`extractJsonObject`) → `L260-368` (`parseAndNormalizeAiJson`, lọc dịch vụ thật `L301-344`, gắn `MSG33` `L366`) → kiểm chứng bởi `tests/aiAssessment.test.js`.
  - Happy Path: Mọi response `201` trả về FE đều 100% đúng schema, `recommendedServices` chỉ chứa `serviceId` có thật trong DB, và luôn có `disclaimer` `MSG33`.
  - Fail / Edge Cases liên quan: Nếu AI trả về văn bản không thể parse thành JSON hợp lệ, lượt thử đó bị coi là lỗi và tự động failover sang key/provider tiếp theo.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Nếu AI bịa toàn bộ mã dịch vụ nên sau khi lọc `recommendedServices` còn mảng rỗng thì backend có ném lỗi hay vẫn lưu `AiAssessment`?"
  - *Cách trả lời*: Backend vẫn lưu hợp lệ và trả `201` với `recommendedServices: []` (đúng schema), đồng thời vẫn giữ đầy đủ `summary`, `urgency`, `careAdvice`, `bookingNoteDraft` và câu miễn trừ `MSG33` thay vì làm hỏng cả lượt đánh giá.

### Q58. [Background Job / JOB-01 Overdue] [Mức độ: Cực gắt] Hãy trace toàn bộ Cron Job `JOB-01` (đánh dấu lịch hẹn quá hạn `Overdue`): lịch chạy, múi giờ, xử lý khi server tắt lúc 23:59, chống Race Condition, quy tắc `BR-08` và Retry?
- **Ý đồ giám khảo**: Kiểm tra thiết kế Background Job production-ready (Timezone, Catch-up, Concurrency, Idempotency, Retry).
- **Trả lời miệng (30–45 giây)**: (1) `startBackgroundJobs()` (`src/jobs/index.js:13-27`) chạy sau khi `connectDB()` thành công trong `server.js:74-76`, đăng ký cron **`59 23 * * *`** theo múi giờ **`Asia/Ho_Chi_Minh`** (cho phép ghi đè qua `OVERDUE_JOB_CRON`, sai cú pháp `cron.validate` tự rơi về mặc định ở `flagOverdueAppointments.job.js:149-150`). (2) **Startup Catch-up** (`jobs/index.js:17-24`): ngay khi server khởi động lại, chạy bất đồng bộ quét bù toàn bộ đơn `Booked` từ **ngày hôm qua trở về trước** (`targetDate = getYesterdayStringVN()`, `includePastDates: true`) để không sót đơn nếu server tắt lúc 23:59, nhưng **tuyệt đối không đụng đơn hôm nay**. (3) **Chống Race Condition & `BR-08`** (`L64-82`): duyệt từng ứng viên dùng `AppointmentOrder.findOneAndUpdate({ _id, derivedStatus: 'Booked', checkedInAt: null }, { $set: { derivedStatus: 'Overdue' } })` — nếu Lễ tân vừa Check-in đúng lúc đó thì trả `null` và bỏ qua an toàn; chỉ khi đơn thực sự đổi sang `Overdue`, mới đánh dấu `Overdue` cho **duy nhất dịch vụ đầu tiên** (`sort: { sequenceNo: 1 }`) còn `Booked`, giữ nguyên các dịch vụ sau và không hủy đơn. (4) **Retry** (`L112-143`): gặp lỗi mạng DB tự thử lại tối đa 3 lần cách nhau 30 giây, thất bại hẳn thì log `CRITICAL` và catch gọn không làm sập Express.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `server.js:73-76` → `src/jobs/index.js:13-27` → `src/jobs/flagOverdueAppointments.job.js:23-174` (`flagOverdueAppointments`, `runFlagOverdueWithRetry`, `scheduleFlagOverdueJob`) + API chạy tay Admin `POST /api/appointment-orders/flag-overdue` (`appointmentOrder.routes.js:23`) → `tests/flagOverdueJob.test.js:82-349`.
  - Happy Path: Cron chạy sạch mỗi 23:59; API chạy tay trả `200` kèm `flaggedCount`, `skippedCount`, `targetDate`.
  - Fail / Edge Cases liên quan: API chạy tay truyền ngày ảo (`2026-02-31`) → `400` (`L43-47`); không phải `Admin` → `403`; đơn đã `CheckedIn`/`Done`/`Cancelled` hoặc ngày tương lai → bỏ qua (`UC-23 Alt 1a`).
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Nếu sau này deploy hệ thống lên 3 instance Node.js (Horizontal Scaling) sau Load Balancer thì cả 3 server đều chạy `JOB-01` lúc 23:59 có bị lỗi không?"
  - *Cách trả lời*: Nhờ câu lệnh nguyên tử `findOneAndUpdate({ _id, derivedStatus: 'Booked', checkedInAt: null })`, dù 3 instance chạy cùng lúc cũng không làm hỏng dữ liệu (instance đầu thắng, 2 instance kia bỏ qua `skippedCount`). Tuy nhiên để không lãng phí tài nguyên DB khi chạy đa instance, chuẩn production sẽ dùng **Distributed Lock** (Redis Redlock) hoặc tách riêng 1 worker service chạy cron.

---

## Phần 6: API Contract, Ma trận HTTP Status Code & Xử lý lỗi (Happy → Fail Cases)

### Bảng Tra Cứu Nhanh Toàn Bộ 12 Mã HTTP Status Trong `VCMOS_BE`

| HTTP | Ý nghĩa chuẩn trong hệ thống | Format Response | Ví dụ tọa độ code (`file:line`) |
| :-: | :--- | :--- | :--- |
| **`200`** | Thành công (GET/PUT/PATCH/Soft DELETE/Pay/PDF/VNPay IPN) | JSON (hoặc binary PDF / `{ RspCode, Message }`) | 22 controllers + `server.js:37` |
| **`201`** | Tạo mới tài nguyên thành công (`POST`) | `{ message, data\|user\|pet\|assessment... }` | 17 controllers có `POST` tạo mới |
| **`400`** | Lỗi input / validate / ObjectId / JSON hỏng / Multer / tham số nghiệp vụ sai | `{ message, errors?: { field: '...' } }` | Validators, `upload.middleware.js:10,70,141`, `pet.routes.js:32` |
| **`401`** | Thiếu `Bearer`, JWT sai/hết hạn, quên `protect`, sai login (lần 1–4) | `{ message }` | `auth.middleware.js:10,46,56`, `auth.service.js:156` |
| **`403`** | Sai role, `Locked`/`Inactive`, `BR-03` sai 5 lần khóa 30p, IDOR, `BR-22` cấm `DELETE` | `{ message }` | `auth.middleware.js:31,60`, `auth.service.js:132-152`, `admin/service/invoice.controller.js` |
| **`404`** | Không tìm thấy bản ghi theo ID/mã, hoặc đã soft delete (`Deleted`/`Removed`) | `{ message }` | 18 services (`pet`, `invoice`, `appointmentOrder`, ...) |
| **`409`** | Trùng Unique Index (`11000`), sai State Machine, chuồng `Occupied`, nhân sự bận, `BR-07`/`BR-12`/`BR-14`/`MSG26` | `{ message }` | 17 services |
| **`415`** | Không gửi `multipart/form-data` lên `POST`/`PUT /api/community-posts` | `{ message }` | `upload.middleware.js:117` |
| **`500`** | Lỗi nội bộ chưa dự kiến, thiếu env PayOS/VNPay, cạn 5 lần retry sinh mã | `{ message }` | `server.js:62`, `payos.js:18-24`, `vnpay.js:60-64`, `auth.service.js:118` |
| **`502`** | Dịch vụ bên thứ 3 lỗi: Cloudinary fail hoặc PayOS trả lỗi/`code !== '00'` | `{ message }` | `payos.js:99`, `aiAssessment.service.js:58`, `communityPost.service.js:31`, `subtaskRequest.service.js:297` |
| **`503`** | AI chưa cấu hình key hoặc tất cả key Gemini & Groq lỗi/hết quota (`MSG34`) | `{ message }` | `src/utils/gemini.js:573,657` |
| **`504`** | Quá thời gian chờ (Timeout): AI vượt `30s` (`MSG34`) hoặc PayOS vượt `15s` | `{ message }` | `src/utils/gemini.js:435,512,657`, `src/utils/payos.js:107` |

---

### Q59. [API Contract / HTTP Status Matrix] [Mức độ: Cơ bản] Hãy trình bày quy chuẩn Format JSON lỗi và các ngoại lệ response trong toàn bộ `VCMOS_BE`?
- **Ý đồ giám khảo**: Kiểm tra tính nhất quán của API Contract giữa BE và FE.
- **Trả lời miệng (30–45 giây)**: Hệ thống chuẩn hóa 2 mẫu lỗi JSON: (1) Mọi lỗi chung (`401`, `403`, `404`, `409`, `415`, `500`, `502`, `503`, `504`) đều trả `{ "message": "<tiếng Việt rõ ràng>" }`; (2) Mọi lỗi validate đầu vào `400` trả `{ "message": "Dữ liệu không hợp lệ", "errors": { "<tênField>": "<lý do chi tiết>" } }` để FE gắn đỏ đúng ô input. Toàn hệ thống chỉ có đúng **2 ngoại lệ** không dùng JSON chuẩn: `GET /api/invoices/vnpay-ipn` luôn trả `200 { RspCode, Message }` theo đặc tả VNPay, và `GET /api/invoices/:id/pdf` trả binary `application/pdf`.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `server.js:61-70` (Global Error Middleware trả `{ message, errors? }`) + `src/controllers/invoice.controller.js:117-125` (VNPay IPN) + `src/controllers/invoice.controller.js:260-263` (PDF binary).
  - Happy Path: FE luôn đọc được `err.response.data.message` và `err.response.data.errors` đồng nhất trên toàn bộ 20 nhóm API.
  - Fail / Edge Cases liên quan: Lưu ý riêng `cage.validator.js:8-15` có `this.errors = errors` kiểu mảng chuỗi (khác chút so với dạng object `{ field: msg }` ở các validator khác).
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Tại sao lỗi `400` lại không trả stack trace của Node.js về client?"
  - *Cách trả lời*: Trả stack trace ra response sẽ làm lộ cấu trúc thư mục, tên file và phiên bản thư viện nội bộ cho kẻ tấn công. Hệ thống chỉ log `console.error` ở phía server và trả duy nhất `message` tiếng Việt sạch cho client.

### Q60. [Error Handling / CastError & ValidationError] [Mức độ: Nâng cao] Nếu client gọi `GET /api/pets/abc123` (sai định dạng ObjectId) hoặc gửi body JSON bị thiếu dấu `}` thì từng lớp bắt lỗi ở đâu và trả status gì?
- **Ý đồ giám khảo**: Kiểm tra cách chặn input hỏng sớm trước khi để Mongoose ném `CastError` làm nổ `500`.
- **Trả lời miệng (30–45 giây)**: (1) Với `:id` sai định dạng ObjectId (`/api/pets/abc123`), toàn bộ tầng **Validator** (và helper `assertValidId` ở `cageAssignment.service.js:12-16`) đều kiểm tra **`mongoose.isValidObjectId(id)`** ngay trước khi chạm DB để trả **`400`** sạch sẽ. Ngoài ra các controller như `product.controller.js`, `cage.controller.js`, `appointmentOrder.controller.js` còn bắt phòng thủ thêm `err.name === 'CastError'` và `err.name === 'ValidationError'` chuyển về **`400`**. (2) Với body JSON hỏng cú pháp (thiếu dấu `}`), middleware `express.json()` ở `server.js:34` tự động ném `SyntaxError` có `err.status = 400`, Express 5 chuyển thẳng xuống Global Error Middleware (`server.js:61-70`) và trả JSON **`400`**.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `server.js:34,61-70` (`express.json()` → Global Error Handler `err.status || 500`), `src/validators/cage.validator.js:93-98` (`mongoose.isValidObjectId`), `src/services/cageAssignment.service.js:12-16` (`assertValidId`).
  - Happy Path: ObjectId 24 ký tự hex hợp lệ và JSON chuẩn đi tiếp vào xử lý nghiệp vụ.
  - Fail / Edge Cases liên quan: `GET /api/cages/not-an-id` → `400` (`'Mã chuồng không hợp lệ'`); body `{"name": ` hỏng cú pháp → `400` JSON.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Nếu một route mới quên kiểm tra `isValidObjectId` rồi truyền thẳng vào `Model.findById` và controller gọi `next(err)` thì Global Error Middleware ở `server.js:61-70` sẽ trả mã mấy?"
  - *Cách trả lời*: Em nói thẳng: `CastError` mặc định của Mongoose không có thuộc tính `statusCode` hay `status`, nên nếu lọt thẳng tới `server.js:62` sẽ rơi về `500`. Cách cải thiện rất gọn là thêm 2 dòng kiểm tra `if (err.name === 'CastError' || err.name === 'ValidationError') statusCode = 400` ngay trong Global Error Middleware `server.js:61-70` để làm lưới an toàn toàn cục.

### Q61. [Error Handling / Mongo 11000] [Mức độ: Cực gắt] Khi MongoDB ném lỗi trùng khóa `E11000`, làm sao hệ thống phân biệt giữa trùng mã ngẫu nhiên tự sinh (phải retry) với trùng dữ liệu người dùng (phải trả `409 Conflict`)?
- **Ý đồ giám khảo**: Kiểm tra xử lý thực tế với lỗi duplicate key của MongoDB trong môi trường đồng thời.
- **Trả lời miệng (30–45 giây)**: Các mã nghiệp vụ (`USR-XXXXXX`, `PET-XXXXXX`, `APT-XXXXXX`, `INV-XXXXXX`) được sinh ngẫu nhiên ở tầng Service nên về mặt xác suất có thể trùng rất hiếm. Khi bắt `err.code === 11000`, các service dùng hàm kiểm tra `err.keyPattern?.code` (như `isDuplicateCodeError`) để phân nhánh: (1) Nếu đúng là trùng trường **`code`** → tự động **retry tối đa 5 lần** để sinh mã khác, chỉ khi cạn cả 5 lần mới ném **`500`**; (2) Nếu là trùng trường nghiệp vụ (`phone`, `email`, `(category, name)`, `(name, unit)`, `label`, `(staffId, date, shift)`, `(postId, userId)`, `orderId` trên `Invoice`, `cageId` trên `CageAssignment`) → lập tức ném **`409 Conflict`** kèm thông báo tiếng Việt rõ ràng.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/services/auth.service.js:118` (retry `USR`), `src/services/pet.service.js:102` (retry `PET`), `src/services/walkIn.service.js:75,99,142,441-448` (retry `USR`/`PET`/`APT`), `src/services/invoice.service.js:411` (retry `INV`), `src/services/staffSchedule.service.js:126-139` (check `err.keyPattern` trước khi trả `409`).
  - Happy Path: Trùng mã tự sinh được retry trong suốt, client vẫn nhận `201` bình thường.
  - Fail / Edge Cases liên quan: Trùng dữ liệu thật → `409`; trùng mã quá 5 lần liên tiếp → `500`; trùng index thừa không khớp `keyPattern` (ở `staffSchedule.service.js:129-138`) → ném lại `500` để không che giấu lỗi DB.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Tại sao không dùng auto-increment counter (`counters` collection) để sinh mã `USR-000001` cho khỏi cần retry?"
  - *Cách trả lời*: Dùng random 6 chữ số + retry 5 lần kết hợp Unique Index tránh được 1 nút thắt cổ chai (hot document lock trên bảng `counters` khi nhiều request tạo đồng thời) và tránh lộ tổng số lượng khách hàng/hóa đơn của phòng khám ra ngoài URL/biên lai.

### Q62. [Error Handling / Multer & Upload] [Mức độ: Nâng cao] Hãy liệt kê chính xác mã HTTP status trả về cho mọi lỗi upload ảnh: vượt 5MB, quá số ảnh, sai tên field, sai MIME, sai Magic Bytes và gửi JSON vào bài cộng đồng?
- **Ý đồ giám khảo**: Kiểm tra chi tiết kỹ năng bọc `multer` — rất nhiều dự án để lỗi Multer rơi thẳng thành `500`.
- **Trả lời miệng (30–45 giây)**: Mọi middleware Multer trong `upload.middleware.js:60-77` và `pet.routes.js:24-35` đều được bọc bằng callback `(err) => ...` để tuyệt đối không rơi thành `500`: (1) `LIMIT_FILE_SIZE` (> 5MB) → **`400`** (`upload.middleware.js:66`, `pet.routes.js:29`); (2) `LIMIT_UNEXPECTED_FILE` (sai tên field) → **`400`** (`L67`); (3) `LIMIT_FILE_COUNT` (hoặc tổng `photos` + `photo` > 3 ở AI `L138-144`) → **`400`**; (4) Sai MIME type trong `fileFilter` (`L8-11`) → **`400`**; (5) Giả mạo đuôi/MIME nhưng sai Magic Bytes ở Pet & AI → **`400`** (`pet.validator.js:58-60`, `aiAssessment.validator.js:137-138`); (6) Gửi JSON (`application/json`) thay vì `multipart/form-data` lên `POST/PUT /api/community-posts` → **`415 Unsupported Media Type`** (`upload.middleware.js:116-118`).
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/middlewares/upload.middleware.js:7-14,60-77,115-120,132-147` + `src/routes/pet.routes.js:13-37` + `src/validators/pet.validator.js:39-60` + `src/validators/aiAssessment.validator.js:18-26,137-138`.
  - Happy Path: Ảnh hợp lệ nằm gọn trong `req.file` / `req.files` dạng `Buffer` và đi tiếp vào controller.
  - Fail / Edge Cases liên quan: Mọi lỗi Multer trả `400` JSON tiếng Việt; sai Content-Type bài cộng đồng trả `415`; lỗi mạng lúc đẩy lên Cloudinary trả `502`.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Tại sao upload middleware luôn phải đặt SAU `protect` và `authorize` trong file route?"
  - *Cách trả lời*: Vì `multer.memoryStorage()` đọc toàn bộ file (tối đa 5MB × 11 ảnh ở Community = 55MB) vào RAM server. Đặt sau `protect` và `authorize` (`appointmentService.routes.js:47-53`, `subtaskRequest.routes.js:49`, `aiAssessment.routes.js`) đảm bảo người chưa đăng nhập hoặc sai vai trò bị đá `401`/`403` ngay trước khi server tốn 1 byte RAM đọc file.

### Q63. [Error Handling / Business Rule 400 vs 403 vs 409] [Mức độ: Cực gắt] Khi vi phạm quy tắc nghiệp vụ, ranh giới nào để codebase quyết định trả `400`, `403` hay `409`? Hãy dẫn chứng bằng các mã BR/MSG thật trong code?
- **Ý đồ giám khảo**: Kiểm tra tư duy thiết kế REST API chuẩn mực (không ném bừa `400` cho mọi thứ).
- **Trả lời miệng (30–45 giây)**: Bọn em phân định rành mạch 3 mã theo ngữ nghĩa HTTP:
  - **`400 Bad Request`**: Dữ liệu/tham số gửi lên không thỏa điều kiện tĩnh hoặc thiếu dữ liệu bắt buộc — vd thiếu O/A/P khi hoàn tất khám (`BR-13`), kê thuốc `Inactive` (`BR-16`), chọn chuồng quá cân nặng/không cách ly (`BR-11`), ngày quá khứ, reply cấp 2 (`BR-19`), `cashReceived < totalAmount`, gỡ dòng `Service` trên hóa đơn (`invoice.service.js:437-439`).
  - **`403 Forbidden`**: Vi phạm quyền hoặc chính sách bị cấm — vd sai role, tài khoản `Locked`/`Inactive`, sai mật khẩu 5 lần tạm khóa 30 phút (`BR-03`), truy cập tài nguyên người khác (IDOR / `BR-23`), gọi `DELETE` vật lý bị cấm (`BR-22`).
  - **`409 Conflict`**: Xung đột với **trạng thái hiện tại của tài nguyên trong DB** hoặc trùng dữ liệu — vd hủy lịch `< 2h` (`BR-07`), khám dịch vụ #2 khi dịch vụ #1 chưa xong (`BR-12`), hoàn tất khám khi còn Subtask chưa xong (`MSG26`), sửa bệnh án đã `Finalized` (`BR-14`), chuồng đang `Occupied`, nhân sự đã báo bận, lập hóa đơn khi nội trú chưa xuất viện, hoặc lỗi `E11000`.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/services/auth.service.js:132-152` (`BR-03 → 403`), `src/services/medicalRecord.service.js:147-467` (`BR-13/BR-16/BR-11 → 400`, `BR-12/BR-14/MSG26 → 409`), `src/services/appointmentService.service.js:325-437` (`BR-07 → 409`), `src/controllers/invoice.controller.js:294-297` (`BR-22 → 403`).
  - Happy Path: Thỏa toàn bộ điều kiện → `200`/`201`.
  - Fail / Edge Cases liên quan: Lưu ý mã `422` hay `423` không dùng trong `VCMOS_BE` — nhóm thống nhất gom khóa tạm thời `BR-03` vào `403` và xung đột state machine vào `409` để FE xử lý gọn.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Tại sao `BR-03` khóa tài khoản 30 phút không dùng HTTP `423 Locked` hay `429 Too Many Requests`?"
  - *Cách trả lời*: Nhóm thống nhất bộ mã REST chuẩn gọn (`400/401/403/404/409`) xuyên suốt toàn bộ 20 nhóm API backend, trong đó mọi trạng thái tài khoản bị khóa (cả tạm khóa 30 phút `BR-03` lẫn khóa vĩnh viễn `UC-62`) đều trả `403` kèm `message` tiếng Việt phân biệt rõ nguyên nhân.

### Q64. [Error Handling / Upstream 502 vs 503 vs 504] [Mức độ: Nâng cao] Khi tích hợp bên thứ 3 gặp sự cố (Cloudinary, PayOS, Gemini/Groq), hệ thống phân biệt `502`, `503`, `504` ở chính xác dòng code nào?
- **Ý đồ giám khảo**: Kiểm tra hiểu biết chuẩn xác về nhóm mã lỗi 5xx cho microservices/external APIs.
- **Trả lời miệng (30–45 giây)**: Nhóm phân biệt rất rõ theo chuẩn RFC:
  - **`502 Bad Gateway`**: Server nhận được phản hồi lỗi hoặc mất kết nối khi làm proxy gọi upstream — dùng cho **Cloudinary upload fail** (`aiAssessment.service.js:58`, `communityPost.service.js:31`, `subtaskRequest.service.js:297`, `appointmentService.service.js:582-585`) và **PayOS API trả lỗi HTTP / `code !== '00'`** (`src/utils/payos.js:96-101`).
  - **`503 Service Unavailable`**: Dịch vụ tạm không sẵn sàng / hết quota — dùng cho **Gemini/Groq** khi chưa cấu hình key hoặc mọi key đều lỗi/hết quota (`src/utils/gemini.js:572-574, 657`).
  - **`504 Gateway Timeout`**: Quá thời gian chờ `AbortController` — dùng cho **PayOS vượt 15 giây** (`src/utils/payos.js:105-109`) và **Gemini/Groq vượt ngân sách 30 giây** (`src/utils/gemini.js:435, 512, 657`).
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/utils/payos.js:96-109`, `src/utils/gemini.js:435,512,572-574,657`, `src/services/aiAssessment.service.js:58`, `src/services/communityPost.service.js:31`, `src/services/subtaskRequest.service.js:297`, `src/services/appointmentService.service.js:582-585`.
  - Happy Path: Upstream phản hồi hợp lệ trong giới hạn timeout → `200`/`201`.
  - Fail / Edge Cases liên quan: Riêng cấu hình thiếu `PAYOS_*` hoặc `VNP_*` là lỗi cấu hình nội bộ server nên trả **`500`** (`payos.js:18-24`, `vnpay.js:60-64`) chứ không đổ lỗi cho upstream (`502`).
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Có thể tái hiện lỗi `503` và `504` của AI trong bài test tự động mà không phải đợi 30 giây thật không?"
  - *Cách trả lời*: Có ạ, trong `tests/aiAssessment.test.js`, nhóm mock `global.fetch` và truyền timeout ngắn (hoặc mock `AbortError`) để test chạy pass cả nhánh failover Gemini→Groq, nhánh `503` và nhánh `504` chỉ trong vài trăm mili-giây.

---

## Phần 7: Câu hỏi "Bắt lỗi trực tiếp trên Code thật" (Điểm yếu, Race Condition & Nợ kỹ thuật)

> **Chiến thuật phòng thủ khi bị soi trúng điểm yếu:** Không bao giờ cãi chày cối hay giấu giếm. **Thừa nhận thẳng thắn + chỉ đúng dòng code + đánh giá mức độ ảnh hưởng hiện tại + đưa giải pháp chuẩn production trong 2 câu.** Giám khảo chấm điểm cực cao cho sự trung thực và năng lực tự audit code của mình!

### Q65. [Code Audit / Security Infra] [Mức độ: Cực gắt] Mở `server.js:29-35`: Tại sao hệ thống không có `helmet`, không có `express-rate-limit` cho login/OTP/AI và chưa có 404 JSON catch-all?
- **Ý đồ giám khảo**: Kiểm tra các lỗ hổng hạ tầng HTTP dễ bị soi ngay khi mở `server.js`.
- **Trả lời miệng (30–45 giây)**: Em thừa nhận đây là 3 điểm hạ tầng còn thiếu trong `server.js:29-70`: (1) Chưa gắn `helmet()` để thêm các security headers (`X-Content-Type-Options`, `Strict-Transport-Security`, ẩn `X-Powered-By`); (2) Chưa gắn `express-rate-limit` theo IP nên dù login đã có khóa 5 lần ở DB (`BR-03`), các endpoint gửi OTP (`/api/users/*/send-otp`, `/forgot-password`) hay gọi AI (`/api/ai-assessments`) vẫn có nguy cơ bị spam tốn quota; (3) Chưa có 404 catch-all trước dòng 61 nên URL sai đang trả trang HTML mặc định của Express 5 thay vì JSON `{ message }`.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `server.js:29-35` (chỉ có `cors` và `express.json()`), `server.js:58-70` (thiếu 404 middleware trước Global Error Middleware).
  - Happy Path: Các route hợp lệ hoạt động bình thường.
  - Fail / Edge Cases liên quan: `GET /api/non-existent` trả `404` dạng HTML (`Cannot GET /api/non-existent`); spam 100 request `/forgot-password` liên tục không bị chặn `429`.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Vậy để khắc phục triệt để trong production anh sửa cụ thể thế nào?"
  - *Cách trả lời*: Chỉ cần bổ sung đúng 3 khối trong `server.js` không ảnh hưởng nghiệp vụ: (1) `app.use(helmet())`; (2) gắn `express-rate-limit` (dùng Redis store khi multi-instance) với ngưỡng chặt riêng cho `/api/auth/login`, `/api/users/*/send-otp`, `/api/users/forgot-password` (vd 5 req/phút/IP) và `/api/ai-assessments` (vd 10 req/giờ/user); (3) thêm `app.use((req, res) => res.status(404).json({ message: 'Không tìm thấy đường dẫn API' }))` ngay trước `server.js:61`.

### Q66. [Code Audit / Error Consistency] [Mức độ: Nâng cao] Mở `server.js:61-70`: Tại sao có Global Error Middleware nhưng chỉ 4/22 controller gọi `next(err)`, còn 18/22 tự `try/catch`, và các service lúc gán `err.statusCode` lúc gán `err.status`?
- **Ý đồ giám khảo**: Kiểm tra tính đồng nhất code khi làm việc nhóm nhiều người và nợ kỹ thuật (technical debt).
- **Trả lời miệng (30–45 giây)**: Em thừa nhận đây là nợ kỹ thuật do nhóm có nhiều thành viên code song song theo từng sprint: 4 controller viết sau (`cageAssignment`, `communityPost`, `communityInteraction`, `communityModeration`) đẩy lỗi gọn qua `next(err)` về Global Error Middleware (`server.js:61-70`), còn 18 controller khác viết hàm `handleError` cục bộ trong từng file. Tương tự, `appointmentOrder.service.js:29` và `inpatientRecord.service.js:13` gán `err.status` thay vì `err.statusCode`. Hiện tại nó **không gây bug** vì `server.js:62` đọc cả `err.statusCode || err.status || 500`, nhưng lặp lại code xử lý lỗi ở 18 file.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `server.js:61-70`, `src/services/appointmentOrder.service.js:29` (`err.status`), `src/services/inpatientRecord.service.js:13` (`err.status`), đối chiếu `src/controllers/cageAssignment.controller.js` (`next(err)`).
  - Happy Path: Cả 2 nhánh đều trả JSON lỗi thống nhất về client.
  - Fail / Edge Cases liên quan: Nếu 1 lỗi Mongoose `CastError`/`ValidationError` lọt qua 4 controller dùng `next(err)` tới `server.js:61` sẽ trả `500` thay vì `400` vì Global Error Middleware chưa có nhánh nhận diện 2 lỗi này.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Kế hoạch refactor tầng xử lý lỗi cho sạch là gì?"
  - *Cách trả lời*: Tạo 1 class dùng chung `AppError(statusCode, message, errors)` trong `src/utils/AppError.js`, nâng cấp Global Error Middleware ở `server.js:61-70` bắt tập trung `AppError`, `CastError`, `ValidationError`, `11000`, `JsonWebTokenError` rồi xóa toàn bộ `try/catch` lặp lại ở 18 controller (vì Express 5 đã tự bắt rejected Promise từ `async` route handler và đẩy về `next(err)`).

### Q67. [Code Audit / Orphan Cloudinary Files] [Mức độ: Cực gắt] Soi `communityPost.service.js:25-37` và `aiAssessment.service.js:220-225`: Nếu upload ảnh lên Cloudinary xong mà bước `.create()` vào MongoDB bị lỗi thì có bị sinh ảnh rác trên Cloudinary không?
- **Ý đồ giám khảo**: Kiểm tra độ nhất quán trong việc dọn tài nguyên phân tán (Distributed Resource Cleanup).
- **Trả lời miệng (30–45 giây)**: Em xác nhận đúng là có khoảng hở ở đúng 2 file thầy vừa chỉ: trong khi 3 luồng `pet.service.js:62-70,104`, `appointmentService.service.js:562-575` và `subtaskRequest.service.js:55-62` đều có hàm `destroy` xóa ảnh Cloudinary khi ghi DB lỗi, thì `communityPost.service.js:25-37` và bước `AiAssessment.create` ngay sau `aiAssessment.service.js:220-225` chưa bọc `try/catch` gọi `cloudinary.uploader.destroy` cho các `publicId` vừa upload. Riêng `aiAssessment` giảm được >90% rủi ro nhờ gọi AI xong mới upload ảnh, nhưng nếu MongoDB rớt kết nối đúng mili-giây sau đó thì vẫn có thể sót ảnh mồ côi.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/services/communityPost.service.js:25-37` (upload xong mới tạo `CommunityPost`), `src/services/aiAssessment.service.js:212-225` (gọi AI `L212` → upload Cloudinary `L220-223` → `AiAssessment.create` `L225`).
  - Happy Path: Ghi DB thành công, ảnh gắn URL hợp lệ trong document.
  - Fail / Edge Cases liên quan: Nếu `CommunityPost.create` hay `AiAssessment.create` ném lỗi DB (`500`), ảnh vừa đẩy lên Cloudinary vẫn còn tồn tại mà không có document nào trỏ tới.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Làm sao dọn sạch ảnh mồ côi kể cả khi server bị crash đột ngột sau khi upload?"
  - *Cách trả lời*: Kết hợp 2 lớp: (1) Immediate cleanup — tách 1 helper `deleteCloudinaryImages(publicIds)` dùng chung và gọi trong khối `catch` của cả `communityPost` lẫn `aiAssessment`; (2) Batch reconciliation — chạy 1 Cron Job định kỳ hằng tuần quét các asset trên Cloudinary tạo quá 24h nhưng URL không tồn tại trong DB để xóa sạch.

### Q68. [Code Audit / Upload Magic Bytes Gap] [Mức độ: Cực gắt] Soi `upload.middleware.js:7-14`: Tại sao `pet` và `aiAssessment` kiểm tra Magic Bytes nhưng `Grooming`, `Subtask` và `Community` mới chỉ kiểm tra `file.mimetype`?
- **Ý đồ giám khảo**: Bắt lỗi bảo mật không đồng đều giữa các luồng upload cùng 1 hệ thống.
- **Trả lời miệng (30–45 giây)**: Em thừa nhận 3 luồng `Grooming`, `Subtask`, `Community` ở `upload.middleware.js:7-14` hiện mới kiểm tra `file.mimetype` (có thể bị giả mạo bằng cách sửa header `Content-Type`), còn chỉ `pet.validator.js:39-60` và `aiAssessment.validator.js:18-26,137` mới kiểm tra Magic Bytes trên binary buffer. Dù lớp sau là Cloudinary (`resource_type: 'image'`) sẽ tự từ chối file giả mạo và trả `502` (không lưu được file độc hại), nhưng để kẻ tấn công lọt qua tới tận bước gọi Cloudinary là lãng phí băng thông và trả sai mã (`502` thay vì `400`).
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/middlewares/upload.middleware.js:7-14` (chỉ check `allowedMimeTypes.includes(file.mimetype)`), đối chiếu `src/validators/pet.validator.js:39-60` và `src/validators/aiAssessment.validator.js:18-26,137`.
  - Happy Path: Ảnh thật vượt qua cả MIME lẫn Cloudinary.
  - Fail / Edge Cases liên quan: Gửi file text đổi đuôi `.jpg` + `image/jpeg` vào Community/Grooming/Subtask sẽ vượt qua `upload.middleware.js:8` và chỉ bị Cloudinary từ chối sau đó (`502`).
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Hướng sửa gọn nhất mà không bị lặp code giữa 5 luồng?"
  - *Cách trả lời*: Tách hàm `validateImageMagicBytes(buffer, allowedTypes)` ra 1 file `src/utils/fileSignature.js` (hoặc dùng thư viện `file-type`) rồi chạy ngay trong `createUploadErrorHandler` (`upload.middleware.js:60-77`) sau khi Multer nhận xong `req.file`/`req.files`, trả thẳng `400` cho cả 5 luồng.

### Q69. [Code Audit / Cage Validator Bug] [Mức độ: Cực gắt] Soi `src/validators/cage.validator.js:109,127` đối chiếu `src/models/Cage.js:7-8`: Admin có tạo được chuồng khu `InpatientWard` kèm `maxWeightKg` qua API `POST /api/cages` không?
- **Ý đồ giám khảo**: Bắt một lỗi logic thật sự nằm ẩn giữa Model và Validator trong codebase!
- **Trả lời miệng (30–45 giây)**: Em cảm ơn thầy đã soi trúng bug thật này: ở schema `src/models/Cage.js:7-8` đã hỗ trợ 2 trường `zone` (`Holding | InpatientWard`, mặc định `Holding`) và `maxWeightKg`, và luồng nội trú (`inpatientRecord.service.js`) có kiểm tra 2 trường này. Tuy nhiên trong `src/validators/cage.validator.js:109` và `L127`, hàm `pick(body, ['label', 'size', 'isIsolated', 'status'])` chỉ whitelist đúng 4 trường cũ nên nếu Admin gửi `zone: 'InpatientWard'` hay `maxWeightKg: 25` qua `POST /api/cages` hoặc `PATCH /api/cages/:id` thì **bị âm thầm bỏ qua**, mọi chuồng tạo qua API đều rơi về mặc định `zone: 'Holding'` (chuồng `InpatientWard` hiện tại mới chỉ được tạo qua `seed.js`).
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/models/Cage.js:7-8` (`zone`, `maxWeightKg`) vs `src/validators/cage.validator.js:42-87,109,127` (chỉ `pick` 4 trường `label`, `size`, `isIsolated`, `status`).
  - Happy Path: Tạo chuồng `Holding` vẫn trả `201` bình thường.
  - Fail / Edge Cases liên quan: Gửi `{ label: 'IP-99', size: 'L', zone: 'InpatientWard', maxWeightKg: 30 }` nhận `201` nhưng document lưu xuống DB lại có `zone: 'Holding'` và không có `maxWeightKg` tùy chỉnh.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Sửa lỗi này mất bao lâu và cần lưu ý gì khi cập nhật `zone`?"
  - *Cách trả lời*: Chỉ cần thêm `zone` (`Holding | InpatientWard`) và `maxWeightKg` (số dương > 0) vào `checkCageFields` + mảng `pick` ở `cage.validator.js:109,127` + thêm bộ lọc `zone` vào `validateGetCages`; đồng thời chặn đổi `zone` khi chuồng đang `Occupied` (`409`) và bổ sung test case vào `tests/services/cage.service.test.js`.

### Q70. [Code Audit / Cage Assignment Race & BR-11 Gap] [Mức độ: Cực gắt] Soi `CageAssignment.js:15-18` và `cageAssignment.service.js:66-78`: Nếu gọi trực tiếp `POST /api/cage-assignments` 2 lần cho cùng 1 `orderId` (chọn 2 chuồng khác nhau), hoặc gọi mà không truyền `petWeight` thì sao?
- **Ý đồ giám khảo**: Soi lỗ hổng toàn vẹn dữ liệu khi API bị gọi trực tiếp (qua Postman/script) thay vì đi qua Check-in/Walk-in.
- **Trả lời miệng (30–45 giây)**: Em thừa nhận có 2 điểm hở khi gọi trực tiếp `POST /api/cage-assignments`: (1) `CageAssignment.js:15-18` mới có Partial Unique Index `{ cageId: 1 }` (chặn 1 chuồng chứa 2 thú cưng), **chưa có Partial Unique Index `{ orderId: 1 }` với `releasedAt: null`**, nên nếu gọi trực tiếp 2 lần với cùng `orderId` và 2 `cageId` khác nhau thì 1 đơn khám có thể chiếm cùng lúc 2 chuồng; (2) Ở `cageAssignment.service.js:66-78`, kiểm tra `BR-11` chỉ chạy khi caller chủ động truyền `petWeight` hoặc `requiresIsolation` (các luồng nội bộ Check-in/Walk-in có truyền đủ, nhưng gọi thẳng API mà bỏ trống 2 field này thì sẽ bỏ qua check tải trọng).
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/models/CageAssignment.js:15-19` (chỉ unique `cageId`, `orderId` là index thường `L19`), `src/services/cageAssignment.service.js:66-78` (`if (petWeight !== undefined || requiresIsolation !== undefined)`).
  - Happy Path: Qua luồng Check-in (`appointmentOrder.service.js`) và Walk-in (`walkIn.service.js`), service tự tra cứu cân nặng Pet và truyền đầy đủ nên `BR-11` luôn chạy.
  - Fail / Edge Cases liên quan: Gọi thẳng `POST /api/cage-assignments` chỉ gửi `{ cageId, orderId }` sẽ bỏ qua `BR-11`, và gọi lần 2 với `cageId` khác cho cùng `orderId` vẫn trả `201` thay vì `409`.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Anh khắc phục 2 lỗ hổng này như thế nào?"
  - *Cách trả lời*: (1) Thêm Partial Unique Index `{ orderId: 1 }` với `{ releasedAt: { $type: 'null' } }` trên `CageAssignment.js` (và bắt `11000 → 409` khi đơn đã có chuồng đang mở); (2) Trong `createCageAssignment`, tự truy vấn `AppointmentOrder` (kèm `petId`) từ `orderId` để lấy `pet.weight` và `order.intake.requiresIsolation` trực tiếp từ DB thay vì tin tham số caller truyền lên.

### Q71. [Code Audit / Counter Drift & Transaction Gap] [Mức độ: Cực gắt] Soi `communityInteraction.service.js` và `CommunityPost.js:23-24`: Nếu 10 user bấm Like cùng lúc hoặc server lỗi ngay sau khi tạo `PostLike` nhưng trước khi `$inc` `likeCount` thì số đếm có bị lệch không?
- **Ý đồ giám khảo**: Kiểm tra hiểu biết về Denormalization trade-off và Eventual Consistency.
- **Trả lời miệng (30–45 giây)**: Em phân tích rõ 2 tình huống: (1) Nếu 10 user bấm Like cùng 1 giây mà không có lỗi mạng, `PostLike` có Unique Index `{ postId, userId }` chống trùng và MongoDB `$inc: { likeCount: 1 }` là phép cộng nguyên tử ở mức document nên **không bị mất cộng (lost update)**; (2) Tuy nhiên, thao tác tạo/xóa `PostLike` (hoặc `PostComment`) và thao tác cập nhật `CommunityPost.likeCount`/`commentCount` hiện **chưa bọc trong `runWithTransaction`**, nên nếu DB đứt kết nối đúng giữa 2 lệnh ghi thì `likeCount` có thể lệch ±1 so với số bản ghi thật trong `post_likes`.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/models/CommunityPost.js:23-24` (`likeCount`, `commentCount`, `min: 0`), `src/models/PostLike.js:11` (Unique `{ postId, userId }`), `src/services/communityInteraction.service.js:1-288`.
  - Happy Path: Like/Unlike, Comment/Delete đồng bộ số đếm ngay và trả `200`/`201`.
  - Fail / Edge Cases liên quan: Rủi ro lệch số đếm (counter drift) nếu lệnh ghi thứ 2 thất bại do lỗi mạng tạm thời.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Cách xử lý triệt để cho mạng xã hội có traffic cao hơn là gì?"
  - *Cách trả lời*: Ở quy mô phòng khám vừa, chỉ cần bọc cặp `PostLike` + `CommunityPost.$inc` vào `runWithTransaction` sẵn có (`src/utils/transaction.js`) là tuyệt đối chính xác. Ở quy mô mạng xã hội cực lớn (hàng nghìn like/giây tránh tranh chấp lock trên 1 document bài viết), sẽ ghi nhận vào Redis counter rồi flush định kỳ và có job đối soát `countDocuments` hằng đêm.

### Q72. [Code Audit / Stale DB Index Incident] [Mức độ: Cực gắt] `src/config/db.js` không tự gọi `syncIndexes()`. Điều gì xảy ra trên MongoDB Atlas khi nhóm đổi tên field trong Unique Index của Schema nhưng không drop index cũ?
- **Ý đồ giám khảo**: Kiểm tra kinh nghiệm thực chiến vận hành MongoDB (Schema Migration / Index Drift).
- **Trả lời miệng (30–45 giây)**: Mongoose mặc định chỉ chạy `createIndex` (thêm index mới), **không tự xóa index cũ đã bỏ khỏi schema**. Nhóm đã từng gặp đúng sự cố thật này trên Atlas ở collection `staff_schedules`: trước đây index là `staffId_1_workDate_1` (unique), sau đó nhóm đổi trường `workDate` sang `date` và thêm `shift` (`staffId_1_date_1_shift_1`). Vì index cũ `staffId_1_workDate_1` vẫn tồn tại trên Atlas, mọi document mới đều có `workDate: null`, dẫn đến từ ca báo bận thứ 2 trở đi của cùng 1 nhân sự đều nổ `E11000` dù khác ngày/khác ca! Nhóm đã rà soát toàn bộ 18 collection trên Atlas, drop sạch index thừa `staffId_1_workDate_1` (0 mất dữ liệu) và siết thêm kiểm tra `err.keyPattern` ở `staffSchedule.service.js:126-139` để lỗi index thừa không bị nuốt thành `409`.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/config/db.js:1-13` (không gọi `syncIndexes()`), `src/models/StaffSchedule.js:17-18`, `src/services/staffSchedule.service.js:126-139` (guard `isScheduleKey` theo `err.keyPattern`).
  - Happy Path: Báo nhiều ca bận khác ngày/khác ca cho cùng 1 bác sĩ trả `201` bình thường (kiểm chứng `tests/staffSchedule.test.js`).
  - Fail / Edge Cases liên quan: Nếu còn index thừa và chưa check `err.keyPattern`, `POST /api/staff-schedules` trả nhầm `409` báo trùng lịch; sau khi thêm guard ở `L129-138`, nếu có lỗi index lạ sẽ ném `500` lộ ngay ra log để xử lý kịp thời.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Tại sao không bật `Model.syncIndexes()` tự động chạy mỗi lần server khởi động trong `db.js`?"
  - *Cách trả lời*: Tự động gọi `syncIndexes()` lúc startup trên production rất nguy hiểm vì nó có thể drop nhầm index đang phục vụ truy vấn nặng hoặc gây lock/tốn CPU khi rebuild index trên bảng hàng triệu dòng. Chuẩn production là quản lý thay đổi index qua **migration script có version** (vd `migrate-mongo`) chạy có kiểm soát trước khi deploy code mới.

### Q73. [Code Audit / Test Coverage & Seed Gap] [Mức độ: Nâng cao] Trong 26 bộ Jest Test, tại sao module `inpatientRecord` chưa có file test riêng? Và tại sao `seed.js` mới tạo dữ liệu mẫu cho 17/18 collection?
- **Ý đồ giám khảo**: Kiểm tra độ trung thực về độ phủ kiểm thử (Test Coverage) và quản lý kỹ thuật.
- **Trả lời miệng (30–45 giây)**: Em thừa nhận đúng 2 khoảng trống này: (1) Trong `tests/` đã có 26 file test bao phủ gần như toàn bộ hệ thống (kể cả `auth`, `authorize`, `admin`, `pet`, `petTimeline`, `service`, `product`, `cage`, `cageAssignment`, `staffSchedule`, `appointmentOrder`, `walkIn`, `appointmentService`, `medicalRecord`, `subtaskRequest`, `invoice`, `report`, `communityPost`, `aiAssessment`, `flagOverdueJob`), nhưng `inpatientRecord` hiện mới được kiểm chứng gián tiếp trong `tests/petTimeline.test.js` và `tests/invoice.test.js` chứ chưa có file `tests/inpatientRecord.test.js` chuyên biệt; (2) `src/seeds/seed.js:14-18` mới tạo dữ liệu mẫu cho 17/18 collection, chưa seed mẫu cho `AiAssessment` (dù `L11-13` vẫn tự require `AiAssessment.js` để đồng bộ index).
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `src/services/inpatientRecord.service.js:1-400` (chưa có `tests/inpatientRecord.test.js`), `src/seeds/seed.js:11-18`.
  - Happy Path: 26/26 bộ test hiện tại pass sạch trên `vcmos_test` local với `npm test`.
  - Fail / Edge Cases liên quan: Rủi ro regression nếu sửa `inpatientRecord.service.js` (vd logic chuyển chuồng `Holding → InpatientWard` hoặc validate ngày Daily Log) mà không có test tự động riêng.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Nếu cần viết ngay `tests/inpatientRecord.test.js`, anh ưu tiên test những case nào đầu tiên?"
  - *Cách trả lời*: Ưu tiên 6 case trọng yếu nhất: (1) Nhập viện thành công tự nhả chuồng `Holding` cũ & chiếm `InpatientWard`; (2) Chặn nhập viện đợt 2 khi đang `Admitted` (`409` + Partial Index); (3) Chặn chuồng sai phân khu/quá tải trọng (`400`); (4) Ghi Daily Log sai ngày tương lai/trước ngày nhập viện hoặc sau khi `Discharged` (`400`); (5) `PetOwner` xem Daily Log thú cưng người khác (`404` chống IDOR); (6) Xuất viện 2 lần (`409`) và giữ nguyên chuồng chờ thanh toán.

### Q74. [Code Audit / Legacy Code, Unused Deps & Future Scope] [Mức độ: Nâng cao] Soi `package.json` và `invoice.routes.js`: Tại sao vẫn còn thư viện `exceljs`/`moment` không import ở đâu, vẫn còn code VNPay bên cạnh PayOS, và những hạng mục nào (như `JOB-02` / Notification Center) đang là hướng phát triển tiếp theo?
- **Ý đồ giám khảo**: Kiểm tra mức độ sạch của repo (code hygiene) và sự minh bạch về phạm vi hoàn thiện so với SRS.
- **Trả lời miệng (30–45 giây)**: Em báo cáo thẳng 3 mục nợ dọn dẹp và phạm vi mở rộng: (1) `package.json:21,24` còn khai báo `exceljs` và `moment` từ giai đoạn đầu dự kiến làm xuất báo cáo Excel/xử lý ngày, sau đó nhóm chuyển sang dùng `pdfmake` và xử lý ngày thuần JS để nhẹ hơn nhưng chưa gỡ 2 dependency này ra khỏi `package.json`; (2) Code VNPay (`src/utils/vnpay.js`, `invoice.routes.js:8-9,30`) được giữ lại như lớp cổng dự phòng sau khi dự án chọn PayOS VietQR làm cổng thanh toán chính (toàn bộ Report 3/Report 4 đã cập nhật đồng bộ sang PayOS); (3) Trong định hướng mở rộng sau đồ án còn cụm `JOB-02` (nhắc lịch hẹn tự động) và Notification Center trong app (`UC-69..71`) cùng Redis cache/rate-limit.
- **Trace Code & HTTP Status (Mở chứng minh ngay)**:
  - Luồng chạy: `package.json:21,24` (`exceljs`, `moment` — 0 lần `require` trong `src/`), `src/utils/vnpay.js` & `src/routes/invoice.routes.js:8-9,30` (VNPay legacy), `src/utils/payos.js` (PayOS chính).
  - Happy Path: Không ảnh hưởng runtime vì `exceljs` và `moment` không được nạp vào RAM; toàn bộ test thanh toán PayOS & VNPay đều pass.
  - Fail / Edge Cases liên quan: Để thừa package làm tăng kích thước `node_modules` và diện tích kiểm tra lỗ hổng npm audit; giữ route VNPay không dùng trên production làm tăng attack surface không cần thiết.
- **Hỏi dồn tiếp theo (Follow-up) & Cách đỡ**:
  - *Hội đồng hỏi xoáy*: "Trước khi đưa lên môi trường thật (Production Go-Live), checklist dọn dẹp 3 bước của anh là gì?"
  - *Cách trả lời*: (1) `npm uninstall exceljs moment` và chạy `npm audit`; (2) Bật cờ feature flag (hoặc gỡ hẳn route VNPay nếu phòng khám chỉ ký hợp đồng PayOS); (3) Bổ sung `helmet`, `express-rate-limit`, 404 JSON catch-all, Magic Bytes chung cho cả 5 luồng upload và bộ test `inpatientRecord.test.js`.
