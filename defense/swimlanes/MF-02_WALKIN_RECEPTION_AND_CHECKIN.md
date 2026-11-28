# MF-02 — Tiếp Nhận Khách Vãng Lai Tại Quầy & Check-in (`SRS 2.2`, trang 10)

> **Tóm tắt nói trước hội đồng (20 giây):** *"Luồng `MF-02` xử lý khách đến trực tiếp quầy không đặt trước (Walk-in). Lễ tân chỉ cần nhập SĐT để tra cứu nhanh: nếu là khách cũ thì chọn ngay thú cưng có sẵn; nếu là khách hoàn toàn mới thì hệ thống tự cấp 1 tài khoản tạm ở trạng thái `Inactive` (chưa có mật khẩu theo `BR-04`), tạo hồ sơ thú cưng, tạo đơn ở trạng thái `CheckedIn` ngay lập tức, phân công nhân sự (có thể đánh dấu cấp cứu `isEmergency`) và gán chuồng — ghi đồng bộ tối đa **6 collection** trong 1 Transaction duy nhất."*

---

## 1. Tái hiện đúng sơ đồ Swimlane `MF-02` trong `Report-3 SRS`

```text
┌──────────────────┐
│   Receptionist   │  (●)─►[Check immediate Vet/Groomer availability]
│                  │                        │
│                  │               <Any staff free?>─[no]─►[Advise customer to wait]──┐
│                  │                     [yes]                                        │
│                  │                       ▼                                          │
│                  │             [Enter customer phone]                               │
│                  │                       │        ┌─[Select existing pet]──►[Select services]
│                  │                       │        │           ▲                     │
│                  │                       │        │         [yes]             [Assign staff]
│                  │                       │        │           │                     │
│                  │                       │        │      <Pet exist?>        [Pick available cage]
│                  │                       │        │           │   [no]              │
│                  │                       │        │           │     ▼         [Submit walk-in check-in]
│                  │                       │        │           │ [Register new pet]  │
│                  │                       │        │           │         ▲           ▼
│                  │                       │        │           │         │    <Account exists?>
├──────────────────┤                       ▼        │           │         │     [no]       [yes]
│   System         │          [Search account by phone]         │         │      ▼           │
│                  │                       │                    │         │ [Create inactive │
│                  │                       ▼                    │         │   account]       │
│                  │               <Account exists?>──[yes]─────┘         │      │           ▼
│                  │                       │                              │      └──►[Create appointment]
│                  │                      [no]────────────────────────────┘                  │
│                  │                                                                 [Set cage status]
│                  │                                                                         │
│                  │  ◄──────────────────────────────────────────────────────────────────────┴──►(◉)
└──────────────────┘
```

- **Trigger:** Khách đưa thú cưng đến trực tiếp quầy lễ tân không có lịch hẹn trước.
- **End condition:** Đã kiểm tra nhân sự rảnh, định danh khách (hoặc tạo tài khoản `Inactive` chờ kích hoạt), tạo đơn trực tiếp trạng thái **`CheckedIn`**, đẩy dịch vụ vào hàng đợi `Waiting`, và gán chuồng `Holding` sang **`Occupied`**.

---

## 2. Kịch bản Demo từng bước (Bấm gì — Nói gì — BE & DB làm gì)

