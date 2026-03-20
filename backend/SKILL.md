---
name: backend
description: >
  Expert backend development skill. Use for ANY backend question: HTTP API design
  (REST, архитектура эндпоинтов), Bun/Node.js сервисы (Hono, Elysia, Express, Fastify),
  Go HTTP сервисы (chi, net/http), базы данных (DrizzleORM, pgx, sqlc, Prisma),
  PostgreSQL (схема, индексы, запросы, миграции), Redis (кэш, очереди, pub/sub),
  аутентификация (JWT, сессии, OAuth), архитектура (layered, DDD lite, CQRS),
  паттерны (Repository, DI, Result type, Event bus), error handling, валидация (Zod),
  очереди задач (BullMQ), деплой (Docker, systemd). Trigger на: написание API,
  вопросы про структуру проекта, paste бэкенд кода с ошибками, code review,
  "как правильно сделать X на бэке", вопросы про ORM/SQL, кэширование, производительность.
---

# Backend Skill

## Identity & Style

Senior backend-разработчик. Основной стек: Bun + TypeScript. Go — для системных задач.
Пишу явный, типизированный, тестируемый код без магии.

**Правила:**
- Русский, кратко
- Называй проблемы: "это N+1", "тут race condition", "нет валидации входных данных"
- Production-ready: валидация, логирование ошибок, таймауты, graceful shutdown
- Адаптируй глубину: "дай команду" → сниппет; "как организовать" → архитектура с trade-offs

---

## Структура проекта

```
src/
├── index.ts              # entry point — только wire-up зависимостей
├── config.ts             # env через zod
├── db/
│   ├── schema.ts
│   ├── migrations/
│   └── index.ts
├── modules/
│   ├── users/
│   │   ├── users.router.ts
│   │   ├── users.service.ts
│   │   ├── users.repo.ts
│   │   └── users.types.ts
│   └── orders/
├── middleware/
│   ├── auth.ts
│   └── errors.ts
└── lib/
    ├── errors.ts
    └── result.ts
```

---

## Hono — production setup

```typescript
import { Hono } from 'hono'
import { cors } from 'hono/cors'
import { logger } from 'hono/logger'
import { zValidator } from '@hono/zod-validator'
import { z } from 'zod'

type AppEnv = {
  Variables: { userId: string; requestId: string }
}

const app = new Hono<AppEnv>()

app.use('*', logger())
app.use('*', cors({ origin: config.allowedOrigins }))
app.use('*', async (c, next) => {
  c.set('requestId', crypto.randomUUID())
  await next()
})

app.onError((err, c) => {
  if (err instanceof AppError) {
    return c.json({ error: err.message, code: err.code }, err.status as any)
  }
  console.error({ requestId: c.get('requestId'), err })
  return c.json({ error: 'Internal server error' }, 500)
})

const createUserBody = z.object({
  name: z.string().min(1).max(100).trim(),
  email: z.string().email().toLowerCase(),
})

app.post('/users', authMiddleware, zValidator('json', createUserBody), async (c) => {
  const body = c.req.valid('json')
  const user = await userService.create(body)
  return c.json(user, 201)
})
```

---

## Error hierarchy

```typescript
export class AppError extends Error {
  constructor(
    message: string,
    public readonly code: string,
    public readonly status: number = 500,
  ) {
    super(message)
    this.name = this.constructor.name
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string, id?: string) {
    super(id ? `${resource} '${id}' not found` : `${resource} not found`, 'NOT_FOUND', 404)
  }
}
export class ValidationError extends AppError {
  constructor(msg: string, public readonly fields?: Record<string, string>) {
    super(msg, 'VALIDATION_ERROR', 400)
  }
}
export class ConflictError extends AppError {
  constructor(msg: string) { super(msg, 'CONFLICT', 409) }
}
export class UnauthorizedError extends AppError {
  constructor(msg = 'Unauthorized') { super(msg, 'UNAUTHORIZED', 401) }
}
```

---

## Result type

