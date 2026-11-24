# 03 — Tích Hợp Bên Thứ 3 & Tiến Trình Ngầm Background Jobs (VCMOS_BE)

> **Phạm vi:** Toàn bộ 6 dịch vụ tích hợp ngoài (**Cloudinary**, **VNPay**, **PayOS**, **Google Gemini & Groq AI**, **Nodemailer Gmail SMTP**, **PDFMake**) và tiến trình chạy ngầm **Cron Job `JOB-01`** tự động đánh dấu lịch hẹn quá hạn (`Overdue`).

---

## 1. Bản đồ Code (Code Map)

| Hạng mục | File mã nguồn chính | Nơi gọi / Test liên quan |
| :--- | :--- | :--- |
| **Cloudinary** | `src/config/cloudinary.js` (`L1-17`) | `pet.service.js:37-70`, `appointmentService.service.js:527-575`, `subtaskRequest.service.js:46-62`, `communityPost.service.js:25-37`, `aiAssessment.service.js:51-65` |
| **VNPay v2.1.0** | `src/utils/vnpay.js` (`L1-132`) | `invoice.service.js:16`, `invoice.controller.js:105-138` · `tests/invoice.test.js` |
| **PayOS v2** | `src/utils/payos.js` (`L1-173`) | `invoice.service.js:17-22`, `invoice.controller.js:144-187` · `tests/invoice.test.js` |
| **Gemini & Groq AI** | `src/utils/gemini.js` (`L1-675`) | `aiAssessment.service.js:212-218` · `tests/aiAssessment.test.js` |
| **Nodemailer SMTP** | `user.service.js:26-70`, `appointmentOrder.service.js:52-120`, `inpatientRecord.service.js:26-70` | 3 luồng gửi email (OTP, Đặt lịch mới, Nhập/Xuất viện) |
| **PDFMake** | `src/services/invoice.service.js:30-37, 1058-1174` | `invoice.controller.js:260-263` (`GET /api/invoices/:id/pdf`) |
| **Background Job (`JOB-01`)** | `src/jobs/index.js` (`L1-31`), `src/jobs/flagOverdueAppointments.job.js` (`L1-187`) | `server.js:74-76`, `appointmentOrder.routes.js:23` · `tests/flagOverdueJob.test.js` (`L1-349`) |

---

## 2. Chi tiết 6 Tích hợp Bên Thứ 3

### 2.1. Cloudinary — Lưu trữ ảnh đám mây (`src/config/cloudinary.js`)
- **Fail-fast khi khởi động (`cloudinary.js:6-8`):** Kiểm tra đủ `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET`; thiếu là ném `Error` dừng app ngay (tránh lỗi muộn lúc khách upload). Bật `secure: true` (`L14`) để luôn trả HTTPS URL.
- **Kỹ thuật đẩy file:** Tất cả 5 luồng đều stream trực tiếp `Buffer` từ RAM (`multer.memoryStorage()`) bằng `cloudinary.uploader.upload_stream({ folder, resource_type: 'image' })` — không lưu file rác trên đĩa server.
- **So sánh chiến lược dọn ảnh mồ côi (Orphan Image Cleanup) khi lỗi DB:**
  - **`pet.service.js:62-70, 104`:** Có `removePhoto(url)` tách `publicId` bằng regex (`L55-59`) và gọi `cloudinary.uploader.destroy` (best-effort) nếu `Pet.create` lỗi hoặc khi thay ảnh mới.
  - **`appointmentService.service.js:540-543, 562-575`:** Upload song song bằng `Promise.allSettled`; nếu ghi DB (`groomingNote: null` atomic) thất bại, gọi `deleteCloudinaryImages(uploadedPublicIds)` xóa sạch ảnh vừa đẩy.
  - **`subtaskRequest.service.js:55-62`:** Có `deleteCloudinaryImage(publicId)` dọn ảnh khi thao tác cập nhật kết quả thất bại.
  - **`aiAssessment.service.js:212-223`:** **Chủ động gọi AI thành công TRƯỚC rồi mới upload ảnh lên Cloudinary** (`L220-223`), nên nếu AI lỗi/timeout (`503`/`504`) sẽ không tốn lượt upload và không sinh ảnh mồ côi. *(Lưu ý: nếu bước `AiAssessment.create` ngay sau đó lỗi DB thì chưa có đoạn `destroy` lại ảnh — tương tự `communityPost.service.js:25-37`).*

