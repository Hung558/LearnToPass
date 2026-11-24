# 15 — Tiếp Nhận Khách Vãng Lai Tại Quầy (`/api/appointment-orders/walk-in` — `UC-19..21`, `BR-04`)

> **Tóm tắt nói trong 20 giây:** Route lồng trong `/api/appointment-orders/walk-in`, giúp Lễ tân tra cứu nhanh khách theo SĐT (`UC-20`) và đăng ký khám vãng lai 1 chạm: tự tạo tài khoản `PetOwner` (`status: 'Inactive', passwordHash: null` theo `BR-04`), tạo thú cưng mới (hoặc chọn cũ), tạo đơn ở trạng thái **`CheckedIn`** ngay (`isOnline: false`), đẩy dịch vụ vào hàng đợi `Waiting + queueNo` nếu gán nhân sự, và gán chuồng — toàn bộ chạy trong **`runWithTransaction` + manual rollback**.

---

## 1. Mở nhanh code trong VS Code (Code Map)

| Tầng | File | Chức năng |
| :--- | :--- | :--- |
| **Route** | `src/routes/walkIn.routes.js` (`L1-16`), mount tại `appointmentOrder.routes.js:9` | `GET /customer-lookup` (`Receptionist`, `Admin`); `POST /` (`Receptionist`) |
| **Validator** | `src/validators/walkIn.validator.js` (`L1-301`) | Validate SĐT, `petId` XOR `petInfo`, `intake`, mảng `services`, `cageId` |
| **Controller** | `src/controllers/walkIn.controller.js` (`L1-66`) | Điều phối 2 API |
| **Service** | `src/services/walkIn.service.js` (`L1-457`) | `lookupCustomerByPhone` (`L148-176`) và `createWalkInBooking` (`L188-452`) |
| **Models (7)** | `User`, `Pet`, `Service`, `Cage`, `CageAssignment`, `AppointmentOrder`, `AppointmentService` | Ghi tối đa 6 bảng đồng bộ trong Transaction |
| **Test** | `tests/walkIn.test.js` | Test tra cứu, khách mới/cũ, pet mới/cũ, phân công + chuồng, rollback |

---

## 2. Chi tiết 2 API — Luồng code & Toàn bộ Case

### 2.1. `GET /api/appointment-orders/walk-in/customer-lookup?phone=...` (`UC-20`)
- **Quyền:** `Receptionist`, `Admin` — `walkIn.routes.js:10`.
- **Luồng (`walkIn.service.js:148-176`):**
  1. Validate `phone` đúng 10 số bắt đầu bằng 0 (`walkIn.validator.js:29-44`).
  2. Tìm `User.findOne({ phone }).lean()`. Không thấy → trả **`200`** `{ message: 'Khách hàng mới...', data: { found: false, customer: null, pets: [] } }` (không trả `404` để FE mở luôn form nhập khách mới!).
  3. Nếu SĐT thuộc nhân sự nội bộ (`role !== 'PetOwner'`) → **`409`** (`L159-161`).
  4. Nếu khách hàng đang `Locked` → **`403`** (`L163-165`).
  5. Lấy danh sách `Pet` đang `Active` của khách → trả **`200`** `{ data: { found: true, customer, pets } }`.

### 2.2. `POST /api/appointment-orders/walk-in` — Đăng ký khám vãng lai 1 chạm (`UC-19`, `UC-21`, `BR-04`)
- **Quyền:** Chỉ `Receptionist` — `walkIn.routes.js:13`.
- **Body:** `{ phone*, fullName? (bắt buộc nếu khách mới), email?, petId? XOR petInfo?, date? (mặc định hôm nay VN), note?, intake? ({ symptoms*, initialCondition* }), services*: [{ serviceId*, assignedStaffId?, isEmergency? }], cageId?, requiresIsolation? }`.
- **Luồng 2 pha an toàn (`walkIn.service.js:188-452`):**
  - **Pha 1 — Kiểm tra toàn bộ điều kiện trước khi ghi bất kỳ bản ghi nào (`L203-342`):**
    1. Kiểm tra tất cả `services` tồn tại (`404`) và `Active` (`400`) (`L204-216`).
    2. Nếu có `assignedStaffId`: kiểm tra nhân sự tồn tại (`404`), là `Veterinarian|Groomer` (`400`), `Active` (`400`), khớp `requiredRole` của dịch vụ (`400`), và **không có lịch báo bận trong ca hiện tại** qua `findStaffBusyBlock` (`409`, `L251-262`).
    3. Nếu có `cageId`: kiểm tra chuồng tồn tại (`404`), đang `Available` (`409`) (`L268-276`).
    4. Kiểm tra khách & thú cưng (`L279-327`):
       - Khách cũ: chặn nhân sự nội bộ (`409`), chặn `Locked` (`403`); nếu dùng `petId` thì kiểm tra đúng chủ (`404`), chưa `Deleted` (`400`), và **chống trùng đơn chưa hủy trong cùng ngày** (`409`, `L302-309`).
       - Khách mới: không được truyền `petId` (`400`), bắt buộc `fullName` (`400`), email không trùng ai (`409`).
    5. Kiểm tra tải trọng & cách ly chuồng `BR-11` qua `canCageAccommodatePet` (`400`, `L330-342`).
  - **Pha 2 — Ghi đa bảng trong `runWithTransaction` (`L345-451`):**
    1. Nếu khách mới → `createInactiveWalkInUser` (`L50-76`, `USR-XXXXXX`, `status: 'Inactive'`, `passwordHash: null` theo `BR-04`).
    2. Nếu pet mới → `createWalkInPet` (`L81-100`, `PET-XXXXXX`).
    3. Tạo `AppointmentOrder` (`L105-143`, `APT-XXXXXX`, `isOnline: false`, `derivedStatus: 'CheckedIn'`, `checkedInAt: now`, kèm `intake` nếu có).
    4. Tạo từng `AppointmentService` theo `sequenceNo = 1..N` (`L376-416`): nếu có `assignedStaffId` → đẩy thẳng sang **`Waiting` + cấp `queueNo = max + 1` + `enqueueAt`** (`L392-404`, `BR-12`).
    5. Nếu có `cageId` → gọi `createCageAssignment` cùng `session` (`L419-428`).
    6. **Nếu lỗi trên MongoDB standalone (`!session`):** `catch` (`L439-449`) dùng `Promise.allSettled` **xóa ngược** `AppointmentService`, `AppointmentOrder`, `Pet` mới, `User` mới để không để lại dữ liệu rác.
  - Trả **`201`** `{ message, data: { isNewCustomer, customer, isNewPet, pet, appointment, services, cageAssignment } }`.

