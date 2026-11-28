# MF-04 — Điều Trị Ca Nặng & Chỉ Định Cận Lâm Sàng / Dịch Vụ Phụ (`SRS 2.4`, trang 13–14)

> **Tóm tắt nói trước hội đồng (20 giây):** *"Luồng `MF-04` diễn ra ngay trong quá trình Bác sĩ đang khám (`InProgress`) khi cần thêm xét nghiệm máu, X-quang, siêu âm hoặc dịch vụ phụ trợ (tắm thuốc, cạo lông) do đồng nghiệp Bác sĩ hoặc Groomer hỗ trợ. Hệ thống kiểm tra đúng vai trò nhân sự & lịch bận, cho nhân sự nhận việc và tải ảnh kết quả lên Cloudinary CDN, rồi trả kết quả trực tiếp lên màn hình của Bác sĩ chính. Điểm ăn tiền nhất là chốt chặn an toàn lâm sàng **`MSG26`**: cấm Bác sĩ chính đóng bệnh án khi vẫn còn chỉ định chưa trả kết quả — đồng thời các chỉ định có gắn dịch vụ tính phí sẽ tự động cộng vào hóa đơn ở `MF-07`."*

---

## 1. Tái hiện đúng sơ đồ Swimlane `MF-04` trong `Report-3 SRS`

```text
┌───────────────────┐
│  Groomer          │                 ┌─►[Perform Spa Service]─►[Upload Before/After]──┐   [Click Submit Results]
├───────────────────┤                 │                                                ▼              ▲       │
│  Cloudinary CDN   │                 │                         ┌─►[Store file]─►[Return CDN URL]─►<Spa?>     │
├───────────────────┤                 │                         │                      │             [no]     │
│  System           │         ┌──►[Notify Groomer]              │                      │              │       ▼
│                   │         │ [yes]                           │                      │         ┌────┴──►[Set Sub-Task = Completed]
│                   │  [Save Sub-Task]─►<Spa Service?>          │                      │         │                   │
│                   │    (Pending)           [no]               │                      │         │         [Notify Attending Vet]
│                   │         ▲               ▼                 │                      │         │                   │
│                   │         │        [Notify Vet]             │                      │         │                   │  [Set Service = Done]─►(◉)
├───────────────────┤         │               │                 │                      │         │                   │        ▲
│  Veterinarian     │ (●)─►[Create Sub-Task]  ▼                 │                      ▼         │                   ▼        │
│                   │                 [Perform Procedure]─►[Upload Image]  [Input Quant. Results]┴►[Submit]  [Review Results] │
│                   │                                                                                               ▼         │
│                   │                                                                                    [Treatment Plan & Rx]│
│                   │                                                                                               ▼         │
│                   │                                                                                     [Save SOAP (Draft)]─┘
└───────────────────┘
```

- **Trigger:** Trong khi khám (`MF-03`), Bác sĩ chính xác định cần chỉ định xét nghiệm cận lâm sàng (X-quang, xét nghiệm máu, siêu âm) hoặc hỗ trợ spa y tế.
- **End condition:** Sub-task hoàn tất (`Completed`) kèm ảnh kết quả trên Cloudinary CDN và nhận định văn bản; Bác sĩ chính xem kết quả trên màn hình SOAP, hoàn thiện phác đồ/đơn thuốc và hoàn tất ca khám.

---

## 2. Kịch bản Demo từng bước (Bấm gì — Nói gì — BE & DB làm gì)

