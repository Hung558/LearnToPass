# 01 — Cơ Sở Dữ Liệu, 18 Models, Indexes, Transactions & Seed (VCMOS_BE)

> **Phạm vi:** Tầng dữ liệu MongoDB/Mongoose 9 của toàn bộ hệ thống VCMOS_BE. Tra cứu nhanh Schema, quan hệ ERD, toàn bộ Index, cơ chế Transaction + Fallback Rollback và Script Seed.

---

## 1. Bản đồ Code (Code Map)

| Tầng | File mã nguồn | Vai trò |
| :--- | :--- | :--- |
| **Config DB** | `src/config/db.js` (`L1-13`) | Kết nối MongoDB qua `process.env.MONGO_URI`, fail-fast `process.exit(1)` khi lỗi |
| **Transaction** | `src/utils/transaction.js` (`L1-40`) | Nhận diện Replica Set/Atlas vs Standalone local, bọc `session.withTransaction()` |
| **Models (18)** | `src/models/User.js` → `AiAssessment.js` | 18 Mongoose Schema ánh xạ 18 collection đặt tên `snake_case` |
| **Seed** | `src/seeds/seed.js` (`L1-138`) | Khởi tạo dữ liệu mẫu cho 17/18 collection, có chốt an toàn `--force` |
| **Transaction Callers** | `src/services/appointmentOrder.service.js:935`, `src/services/walkIn.service.js:345`, `src/services/invoice.service.js:592` | 3 luồng ghi đa bảng dùng `runWithTransaction` |

---

## 2. Kết nối DB & Tổng quan 18 Models (`src/models/*.js`)

### 2.1. Kết nối MongoDB (`src/config/db.js:3-11`)
- `connectDB()` gọi `mongoose.connect(process.env.MONGO_URI)`.
- Nếu kết nối lỗi → in `MongoDB connection error` và gọi **`process.exit(1)`** (`db.js:9`) để dừng server ngay lập tức (không để server chạy "chết lâm sàng").
- Khi chạy test (`npm test`), `server.js:73` không gọi `connectDB()`; từng file test tự kết nối MongoDB local `vcmos_test` (`mongodb://127.0.0.1:27017/vcmos_test`).

### 2.2. Danh sách 18 Models & Collection

