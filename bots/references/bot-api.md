# Telegram Bot API — справочник

## Лимиты и ограничения

| Параметр | Лимит |
|----------|-------|
| Длина сообщения | 4096 символов |
| Caption (фото/видео) | 1024 символа |
| Сообщений в секунду (общий) | 30 |
| Сообщений в секунду (один чат) | 1 |
| Размер файла (sendDocument) | 50 MB |
| Размер файла (download) | 20 MB |
| Кнопок в InlineKeyboard | 100 (8 в ряду) |
| Время ответа на pre_checkout_query | 10 секунд |
| Время ответа на callback_query | нет лимита, но answerCallbackQuery — до 30 сек |
| Webhook timeout | 60 секунд |

## Типы чатов

```typescript
ctx.chat.type // 'private' | 'group' | 'supergroup' | 'channel'
ctx.from      // undefined в channel posts
```

## Методы — частые

```typescript
// Отправка
bot.api.sendMessage(chatId, text, { parse_mode: 'HTML' })
bot.api.sendPhoto(chatId, fileIdOrUrl, { caption: '...' })
bot.api.sendDocument(chatId, new InputFile(buffer, 'file.pdf'))
bot.api.sendMediaGroup(chatId, [InputMediaPhoto, ...])

// Редактирование
bot.api.editMessageText(chatId, messageId, newText)
ctx.editMessageText(newText)  // если из callback_query

// Удаление
bot.api.deleteMessage(chatId, messageId)

// Реакции
bot.api.setMessageReaction(chatId, messageId, [{ type: 'emoji', emoji: '👍' }])

// Chat actions
bot.api.sendChatAction(chatId, 'typing' | 'upload_photo' | 'upload_document')

// Pin
bot.api.pinChatMessage(chatId, messageId, { disable_notification: true })

// Получить информацию
bot.api.getChat(chatId)
bot.api.getChatMember(chatId, userId)
bot.api.getChatMembersCount(chatId)
```

## Parse modes

```typescript
// HTML — предпочтительный
const text = `<b>Жирный</b> <i>Курсив</i> <code>код</code>
<a href="https://example.com">Ссылка</a>
<pre>блок кода</pre>
<blockquote>цитата</blockquote>`

// Экранирование HTML
const safe = text
  .replace(/&/g, '&amp;')
  .replace(/</g, '&lt;')
  .replace(/>/g, '&gt;')

// MarkdownV2 — неудобный, много символов нужно экранировать
// Использовать только если действительно нужен
```

## Inline режим (в строке поиска)

```typescript
bot.on('inline_query', async (ctx) => {
  const query = ctx.inlineQuery.query

  await ctx.answerInlineQuery([
    {
      type: 'article',
      id: '1',
      title: 'Результат',
      input_message_content: {
        message_text: `Вы искали: ${query}`,
      },
    },
  ], { cache_time: 0 })
})
```

## My chat member — отслеживание блокировок

```typescript
bot.on('my_chat_member', async (ctx) => {
  const newStatus = ctx.myChatMember.new_chat_member.status

  if (newStatus === 'kicked' || newStatus === 'left') {
    // Пользователь заблокировал бота
    await userService.deactivate(String(ctx.from.id))
  }

  if (newStatus === 'member') {
    // Пользователь разблокировал (или новый старт после блока)
    await userService.reactivate(String(ctx.from.id))
  }
})
```

## Deep links

```typescript
// Ссылка: https://t.me/botname?start=ref_userId123
bot.command('start', async (ctx) => {
  const payload = ctx.match  // 'ref_userId123'
  if (payload?.startsWith('ref_')) {
    const referrerId = payload.slice(4)
    await referralService.register(String(ctx.from!.id), referrerId)
  }
  await ctx.reply('Добро пожаловать!', { reply_markup: mainMenu })
})

// Webapp (Mini App)
const keyboard = new InlineKeyboard()
  .webApp('Открыть приложение', 'https://your-webapp.com')
```

## Группы и каналы

```typescript
// Проверка подписки на канал
async function isSubscribed(userId: number, channelId: string): Promise<boolean> {
  try {
    const member = await bot.api.getChatMember(channelId, userId)
    return ['member', 'administrator', 'creator'].includes(member.status)
  } catch {
    return false  // бот не в канале или канал не существует
  }
}

// Требовать подписку
bot.command('start', async (ctx) => {
  const subscribed = await isSubscribed(ctx.from!.id, '@required_channel')
  if (!subscribed) {
    const keyboard = new InlineKeyboard()
      .url('Подписаться', 'https://t.me/required_channel')
      .row()
      .text('Проверить подписку', 'check_subscription')
    return ctx.reply('Подпишитесь на канал для доступа:', { reply_markup: keyboard })
  }
  // ...
})
```

## Errors — типичные

| error_code | Описание | Решение |
|------------|----------|---------|
| 400 Bad Request | Неверный запрос | Проверить параметры |
| 400 message is not modified | Текст не изменился при edit | Сравнивать перед edit |
| 403 Forbidden | Бот заблокирован | Деактивировать пользователя |
| 429 Too Many Requests | Rate limit | Respect retry_after |
| 500 | Ошибка Telegram | Retry с экспоненциальным backoff |

```typescript
import { GrammyError } from 'grammy'

try {
  await bot.api.sendMessage(userId, text)
} catch (err) {
  if (err instanceof GrammyError) {
    if (err.error_code === 429) {
      await Bun.sleep(err.parameters?.retry_after ?? 1000)
      // retry
    }
    if (err.error_code === 403) {
      await userService.deactivate(userId)
    }
  }
}
```