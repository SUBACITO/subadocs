# 05 — Flow (Parent-Child)

## Khi nào dùng

Khi cần **pipeline nhiều bước**, parent chỉ complete sau khi **tất cả children** xong.

```
process-order  ← parent (chạy sau cùng)
  ├── send-confirmation-email
  ├── update-inventory
  └── push-notification
```

---

## Setup

```ts
// order.module.ts
@Module({
  imports: [
    BullModule.registerQueue(
      { name: Q.EMAIL },
      { name: Q.ORDER },
    ),
    BullModule.registerFlowProducer({ name: 'order-flow' }),
  ],
  providers: [OrderFlowService, OrderProcessor],
})
export class OrderModule {}
```

---

## FlowProducer

```ts
@Injectable()
export class OrderFlowService {
  constructor(
    @InjectFlowProducer('order-flow')
    private readonly flow: FlowProducer,
  ) {}

  async createOrderFlow(orderId: string) {
    await this.flow.add({
      name: 'process-order',
      queueName: Q.ORDER,
      data: { orderId },

      children: [
        {
          name: 'send-confirmation-email',
          queueName: Q.EMAIL,
          data: { orderId },
          opts: { attempts: 5 },
        },
        {
          name: 'update-inventory',
          queueName: Q.ORDER,
          data: { orderId },
        },
      ],
    });
  }
}
```

---

## Đọc kết quả children trong Parent

```ts
@Processor(Q.ORDER)
export class OrderProcessor extends WorkerHost {
  async process(job: Job) {
    if (job.name === 'process-order') {
      // Lấy return value của tất cả children
      const childResults = await job.getChildrenValues();
      // { 'email:send-confirmation-email': { sent: true }, ... }
      return { done: true, childResults };
    }
  }
}
```

---

> [!note]
> 
> - Children **chạy trước**, parent **chạy sau cùng**
> - Không thể thêm children vào job đang `active`
> - Dùng `addBulk()` để tạo nhiều flows cùng lúc