| # | Model (`file`) | Collection | Khóa ngoại chính (`ref`) | Trường đặc biệt / Ẩn / Subdocument |
| :-: | :--- | :--- | :--- | :--- |
| 1 | `User` (`User.js:1-29`) | `users` | — | `passwordHash`, `emailOtpHash`, `emailOtpExpiresAt` đều có **`select: false`** (`L10-12`); `failedLoginAttempts`, `lockUntil` (`L19-20` cho `BR-03`) |
| 2 | `Pet` (`Pet.js:1-23`) | `pets` | `ownerId → User` | Soft delete qua `status: 'Active' \| 'Deleted'` (`L15`) |
| 3 | `Service` (`Service.js:1-31`) | `services` | — | `requiredRole` có **dynamic default function** (`L16-18`): `Grooming → 'Groomer'`, còn lại `'Veterinarian'` |
| 4 | `Product` (`Product.js:1-18`) | `products` | — | `category`: `Medicine \| Vaccine \| Retail`; `status`: `Active \| Inactive` |
| 5 | `StaffSchedule` (`StaffSchedule.js:1-21`) | `staff_schedules` | `staffId`, `createdBy → User` | Mô hình **Blacklist**: chỉ tạo record khi nhân sự **báo bận** theo `(staffId, date, shift)` (`MORNING \| AFTERNOON \| EVENING`) |
| 6 | `Cage` (`Cage.js:1-16`) | `cages` | — | `size`: `S \| M \| L \| XL`, `zone`: `Holding \| InpatientWard`, `isIsolated`, `status`: `Available \| Occupied` |
| 7 | `CageAssignment` (`CageAssignment.js:1-22`) | `cage_assignments` | `cageId → Cage`, `orderId → AppointmentOrder`, `assignedById → User` | `releasedAt: null` đánh dấu chuồng đang được giữ (`L9`) |
| 8 | `AppointmentOrder` (`AppointmentOrder.js:1-46`) | `appointment_orders` | `petId → Pet`, `ownerId → User`, `reservedCageId → Cage`, `cancelledBy → User` | Subdoc `intakeSchema` (`L4-12`, `_id: false`); `date` dạng chuỗi `YYYY-MM-DD` (`L19`) |
| 9 | `AppointmentService` (`AppointmentService.js:1-46`) | `appointment_services` | `orderId → AppointmentOrder`, `serviceId → Service`, `preferredStaffId`, `assignedStaffId`, `cancelledBy → User` | Subdoc `groomingNoteSchema` (`L3-13`, `_id: false`); `queueNo`, `enqueueAt`, `dequeueAt`, `isEmergency` |
| 10 | `MedicalRecord` (`MedicalRecord.js:1-36`) | `medical_records` | `appointmentServiceId → AppointmentService` (`unique`), `petId → Pet` | Subdoc `prescriptionSchema` (`L3-11`) lưu snapshot `productNameSnapshot`, `unitSnapshot`; `status: Draft \| Finalized` |
| 11 | `SubtaskRequest` (`SubtaskRequest.js:1-23`) | `subtask_requests` | `appointmentServiceId → AppointmentService`, `serviceId → Service`, `requestedById`, `assignedStaffId → User` | `status: Pending \| InProgress \| Completed`, `resultText`, `resultImageUrl`, `completedAt` |
| 12 | `InpatientRecord` (`InpatientRecord.js:1-34`) | `inpatient_records` | `petId → Pet`, `orderId → AppointmentOrder`, `attendingVetId → User`, `cageId → Cage` | Subdoc mảng `dailyLogs` (`dailyLogSchema` `L3-9`: `date`, `recordedById`, `observations`, `notes`); `status: Admitted \| Discharged` |
| 13 | `Invoice` (`Invoice.js:1-54`) | `invoices` | `orderId → AppointmentOrder`, `inpatientRecordId → InpatientRecord`, `issuedById → User` | Subdoc mảng `items` (`invoiceItemSchema` `L3-10`: `type: Service \| Product \| InpatientDay`, lưu snapshot tên & đơn giá); `vnpayTxnRef`, `payosOrderCode`, `payosPaymentLinkId` |
| 14 | `CommunityPost` (`CommunityPost.js:1-33`) | `community_posts` | `authorId → User`, `petId → Pet` | Subdoc mảng `timelineItems` (`L3-7`); denormalized counters `likeCount`, `commentCount` (`L23-24`, `min: 0`) |
| 15 | `PostLike` (`PostLike.js:1-14`) | `post_likes` | `postId → CommunityPost`, `userId → User` | Khóa cặp duy nhất chống like trùng |
| 16 | `PostComment` (`PostComment.js:1-17`) | `post_comments` | `postId → CommunityPost`, `authorId → User`, `parentCommentId → PostComment` | Hỗ trợ reply 1 cấp (`parentCommentId: null` là gốc); soft delete `Published \| Deleted \| Hidden` |
| 17 | `ContentReport` (`ContentReport.js:1-26`) | `content_reports` | `reporterId`, `resolvedById → User`, `targetId` (dynamic `ref`) | **Dynamic `ref` function** (`L13`): `doc.targetType === 'Post' ? 'CommunityPost' : 'PostComment'` |
| 18 | `AiAssessment` (`AiAssessment.js:1-87`) | `ai_assessments` | `petId → Pet`, `ownerId → User`, `recommendedServices.serviceId → Service` | **Tách biệt hoàn toàn với `MedicalRecord` (`BR-20`)**; lưu 3 subdoc `_id: false`: `petSnapshot`, `recommendedServices`, `clinicalContextSummary`; `aiProvider: gemini \| groq` |

> **Lưu ý về Pre-save Hook & Virtual:** Toàn bộ 18 model **không dùng pre-save hook hay Mongoose virtual** nào — mọi logic tính toán/băm mật khẩu/sinh mã đều viết tường minh ở tầng `services/` để dễ đọc và tránh tác dụng phụ ẩn. Model duy nhất có logic nội tại là `Service.js:16-18` (`defaultRequiredRole`) và `ContentReport.js:13` (dynamic `ref`).

