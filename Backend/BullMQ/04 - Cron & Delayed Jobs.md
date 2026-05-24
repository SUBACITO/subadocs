# 04 — Cron & Delayed Jobs

## Delayed (1 lần)

```ts
// Gửi email sau 24h
await queue.add('send-reminder', data, {
  delay: 24 * 60 * 60 * 1000,
});

// Chạy vào thời điểm cụ thể
await queue.add('send-newsletter', data, {
  delay: new Date('2025-06-01T08:00:00+07:00').getTime() - Date.now(),
});
```

---

## Repeatable (Cron)

```ts
await queue.add('generate-report', data, {
  repeat: {
    pattern: '0 8 * * 1',         // Mỗi thứ 2 lúc 8AM
    tz: 'Asia/Ho_Chi_Minh',
  },
  jobId: 'weekly-report',          // unique ID, tránh duplicate khi restart
});
```

### Cron cheat sheet

```
* * * * *
│ │ │ │ └── ngày trong tuần (0=CN, 1=T2...)
│ │ │ └──── tháng (1-12)
│ │ └────── ngày trong tháng (1-31)
│ └──────── giờ (0-23)
└────────── phút (0-59)

0 8 * * *       → mỗi ngày 8:00
0 8 * * 1-5     → T2-T6 lúc 8:00
0 9,18 * * *    → 9:00 và 18:00 mỗi ngày
*/15 * * * *    → mỗi 15 phút
0 8 1 * *       → ngày 1 hàng tháng
```

---

## Đăng ký đúng cách — dùng `OnApplicationBootstrap`

```ts
@Injectable()
export class SchedulerBootstrap implements OnApplicationBootstrap {
  constructor(
    @InjectQueue(Q.REPORT) private readonly queue: Queue,
  ) {}

  async onApplicationBootstrap() {
    // Xóa jobs cũ trước khi đăng ký lại
    // (quan trọng khi thay đổi cron pattern)
    const existing = await this.queue.getRepeatableJobs();
    await Promise.all(existing.map(j => this.queue.removeRepeatableByKey(j.key)));

    await this.queue.add('generate-report', {}, {
      repeat: { pattern: '0 8 * * 1', tz: 'Asia/Ho_Chi_Minh' },
      jobId: 'weekly-report',
    });
  }
}
```

> [!warning] Không gọi `queue.add` với repeat trong constructor DI chưa init xong → Redis chưa sẵn sàng. Dùng `OnApplicationBootstrap`.

---

## Quản lý Repeatable Jobs

```ts
const jobs = await queue.getRepeatableJobs();
// [{ key, name, pattern, tz, next }]

await queue.removeRepeatableByKey(jobs[0].key);
```