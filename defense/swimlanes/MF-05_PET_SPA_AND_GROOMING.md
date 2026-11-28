# MF-05 — Dịch Vụ Spa & Grooming Thú Cưng (`SRS 2.5`, trang 15–16)

> **Tóm tắt nói trước hội đồng (20 giây):** *"Luồng `MF-05` là quy trình chăm sóc Spa & Grooming cho `Groomer`. Ngay sau khi dịch vụ Khám ở `MF-03` hoàn tất (đạt thứ tự `BR-12`), Groomer mở hàng đợi Spa, bấm Bắt đầu thì chuồng chờ tự động mở về `Available` (`BR-10`). Sau khi tắm cắt xong, Groomer tải song song 2 ảnh **Before & After** lên Cloudinary (có cơ chế tự dọn ảnh nếu upload hoặc ghi DB lỗi), ghi nhận xét chăm sóc lưu thẳng vào Timeline của bé, rồi bấm Hoàn tất. Vì đây là dịch vụ cuối cùng trong `Đơn A`, hệ thống tự động đẩy `AppointmentOrder` sang trạng thái **`WaitOwner` (Chờ chủ đón & thanh toán)** — sẵn sàng chuyển sang `MF-07`!"*

---

## 1. Tái hiện đúng sơ đồ Swimlane `MF-05` trong `Report-3 SRS`

```text
┌─────────────────────────────────────────────┬─────────────────────────────────────────────┐
│                   Groomer                   │                   System                    │
├─────────────────────────────────────────────┼─────────────────────────────────────────────┤
│                    (●)                      │                                             │
│                     ▼                       │                                             │
│      [Open Spa Queue & Review Requests]─────┼────►[Set Service = In Progress]             │
│                     ▼                       │                  │                          │
│            [Click Start Service]            │                  ▼                          │
│                     │                       │     [Set Cage status = Available]           │
│       ┌─────────────┘                       │                  │                          │
│       ▼                                     │                  │                          │
│ [Perform Service]──►[Fill Outcome Notes]    │                  │                          │
│                              │              │                  │                          │
│       ┌──────────────────────┘              │                  │                          │
│       ▼                                     │                  │                          │
│ [Add Before & After]─►[Click Complete]──────┼────►[Save Notes to Timeline]                │
│                                             │                  │                          │
│                                             │                  ▼                          │
│                                             │          [Set Service = Done]               │
│                                             │                  │                          │
│                     ┌───────────────────────┼──────────────────┘                          │
│                     ▼                       │                                             │
│     [Place Pet in an available Cage]────────┼────►[Set Cage status = Occupied]            │
│                                             │                  │                          │
│                                             │                  ▼                          │
│                                             │       <More Services in Order?>             │
│                                             │          [yes]             [no]             │
│                                             │            │                ▼               │
│                                             │            │     [Set Order = Wait Owner]   │
│                                             │            │                │               │
│                                             │            └──────►(◉)◄─────┘               │
└─────────────────────────────────────────────┴─────────────────────────────────────────────┘
```

- **Trigger:** Lịch hẹn đã `CheckedIn` có dịch vụ Spa/Grooming đến lượt thực hiện (dịch vụ đứng trước đã `Done`/`Cancelled` theo `BR-12`).
- **End condition:** Groomer nhận thú cưng (chuồng cũ tự về `Available`), thực hiện Spa, tải ảnh Before/After + ghi chú vào Timeline, đặt bé lại vào chuồng chờ đón (`Occupied`), đánh dấu dịch vụ `Done` và đẩy đơn sang **`WaitOwner`** nếu hết dịch vụ.

---

## 2. Kịch bản Demo từng bước (Bấm gì — Nói gì — BE & DB làm gì)

