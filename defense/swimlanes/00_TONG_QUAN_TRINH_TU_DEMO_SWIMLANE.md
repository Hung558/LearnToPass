# 00 — Bản Đồ Tổng Đạo Diễn Demo Hội Đồng Theo 8 Swimlane (`MF-01` → `MF-08`)

> **Mục tiêu:** Đi **đúng 100% trình tự 8 Main Business Flows (`MF-01` → `MF-08`)** trong `Report-3 SRS (trang 8–19)` như giảng viên hướng dẫn yêu cầu, nối thành **1 câu chuyện vận hành phòng khám liền mạch**, rõ từng bước màn hình FE ↔ API BE ↔ Database ↔ câu trả lời khi hội đồng hỏi xoáy.

---

## 1. Mục lục 10 tài liệu trong `docs/defense/swimlanes/`

| File | Luồng SRS | Trang SRS | Vai trò tham gia | Thời lượng demo gợi ý |
| :--- | :--- | :-: | :--- | :-: |
| `00_TONG_QUAN_TRINH_TU_DEMO_SWIMLANE.md` *(file này)* | Bản đồ tổng + chuẩn bị + bảng chênh lệch SRS vs Code | 8–21 | Cả 5 role | *(Đọc trước buổi bảo vệ)* |
| `MF-01_ONLINE_BOOKING_AND_CHECKIN.md` | **2.1 Online Appointment Booking, Confirmation & Check-in** | 8–9 | PetOwner → System → Receptionist | ~2.0 phút |
| `MF-02_WALKIN_RECEPTION_AND_CHECKIN.md` | **2.2 Walk-in Customer Reception & Check-in** | 10 | Receptionist → System | ~1.5 phút |
| `MF-03_CLINICAL_EXAMINATION_SOAP.md` | **2.3 Clinical Examination & Prescription** | 11–12 | Veterinarian → System | ~2.0 phút |
| `MF-04_SUBTASK_DIAGNOSTICS_AND_DELEGATION.md` | **2.4 Serious Illness Treatment & Sub-Task Diagnostics** | 13–14 | Veterinarian → System → Cloudinary → Groomer/Vet | ~2.0 phút |
| `MF-05_PET_SPA_AND_GROOMING.md` | **2.5 Pet Spa & Grooming Service** | 15–16 | Groomer → System | ~1.5 phút |
| `MF-06_INPATIENT_CARE_DAILY_ROUNDS_DISCHARGE.md` | **2.6 Inpatient Care, Daily Rounds & Discharge** | 17 | Veterinarian → System → PetOwner | ~2.0 phút |
| `MF-07_CONSOLIDATED_BILLING_PAYOS_HANDOVER.md` | **2.7 Consolidated Billing, Cash / PayOS VietQR & Handover** | 18 | PetOwner → Receptionist → System → PayOS | ~2.5 phút |
| `MF-08_OPERATIONAL_AND_REVENUE_ANALYTICS.md` | **2.8 Clinic Operational & Revenue Analytics** | 19 | Admin → System | ~1.5 phút |
| `MF-BONUS_SUPPORTING_FLOWS_AI_COMMUNITY_ADMIN.md` | **2.9 → 2.19 AI Health, Timeline, Community, Admin, JOB-01** | 20–21 | Guest, PetOwner, Admin | *(Khi hội đồng hỏi thêm)* |

> **Tổng thời gian demo chính (`MF-01` → `MF-08`):** ~**15 phút** (nếu hội đồng yêu cầu rút gọn còn 10 phút, ghép `MF-03 + MF-04` trên cùng 1 ca khám và rút ngắn phần nhập liệu).

---

## 2. Mạch câu chuyện xuyên suốt (Demo 2 đơn song song — Siêu mượt, không bị đứt mạch)

Thay vì mỗi MF lại tạo lại dữ liệu từ đầu, hãy dùng **2 kịch bản nối tiếp nhau** để **mọi Swimlane đều ăn khớp tự nhiên**:

