---
name: bots
description: >
  Expert Telegram bot development skill. Use for ANY вопрос про Telegram ботов:
  GrammY (основной фреймворк), GramIO, Telegram Bot API (все методы, типы, ограничения),
  FSM и conversation flows, inline keyboards и меню, webhook vs polling, payments
  (Telegram Stars, ЮKassa, платёжные провайдеры), подписки и доступ, middleware,
  session storage (Redis, PostgreSQL), мультиязычность (i18n), rate limiting,
  деплой бота (Docker, webhook через Nginx), Remnawave/3x-ui интеграция для VPN-ботов,
  обработка ошибок, логирование. Trigger на: написание бота, вопросы про GrammY API,
  "как сделать меню", "как принимать платежи", paste кода бота с ошибками,
  code review бота, вопросы про архитектуру multi-feature бота.
---

# Bots Skill

## Identity & Style

Senior Telegram bot разработчик. GrammY как основной фреймворк, TypeScript, Bun.
Знаю Bot API наизусть — что возможно, что нет, какие лимиты.

**Правила:**
- Русский, кратко
- Указывай ограничения Bot API когда они важны (размер сообщения 4096 символов, etc.)
- Production-ready: error handling, graceful stop, не теряем updates

---

## GrammY — базовая структура

```typescript
import { Bot, Context, session, SessionFlavor, GrammyError, HttpError } from 'grammy'
import { Menu } from '@grammyjs/menu'
import { Router } from '@grammyjs/router'

// Типизированный контекст
interface SessionData {
  step: string
  data: Record<string, unknown>
}

type MyContext = Context & SessionFlavor<SessionData>

const bot = new Bot<MyContext>(config.botToken)

// Session
bot.use(session({
  initial: (): SessionData => ({ step: 'idle', data: {} }),
  // storage: new RedisAdapter({ instance: redis }) — для продакшна
}))

// Глобальный error handler
bot.catch((err) => {
  const ctx = err.ctx
  console.error(`Error for update ${ctx.update.update_id}:`, err.error)

  if (err.error instanceof GrammyError) {
    console.error('Telegram API error:', err.error.description)
  } else if (err.error instanceof HttpError) {
    console.error('HTTP error:', err.error)
  }
})

// Graceful stop
process.once('SIGINT', () => bot.stop())
process.once('SIGTERM', () => bot.stop())
```

---

## Меню и inline keyboards

```typescript
import { Menu } from '@grammyjs/menu'
import { InlineKeyboard } from 'grammy'

// Динамическое меню (Menu plugin)
const mainMenu = new Menu<MyContext>('main')
  .text('💳 Подписка', ctx => ctx.reply('Выберите план:', { reply_markup: plansMenu }))
  .row()
  .text('👤 Профиль', handleProfile)
  .text('❓ Помощь', ctx => ctx.reply(helpText))

const plansMenu = new Menu<MyContext>('plans')
  .text('Basic — 199₽/мес', ctx => ctx.session.data.plan = 'basic' && handlePayment(ctx))
  .row()
  .text('Pro — 499₽/мес', ctx => ctx.session.data.plan = 'pro' && handlePayment(ctx))
  .row()
  .back('← Назад')

mainMenu.register(plansMenu)
bot.use(mainMenu)

// Одноразовый InlineKeyboard (без состояния)
const keyboard = new InlineKeyboard()
  .text('✅ Подтвердить', 'confirm_order')
  .text('❌ Отмена', 'cancel_order')

await ctx.reply('Подтвердите заказ:', { reply_markup: keyboard })

// Обработчик callback
bot.callbackQuery('confirm_order', async (ctx) => {
  await ctx.answerCallbackQuery()  // убрать загрузку
  await ctx.editMessageText('✅ Заказ подтверждён')
})
```

---

## FSM через session

```typescript
// Простой FSM без библиотек
const STEPS = {
  IDLE: 'idle',
  AWAITING_EMAIL: 'awaiting_email',
  AWAITING_PHONE: 'awaiting_phone',
  CONFIRMING: 'confirming',
} as const

bot.command('register', async (ctx) => {
  ctx.session.step = STEPS.AWAITING_EMAIL
  ctx.session.data = {}
  await ctx.reply('Введите ваш email:')
})

bot.on('message:text', async (ctx) => {
  switch (ctx.session.step) {
    case STEPS.AWAITING_EMAIL: {
      const email = ctx.message.text
      if (!email.includes('@')) {
        return ctx.reply('Некорректный email. Попробуйте ещё раз:')
      }
      ctx.session.data.email = email
      ctx.session.step = STEPS.AWAITING_PHONE
      return ctx.reply('Введите номер телефона:')
    }

    case STEPS.AWAITING_PHONE: {
      ctx.session.data.phone = ctx.message.text
      ctx.session.step = STEPS.CONFIRMING

      const keyboard = new InlineKeyboard()
        .text('✅ Подтвердить', 'confirm_registration')
        .text('❌ Отмена', 'cancel_registration')

      return ctx.reply(
        `Проверьте данные:\nEmail: ${ctx.session.data.email}\nТелефон: ${ctx.session.data.phone}`,
        { reply_markup: keyboard },
      )
    }
  }
})

bot.callbackQuery('confirm_registration', async (ctx) => {
  await ctx.answerCallbackQuery()
  const { email, phone } = ctx.session.data
  await userService.register({ userId: String(ctx.from!.id), email, phone })
  ctx.session.step = STEPS.IDLE
  ctx.session.data = {}
  await ctx.editMessageText('✅ Регистрация завершена!')
})

bot.callbackQuery('cancel_registration', async (ctx) => {
  await ctx.answerCallbackQuery()
  ctx.session.step = STEPS.IDLE
  ctx.session.data = {}
  await ctx.editMessageText('Регистрация отменена.')
})
```

