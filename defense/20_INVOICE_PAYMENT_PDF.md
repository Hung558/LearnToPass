# 20 — Hóa Đơn Tổng Hợp, Thanh Toán Tiền Mặt / VNPay / PayOS & Xuất PDF (`/api/invoices`)

> **Tóm tắt nói trong 20 giây:** Module tài chính lõi có **15 endpoint**: Lễ tân lập hóa đơn tự động gom Dịch vụ `Done` + Thuốc kê đơn + Sub-task có phí + Ngày nội trú `Discharged` + Sản phẩm bán lẻ (lưu snapshot giá), thêm/bớt sản phẩm bán lẻ khi còn `Unpaid`, thu tiền qua 3 kênh (**Tiền mặt**, **VNPay IPN**, **PayOS Webhook/Polling**). Khi chốt `Paid` trong **`runWithTransaction`**, hệ thống đồng bộ 4 việc: **Hóa đơn `Paid` + Đơn `Done` + Khóa bệnh án `Finalized` (`BR-14`) + Nhả chuồng về `Available`**. Kèm xuất biên lai PDF, báo cáo top sản phẩm và chặn xóa cứng `403` (`BR-22`).

---

## 1. Mở nhanh code trong VS Code (Code Map)

| Tầng | File | Chức năng |
| :--- | :--- | :--- |
| **Route** | `src/routes/invoice.routes.js` (`L1-41`) | 3 callback public (`vnpay-ipn`, `vnpay-return`, `payos-webhook`) + 12 API bảo vệ |
| **Validator** | `src/validators/invoice.validator.js` (`L1-364`) | Validate lập HĐ, sửa retail, tiền mặt, lọc HĐ, báo cáo sản phẩm (≤366 ngày) |
| **Controller** | `src/controllers/invoice.controller.js` (`L1-315`) | 15 handler + stream PDF (`L260-263`) + chặn xóa cứng `403` (`L294-297`) |
| **Service** | `src/services/invoice.service.js` (`L1-1262`) | Lập HĐ (`L179-412`), `finalizeAfterPayment` (`L480-616`), 3 kênh thanh toán (`L621-894`), PDF (`L1058-1174`), Top SP (`L1183-1262`) |
| **Utils** | `src/utils/vnpay.js`, `src/utils/payos.js`, `src/utils/transaction.js` | Ký/xác thực HMAC, gọi PayOS có timeout 15s, Transaction |
| **Models (10)** | `Invoice`, `AppointmentOrder`, `AppointmentService`, `MedicalRecord`, `InpatientRecord`, `SubtaskRequest`, `Service`, `Product`, `Cage`, `CageAssignment` | 4 Partial Unique Index (`orderId`, `inpatientRecordId`, `vnpayTxnRef`, `payosOrderCode`) + Compound Index `{ paymentStatus, paidAt }` |
| **Test** | `tests/invoice.test.js` | 649 dòng test đủ lập HĐ, sửa retail, Tiền mặt, VNPay IPN/Return, PayOS, PDF, IDOR, `BR-22`, Top SP |

---

## 2. Chi tiết 15 API — Luồng code & Toàn bộ Case

