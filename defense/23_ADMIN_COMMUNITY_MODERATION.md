# 23 — Admin Kiểm Duyệt Cộng Đồng & Xử Lý Báo Cáo Vi Phạm (`/api/admin/community` — `SCR-33`)

> **Tóm tắt nói trong 20 giây:** 4 endpoint dành riêng cho `Admin`: xem hàng đợi bài viết chờ duyệt (xếp FIFO + tự động quét Regex cảnh báo từ khóa y tế/giá tiền `hasSensitiveKeywords`), phê duyệt (`Approve → Published`) hoặc từ chối (`Reject → Hidden`, bắt buộc lý do `MSG22`), xem danh sách báo cáo vi phạm kèm ngữ cảnh và số lượt báo cáo (batch `$in` tránh N+1), và xử lý vi phạm (`Keep | Hide | Delete`) tự động đóng toàn bộ báo cáo `Pending` cùng đối tượng.

---

## 1. Mở nhanh code trong VS Code (Code Map)

| Tầng | File | Chức năng |
| :--- | :--- | :--- |
| **Route** | `src/routes/communityModeration.routes.js` (`L1-15`) | Gắn chung `router.use(protect, authorize('Admin'))` cho cả 4 API |
| **Validator** | `src/validators/communityModeration.validator.js` | Validate query lọc, `decision` (`Approve\|Reject`, `Keep\|Hide\|Delete`), bắt buộc lý do khi Reject/Hide/Delete |
| **Controller** | `src/controllers/communityModeration.controller.js` | 4 handler đẩy lỗi qua `next(err)` về Global Error Handler |
| **Service** | `src/services/communityModeration.service.js` (`L1-293`) | `detectSensitiveKeywords` (`L10-49`), duyệt bài (`L92-121`), list báo cáo batch (`L127-205`), xử lý hàng loạt (`L212-285`) |
| **Models (4)** | `CommunityPost`, `PostComment`, `ContentReport`, `User` | Index `{ status: 1, createdAt: 1 }` trên `ContentReport` |
| **Test** | `tests/communityPost.test.js` | Bao phủ toàn bộ luồng duyệt bài & xử lý báo cáo vi phạm |

---

## 2. Chi tiết 4 API — Luồng code & Toàn bộ Case

### 2.1. `GET /api/admin/community/posts` — Hàng đợi duyệt bài (`UC-57`)
- **Query:** `status?` (`PendingReview` mặc định | `Published` | `Hidden` | `Removed` | `All`), `postType?`, `page`, `limit`.
- **Luồng (`communityModeration.service.js:54-87`):**
  - Bài `PendingReview` xếp **cũ → mới (FIFO công bằng)** (`createdAt: 1`), các trạng thái khác xếp mới cập nhật trước (`updatedAt: -1`).
  - Với mỗi bài, gọi `detectSensitiveKeywords(post)` (`L10-49`) chạy 4 Regex quét cả `content` và `timelineItems[].title` để gắn cờ `hasSensitiveKeywords: boolean` + `sensitiveMatches: string[]` (vd `"500k"`, `"đơn thuốc"`, `"hóa đơn"`, `"250mg"`) giúp Admin soi nhanh vi phạm `BR-18`.

### 2.2. `PATCH /api/admin/community/posts/:id/moderate` — Duyệt / Từ chối bài (`UC-58`)
- **Body:** `{ decision*: 'Approve' | 'Reject', moderationNote? (bắt buộc khi Reject — MSG22) }`.
- **Luồng (`L92-121`):** Kiểm tra bài tồn tại (`404`) và đang **`PendingReview`** (`409` nếu khác). Cập nhật nguyên tử `findOneAndUpdate({ _id: postId, status: 'PendingReview' }, { status: 'Published' | 'Hidden', moderationNote })` chống 2 Admin duyệt đè nhau (`L107-118`) → **`200`**.

