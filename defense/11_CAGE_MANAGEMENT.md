# 11 — Quản Lý Chuồng Nuôi & Quy Tắc Tải Trọng / Cách Ly (`/api/cages` — `BR-10`, `BR-11`)

> **Tóm tắt nói trong 20 giây:** Quản lý danh mục chuồng (`S | M | L | XL`, khu `Holding | InpatientWard`, cờ cách ly `isIsolated`). Tất cả nhân sự phòng khám được xem; chỉ `Admin` được tạo/sửa. Service chứa 2 hàm nghiệp vụ cốt lõi `canCageAccommodatePet` và `getPreferredCageSizeOrder` (`BR-11`) dùng chung cho Check-in, Walk-in và Phân chuồng.

---

## 1. Mở nhanh code trong VS Code (Code Map)

| Tầng | File | Chức năng |
| :--- | :--- | :--- |
| **Route** | `src/routes/cage.routes.js` (`L1-25`) | `protect` toàn bộ; GET cho 4 vai trò nhân sự; POST/PATCH chỉ `Admin` |
| **Validator** | `src/validators/cage.validator.js` (`L1-211`) | Validate `label`, `size`, `isIsolated`, `status`, query phân trang + `sort` |
| **Controller** | `src/controllers/cage.controller.js` (`L1-95`) | Phân biệt `ValidationError` (`400`) vs `ServiceError` (`404/409`) |
| **Service** | `src/services/cage.service.js` (`L1-159`) | CRUD + quy tắc `BR-11` (`CAGE_MAX_WEIGHT_BY_SIZE`, `canCageAccommodatePet`) |
| **Model** | `src/models/Cage.js` (`L1-16`) | Unique `label`; `size`, `maxWeightKg`, `zone`, `isIsolated`, `status` |
| **Test** | `tests/services/cage.service.test.js` | Unit test service |

---

## 2. Chi tiết 4 API & Quy tắc `BR-11`

### 2.1. `GET /api/cages` & `GET /api/cages/:id`
- **Quyền:** `Admin`, `Receptionist`, `Veterinarian`, `Groomer` (`PetOwner` bị **`403`**) — `cage.routes.js:11-20`.
- **Query (`GET /`):** `status?` (`Available|Occupied`), `size?` (`S|M|L|XL`), `isIsolated?` (`true|false`), `page?`, `limit?` (`1–100`), `sort?` (`label|size|status|createdAt|updatedAt`, thêm `-` để giảm dần).
- **Luồng (`cage.service.js:36-64`):** Chạy song song `Cage.find` + `countDocuments` → **`200`** `{ success: true, data: { items, total, page, limit, totalPages } }`.

### 2.2. `POST /api/cages` & `PATCH /api/cages/:id`
- **Quyền:** Chỉ `Admin` — `cage.routes.js:22-23`.
- **Body:** `POST` yêu cầu `label*`, `size*`, optional `isIsolated`, `status`; `PATCH` yêu cầu ít nhất 1 trong 4 trường.
- **Luồng (`cage.service.js:23-29, 69-80`):** Ghi DB → bắt Mongo `11000` → **`409`** `'Nhãn chuồng đã tồn tại'` (`L14-16`); `PATCH` không thấy → **`404`** (`L75`).

### 2.3. Logic nghiệp vụ `BR-11` dùng chung (`cage.service.js:83-148`)
- **Bảng tải trọng mặc định (`L83-88`):** `S ≤ 5kg`, `M ≤ 15kg`, `L ≤ 30kg`, `XL = Infinity` (nếu chuồng có cấu hình `maxWeightKg` riêng thì ưu tiên `maxWeightKg`, `L134-137`).
- **`getPreferredCageSizeOrder(petWeight)` (`L96-104`):** Luôn chọn cỡ nhỏ nhất đủ vừa để không lãng phí chuồng lớn (vd chó 4kg → ưu tiên `S → M → L → XL`; chó 18kg → `L → XL`).
- **`canCageAccommodatePet(...)` (`L115-148`):**
  1. Kiểm tra cách ly 2 chiều (`L122-131`): thú cưng nghi truyền nhiễm (`requiresIsolation = true`) **bắt buộc** vào chuồng `isIsolated = true`; thú cưng thường **không được** xếp vào chuồng cách ly.
  2. Kiểm tra tải trọng (`L133-145`): nếu `petWeight > limit` → trả `{ allowed: false, reason }`.

---

## 3. Bảng tổng hợp Case (Tốt → Fail)

| Endpoint | Case | HTTP | Message / Kết quả | Code ở đâu |
| :--- | :--- | :-: | :--- | :--- |
| `GET /`, `GET /:id` | Nhân sự xem thành công | **`200`** | `{ success: true, data }` | `cage.controller.js:51, 65` |
| `POST /`, `PATCH /:id` | Admin tạo / sửa thành công | **`201`** / **`200`** | `{ success: true, message, data: cage }` | `cage.controller.js:36, 80` |
| Mọi API | Thiếu token / `PetOwner` gọi / Nhân sự thường gọi POST-PATCH | **`401`** / **`403`** | `'Chưa đăng nhập'` / `'Bạn không có quyền...'` | `cage.routes.js:8-23` |
| Mọi API | ID sai, thiếu `label`/`size`, sai enum, `isIsolated` không phải boolean, `PATCH` body rỗng | **`400`** | `{ success: false, message, errors: [...] }` | `cage.validator.js:42-203` |
| `GET /:id`, `PATCH /:id` | Không tìm thấy chuồng | **`404`** | `'Không tìm thấy chuồng'` | `cage.service.js:62, 75` |
| `POST /`, `PATCH /:id` | Trùng nhãn chuồng (`label` `11000`) | **`409`** | `'Nhãn chuồng đã tồn tại'` | `cage.service.js:14-16` |

---

## 4. Câu hỏi hội đồng hay xoáy (Defense Q&A)

1. **Làm sao hệ thống tránh xếp nhầm chó nặng 20kg vào chuồng cỡ S hoặc xếp thú thường vào chuồng cách ly?**
   - **Trả lời:** Toàn bộ luồng đặt chỗ, check-in, walk-in và gán chuồng đều gọi chung hàm `canCageAccommodatePet` (`cage.service.js:115-148`) kiểm tra cả tải trọng theo `BR-11` lẫn sự khớp 2 chiều của cờ `isIsolated`.
2. **Điểm hạn chế nhỏ cần biết nếu hội đồng soi `cage.validator.js`:**
   - `Cage.js:8` có trường `zone` (`Holding | InpatientWard`) và `maxWeightKg`, nhưng `cage.validator.js:109, 127` hiện mới pick 4 field `['label', 'size', 'isIsolated', 'status']` (nên chuồng tạo qua API mặc định nhận `zone: 'Holding'`). Nếu cần Admin tạo chuồng `InpatientWard` trực tiếp từ UI, chỉ cần thêm `'zone', 'maxWeightKg'` vào `pick` và validate enum trong `cage.validator.js`.