| Bước | Làn (Swimlane) | Màn hình FE (`SCR` / URL) | Thao tác thực tế khi demo & Câu dẫn | API BE & Code (`file:line`) | Thay đổi trong Database |
| :-: | :--- | :--- | :--- | :--- | :--- |
| **1** | **Receptionist** | `SCR-17` / `SCR-18` <br>`/clinic/appointments?walkin=new` | Trên **Cửa sổ #2 (Receptionist)**, xem nhanh lịch trực/hàng đợi nhân sự hôm nay rồi bấm nút **Tiếp nhận vãng lai (Walk-in)**. | `GET /api/users/staff`, `GET /api/staff-schedules` | Kiểm tra nhanh Bác sĩ/Groomer đang rảnh ca hiện tại. |
| **2** | **Receptionist → System** | `SCR-18` Walk-in Modal | Nhập **1 SĐT hoàn toàn mới** (vd: `0987654321`) để demo trọn vẹn nhánh **"Khách mới + Thú cưng mới"** (nhánh phức tạp nhất của sơ đồ!). | `GET /api/appointment-orders/walk-in/customer-lookup?phone=0987654321` <br>(`walkIn.service.js:148-176`) | Trả **`200`** `{ found: false, customer: null, pets: [] }` (cố ý không trả `404` để UI bật form nhập mới mượt mà, không hiện lỗi đỏ). |
| **3** | **Receptionist** | `SCR-18` Walk-in Modal | Nhập tên khách (vd: *Lê Minh Tuấn*), thông tin bé thú cưng mới (vd: *Bông*, mèo 4kg), chọn dịch vụ **`Khám tổng quát`**, tích **Cấp cứu (`isEmergency = true`)** *(để sang `MF-03` thấy bé nhảy lên đầu hàng đợi!)*, chọn Bác sĩ, nhập triệu chứng ban đầu `intake` (*"Nôn ói liên tục, mệt lả"*), chọn 1 chuồng `Holding` trống → bấm **Xác nhận Check-in**. | `POST /api/appointment-orders/walk-in` <br>(`walkIn.service.js:188-452`) | **Pha 1:** Validate toàn bộ dịch vụ `Active`, nhân sự không bận `StaffSchedule`, chuồng `Available` đạt `BR-11`.<br>**Pha 2 (`runWithTransaction` ghi đồng bộ 6 bảng):**<br>1. `User`: `USR-XXXXXX`, `role: 'PetOwner'`, **`status: 'Inactive'`**, **`passwordHash: null`** (`BR-04`).<br>2. `Pet`: `PET-XXXXXX`, `Active`.<br>3. `AppointmentOrder`: `APT-XXXXXX`, `isOnline: false`, **`derivedStatus: 'CheckedIn'`**, `intake`.<br>4. `AppointmentService`: `status: 'Waiting'`, `isEmergency: true`, `queueNo`.<br>5. `CageAssignment`: `releasedAt: null`.<br>6. `Cage`: `Available → Occupied`. |
| **4** | **System → Receptionist** | `SCR-17` / `SCR-19` | Cho hội đồng thấy đơn vãng lai xuất hiện ngay ở trạng thái **`CheckedIn`** (có gắn nhãn **Cấp cứu**), và bên `/cages` chuồng đã chuyển sang **`Occupied`**. | `GET /api/appointment-orders`, `GET /api/cages` | Hoàn tất `MF-02` và sẵn sàng nối sang `MF-03` / `MF-06`! |

---

## 3. Các chốt chặn nghiệp vụ & kỹ thuật nên nhấn mạnh

1. **Chiến lược 2 pha (Validate trước — Ghi sau trong Transaction):** Toàn bộ bước kiểm tra 5 điều kiện (dịch vụ, nhân sự rảnh, chuồng trống, tài khoản không `Locked`, tải trọng `BR-11`) chạy ở Pha 1 (`L203-342`) trước khi ghi bất kỳ dòng nào vào DB.
2. **Bảo vệ dữ liệu sạch (Transaction + Rollback thủ công):** Nếu bước thứ 5 (gán chuồng) đụng race condition với lễ tân khác, toàn bộ `User`, `Pet`, `AppointmentOrder`, `AppointmentService` vừa tạo ở bước 1–4 sẽ bị rollback sạch (`walkIn.service.js:345-451`) — **không bao giờ sinh tài khoản rác**.
3. **Chống nhập nhầm SĐT nhân sự / tài khoản bị khóa:** Nếu gõ SĐT của Bác sĩ/Admin → chặn **`409`**; nếu gõ SĐT khách đang `Locked` → chặn **`403`** (`walkIn.service.js:159-165, 285-290`).

---

## 4. Câu hỏi hội đồng hay xoáy ở `MF-02`

1. **Hỏi: Tại sao tài khoản khách vãng lai lại đặt `status: 'Inactive'` và `passwordHash: null`? Sau này khách muốn đăng nhập xem hồ sơ thì sao?**
   - **Đáp:** Vì khách đứng tại quầy chỉ đọc tên + SĐT chứ không đặt mật khẩu nên hệ thống không cho đăng nhập bừa bằng tài khoản tạm (`BR-04`). Nếu ai đó cố dùng SĐT đó để đăng nhập sẽ bị chặn `403`, và cố dùng SĐT đó tự đăng ký mới trên web sẽ bị chặn `409 Conflict` để chống người lạ chiếm đoạt hồ sơ bệnh án thú cưng khi chưa xác thực SMS OTP.
2. **Hỏi: Nhánh `[no]` của `<Any staff free?>` (không còn nhân sự rảnh) được xử lý thế nào trong hệ thống?**
   - **Đáp:** Lễ tân xem lịch bận (`StaffSchedule`) và hàng đợi trên `SCR-17`. Nếu vẫn muốn tiếp nhận cho khách ngồi chờ, `POST /api/appointment-orders/walk-in` cho phép không truyền `assignedStaffId` — đơn vẫn tạo `CheckedIn`, dịch vụ nằm ở trạng thái chờ phân công để Lễ tân gọi `PATCH /api/appointment-services/:id/assign-staff` ngay khi có bác sĩ trống.
3. **Hỏi: Tại sao `GET /customer-lookup` không tìm thấy khách hàng lại trả `200 { found: false }` thay vì `404`?**
   - **Đáp:** Khách mới đến lần đầu là nghiệp vụ hoàn toàn bình thường, không phải lỗi hệ thống. Trả `200` kèm `found: false` giúp FE nhận diện gọn gàng để mở form đăng ký khách mới mà không báo lỗi đỏ trên Console/Toast.
