# 07 — Bull Board & Docker

## Bull Board Setup

```ts
// app.module.ts
@Module({
  imports: [
    BullBoardModule.forRoot({
      route: '/queues',
      adapter: ExpressAdapter,
    }),
    BullBoardModule.forFeature({ name: Q.EMAIL,  adapter: BullMQAdapter }),
    BullBoardModule.forFeature({ name: Q.REPORT, adapter: BullMQAdapter }),

    EmailModule,
    ReportModule,
  ],
})
```

### Bảo vệ route `/queues`

```ts
// main.ts
app.use('/queues', (req, res, next) => {
  if (req.headers['x-api-key'] !== process.env.QUEUE_BOARD_KEY) {
    return res.status(401).json({ message: 'Unauthorized' });
  }
  next();
});
```

> [!warning] Production Đừng expose `/queues` ra internet. Đặt sau VPN hoặc IP whitelist ở Nginx.

---

## `docker-compose.yml`

```yaml
services:
  app:
    build: .
    ports: ['3000:3000']
    environment:
      REDIS_HOST: redis
      REDIS_PORT: 6379
    depends_on:
      redis:
        condition: service_healthy

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data
      - ./redis.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
      interval: 10s
      retries: 5

volumes:
  redis-data:
```

---

## `redis.conf` tối thiểu

```conf
# Persistence
appendonly yes
appendfsync everysec

# Memory
maxmemory 512mb
maxmemory-policy allkeys-lru

# Performance
maxRetriesPerRequest null
```

---

## `Dockerfile`

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
USER node
EXPOSE 3000
CMD ["node", "dist/main"]
```

---

## Redis CLI hay dùng

```bash
# Vào Redis CLI
docker compose exec redis redis-cli

# Monitor tất cả commands
docker compose exec redis redis-cli monitor

# Xem memory usage
docker compose exec redis redis-cli info memory

# Xem tất cả keys của BullMQ
docker compose exec redis redis-cli keys "bull:*"

# Flush (dev only!)
docker compose exec redis redis-cli flushdb
```