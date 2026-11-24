# 12 — Phân Công & Giải Phóng Chuồng (`/api/cage-assignments` — `UC-27`, `UC-29`, `BR-10`)

> **Tóm tắt nói trong 20 giây:** Quản lý vòng đời giữ chuồng của từng đơn khám (`CageAssignment`). Dùng cập nhật nguyên tử (`findOneAndUpdate`) kết hợp Unique Partial Index `{ cageId: 1, releasedAt: null }` để chống 2 nhân viên chọn trùng 1 chuồng hoặc giải phóng trùng 2 lần, kèm cơ chế tự rollback trạng thái chuồng nếu bước sau lỗi.

---

## 1. Mở nhanh code trong VS Code (Code Map)

| Tầng | File | Chức năng |
| :--- | :--- | :--- |
| **Route** | `src/routes/cageAssignment.routes.js` (`L1-41`) | `protect` + `authorize('Receptionist', 'Veterinarian', 'Groomer')` trên cả 5 API |
| **Validator** | `src/validators/cageAssignment.validator.js` (`L1-123`) | Check ObjectId 24 hex (chống NoSQL injection `{ "$ne": null }`), lọc `active` |
| **Controller** | `src/controllers/cageAssignment.controller.js` (`L1-95`) | Lấy `assignedById` từ `req.user.id`, đẩy lỗi qua `next(err)` về Global Error Handler |
| **Service** | `src/services/cageAssignment.service.js` (`L1-205`) | Atomic claim chuồng (`L50-54`), kiểm tra `BR-11` (`L66-78`), atomic release (`L170-174`) + rollback |
| **Models** | `CageAssignment.js`, `Cage.js` | Partial Unique `{ cageId: 1 }` khi `releasedAt: null` (`CageAssignment.js:15-18`) |
| **Test** | `tests/services/cageAssignment.service.test.js` | Unit test service |

---

## 2. Chi tiết 5 API — Luồng code & Toàn bộ Case

### 2.1. `POST /api/cage-assignments` — Phân công chuồng cho đơn khám (`UC-27`)
- **Quyền:** `Receptionist`, `Veterinarian`, `Groomer` (`cageAssignment.routes.js:21-25`).
- **Body:** `{ cageId*, orderId* }` (`assignedById` tự lấy từ `req.user.id` tại `cageAssignment.controller.js:14`).
- **Luồng chống Race Condition (`cageAssignment.service.js:34-98`):**
  1. **Chiếm chuồng nguyên tử (`L50-54`):** `Cage.findOneAndUpdate({ _id: cageId, status: 'Available' }, { $set: { status: 'Occupied' } })`.
     - Nếu trả `null` (`L56-63`): kiểm tra `Cage.exists` → không tồn tại ném **`404`**, đang `Occupied` ném **`409`**.
  2. **Kiểm tra `BR-11` (khi truyền `petWeight`/`requiresIsolation` từ Check-in/Walk-in, `L66-78`):** Không phù hợp → **rollback chuồng về `Available`** (`L75`) và ném **`400`**.
  3. **Tạo bản ghi `CageAssignment` (`L80-97`):** Hỗ trợ nhận `session` khi chạy trong Transaction; nếu tạo thất bại (vd dính Partial Unique Index `11000`) và không có `session` → **rollback chuồng về `Available`** (`L88-90`), ném **`409`** (`L93-95`).
  4. Populate `cageId`, `orderId`, `assignedById` → trả **`201`**.

### 2.2. `GET /api/cage-assignments` & `GET /:id` & `GET /order/:orderId` — Tra cứu chuồng
- `GET /`: lọc tùy chọn `cageId`, `orderId`, `active` (`true` → `releasedAt: null`, `false` → `releasedAt: { $ne: null }`), sắp mới nhất trước (`L106-123`).
- `GET /:id`: lấy theo ID (`404` nếu không thấy, `L128-135`).
- `GET /order/:orderId`: tìm phân công **đang mở (`releasedAt: null`)** mới nhất của đơn (`L143-154`, không có trả **`404`**).

