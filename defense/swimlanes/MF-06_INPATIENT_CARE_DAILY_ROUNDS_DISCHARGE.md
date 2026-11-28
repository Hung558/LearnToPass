# MF-06 — Điều Trị Nội Trú, Nhật Ký Hằng Ngày & Xuất Viện (`SRS 2.6`, trang 17)

> **Tóm tắt nói trước hội đồng (20 giây):** *"Luồng `MF-06` dành cho ca bệnh nặng từ `MF-02`/`MF-03` cần lưu viện nhiều ngày. Bác sĩ tạo hồ sơ nhập viện vào chuồng chuyên dụng khu `InpatientWard` (hệ thống tự động trả chuồng chờ `Holding` cũ về `Available` và chặn nhập viện trùng bằng Partial Unique Index, đồng thời gửi email báo chủ). Mỗi ngày Bác sĩ cập nhật **Daily Log**, Chủ nuôi ở nhà mở app xem được tiến trình phục hồi từ xa (bảo vệ IDOR trả `404`). Khi thú cưng khỏe hẳn, Bác sĩ bấm Xuất viện (`Discharged`) và gửi email báo đón — nhưng **giữ nguyên chuồng nội trú đang `Occupied`** cho tới khi chủ đến quầy thanh toán ở `MF-07` mới mở chuồng!"*

---

## 1. Tái hiện đúng sơ đồ Swimlane `MF-06` trong `Report-3 SRS`

```text
┌───────────────────┐
│   Veterinarian    │  (●)─►[Create Inpatient Admission]  ┌───[No]────────────────────┐
│                   │                 │                   ▼                           │
│                   │                 │       [Conduct Daily Rounds]─►[Record Daily Log]─►<Pet Fully Recovered?>─[Yes]─►[Issue Discharge Order]
├───────────────────┤                 ▼                   ▲                           │                                          │
│   Receptionist    │   [Assign Available Inpatient Cage] │                           │                                          │
├───────────────────┤                 │                   │                           │                                          ▼
│   System          │                 ▼                   │         [Append Daily Log to Timeline]                  [Set Inpatient = Discharged]
│                   │   [Create InpatientRecord &         │                   │                                                  │
│                   │    Set Cage = Occupied]             │                   │                                                  ▼
│                   │                 │                   │                   │                                    [Set Order = Wait Owner & Count Days]
├───────────────────┤                 ▼                   │                   ▼                                                  │
│   Pet Owner       │   [Receive Admission Notification]──┘     [View Daily Progress Remotely]                     [Receive Pick-up Notification]─►(◉)
└───────────────────┘
```

- **Trigger:** Thú cưng khám ở `MF-03`/`MF-04` (hoặc ca cấp cứu `MF-02` — `Đơn B` của bé *Bông*) cần nhập viện điều trị nhiều ngày.
- **End condition:** Hồ sơ nội trú ở trạng thái `Discharged`, có đầy đủ lịch sử `dailyLogs`, gửi thông báo đón bé; số ngày nội trú được tính tự động (`max(1, ceil((dischargedAt - admittedAt) / 1 ngày))`) khi lập hóa đơn ở `MF-07`.

---

## 2. Kịch bản Demo từng bước (Bấm gì — Nói gì — BE & DB làm gì)

