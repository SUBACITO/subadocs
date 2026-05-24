# NestJS + BullMQ + Redis

> Stack: `@nestjs/bullmq` · `bullmq` · `ioredis` · `@bull-board/nestjs`

---

## Docs

- [[01 - Setup & Config]]
- [[02 - Producer & Consumer]]
- [[03 - Job Options & Lifecycle]]
- [[04 - Cron & Delayed Jobs]]
- [[05 - Flow (Parent-Child)]]
- [[06 - Events & DLQ]]
- [[07 - Bull Board & Docker]]

## Install

```bash
npm install @nestjs/bullmq bullmq ioredis
npm install @bull-board/nestjs @bull-board/api @bull-board/express
```

## Queue Names — đặt 1 chỗ, dùng mọi nơi

```ts
// queue.constants.ts
export const Q = {
  EMAIL:  'email',
  REPORT: 'report',
} as const;

export const EMAIL_JOBS = {
  WELCOME:        'send-welcome',
  RESET_PASSWORD: 'send-reset-password',
} as const;
```