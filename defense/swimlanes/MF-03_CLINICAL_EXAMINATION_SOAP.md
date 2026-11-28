# MF-03 — Khám Lâm Sàng Chuẩn SOAP & Kê Đơn Thuốc (`SRS 2.3`, trang 11–12)

> **Tóm tắt nói trước hội đồng (20 giây):** *"Luồng `MF-03` là quy trình khám cốt lõi của Bác sĩ thú y (`Veterinarian`). Bác sĩ mở hàng đợi ưu tiên cấp cứu (`isEmergency` luôn lên đầu), bấm Bắt đầu khám thì hệ thống tự giải phóng chuồng chờ `Holding` (`BR-10`) và tự copy triệu chứng tiếp nhận vào mục **S (Subjective)** ở chế độ chỉ đọc. Bác sĩ xem lại tiền sử bệnh án cũ, điền **O-A-P**, kê đơn thuốc từ danh mục đang `Active` (lưu snapshot tên & đơn vị theo `BR-16`), cập nhật cân nặng mới, rồi hoàn tất khám và chuyển thú cưng sang chuồng chờ đón — giữ bệnh án ở trạng thái `Draft` tới khi thanh toán mới khóa cứng `Finalized` (`BR-14`)."*

---

## 1. Tái hiện đúng sơ đồ Swimlane `MF-03` trong `Report-3 SRS`