### 2.2. VNPay v2.1.0 — Thanh toán QR & IPN (`src/utils/vnpay.js`)
- **Cấu hình (`vnpay.js:54-67`):** Bắt buộc `VNP_TMN_CODE`, `VNP_HASH_SECRET` (thiếu ném `500`); mặc định dùng sandbox `VNP_URL` và `VNP_RETURN_URL`.
- **Tạo URL thanh toán (`buildVnpayPaymentUrl`, `L72-102`):**
  - Thời gian `vnp_CreateDate` / `vnp_ExpireDate` tính thủ công theo UTC+7 (`formatVnpayDateVN` `L9-20`), hiệu lực **15 phút** (`VNPAY_EXPIRE_MINUTES = 15`, `L6`).
  - Quy đổi số tiền theo chuẩn VNPay: `vnp_Amount = Math.round(amount) * 100` (`L88`).
  - Chuẩn hóa IPv6 (`::1`, `::ffff:x.x.x.x`) về IPv4 (`normalizeClientIp` `L23-29`).
  - Sắp xếp key alphabet, encode URI đổi `%20 → +` (`sortAndEncodeVnpayParams` `L32-44`), ký **HMAC-SHA512** (`signVnpayData` `L46-52`).
- **Xác thực chữ ký (`verifyVnpaySignature`, `L107-121`):** Loại bỏ `vnp_SecureHash` & `vnp_SecureHashType`, ký lại dữ liệu và so sánh bằng **`crypto.timingSafeEqual`** (`L120`) chống timing attack.
- **Tách biệt IPN vs Return URL (`invoice.controller.js:117-138`):**
  - `GET /api/invoices/vnpay-ipn` (server-to-server): là **nguồn duy nhất** cập nhật hóa đơn sang `Paid`, luôn trả HTTP `200` với `{ RspCode, Message }` (lỗi hệ thống trả `RspCode: '99'`).
  - `GET /api/invoices/vnpay-return` (browser callback): chỉ xác thực chữ ký và trả kết quả hiển thị, **tuyệt đối không tự đổi trạng thái `Paid`** (`BR-17`).

### 2.3. PayOS API v2 — Thanh toán VietQR thật (`src/utils/payos.js`)
- **Cấu hình (`payos.js:7-27`):** Cần `PAYOS_CLIENT_ID`, `PAYOS_API_KEY`, `PAYOS_CHECKSUM_KEY` (thiếu ném `500`).
- **Tạo chữ ký khởi tạo (`buildPayosCreateSignature`, `L55-58`):** Ký **HMAC-SHA256** đúng 5 trường xếp alphabet: `amount`, `cancelUrl`, `description` (cắt tối đa 25 ký tự tại `L123`), `orderCode`, `returnUrl`.
- **Gọi HTTP có Timeout (`callPayosApi`, `L78-114`):** Dùng native `fetch` + `AbortController` giới hạn **15 giây** (`REQUEST_TIMEOUT_MS = 15000`, `L5`):
  - Vượt 15s (`AbortError`) → ném **`504`** (`L105-108`).
  - HTTP không `ok` hoặc `json.code !== '00'` → ném **`502`** (`L96-100`).
- **Xác thực Webhook (`verifyPayosWebhookSignature`, `L63-76`):** Sắp xếp key của `webhookBody.data` theo alphabet (`sortObjectByKey` `L30-36`), chuyển chuỗi query (`convertObjectToQueryStr` `L38-50`, chuẩn hóa `null`/`undefined` thành chuỗi rỗng), ký HMAC-SHA256 và so bằng **`crypto.timingSafeEqual`** (`L75`).
- **Chủ động đồng bộ trạng thái (`getPayosPaymentInfo`, `L159-163`):** Gọi `GET /v2/payment-requests/{orderCode}` phục vụ endpoint `GET /api/invoices/:id/payos-status` giúp chốt đơn `Paid` ngay cả khi chạy trên `localhost` (nơi PayOS không bắn được Webhook về).

