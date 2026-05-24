# 06 — Events & DLQ

## QueueEvents — lắng nghe từ bất kỳ process nào

```ts
@Injectable()
export class QueueEventsService implements OnModuleInit, OnModuleDestroy {
  private events: QueueEvents[] = [];

  onModuleInit() {
    const conn = {
      host: process.env.REDIS_HOST,
      port: +process.env.REDIS_PORT!,
    };

    [Q.EMAIL, Q.REPORT].forEach(name => {
      const qe = new QueueEvents(name, { connection: conn });

      qe.on('completed', ({ jobId }) =>
        console.log(`✅ [${name}] #${jobId} completed`),
      );
      qe.on('failed', ({ jobId, failedReason }) =>
        console.error(`❌ [${name}] #${jobId} — ${failedReason}`),
      );
      qe.on('stalled', ({ jobId }) =>
        console.warn(`⚠️ [${name}] #${jobId} stalled`),
      );

      this.events.push(qe);
    });
  }

  async onModuleDestroy() {
    await Promise.all(this.events.map(e => e.close()));
  }
}
```

---

## Dead Letter Queue (DLQ)

BullMQ không có DLQ built-in — tự implement bằng `@OnWorkerEvent('failed')`.

### 1. Tạo queue DLQ

```ts
export const DLQ = 'dead-letter-queue';

// dlq.module.ts
BullModule.registerQueue({ name: DLQ })
```

### 2. Move vào DLQ khi hết retry

```ts
// Trong processor bất kỳ
@OnWorkerEvent('failed')
async onFailed(job: Job, err: Error) {
  const maxAttempts = job.opts.attempts ?? 1;

  if (job.attemptsMade >= maxAttempts) {
    await this.dlqQueue.add('failed-job', {
      originalQueue: job.queueName,
      originalName:  job.name,
      originalData:  job.data,
      reason:        err.message,
      failedAt:      Date.now(),
    });
  }
}
```

### 3. Replay từ DLQ

```ts
async replayFromDlq(dlqJobId: string) {
  const dlqJob = await this.dlqQueue.getJob(dlqJobId);
  if (!dlqJob) throw new NotFoundException();

  const { originalQueue, originalName, originalData } = dlqJob.data;

  // Tạo queue tạm để add lại job
  const q = new BullQueue(originalQueue, { connection: { ... } });
  await q.add(originalName, originalData, { attempts: 3 });
  await q.close();
  await dlqJob.remove();
}
```

---

## Metrics nhanh

```ts
const metrics = await Promise.all([
  queue.getWaitingCount(),
  queue.getActiveCount(),
  queue.getCompletedCount(),
  queue.getFailedCount(),
  queue.getDelayedCount(),
]);
// [waiting, active, completed, failed, delayed]
```