### 2.1. `POST /api/invoices` — Lập hóa đơn tổng hợp (`UC-45`, `Receptionist`)
- **Body:** `{ orderId?, inpatientRecordId?, retailItems?: [{ productId, quantity (1-99) }], prescriptionQuantities?: [{ productId, quantity (1-999) }], inpatientDailyRate? }`. Phải có ít nhất `orderId` hoặc `inpatientRecordId`.
- **Luồng (`invoice.service.js:179-412`):**
  1. Kiểm tra `inpatientRecord` (nếu có): tồn tại (`404`), khớp `orderId` (`400`), **phải đã `Discharged`** (đang `Admitted` → **`409`**, `L202-204`).
  2. Kiểm tra `order` (nếu có): tồn tại (`404`), chưa `Cancelled`/`Done` (`409`); nếu không phải nội trú thì đơn phải đang **`WaitOwner`** (`409`, `L219-224`).
  3. Chống lập trùng hóa đơn cho cùng `orderId` / `inpatientRecordId` (`409`, `L228-236` + Partial Unique Index `11000` ở `L404-406`).
  4. **Tự động gom 4 nguồn chi phí (lưu snapshot tên & giá):**
     - Dịch vụ `Done` (bỏ qua dịch vụ `category === 'Inpatient'` nếu đã có `inpatientRecord` để không tính trùng, `L243-270`).
     - Đơn thuốc trong `MedicalRecord` của các dịch vụ `Done` (hỗ trợ `prescriptionQuantities` ghi đè số lượng, `L274-306`).
     - `SubtaskRequest` đã `Completed` có gắn `serviceId` (`L309-329`).
     - Ngày nội trú `InpatientDay` (`days = max(1, ceil((dischargedAt - admittedAt)/1 ngày))`, `L334-371`).
     - Sản phẩm bán lẻ `retailItems` (bắt buộc `Product.status === 'Active'` theo `BR-16`, `L108-141, 374`).
  5. Không có dòng nào → **`400`** (`L377-379`). Sinh mã `INV-XXXXXX` (retry 5 lần, `L383-411`) → trả **`201`**.

### 2.2. `PATCH /api/invoices/:id/items` — Thêm/bớt sản phẩm bán lẻ khi còn `Unpaid` (`Receptionist`)
- **Luồng (`L417-472`):** Chỉ cho sửa khi `Unpaid` (`409`); **chỉ cho gỡ dòng `type === 'Product'`** (cố gỡ dịch vụ/nội trú → **`400`**, `L437-439`); thêm `addItems` phải `Active`; không được để hóa đơn rỗng (`400`, `L450`); cập nhật nguyên tử với điều kiện `paymentStatus: 'Unpaid'` (`L456-466`) → **`200`**.

### 2.3. Hàm lõi đồng bộ sau thanh toán: `markInvoicePaidAndFinalize` (`L480-616`)
Bọc trong **`runWithTransaction`** — khi hóa đơn chuyển `Unpaid → Paid` (bằng bất kỳ kênh nào):
1. Cập nhật nguyên tử `Invoice.findOneAndUpdate({ _id, paymentStatus: 'Unpaid' }, { paymentMethod, paymentStatus: 'Paid', paidAt })` (chống thanh toán đúp, `L594-600`).
2. Đổi `AppointmentOrder.derivedStatus` → **`Done`** (`L501-511`).
3. Khóa toàn bộ `MedicalRecord` đang `Draft` của đơn → **`Finalized`** (`BR-14`, `L517-532`).
4. Đóng toàn bộ `CageAssignment` đang mở (`releasedAt = paidAt`) và mở lại `Cage.status = 'Available'` (cả chuồng chờ lẫn chuồng nội trú, `L534-566`).
5. Nếu lỗi trên MongoDB standalone (`!session`), hoàn tác thủ công cả 5 bảng (`L568-583, L605-610`).

