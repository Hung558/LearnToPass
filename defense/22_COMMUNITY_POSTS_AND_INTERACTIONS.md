# 22 — Cộng Đồng, Chia Sẻ Hành Trình Ẩn Thông Tin Nhạy Cảm & Tương Tác (`/api/community-posts`)

> **Tóm tắt nói trong 20 giây:** Gồm **16 endpoint** phục vụ 3 luồng: (1) Bảng tin công khai `Published` cho cả `Guest` (chỉ lộ tên tác giả, ẩn SĐT/email/`moderationNote`), (2) `PetOwner` đăng/sửa/xóa mềm bài viết (ép `PendingReview` chờ duyệt, ảnh bắt buộc upload `multipart/form-data` lên Cloudinary) & **Chia sẻ hành trình tự động cắt bỏ SOAP/đơn thuốc/giá tiền (`BR-18`)**, và (3) Tương tác: Thu hồi bài đã đăng (`UC-50`), Like toggle chống trùng (`BR-19`), Bình luận tối đa **1 cấp reply (`BR-19`)**, và Báo cáo vi phạm (`UC-56`).

---

## 1. Mở nhanh code trong VS Code (Code Map)

| Tầng | File | Chức năng |
| :--- | :--- | :--- |
| **Route** | `src/routes/communityPost.routes.js` (`L1-38`) | 3 route public (đặt trước `/:id`) + 13 route `PetOwner` (`uploadCommunityPostPhotos` trên POST/PUT) |
| **Validators** | `communityPost.validator.js`, `communityInteraction.validator.js` | Validate bài viết, `timelineItems`, `comment`, `report` |
| **Controllers** | `communityPost.controller.js`, `communityInteraction.controller.js` | Cả 2 đẩy lỗi qua `next(err)` về Global Error Handler |
| **Services** | `communityPost.service.js` (`L1-340`), `communityInteraction.service.js` (`L1-288`) | CRUD + `TimelineShare` (`BR-18`) + Like/Comment/Report/Withdraw (`BR-19`) |
| **Models (5)** | `CommunityPost`, `PostLike`, `PostComment`, `ContentReport`, `Pet` | Unique `(postId, userId)` trên `PostLike`; Unique `(reporterId, targetType, targetId)` trên `ContentReport` |
| **Test** | `tests/communityPost.test.js` | 376 dòng test cả bài viết, `BR-18`, Like, Reply 1 cấp, Report, Withdraw |

---

## 2. Chi tiết 16 API — Luồng code & Toàn bộ Case

### 2.1. Nhóm Public — Không cần đăng nhập (`Guest` + `PetOwner`)
- `GET /public` (`post.service.js:296-316`): lọc `status: 'Published'`, `postType?`, phân trang; `.select('-moderationNote')`, tác giả chỉ lấy `fullName`, pet chỉ lấy `name species breed photoUrl` (nếu pet đã xóa thì `null`).
- `GET /public/:id` (`L319-328`): chỉ trả bài `Published`; bài chưa duyệt/bị ẩn đều trả **`404`** để không lộ bài bị ẩn.
- `GET /:id/comments` (`interaction.service.js:113-164`): kiểm tra bài `Published` (`404`), phân trang bình luận gốc mới nhất trước, dùng **1 query `$in: rootIds`** gom toàn bộ reply cấp 1 (xếp cũ → mới) tránh N+1.

