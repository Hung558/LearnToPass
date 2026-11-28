# MF-08 — Báo Cáo Vận Hành & Thống Kê Doanh Thu Phòng Khám (`SRS 2.8`, trang 19)

> **Tóm tắt nói trước hội đồng (20 giây):** *"Luồng cuối cùng `MF-08` là bảng điều khiển phân tích dành riêng cho `Admin` (`SCR-28`). Ngay sau khi Lễ tân thu tiền các đơn ở `MF-07`, Admin mở Dashboard là thấy ngay **tổng doanh thu, tỷ lệ hoàn thành đơn, biểu đồ tăng trưởng, cơ cấu phương thức thanh toán (Tiền mặt vs PayOS), công suất chuồng và Top sản phẩm bán chạy** cập nhật theo thời gian thực qua 4 chế độ lọc `Day / Week / Month / Custom` (tối đa 366 ngày). Toàn bộ dùng MongoDB Aggregation Pipeline + `$facet` + Compound Index `{ paymentStatus: 1, paidAt: -1 }` nên phản hồi cực nhanh `<= 3s`!"*

---

## 1. Tái hiện đúng sơ đồ Swimlane `MF-08` trong `Report-3 SRS`

```text
┌───────────────────┐
│   Administrator   │  (●)─►[Access Executive     ]─►[Select Time Filter]                       [Review Performance]─►(◉)
│                   │       [Analytics Dashboard  ]  [(Day, Week, Month)]                       [     Metrics      ]
├───────────────────┤                                         │                                           ▲
│   System          │                                         ▼                                           │
│                   │                      [Aggregate Invoices]─►[Aggregate Orders]─►[Aggregate Best- ]─►[Render KPI Cards, ]
│                   │                                             [   by Status   ]  [selling Products]  [Revenue Line & Bar]
└───────────────────┘
```

- **Trigger:** Quản trị viên (`Admin`) truy cập màn hình phân tích quản trị để đánh giá hiệu suất vận hành và doanh thu phòng khám.
- **End condition:** Hiển thị trực quan KPI doanh thu thực thu (kèm % tăng trưởng so kỳ trước), chuỗi biểu đồ doanh thu liên tục (đã điền số `0`), phân bổ dịch vụ/trạng thái đơn hẹn, công suất chuồng và Top sản phẩm bán chạy.

---

## 2. Kịch bản Demo từng bước (Bấm gì — Nói gì — BE & DB làm gì)

| Bước | Làn (Swimlane) | Màn hình FE (`SCR` / URL) | Thao tác thực tế khi demo & Câu dẫn | API BE & Code (`file:line`) | Cơ chế xử lý phía Backend |
| :-: | :--- | :--- | :--- | :--- | :--- |
| **1** | **Administrator** | `SCR-28` <br>`/admin/reports` | Mở **Cửa sổ #5 (Admin)** → vào mục **Báo cáo & Thống kê (`/admin/reports`)**. Mặc định tải thống kê theo **`Month` (Tháng hiện tại)**. | `GET /api/reports/revenue?period=month`<br>`GET /api/reports/operations?period=month`<br>`GET /api/invoices/reports/best-selling-products` | 3 API chạy song song: **Doanh thu**, **Vận hành**, và **Top sản phẩm bán chạy** (cả 3 đều khóa quyền chỉ `Admin`, role khác trả `403`). |
| **2** | **Admin → System** | `SCR-28` Bộ lọc thời gian | Bấm chuyển bộ lọc sang **`Day` (Hôm nay)** → chỉ thẳng vào thẻ **Doanh thu hôm nay** và **Số đơn hoàn thành (`Done`)**: *"Thầy thấy số tiền vừa tăng đúng bằng giá trị hóa đơn ở `MF-07`, và sản phẩm bán lẻ/thuốc vừa kê đã nhảy lên bảng Top sản phẩm bán chạy!"* Sau đó chuyển thử sang **`Week`** và **`Month`** để show biểu đồ tự chuyển `groupBy` (`day` khi ≤ 62 ngày, `month` khi > 62 ngày). | `report.service.js:135-331` (Revenue)<br>`report.service.js:336-490` (Operations)<br>`invoice.service.js:1183-1262` (Top Products) | 1. Quy đổi dải ngày giờ VN (`UTC+7`) sang UTC.<br>2. Chỉ tính doanh thu trên hóa đơn **`paymentStatus: 'Paid'`** theo **`paidAt`** (kế toán thực thu).<br>3. Chạy 4 pipeline song song bằng `Promise.all` + `$facet`.<br>4. Tự điền **`0`** (`zero-filling`) cho các ngày/tháng không có doanh thu. |
| **3** | **Administrator** | `SCR-28` Các khối KPI & Biểu đồ | Lần lượt chỉ nhanh 4 khối thông tin đúng như ảnh `MF-08`:<br>1. **KPI Doanh thu & % tăng trưởng** so kỳ trước.<br>2. **Cơ cấu thanh toán** (`Cash` vs `PayOS`) & **Doanh thu theo loại dịch vụ**.<br>3. **Vận hành:** Tỷ lệ hoàn thành / quá hạn / hủy, kênh `Online` vs `Walk-in`, **Top 5 dịch vụ nổi bật**, **Công suất chuồng Holding & InpatientWard**.<br>4. **Top sản phẩm bán chạy** (lọc theo `Medicine / Vaccine / Retail`, xếp theo số lượng hoặc doanh thu). | `report.service.js:86-127` (Cage occupancy), `L250-267` (% tăng trưởng & zero-filling) | Múi giờ thống nhất `+07:00` để không bị lệch ngày lúc nửa đêm; giới hạn khoảng lọc tối đa **366 ngày** chống quá tải RAM. |

