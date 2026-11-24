# 24 — Trợ Lý AI Đánh Giá Sức Khỏe Thú Cưng Tham Khảo (`/api/ai-assessments` — `UC-14`, `BR-20`, `SCR-10`)

> **Tóm tắt nói trong 20 giây:** Chỉ dành riêng cho **`PetOwner`** (mọi role khác bị chặn `403`, lưu riêng collection `ai_assessments` tách biệt hoàn toàn với `MedicalRecord` theo `BR-20`). Nhận `0–3` ảnh JPG/PNG (kiểm tra Magic Bytes `MSG35`) + triệu chứng + mô tả (`10–1000` ký tự), tự nạp ngữ cảnh tiền sử khám/nội trú/dịch vụ phòng khám, gọi đa cổng **Gemini ↔ Groq** (xoay nhiều key, ngân sách `<= 30s`), đối chiếu dịch vụ thật trong DB, tạo sẵn `bookingNoteDraft`, gắn cảnh báo `MSG33`, rồi mới đẩy ảnh lên Cloudinary và lưu lịch sử.

---

## 1. Mở nhanh code trong VS Code (Code Map)

| Tầng | File | Chức năng |
| :--- | :--- | :--- |
| **Route** | `src/routes/aiAssessment.routes.js` (`L1-15`) | `router.use(protect, authorize('PetOwner'))` cho cả 3 API + `uploadAiAssessmentPhotos` trên `POST /` |
| **Middleware** | `src/middlewares/upload.middleware.js` (`L46-55, 122-147`) | Nhận `photos`/`photo` (tổng ≤ 3 ảnh, JPG/PNG ≤ 5MB); cho phép gửi JSON (0 ảnh) |
| **Validator** | `src/validators/aiAssessment.validator.js` (`L1-227`) | Check Magic Bytes JPG/PNG (`L18-26, 137`), `description` 10–1000 ký tự, `symptoms` tối đa 20 |
| **Controller** | `src/controllers/aiAssessment.controller.js` (`L1-97`) | 3 handler: `create`, `list`, `getById` |
| **Service** | `src/services/aiAssessment.service.js` (`L1-331`) | Gom ngữ cảnh 8 bảng (`L71-180`), gọi AI → upload Cloudinary → lưu `AiAssessment` (`L188-250`) |
| **AI Utility** | `src/utils/gemini.js` (`L1-675`) | Prompt chuẩn AAHA/AVMA, xoay nhiều key Gemini/Groq, tự chọn model Vision, timeout tổng `<= 30s` |
| **Model** | `src/models/AiAssessment.js` (`L1-87`) | Collection `ai_assessments`, index `{ petId, createdAt: -1 }` & `{ ownerId, createdAt: -1 }` |
| **Test** | `tests/aiAssessment.test.js` | 604 dòng test đủ RBAC 5 role, Magic Bytes, failover Gemini→Groq, timeout 504, `BR-20`, Pet Timeline |

---

## 2. Chi tiết 3 API — Luồng code & Toàn bộ Case

### 2.1. `POST /api/ai-assessments` — Gửi đánh giá sức khỏe tham khảo (`UC-14`, `PetOwner`)
- **Input:** `application/json` (0 ảnh) hoặc `multipart/form-data` (`0–3` ảnh qua field `photos` hoặc `photo`):
  - `petId*` (ObjectId), `description*` (`10–1000` ký tự), `symptoms?` (mảng / JSON string / chuỗi dấu phẩy, tối đa 20 thẻ).
- **Luồng 5 bước (`aiAssessment.service.js:188-250`):**
  1. **Kiểm tra thú cưng & chống IDOR (`L189-196`):** `Pet.findById(petId)` → không có hoặc `Deleted` → **`404`**; không thuộc `currentUser.id` → **`403`**.
  2. **Gom ngữ cảnh lâm sàng (`gatherPetAndClinicContext`, `L71-180`):** Chạy song song đọc 10 đơn gần nhất, 10 bệnh án SOAP + đơn thuốc cũ, 5 đợt nội trú, cận lâm sàng, 5 ghi chú Grooming, danh mục `Service` đang `Active`, công suất chuồng trống & số bác sĩ `Active`.
  3. **Gọi AI đa cổng (`analyzePetHealthWithAi`, `gemini.js:561-658`):**
     - Prompt theo chuẩn **AAHA / AVMA Veterinary Teletriage** (`gemini.js:119-218`), cấm chẩn đoán khẳng định & cấm kê đơn thuốc (`BR-20`).
     - Luân chuyển `GEMINI_API_KEY_1..N` → `GROQ_API_KEY_1..N` (tự chuyển sang `qwen/qwen3.8-27b` nếu có ảnh) trong deadline cứng **`30s`**.
     - Chuẩn hóa JSON trả về (`urgency`: `Routine|Monitor|Urgent`, `redFlags`, `visualFindings`, `possibleCauses`, `advice`, `contraindications`, tối đa 3 `recommendedServices` đối chiếu đúng `Service` thật trong DB, `bookingNoteDraft ≤ 500`, câu miễn trừ `MSG33`).
     - Tất cả key đều lỗi → **`503`** (`MSG34`); gặp timeout → **`504`** (`MSG34`).
  4. **Upload ảnh lên Cloudinary SAU khi AI thành công (`L220-223`):** Đẩy song song vào `vcmos/ai-assessments` (lỗi → **`502`**).
  5. **Lưu bảng riêng `AiAssessment` (`L225-249`):** Tuyệt đối không ghi vào `MedicalRecord` → trả **`201`** `{ message, assessment }`.