---

## Payments — Telegram Stars

```typescript
// Отправить инвойс
await ctx.replyWithInvoice(
  'Подписка Pro',
  'Доступ ко всем функциям на 30 дней',
  JSON.stringify({ plan: 'pro', userId: ctx.from!.id }),  // payload
  'XTR',  // валюта Telegram Stars
  [{ label: 'Подписка Pro — 30 дней', amount: 100 }],  // 100 Stars
)

// Pre-checkout — обязательно подтвердить в течение 10 секунд
bot.on('pre_checkout_query', async (ctx) => {
  // Проверить наличие товара, валидность и т.д.
  await ctx.answerPreCheckoutQuery(true)
  // или await ctx.answerPreCheckoutQuery(false, 'Товар недоступен')
})

// Успешная оплата
bot.on('message:successful_payment', async (ctx) => {
  const payment = ctx.message.successful_payment
  const payload = JSON.parse(payment.invoice_payload)

  await subscriptionService.activate(payload.userId, payload.plan, 30)
  await ctx.reply('✅ Оплата прошла! Подписка активирована.')
})
```

---

## Webhook vs Polling

```typescript
// Polling — для разработки
bot.start({ onStart: () => console.log('Bot started (polling)') })

// Webhook — для продакшна
import { webhookCallback } from 'grammy'

const handleUpdate = webhookCallback(bot, 'std/http')

Bun.serve({
  port: config.port,
  async fetch(req) {
    const url = new URL(req.url)
    if (url.pathname === `/webhook/${config.botToken}`) {
      return handleUpdate(req)
    }
    return new Response('Not found', { status: 404 })
  },
})

// Установить webhook
await bot.api.setWebhook(`https://your-domain.com/webhook/${config.botToken}`, {
  secret_token: config.webhookSecret,  // дополнительная защита
  max_connections: 100,
  allowed_updates: ['message', 'callback_query', 'pre_checkout_query', 'my_chat_member'],
})

// Удалить webhook (при переключении на polling)
await bot.api.deleteWebhook()
```

---

## Middleware и разграничение доступа

```typescript
// Только приватные чаты
bot.use(async (ctx, next) => {
  if (ctx.chat?.type !== 'private') return
  await next()
})

// Проверка подписки
const requireSubscription = async (ctx: MyContext, next: NextFunction) => {
  const sub = await subscriptionService.getActive(String(ctx.from!.id))
  if (!sub) {
    const keyboard = new InlineKeyboard().text('💳 Купить подписку', 'buy_subscription')
    await ctx.reply('Для этого нужна активная подписка.', { reply_markup: keyboard })
    return
  }
  await next()
}

// Только для adminов
const adminIds = new Set(config.adminIds)
const adminOnly = async (ctx: MyContext, next: NextFunction) => {
  if (!adminIds.has(ctx.from?.id ?? 0)) return
  await next()
}

bot.command('stats', adminOnly, handleStats)
bot.command('premium_feature', requireSubscription, handlePremiumFeature)
```

---

## Отправка сообщений — лимиты и паттерны

```typescript
// Лимиты Bot API:
// - 30 сообщений/сек общий лимит
// - 1 сообщение/сек в один чат
// - Длина сообщения: 4096 символов
// - Caption: 1024 символа

// Разбивка длинного текста
function splitMessage(text: string, maxLength = 4096): string[] {
  if (text.length <= maxLength) return [text]
  const parts: string[] = []
  let remaining = text
  while (remaining.length > 0) {
    let chunk = remaining.slice(0, maxLength)
    const lastNewline = chunk.lastIndexOf('\n')
    if (lastNewline > 0 && remaining.length > maxLength) {
      chunk = chunk.slice(0, lastNewline)
    }
    parts.push(chunk)
    remaining = remaining.slice(chunk.length).trimStart()
  }
  return parts
}

// Broadcast с rate limiting
async function broadcast(userIds: string[], message: string) {
  for (const userId of userIds) {
    try {
      await bot.api.sendMessage(userId, message)
      await Bun.sleep(50)  // ~20 сообщений/сек
    } catch (err) {
      if (err instanceof GrammyError && err.error_code === 403) {
        // Пользователь заблокировал бота
        await userService.deactivate(userId)
      }
    }
  }
}
```

---

## Session storage — Redis

```typescript
import { RedisAdapter } from '@grammyjs/storage-redis'
import { Redis } from 'ioredis'

const redis = new Redis(config.redisUrl)

bot.use(session({
  initial: (): SessionData => ({ step: 'idle', data: {} }),
  storage: new RedisAdapter({
    instance: redis,
    ttl: 24 * 60 * 60,  // 24 часа
  }),
}))
```

---

## Reference Files

- `references/bot-api.md` — полный справочник методов Bot API, типы, ограничения
- `references/grammy-advanced.md` — conversations plugin, i18n, throttler, hydration