
# Luồng hoạt động của Transaction trong PostgreSQL (chuẩn PostgreSQL)
## Cú pháp tổng quát
```sql
BEGIN;  -- hoặc START TRANSACTION;
-- Thiết lập mức độ cô lập (phải đặt ngay sau BEGIN, trước query đầu tiên)
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- Các câu lệnh SQL
SELECT ... FROM ... WHERE ... FOR UPDATE;
UPDATE ... SET ... WHERE ...;
INSERT INTO ... VALUES ...;
DELETE FROM ... WHERE ...;
-- Kết thúc
COMMIT;  -- hoặc ROLLBACK;

## Các bước chi tiết đúng chuẩn PostgreSQL

### 1. Bắt đầu transaction

sql

BEGIN;                      -- Cách phổ biến nhất trong PostgreSQL
-- hoặc
START TRANSACTION;          -- Chuẩn SQL, PostgreSQL hỗ trợ

### 2. Thiết lập mức độ cô lập

sql

-- Đặt ngay sau BEGIN
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;     -- Mặc định của PostgreSQL
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- LƯU Ý: PostgreSQL không hỗ trợ READ UNCOMMITTED
-- READ UNCOMMITTED trong PG hoạt động như READ COMMITTED

### 3. Các dạng FOR UPDATE trong PostgreSQL

sql

-- Khóa hàng để cập nhật (cấm đọc, cấm ghi)
SELECT * FROM users WHERE id = 1 FOR UPDATE;
-- Khóa hàng nhưng không cấm cập nhật khóa chính (NO KEY)
SELECT * FROM users WHERE id = 1 FOR NO KEY UPDATE;
-- Khóa chia sẻ (cho phép đọc, cấm ghi/xóa)
SELECT * FROM users WHERE id = 1 FOR SHARE;
-- Khóa chia sẻ nhưng không khóa cập nhật khóa chính
SELECT * FROM users WHERE id = 1 FOR KEY SHARE;
-- Thêm NOWAIT hoặc SKIP LOCKED
SELECT * FROM users WHERE id = 1 FOR UPDATE NOWAIT;
SELECT * FROM users WHERE id = 1 FOR UPDATE SKIP LOCKED;

### 4. Kết thúc transaction

sql

COMMIT;                     -- Lưu thay đổi
COMMIT WORK;                -- Tương đương COMMIT
ROLLBACK;                   -- Hủy thay đổi
ROLLBACK WORK;              -- Tương đương ROLLBACK
END;                        -- Tương đương COMMIT trong PostgreSQL

## 📋 Ví dụ thực tế

sql

-- Ví dụ 1: Chuyển tiền giữa 2 tài khoản
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
SELECT balance FROM accounts WHERE id = 2 FOR UPDATE;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

sql

-- Ví dụ 2: Tránh deadlock với NOWAIT
BEGIN;
SELECT * FROM inventory WHERE product_id = 42 FOR UPDATE NOWAIT;
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 42;
COMMIT;

## ⚠️ Đặc điểm quan trọng của PostgreSQL

|Đặc điểm|Giải thích|
|---|---|
|**MVCC**|PostgreSQL dùng Multi-Version Concurrency Control, không dùng lock toàn bảng|
|**READ UNCOMMITTED**|Không hỗ trợ, hoạt động như READ COMMITTED|
|**REPEATABLE READ**|Chống dirty read, non-repeatable read nhưng không chống phantom read|
|**SERIALIZABLE**|Chống tất cả anomalies, có thể báo lỗi `could not serialize access` khi xung đột|
|**FOR UPDATE**|Tạo row-level lock ngay lập tức, không phải đến UPDATE mới khóa|
|**NOWAIT**|Không chờ, báo lỗi ngay nếu không khóa được|
|**SKIP LOCKED**|Bỏ qua các hàng đã bị khóa, chỉ trả về hàng chưa khóa|

## 🧪 Kiểm tra transaction hiện tại

sql

-- Xem thông tin transaction hiện tại
SELECT txid_current();
-- Xem các transaction đang chạy
SELECT * FROM pg_stat_activity WHERE state = 'active';

## 📌 Ghi nhớ nhanh

text

BEGIN
  ├── SET TRANSACTION ISOLATION LEVEL ...
  ├── SELECT ... FOR UPDATE/SHARE ...
  ├── INSERT/UPDATE/DELETE ...
  └── COMMIT (hoặc ROLLBACK nếu có lỗi)