| Bước | Làn (Swimlane) | Màn hình FE (`SCR` / URL) | Thao tác thực tế khi demo & Câu dẫn | API BE & Code (`file:line`) | Thay đổi trong Database |
| :-: | :--- | :--- | :--- | :--- | :--- |
| **1** | **Veterinarian → System** | `SCR-23` / `SCR-24` <br>`/clinic/exam/:id` | Trên **Cửa sổ #3 (Bác sĩ chính)** đang khám dở bé `Milo`: mở khu vực **Chỉ định cận lâm sàng (Sub-Task)** → chọn dịch vụ tính phí (vd: *Xét nghiệm máu* hoặc *Tắm thuốc*), chọn nhân sự thực hiện (Groomer hoặc Bác sĩ phụ), nhập hướng dẫn *"Kiểm tra công thức máu toàn phần"* → bấm **Tạo chỉ định**. | `POST /api/subtask-requests` <br>(`subtaskRequest.service.js:72-170`) | 1. Kiểm tra ca khám đang `InProgress` & chưa `Finalized`.<br>2. Kiểm tra nhân sự `Active`, khớp `requiredRole` của dịch vụ, không báo bận trên `StaffSchedule`.<br>3. Tạo `SubtaskRequest` (`status: 'Pending'`). |
| **2** | **Veterinarian *(Demo chốt chặn `MSG26`!)*** | `SCR-23` SOAP Form | **MẸO ĂN ĐIỂM CAO:** Cố tình bấm **Hoàn tất ca khám** ngay lúc Sub-task còn `Pending` → Hệ thống hiện cảnh báo đỏ chặn lại (*"Còn chỉ định cận lâm sàng chưa hoàn tất"*). Nói với thầy: *"Đây là chốt chặn an toàn lâm sàng `MSG26` giúp bác sĩ không bao giờ bỏ sót kết quả xét nghiệm!"* | `POST /api/medical-records/service/:id/complete` <br>(`medicalRecord.service.js:331-340`) | Trả **`409 Conflict`**, không cho đóng ca khám. |
| **3** | **Groomer / Vet phụ** | `SCR-26` / `SCR-24` <br>`/clinic/my-queue` | Chuyển sang **Cửa sổ #4 (Groomer / BS phụ)** → mở tab **Chỉ định được giao** → thấy Sub-task `Pending` nằm trong hàng đợi FIFO → bấm **Bắt đầu thực hiện**. | `GET /api/subtask-requests/my-queue`, `PATCH /api/subtask-requests/:id/start` <br>(`subtaskRequest.service.js:238-267`) | Atomic `Pending → InProgress`. |
| **4** | **Groomer / Vet phụ → Cloudinary → System** | `SCR-24` Form trả kết quả | Nhập kết quả định lượng (*"Bạch cầu tăng nhẹ 18.5 K/uL, hồng cầu bình thường"*), chọn **1 ảnh kết quả X-quang/xét nghiệm từ Desktop** → bấm **Gửi kết quả**. | `PATCH /api/subtask-requests/:id/complete` <br>(`uploadSubtaskResultPhoto` + `subtaskRequest.service.js:273-330`) | 1. Upload ảnh RAM buffer lên Cloudinary `subtask-results/<id>`.<br>2. Atomic cập nhật `status: 'Completed'`, `resultText`, `resultImageUrl`, `completedAt`.<br>3. Nếu DB lỗi → tự gọi `deleteCloudinaryImage` dọn sạch ảnh mồ côi! |
| **5** | **Veterinarian** | `SCR-23` SOAP Form | Quay lại **Cửa sổ #3 (Bác sĩ chính)** → tải lại thấy ngay kết quả chữ + ảnh từ Cloudinary hiển thị sắc nét ngay trên phiếu SOAP → cập nhật chẩn đoán/đơn thuốc → bấm **Hoàn tất ca khám** → **Thành công `200`**! | `GET /api/subtask-requests/service/:id`, `POST /api/medical-records/service/:id/complete` | `MedicalRecord` lưu `Draft`, `AppointmentService → Done`. Sub-task `Completed` có `serviceId` sẽ **tự động cộng tiền vào Hóa đơn ở `MF-07`**! |

---

## 3. Các chốt chặn bảo mật & nghiệp vụ nên nhấn mạnh

1. **Chống Mass Assignment / Chèn link ảnh giả (`subtaskRequest.validator.js:50-53, 167-172`):** Validator từ chối (`400`) nếu client gửi trường lạ như `status` hay tự gõ `resultImageUrl` trong JSON — mọi ảnh kết quả bắt buộc đi qua `multer` upload lên Cloudinary do BE quản lý.
2. **Hủy chỉ định nhầm (`DELETE /api/subtask-requests/:id`):** Nếu Bác sĩ lỡ tạo nhầm Sub-task mà đồng nghiệp chưa bấm nhận (`Pending`), chính Bác sĩ tạo được xóa nguyên tử (`L335-363`) để không bị kẹt chốt chặn `MSG26`.
3. **Liên kết chặt với Hóa đơn (`MF-07`):** Khi Sub-task có gắn `serviceId` và đã `Completed`, `invoice.service.js:309-329` tự gom vào hóa đơn tổng hợp — tránh thất thoát chi phí xét nghiệm.

---

## 4. Câu hỏi hội đồng hay xoáy ở `MF-04`

1. **Hỏi: Nếu 1 Sub-task chỉ cần trả kết quả bằng chữ (ví dụ đo huyết áp, kết quả test nhanh) mà không có ảnh thì có gửi được không?**
   - **Đáp:** Được! `subtaskRequest.validator.js` yêu cầu có **ít nhất `resultText` hoặc `resultImage`**, không bắt buộc cả hai; nhưng nếu trống cả hai thì chặn `400`.
2. **Hỏi: Nếu mạng gặp lỗi DB ngay sau khi ảnh X-quang vừa tải lên Cloudinary thì ảnh đó có bị bỏ rác trên cloud không?**
   - **Đáp:** Không! `subtaskRequest.service.js:320, 325` bắt mọi lỗi DB hoặc lỗi hoàn tất đúp và gọi ngay `deleteCloudinaryImage` để xóa ảnh vừa upload.
3. **Hỏi: Chủ nuôi có xem được kết quả X-quang/xét nghiệm này không? Chủ nuôi khác có xem trộm được không?**
   - **Đáp:** Chủ sở hữu thú cưng xem được đầy đủ trên Unified Pet Timeline (`SCR-08`) và `GET /api/subtask-requests/service/:id`. Nếu `PetOwner` khác truyền ID vào sẽ bị chặn **`403 Forbidden`** theo `BR-23` (`subtaskRequest.service.js:205, 228`).