### 2.4. Trợ lý AI Đa Cổng — Google Gemini & Groq Failover (`src/utils/gemini.js`)
- **Thu thập & xoay vòng nhiều API Key (`collectProviderApiKeys`, `L50-79`):** Đọc đồng thời cả 2 kiểu khai báo `.env` — kiểu gộp dấu phẩy (`GEMINI_API_KEY=k1,k2`) và kiểu đánh số (`GEMINI_API_KEY_1..N`, `GROQ_API_KEY_1..N`), lọc trùng bằng `Set` và sắp theo số thứ tự.
- **Model mặc định (`L6-8, L529-534`):**
  - Gemini: `gemini-3.5-flash-lite` (multimodal text + ảnh inline `base64` qua `v1beta/models/{model}:generateContent`, `L375-443`).
  - Groq text: `openai/gpt-oss-120b`; tự chuyển sang Groq Vision `qwen/qwen3.8-27b` khi `hasImages === true` (`L448-520, L530-534`).
  - Biến `AI_PRIMARY_PROVIDER` (`L550-554`) quyết định ưu tiên `gemini` trước hay `groq` trước (`buildProviderAttempts`).
- **Ngân sách Timeout tổng `<= 30s` (`analyzePetHealthWithAi`, `L561-658`):**
  - Không có key nào → ném ngay **`503`** (`MSG34`, `L572-574`).
  - Đặt mốc `deadline = startedAt + 30000ms` (`L585`). Ở mỗi lượt thử `i`, tính `remainingMs = deadline - Date.now()`. Nếu còn lượt dự phòng phía sau, chia quỹ thời gian `perAttemptTimeoutMs = Math.ceil(remainingMs / Math.min(remainingAttempts, 3))` (`L598-601`) để đảm bảo 1 key bị treo không nuốt trọn 30s, vẫn còn thời gian chuyển sang key/provider tiếp theo.
  - Nếu mọi lượt đều thất bại → ném **`504`** nếu từng gặp timeout (`sawTimeout === true`), ngược lại ném **`503`** (`L657`), đều mang thông điệp chuẩn `MSG34` (`'Dịch vụ AI đang bận. Vui lòng thử lại sau.'`).
