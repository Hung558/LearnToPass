# 09 — Quản Lý Danh Mục Dịch Vụ & Bảng Giá (`/api/services` — `SCR-29`, `UC-66`)

> **Tóm tắt nói trong 20 giây:** Cho phép mọi người (kể cả Guest chưa đăng nhập) xem danh sách & bảng giá dịch vụ, còn riêng `Admin` được thêm/sửa/bật-tắt `Active/Inactive` (tự sinh mã `SRV-XXXXXX`, chống trùng tên trong cùng danh mục không phân biệt hoa/thường, tự suy diễn vai trò thực hiện `Veterinarian/Groomer`, chặn xóa cứng `403` theo `BR-16/BR-22`).

---

## 1. Mở nhanh code trong VS Code (Code Map)

| Tầng | File | Chức năng |
| :--- | :--- | :--- |
| **Route** | `src/routes/service.routes.js` (`L1-18`) | 2 GET public + 4 API ghi chỉ cho `Admin` (`protect, authorize('Admin')`) |
| **Validator** | `src/validators/service.validator.js` (`L1-108`) | 5 category, duration `5–480` phút, giá `>= 0`, `requiredRole` (`Veterinarian\|Groomer`) |
| **Controller** | `src/controllers/service.controller.js` (`L1-89`) | Điều phối 5 API + `deleteService` trả cứng `403` (`L75-79`) |
| **Service** | `src/services/service.service.js` (`L1-214`) | CRUD, `resolveServiceRequiredRole` (`L18-24`), chống trùng tên case-insensitive |
| **Model** | `src/models/Service.js` (`L1-31`) | Unique `code`, Compound Unique `{ category: 1, name: 1 }`, default `requiredRole` |
| **Test** | `tests/service.test.js`, `tests/services/service.service.test.js` | Test tích hợp & unit test |

---

## 2. Chi tiết 6 API — Luồng code & Toàn bộ Case

### 2.1. `GET /api/services` & `GET /api/services/:id` — Xem danh mục & chi tiết (Public)
- **Quyền:** Công khai (kể cả `Guest`) — `service.routes.js:8-9`.
- **Query (`GET /`):** `category?`, `status?` (`Active|Inactive`), `search?` (tìm regex đã escape trên `name`, `code`, `description` — `service.service.js:36-64`).
- **`GET /:id` (`L67-78`):** Check ObjectId (`400`) → `findById().lean()` (`404`) → trả `200 { service }`.

### 2.2. `POST /api/services` — Admin tạo dịch vụ mới (`UC-66`)
- **Quyền:** `Admin` — `service.routes.js:12`.
- **Body:** `{ name* (3–100), category* (5 loại), defaultDuration* (5–480 nguyên), basePrice* (>=0), description? (≤500), status?, requiredRole? }`.
- **Luồng (`service.service.js:81-128`):**
  1. Check trùng tên trong cùng `category` không phân biệt hoa/thường bằng `new RegExp('^' + escapedName + '$', 'i')` (`L86-94`) → trùng trả **`409`**.
  2. Vòng lặp 5 lần sinh mã `SRV-XXXXXX` (`L96-125`). Nếu không truyền `requiredRole`, Model `Service.js:16-18` tự gán `Grooming → Groomer`, còn lại `Veterinarian`.
  3. Bắt Mongo `11000`: trùng `code` thì retry, trùng `{ category, name }` thì trả **`409`**, hết 5 lần trả **`500`**. Trả **`201`** `{ service }`.

### 2.3. `PUT /api/services/:id` — Admin cập nhật dịch vụ (không cho đổi `code`)
- **Luồng (`service.service.js:131-184`):** Validate như `create` → check ObjectId (`400`) → tìm dịch vụ hiện tại (`404`) → check trùng tên với dịch vụ khác (`_id: { $ne: serviceId }`, `409`) → nếu đổi `category` mà không truyền `requiredRole` thì tự tính lại `requiredRole` (`L168-170`) → cập nhật và trả **`200`** `{ service }`.

