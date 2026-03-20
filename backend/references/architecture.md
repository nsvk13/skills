# Архитектура Backend — паттерны и принципы

## Слоёная архитектура (основной выбор)

```
Router/Handler  →  Service  →  Repository  →  Database
     ↑                ↑             ↑
  HTTP/JSON       Бизнес-        Доступ к
  валидация       логика          данным
```

**Правило:** зависимости идут только вниз. Service не знает про HTTP. Repository не знает про бизнес-логику.

```typescript
// Handler — только HTTP: парсинг, валидация, сериализация
app.post('/orders', authMiddleware, zValidator('json', schema), async (c) => {
  const body = c.req.valid('json')
  const userId = c.get('userId')
  const order = await orderService.create(userId, body)  // вся логика здесь
  return c.json(order, 201)
})

// Service — бизнес-логика, оркестрация
class OrderService {
  constructor(
    private orders: OrderRepository,
    private users: UserRepository,
    private events: EventBus,
  ) {}

  async create(userId: string, data: CreateOrderDTO): Promise<Order> {
    const user = await this.users.findById(userId)
    if (!user) throw new NotFoundError('User', userId)
    if (!user.isVerified) throw new ForbiddenError('Email not verified')

    const order = Order.create({ userId, ...data })
    await this.orders.save(order)
    await this.events.emit('order.created', { orderId: order.id, userId })
    return order
  }
}

// Repository — только доступ к данным, без бизнес-логики
class OrderRepository {
  async save(order: Order): Promise<void> {
    await db.insert(orders).values(order.toRow())
      .onConflictDoUpdate({ target: orders.id, set: order.toRow() })
  }
}
```

## Dependency Injection (без фреймворка)

```typescript
// Composition root — index.ts
const db = drizzle(new Database(config.databaseUrl))

const userRepo = new UserRepository(db)
const orderRepo = new OrderRepository(db)
const redis = new Redis(config.redisUrl)
const eventBus = new EventBus()

const userService = new UserService(userRepo, redis)
const orderService = new OrderService(orderRepo, userRepo, eventBus)

const app = new Hono()
app.route('/users', createUsersRouter(userService))
app.route('/orders', createOrdersRouter(orderService))
```

## Event-driven (внутри сервиса)

```typescript
type DomainEvents = {
  'user.registered': { userId: string; email: string }
  'order.created': { orderId: string; userId: string; amount: number }
  'payment.completed': { orderId: string }
  'subscription.expired': { userId: string; subscriptionId: string }
}

class EventBus<T extends Record<string, unknown>> {
  private handlers = new Map<keyof T, Set<(data: any) => Promise<void>>>()

  on<K extends keyof T>(event: K, handler: (data: T[K]) => Promise<void>) {
    if (!this.handlers.has(event)) this.handlers.set(event, new Set())
    this.handlers.get(event)!.add(handler)
    return this  // chaining
  }

  async emit<K extends keyof T>(event: K, data: T[K]) {
    const handlers = this.handlers.get(event) ?? new Set()
    await Promise.allSettled([...handlers].map(h => h(data)))
  }
}

// Регистрация обработчиков в composition root
eventBus
  .on('user.registered', async ({ userId, email }) => {
    await emailQueue.add('welcome', { userId, template: 'welcome' })
  })
  .on('payment.completed', async ({ orderId }) => {
    await orderService.fulfill(orderId)
  })
```

## CQRS lite (когда read и write модели расходятся)

```typescript
// Command — изменяет состояние
interface CreateOrderCommand {
  userId: string
  items: { productId: string; qty: number }[]
}

// Query — только чтение, может быть денормализованным
interface OrderListQuery {
  userId: string
  status?: 'pending' | 'paid'
  page: number
  limit: number
}

class OrderCommandService {
  async create(cmd: CreateOrderCommand): Promise<{ id: string }> { ... }
  async cancel(orderId: string, userId: string): Promise<void> { ... }
}

class OrderQueryService {
  // Может использовать другую модель данных, кэш, даже другую БД
  async listForUser(query: OrderListQuery): Promise<PaginatedResult<OrderSummary>> { ... }
  async getDetails(orderId: string): Promise<OrderDetails | null> { ... }
}
```

## Когда что применять

| Паттерн | Когда | Когда не нужен |
|---------|-------|----------------|
| Layered Architecture | Всегда | — |
| Repository | Есть несколько хранилищ или нужны тесты | Простой CRUD без тестов |
| Result type | Много путей ошибок, нет exceptions | Простые утилиты |
| Event Bus | Decoupling между модулями | 1-2 модуля, прямая связь ок |
| CQRS | Read/write модели сильно расходятся | CRUD с одной моделью |
| BullMQ | Async задачи, retry, scheduling | Синхронные операции |

## Типичные антипаттерны

```typescript
// ❌ Логика в хендлере
app.post('/users', async (c) => {
  const body = await c.req.json()
  const existing = await db.select().from(users).where(eq(users.email, body.email)).get()
  if (existing) return c.json({ error: 'exists' }, 409)
  // ... 50 строк бизнес-логики прямо здесь
})

// ✅ Тонкий хендлер
app.post('/users', zValidator('json', schema), async (c) => {
  const user = await userService.register(c.req.valid('json'))
  return c.json(user, 201)
})

// ❌ Repository с бизнес-логикой
class UserRepository {
  async registerAndSendEmail(data) {
    // Репозиторий не должен знать про email!
  }
}

// ❌ Circular dependencies
// UserService → OrderService → UserService
// Решение: EventBus или вынести общую логику в третий сервис
```