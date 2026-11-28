# MF-01 — Đặt Lịch Online, Xác Nhận & Check-in Quầy (`SRS 2.1`, trang 8–9)

> **Tóm tắt nói trước hội đồng (20 giây):** *"Luồng đầu tiên `MF-01` là đặt lịch khám & spa trực tuyến của Chủ nuôi (`PetOwner`) và tiếp nhận Check-in tại quầy của Lễ tân (`Receptionist`). Điểm nổi bật ở luồng này là khách đặt 1 chạm nhiều dịch vụ nối tiếp trong khung giờ `08:00–21:00`, hệ thống tự giữ chuồng dự kiến theo ngày và gửi email BCC báo Lễ tân mà không cần chờ duyệt tay; khi khách đến quầy, Lễ tân Check-in đồng bộ 3 việc trong 1 Transaction: đổi đơn sang `CheckedIn`, phân công bác sĩ/groomer không bị bận và khóa chuồng vật lý sang `Occupied`."*

---

## 1. Tái hiện đúng sơ đồ Swimlane `MF-01` trong `Report-3 SRS`

```text
┌──────────────────┐
│   Pet Owner      │  (●)─►[Select Pet & Services]─►[Select Date & Preferred Staff]─┐
│                  │          ┌─◄───────────────────────────────────────────────────┘
│                  │          │   [Pick Slot & Enter Symptom Note]─►[Click Submit Booking]──┐
│                  │          │                                                             │
│                  │          │                              ┌──►[Bring Pet to Clinic]──┐   │
├──────────────────┤          ▼                              │                          │   ▼
│   System         │  [Display Available Slots]──────────────┘      [Save Order]        │   │
│                  │                                                     │              │   │
│                  │                                             [Notify Receptionist]──┘   │
│                  │                                                                        │
│                  │                      ┌───►[Update Order = Checked In]                  │
│                  │                      │              │                                  │
│                  │                      │    [Set Cage = Occupied]──►(◉)                  │
├──────────────────┤                      │                                             ▲   │
│   Receptionist   │                      │    [Reassign Staff / Change time]           │   │
│                  │                      │           ▲              │                  │   │
│                  │                      │        [yes]             ▼                  │   │
│                  │                      │   ┌──►<Staff busy?>─[no]─►[Click Check-in]  │   │
│                  │                      │   │                              │          │   │
│                  │                      └───┼─────────────────[Assign Pet to Cage]◄───┘   │
└──────────────────┘                          └─────────────────────────────────────────────┘
```

- **Trigger:** Chủ nuôi khởi tạo yêu cầu đặt lịch online cho thú cưng.
- **End condition:** Đơn tạo ngay ở trạng thái `Booked` (không cần lễ tân bấm duyệt). Khi thú cưng đến quầy, Lễ tân Check-in → đơn sang `CheckedIn`, phân công nhân sự (`Waiting` + `queueNo`), và gán chuồng giữ thú cưng sang `Occupied`.

---

## 2. Kịch bản Demo từng bước (Bấm gì — Nói gì — BE & DB làm gì)

| Bước | Làn (Swimlane) | Màn hình FE (`SCR` / URL) | Thao tác thực tế khi demo & Câu dẫn | API BE & Code (`file:line`) | Thay đổi trong Database |
| :-: | :--- | :--- | :--- | :--- | :--- |
| **1** | **Pet Owner** | `SCR-07` / `SCR-12` <br>`/dashboard?tab=appointments&book=new` | Mở **Cửa sổ #1 (PetOwner)**. Chọn thú cưng (vd: `Milo`), chọn **2 dịch vụ nối tiếp**: `Khám tổng quát` + `Tắm vệ sinh Spa` (để nối mượt sang `MF-03`, `MF-04`, `MF-05`). | `GET /api/pets`, `GET /api/services` | Đọc thú cưng `Active` & dịch vụ `Active`. |
| **2** | **Pet Owner → System** | `SCR-12` Booking Wizard | Chọn **Ngày hôm nay**, chọn **khung giờ hợp lệ** (từ giờ hiện tại đến trước `21:00`), chọn nhân sự mong muốn (nếu có), nhập ghi chú triệu chứng *"Bé hơi mệt, bỏ ăn 1 ngày"* rồi bấm **Đặt lịch**. | `POST /api/appointment-services/online-booking` <br>(`appointmentService.service.js:107-202` + `appointmentOrder.service.js:352-413`) | 1. Tính `scheduledStart`/`scheduledEnd` nối tiếp theo `defaultDuration`, chặn vượt `21:00`.<br>2. Chống đặt trùng 2 đơn chưa hủy cùng ngày (`409`).<br>3. Tạo `AppointmentOrder` (`APT-XXXXXX`, `derivedStatus: 'Booked'`, `reservedCageId`).<br>4. Tạo tuần tự 2 `AppointmentService` (`sequenceNo: 1, 2`, `status: 'Booked'`). |
| **3** | **System** | *(Chạy ngầm — giải thích cho thầy)* | Nói rõ: *"Hệ thống tự tìm 1 chuồng khu `Holding` đạt tải trọng `BR-11` để lưu vào `reservedCageId` theo ngày (chưa khóa `Occupied`), đồng thời gửi email BCC báo Lễ tân qua Nodemailer."* | `findAvailableReservedCageId` (`appointmentOrder.service.js:156-221`), `notifyReceptionistsNewOnlineBooking` (`L123-150`) | `Cage.status` **vẫn giữ `Available`** (chỉ giữ chỗ theo ngày). Email gửi fire-and-forget. |
| **4** | **Pet Owner** | `SCR-13` <br>`/dashboard?tab=appointments` | Cho hội đồng thấy đơn mới tạo hiện ngay trạng thái **`Booked`** (`Đã đặt`) kèm nút **Hủy lịch** (nói thêm: *"Khách có thể tự hủy trước ≥ 2 tiếng theo `BR-07`"*). | `GET /api/appointment-orders/me` | Đọc batch `$in` tránh N+1 (`enrichOrdersWithCageAndServices`). |
| **5** | **Receptionist** | `SCR-17` <br>`/clinic/appointments` | Chuyển sang **Cửa sổ #2 (Receptionist)**. Mở danh sách lịch hẹn hôm nay → thấy ngay đơn online `APT-XXXXXX` của bé `Milo`. | `GET /api/appointment-orders?from=...&to=...` | Hiển thị đủ thông tin đơn, dịch vụ, chuồng dự kiến. |
| **6** | **Receptionist (`Staff busy?`)** | `SCR-17` Modal Check-in | Bấm **Check-in** trên đơn. Nhập phiếu tiếp nhận ban đầu `intake` (`symptoms` + `initialCondition`). Chọn **Bác sĩ** cho dịch vụ #1, **Groomer** cho dịch vụ #2, và xác nhận **Chuồng `Holding`**. *(Nhấn mạnh: nếu chọn trúng nhân sự đã báo bận ca này trên `StaffSchedule`, BE tự chặn `409` buộc chọn người khác!)* | `PATCH /api/appointment-orders/:id/check-in` <br>(`appointmentOrder.service.js:684-947`) | **Chạy trong `runWithTransaction`:**<br>1. Atomic `Booked → CheckedIn` + lưu `checkedInAt`, `intake`.<br>2. Gán `assignedStaffId`, cấp `queueNo = max + 1`, đẩy dịch vụ sang `Waiting`.<br>3. Tạo `CageAssignment` (`releasedAt: null`) & đổi `Cage.status: Available → Occupied`. |
| **7** | **System → Receptionist** | `SCR-19` <br>`/cages` | Mở nhanh tab **Bảng chuồng (`/cages`)** cho thầy thấy chuồng vừa gán đã chuyển sang màu **`Occupied` (Đang sử dụng)** realtime! | `GET /api/cages` | Khẳng định `End condition` của `MF-01` đã hoàn tất 100%. |

