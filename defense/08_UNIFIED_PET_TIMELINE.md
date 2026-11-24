# 08 — Dòng Thời Gian Thú Cưng Hợp Nhất (`GET /api/pets/:petId/timeline` — `UC-13`, `SCR-08`)

> **Tóm tắt nói trong 20 giây:** API hợp nhất toàn bộ lịch sử 1 thú cưng từ **10 collection** (Đơn hẹn, Dịch vụ, Bệnh án SOAP, Đơn thuốc, Sub-task, Ghi chú Grooming, Nội trú + Nhật ký ngày, Chuồng đang dùng, Bài chia sẻ cộng đồng, Đánh giá AI và Hóa đơn) trong 3 đợt truy vấn lô `$in + Map O(1)` (không N+1), phân quyền riêng tư chặt theo `BR-23` & `BR-20`.

---

## 1. Mở nhanh code trong VS Code (Code Map)

| Tầng | File | Chức năng |
| :--- | :--- | :--- |
| **Route** | `src/routes/pet.routes.js` (`L40, L47`) | `GET /api/pets/:petId/timeline` (sau `protect`, đặt trước `/:petId`) |
| **Validator** | `src/validators/petTimeline.validator.js` (`L1-119`) | Validate `petId`, `type`, `status`, `from`, `to` (chặn ngày ảo & `from > to`), `page`, `limit` (≤50) |
| **Controller** | `src/controllers/petTimeline.controller.js` (`L1-50`) | Gọi validator → gọi `petTimelineService.getPetTimeline(req.user, ...)` |
| **Service** | `src/services/petTimeline.service.js` (`L1-545`) | `getPetTimeline` (`L140-445`) và `getShareablePetHistory` (`L470-540`, dùng cho Cộng đồng `UC-49`) |
| **Models (10)** | `Pet`, `AppointmentOrder`, `AppointmentService`, `MedicalRecord`, `SubtaskRequest`, `InpatientRecord`, `CageAssignment`, `Invoice`, `CommunityPost`, `AiAssessment` | Chỉ đọc (`.lean()` toàn bộ) |
| **Test** | `tests/petTimeline.test.js` | Test đầy đủ hợp nhất dữ liệu, lọc, phân trang, `BR-23`, ẩn `invoice` & `aiAssessments` |

---

## 2. Luồng chạy chi tiết `GET /api/pets/:petId/timeline`

- **Quyền:** Mọi vai trò đã đăng nhập (`protect`), phân cấp dữ liệu trả về theo từng role:
  - `PetOwner`: chỉ xem thú cưng **của chính mình** (khác chủ → **`403`** ở `L151-153`); thấy đầy đủ cả **`invoice`** và **`aiAssessments`**.
  - `Receptionist`, `Admin`: xem mọi thú cưng; thấy **`invoice`**, nhưng **`aiAssessments: []`** (`L158, 177-184`).
  - `Veterinarian`, `Groomer`: xem mọi thú cưng (bao gồm toàn bộ tiền sử SOAP, đơn thuốc, cận lâm sàng, nội trú, grooming); nhưng **`invoice: null`** (`L16, 155, 209, 350`) và **`aiAssessments: []`** (`L158`).
- **Query lọc:** `type` (`All|Examination|Vaccination|Grooming|Paraclinical|Inpatient`), `status` (7 trạng thái đơn), `from`, `to` (`YYYY-MM-DD`), `page` (≥1), `limit` (`1–50`, mặc định 20).

**Luồng xử lý 6 bước tối ưu hiệu năng (`petTimeline.service.js:140-445`):**
1. **Kiểm tra thú cưng & chống IDOR (`L141-153`):** `Pet.findById(petId).populate('ownerId', ...)` → không thấy `404` → nếu `PetOwner` không trùng `ownerId` → `403`.
2. **Đợt query song song #1 (`L159-185`):** `Promise.all` lấy 4 bảng gốc theo `petId`: `AppointmentOrder`, `InpatientRecord`, `CommunityPost` (`postType: 'TimelineShare'`, `status !== 'Removed'`), và `AiAssessment` (chỉ query khi `role === 'PetOwner'`, các role khác trả ngay `[]`).
3. **Đợt query song song #2 (`L195-214`):** Dùng `$in: orderIds / inpatientIds` lấy gom `AppointmentService`, `CageAssignment`, và `Invoice` (chỉ query khi `canViewInvoice === true`).
4. **Đợt query song song #3 (`L219-231`):** Dùng `$in: appointmentServiceIds` lấy gom `MedicalRecord` và `SubtaskRequest`.
5. **Ghép dữ liệu O(1) bằng `Map` (`L234-378`):** Không query trong vòng lặp! Tự tính nhãn trạng thái thời gian thực `statusLabel` (`BR-09`, `L39-48`: nếu đang `InProgress` thì hiện `"Đang <tên dịch vụ>"`, vd `"Đang Khám tổng quát"`), tính `daysAdmitted` (`L53-60`), gắn cờ `canShareToCommunity = (PetOwner && Done)` (`L375`).
6. **Lọc, phân trang & trả về (`L380-444`):** Lọc `type`, `status`, `from`, `to`, cắt trang `slice`, trả **`200`** `{ pet, summary (7 chỉ số), activeInpatient, sharedPosts, aiAssessments, timeline, pagination }`.