```text
┌─────────────────────────────────────────────┬─────────────────────────────────────────────┐
│                Veterinarian                 │                   System                    │
├─────────────────────────────────────────────┼─────────────────────────────────────────────┤
│                    (●)                      │                                             │
│                     ▼                       │                                             │
│     [Open Daily Queue & Select Patient]─────┼────►[Set Service = In Progress]             │
│                     ▼                       │                  │                          │
│         [Click Start Examination]           │                  ▼                          │
│                     │                       │     [Set Cage status = Available]           │
│       ┌─────────────┘                       │                  │                          │
│       ▼                                     │                  │                          │
│ [Review Past SOAP]──►[Record S & O Findings]│                  │                          │
│       │                         │           │                  │                          │
│       ▼                         ▼           │                  │                          │
│ [Record Diagnosis]──►[Create Prescription]  │                  │                          │
│                                 │           │                  │                          │
│                                 ▼           │                  │                          │
│      [Click Finalize SOAP & Complete]───────┼────►[Save SOAP (Draft) & Attach Timeline]   │
│                                 │           │                  │                          │
│                                 │           │                  ▼                          │
│                                 │           │          [Set Service = Done]               │
│                                 │           │                  │                          │
│                                 ▼           │                  │                          │
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

- **Trigger:** Lịch hẹn đã `CheckedIn`, dịch vụ khám nằm trong hàng đợi `Waiting` của Bác sĩ được phân công.
- **End condition:** Chuồng chờ được giải phóng khi bắt đầu khám; ghi xong bệnh án SOAP (`Draft`) + đơn thuốc vào Timeline; dịch vụ chuyển `Done`; nếu hết dịch vụ trong đơn thì đơn lên `WaitOwner`, nếu còn dịch vụ kế tiếp (vd Spa ở `MF-05`) thì đơn giữ `InProgress`.

---

## 2. Kịch bản Demo từng bước (Bấm gì — Nói gì — BE & DB làm gì)

| Bước | Làn (Swimlane) | Màn hình FE (`SCR` / URL) | Thao tác thực tế khi demo & Câu dẫn | API BE & Code (`file:line`) | Thay đổi trong Database |
| :-: | :--- | :--- | :--- | :--- | :--- |
| **1** | **Veterinarian** | `SCR-22` <br>`/clinic/my-queue` | Mở **Cửa sổ #3 (Veterinarian)**. Chỉ ngay cho hội đồng thấy **bé cấp cứu (`isEmergency`) từ `MF-02` đang đứng ĐẦU hàng đợi** dù đến sau bé đặt online ở `MF-01`! | `GET /api/appointment-services/my-queue` <br>(`appointmentService.service.js:283-299`) | Query sort `{ isEmergency: -1, queueNo: 1, enqueueAt: 1 }`. |
| **2** | **Vet → System** | `SCR-22` → `SCR-23` <br>`/clinic/exam/:appointmentServiceId` | Chọn ca khám của bé `Milo` (`Đơn A`) → bấm **Bắt đầu khám**. Chỉ rõ 2 thay đổi tự động: dịch vụ sang `InProgress` và **chuồng chờ `Holding` tự động về `Available`** (`BR-10`)! | `POST /api/medical-records/service/:id/start` <br>(`medicalRecord.service.js:147-214`) | 1. Kiểm tra `BR-12` (dịch vụ trước phải xong).<br>2. Đóng `CageAssignment` mở & đổi `Cage.status = 'Available'`.<br>3. `AppointmentService → InProgress` (`dequeueAt`), `AppointmentOrder → InProgress`.<br>4. Tạo nháp `MedicalRecord` (`Draft`) và **copy `order.intake.symptoms \|\| order.note` vào `subjectiveNote`**. |
| **3** | **Veterinarian** | `SCR-23` Clinical SOAP Form | Bấm xem **Tiền sử khám bệnh / Timeline** của bé ngay trên màn hình khám (chỉ cho thầy thấy Bác sĩ thấy đầy đủ SOAP/đơn thuốc cũ nhưng **không thấy hóa đơn tiền và không thấy AI triage**!). | `GET /api/medical-records/pet/:petId`, `GET /api/pets/:petId/timeline` | Đọc lịch sử theo `BR-23` (`invoice: null`, `aiAssessments: []`). |
| **4** | **Veterinarian** | `SCR-23` SOAP Form | Chỉ rõ ô **S (Subjective)** đã tự điền và **khóa chỉ đọc**. Nhập **O (Objective)** (*"Nhiệt độ 39.2°C, niêm mạc hồng"*), cập nhật **Cân nặng**, nhập **A (Assessment)** (*"Viêm dạ dày nhẹ"*), **P (Plan)** (*"Uống kháng sinh 5 ngày, theo dõi ăn uống"*), và **Kê đơn thuốc** chọn từ danh mục `Active` (vd: *Amoxicillin 250mg*, liều dùng, tần suất, 5 ngày) → bấm **Lưu nháp**. | `PUT /api/medical-records/service/:id` <br>(`medicalRecord.service.js:222-297`) | 1. Kiểm tra `MedicalRecord.status !== 'Finalized'` (`BR-14`).<br>2. Validate tất cả thuốc trong `prescriptions` đều `Product.status === 'Active'` (`BR-16`), lưu `productNameSnapshot` & `unitSnapshot`.<br>3. Cập nhật `Pet.weight`.<br>4. Lưu `MedicalRecord` (`status: 'Draft'`). |
| **5** | *(Nối sang `MF-04`)* | `SCR-23` / `SCR-24` | **ĐIỂM NỐI SIÊU MƯỢT:** Trước khi bấm Hoàn tất khám, rẽ ngang sang **`MF-04`** (chỉ định 1 Sub-task Xét nghiệm máu/X-quang + thử bấm Hoàn tất để show chốt chặn `MSG26`)! | *(Xem chi tiết ở `MF-04`)* | — |
| **6** | **Vet → System** | `SCR-23` SOAP Form | Sau khi Sub-task ở `MF-04` hoàn tất, chọn **Chuồng chờ sau khám (`waitingCageId`)** và bấm **Hoàn tất ca khám**. | `POST /api/medical-records/service/:id/complete` <br>(`medicalRecord.service.js:308-467`) | 1. Check `MSG26` (0 sub-task mở) + `BR-13` (đủ O, A, P).<br>2. Khóa chuồng chờ mới `Available → Occupied`.<br>3. Giữ `MedicalRecord.status = 'Draft'` (`BR-14`).<br>4. `AppointmentService → Done`.<br>5. Vì `Đơn A` **còn dịch vụ #2 (Spa)** → `More Services in Order? = [yes]` → `AppointmentOrder` giữ nguyên **`InProgress`**! *(Nếu đơn chỉ có 1 dịch vụ thì tự lên `WaitOwner`)*. |

---

## 3. Các chốt chặn nghiệp vụ & kỹ thuật nên nhấn mạnh

1. **Chữ S chỉ đọc (`subjectiveNote`):** Bảo toàn nguyên văn lời khai của chủ nuôi từ lúc đặt lịch/check-in (`medicalRecord.service.js:22-28`).
2. **Bắt buộc đủ O-A-P (`BR-13`):** Lưu nháp thì cho điền từng phần, nhưng bấm Hoàn tất mà thiếu bất kỳ O, A hoặc P đều trả **`400`** kèm danh sách trường thiếu.
3. **Kê đơn thuốc Snapshot (`BR-16`):** Chỉ cho kê thuốc `Active`; lưu cứng `productNameSnapshot`, `unitSnapshot` để sau này Admin đổi tên/giá thuốc cũng không làm sai lệch bệnh án lịch sử.
4. **Giữ `Draft` sau `/complete` (`BR-14`):** Chỉ khi Lễ tân thu tiền ở `MF-07`, `invoice.service.js` mới khóa vĩnh viễn bệnh án thành `Finalized` (từ đó mọi request sửa đều bị chặn `409`).

---

## 4. Câu hỏi hội đồng hay xoáy ở `MF-03`

1. **Hỏi: Nếu 1 đơn có 2 dịch vụ nối tiếp (Khám trước, Spa sau), Groomer cố bấm làm Spa trước khi Bác sĩ khám xong thì sao?**
   - **Đáp:** Bị chặn ngay bởi quy tắc thứ tự **`BR-12`** (`ensurePreviousServicesFinished` trả `409 Conflict`) — dịch vụ có `sequenceNo` lớn hơn không thể bắt đầu khi dịch vụ đứng trước chưa `Done` hoặc `Cancelled`.
2. **Hỏi: Bác sĩ A cố vào API sửa bệnh án của ca khám được giao cho Bác sĩ B có được không?**
   - **Đáp:** Không! `loadAndVerifyAssignedService` (`medicalRecord.service.js:79-101`) kiểm tra `assignedStaffId === req.user.id`, khác người trả ngay **`403 Forbidden`**.
3. **Hỏi: Tại sao trên ảnh Swimlane ghi `Record Subjective & Objective Findings` mà trên màn hình mục S không sửa được?**
   - **Đáp:** Trong hệ thống thực tế, phần S đã được ghi nhận từ bước tiếp nhận quầy (`intake.symptoms`) và tự động hiển thị trên phiếu SOAP ở dạng chỉ đọc để đảm bảo tính khách quan giữa lời khai chủ nuôi và đánh giá lâm sàng của bác sĩ.