### 2.3. `GET /api/admin/community/reports` — Hàng đợi báo cáo vi phạm (`UC-59`)
- **Query:** `status?` (`Pending` mặc định | `Resolved` | `All`), `targetType?` (`Post` | `Comment`), `page`, `limit`.
- **Luồng tối ưu chống N+1 (`L127-205`):** Lấy danh sách báo cáo → tách `postTargetIds` & `commentTargetIds` → chạy song song 3 lệnh gom lô (`CommunityPost.find({ $in })`, `PostComment.find({ $in })`, và `ContentReport.aggregate` đếm tổng `reportCount` trên mỗi nội dung) → ghép bằng `Map` O(1) → **`200`**.

### 2.4. `PATCH /api/admin/community/reports/:id/resolve` — Xử lý báo cáo vi phạm (`UC-59`)
- **Body:** `{ decision*: 'Keep' | 'Hide' | 'Delete', note? (bắt buộc khi Hide hoặc Delete) }`.
- **Luồng (`L212-285`):**
  1. Kiểm tra báo cáo tồn tại (`404`), chưa `Resolved` (`409`, `L217-219`).
  2. Áp quyết định lên đối tượng (`L223-256`):
     - `Post`: `Hide → Hidden`, `Delete → Removed` (kèm ghi `moderationNote: note`), `Keep →` giữ nguyên.
     - `Comment`: `Hide → Hidden`, `Delete → Deleted` và gọi **`syncPostCommentCount(postId)`** cập nhật lại số bình luận trên bài viết (`L248-250`).
  3. **Đóng hàng loạt (`L259-271`):** `ContentReport.updateMany({ targetType, targetId, status: 'Pending' }, { status: 'Resolved', resolvedById: adminId })` — đóng toàn bộ báo cáo đang chờ của cùng nội dung đó trong 1 thao tác, trả về `resolvedCount` → **`200`**.

---

## 3. Bảng tổng hợp Case (Tốt → Fail)

| Endpoint | Case | HTTP | Message / Kết quả | Code ở đâu |
| :--- | :--- | :-: | :--- | :--- |
| Cả 4 API | Thành công | **`200`** | Dữ liệu kiểm duyệt / xử lý | `communityModeration.controller.js` |
| Cả 4 API | Thiếu token / Role không phải `Admin` | **`401`** / **`403`** | `'Chưa đăng nhập'` / `'Bạn không có quyền...'` | `communityModeration.routes.js:8` |
| `moderate`, `resolve`, `GET` | Sai ID, sai `decision`, `Reject`/`Hide`/`Delete` thiếu lý do, sai query | **`400`** | `{ message, errors }` | `communityModeration.validator.js` |
| `moderate`, `resolve` | Không thấy bài viết / báo cáo | **`404`** | `'Không tìm thấy bài viết'` / `'Không tìm thấy báo cáo vi phạm'` | `service.js:95, 215` |
| `moderate`, `resolve` | Bài không còn `PendingReview` (hoặc vừa bị Admin khác duyệt) / Báo cáo đã `Resolved` | **`409`** | Chi tiết xung đột trạng thái | `service.js:98, 117, 218` |

---

## 4. Câu hỏi hội đồng hay xoáy (Defense Q&A)

1. **Nếu 1 bài viết bị 5 người khác nhau báo cáo vi phạm, Admin có phải bấm xử lý 5 lần không?**
   - **Trả lời:** Không. Theo `UC-59`, `listContentReports` (`L169-173`) gộp hiển thị `reportCount: 5`, và khi Admin xử lý 1 báo cáo ở `resolveContentReport` (`L259-271`), hệ thống chạy `ContentReport.updateMany` đóng toàn bộ 5 báo cáo `Pending` cùng trỏ tới bài đó sang `Resolved` cùng lúc.
2. **Cơ chế phát hiện từ khóa nhạy cảm (`detectSensitiveKeywords`) có tự động chặn bài không?**
   - **Trả lời:** Không tự chặn cứng (tránh chặn nhầm oan, ví dụ khách khen "bé ăn hết 200k hạt"), mà gắn cờ cảnh báo `hasSensitiveKeywords` + `sensitiveMatches` (`L27-49`) để tô sáng trên màn hình `SCR-33` hỗ trợ Admin ra quyết định cuối cùng nhanh hơn.