---

## 3. Các chốt chặn nghiệp vụ & kỹ thuật nên nhấn mạnh (Để ăn điểm thiết kế)

1. **Chống khóa chuồng sớm (`reservedCageId` vs `Cage.status`):** Đặt cho tuần sau chỉ giữ chỗ theo `date` trên `AppointmentOrder`, đến lúc Check-in thật mới khóa `Cage.status = 'Occupied'`.
2. **Chống 2 Lễ tân bấm Check-in trùng 1 đơn:** `AppointmentOrder.findOneAndUpdate({ _id, derivedStatus: 'Booked' }, ...)` (`appointmentOrder.service.js:771-790`) — người thứ 2 nhận ngay **`409 Conflict`**.
3. **Chống 2 Lễ tân gán cùng 1 chuồng:** Atomic `Cage.findOneAndUpdate({ _id, status: 'Available' })` + Partial Unique Index `{ cageId: 1 }` nơi `releasedAt: null` (`CageAssignment.js`).
4. **Kiểm tra khung giờ mở cửa thống nhất `08:00–21:00`:** Khớp hoàn toàn với 3 ca `MORNING` (`08:00–12:00`), `AFTERNOON` (`12:00–17:00`), `EVENING` (`17:00–21:00`) của `StaffSchedule`.
5. **Quy tắc hủy `BR-07`:** Chủ nuôi chỉ hủy khi còn `Booked` và trước `slotStart` ≥ 2 tiếng; Lễ tân hủy được cả khi đã `CheckedIn` (chưa có dịch vụ `InProgress`/`Done`) và BE tự động nhả chuồng về `Available`.

---

## 4. Câu hỏi hội đồng hay xoáy ở `MF-01`

1. **Hỏi: Nếu khách đặt 2 dịch vụ (Khám 30p + Spa 60p) lúc 20:00 thì hệ thống có cho đặt không?**
   - **Đáp:** Không! `appointmentService.service.js:117-159` cộng dồn `defaultDuration` nối tiếp nhau (`20:00 → 20:30 → 21:30`), phát hiện vượt quá giờ đóng cửa `21:00` (`CLINIC_CLOSE_TIME`) và chặn ngay với **`400 Bad Request`**.
2. **Hỏi: Nếu khách đặt lịch rồi không đến thì đơn có treo `Booked` mãi không?**
   - **Đáp:** Không! Cuối ngày lúc `23:59` (giờ VN), Background Job `JOB-01` (`flagOverdueAppointments.job.js`) tự quét và đánh dấu đơn + dịch vụ đầu tiên sang `Overdue` (`BR-08`), đồng thời chạy quét bù khi server khởi động lại nếu lỡ tắt máy qua đêm.
3. **Hỏi: Nếu đang Check-in mà bước gán chuồng bị lỗi (ví dụ ai đó vừa lấy mất chuồng) thì đơn có bị treo ở `CheckedIn` mà không có chuồng không?**
   - **Đáp:** Không! Toàn bộ pha ghi nằm trong `runWithTransaction` (`appointmentOrder.service.js:767-935`): trên MongoDB Atlas sẽ abort cả transaction; trên local standalone có khối rollback thủ công hoàn tác `AppointmentOrder` về `Booked` và khôi phục trạng thái từng `AppointmentService`.