```text
[ĐƠN A — Kịch bản Online đa dịch vụ: Khám + Cận lâm sàng + Spa + Thanh toán PayOS/Tiền mặt]
  MF-01 (PetOwner đặt Online: Khám tổng quát + Tắm Spa → Lễ tân Check-in & gán chuồng Holding)
    └─► MF-03 (Bác sĩ mở hàng đợi → Khám SOAP + kê đơn thuốc)
         └─► MF-04 (Ngay trong ca khám đó: BS chỉ định Sub-task Xét nghiệm máu/X-quang → đồng nghiệp upload ảnh kết quả → BS hoàn tất SOAP!)
              └─► MF-05 (Dịch vụ #2 trong cùng Đơn A: Groomer nhận ca Spa → ảnh Before/After → Hoàn tất → Đơn A tự lên WaitOwner!)
                   └─► MF-07 (Lễ tân lập Hóa đơn tổng hợp: Dịch vụ + Sub-task có phí + Thuốc + Bán lẻ → Thanh toán PayOS VietQR/Tiền mặt → Xuất PDF!)

[ĐƠN B — Kịch bản Walk-in ca nặng → Nội trú → Xuất viện → Thanh toán]
  MF-02 (Khách vãng lai chưa có tài khoản đến quầy → Lễ tân tra SĐT → Tạo mới 1 chạm CheckedIn + cấp cứu isEmergency)
    └─► MF-06 (Bác sĩ khám thấy nặng → Nhập viện chuồng InpatientWard → Ghi Daily Log → Chủ nuôi xem từ xa → Xuất viện)
         └─► MF-07 (Gom tiền ngày nằm viện + thuốc → Thanh toán → Nhả chuồng nội trú)

[TỔNG KẾT]
  MF-08 (Admin mở Dashboard Doanh thu & Vận hành → Thấy NGAY doanh thu vừa tăng từ Đơn A & Đơn B + Top sản phẩm + Công suất chuồng!)
```

---

## 3. Mẹo thiết lập 5 cửa sổ trình duyệt (Không bao giờ mất thời gian đăng xuất/đăng nhập!)

Trước khi bước vào phòng bảo vệ, mở sẵn **5 Chrome Profile (hoặc 1 Chrome thường + 1 Incognito + Edge + Brave...)**, mỗi cửa sổ đăng nhập sẵn 1 vai trò và ghim sẵn tab tương ứng:

| Cửa sổ | Role | Route chính hay bấm nhất (`VCMOS_FE`) | Dùng ở Swimlane nào |
| :-: | :--- | :--- | :--- |
| **#1** | **`PetOwner`** | `/` (Home), `/dashboard?tab=appointments&book=new` (Đặt lịch), `/pet-journeys` (Timeline), `/invoices`, `/ai-assessment`, `/community` | `MF-01`, `MF-06`, `MF-07`, Bonus |
| **#2** | **`Receptionist`** | `/clinic/dashboard` (Tổng quan quầy), `/clinic/appointments` (Lịch hẹn & Walk-in), `/cages` (Bảng chuồng), `/clinic/invoices` (Thu ngân) | `MF-01`, `MF-02`, `MF-07` |
| **#3** | **`Veterinarian`** *(BS chính)* | `/clinic/my-queue` (Hàng đợi khám), `/clinic/exam/:id` (Bệnh án SOAP & Sub-task), `/clinic/inpatient` (Nội trú) | `MF-03`, `MF-04`, `MF-06` |
| **#4** | **`Groomer`** *(hoặc BS phụ)* | `/clinic/my-queue` (Hàng đợi Spa & Sub-task được giao) | `MF-04`, `MF-05` |
| **#5** | **`Admin`** | `/admin/reports` (Dashboard phân tích), `/admin/users`, `/admin/services`, `/admin/products`, `/cages`, `/staff-schedules`, `/admin/community-moderation` | `MF-08`, Bonus |

---

## 4. Bảng "Giáp Chống Xoáy": Những điểm chênh nhỏ giữa Ảnh Swimlane SRS vs Code thực tế

> **Cực kỳ quan trọng:** Thầy sẽ mở song song ảnh Swimlane trong `Report-3` và màn hình web của bạn. Học thuộc 5 điểm khác biệt nhỏ dưới đây để nếu thầy hỏi, bạn trả lời **tự tin và chủ động** (cho thấy bạn nắm sâu cả SRS lẫn Code!).

