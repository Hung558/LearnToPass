# 21 — Báo Cáo & Thống Kê Dashboard Admin / Quầy Lễ Tân (`/api/reports` — `SCR-28`, `SCR-16`)

> **Tóm tắt nói trong 20 giây:** 3 API Aggregation tốc độ cao (chạy song song bằng `Promise.all` + `$facet`, tận dụng Compound Index `{ paymentStatus: 1, paidAt: -1 }`, giới hạn tối đa **366 ngày / 12 tháng**): (1) Doanh thu + % tăng trưởng so kỳ trước + chuỗi thời gian đã điền số `0`, (2) Vận hành (lịch hẹn, kênh Online/Walk-in, Top 5 dịch vụ, công suất chuồng), và (3) Thống kê nhanh quầy Lễ tân (đếm trực tiếp hiện tại + thời gian chờ trung bình).

---

## 1. Mở nhanh code trong VS Code (Code Map)

| Tầng | File | Chức năng |
| :--- | :--- | :--- |
| **Route** | `src/routes/report.routes.js` (`L1-14`) | `GET /revenue` (`Admin`), `GET /operations` (`Admin`), `GET /front-desk` (`Receptionist`, `Admin`) |
| **Validator** | `src/validators/report.validator.js` (`L1-168`) | Validate `period` (`day\|week\|month\|custom`), `from`/`to` (≤366 ngày), `groupBy` (`day\|month`) |
| **Controller** | `src/controllers/report.controller.js` (`L1-91`) | 3 handler (mặc định `month` cho Admin, `day` cho Front-desk) |
| **Service** | `src/services/report.service.js` (`L1-678`) | `getRevenueAnalytics` (`L135-331`), `getOperationalAnalytics` (`L336-490`), `getFrontDeskStatistics` (`L498-672`) |
| **Models (8)** | `Invoice`, `AppointmentOrder`, `AppointmentService`, `Service`, `Product`, `Cage`, `Pet`, `User` | Chỉ đọc bằng Aggregation & `countDocuments` |
| **Test** | `tests/report.test.js` | 379 dòng test đủ KPI, zero-filling, % tăng trưởng, wait time, RBAC |

---

## 2. Chi tiết 3 API — Luồng code & Toàn bộ Case

### 2.1. Quy tắc lọc thời gian dùng chung (`report.validator.js:51-156`)
- `period`: `day` (hôm nay VN), `week` (Thứ 2 → Chủ nhật tuần hiện tại), `month` (ngày 1 → cuối tháng), `custom` (bắt buộc truyền `from` hoặc `to`).
- `from > to` → **`400`**; vượt quá **`366` ngày** → **`400`** (`L131-137`).
- `groupBy` (chỉ ở `/revenue`): tự động chọn `'day'` khi `≤ 62` ngày, hoặc `'month'` khi `> 62` ngày (`L143`).

### 2.2. `GET /api/reports/revenue` — Thống kê Doanh thu (`UC-63`, `Admin`)
- **Luồng (`report.service.js:135-331`):**
  1. Quy đổi `[from, to]` giờ VN (UTC+7) sang dải UTC `[rangeStart, rangeEnd)` (`toVnDateBounds`, `L26-32`) và tính dải kỳ trước liền kề (`getPreviousPeriodRange`, `L39-54`).
  2. Chạy song song **4 Aggregation Pipeline** trên hóa đơn **`paymentStatus: 'Paid'`** theo **`paidAt`** (`L143-241`):
     - `$facet` lấy 1 lượt: `totals` (tổng thu + số HĐ), `series` (nhóm theo `%Y-%m-%d` hoặc `%Y-%m` với `timezone: '+07:00'`), `byMethod` (`Cash|VNPay|PayOS`).
     - Doanh thu kỳ trước để tính `revenueChangePercent` (nếu kỳ trước `= 0` mà kỳ này `> 0` trả **`null`** thay vì chia cho 0, `L250-256`).
     - `$unwind '$items'` + `$lookup Service` gom theo 5 nhóm dịch vụ (`InpatientDay` gom vào `'Inpatient'`, `L194-227`).
     - `$unwind '$items'` gom theo 3 loại dòng `Service | Product | InpatientDay` (`L230-240`).
  3. **Zero-filling (`buildSeriesBuckets`, `L57-83, 259-267`):** tự điền `0` cho mọi ngày/tháng không phát sinh doanh thu để biểu đồ FE liên tục không bị gãy nét.
  4. Trả **`200`** `{ period, from, to, groupBy, kpis, revenueSeries, byPaymentMethod, byServiceCategory, byItemType }`.

