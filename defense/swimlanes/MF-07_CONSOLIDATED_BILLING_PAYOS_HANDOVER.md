# MF-07 — Lập Hóa Đơn Tổng Hợp, Thanh Toán Tiền Mặt / PayOS VietQR & Bàn Giao Thú Cưng (`SRS 2.7`, trang 18)

> **Tóm tắt nói trước hội đồng (20 giây):** *"Luồng `MF-07` là màn chốt hạ tài chính của toàn bộ lượt khám. Lễ tân chỉ cần bấm 1 nút Lập hóa đơn cho đơn `WaitOwner` (hoặc đợt nội trú `Discharged`), hệ thống tự gom toàn bộ **5 nguồn chi phí** (Dịch vụ `Done` + Sub-task cận lâm sàng có phí + Thuốc kê đơn + Ngày nội trú + Sản phẩm bán lẻ) và chụp cứng **snapshot tên & đơn giá**. Khi còn `Unpaid`, Lễ tân được thêm/bớt sản phẩm bán lẻ nhưng cấm xóa dịch vụ để chống thất thoát. Khi thanh toán bằng **Tiền mặt** hoặc **PayOS VietQR** (ký HMAC-SHA256 + idempotent), 1 Transaction đồng bộ cùng lúc **5 bảng**: Hóa đơn `Paid` + Đơn `Done` + Khóa vĩnh viễn bệnh án `Finalized` (`BR-14`) + Đóng `CageAssignment` + Mở chuồng về `Available` — rồi xuất ngay **biên lai PDF tiếng Việt**!"*

---

## 1. Tái hiện đúng sơ đồ Swimlane `MF-07` trong `Report-3 SRS`

```text
┌───────────────────┬──────────────────────┬───────────────────────────────────────────┬──────────────────────────┐
│    Pet Owner      │     Receptionist     │                  System                   │      PayOS Gateway       │
├───────────────────┼──────────────────────┼───────────────────────────────────────────┼──────────────────────────┤
│       (●)         │                      │                                           │                          │
│        ▼          │                      │                                           │                          │
│ [Arrive for       │──►[Click Generate    │──►[Aggregate Services, Meds,              │                          │
│  pickup]          │    Consolidated      │    Inpatient Days & Retail Products]      │                          │
│                   │    Invoice]          │                    │                      │                          │
│                   │                      │                    ▼                      │                          │
│                   │         ┌────────────┼────[Display Total Payable]                │                          │
│                   │         ▼            │                                           │                          │
│                   │ <Payment Method?>    │                                           │                          │
│                   │  [Cash]    [PayOS]   │                                           │                          │
│                   │    │          │      │                                           │                          │
│                   │    ├──────────┼──────┼──►[Set Invoice = Paid]                    │                          │
│                   │    │          │      │             │                             │                          │
│                   │    │          └──────┼──►[Call PayOS API]────────────────────────┼─►[Return Dynamic QR]     │
│                   │    │                 │             │                             │           │              │
│ [Scan VietQR &    │◄───┼─────────────────┼─────────────┼─────────────────────────────┼───────────┘              │
│  complete payment]│────┼─────────────────┼─────────────┼─────────────────────────────┼─►[Return Webhook Callback]
│                   │    │                 │             │                             │           │              │
│                   │    │                 │  [Update Order = Done &   ◄──[Verify & Set│◄──────────┘              │
│                   │    │                 │   Finalize Medical Record]    Invoice=Paid]                          │
│                   │    │                 │             │                                                        │
│                   │    │                 │             ▼                                                        │
│                   │    │                 │  [Attach Receipt to   ──►[Release Cage                               │
│                   │    │                 │   Timeline]               = Available]──►(◉)                         │
└───────────────────┴──────────────────────┴───────────────────────────────────────────┴──────────────────────────┘
```

- **Trigger:** Tất cả dịch vụ trong đơn đã xong (`WaitOwner`) hoặc thú cưng nội trú đã `Discharged`, chủ nuôi đến quầy đón bé.
- **End condition:** Hóa đơn đã `Paid`, đơn hẹn `Done`, bệnh án khóa cứng `Finalized`, chuồng mở về `Available`, biên lai PDF hiển thị trên Timeline của chủ nuôi và nút **Chia sẻ hành trình lên Cộng đồng** được mở khóa.