- **Prompt chuẩn thú y & Chuẩn hóa đầu ra (`L119-368`):**
  - System Prompt theo chuẩn **AAHA / AVMA Veterinary Teletriage** (`L126-149`), cấm chẩn đoán khẳng định và cấm kê thuốc (`BR-20`).
  - `extractJsonObject` (`L223-254`): bóc JSON trần, block ```` ```json ```` hoặc cắt `{...}` ngoài cùng.
  - `parseAndNormalizeAiJson` (`L260-368`): chuẩn hóa `urgency` về `Routine | Monitor | Urgent`, **đối chiếu mã dịch vụ gợi ý với danh sách `activeServices` thật trong DB** (chống AI bịa dịch vụ ảo, tối đa 3 dịch vụ, `L301-344`), tạo sẵn `bookingNoteDraft` (≤500 ký tự, `L350-353`) và luôn gắn câu miễn trừ `MSG33` (`L366`).

### 2.5. Nodemailer — 3 Luồng gửi Email Gmail SMTP
Cả 3 luồng đều kiểm tra `if (!emailUser || !emailPass || process.env.NODE_ENV === 'test')`: ở môi trường dev chưa điền Gmail thì chỉ `console.log` (không crash), ở môi trường test thì im lặng hoàn toàn (`return false`):
1. **OTP xác thực / đổi & quên mật khẩu** (`user.service.js:26-70`): gửi OTP 6 số hiệu lực 5 phút (`OTP_TTL_MS`), dev mode trả thêm `devToken` để test dễ dàng.
2. **Thông báo lịch hẹn online mới cho Lễ tân** (`appointmentOrder.service.js:52-120`): gửi `to: emailUser`, **`bcc: recipients`** (`L79-80`) để không lộ email giữa các lễ tân; dùng `escapeHtml` (`L44-50`); chạy fire-and-forget (lỗi SMTP chỉ log `console.error`, không làm hỏng đơn hẹn).
3. **Thông báo Nhập viện / Xuất viện cho Chủ nuôi** (`inpatientRecord.service.js:26-70`): chỉ gửi nếu chủ nuôi có email, dùng `escapeHtml` (`L17-18`), lỗi SMTP chỉ log không làm fail giao dịch nội trú.

### 2.6. PDFMake — Xuất Biên lai Hóa đơn PDF (`src/services/invoice.service.js:30-37, 1058-1174`)
- Khởi tạo `PdfPrinter` 1 lần ở module scope (`L30-37`) nạp 4 font `Roboto` từ `pdfmake/build/vfs_fonts` (hỗ trợ đầy đủ dấu tiếng Việt).
- `generateInvoicePdf` (`L1058-1174`): kiểm tra tồn tại (`404`) + quyền truy cập (`assertCanAccessInvoice` chống IDOR cho `PetOwner`), render `docDefinition` từ **dữ liệu snapshot** trong `invoice.items`, gom chunks thành `Buffer` trong RAM (`L1161-1168`).
- Controller (`invoice.controller.js:260-263`) trả về với header `Content-Type: application/pdf` và `Content-Disposition: inline; filename="INV-XXXXXX.pdf"`.

---

## 3. Trace chi tiết Tiến trình ngầm `JOB-01` (Đánh dấu Quá hạn `Overdue`)

### 3.1. Cơ chế kích hoạt (`server.js:73-76`, `src/jobs/index.js:13-27`)
1. Chỉ khởi chạy khi chạy server trực tiếp (`require.main === module`), **không chạy trong Jest test**.
2. `startBackgroundJobs()` (`jobs/index.js:13-27`):
   - Đăng ký Cron Job chạy lúc **`23:59` hằng ngày giờ Việt Nam (`Asia/Ho_Chi_Minh`)** qua `scheduleFlagOverdueJob()` (`flagOverdueAppointments.job.js:148-174`). Cho phép ghi đè biểu thức cron qua biến môi trường `OVERDUE_JOB_CRON` (nếu sai cú pháp `cron.validate()` tự rơi về mặc định `'59 23 * * *'`, `L149-150`).
   - **Startup Catch-up (`jobs/index.js:17-24`):** Ngay khi DB kết nối lúc server bật lên, chạy bất đồng bộ `runFlagOverdueWithRetry({ targetDate: getYesterdayStringVN(), includePastDates: true, maxRetries: 2, retryDelayMs: 5000 })` để quét bù các đơn từ **ngày hôm qua trở về trước** (phòng trường hợp server tắt lúc 23:59), **tuyệt đối không đụng vào đơn `Booked` của ngày hôm nay**.
3. **API chạy tay cho Admin:** `POST /api/appointment-orders/flag-overdue` (`appointmentOrder.routes.js:23`, `protect` + `authorize('Admin')`).

### 3.2. Logic nghiệp vụ `flagOverdueAppointments` (`flagOverdueAppointments.job.js:39-107`)
1. Validate `targetDate` đúng `YYYY-MM-DD` lịch thật (chặn ngày ảo như `2026-02-31`, `L23-27, 43-47`) → sai ném **`400`**.
2. Quét ứng viên (`L49-58`):
   ```js
   { date: includePastDates ? { $lte: targetDate } : targetDate, derivedStatus: 'Booked', checkedInAt: null }
   ```
   → Đơn đã `CheckedIn` (kể cả dịch vụ sau bị trễ), `InProgress`, `Done`, `Cancelled`, hoặc `date > targetDate` đều bị bỏ qua (`UC-23 Alt 1a`).
3. **Chống Race Condition với Lễ tân Check-in (`L64-75`):** Với từng đơn ứng viên, không dùng `updateMany` mù quáng mà dùng lệnh nguyên tử **`AppointmentOrder.findOneAndUpdate({ _id: order._id, derivedStatus: 'Booked', checkedInAt: null }, { $set: { derivedStatus: 'Overdue' } })`**. Nếu Lễ tân vừa Check-in đúng mili-giây trước đó, điều kiện lọc không khớp → trả `null` → tăng `skippedCount` và bỏ qua an toàn.
4. **Quy tắc `BR-08` (`L77-82`):** Chỉ đánh dấu `Overdue` cho **dịch vụ đầu tiên (`sort: { sequenceNo: 1 }`)** còn `Booked` bằng `AppointmentService.findOneAndUpdate`. Các dịch vụ thứ 2, 3 giữ nguyên `Booked`; đơn **không bị hủy** (`cancelReason: null`).
5. **Retry theo SRS 6.1 (`runFlagOverdueWithRetry`, `L112-143`):** Khi DB gián đoạn, thử lại tối đa **3 lần**, mỗi lần cách nhau **30 giây** (`DEFAULT_RETRY_DELAY_MS = 30000`). Thất bại đủ 3 lần thì ghi log `[JOB-01 Overdue][CRITICAL]` và catch trong cron callback (`L160-162`) để không làm sập Express server.
6. **Kiểm chứng test (`tests/flagOverdueJob.test.js:82-349`):** Đã phủ đủ 6 kịch bản: `BR-08` chỉ dịch vụ #1, lọc bỏ tương lai/CheckedIn/Done/Cancelled, giả lập race condition check-in, retry 3 lần khi `MongoNetworkError`, đăng ký & dừng cron sạch sẽ, và API chạy tay (`401`/`403`/`400`/`200` + đồng bộ Pet Timeline & Reports).

---

## 4. Ma trận Lỗi Tích hợp Bên thứ 3 & Background Job

| Tích hợp / Job | Kịch bản lỗi | HTTP Status | Xử lý / Message thực tế | Tọa độ code (`file:line`) |
| :--- | :--- | :-: | :--- | :--- |
| **Cloudinary** | Thiếu 1 trong 3 env `CLOUDINARY_*` lúc khởi động | *(crash startup)* | `Error('Missing Cloudinary credentials...')` | `src/config/cloudinary.js:6-8` |
| **Cloudinary** | Lỗi mạng / API Cloudinary khi upload ảnh | **`502`** | `'Không thể tải ảnh ... lên hệ thống, vui lòng thử lại sau'` | `aiAssessment.service.js:58`, `communityPost.service.js:31`, `subtaskRequest.service.js:297`, `appointmentService.service.js` |
| **VNPay** | Thiếu `VNP_TMN_CODE` hoặc `VNP_HASH_SECRET` | **`500`** | `'Chưa cấu hình đầy đủ VNP_TMN_CODE và VNP_HASH_SECRET...'` | `src/utils/vnpay.js:60-64` |
| **VNPay IPN** | Sai chữ ký HMAC / Exception không lường trước | **`200`** *(chuẩn VNPay)* | `{ RspCode: '97', Message: 'Invalid signature' }` / `{ RspCode: '99', Message: 'Unknown error' }` | `vnpay.js:107-121`, `invoice.controller.js:121-124` |
| **PayOS** | Thiếu `PAYOS_CLIENT_ID / API_KEY / CHECKSUM_KEY` | **`500`** | `'Chưa cấu hình đầy đủ PAYOS_CLIENT_ID...'` | `src/utils/payos.js:18-24` |
| **PayOS** | API PayOS trả lỗi HTTP hoặc `json.code !== '00'` | **`502`** | `json.desc || 'Lỗi kết nối cổng thanh toán PayOS...'` | `src/utils/payos.js:96-101` |
| **PayOS** | Gọi API PayOS vượt quá 15 giây (`AbortError`) | **`504`** | `'Kết nối tới PayOS quá thời gian chờ, vui lòng thử lại'` | `src/utils/payos.js:105-109` |
| **Gemini/Groq** | Chưa cấu hình key nào hoặc tất cả key lỗi / hết quota | **`503`** | `'Dịch vụ AI đang bận. Vui lòng thử lại sau.'` (`MSG34`) | `src/utils/gemini.js:572-574, 657` |
| **Gemini/Groq** | Tất cả lượt gọi thất bại và có ít nhất 1 lượt quá timeout (`<= 30s`) | **`504`** | `'Dịch vụ AI đang bận. Vui lòng thử lại sau.'` (`MSG34`) | `src/utils/gemini.js:435, 512, 657` |
| **Nodemailer** | Chưa cấu hình `EMAIL_USER/EMAIL_PASS` | *(fallback dev)* | In log console, trả `false` (không làm lỗi nghiệp vụ chính) | `user.service.js:30-35`, `appointmentOrder.service.js:57-64`, `inpatientRecord.service.js:30-35` |
| **JOB-01 API** | `POST /api/appointment-orders/flag-overdue` sai định dạng ngày (vd `2026-02-31`) | **`400`** | `{ message: 'Dữ liệu không hợp lệ', errors: { date: ... } }` | `flagOverdueAppointments.job.js:43-47`, `appointmentOrder.controller.js` |
| **JOB-01 Cron** | DB mất kết nối khi cron 23:59 chạy | *(background)* | Retry 3 lần cách nhau 30s → log `CRITICAL`, không crash Express | `flagOverdueAppointments.job.js:121-143, 160-162` |

---

## 5. Điểm nóng kỹ thuật & Bộ câu hỏi phản biện (Defense Q&A)

### Q1: Tại sao VNPay có 2 URL (`vnpay-return` và `vnpay-ipn`)? Nếu khách tự sửa query `vnp_ResponseCode=00` trên URL trình duyệt thì có gian lận được không?
- **Trả lời (30s):** Tuyệt đối không gian lận được vì 2 lớp bảo vệ: (1) Mọi tham số đều được ký **HMAC-SHA512** với `VNP_HASH_SECRET` nằm trên server, sửa 1 ký tự là `verifyVnpaySignature` trả `false` ngay; (2) Theo `BR-17`, `vnpay-return` chỉ dùng để hiển thị giao diện kết quả cho khách và **không bao giờ sửa DB**, chỉ có `vnpay-ipn` (server VNPay gọi thẳng server BE) mới được chuyển hóa đơn sang `Paid`.
- **Code chứng minh:** `src/utils/vnpay.js:107-121`, `src/controllers/invoice.controller.js:117-138`.

### Q2: Tại sao khi so sánh chữ ký VNPay và PayOS lại dùng `crypto.timingSafeEqual` thay vì dấu `===`?
- **Trả lời (30s):** So sánh chuỗi thông thường bằng `===` dừng ngay ở ký tự đầu tiên khác nhau, tạo ra chênh lệch thời gian nano-giây mà kẻ tấn công có thể khai thác (Timing Attack) để dò từng byte của chữ ký HMAC. `crypto.timingSafeEqual` luôn duyệt hết toàn bộ độ dài Buffer trong thời gian hằng số, chuẩn bảo mật bắt buộc cho Payment Webhook.
- **Code chứng minh:** `src/utils/vnpay.js:119-120`, `src/utils/payos.js:74-75`.

### Q3: Tại sao PayOS đã có Webhook (`POST /payos-webhook`) rồi mà vẫn phải làm thêm API `GET /:id/payos-status`?
- **Trả lời (30s):** Khi chạy demo trên máy cá nhân (`localhost`) hoặc khi Webhook bị trễ/rớt mạng, server PayOS bên ngoài không gọi vào được localhost. API `payos-status` cho phép FE/Lễ tân chủ động gọi `getPayosPaymentInfo()` lên PayOS để đối chiếu trạng thái `PAID` trực tiếp từ cổng thanh toán và chốt hóa đơn an toàn mà vẫn đảm bảo idempotency.
- **Code chứng minh:** `src/utils/payos.js:159-163`, `src/controllers/invoice.controller.js:175-187`.

### Q4: Cơ chế Failover AI hoạt động thế nào nếu Gemini bị 503 hoặc hết quota 429 giữa lúc bảo vệ? Và làm sao tổng thời gian không vượt 30 giây?
- **Trả lời (30s):** `buildProviderAttempts` xếp hàng toàn bộ key Gemini (`GEMINI_API_KEY_1..N`) nối tiếp bằng toàn bộ key Groq (`GROQ_API_KEY_1..N`). Em đặt 1 deadline cứng `startedAt + 30s` cho cả phiên. Khi còn key dự phòng, mỗi lượt thử chỉ được cấp tối đa `remainingMs / min(remainingAttempts, 3)` qua `AbortController`. Nhờ đó nếu Gemini key #1 sập hay treo, hệ thống lập tức xoay sang key #2 rồi tự chuyển sang Groq (tự dùng `qwen/qwen3.8-27b` nếu có ảnh) trong cùng 1 request dưới 30s.
- **Code chứng minh:** `src/utils/gemini.js:525-658`.

### Q5: Nếu AI gợi ý một mã dịch vụ bịa (hallucination) không có trong phòng khám thì sao?
- **Trả lời (30s):** Trong `parseAndNormalizeAiJson`, em build 2 `Map` (`serviceByCode`, `serviceByName`) từ danh sách dịch vụ `Active` thật đọc từ MongoDB. Mọi gợi ý từ AI đều phải đối chiếu khớp với `Map` đó (`L327-328`), nếu AI bịa mã lạ sẽ bị lọc bỏ hoàn toàn, không bao giờ trả dịch vụ ảo về FE.
- **Code chứng minh:** `src/utils/gemini.js:301-344`.

### Q6: Trong luồng AI Assessment, tại sao lại gọi AI trước rồi mới upload ảnh lên Cloudinary?
- **Trả lời (30s):** Ảnh gửi từ Multer đang nằm sẵn dạng `Buffer` trong RAM nên có thể mã hóa `base64` gửi thẳng cho Gemini/Groq ngay lập tức. Gọi AI trước giúp: (1) Nếu AI lỗi/timeout (`503`/`504`) thì không đẩy rác lên Cloudinary; (2) Giảm tổng độ trễ trước khi gọi AI.
- **Code chứng minh:** `src/services/aiAssessment.service.js:212-223`.
- **Hạn chế & cải thiện:** Hiện tại nếu bước `AiAssessment.create()` ngay sau `L225` bị lỗi DB thì ảnh vừa đẩy ở `L222` chưa có `try/catch` xóa ngược như ở `pet.service.js` — có thể bổ sung `destroy` tương tự để triệt để hơn.

### Q7: Trong `JOB-01`, tại sao không dùng 1 câu `AppointmentOrder.updateMany(...)` cho nhanh mà phải duyệt vòng `for` + `findOneAndUpdate`?
- **Trả lời (30s):** Có 2 lý do: (1) `findOneAndUpdate` kiểm tra nguyên tử `{ _id, derivedStatus: 'Booked', checkedInAt: null }` cho từng đơn, giúp biết chính xác đơn nào thực sự vừa chuyển sang `Overdue` để chỉ cập nhật tiếp **đúng dịch vụ đầu tiên** (`sort: { sequenceNo: 1 }`) của đơn đó theo `BR-08`; (2) Nếu Lễ tân vừa Check-in đúng lúc job chạy, `findOneAndUpdate` trả `null` và bỏ qua an toàn thay vì ghi đè sai trạng thái dịch vụ.
- **Code chứng minh:** `src/jobs/flagOverdueAppointments.job.js:64-91`.

### Q8: Nếu lúc `23:59` server đang tắt (hoặc deploy lại) thì các lịch hẹn hôm đó có bị bỏ sót mãi ở trạng thái `Booked` không?
- **Trả lời (30s):** Không bị sót. Trong `src/jobs/index.js`, ngay sau khi `connectDB()` thành công lúc server bật lại, hệ thống tự chạy một lượt **startup catch-up** với `targetDate = getYesterdayStringVN()` và `includePastDates: true`. Nó quét bù toàn bộ đơn `Booked` từ hôm qua trở về trước, đồng thời không bao giờ đụng vào đơn của ngày hôm nay. Ngoài ra Admin cũng có API chạy tay `POST /api/appointment-orders/flag-overdue`.
- **Code chứng minh:** `src/jobs/index.js:16-24`, `src/routes/appointmentOrder.routes.js:23`.

### Q9: Tại sao chạy `npm test` không bị Cron Job `JOB-01` chạy ngầm làm treo Jest (open handles)?
- **Trả lời (30s):** `server.js:73-76` đặt `connectDB().then(() => startBackgroundJobs())` bên trong khối `if (require.main === module)`. Khi Supertest `require('../server')`, khối này không thực thi. Riêng bài test kiểm tra đăng ký cron (`tests/flagOverdueJob.test.js:276-282`) chủ động gọi `task.stop()` ngay sau khi assert để đóng timer sạch sẽ.
- **Code chứng minh:** `server.js:73-79`, `tests/flagOverdueJob.test.js:276-282`.