| Bước | Làn (Swimlane) | Màn hình FE (`SCR` / URL) | Thao tác thực tế khi demo & Câu dẫn | API BE & Code (`file:line`) | Thay đổi trong Database |
| :-: | :--- | :--- | :--- | :--- | :--- |
| **1** | **Groomer** | `SCR-26` <br>`/clinic/my-queue` | Mở **Cửa sổ #4 (Groomer)**. Chỉ cho hội đồng thấy dịch vụ #2 **`Tắm vệ sinh Spa`** của bé `Milo` (`Đơn A`) đang hiện sẵn trong hàng đợi. | `GET /api/appointment-services/my-queue` <br>(`appointmentService.service.js:283-299`) | Sắp xếp ưu tiên `{ isEmergency: -1, queueNo: 1, enqueueAt: 1 }`. |
| **2** | **Groomer → System** | `SCR-26` → `SCR-27` | Bấm **Bắt đầu làm Spa**. Nhấn mạnh với thầy: *"Lúc này chuồng chờ sau khám ở `MF-03` tự động giải phóng về `Available` (`BR-10`), còn trạng thái real-time trên Timeline của chủ nuôi tự đổi thành `Đang Tắm vệ sinh Spa` (`BR-09`)!"* | `PATCH /api/appointment-services/:id/start` <br>(`appointmentService.service.js:515-525`) | 1. Kiểm tra `BR-12` (dịch vụ #1 Khám đã `Done`).<br>2. Đóng `CageAssignment` mở & trả `Cage.status = 'Available'`.<br>3. `AppointmentService: Waiting → InProgress` (`dequeueAt`). |
| **3** | **Groomer → System** | `SCR-27` Form Grooming Notes | Nhập ghi chú trước & sau tắm (*"Lông hơi rối ở tai → Đã gỡ rối, sấy bông, cắt móng gọn"*), khuyến nghị (*"Chải lông 2 ngày/lần"*), chọn **2 ảnh Before & After** từ Desktop → bấm **Lưu ghi chú Grooming**. | `POST /api/appointment-services/:id/grooming-note` <br>(`uploadGroomingNotePhotos` + `service.js:562-615`) | 1. Upload song song 2 ảnh lên Cloudinary bằng `Promise.allSettled`.<br>2. Nếu 1 trong 2 ảnh lỗi → tự xóa ảnh còn lại & trả `502`.<br>3. Atomic `findOneAndUpdate({ _id, assignedStaffId, groomingNote: null })` chống ghi đúp (trùng → tự xóa ảnh & trả `409`). |
| **4** | **Groomer → System** | `SCR-27` / `SCR-19` | Bấm **Hoàn tất dịch vụ Spa** (và đặt bé vào 1 chuồng chờ đón trống nếu cần, hoặc gán qua Bảng chuồng `/cages`). | `PATCH /api/appointment-services/:id/complete` <br>(`appointmentService.service.js:453-510`) + `POST /api/cage-assignments` | 1. `AppointmentService: InProgress → Done`.<br>2. Đếm số dịch vụ chưa `Done`/`Cancelled` còn lại trong `Đơn A` → bằng `0` (`More Services? = [no]`) → **tự động cập nhật `AppointmentOrder.derivedStatus = 'WaitOwner'`!**<br>3. Chuồng chờ đón chuyển `Occupied`. |

---

## 3. Các chốt chặn nghiệp vụ & kỹ thuật nên nhấn mạnh

1. **Tải ảnh an toàn tuyệt đối (không sinh ảnh mồ côi trên Cloudinary):** Dùng `Promise.allSettled` upload song song 2 ảnh; nếu 1 ảnh fail hoặc 2 request bấm đúp cùng lúc (`groomingNote: null`), BE tự gọi `deleteCloudinaryImages` dọn sạch (`appointmentService.service.js:572-615`).
2. **Ảnh Before/After tự động tái sử dụng cho 2 tính năng khác:**
   - Tự xuất hiện trên **Unified Pet Timeline** (`SCR-08`) của Chủ nuôi.
   - Là nguồn ảnh chính khi Chủ nuôi bấm **Chia sẻ hành trình lên Cộng đồng** (`BR-18` — không lộ bệnh án SOAP và không lộ giá tiền!).
3. **Tự động tính trạng thái đơn cha (`BR-09`):** Groomer không cần tự sửa trạng thái đơn; BE tự đếm các dịch vụ còn lại để chỉ đẩy `AppointmentOrder → WaitOwner` khi **tất cả** dịch vụ không bị hủy đã `Done`.

---

## 4. Câu hỏi hội đồng hay xoáy ở `MF-05`

1. **Hỏi: Groomer có thể vào xem bệnh án SOAP hay đơn thuốc hoặc hóa đơn tiền của thú cưng không?**
   - **Đáp:** Groomer xem được tiền sử chăm sóc/y tế cơ bản trên Timeline để nắm tình trạng da/dị ứng của thú cưng trước khi tắm, nhưng **không thể sửa SOAP** (`403` — chỉ `Veterinarian`), và **`invoice` luôn bị ẩn (`null`)**, **`aiAssessments` luôn là `[]`** (`petTimeline.service.js:16, 155, 158`).
2. **Hỏi: Nếu Groomer bấm lưu Grooming Note 2 lần liên tiếp (double-click mạng lag) thì sao?**
   - **Đáp:** Câu lệnh cập nhật nguyên tử có điều kiện `groomingNote: null` (`L595-612`): lần 1 ghi thành công trả `201`; lần 2 nhận `null`, tự xóa ảnh vừa tải lên Cloudinary rồi trả `409 Conflict`.
3. **Hỏi: Nếu đơn ban đầu có 2 dịch vụ (Khám + Spa), khách khám xong đổi ý hủy dịch vụ Spa thì đơn có kẹt mãi ở `InProgress` không?**
   - **Đáp:** Không! Trong `cancelAppointmentService` (`appointmentService.service.js:424-431`), ngay khi Lễ tân hủy dịch vụ Spa còn lại, hệ thống kiểm tra thấy dịch vụ Khám kia đã `Done` nên **tự đẩy đơn cha sang `WaitOwner`** để Lễ tân lập hóa đơn được ngay.