```typescript
export type Result<T, E = AppError> =
  | { ok: true; value: T }
  | { ok: false; error: E }

export const ok = <T>(value: T): Result<T> => ({ ok: true, value })
export const fail = <T>(error: AppError): Result<T> => ({ ok: false, error })

// В сервисе
async function findUser(id: string): Promise<Result<User>> {
  const user = await userRepo.findById(id)
  if (!user) return fail(new NotFoundError('User', id))
  return ok(user)
}

// В хендлере
const result = await findUser(id)
if (!result.ok) throw result.error
return c.json(result.value)
```

---

## DrizzleORM

```typescript
// Schema
export const users = pgTable('users', {
  id: text('id').primaryKey().$defaultFn(() => crypto.randomUUID()),
  name: text('name').notNull(),
  email: text('email').notNull().unique(),
  role: text('role', { enum: ['user', 'admin'] }).notNull().default('user'),
  createdAt: timestamp('created_at').notNull().defaultNow(),
  updatedAt: timestamp('updated_at').notNull().defaultNow(),
})

// Repository
export class UserRepository {
  async findById(id: string) {
    return db.select().from(users).where(eq(users.id, id)).get() ?? null
  }

  async findByEmail(email: string) {
    return db.select().from(users).where(eq(users.email, email)).get() ?? null
  }

  async create(data: typeof users.$inferInsert) {
    return db.insert(users).values(data).returning().get()
  }

  async update(id: string, data: Partial<typeof users.$inferInsert>) {
    return db
      .update(users)
      .set({ ...data, updatedAt: new Date() })
      .where(eq(users.id, id))
      .returning()
      .get()
  }
}

// Транзакция
const user = await db.transaction(async (tx) => {
  const u = await tx.insert(users).values({ name, email }).returning().get()
  await tx.insert(subscriptions).values({ userId: u.id, plan: 'basic', expiresAt })
  return u
})

// Избегай N+1 — используй with
const usersWithSubs = await db.query.users.findMany({
  with: { subscriptions: { where: eq(subscriptions.active, true) } },
})
```

---

## Redis — кэш и очереди

```typescript
// Cache-aside паттерн
async function cachedGet<T>(key: string, ttl: number, fn: () => Promise<T>): Promise<T> {
  const cached = await redis.get(key)
  if (cached) return JSON.parse(cached) as T
  const value = await fn()
  await redis.setex(key, ttl, JSON.stringify(value))
  return value
}

const user = await cachedGet(`user:${id}`, 300, () => userRepo.findById(id))
await userRepo.update(id, data)
await redis.del(`user:${id}`)  // инвалидация

// BullMQ
const emailQueue = new Queue('email', {
  connection: redis,
  defaultJobOptions: { attempts: 3, backoff: { type: 'exponential', delay: 1000 } },
})

const worker = new Worker('email', async (job) => {
  await emailService.send(job.data.userId, job.data.template)
}, { connection: redis, concurrency: 5 })
```

---

## JWT Auth middleware

```typescript
import { SignJWT, jwtVerify } from 'jose'

const SECRET = new TextEncoder().encode(config.jwtSecret)

export async function signToken(payload: { sub: string; role: string }) {
  return new SignJWT(payload)
    .setProtectedHeader({ alg: 'HS256' })
    .setIssuedAt()
    .setExpirationTime('7d')
    .sign(SECRET)
}

export const authMiddleware = createMiddleware<AppEnv>(async (c, next) => {
  const token = c.req.header('Authorization')?.slice(7)
  if (!token) throw new UnauthorizedError()
  try {
    const { payload } = await jwtVerify(token, SECRET)
    c.set('userId', payload.sub as string)
    await next()
  } catch {
    throw new UnauthorizedError('Invalid token')
  }
})
```

---

## Config через zod

```typescript
const configSchema = z.object({
  port: z.coerce.number().default(3000),
  databaseUrl: z.string(),
  redisUrl: z.string().default('redis://localhost:6379'),
  jwtSecret: z.string().min(32),
  logLevel: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
})

const result = configSchema.safeParse(Bun.env)
if (!result.success) {
  console.error('Invalid config:', result.error.flatten().fieldErrors)
  process.exit(1)
}
export const config = result.data
```

---

## Reference Files

- `references/go-backend.md` — Go HTTP сервисы, chi, pgx, sqlc, graceful shutdown
- `references/architecture.md` — слоёная архитектура, DI, event-driven, CQRS
- `references/postgres.md` — индексы, EXPLAIN ANALYZE, оконные функции, migrations