### 2.3. `PATCH /api/cage-assignments/:id/release` — Giải phóng chuồng (`UC-29`)
- **Luồng chống giải phóng đúp (`cageAssignment.service.js:166-197`):**
  1. **Đóng nguyên tử (`L170-174`):** `CageAssignment.findOneAndUpdate({ _id: id, releasedAt: null }, { $set: { releasedAt: new Date() } })`.
     - Nếu trả `null` (`L176-180`): không tồn tại → **`404`**, đã giải phóng trước đó → **`409`**.
  2. **Mở lại chuồng (`L182-194`):** `Cage.updateOne({ _id: assignment.cageId }, { $set: { status: 'Available' } })`. Nếu bước này lỗi → **rollback `releasedAt: null`** (`L189-192`) để không bao giờ bị lệch dữ liệu giữa 2 bảng.
  3. Trả **`200`** `{ message: 'Giải phóng chuồng thành công', data: assignment }`.

---

## 3. Bảng tổng hợp Case (Tốt → Fail)

| Endpoint | Case | HTTP | Message / Kết quả | Code ở đâu |
| :--- | :--- | :-: | :--- | :--- |
| `POST /` | Phân chuồng thành công | **`201`** | `{ message: 'Phân công chuồng thành công', data }` | `cageAssignment.controller.js:19` |
| `GET /`, `GET /:id`, `GET /order/:orderId`, `PATCH /:id/release` | Thành công | **`200`** | `{ message, data }` | `cageAssignment.controller.js:34,50,65,80` |
| Mọi API | Thiếu token / `PetOwner` hoặc `Admin` gọi | **`401`** / **`403`** | `'Chưa đăng nhập'` / `'Bạn không có quyền...'` | `cageAssignment.routes.js:8-39` |
| Mọi API | Thiếu/sai ObjectId, `active` sai (`true`/`false`), sai tải trọng/cách ly `BR-11` | **`400`** | `{ message, errors }` / lý do `BR-11` | `cageAssignment.validator.js`, `cageAssignment.service.js:76` |
| `POST /`, `GET /:id`, `GET /order/:orderId`, `PATCH /:id/release` | Không thấy chuồng / assignment / đơn chưa có chuồng mở | **`404`** | `'Không tìm thấy chuồng'` / `'Không tìm thấy phân công chuồng'` / `'Đơn hẹn này chưa có...'` | `cageAssignment.service.js:61,132,150,178` |
| `POST /` | Chuồng đang `Occupied` hoặc trùng Partial Index `11000` | **`409`** | `'Chuồng không ở trạng thái trống (Available)'` / `'Chuồng đã có phân công đang hoạt động'` | `cageAssignment.service.js:62, 94` |
| `PATCH /:id/release` | Assignment đã giải phóng trước đó | **`409`** | `'Phân công chuồng này đã được giải phóng trước đó'` | `cageAssignment.service.js:179` |

---

## 4. Câu hỏi hội đồng hay xoáy (Defense Q&A)

1. **Tại sao phải dùng `Cage.findOneAndUpdate({ _id, status: 'Available' }, { $set: { status: 'Occupied' } })` thay vì `findById` rồi `if (cage.status === 'Available')`?**
   - **Trả lời:** Nếu tách thành 2 lệnh `findById` rồi `save`, 2 lễ tân bấm chọn cùng 1 chuồng trong cùng 1 mili-giây sẽ cùng thấy `Available` và ghi đè nhau. `findOneAndUpdate` kiểm tra điều kiện và cập nhật trong 1 thao tác nguyên tử của MongoDB nên chỉ đúng 1 người lấy được chuồng, người thứ 2 nhận `null → 409`.
2. **Nếu cập nhật `Cage` sang `Occupied` thành công nhưng bước `CageAssignment.create` bị lỗi mạng thì chuồng có bị kẹt mãi ở `Occupied` không?**
   - **Trả lời:** Không. Dòng `88-90` của `cageAssignment.service.js` bắt lỗi và rollback ngay `Cage` về `Available` (còn khi chạy trong `session` Transaction thì MongoDB tự abort toàn bộ). Tương tự ở hàm `releaseCageAssignment:188-192` cũng rollback `releasedAt: null` nếu update `Cage` lỗi.
3. **Tại sao trong `cageAssignment.service.js:21` ghi `select: '-password'` trong khi `User` lưu field `passwordHash`?**
   - **Trả lời:** Dù ghi `-password`, thực tế `User.js:10-12` đã gắn `select: false` trên `passwordHash`, `emailOtpHash`, `emailOtpExpiresAt` nên không bao giờ lộ ra ngoài; tuy nhiên nên sửa lại thành `.select('code fullName role')` cho gọn và tường minh hơn.
