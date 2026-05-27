![[Pasted image 20260527080108.png]]

Tóm gọn lại logic quan trọng nhất:

**Hai URL, hai thế giới:**

`NEXT_PUBLIC_BACKEND_URL` — phải là domain/IP public thật sự vì browser của user sẽ dùng cái này để gọi thẳng. Docker internal không có nghĩa lý gì với browser bên ngoài.

`INTERNAL_BACKEND_URL` (không có `NEXT_PUBLIC_`) — dùng `http://nestjs:3000` (tên container Docker), chỉ có Next.js server dùng khi chạy server-side code.

**Hai kiểu lấy session tương ứng:**

- `getSession()` — chạy ở client (browser), sẽ fetch qua `NEXT_PUBLIC_BACKEND_URL`
- `getServerSession()` — chạy ở server (Next.js container), sẽ fetch qua `INTERNAL_BACKEND_URL` → gọi trực tiếp qua Docker network đến container `nestjs`

**Lỗi hay gặp:**

- Dùng `INTERNAL_BACKEND_URL` trong component client-side → browser không ping được, 502/ECONNREFUSED
- Dùng `NEXT_PUBLIC_BACKEND_URL` trong `getServerSession` → vẫn chạy được nhưng đi vòng ra ngoài rồi vào lại, chậm hơn và phụ thuộc network public không cần thiết
- Quên prefix `NEXT_PUBLIC_` → Next.js không expose ra browser, `undefined` ở client