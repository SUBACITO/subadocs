# 02 — Producer & Consumer

## Job DTO

```ts
// email.job.dto.ts
export interface SendWelcomeJobDto {
  to: string;
  name: string;
  correlationId?: string;
}
```

---

## Producer

> Rule: Controller/Service gọi Producer — **không** inject `Queue` trực tiếp.

```ts
@Injectable()
export class EmailProducer {
  constructor(
    @InjectQueue(Q.EMAIL) private readonly queue: Queue,
  ) {}

  async sendWelcome(data: SendWelcomeJobDto, opts?: JobsOptions) {
    return this.queue.add(EMAIL_JOBS.WELCOME, data, {
      jobId: `welcome-${data.to}`,   // chống duplicate
      priority: 1,
      ...opts,
    });
  }

  async sendBulk(list: SendWelcomeJobDto[]) {
    return this.queue.addBulk(
      list.map(data => ({ name: EMAIL_JOBS.WELCOME, data })),
    );
  }
}
```

---

## Consumer (Processor)

```ts
@Processor(Q.EMAIL, {
  concurrency: 5,               // xử lý song song tối đa 5 jobs
  limiter: { max: 10, duration: 1000 }, // rate limit: 10 jobs/giây
})
export class EmailProcessor extends WorkerHost {
  private readonly logger = new Logger(EmailProcessor.name);

  // Dispatcher — phân loại theo job.name
  async process(job: Job): Promise<unknown> {
    switch (job.name) {
      case EMAIL_JOBS.WELCOME:        return this.handleWelcome(job);
      case EMAIL_JOBS.RESET_PASSWORD: return this.handleResetPassword(job);
      default: throw new Error(`Unknown job: ${job.name}`);
    }
  }

  private async handleWelcome(job: Job<SendWelcomeJobDto>) {
    await job.updateProgress(10);
    // ... logic gửi mail
    await job.updateProgress(100);
    return { sent: true };
  }

  private async handleResetPassword(job: Job) {
    // ...
    return { sent: true };
  }

  // Worker Events
  @OnWorkerEvent('completed')
  onCompleted(job: Job) {
    this.logger.log(`✅ #${job.id} ${job.name}`);
  }

  @OnWorkerEvent('failed')
  onFailed(job: Job, err: Error) {
    this.logger.error(`❌ #${job.id} attempt ${job.attemptsMade} — ${err.message}`);
  }
}
```

> [!tip] Luôn `throw` error trong handler Nếu catch xong nuốt lỗi → BullMQ không retry. Chỉ catch để transform rồi rethrow.

---

## Concurrency nên đặt bao nhiêu?

| Loại task                  | `concurrency` |
| -------------------------- | ------------- |
| HTTP / Email call          | 10–20         |
| DB query                   | 5–10          |
| File I/O                   | 2–5           |
| CPU heavy (PDF, report)    | 1–2           |
| External API có rate limit | 1 + `limiter` |