### 2.3. `GET /api/reports/operations` — Thống kê Vận hành (`UC-64`, `Admin`)
- **Luồng (`report.service.js:336-490`):** Chạy song song 6 truy vấn:
  - Đơn hẹn theo 7 trạng thái & kênh `Online` vs `Walk-in` (`$facet`), tự tính `completionRate`, `overdueRate`, `cancellationRate`.
  - Lượt dùng dịch vụ theo 5 danh mục + **Top 5 dịch vụ nổi bật** (loại trừ dịch vụ `Cancelled`).
  - Công suất chuồng hiện tại theo khu `Holding` / `InpatientWard` / `isolatedCages` (`buildCageOccupancySnapshot`, `L86-127`).
  - Tổng khách `PetOwner` không bị khóa, tổng `Pet` `Active`, tổng `Product` theo trạng thái.

### 2.4. `GET /api/reports/front-desk` — Widget Thống kê nhanh Quầy Lễ tân (`SCR-16`, `Receptionist` & `Admin`)
- **Luồng (`report.service.js:498-672`):** Mặc định `period = 'day'`. Trả về 2 khối:
  - `todayCounters` (thời gian thực): `expectedToday` (`Booked` hôm nay), `checkedInNow`, `inProgressNow`, `waitingForPickup` (`WaitOwner`), tình trạng chuồng hiện tại.
  - `periodStats`: tổng đơn theo trạng thái, Online/Walk-in, **thời gian chờ trung bình (`averageWaitMinutes`)** tính từ `AppointmentOrder.checkedInAt` tới `min(AppointmentService.dequeueAt)` (lọc bỏ giá trị âm nếu lệch đồng hồ, `L535-575`), số hóa đơn lập/đã thu/chưa thu và tổng tiền thu theo phương thức.

---

## 3. Bảng tổng hợp Case (Tốt → Fail)

| Endpoint | Case | HTTP | Message / Kết quả | Code ở đâu |
| :--- | :--- | :-: | :--- | :--- |
| Cả 3 API | Thành công | **`200`** | JSON phân tích tương ứng (kèm zero-filled series) | `report.controller.js:33, 57, 81` |
| `/revenue`, `/operations` | Thiếu token / Không phải `Admin` | **`401`** / **`403`** | `'Chưa đăng nhập'` / `'Bạn không có quyền...'` | `report.routes.js:8-9` |
| `/front-desk` | Thiếu token / Không phải `Receptionist` hoặc `Admin` | **`401`** / **`403`** | `'Chưa đăng nhập'` / `'Bạn không có quyền...'` | `report.routes.js:12` |
| Cả 3 API | `period` sai, `from`/`to` sai format hoặc ngày ảo, `custom` thiếu ngày, `from > to`, khoảng `> 366` ngày, `groupBy` sai | **`400`** | `{ message: 'Tham số thống kê ... không hợp lệ', errors }` | `report.validator.js:88-145`, `controller.js:26,50,74` |

---

## 4. Câu hỏi hội đồng hay xoáy (Defense Q&A)

1. **Khi lọc báo cáo 12 tháng với hàng chục nghìn hóa đơn, làm sao đảm bảo thời gian phản hồi nhanh `<= 3s`?**
   - **Trả lời:** Ba tối ưu kết hợp: (1) Compound Index `{ paymentStatus: 1, paidAt: -1 }` trên `Invoice.js:50` giúp `$match` dùng Index Range Scan ngay tầng đầu tiên; (2) Dùng **`$facet`** (`report.service.js:148`) tính cùng lúc KPI, chuỗi biểu đồ và phương thức thanh toán trong 1 lần quét dữ liệu; (3) Chạy song song các nhánh bằng `Promise.all` và giới hạn cứng tối đa 366 ngày (`report.validator.js:7`).
2. **Tại sao báo cáo doanh thu lọc theo `paidAt` thay vì `issuedAt`?**
   - **Trả lời:** Theo nguyên tắc kế toán thực thu (cash basis), hóa đơn lập ngày 31/10 (`issuedAt`) nhưng khách trả tiền ngày 01/11 (`paidAt`) thì doanh thu phải tính vào tháng 11, và chỉ tính hóa đơn `Paid` (bỏ qua `Unpaid`/`Refunded`).
3. **Nếu doanh thu tháng trước bằng 0 mà tháng này có 5 triệu thì `%` tăng trưởng trả về gì? Có bị lỗi chia 0 (`Infinity`/`NaN`) không?**
   - **Trả lời:** `report.service.js:250-256` chủ động kiểm tra `prevTotalRevenue > 0`; nếu kỳ trước bằng 0 mà kỳ này `> 0` sẽ trả `revenueChangePercent: null` để FE hiển thị nhãn "Mới phát sinh" thay vì `Infinity` hay `NaN`.