| Bước | Làn (Swimlane) | Màn hình FE (`SCR` / URL) | Thao tác thực tế khi demo & Câu dẫn | API BE & Code (`file:line`) | Thay đổi trong Database |
| :-: | :--- | :--- | :--- | :--- | :--- |
| **1** | **Veterinarian → System** | `SCR-25` <br>`/clinic/inpatient` | Mở **Cửa sổ #3 (Veterinarian)**, dùng ca cấp cứu `Đơn B` (bé *Bông* từ `MF-02`). Bấm **Cho nhập viện** → chọn 1 chuồng trống khu **`InpatientWard`** (vd: `W-01`), nhập chẩn đoán ban đầu (*"Viêm dạ dày cấp mất nước"*) và phác đồ (*"Truyền dịch + kháng sinh tĩnh mạch 2 ngày"*) → bấm **Xác nhận nhập viện**. | `POST /api/inpatient-records` <br>(`inpatientRecord.service.js:156-236`) | 1. Chặn nhập viện trùng (`Admitted` → `409` + Partial Unique Index `{ petId }` nơi `status: 'Admitted'`).<br>2. Ép chuồng phải thuộc **`zone === 'InpatientWard'`** (`400`) & đạt `maxWeightKg`.<br>3. Atomic `Cage.findOneAndUpdate({ _id, status: 'Available' } → 'Occupied')`.<br>4. **Tự đóng `CageAssignment` chuồng `Holding` cũ & trả về `Available`!**<br>5. Tạo `InpatientRecord` (`status: 'Admitted'`) + gửi email báo chủ (nếu đã verify email). |
| **2** | **Veterinarian → System** | `SCR-25` Chi tiết nội trú | Mở hồ sơ vừa tạo → bấm **Ghi nhật ký hằng ngày (Daily Log)** → chọn ngày hôm nay, nhập tình trạng (*"Nhiệt độ giảm còn 38.6°C, hết nôn, uống được nước"*) và ghi chú (*"Tiếp tục theo dõi đến chiều"*) → bấm **Lưu nhật ký**. | `POST /api/inpatient-records/:id/daily-logs` <br>(`inpatientRecord.service.js:274-310`) | 1. Validate ngày không ở tương lai & không trước `admittedAt`.<br>2. Atomic `findOneAndUpdate({ _id, status: 'Admitted' }, { $push: { dailyLogs } })` chống race condition với lệnh xuất viện. |
| **3** | **Pet Owner** | `SCR-11` / `SCR-08` <br>`/pet-journeys` *(hoặc Dashboard)* | *(Nếu demo trên bé thuộc tài khoản `PetOwner` ở Cửa sổ #1)* Mở **Cửa sổ #1 (PetOwner)** → cho thầy thấy chủ nuôi xem được ngay nhật ký điều trị hôm nay của bé từ xa mà **chỉ hiện tên bác sĩ**, ẩn các dữ liệu nội bộ khác! | `GET /api/inpatient-records/pets/:petId/daily-logs` <br>(`inpatientRecord.service.js:333-360`) | Lọc theo `ownerId === req.user.id` — chủ khác gọi trả **`404`** (giấu luôn sự tồn tại để chống dò IDOR). |
| **4** | **Veterinarian → System** | `SCR-25` Chi tiết nội trú | Quay lại **Cửa sổ #3 (Veterinarian)** → giải thích nhánh `<Pet Fully Recovered? = [Yes]>` → bấm **Cho xuất viện (Discharge)**. | `PATCH /api/inpatient-records/:id/discharge` <br>(`inpatientRecord.service.js:368-389`) | 1. Atomic `findOneAndUpdate({ _id, status: 'Admitted' }, { status: 'Discharged', dischargedAt: now })` (đã xuất viện rồi → `409`).<br>2. **Cố ý KHÔNG nhả chuồng ngay** vì bé vẫn nằm chờ chủ tới thanh toán & đón về ở `MF-07`!<br>3. Gửi email báo chủ đón bé. |

---

## 3. Lưu ý điểm chênh giữa Ảnh SRS `MF-06` vs Code (Chuẩn bị trước để đỡ đòn)

- **Trên ảnh SRS:** Ô `Assign Available Inpatient Cage` nằm ở làn `Receptionist`, và sau khi Discharge hệ thống set `Order = Wait Owner & Count Stay Days`.
- **Trong code thực tế:**
  1. Chính **`Veterinarian`** chọn trực tiếp chuồng trống khu `InpatientWard` trong 1 thao tác `POST /api/inpatient-records` (vì thực tế ca cấp cứu nặng cần đưa thẳng vào phòng nội trú ngay, không thể đợi lễ tân duyệt chuồng mới nhập viện!).
  2. Số ngày nằm viện `days = Math.max(1, Math.ceil((dischargedAt - admittedAt) / 86400000))` được tính chính xác tại thời điểm lập hóa đơn `POST /api/invoices` (`invoice.service.js:334-371`) — đảm bảo nằm trong ngày vẫn tính tối thiểu 1 ngày và không bao giờ tính trùng với dịch vụ `category === 'Inpatient'`!

---

## 4. Câu hỏi hội đồng hay xoáy ở `MF-06`

1. **Hỏi: Nếu thú cưng đang nằm chuồng chờ `Holding` rồi chuyển vào chuồng `InpatientWard`, nếu tạo `InpatientRecord` bị lỗi giữa chừng thì chuồng chờ cũ có bị mất không?**
   - **Đáp:** Không! `inpatientRecord.service.js:218-223` có khối rollback thủ công: nếu ghi lỗi sẽ trả chuồng nội trú về `Available`, xóa `CageAssignment` mới và khôi phục `releasedAt: null` cho chuồng chờ `Holding` cũ.
2. **Hỏi: Tại sao xuất viện rồi (`Discharged`) mà chuồng nội trú vẫn là `Occupied`? Khi nào mới về `Available`?**
   - **Đáp:** Vì bác sĩ bấm "Xuất viện" nghĩa là chốt lâm sàng "bé đã đủ khỏe để về nhà" và báo chủ đến đón; bé vẫn đang ở trong chuồng đó! Chỉ khi chủ tới quầy hoàn tất thanh toán ở `MF-07`, `markInvoicePaidAndFinalize` (`invoice.service.js:534-566`) mới đóng `CageAssignment` và trả chuồng nội trú về `Available` trong cùng 1 Transaction.
3. **Hỏi: Bác sĩ lỡ nhập nhầm `Daily Log` cho 1 ngày ở tương lai hoặc sau khi đã bấm Xuất viện thì sao?**
   - **Đáp:** Bị chặn ngay với **`400 Bad Request`** (`inpatientRecord.service.js:278-285`). Nếu 2 bác sĩ cùng thao tác (1 người ghi log, 1 người bấm xuất viện đúng 1 tích tắc), điều kiện nguyên tử `{ _id, status: 'Admitted' }` (`L289, L369`) đảm bảo không bao giờ chèn log vào hồ sơ đã xuất viện.