### 2.4. 3 Kênh Thanh Toán (Tiền mặt, VNPay, PayOS)
- **`PATCH /:id/pay-cash`** (`Receptionist`, `L621-652`): kiểm tra `Unpaid` (`409`), nếu truyền `cashReceived < totalAmount` → **`400`** → gọi `markInvoicePaidAndFinalize(..., 'Cash')` → trả **`200`** kèm `changeAmount` (tiền thối).
- **`POST /:id/vnpay-url`** (`Receptionist`, `L657-690`): chặn nếu `totalAmount <= 0` (`400`) hoặc không `Unpaid` (`409`) → tạo `vnpayTxnRef` & URL ký HMAC-SHA512 (hiệu lực 15p) → **`200`**.
- **`GET /vnpay-ipn`** (Public webhook từ VNPay, `L696-726`): **luôn trả HTTP `200`** theo giao thức VNPay: sai chữ ký → `RspCode: '97'`; không thấy HĐ → `'01'`; sai số tiền (`vnp_Amount !== total * 100`) → `'04'`; đã `Paid` → `'02'` (idempotent); `vnp_ResponseCode === '00'` → gọi `markInvoicePaidAndFinalize(..., 'VNPay')` → `'00'`.
- **`GET /vnpay-return`** (Public redirect browser, `L865-894`): kiểm tra chữ ký (`400` nếu sai), chỉ trả JSON hiển thị kết quả, **không sửa DB (`BR-17`)**.
- **`POST /:id/payos-url`** (`Receptionist`, `L731-779`): tạo `orderCode` số nguyên, gọi PayOS API v2 tạo `checkoutUrl` + chuỗi `qrCode` VietQR thật (timeout 15s → `504`, lỗi PayOS → `502`) → lưu `payosOrderCode` → **`200`**.
- **`POST /payos-webhook`** (Public, `L784-815`): xác thực HMAC-SHA256 (`400` nếu sai); nếu `orderCode` không khớp HĐ (request ping test `123` của PayOS) trả `200 { error: 0, message: 'Ok' }`; kiểm tra `amount >= totalAmount` (`400`); đã `Paid` thì idempotent; thành công → `markInvoicePaidAndFinalize(..., 'PayOS')`.
- **`GET /:id/payos-status`** (`Receptionist`, `Admin`, `PetOwner`, `L821-859`): kiểm tra IDOR cho `PetOwner` (`403`), gọi `getPayosPaymentInfo` chốt `Paid` ngay cả khi chạy `localhost`.

### 2.5. Tra cứu, Xuất PDF, Báo cáo sản phẩm & Chặn xóa
- **`GET /`** (`Receptionist`, `Admin`, `L938-957`): lọc `paymentStatus`, `paymentMethod`, `orderId`, `inpatientRecordId`, `petId`, `code`, `from`–`to` theo `issuedAt` giờ VN, phân trang.
- **`GET /me`** (`PetOwner`, `L962-1006`): chỉ trả hóa đơn thuộc thú cưng của mình (truyền `petId` người khác → **`403`**).
- **`GET /:id`** (`Receptionist`, `Admin`, `PetOwner`, `L1011-1035`): check IDOR (`403`), nếu đang `Unpaid` có `payosOrderCode` thì tự thử poll PayOS êm ái (bỏ qua lỗi mạng).
- **`GET /:id/pdf`** (`Receptionist`, `Admin`, `PetOwner`, `L1058-1174`): check IDOR (`403`), render biên lai tiếng Việt (`pdfmake` Roboto) trả **`200` `application/pdf`**.
- **`GET /reports/best-selling-products`** (`Admin`, `L1183-1262`): Aggregation trên hóa đơn `Paid` theo `paidAt` (tối đa 366 ngày), lọc `category` (`Medicine|Vaccine|Retail`), sắp theo `quantity` hoặc `revenue`, `limit` 1–50.
- **`DELETE /:id`** (`Receptionist`, `Admin`, `invoice.controller.js:294-297`): chặn cứng trả **`403`** (`BR-22`).

---

## 3. Bảng tổng hợp Case (Tốt → Fail)