### Hàm dùng chung `getShareablePetHistory` (`petTimeline.service.js:470-540`)
- Phục vụ tính năng Chia sẻ hành trình lên Cộng đồng (`UC-49`, `BR-18`): chỉ cần **3 truy vấn nhẹ**, chỉ chọn đơn `Done` & dịch vụ `Done`, chỉ lấy `code`, `date`, `service.name`, `beforePhotoUrl`, `afterPhotoUrl` — **tuyệt đối không đọc SOAP, đơn thuốc, giá tiền, note/intake hay thông tin chủ nuôi**.

---

## 3. Bảng tổng hợp Case (Tốt → Fail)

| Case | Điều kiện kích hoạt | HTTP | Message / Kết quả | Code ở đâu |
| :--- | :--- | :-: | :--- | :--- |
| **Tốt (`PetOwner`)** | Xem pet của mình | **`200`** | Đầy đủ `timeline`, `invoice`, `aiAssessments`, `canShareToCommunity` | `petTimeline.service.js:155-158, 410-444` |
| **Tốt (`Vet`/`Groomer`)** | Xem pet bất kỳ | **`200`** | Đầy đủ SOAP/subtask/inpatient/grooming; `invoice: null`, `aiAssessments: []` | `petTimeline.service.js:16, 155, 158` |
| **Tốt (`Recep`/`Admin`)** | Xem pet bất kỳ | **`200`** | Có `invoice`, nhưng `aiAssessments: []` | `petTimeline.service.js:16, 158` |
| **Fail Auth** | Chưa đăng nhập / token hết hạn | **`401`** | `'Chưa đăng nhập'` / `'Token không hợp lệ...'` | `pet.routes.js:40`, `auth.middleware.js` |
| **Fail Validate ID** | `petId` không phải 24 ký tự hex | **`400`** | `'ID thú cưng không hợp lệ'` | `petTimeline.controller.js:13-19`, `petTimeline.validator.js:23-31` |
| **Fail Validate Query** | `type`/`status` sai enum, ngày ảo (`2026-02-31`), `from > to`, `limit > 50` | **`400`** | `'Tham số lọc dòng thời gian thú cưng không hợp lệ'` + `errors` | `petTimeline.controller.js:21-27`, `petTimeline.validator.js:37-111` |
| **Fail IDOR (`BR-23`)** | `PetOwner` xem timeline thú cưng của chủ khác | **`403`** | `'Bạn không có quyền xem dòng thời gian của thú cưng này'` | `petTimeline.service.js:150-153` |
| **Fail Không tồn tại** | `petId` không có trong DB | **`404`** | `'Không tìm thấy hồ sơ thú cưng'` | `petTimeline.service.js:145-147` |

---

## 4. Câu hỏi hội đồng hay xoáy (Defense Q&A)

1. **API này tổng hợp tận 10 bảng, nếu 1 thú cưng có 50 lần khám thì có bị chậm vì N+1 query không?**
   - **Trả lời:** Không bị N+1. Dù thú cưng có 5 hay 500 đơn khám, `petTimeline.service.js:159-231` luôn chạy cố định tối đa **3 đợt `Promise.all`** dùng `$in` (tối đa ~9 câu query nhỏ có index và `.lean()`), sau đó ghép toàn bộ trên RAM bằng `Map` với độ phức tạp **O(1)**.
2. **Làm sao đảm bảo Bác sĩ và Groomer không xem lén được doanh thu / hóa đơn của khách trên Timeline?**
   - **Trả lời:** Ở `petTimeline.service.js:16, 155, 209`, với `Veterinarian` và `Groomer`, biến `canViewInvoice = false` nên hệ thống **thậm chí không query bảng `Invoice`** và trả `invoice: null` ở cả mốc đơn khám lẫn đợt nội trú.
3. **Tại sao `aiAssessments` lại chỉ trả cho `PetOwner` mà ẩn với cả Bác sĩ và Admin?**
   - **Trả lời:** Theo `BR-20`, đánh giá AI chỉ là công cụ tham khảo riêng tư cho chủ nuôi trước khi đặt lịch, không phải hồ sơ y tế chính thức. Do đó `L158, 177-184` chỉ đọc `AiAssessment` khi `currentUser.role === 'PetOwner'`, các role khác nhận `[]`.
4. **Hội đồng hỏi: Tại sao route `GET /:petId/timeline` phải đặt ở `pet.routes.js:47` (trước `GET /:petId` ở `L50`)?**
   - **Trả lời:** Tuy `/:petId/timeline` khác số segment với `/:petId`, việc gom rõ các route chuyên biệt trước route CRUD chuẩn giúp dễ kiểm soát thứ tự match của Express Router và tránh nhầm lẫn khi bảo trì.