---

## 3. Toàn bộ Index & Tác dụng thực tế (Anti-Duplicate & Query Performance)

### 3.1. Nhóm Unique Partial Index (Chống trùng có điều kiện — Điểm cộng cực lớn khi bảo vệ)

> **Tại sao dùng `partialFilterExpression` thay vì `sparse: true`?**
> Trong MongoDB, `{ unique: true, sparse: true }` chỉ bỏ qua document **không có key** (`undefined`), nhưng nếu document lưu **`field: null`** (mặc định của Mongoose khi có `default: null`), MongoDB vẫn index giá trị `null` và ném `E11000` ngay từ bản ghi thứ 2. Do đó hệ thống dùng `partialFilterExpression` theo `$type` hoặc `status`:

| Collection | Tọa độ (`file:line`) | Định nghĩa Index | Tác dụng nghiệp vụ |
| :--- | :--- | :--- | :--- |
| `users` | `User.js:26` | `{ email: 1 }`, `unique: true`, `partial: { email: { $type: 'string' } }` | SĐT là định danh bắt buộc (`unique` tại `L7`); nhiều user không có email (`null`/`undefined`) không bị trùng, nhưng đã có email thì duy nhất |
| `cage_assignments` | `CageAssignment.js:15-18` | `{ cageId: 1 }`, `unique: true`, `partial: { releasedAt: { $type: 'null' } }` | **Chống double-booking chuồng ở tầng DB:** mỗi chuồng chỉ có tối đa 1 assignment đang mở (`releasedAt: null`), nhưng vẫn lưu được vô hạn lịch sử đã giải phóng |
| `inpatient_records` | `InpatientRecord.js:28-31` | `{ petId: 1 }`, `unique: true`, `partial: { status: 'Admitted' }` | Mỗi thú cưng chỉ được có tối đa **1 đợt nội trú đang `Admitted`** cùng lúc; sau khi `Discharged` có thể nhập viện đợt mới |
| `invoices` | `Invoice.js:33-36` | `{ orderId: 1 }`, `unique: true`, `partial: { orderId: { $type: 'objectId' } }` | 1 đơn khám chỉ lập được tối đa 1 hóa đơn (nhiều hóa đơn nội trú/bán lẻ có `orderId: null` không đụng nhau) |
| `invoices` | `Invoice.js:37-40` | `{ inpatientRecordId: 1 }`, `unique: true`, `partial: { inpatientRecordId: { $type: 'objectId' } }` | 1 đợt nội trú chỉ lập được tối đa 1 hóa đơn |
| `invoices` | `Invoice.js:41-44` | `{ vnpayTxnRef: 1 }`, `unique: true`, `partial: { vnpayTxnRef: { $type: 'string' } }` | Chống trùng mã giao dịch VNPay; hóa đơn tiền mặt/PayOS để `null` không bị lỗi |
| `invoices` | `Invoice.js:45-48` | `{ payosOrderCode: 1 }`, `unique: true`, `partial: { payosOrderCode: { $type: 'number' } }` | Chống trùng mã đơn PayOS (kiểu số nguyên); hóa đơn khác để `null` không bị lỗi |

### 3.2. Nhóm Single & Compound Unique Index