### 2.4. `PATCH /api/services/:id/status` — Bật/tắt `Active` ↔ `Inactive`
- **Body:** `{ status*: 'Active' | 'Inactive' }`.
- **Luồng (`service.service.js:187-203`):** Check ObjectId (`400`) → `findByIdAndUpdate` → không thấy `404` → trả **`200`** `{ service }`.

### 2.5. `DELETE /api/services/:id` — Chặn xóa vật lý (`BR-16`, `BR-22`)
- **Luồng (`service.controller.js:75-79`):** Trả cứng **`403`** yêu cầu chuyển sang `Inactive`.

---

## 3. Bảng tổng hợp Case (Tốt → Fail)

| Endpoint | Case | HTTP | Message / Kết quả | Code ở đâu |
| :--- | :--- | :-: | :--- | :--- |
| `GET /`, `GET /:id` | Tốt (kể cả không token) | **`200`** | `{ services }` / `{ service }` | `service.controller.js:11, 20` |
| `GET /` | Lọc `category` hoặc `status` sai | **`400`** | `'Danh mục/Trạng thái lọc không hợp lệ'` | `service.service.js:41, 48` |
| `GET/PUT/PATCH /:id` | ID sai / Không tìm thấy | **`400`** / **`404`** | `'ID dịch vụ không hợp lệ'` / `'Không tìm thấy dịch vụ'` | `service.service.js:69,74,133,138,189,199` |
| `POST/PUT/PATCH/DELETE` | Thiếu token / Không phải `Admin` | **`401`** / **`403`** | `'Chưa đăng nhập'` / `'Bạn không có quyền...'` | `service.routes.js:12-15` |
| `POST /`, `PUT /:id` | Validate sai (`name`, `category`, `defaultDuration`, `basePrice`...) | **`400`** | `{ message: firstErrorMessage, errors }` | `service.validator.js:24-79` |
| `POST /`, `PUT /:id` | Trùng tên trong cùng danh mục (hoặc `11000`) | **`409`** | `'Tên dịch vụ đã tồn tại trong danh mục này'` | `service.service.js:93, 121, 152, 180` |
| `POST /` | Tạo thành công / 5 lần trùng mã | **`201`** / **`500`** | `{ service }` / `'Không thể tạo mã dịch vụ...'` | `service.controller.js:36`, `service.service.js:127` |
| `DELETE /:id` | Mọi request xóa cứng | **`403`** | `'Không được phép xóa vật lý dịch vụ...'` | `service.controller.js:76-78` |

---

## 4. Câu hỏi hội đồng hay xoáy (Defense Q&A)

1. **Tại sao cho phép 2 dịch vụ trùng tên nếu khác danh mục, nhưng cấm trùng trong cùng danh mục?**
   - **Trả lời:** `Service.js:28` đặt Compound Unique Index `{ category: 1, name: 1 }` kết hợp check regex không phân biệt hoa/thường ở `service.service.js:87-94`, tránh nhân viên/khách chọn nhầm dịch vụ giống hệt nhau trong cùng 1 nhóm mà vẫn linh hoạt giữa các nhóm khác nhau.
2. **Nếu Admin sửa `basePrice` của 1 dịch vụ tăng từ 150k lên 200k, các hóa đơn đã lập trước đó có bị tăng tiền theo không?**
   - **Trả lời:** Không, vì `Invoice.items` lưu `unitPriceSnapshot` tại thời điểm lập hóa đơn; dịch vụ sửa giá chỉ áp dụng cho các hóa đơn lập sau đó.
3. **Vai trò `requiredRole` dùng để làm gì?**
   - **Trả lời:** Để hệ thống biết dịch vụ nào chỉ giao cho `Veterinarian` (Khám, Tiêm, Cận lâm sàng, Nội trú) và dịch vụ nào giao cho `Groomer` (`Grooming`) — dùng trực tiếp khi lọc nhân sự ở `GET /api/users/staff?serviceId=...` và khi phân công dịch vụ / Sub-task (`service.service.js:18-24`).
