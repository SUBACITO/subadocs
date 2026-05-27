## `Install`
```bash
$ bun add --save @nestjs/bullmq bullmq
```
## `app.module.ts`

```ts
import { ConfigModule, ConfigService } from '@nestjs/config';
@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    BullModule.forRootAsync({
      useFactory: (config: ConfigService) => ({
        connection: {
          host: config.get<string>('REDIS_HOST'),
		  port: config.get<number>('REDIS_PORT'),
          password: config.get<number>('REDIS_PASSWORD'),
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
  ],
})
export class AppModule {}
```

## `Cấu trúc`

![[Pasted image 20260527162211.png]]
## `Create QueueModule`

```ts
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bullmq';
import { ProfilesProcessor } from './profiles/profiles.processor';
import { ProfilesQueueService } from './profiles/profiles.queue.service';

export const QUEUE_NAMES = {
  PROFILE: 'profiles',
} as const;

@Module({
  imports: [
    BullModule.registerQueue({ 
      name: QUEUE_NAMES.PROFILE 
    }),
  ],
  providers: [
    ProfilesProcessor, 
    ProfilesQueueService
  ],
  exports: [
    ProfilesQueueService
  ],
})
export class QueuesModule {}
```

`profiles.processor.ts`

```ts
import { OnWorkerEvent, Processor, WorkerHost } from '@nestjs/bullmq';
import { Logger } from '@nestjs/common';
import { Job } from 'bullmq';
import { ProfileService } from './profile.service'; // Đảm bảo đã import ProfileService đúng đường dẫn

@Processor('profiles', {
  concurrency: 5,
  limiter: {
    max: 10,
    duration: 1000,
  },
})
export class ProfilesProcessor extends WorkerHost {
  private readonly logger = new Logger(ProfilesProcessor.name);

  constructor(private readonly profileService: ProfileService) {
    super();
  }

  async process(job: Job): Promise<void> {
    switch (job.name) {
      case 'send':
        return await this.handleSend(job);
      default:
        throw new Error(`Unknown job: ${job.name}`);
    }
  }

  private async handleSend(job: Job) {
    // Logic Handle
    // return await this.profileService.doSomeThing();
  }

  @OnWorkerEvent('active')
  onAdded(job: Job) {
    this.logger.log(`Job ${job.id} đang được xử lý`);
  }

  @OnWorkerEvent('completed')
  onCompleted(job: Job, result: any) {
    this.logger.log(`Job ${job.id} hoàn thành thành công. Kết quả: ${JSON.stringify(result)}`);
  }

  @OnWorkerEvent('failed')
  onFailed(job: Job, error: Error) {
    this.logger.error(`Job ${job.id} thất bại sau ${job.attemptsMade} lần thử. Lỗi: ${error.message}`);
  }

  @OnWorkerEvent('progress')
  onProgress(job: Job, progress: number) {
    this.logger.log(`Job ${job.id} đạt ${progress}% tiến trình`);
  }
}
```

`profiles.queue.service.ts`

```ts
import { InjectQueue } from '@nestjs/bullmq';
import { Injectable } from '@nestjs/common';
import { Queue } from 'bullmq';

@Injectable()
export class ProfilesQueueService {
  constructor(
    @InjectQueue('profiles') private readonly profilesQueue: Queue
  ) {}

  async addQueue(data: { data1: string, data2: number, data3: number }) {
    const job = await this.profilesQueue.add('send', data, {
      attempts: 3,           // retry 3 lần nếu fail
      backoff: {
        type: 'exponential',
        delay: 1000,         // 1s, 2s, 4s...
      },
      removeOnComplete: 100, // giữ lại 100 job completed gần nhất
      removeOnFail: 50,
    });

    return job;
  }

  // Delayed job
  // async scheduleNotification(data: any, delayMs: number) {
  //   await this.notificationQueue.add('send', data, { delay: delayMs });
  // }
}
```