### 2.2. Nhóm Chia sẻ Hành trình Tự động ẩn dữ liệu nhạy cảm (`UC-49`, `BR-18` — `PetOwner`)
- `GET /timeline-share/preview?petId=...` (`post.service.js:254-257`) & `POST /timeline-share` (`L262-281`):
  1. Gọi `getShareablePetHistory` (`petTimeline.service.js:470-540`): kiểm tra thú cưng thuộc `PetOwner` (`403` nếu khác chủ, `404` nếu không tồn tại/đã xóa), chỉ đọc tối đa 50 đơn `Done` và dịch vụ `Done` (tuyệt đối không đọc SOAP, đơn thuốc, hóa đơn, `note`/`intake`).
  2. Lớp bảo vệ thứ 2 `toShareableMilestone` (`L207-223`) dựng object mới chỉ chứa `date`, `title` (tên dịch vụ), `imageUrl` (ảnh Grooming `afterPhotoUrl || beforePhotoUrl`).
  3. Nếu truyền `orderIds` có đơn chưa `Done`/không thuộc pet → **`400`** (`L240-244`); thú cưng chưa có đơn `Done` nào → **`400`** (`L265-269`).
  4. Tạo bài `postType: 'TimelineShare'`, ép **`status: 'PendingReview'`** (`L277`) → **`201`**.

### 2.3. Nhóm CRUD Bài viết cá nhân (`PetOwner`)
- `POST /` & `PUT /:id` (`uploadCommunityPostPhotos` bắt buộc `multipart/form-data`, gửi JSON bị **`415`** ở `upload.middleware.js:117`):
  - `attachUploadedImages` (`post.service.js:77-118`): mọi ảnh mới phải upload lên Cloudinary (`vcmos/community-posts`, lỗi → **`502`**); chỉ cho giữ lại `imageUrl` đang có của chính bài đó khi update (gửi URL ngoài tùy ý → **`400`**, chống chèn link độc hại).
  - `POST /` luôn ép `status: 'PendingReview'` (`L127`). `PUT /:id` & `DELETE /:id` dùng điều kiện nguyên tử `{ _id, authorId, status: 'PendingReview' }` (`L171-175, 183-187`): không phải bài mình → **`404`**, đã duyệt/ẩn/gỡ → **`409`**.
- `GET /` & `GET /:id`: xem bài viết của chính mình ở mọi trạng thái.

### 2.4. Nhóm Tương tác & Báo cáo (`PetOwner` — `communityInteraction.service.js`)
- **`PATCH /:id/withdraw`** (`UC-50`, `L47-73`): cho phép chủ bài tự thu hồi (`Removed`) kể cả khi bài đang **`Published`** hoặc `PendingReview`. Chặn **`409`** nếu bài đang bị Admin ẩn (`Hidden`) hoặc đã `Removed`.
- **`POST /:id/like` & `GET /:id/like-status`** (`UC-53`, `BR-19`, `L79-106`): chỉ trên bài `Published` (`404`). Toggle bằng `PostLike.findOneAndDelete` → nếu không có thì `PostLike.create` (chạm `11000` khi click đúp thì nuốt lỗi an toàn, `L90-93`) → đồng bộ lại `likeCount` bằng `countDocuments` (`L17-21`) nên không bao giờ lệch số đếm.
- **`POST /:id/comments`** (`UC-54`, `UC-55`, `BR-19`, `L170-208`): nếu có `parentCommentId`, kiểm tra bình luận cha thuộc đúng bài & `Published` (`404`); nếu bình luận cha **đã có `parentCommentId` (tức đã là cấp 1)** → chặn **`400`** `'Chỉ hỗ trợ trả lời bình luận tối đa 1 cấp (BR-19)'` (`L187-191`). Tạo xong gọi `syncPostCommentCount` → **`201`**.
- **`DELETE /comments/:commentId`** (`L213-232`): chỉ đúng tác giả (`403`), đang `Published` (`409`) → xóa mềm sang `Deleted` + đồng bộ `commentCount` → **`200`**.
- **`POST /reports`** (`UC-56`, `L239-277`): `targetType: 'Post' | 'Comment'` phải đang `Published` (`404`); **chặn tự báo cáo nội dung của mình (`400`, `L247-249, 257-259`)**; trùng báo cáo `11000` → **`409`** (`L272-274`) → **`201`**.

---

## 3. Bảng tổng hợp Case (Tốt → Fail)

