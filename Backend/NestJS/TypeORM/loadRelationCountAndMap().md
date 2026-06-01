## 1. Bản chất `.loadRelationCountAndMap()` là gì?

Hàm này có nhiệm vụ:

1. **Đếm (COUNT)** số lượng bản ghi trong một mối quan hệ (relation).
    
2. **Ánh xạ (Map)** kết quả đếm được thành một thuộc tính mới (custom property) nằm trực tiếp trong Object kết quả trả về, dù thuộc tính đó không hề tồn tại trong Database Column của Entity.
    

## 2. Phân tích chi tiết 2 câu lệnh của bạn

### Câu lệnh 1: Đếm tổng số like

TypeScript

```
.loadRelationCountAndMap('feed.likeCount', 'feed.likes')
```

- **`feed.likeCount`**: Bạn đang ra lệnh cho TypeORM: _"Hãy tạo ra một biến tên là `likeCount` nằm trong đối tượng `feed` trả về"_.
    
- **`feed.likes`**: Đây là mối quan hệ (Relation) được định nghĩa trong Entity `Feed`. TypeORM sẽ chạy một câu lệnh `COUNT` trên bảng `like` dựa theo mối quan hệ này.
    
- **Kết quả:** Mỗi bài viết trả về sẽ tự động có thêm một trường: `feed.likeCount = 15` (ví dụ bài viết có 15 likes).
### Câu lệnh 2: Kiểm tra User hiện tại đã like chưa (Trả về số `1` hoặc `0`)

TypeScript

```
.loadRelationCountAndMap('feed.isLikedByCurrentUser', 'feed.likes', 'like', (qb) => qb.andWhere('like.userId = :userId', { userId }))
```

Đoạn này nâng cao hơn vì nó dùng thêm **điều kiện lọc (Query Builder phụ)**:

- **`feed.isLikedByCurrentUser`**: Tạo một biến tên là `isLikedByCurrentUser` trong đối tượng `feed`.
    
- **`feed.likes`**: Vẫn là đếm dựa trên mối quan hệ `likes`.
    
- **`'like'`**: Đặt bí danh (Alias) cho bảng `likes` trong câu lệnh đếm này để lát nữa viết điều kiện `WHERE`.
    
- **`(qb) => qb.andWhere(...)`**: Đây là phần cốt lõi. Bạn bảo SQL rằng: _"Đừng đếm hết! Chỉ đếm những dòng nào trong bảng Like mà có `userId` bằng với `userId` của người đang xem feed này thôi!"_.
    
- **Kết quả:** * Nếu user đó **đã like** bài viết $\rightarrow$ SQL đếm được **`1`** $\rightarrow$ `feed.isLikedByCurrentUser = 1`.
    
    - Nếu user đó **chưa like** $\rightarrow$ SQL đếm được **`0`** $\rightarrow$ `feed.isLikedByCurrentUser = 0`.
        
    - _(Ở Front-end, họ chỉ cần check `likeCount > 0` hoặc ép kiểu về Boolean là biết nút Like có cần sáng lên hay không)._