---

## 3. Bảng tổng hợp Case (Tốt → Fail)

| Endpoint | Case | HTTP | Message / Kết quả | Code ở đâu |
| :--- | :--- | :-: | :--- | :--- |
| `GET /customer-lookup` | Thấy khách / Không thấy khách (khách mới) | **`200`** | `data: { found: true/false, customer, pets }` | `walkIn.service.js:151-175` |
| `GET /customer-lookup` | SĐT sai format / Tài khoản `Locked` / SĐT nhân sự | **`400`** / **`403`** / **`409`** | Lỗi tương ứng | `walkIn.validator.js:33-37`, `walkIn.service.js:160, 164` |
| `POST /` | Tạo đơn Walk-in thành công | **`201`** | `{ message, data: { isNewCustomer, customer, isNewPet, pet, appointment, services, cageAssignment } }` | `walkIn.controller.js:53`, `walkIn.service.js:430-438` |
| `POST /` | Sai vai trò (không phải `Receptionist`) | **`403`** | `'Bạn không có quyền...'` | `walkIn.routes.js:13` |
| `POST /` | Validate sai (`petId` & `petInfo` cùng gửi/cùng thiếu, ngày quá khứ, dịch vụ rỗng, `BR-11` sai tải trọng/cách ly, dịch vụ `Inactive`, sai `requiredRole`, khách mới thiếu `fullName`, pet `Deleted`) | **`400`** | Thông báo chi tiết | `walkIn.validator.js`, `walkIn.service.js:214,238,241,246,298,314,319,340` |
| `POST /` | Khách hàng đang bị `Locked` | **`403`** | `'Tài khoản khách hàng này đang bị khóa...'` | `walkIn.service.js:289` |
| `POST /` | Không thấy dịch vụ / nhân sự / chuồng / pet của khách | **`404`** | Lỗi tương ứng | `walkIn.service.js:211,235,271,295` |
| `POST /` | Nhân sự báo bận ca hiện tại / Chuồng `Occupied` / SĐT nhân sự / Email trùng / Trùng đơn trong ngày | **`409`** | Lỗi tương ứng | `walkIn.service.js:259,274,286,308,324` |
| `POST /` | Hết 5 lần sinh mã `USR`/`PET`/`APT` | **`500`** | `'Không thể tạo mã ...'` (đã rollback sạch) | `walkIn.service.js:75,99,142,441-448` |

---

## 4. Câu hỏi hội đồng hay xoáy (Defense Q&A)

1. **Tại sao tạo tài khoản khách vãng lai ở trạng thái `Inactive` và `passwordHash: null`?**
   - **Trả lời:** Theo `BR-04` (`walkIn.service.js:48-59`), khách đến trực tiếp tại quầy chưa đăng ký mật khẩu nên không được phép tự đăng nhập trước khi kích hoạt. Sau này khi khách về nhà tự đăng ký trên web bằng chính SĐT đó (`POST /api/auth/register`), `auth.service.js:62-87` sẽ tự kích hoạt tài khoản này lên `Active` để kế thừa nguyên hồ sơ thú cưng và lịch sử khám.
2. **Luồng Walk-in ghi tới 6 collection — nếu tạo xong `User`, `Pet`, `Order` mà bước gán chuồng bị lỗi (vd lễ tân khác vừa chiếm mất chuồng) thì có bị sinh tài khoản rác không?**
   - **Trả lời:** Không. Toàn bộ pha ghi nằm trong `runWithTransaction` (`walkIn.service.js:345-451`): trên MongoDB Atlas sẽ abort toàn bộ transaction; trên MongoDB standalone (`!session`), khối `catch` ở `L441-448` tự xóa ngược `AppointmentService`, `AppointmentOrder`, `Pet` mới tạo và `User` mới tạo.
3. **Tại sao `GET /customer-lookup` không thấy khách lại trả `200 { found: false }` thay vì `404`?**
   - **Trả lời:** Vì khách mới chưa có hồ sơ là một kịch bản nghiệp vụ bình thường (không phải lỗi API). Trả `200` với cờ `found: false` giúp FE nhận biết êm ái để bật form tạo mới mà không hiện toast lỗi đỏ.