| Collection | Tọa độ (`file:line`) | Index | Tác dụng |
| :--- | :--- | :--- | :--- |
| `users` | `User.js:5,7` | `{ code: 1 }`, `{ phone: 1 }` (`unique`) | Định danh mã `USR-XXXXXX` và SĐT duy nhất toàn hệ thống |
| `pets` | `Pet.js:5` | `{ code: 1 }` (`unique`) | Mã `PET-XXXXXX` duy nhất |
| `services` | `Service.js:5,28` | `{ code: 1 }`, `{ category: 1, name: 1 }` (`unique`) | Mã `SRV-XXXXXX` duy nhất; không trùng tên dịch vụ trong cùng danh mục |
| `products` | `Product.js:5,15` | `{ code: 1 }`, `{ name: 1, unit: 1 }` (`unique`) | Mã `PRD-XXXXXX` duy nhất; cho phép cùng tên nhưng khác đơn vị (vd: viên vs hộp) |
| `cages` | `Cage.js:5` | `{ label: 1 }` (`unique`) | Nhãn chuồng (`C-01`, `IP-01`) duy nhất |
| `staff_schedules` | `StaffSchedule.js:17` | `{ staffId: 1, date: 1, shift: 1 }` (`unique`) | 1 nhân sự chỉ báo bận 1 lần cho mỗi `(Ngày, Ca)` |
| `appointment_orders` | `AppointmentOrder.js:16` | `{ code: 1 }` (`unique`) | Mã `APT-XXXXXX` duy nhất |
| `appointment_services` | `AppointmentService.js:41` | `{ orderId: 1, sequenceNo: 1 }` (`unique`) | Số thứ tự dịch vụ trong 1 đơn không bị trùng |
| `medical_records` | `MedicalRecord.js:15-19` | `{ appointmentServiceId: 1 }` (`unique`) | Quan hệ 1-1: mỗi dịch vụ khám chỉ có tối đa 1 bệnh án SOAP |
| `invoices` | `Invoice.js:14` | `{ code: 1 }` (`unique`) | Mã `INV-XXXXXX` duy nhất |
| `post_likes` | `PostLike.js:11` | `{ postId: 1, userId: 1 }` (`unique`) | Chống 1 user thả tim 2 lần trên cùng 1 bài (`BR-19`) |
| `content_reports` | `ContentReport.js:23` | `{ reporterId: 1, targetType: 1, targetId: 1 }` (`unique`) | Chống spam báo cáo trùng 1 nội dung (`UC-56`) |

### 3.3. Nhóm Compound Query Index (Tối ưu truy vấn & báo cáo)

| Collection | Tọa độ (`file:line`) | Index | Tối ưu cho truy vấn nào |
| :--- | :--- | :--- | :--- |
| `pets` | `Pet.js:20` | `{ ownerId: 1 }` | Lọc danh sách thú cưng theo chủ nuôi |
| `staff_schedules` | `StaffSchedule.js:18` | `{ date: 1, shift: 1 }` | Kiểm tra nhanh ai đang bận trong 1 ca khi đặt lịch / phân công |
| `appointment_orders` | `AppointmentOrder.js:40-43` | `{ petId: 1, date: 1 }`, `{ ownerId: 1, date: 1 }`, `{ date: 1, derivedStatus: 1 }`, `{ date: 1, reservedCageId: 1 }` | Timeline thú cưng, đơn của chủ nuôi, `JOB-01` quét `Booked`, báo cáo vận hành và kiểm tra chuồng đã giữ chỗ theo ngày |
| `appointment_services` | `AppointmentService.js:42-43` | `{ assignedStaffId: 1, status: 1, enqueueAt: 1 }`, `{ orderId: 1 }` | Hàng đợi `my-queue` của Bác sĩ/Groomer sắp xếp theo thời điểm vào hàng đợi & load dịch vụ theo đơn |
| `cage_assignments` | `CageAssignment.js:19` | `{ orderId: 1 }` | Tra cứu nhanh chuồng đang gắn với 1 đơn khám |
| `medical_records` | `MedicalRecord.js:33` | `{ petId: 1, createdAt: -1 }` | Tiền sử bệnh án & Pet Timeline sắp mới nhất trước |
| `subtask_requests` | `SubtaskRequest.js:19-20` | `{ assignedStaffId: 1, status: 1 }`, `{ appointmentServiceId: 1 }` | Hàng đợi cận lâm sàng của nhân sự & lấy subtask theo ca khám |
| `invoices` | `Invoice.js:50-51` | `{ paymentStatus: 1, paidAt: -1 }`, `{ issuedAt: -1 }` | **Aggregation thống kê doanh thu & sản phẩm bán chạy 12 tháng `<= 3s` (`T-52`)** và danh sách hóa đơn mới nhất |
| `community_posts` | `CommunityPost.js:29-30` | `{ status: 1, createdAt: -1 }`, `{ authorId: 1 }` | Bảng tin `Published` / hàng đợi `PendingReview` & bài viết của tôi |
| `post_comments` | `PostComment.js:14` | `{ postId: 1, createdAt: 1 }` | Load bình luận theo thứ tự thời gian của bài viết |
| `content_reports` | `ContentReport.js:22` | `{ status: 1, createdAt: 1 }` | Hàng đợi xử lý báo cáo `Pending` FIFO cho Admin |
| `ai_assessments` | `AiAssessment.js:83-84` | `{ petId: 1, createdAt: -1 }`, `{ ownerId: 1, createdAt: -1 }` | Lịch sử đánh giá AI theo thú cưng / chủ nuôi & tích hợp Pet Timeline |