---

## 3. Kỹ thuật tối ưu hiệu năng nên khoe với hội đồng ở `MF-08`

1. **Compound Index `{ paymentStatus: 1, paidAt: -1 }` (`Invoice.js:50`):** Tầng `$match` đầu tiên quét trực tiếp trên Index Range Scan, không phải full collection scan cả bảng hóa đơn.
2. **Gom nhiều chỉ số trong 1 lượt quét bằng `$facet` (`report.service.js:148`):** Tính cùng lúc Tổng doanh thu, Chuỗi biểu đồ thời gian và Phân bổ phương thức thanh toán chỉ trong 1 lần duyệt tập kết quả.
3. **Zero-filling (`buildSeriesBuckets`, `L57-83`):** Tự chèn giá trị `0` cho những ngày/tháng không có giao dịch để biểu đồ đường trên FE không bao giờ bị gãy khúc hay thiếu trục X.
4. **Chống lỗi chia cho 0 (`L250-256`):** Nếu kỳ trước doanh thu = `0` mà kỳ này `> 0`, BE trả `revenueChangePercent: null` thay vì `Infinity` hay `NaN`.

---

## 4. Câu hỏi hội đồng hay xoáy ở `MF-08`

1. **Hỏi: Nếu 1 hóa đơn được lập ngày 31/10 nhưng sang ngày 01/11 khách mới trả tiền thì doanh thu tính vào tháng 10 hay tháng 11?**
   - **Đáp:** Tính vào **tháng 11**! Vì `getRevenueAnalytics` lọc trên trường **`paidAt`** với điều kiện `paymentStatus: 'Paid'` theo nguyên tắc kế toán thực thu (cash basis), không dùng `issuedAt` và bỏ qua hóa đơn `Unpaid`.
2. **Hỏi: Nếu Lễ tân hoặc Bác sĩ cố gọi API `/api/reports/revenue` xem doanh thu của phòng khám thì sao?**
   - **Đáp:** Bị chặn ngay bởi `authorize('Admin')` (`report.routes.js:8-9`) với mã **`403 Forbidden`**. Lễ tân chỉ có quyền gọi riêng `GET /api/reports/front-desk` (`SCR-16`) để xem số lượng khách hôm nay và thời gian chờ trung bình phục vụ điều phối quầy.
3. **Hỏi: Nếu Admin chọn khoảng thời gian tùy chỉnh (`custom`) dài 5 năm liên tục để ép server chạy nặng thì hệ thống xử lý sao?**
   - **Đáp:** `report.validator.js:7, 131-137` chặn cứng tối đa **`366` ngày (12 tháng)** và kiểm tra `from <= to`, vi phạm trả ngay **`400 Bad Request`** trước khi chạm vào database.
