# 10 — Quản Lý Sản Phẩm, Thuốc & Vật Tư (`/api/products` — `SCR-31`)

> **Tóm tắt nói trong 20 giây:** Cho phép mọi người (kể cả Guest) xem danh mục thuốc/vắc-xin/bán lẻ (`Medicine | Vaccine | Retail`), còn riêng `Admin` được thêm/sửa và bật-tắt qua 2 route chuyên biệt `activate`/`inactivate`. Chặn Mass Assignment `code`/`status`, chống trùng cặp `(name, unit)` qua Compound Unique Index.

---

## 1. Mở nhanh code trong VS Code (Code Map)

| Tầng | File | Chức năng |
| :--- | :--- | :--- |
| **Route** | `src/routes/product.routes.js` (`L1-18`) | 2 GET public + 4 API ghi `Admin` (`POST`, `PUT`, `PATCH /activate`, `PATCH /inactivate`) |
| **Validator** | `src/validators/product.validator.js` (`L1-257`) | Đọc enum trực tiếp từ Model (`L4-5`), chặn client gửi `code`/`status` (`L82-92`) |
| **Controller** | `src/controllers/product.controller.js` (`L1-121`) | Trả format `{ success, data }`; `handleError` bắt cả `CastError` & `ValidationError` |
| **Service** | `src/services/product.service.js` (`L1-155`) | CRUD, tự sinh `PRD-XXXXXX` (retry 5 lần), bắt `11000` → `409` |
| **Model** | `src/models/Product.js` (`L1-18`) | Unique `code`, Compound Unique `{ name: 1, unit: 1 }` |
| **Test** | `tests/services/product.service.test.js` | Unit test service |

---

## 2. Chi tiết 6 API — Luồng code & Toàn bộ Case

### 2.1. `GET /api/products` & `GET /api/products/:id` — Xem danh sách & chi tiết (Public)
- **Query (`GET /`):** `category?` (`Medicine|Vaccine|Retail`), `status?` (`Active|Inactive`), `search?` (≤100 ký tự, tìm trên `name` & `code` đã escape regex), `page?` (mặc định 1), `limit?` (`1–100`, mặc định 20).
- **Luồng (`product.service.js:36-69`):** Chạy song song `Product.find` + `countDocuments` → trả **`200`** `{ success: true, data: { items, total, page, limit, totalPages } }`. `GET /:id` không thấy ném **`404`**.

### 2.2. `POST /api/products` — Admin thêm mới sản phẩm
- **Body:** `{ name* (≤200), category*, unit* (≤50), unitPrice? (>=0) }`. Cấm gửi `code` và `status` (`product.validator.js:82-92`).
- **Luồng (`product.service.js:75-96`):** Vòng lặp 5 lần sinh mã `PRD-XXXXXX`, luôn set `status: 'Active'`, bắt `11000`: trùng `code` → retry; trùng `(name, unit)` → **`409`**; hết 5 lần → **`500`**. Trả **`201`** `{ success: true, data: product }`.

### 2.3. `PUT /api/products/:id` — Admin cập nhật sản phẩm (partial update)
- **Body:** Ít nhất 1 trong `{ name, category, unit, unitPrice }`; cấm gửi `code`/`status`.
- **Luồng (`product.service.js:102-120`):** `findByIdAndUpdate(..., { new: true, runValidators: true })` → không thấy **`404`**, trùng `(name, unit)` **`409`** → trả **`200`** `{ success: true, data: product }`.

### 2.4. `PATCH /api/products/:id/activate` & `PATCH /api/products/:id/inactivate` — Kích hoạt / Ngừng kinh doanh
- **Luồng (`product.service.js:125-146`):** Đổi `status` sang `'Active'` / `'Inactive'` (idempotent, không xóa cứng để giữ lịch sử kê đơn & hóa đơn) → không thấy **`404`** → trả **`200`**.

---

## 3. Bảng tổng hợp Case (Tốt → Fail)

| Endpoint | Case | HTTP | Message / Kết quả | Code ở đâu |
| :--- | :--- | :-: | :--- | :--- |
| `GET /`, `GET /:id` | Tốt (kể cả không token) | **`200`** | `{ success: true, data }` | `product.controller.js:40, 53` |
| `POST /` | Tạo thành công | **`201`** | `{ success: true, data: product }` | `product.controller.js:67` |
| `POST/PUT/PATCH` | Chưa đăng nhập / Không phải `Admin` | **`401`** / **`403`** | `'Chưa đăng nhập'` / `'Bạn không có quyền...'` | `product.routes.js:9-16` |
| Mọi API | Sai `id` 24 hex / sai query / thiếu field / gửi `code` hoặc `status` / body rỗng khi PUT | **`400`** | `{ success: false, message, errors: [{ field, message }] }` | `product.validator.js:82-92, 102-249` |
| `GET/PUT/PATCH /:id` | Không tìm thấy | **`404`** | `'Không tìm thấy sản phẩm'` | `product.service.js:67, 112, 131, 144` |
| `POST /`, `PUT /:id` | Trùng cặp `(name, unit)` (`11000`) | **`409`** | `'Sản phẩm với cùng tên và đơn vị tính đã tồn tại'` | `product.service.js:90, 116` |
| `POST /` | 5 lần sinh mã `PRD-XXXXXX` đều trùng | **`500`** | `'Không thể tạo mã sản phẩm duy nhất...'` | `product.service.js:95` |

---

## 4. Câu hỏi hội đồng hay xoáy (Defense Q&A)

1. **Tại sao Unique Index của `Product` là `{ name: 1, unit: 1 }` thay vì chỉ `{ name: 1 }`?**
   - **Trả lời:** Thực tế phòng khám có thể bán cùng 1 loại thuốc dưới 2 đơn vị tính khác nhau (vd `Amoxicillin 250mg` bán lẻ theo `viên` và bán nguyên `hộp` với đơn giá khác nhau). Index cặp `(name, unit)` (`Product.js:15`) cho phép điều này nhưng vẫn chặn nhập trùng lặp hoàn toàn.
2. **Tại sao validator đọc `CATEGORIES` từ `Product.schema.path('category').enumValues` (`product.validator.js:4-5`)?**
   - **Trả lời:** Để làm Single Source of Truth — nếu sau này thêm loại sản phẩm mới vào Model, Validator tự động cập nhật theo mà không bị lệch.
3. **Khi Admin cho sản phẩm `Inactive`, điều gì xảy ra ở các luồng khác?**
   - **Trả lời:** Bác sĩ không kê thêm được thuốc `Inactive` và Lễ tân không thêm được vào hóa đơn bán lẻ mới (`BR-16`), nhưng các hóa đơn và đơn thuốc cũ vẫn hiển thị nguyên vẹn nhờ snapshot.