### 2.2. `GET /api/ai-assessments` & `GET /api/ai-assessments/:id` — Lịch sử & Chi tiết (`PetOwner`)
- **`GET /` (`L257-300`):** Luôn khóa `ownerId = currentUser.id`; lọc tùy chọn `petId` (không phải pet mình → **`403`**, không có/xóa → **`404`**), `urgency`, `page`, `limit` (≤50) → **`200`** `{ items, pagination }`.
- **`GET /:id` (`L305-323`):** Tìm bản ghi (`404`), kiểm tra `ownerId === currentUser.id` (`403` nếu khác chủ) → **`200`** `{ assessment }`.

---

## 3. Bảng tổng hợp Case (Tốt → Fail)

| Endpoint | Case | HTTP | Message / Kết quả | Code ở đâu |
| :--- | :--- | :-: | :--- | :--- |
| `POST /` | Đánh giá thành công | **`201`** | `{ message, assessment }` (kèm `disclaimer` `MSG33`, `aiProvider`, `aiModel`, `responseTimeMs`) | `aiAssessment.controller.js:42`, `service.js:225-249` |
| `GET /`, `GET /:id` | Lấy lịch sử / chi tiết thành công | **`200`** | `{ items, pagination }` / `{ assessment }` | `controller.js:66, 87` |
| Cả 3 API | Thiếu token / Role không phải `PetOwner` (`Vet`, `Groomer`, `Recep`, `Admin`) | **`401`** / **`403`** | `'Chưa đăng nhập'` / `'Bạn không có quyền...'` | `aiAssessment.routes.js:9`, `service.js:259, 307` |
| `POST /` | `description` `<10` hoặc `>1000`, sai `petId`, `>3` ảnh, `>5MB`, sai định dạng / sai Magic Bytes JPG-PNG (`MSG35`), `>20` triệu chứng | **`400`** | `{ message, errors }` | `upload.middleware.js:70,141`, `aiAssessment.validator.js:96-149` |
| Cả 3 API | `PetOwner` gửi/xem cho thú cưng hoặc bản ghi của chủ khác (IDOR) | **`403`** | `'Bạn chỉ được gửi/không có quyền xem...'` | `service.js:195, 270, 319` |
| Cả 3 API | Không thấy pet (hoặc pet `Deleted`) / không thấy bản ghi AI | **`404`** | `'Không tìm thấy hồ sơ thú cưng'` / `'Không tìm thấy bản ghi đánh giá AI'` | `service.js:191, 267, 315` |
| `POST /` | Upload ảnh lên Cloudinary thất bại | **`502`** | `'Không thể tải ảnh đánh giá AI lên hệ thống...'` | `service.js:58` |
| `POST /` | Tất cả nhà cung cấp/key AI lỗi, hết quota hoặc chưa cấu hình (`MSG34`) | **`503`** | `'Dịch vụ AI đang bận. Vui lòng thử lại sau.'` | `gemini.js:573, 657` |
| `POST /` | Quá thời gian chờ `<= 30s` (`MSG34`) | **`504`** | `'Dịch vụ AI đang bận. Vui lòng thử lại sau.'` | `gemini.js:435, 512, 657` |

---

## 4. Câu hỏi hội đồng hay xoáy (Defense Q&A)

1. **Làm sao đảm bảo kết quả AI không làm sai lệch bệnh án chính thức của Bác sĩ?**
   - **Trả lời:** Theo `BR-20`, kết quả AI lưu ở bảng riêng `ai_assessments`, tách biệt hoàn toàn với `medical_records`. Toàn bộ endpoint `/api/ai-assessments` và nhánh `aiAssessments` trên Pet Timeline chỉ mở cho `PetOwner` sở hữu thú cưng; Bác sĩ không bao giờ bị lẫn dữ liệu AI vào bệnh án SOAP, và mọi kết quả đều gắn cứng câu miễn trừ `MSG33`.
2. **Nếu AI bị lỗi hoặc timeout, bản ghi lỗi có bị lưu rác vào DB và ảnh có bị đẩy rác lên Cloudinary không?**
   - **Trả lời:** Không. `aiAssessment.service.js:212-247` gọi AI trước trên `Buffer` trong RAM; chỉ khi AI trả về JSON hợp lệ mới upload ảnh lên Cloudinary (`L222`) rồi tạo bản ghi `AiAssessment` (`L225`). Nếu AI lỗi `503`/`504`, không hề ghi gì vào DB hay Cloudinary.
3. **Điểm gì khiến tính năng AI của nhóm khác với việc gọi ChatGPT hỏi đại một câu?**
   - **Trả lời:** Ba điểm: (1) Tự động bơm ngữ cảnh thực tế từ 8 bảng của phòng khám (tuổi, cân nặng, dị ứng, bệnh án SOAP, đơn thuốc cũ, nội trú, dịch vụ `Active` đang bán — `gatherPetAndClinicContext:71-180`); (2) Đối chiếu dịch vụ gợi ý với danh mục thật của phòng khám chống AI bịa (`gemini.js:301-344`) và sinh sẵn `bookingNoteDraft` để đặt lịch 1 chạm; (3) Cơ chế xoay vòng nhiều key & failover tự động `Gemini ↔ Groq` (tự chuyển model Vision khi có ảnh) trong ngân sách `<= 30s`.