| Swimlane | Trên ảnh SRS ghi | Code thực tế triển khai | Cách giải thích "ăn điểm" với hội đồng |
| :--- | :--- | :--- | :--- |
| **`MF-01`** | Text ghi `service status = Assigned`; trên ảnh có nhánh `Staff busy? → Reassign Staff / Change time` | Khi đặt online, `AppointmentOrder.derivedStatus = 'Booked'`, `AppointmentService.status = 'Booked'` (lưu `preferredStaffId`). Khi Lễ tân Check-in & phân công nhân sự, BE tự kiểm tra `StaffSchedule` Blacklist 3 ca → nếu bận trả `409` buộc Lễ tân đổi nhân sự; phân công xong dịch vụ mới sang `Waiting` + `queueNo`. | *"Bọn em dùng `Booked` trước khi đến quầy và `Waiting` sau khi Check-in có số thứ tự `queueNo` để tách rõ khách chưa tới với hàng đợi thực tế của bác sĩ. Ngoài ra lúc đặt online bọn em chỉ lưu `reservedCageId` theo ngày chứ chưa khóa `Cage.status = Occupied` để chuồng hôm nay vẫn phục vụ khách khác."* |
| **`MF-03`** | Ảnh ghi `Record Subjective & Objective Findings` | Code khóa cứng **chữ S (`subjectiveNote`) chỉ đọc**, tự copy từ `order.intake.symptoms \|\| order.note` lúc `/start`; bác sĩ chỉ nhập **O, A, P** và đơn thuốc. | *"Theo phản hồi BA và tính pháp lý của bệnh án, triệu chứng chủ nuôi khai lúc tiếp nhận (S) phải giữ nguyên văn, bác sĩ chỉ ghi nhận khám lâm sàng (O), chẩn đoán (A) và phác đồ (P)."* |
| **`MF-04`** | Trên ảnh ghi `Notify assigned Groomer / Vet` và `Notify Attending Vet` | Hiện hệ thống hiển thị realtime qua hàng đợi `GET /api/subtask-requests/my-queue` và danh sách sub-task trên màn hình SOAP (Notification Center in-app `UC-69..71` để ở giai đoạn sau). | *"Nhân sự nhận việc ngay trên hàng đợi `my-queue` xếp theo FIFO và bác sĩ chính thấy kết quả kèm ảnh ngay trên màn hình SOAP. Chốt chặn `MSG26` ở BE ngăn bác sĩ hoàn tất bệnh án nếu còn sub-task chưa `Completed`."* |
| **`MF-06`** | Ô `Assign Available Inpatient Cage` nằm trên làn `Receptionist` | Trong code (`POST /api/inpatient-records`), chính **`Veterinarian`** chọn trực tiếp chuồng trống khu `InpatientWard` (BE tự kiểm tra tải trọng + tự nhả chuồng `Holding` cũ) để xử lý ca nặng nhanh trong 1 bước. | *"Khi thiết kế chi tiết và test thực tế, việc để bác sĩ đang khám ca nặng phải chờ lễ tân gán chuồng nội trú sẽ làm chậm cấp cứu. Nên bọn em cho bác sĩ chọn trực tiếp chuồng khu `InpatientWard` trống ngay lúc nhập viện, hệ thống tự nhả chuồng chờ `Holding` cũ bằng cơ chế atomic."* |
| **`MF-07`** | Ảnh chỉ vẽ `PayOS Gateway → Return Webhook Callback` | Ngoài `POST /api/invoices/payos-webhook` (cho môi trường có domain HTTPS), BE bổ sung thêm **`GET /api/invoices/:id/payos-status`** (polling trực tiếp PayOS API v2) để chạy mượt cả trên `localhost` khi demo! | *"Vì môi trường demo/localhost không nhận được public webhook callback từ PayOS, bọn em cài đặt song song cả Webhook ký HMAC-SHA256 (`timingSafeEqual`) lẫn Polling chủ động, cả hai đều đi qua 1 hàm chốt thanh toán `idempotent` nên không bao giờ xử lý trùng."* |

---

## 5. Checklist kiểm tra 5 phút trước khi bước lên bảo vệ

1. **Backend (`VCMOS_BE`):** Chạy `npm run dev` trên cổng `9999`, test `GET http://localhost:9999/` trả `200 { message: 'VCMOS API is running' }`. Kiểm tra `.env` đủ `MONGO_URI`, `CLOUDINARY_*`, `PAYOS_*`, `GEMINI_API_KEY_*` / `GROQ_API_KEY_*`, `EMAIL_USER/PASS`.
2. **Frontend (`VCMOS_FE`):** Chạy `npm start` trên cổng `3000`, đăng nhập sẵn 5 cửa sổ cho 5 vai trò.
3. **Dữ liệu sẵn sàng cho Demo (tuyệt đối KHÔNG chạy `seed --force` lên Atlas chung!):**
   - Ít nhất **2 chuồng `Holding` đang `Available`** (cỡ M/L) và **1 chuồng `InpatientWard` đang `Available`** (cho `MF-06`).
   - Ít nhất **1 dịch vụ `Examination`** (`Khám tổng quát`), **1 dịch vụ `Paraclinical`** (có giá, dùng cho Sub-task ở `MF-04`), **1 dịch vụ `Grooming`** (`Tắm vệ sinh`, cho `MF-05`), và **2 sản phẩm `Active`** (1 thuốc kê đơn + 1 sản phẩm bán lẻ).
   - **1 Pet `Active`** chưa có lịch hẹn nào trong hôm nay (vì hệ thống chặn 1 pet đặt 2 đơn chưa hủy cùng ngày!).
   - **1 ảnh thú cưng / X-quang / ảnh Spa nhỏ (`< 1MB`, đuôi `.jpg`/`.png`)** để sẵn ngoài Desktop để bấm upload nhanh ở `MF-04`, `MF-05` và `AI Health Assessment`.
   - **Tài khoản ngân hàng / ví QR nhỏ (vd 2.000đ–5.000đ)** nếu muốn demo quét **PayOS VietQR thật** ở `MF-07` (nhớ tạo dịch vụ test hoặc giá nhỏ nếu muốn quét thật, còn không thì demo tạo QR thật + chuyển sang Tiền mặt nếu cần nhanh).