---

## 2. Kịch bản Demo từng bước (Bấm gì — Nói gì — BE & DB làm gì)

| Bước | Làn (Swimlane) | Màn hình FE (`SCR` / URL) | Thao tác thực tế khi demo & Câu dẫn | API BE & Code (`file:line`) | Thay đổi trong Database |
| :-: | :--- | :--- | :--- | :--- | :--- |
| **1** | **Receptionist → System** | `SCR-20` <br>`/clinic/invoices` *(hoặc từ `SCR-17`)* | Mở **Cửa sổ #2 (Receptionist)** → chọn `Đơn A` (`WaitOwner`) của bé `Milo` → bấm **Lập hóa đơn tổng hợp**. Chỉ rõ cho thầy bảng chi tiết tự gom đủ: **Khám tổng quát + Tắm Spa + Sub-task Xét nghiệm (`MF-04`) + Thuốc Amoxicillin (`MF-03`)**! | `POST /api/invoices` <br>(`invoice.service.js:179-412`) | 1. Kiểm tra đơn `WaitOwner` (hoặc nội trú `Discharged`) & chống lập trùng (`409` + Partial Unique Index).<br>2. Gom tự động 4–5 nguồn, chụp cứng `itemNameSnapshot`, `unitPriceSnapshot`.<br>3. Sinh `INV-XXXXXX`, `paymentStatus: 'Unpaid'`. |
| **2** | **Receptionist** *(Demo thêm bán lẻ)* | `SCR-20` Chi tiết hóa đơn `Unpaid` | Giả lập chủ nuôi mua thêm 1 hộp pate/sữa tắm tại quầy → bấm **Thêm sản phẩm bán lẻ** (chọn sản phẩm `Active`) → tổng tiền tự cộng lại ngay! *(Nói thêm: lễ tân chỉ sửa được dòng `Product`, cố xóa dòng `Service` sẽ bị BE chặn `400` chống thất thoát doanh thu!)* | `PATCH /api/invoices/:id/items` <br>(`invoice.service.js:417-472`) | Atomic update khi `paymentStatus: 'Unpaid'`, tính lại `totalAmount`. |
| **3A** | **Nhánh `[PayOS VietQR]`** *(Khuyên demo tạo QR thật!)* | `SCR-20` Thanh toán PayOS | Bấm **Thanh toán PayOS VietQR** → hệ thống gọi PayOS API v2 trả về **mã VietQR thật** ngay trên màn hình! Nếu đơn giá nhỏ có thể quét app ngân hàng thanh toán thật → FE gọi `/payos-status` (hoặc nhận Webhook) tự động chốt `Paid` ngay lập tức! | `POST /api/invoices/:id/payos-url` (`L731-779`), `POST /payos-webhook` (`L784-815`), `GET /:id/payos-status` (`L821-859`) | Gọi PayOS API v2 (timeout 15s → `504`, lỗi → `502`). Khi PayOS báo PAID → chạy **`markInvoicePaidAndFinalize`**! |
| **3B** | **Nhánh `[Cash]`** *(Dùng cho đơn còn lại hoặc chốt nhanh)* | `SCR-20` Thanh toán tiền mặt | Chọn **Tiền mặt**, nhập số tiền khách đưa (vd đưa thừa để show hệ thống tự tính **tiền thối `changeAmount`**!) → bấm **Xác nhận thu tiền**. | `PATCH /api/invoices/:id/pay-cash` <br>(`invoice.service.js:621-652`) | Kiểm tra `cashReceived >= totalAmount` (`400`) → chạy **`markInvoicePaidAndFinalize(..., 'Cash')`**. |
| **4** | **System (5-Table Atomic Sync!)** | `SCR-19` `/cages` & `SCR-23` SOAP | **MÀN CHỨNG MINH ĂN ĐIỂM NHẤT BUỔI BẢO VỆ — Show liền 3 nơi vừa tự đổi cùng lúc:**<br>1. Hóa đơn → **`Paid`**, Đơn → **`Done`**.<br>2. Qua `/cages`: chuồng của bé đã tự nhảy về **`Available` (Trống)**!<br>3. Qua **Cửa sổ #3 (Vet)**: bệnh án SOAP của bé đã tự khóa sang **`Finalized`** (không còn sửa được nữa — `BR-14`)! | `markInvoicePaidAndFinalize` <br>(`invoice.service.js:480-616` trong `runWithTransaction`) | **5 bảng đồng bộ trong 1 Transaction:**<br>1. `Invoice`: `Unpaid → Paid` (`paidAt`)<br>2. `AppointmentOrder`: `→ Done`<br>3. `MedicalRecord`: `Draft → Finalized`<br>4. `CageAssignment`: `releasedAt = paidAt`<br>5. `Cage`: `Occupied → Available` |
| **5** | **Pet Owner** | `SCR-15` `/invoices` & `SCR-08` `/pet-journeys` | Mở **Cửa sổ #1 (PetOwner)** → bấm **Tải biên lai PDF** (mở ra file PDF tiếng Việt có dấu đẹp mắt sinh in-memory bởi `pdfmake`) → mở **Timeline** thấy đủ toàn bộ hành trình + nút **Chia sẻ lên Cộng đồng** đã bật sáng! | `GET /api/invoices/:id/pdf` (`L1058-1174`), `GET /api/pets/:petId/timeline` | Trả `application/pdf` buffer tiếng Việt (font Roboto), check IDOR (`403` nếu chủ khác truy cập). |

