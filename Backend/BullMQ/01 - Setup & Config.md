# 01 — Setup & Config

## `.env`

```env
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_DB=0
```

---

## `app.module.ts`

```ts
@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),

    BullModule.forRootAsync({
      useFactory: () => ({
        connection: {
          host: process.env.REDIS_HOST ?? 'localhost',
          port: +( process.env.REDIS_PORT ?? 6379),
          password: process.env.REDIS_PASSWORD || undefined,
          maxRetriesPerRequest: null,  // bắt buộc với BullMQ
          enableReadyCheck: false,     // bắt buộc với BullMQ
        },
        defaultJobOptions: {
          attempts: 3,
          backoff: { type: 'exponential', delay: 5000 },
          removeOnComplete: { count: 100 },
          removeOnFail:     { count: 500 },
        },
      }),
    }),

    EmailModule,
  ],
})
export class AppModule {}
```

> [!warning] 2 option bắt buộc `maxRetriesPerRequest: null` và `enableReadyCheck: false` — thiếu là lỗi ngay.

---

## Feature Module

```ts
// email.module.ts
@Module({
  imports: [
    BullModule.registerQueue({ name: Q.EMAIL }),
  ],
  providers: [EmailProducer, EmailProcessor],
  exports: [EmailProducer],
})
export class EmailModule {}
```