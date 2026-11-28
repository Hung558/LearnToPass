# BONUS — Các Luồng Hỗ Trợ (`SRS 2.9 → 2.19`: AI Health Triage, Timeline, Cộng Đồng & Quản Trị)

> **Khi nào dùng file này?** Sau khi hoàn tất mạch chính **`MF-01` → `MF-08`**, hội đồng thường sẽ hỏi thêm về **điểm nhấn sáng tạo của dự án (AI Health Assessment, Cộng đồng bảo mật dữ liệu y tế)** hoặc kiểm tra các chức năng **Quản trị nền tảng & Bảo mật**. Mở nhanh bảng dưới đây để demo thêm đúng tính năng thầy hỏi trong 1–2 phút!

---

## 1. Luồng 2.11 — Trợ Lý AI Đánh Giá Sức Khỏe Tham Khảo (`SCR-10`, `/ai-assessment`)

- **Nói trong 15 giây:** *"Khác với việc hỏi ChatGPT thông thường, AI của VCMOS tự động đọc ngữ cảnh từ **8 bảng dữ liệu thực của thú cưng & phòng khám** (bệnh án cũ, dị ứng, đơn thuốc, dịch vụ `Active`), hỗ trợ 0–3 ảnh (kiểm tra Magic Bytes), xoay vòng tự động đa nhà cung cấp **Gemini ↔ Groq** trong ngân sách `<= 30s`, đối chiếu dịch vụ thật chống AI bịa, sinh sẵn ghi chú `bookingNoteDraft` chuyển thẳng sang `MF-01`, và tách riêng hoàn toàn khỏi bệnh án chính thức (`BR-20`)."*
- **Demo nhanh (Cửa sổ #1 — `PetOwner`):**
  1. Vào `/ai-assessment` → chọn thú cưng → nhập triệu chứng (*"Bé gãi tai nhiều, vành tai hơi đỏ, lắc đầu liên tục 2 ngày"*) + chọn 1 ảnh JPG/PNG → bấm **Phân tích AI** (`POST /api/ai-assessments`).
  2. Chỉ rõ 4 điểm trên kết quả: (a) Câu cảnh báo miễn trừ không thay thế bác sĩ (`MSG33`), (b) Mức độ khẩn cấp (`Routine / Monitor / Urgent`) & dấu hiệu cảnh báo, (c) Dịch vụ gợi ý khớp đúng dịch vụ thật của phòng khám, (d) Bấm nút **Đặt lịch ngay** → tự điền sẵn `bookingNoteDraft` sang form `MF-01`!
- **Chi tiết code:** `VCMOS_BE/docs/defense/24_AI_HEALTH_ASSESSMENT.md` (`aiAssessment.service.js`, `src/utils/gemini.js`).

---

## 2. Luồng 2.12 — Dòng Thời Gian Hợp Nhất & Phân Quyền Riêng Tư (`SCR-08`, `/pet-journeys`)

- **Nói trong 15 giây:** *"Gom dữ liệu từ **10 collection** trong đúng **3 đợt `Promise.all` dùng `$in` + `Map O(1)`** nên không bao giờ bị lỗi N+1 query dù thú cưng có hàng trăm lượt khám."*
- **Demo nhanh (2 cửa sổ song song để chứng minh bảo mật `BR-23` & `BR-20`):**
  - **Cửa sổ #1 (`PetOwner`):** Mở `/pet-journeys` của bé `Milo` → thấy ĐẦY ĐỦ lịch sử khám, ảnh Grooming, Sub-task, **Hóa đơn** và **Đánh giá AI**.
  - **Cửa sổ #3 (`Veterinarian`):** Mở Timeline của chính bé `Milo` → thấy đủ bệnh án & Grooming để hỗ trợ chuyên môn, nhưng **Hóa đơn tiền (`invoice: null`) và Đánh giá AI (`aiAssessments: []`) bị ẩn hoàn toàn**!
- **Chi tiết code:** `VCMOS_BE/docs/defense/08_UNIFIED_PET_TIMELINE.md`.

---

## 3. Luồng 2.15 & 2.16 — Cộng Đồng Ẩn Thông Tin Y Tế & Kiểm Duyệt (`SCR-03`, `SCR-14`, `SCR-33`)

- **Nói trong 15 giây:** *"Chủ nuôi có thể chia sẻ hành trình chăm sóc thú cưng sau khi đơn hoàn tất (`Done`). Hệ thống bảo vệ quyền riêng tư 2 lớp (`BR-18`): server chỉ đọc ngày khám, tên dịch vụ và ảnh Grooming — **tuyệt đối không đụng vào bệnh án SOAP, đơn thuốc hay tổng tiền hóa đơn**. Mọi bài đăng mới đều phải qua hàng đợi `PendingReview` để Admin kiểm duyệt trước khi ra bảng tin công khai."*
- **Demo nhanh (2 cửa sổ):**
  1. **Cửa sổ #1 (`PetOwner`):** Từ đơn `Done` hoặc `/community/my-posts` → bấm **Chia sẻ hành trình (`TimelineShare`)** (`GET /timeline-share/preview` → `POST /timeline-share`) → bài vào trạng thái **`PendingReview`** (chưa hiện ra ngoài).
  2. **Cửa sổ #5 (`Admin`):** Mở `/admin/community-moderation` (`SCR-33`) → thấy bài chờ duyệt (có hệ thống tự đánh dấu từ khóa nhạy cảm) → bấm **Duyệt (Approve)** (`PATCH /api/admin/community/posts/:id/moderate`).
  3. **Cửa sổ #1 (`PetOwner`):** Mở `/community` (`SCR-14`) → bài đã lên công khai → thử **Like** (toggle chống trùng) và **Comment / Reply cấp 1** (BE chặn reply cấp 2 theo `BR-19` trả `400`).
- **Chi tiết code:** `22_COMMUNITY_POSTS_AND_INTERACTIONS.md`, `23_ADMIN_COMMUNITY_MODERATION.md`.

---

## 4. Luồng 2.14 & 2.17–2.19 — Quản Trị Nền Tảng, Bảo Mật & Cron Job Quá Hạn

| Luồng SRS | Màn hình FE (`SCR` / URL) | Điểm nhấn 10 giây nếu hội đồng hỏi | Tài liệu chi tiết |
| :--- | :--- | :--- | :--- |
| **2.9 Auth & Security** | `/login`, `/register`, `/profile` | Đăng nhập SĐT/Email; sai mật khẩu 5 lần khóa 30 phút (`BR-03`); OTP email 6 số băm bcrypt hiệu lực 5 phút; `protect` query live DB để khóa tài khoản là **đá văng ngay lập tức (`403`)**. | `04_AUTH.md`, `05_USER_PROFILE_OTP_STAFF.md` |
| **2.17 Account Mgmt** | `SCR-32` `/admin/users` | Admin tạo tài khoản nhân sự `USR-XXXXXX`, chuyển vai trò, khóa/mở khóa (chặn tự khóa mình), và **chặn xóa vật lý `DELETE` trả `403` (`BR-22`)**. | `06_ADMIN_USER_MANAGEMENT.md` |
| **2.18 Services & Roster** | `SCR-29` `/admin/services`<br>`SCR-30` `/staff-schedules` | Dịch vụ khóa xóa vật lý (`403`); Lịch trực dùng **Blacklist 3 ca (`MORNING/AFTERNOON/EVENING` 08:00–21:00)** giúp giảm 90% thao tác nhập lịch và tự cảnh báo xung đột đơn đang chờ! | `09_SERVICE_CATALOG.md`, `13_STAFF_SCHEDULE_BLACKLIST.md` |
| **2.19 Products & Cages** | `SCR-31` `/admin/products`<br>`SCR-34` `/cages` | Sản phẩm dùng cờ `Active/Inactive` (Available / Out of Stock) không làm kho đếm tồn; Quản lý chuồng (`SCR-34`) phân rõ 2 khu `Holding` và `InpatientWard`. | `10_PRODUCT_CATALOG.md`, `11_CAGE_MANAGEMENT.md` |
| **2.14 Overdue Job** | `POST /api/appointment-orders/flag-overdue` | `JOB-01` chạy tự động `23:59` VN + chạy quét bù khi khởi động server, chỉ đánh `Overdue` đơn chưa check-in & dịch vụ #1 (`BR-08`), dùng điều kiện atomic chống đè lên lễ tân vừa check-in. | `03_INTEGRATIONS_AND_BACKGROUND_JOBS.md` |