| Endpoint | Case | HTTP | Message / Kết quả | Code ở đâu |
| :--- | :--- | :-: | :--- | :--- |
| `POST /`, `/timeline-share`, `/:id/comments`, `/reports` | Tạo thành công | **`201`** | Dữ liệu tạo mới | `communityPost/Interaction.controller.js` |
| `POST /`, `PUT /:id` | Không gửi `multipart/form-data` | **`415`** | `'Chỉ chấp nhận multipart/form-data'` | `upload.middleware.js:117` |
| `POST /`, `PUT /:id`, `timeline-share`, `comments`, `reports` | URL ảnh ngoài / `petId` không thuộc mình / `orderIds` chưa `Done` / reply cấp 2 (`BR-19`) / tự báo cáo mình | **`400`** | Thông báo chi tiết + `errors` | `post.service.js:44,96,241,266`, `interaction.service.js:188,248,258` |
| `timeline-share`, `DELETE /comments/:id`, mọi route `PetOwner` | Role khác `PetOwner` / chia sẻ pet chủ khác (`BR-23`) / xóa comment người khác | **`403`** | Chi tiết quyền bị chặn | `routes.js:17`, `petTimeline.service.js:479`, `interaction.service.js:222` |
| Các API `/:id` | Không thấy bài/comment (hoặc bài chưa `Published` ở public/tương tác) | **`404`** | `'Không tìm thấy bài viết/bình luận...'` | `post.service.js:53,160,326`, `interaction.service.js:36,53,183,219,245,255` |
| `PUT/DELETE /:id`, `withdraw`, `DELETE comment`, `POST /reports` | Sửa/xóa bài không còn `PendingReview` / thu hồi bài `Hidden` hoặc `Removed` / báo cáo trùng (`11000`) | **`409`** | Chi tiết xung đột | `post.service.js:54`, `interaction.service.js:56,59,69,225,273` |
| `POST /`, `PUT /:id` | Lỗi upload Cloudinary | **`502`** | `'Không thể tải ảnh lên, vui lòng thử lại sau'` | `post.service.js:31` |

---

## 4. Câu hỏi hội đồng hay xoáy (Defense Q&A)

1. **Làm sao chứng minh tính năng Chia sẻ hành trình (`BR-18`) không bao giờ làm lộ bệnh án SOAP, đơn thuốc hay giá tiền?**
   - **Trả lời:** Bảo vệ 2 lớp: (1) `getShareablePetHistory` (`petTimeline.service.js:470-540`) chỉ truy vấn 3 bảng (`Pet`, `AppointmentOrder`, `AppointmentService`) và chỉ `.select()` đúng `code, date, name, beforePhotoUrl, afterPhotoUrl` — **không hề truy vấn `MedicalRecord` hay `Invoice`**; (2) `toShareableMilestone` (`communityPost.service.js:207-223`) dựng lại object mới hoàn toàn, và bài tạo ra vẫn phải qua trạng thái `PendingReview` để Admin kiểm duyệt.
2. **Tại sao không cho `PetOwner` tự thu hồi (`withdraw`) bài đang ở trạng thái `Hidden`?**
   - **Trả lời:** Bài `Hidden` là bài bị Admin từ chối hoặc ẩn do vi phạm quy tắc cộng đồng. Không cho chủ bài chuyển sang `Removed` (`communityInteraction.service.js:58-60`) nhằm bảo toàn lịch sử vi phạm và bằng chứng kiểm duyệt của Admin.
3. **Nếu 10 người cùng bấm Like 1 bài viết tại cùng 1 giây, `likeCount` có bị sai số không?**
   - **Trả lời:** Không, vì `PostLike` có Unique Index `(postId, userId)` chống trùng, và `syncPostLikeCount` (`L17-21`) đếm trực tiếp từ `PostLike.countDocuments({ postId })` thay vì chỉ cộng trừ `likeCount + 1` mù quáng.
