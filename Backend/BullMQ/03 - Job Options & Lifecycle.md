# 03 — Job Options & Lifecycle

## Trạng thái

```
add() → waiting → active → completed
                    ↓ (throw)
                  failed ──→ retry (waiting + delay)
                    ↓ (hết attempts)
                  failed (final) → DLQ
```

---

## `JobsOptions` hay dùng

```ts
await queue.add('job-name', data, {
  // Retry
  attempts: 5,
  backoff: { type: 'exponential', delay: 2000 },
  // delay:  2s → 4s → 8s → 16s → 32s

  // Timing
  delay: 60_000,         // chạy sau 60 giây
  priority: 1,           // 1 = cao nhất

  // Chống duplicate
  jobId: `order-${orderId}`,  // skip nếu jobId đã tồn tại trong queue

  // Cleanup
  removeOnComplete: { count: 100, age: 86400 }, // giữ 100 job hoặc 24h
  removeOnFail:     { count: 500 },

  // Timeout
  ttl: 24 * 60 * 60 * 1000,  // tự xóa sau 24h nếu chưa xử lý
});
```

---

## Backoff types

|Type|Pattern|Dùng khi|
|---|---|---|
|`exponential`|2s → 4s → 8s...|Default, phổ biến nhất|
|`fixed`|5s → 5s → 5s|External API ổn định|

---

## Retry chọn lọc

```ts
// Phân biệt lỗi có thể retry hay không
export class NonRetriableError extends Error {}

// Trong handler:
try {
  await callExternalApi();
} catch (err) {
  if (err instanceof NonRetriableError) {
    await job.discard(); // bỏ qua retry, vào failed luôn
  }
  throw err; // các lỗi khác → BullMQ tự retry
}
```

---

## Kiểm tra job

```ts
const job = await queue.getJob(jobId);
const state = await job?.getState();
// 'waiting' | 'active' | 'completed' | 'failed' | 'delayed'

const result      = job?.returnvalue;
const failReason  = job?.failedReason;
```

---

## Xóa job

```ts
await job.remove();                           // 1 job
await queue.clean(0, 0, 'failed');            // tất cả failed
await queue.clean(86400 * 1000, 100, 'completed'); // completed cũ hơn 24h
```