---

## 4. Tổng hợp State Machine các thực thể cốt lõi

| Thực thể | Các trạng thái (`enum`) | Chuyển trạng thái chính |
| :--- | :--- | :--- |
| `User.status` | `Active` ↔ `Locked` / `Inactive` | Admin khóa/mở khóa (`UC-62`), không xóa cứng (`BR-22`) |
| `Pet.status` | `Active` → `Deleted` | Soft delete, chặn xóa `409` nếu còn đơn chưa hoàn tất |
| `Cage.status` | `Available` ↔ `Occupied` | Đồng bộ khi phân chuồng / giải phóng chuồng / nhập-xuất viện |
| `AppointmentOrder.derivedStatus` | `Booked` → `CheckedIn` → `InProgress` → `WaitOwner` → `Done`; nhánh `Booked` → `Overdue` (`JOB-01`); nhánh `Booked`/`CheckedIn` → `Cancelled` | Tính suy dẫn từ trạng thái các `AppointmentService` con & hóa đơn `Paid` |
| `AppointmentService.status` | `Booked` → `Waiting` → `InProgress` → `Done`; nhánh `Booked` → `Overdue` (chỉ dịch vụ #1); nhánh → `Cancelled` | Lễ tân gán nhân sự đẩy `Waiting`, BS/Groomer `start` → `InProgress`, hoàn tất → `Done` |
| `MedicalRecord.status` | `Draft` → `Finalized` | Chốt `Finalized` xong thì khóa chỉnh sửa (`409`) |
| `SubtaskRequest.status` | `Pending` → `InProgress` → `Completed` | Nhân sự cận lâm sàng nhận ca rồi trả kết quả |
| `InpatientRecord.status` | `Admitted` → `Discharged` | Nhập viện chiếm chuồng `InpatientWard` (và nhả chuồng `Holding` cũ); xuất viện giữ chuồng tới khi chủ đón/thanh toán |
| `Invoice.paymentStatus` | `Unpaid` → `Paid` (`Refunded` dự phòng trong schema) | Thanh toán Tiền mặt / VNPay IPN / PayOS Webhook hoặc Polling; `Paid` tự đồng bộ đơn sang `Done` |
| `CommunityPost.status` | `PendingReview` → `Published` / `Hidden` → `Removed` | Chủ bài chỉ sửa/xóa khi `PendingReview`; có thể thu hồi (`Removed`) kể cả khi đã `Published` |

---

## 5. Cơ chế MongoDB Transaction & Fallback Rollback (`src/utils/transaction.js`)

### 5.1. Cách hoạt động (`src/utils/transaction.js:7-34`)
1. `supportsMongoTransaction()` (`L7-13`): đọc `mongoose.connection?.client?.topology?.description?.type`. Trả về `true` **chỉ khi** topology là `'ReplicaSetWithPrimary'` (MongoDB Atlas / Replica Set) hoặc `'Sharded'`.
2. `runWithTransaction(workFn)` (`L19-34`):
   - Nếu là **MongoDB Standalone local** (môi trường Jest test `vcmos_test`): bỏ qua session, gọi thẳng `workFn(null)` — mỗi service tự thực hiện **manual compensating rollback** khi `session === null`.
   - Nếu là **Atlas / Replica Set**: gọi `mongoose.startSession()` → `session.withTransaction(async () => { result = await workFn(session); })` → `finally { await session.endSession(); }`.

### 5.2. 3 luồng ghi đa bảng dùng `runWithTransaction`

| Luồng | Tọa độ gọi | Các collection ghi đồng bộ | Xử lý khi lỗi trên Standalone (`session = null`) |
| :--- | :--- | :--- | :--- |
| **Check-in đơn online** | `appointmentOrder.service.js:935` (`runCheckInWrite` `L767-933`) | `AppointmentOrder` (`CheckedIn`, `intake`) + `AppointmentService` (gán `assignedStaffId`, `queueNo`, `Waiting`) + `CageAssignment` + `Cage` (`Occupied`) | Lưu snapshot trạng thái trước khi ghi (`previousOrder`, `previousServiceSnapshots`, `createdAssignmentId`), bắt `catch` và hoàn tác ngược từng collection |
| **Tiếp nhận Walk-in** | `walkIn.service.js:345` (`createWalkInBooking` `L188-452`) | `User` (tạo mới nếu khách mới) + `Pet` (tạo mới nếu pet mới) + `AppointmentOrder` (`CheckedIn`) + `AppointmentService` (1..N) + `CageAssignment` + `Cage` (`Occupied`) | Lưu cờ `createdCustomerId`, `createdPetId`, `createdOrderId`, `createdServiceIds`, `createdAssignmentId` để rollback xóa ngược khi fail giữa chừng |
| **Thanh toán hóa đơn** | `invoice.service.js:592` (`finalizeAfterPayment` `L480-586` trong `payCash` / VNPay IPN / PayOS) | `Invoice` (`Paid`, `paidAt`, `paymentMethod`) + `AppointmentOrder` (`Done`) + `CageAssignment` (giải phóng chuồng nếu còn mở) + `Cage` (`Available`) | Trên Atlas chạy atomic trong 1 transaction; kiểm tra điều kiện `paymentStatus: 'Unpaid'` tránh thanh toán đúp |

---

## 6. Cơ chế Seed dữ liệu (`src/seeds/seed.js`)

- **Chốt an toàn (`seed.js:28-35`):** Đếm bản ghi trên toàn bộ model đã nạp; nếu bất kỳ collection nào đã có `count > 0` và lệnh **không** có cờ `--force` → ném lỗi dừng ngay để bảo vệ DB Atlas dùng chung của team.
- **Đồng bộ Index (`seed.js:37-40`):** Chạy `Model.deleteMany({})` và `Model.syncIndexes()` trước khi tạo dữ liệu mẫu.
- **Tài khoản mẫu (`seed.js:45-48`):** `0901000001 / 123456` (`PetOwner` - `USR-000001`), `0901000002 / 123456` (`Veterinarian` - `USR-000002`).
- **Khoảng trống cần nhớ khi bảo vệ:** `seed.js:14-18` destructure 17 model (chưa tạo document mẫu cho `AiAssessment`, dù `fs.readdirSync` ở `L11-13` vẫn require `AiAssessment.js` để sync index).

---

## 7. Ma trận lỗi tầng Database & Transaction

| Kịch bản kích hoạt | Lỗi gốc Mongo/Mongoose | HTTP Status | Cách hệ thống xử lý (`file:line`) |
| :--- | :--- | :-: | :--- |
| Sai URI / mất mạng MongoDB khi khởi động | `MongoServerSelectionError` | *(process exit)* | `src/config/db.js:8-9` ghi log và `process.exit(1)` |
| Trùng Unique Index (`phone`, `email`, `code`, `(category,name)`, `(postId,userId)`, `orderId` hóa đơn, `cageId` đang mở...) | `MongoServerError` (`err.code === 11000`) | **`409`** (hoặc retry nếu trùng `code` ngẫu nhiên) | Bắt trong từng Service (`isDuplicateCodeError` retry tối đa 5 lần; trùng dữ liệu nghiệp vụ ném `409`) |
| Hết 5 lần retry sinh mã ngẫu nhiên vẫn trùng `11000` | Custom `500` | **`500`** | `auth.service.js:118`, `pet.service.js:102`, `invoice.service.js:411`, `appointmentOrder.service.js`, `walkIn.service.js` |
| Truyền ObjectId sai định dạng (vd `/api/pets/abc`) | Validator chặn trước (hoặc `CastError`) | **`400`** | Validator kiểm tra `mongoose.isValidObjectId` trước khi chạm DB; `product.controller.js` bắt thêm `CastError` |
| Sai kiểu/enum/minlength khi gọi `.create()` / `.save()` | Mongoose `ValidationError` | **`400`** | Validator chặn trước; 6 controller (`appointmentOrder`, `cage`, `product`, 3 `community*`) bắt thêm `err.name === 'ValidationError'` |
| Chạy transaction trên MongoDB local standalone | Không mở session | *(không lỗi)* | `src/utils/transaction.js:20-22` bỏ qua session và dùng fallback rollback trong service |

---

## 8. Điểm nóng kỹ thuật & Bộ câu hỏi phản biện (Defense Q&A)

### Q1: Tại sao anh dùng `partialFilterExpression` thay vì `unique: true, sparse: true` cho `User.email` và `Invoice.orderId`?
- **Trả lời (30s):** Trong Mongoose, khi khai báo `default: null`, document được lưu thực sự chứa khóa `orderId: null`. Index `sparse: true` chỉ bỏ qua document *vắng mặt key*, còn `null` vẫn bị đưa vào index nên hóa đơn thứ 2 có `orderId: null` sẽ nổ `E11000`. Dùng `partialFilterExpression: { orderId: { $type: 'objectId' } }` ép MongoDB chỉ index khi trường đó mang giá trị thật đúng kiểu, giải quyết triệt để.
- **Code chứng minh:** `src/models/User.js:25-26`, `src/models/Invoice.js:31-48`.
- **Hạn chế & cải thiện:** Cần đảm bảo `syncIndexes()` được chạy khi deploy DB mới để partial index được tạo đúng trên production.

### Q2: Làm sao đảm bảo 2 lễ tân không gán cùng 1 chuồng cho 2 thú cưng khác nhau tại cùng 1 mili-giây?
- **Trả lời (30s):** Không chỉ check bằng `if` trong JavaScript (dễ bị race condition), bọn em khóa cứng ngay ở tầng DB bằng Unique Partial Index `{ cageId: 1 }` với điều kiện `{ releasedAt: { $type: 'null' } }`. Nếu 2 request lọt qua bước check cùng lúc, MongoDB chỉ cho 1 lệnh `insert` thành công, lệnh thứ 2 dính `E11000` và bị service bắt trả `409 Conflict`.
- **Code chứng minh:** `src/models/CageAssignment.js:14-18`.

### Q3: Tại sao trong `Invoice.items`, `MedicalRecord.prescriptions` và `AiAssessment.petSnapshot` lại lưu bản sao tên và giá (snapshot) thay vì chỉ `ref` ObjectId rồi `populate`?
- **Trả lời (30s):** Vì bảng giá dịch vụ, giá thuốc và cân nặng thú cưng thay đổi theo thời gian. Nếu chỉ `populate` sang bảng gốc `Service`/`Product`, khi Admin tăng giá dịch vụ tháng sau thì toàn bộ hóa đơn cũ từ tháng trước sẽ bị đổi số tiền, sai lệch báo cáo tài chính. Lưu snapshot tại thời điểm phát sinh đảm bảo tính bất biến của chứng từ kế toán và hồ sơ y tế.
- **Code chứng minh:** `src/models/Invoice.js:3-10`, `src/models/MedicalRecord.js:3-11`, `src/models/AiAssessment.js:3-16`.

### Q4: Tại sao trường `date` trong `AppointmentOrder`, `StaffSchedule`, `InpatientRecord.dailyLogs` lưu kiểu `String` (`YYYY-MM-DD`) thay vì `Date`?
- **Trả lời (30s):** Nghiệp vụ phòng khám làm việc theo **ngày lịch địa phương Việt Nam (UTC+7)**, không có nhu cầu so sánh mili-giây. Lưu kiểu `Date` UTC dễ gây lệch ngày giữa `00:00–06:59` giờ VN (bị đẩy về hôm qua theo UTC). Lưu chuỗi ISO `YYYY-MM-DD` có regex kiểm tra vừa so sánh được bằng `$lte`/`$gte` theo thứ tự từ điển, vừa dùng làm key trong Compound Index cực gọn.
- **Code chứng minh:** `src/models/AppointmentOrder.js:19`, `src/models/StaffSchedule.js:9`, `src/models/InpatientRecord.js:4`.

### Q5: MongoDB local chạy test không phải Replica Set, vậy làm sao test được các luồng có Transaction?
- **Trả lời (30s):** Em tách hàm `runWithTransaction` trong `src/utils/transaction.js`: nó kiểm tra `topology.description.type` của kết nối hiện tại. Trên MongoDB Atlas (`ReplicaSetWithPrimary`), nó mở `mongoose.startSession()` và chạy `session.withTransaction()`. Khi chạy test trên MongoDB standalone local, nó truyền `session = null` vào callback và để service chạy nhánh compensating rollback thủ công.
- **Code chứng minh:** `src/utils/transaction.js:7-34`, `src/services/appointmentOrder.service.js:935`, `src/services/walkIn.service.js:345`.
- **Hạn chế & cải thiện:** Trong môi trường CI/CD chuyên nghiệp hơn có thể chạy MongoDB local 1-node Replica Set (hoặc `mongodb-memory-server` mode ReplSet) để test trực tiếp cả nhánh commit/abort của session thật.

### Q6: Tại sao tách `AiAssessment` thành collection riêng mà không nhúng vào `MedicalRecord`?
- **Trả lời (30s):** Tuân thủ quy tắc nghiệp vụ `BR-20`: kết quả AI chỉ là sàng lọc tham khảo cho chủ nuôi, tuyệt đối không phải chẩn đoán y khoa của bác sĩ. Tách riêng collection `ai_assessments` đảm bảo không làm nhiễm dữ liệu bệnh án chính thức, dễ áp chính sách bảo mật riêng (chỉ `PetOwner` được xem) và dễ truy vấn lịch sử riêng theo index `{ petId: 1, createdAt: -1 }`.
- **Code chứng minh:** `src/models/AiAssessment.js:42-46, 83-86`.

### Q7: Với dữ liệu hóa đơn 12 tháng tăng lớn, làm sao truy vấn báo cáo doanh thu không bị full collection scan?
- **Trả lời (30s):** Trên `Invoice` em đã tạo Compound Index `{ paymentStatus: 1, paidAt: -1 }` đúng theo thứ tự lọc của Aggregation Pipeline (`$match: { paymentStatus: 'Paid', paidAt: { $gte, $lte } }`). MongoDB lọc thẳng trên nhánh `Paid` và quét dải `paidAt` bằng Index Range Scan thay vì đọc toàn bộ bảng.
- **Code chứng minh:** `src/models/Invoice.js:49-51`.

### Q8: `CommunityPost` lưu `likeCount` và `commentCount` trực tiếp trên bài viết — có rủi ro lệch số đếm không?
- **Trả lời (30s):** Em denormalize 2 biến đếm này để màn hình bảng tin không phải chạy `countDocuments` trên từng bài (tránh N+1 query). Khi Like/Unlike hay Comment/Delete, service cập nhật đồng bộ `likeCount`/`commentCount` bằng toán tử nguyên tử `$inc` (hoặc đồng bộ lại từ bảng `PostLike`/`PostComment`) và schema ràng buộc `min: 0` để không bao giờ âm.
- **Code chứng minh:** `src/models/CommunityPost.js:23-24`, `src/models/PostLike.js:11`.
- **Hạn chế & cải thiện:** Hiện tại thao tác tạo `PostLike` và cập nhật `CommunityPost.likeCount` chưa bọc trong `runWithTransaction`; có thể bọc thêm transaction hoặc job đối soát định kỳ nếu traffic cực lớn.
