# SSE in Next.js – Real-Time Notifications

> Source: [pedroalonso.net](https://www.pedroalonso.net/blog/sse-nextjs-real-time-notifications)

## SSE là gì?

**Server-Sent Events (SSE)** là cơ chế cho phép server đẩy dữ liệu xuống client qua HTTP — **một chiều**, không cần WebSocket.

- Client dùng browser API `EventSource` để lắng nghe
- Server giữ connection mở và gửi dữ liệu theo format `data: {...}\n\n`
- Tự động reconnect khi mất kết nối (built-in)

## SSE vs WebSocket

| | SSE | WebSocket |
|---|---|---|
| Chiều giao tiếp | Server → Client | 2 chiều |
| Protocol | HTTP | `ws://` |
| Reconnect | Tự động | Phải tự xử lý |
| Độ phức tạp | Thấp | Trung bình–Cao |
| Dùng khi | Notification, feed, progress | Chat, game, collaborative |

## Dùng SSE khi nào?

- Chỉ cần server push (không cần client gửi ngược)
- Notification, activity feed, progress bar
- Muốn đơn giản, ít code
- Cần hoạt động qua firewall/proxy

## Cách implement trong Next.js 15

### Server (Route Handler)

```ts
// app/api/sse/route.ts
export const dynamic = 'force-dynamic'

export async function GET(req: NextRequest) {
  const stream = new ReadableStream({
    start(controller) {
      const encoder = new TextEncoder()

      const interval = setInterval(() => {
        controller.enqueue(encoder.encode(`data: ${JSON.stringify(data)}\n\n`))
      }, 1000)

      req.signal.addEventListener('abort', () => {
        clearInterval(interval)
        controller.close()
      })
    }
  })

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache, no-transform',
      'Connection': 'keep-alive',
      'X-Accel-Buffering': 'no', // tắt buffer cho Nginx
    }
  })
}
```

### Client

```ts
const eventSource = new EventSource('/api/sse/...')

eventSource.onmessage = (event) => {
  const data = JSON.parse(event.data)
  // xử lý data
}

// cleanup
return () => eventSource.close()
```

## Các use case trong bài

- **Notification system** – in-memory service, broadcast đến tất cả listeners
- **Progress tracker** – theo dõi long-running task theo `taskId`
- **Activity feed** – stream sự kiện người dùng real-time

## Production checklist

- [ ] Giới hạn số connection per IP
- [ ] Gửi heartbeat mỗi 30s để giữ connection
- [ ] Dùng Redis pub/sub thay in-memory nếu multi-instance
- [ ] Auth trước khi mở stream
- [ ] Cấu hình Nginx: `proxy_buffering off`, `proxy_read_timeout 86400s`
- [ ] Reconnect logic phía client nếu cần custom behavior