---

## 3. Các chốt chặn bảo mật & kỹ thuật cực mạnh ở `MF-07`

1. **Thanh toán Idempotent (Chống thu tiền / xử lý đúp):** Lệnh cập nhật nguyên tử `Invoice.findOneAndUpdate({ _id, paymentStatus: 'Unpaid' }, ...)` (`L594-600`) kết hợp kiểm tra đã `Paid` ở cả Webhook và Polling — dù PayOS retry webhook 5 lần cũng chỉ xử lý đúng 1 lần đầu.
2. **Bảo mật Webhook chống Timing Attack:** `src/utils/payos.js` xác thực chữ ký HMAC-SHA256 bằng **`crypto.timingSafeEqual`** (không dùng `===` thường) và kiểm tra `amount >= totalAmount`.
3. **Chống xóa hóa đơn gian lận (`BR-22`):** `DELETE /api/invoices/:id` cố ý trả **`403 Forbidden`** để giữ nguyên vẹn sổ sách tài chính.
4. **Không tính tiền nội trú 2 lần (`L256-259`):** Nếu đơn có `InpatientRecord`, dịch vụ `category === 'Inpatient'` được dùng làm đơn giá/ngày cho dòng `InpatientDay` chứ không cộng trùng thành 2 dòng!

---

## 4. Câu hỏi hội đồng hay xoáy ở `MF-07`

1. **Hỏi: Nếu sau 6 tháng Admin tăng giá dịch vụ "Khám tổng quát" từ 150k lên 250k thì hóa đơn cũ in lại PDF có bị nhảy thành 250k không?**
   - **Đáp:** Tuyệt đối không! Khi lập hóa đơn (`invoice.service.js:243-375`), hệ thống chụp cứng **snapshot** `itemNameSnapshot`, `unitPriceSnapshot`, `quantity`, `subtotal` vào mảng `Invoice.items`. Mọi lần xem lại hay xuất PDF sau này đều đọc từ snapshot bất biến này.
2. **Hỏi: Tại sao phải có cả PayOS Webhook lẫn API Polling `/payos-status`?**
   - **Đáp:** Trên môi trường Production HTTPS, PayOS đẩy Webhook tự động về `POST /api/invoices/payos-webhook` dù khách đóng trình duyệt. Trên môi trường localhost/demo (không có public IP) hoặc khi mạng webhook chậm, FE gọi `GET /api/invoices/:id/payos-status` hỏi trực tiếp máy chủ PayOS v2. Cả 2 luồng đều hội tụ về cùng hàm `markInvoicePaidAndFinalize` idempotent nên tuyệt đối an toàn.
3. **Hỏi: Nếu hóa đơn có 0đ (ví dụ dịch vụ miễn phí) mà lễ tân bấm tạo link PayOS thì sao?**
   - **Đáp:** BE chặn ngay với **`400 Bad Request`** (`invoice.service.js:740`) vì cổng thanh toán không chấp nhận giao dịch ≤ 0đ; lễ tân sẽ chốt bằng kênh Tiền mặt (`pay-cash`).