| Endpoint | Case | HTTP | Message / Kết quả | Code ở đâu |
| :--- | :--- | :-: | :--- | :--- |
| `POST /` | Lập hóa đơn thành công | **`201`** | `{ message: 'Lập hóa đơn thành công', data }` | `invoice.controller.js:35` |
| `GET /:id/pdf` | Xuất PDF thành công | **`200`** | Binary PDF (`Content-Type: application/pdf`) | `invoice.controller.js:261-263` |
| `GET /vnpay-ipn` | Mọi kết quả (thành công / sai ký / sai tiền / đã Paid / lỗi) | **`200`** | `{ RspCode: '00'\|'01'\|'02'\|'04'\|'97'\|'99', Message }` | `invoice.service.js:697-725`, `controller.js:123` |
| `POST /`, `PATCH items`, `pay-cash`, `vnpay-url`, `payos-url`, `webhook` | Validate sai / SP `Inactive` / gỡ dòng `Service` / `cashReceived < total` / `total === 0` đi VNPay-PayOS / sai chữ ký PayOS hoặc VNPay Return | **`400`** | Thông báo cụ thể | `service.js:123,199,378,438,451,631,666,740,786,802,838,868` |
| `GET /me`, `/:id`, `/:id/pdf`, `/:id/payos-status`, `DELETE /:id` | `PetOwner` xem hóa đơn người khác / Gọi `DELETE` (`BR-22`) | **`403`** | `'Bạn không có quyền truy cập hóa đơn này'` / `'Không thể xóa vĩnh viễn hóa đơn...'` | `service.js:170, 967`, `controller.js:295` |
| Các API `/:id`, `POST /` | Không thấy hóa đơn / đơn hẹn / hồ sơ nội trú / sản phẩm / dòng hóa đơn | **`404`** | Lỗi tương ứng | `service.js:120,192,211,420,435,624,660,734,824,1014,1061` |
| `POST /`, `PATCH items`, `pay-cash`, `vnpay-url`, `payos-url` | Nội trú chưa xuất viện / đơn chưa `WaitOwner` / đã lập HĐ rồi (`11000`) / HĐ đã `Paid` | **`409`** | Chi tiết xung đột | `service.js:203,214,217,220,234,405,423,465,627,639,663,737` |
| `POST payos-url`, `GET payos-status` | Lỗi kết nối PayOS / Quá timeout 15s | **`502`** / **`504`** | Lỗi PayOS / quá thời gian chờ | `src/utils/payos.js:99, 107` |

---

## 4. Câu hỏi hội đồng hay xoáy (Defense Q&A)

1. **Khi khách thanh toán thành công (Tiền mặt / VNPay / PayOS), những bảng nào trong DB thay đổi đồng thời?**
   - **Trả lời:** Trong 1 `runWithTransaction` (`invoice.service.js:480-616`), có đúng **5 bảng** được đồng bộ cùng lúc: (1) `Invoice` → `Paid`, (2) `AppointmentOrder` → `Done`, (3) toàn bộ `MedicalRecord` `Draft` → `Finalized` (`BR-14`), (4) đóng `CageAssignment` (`releasedAt = paidAt`), và (5) mở `Cage` → `Available`.
2. **Nếu cổng VNPay hoặc PayOS gửi lại cùng 1 Webhook 2 lần liên tiếp (network retry), hệ thống có xử lý đúp không?**
   - **Trả lời:** Không — hoàn toàn **idempotent**. Cả `handleVnpayIpn:712-714` (trả `RspCode: '02'`), `handlePayosWebhook:805-807` (trả `error: 0`), lẫn lệnh nguyên tử `Invoice.findOneAndUpdate({ _id, paymentStatus: 'Unpaid' }, ...)` (`L594-600`) đảm bảo chỉ duy nhất lần đầu cập nhật DB.
3. **Nếu khách vừa khám vừa nhập viện nội trú trong cùng 1 đơn, làm sao tránh tính tiền dịch vụ nội trú 2 lần?**
   - **Trả lời:** Ở `invoice.service.js:256-259`, nếu có `inpatientRecord` thì dịch vụ `category === 'Inpatient'` trong `doneServices` sẽ không tạo dòng `Service` mà được giữ làm đơn giá `dailyRate` cho dòng `InpatientDay` nhân với số ngày nằm viện (`L343-346`).
4. **Tại sao lễ tân có thể xóa dòng `Product` trên hóa đơn `Unpaid` nhưng không xóa được dòng `Service`?**
   - **Trả lời:** Để chống thất thoát doanh thu tại quầy (`invoice.service.js:437-439`): dịch vụ đã thực hiện xong (`Done`) không được tự ý gỡ bỏ; lễ tân chỉ được thêm/bớt các sản phẩm/thuốc bán lẻ nếu khách đổi ý mua thêm hoặc bỏ bớt trước khi trả tiền.
