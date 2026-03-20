---
name: frontend
description: >
  Expert frontend development skill. Use for ANY frontend question: React (hooks,
  context, suspense, server components), Next.js (App Router, RSC, Server Actions,
  ISR, middleware), Astro (islands, content collections, SSG/SSR), Svelte 5 (runes,
  snippets, SvelteKit), Bun as frontend runtime/bundler, TypeScript на фронте,
  Vite, TailwindCSS, CSS (animations, grid, flexbox), state management (Zustand,
  Jotai, Svelte stores), routing (TanStack Router, React Router), data fetching
  (TanStack Query, SWR), формы (React Hook Form), Telegram Mini Apps (WebApp API),
  производительность (Core Web Vitals, bundle optimization, virtualization),
  accessibility. Trigger на: написание компонентов, "как сделать на Next.js",
  вопросы про App Router vs Pages Router, Astro islands, SSR/SSG/ISR выбор,
  paste JSX/Svelte/Astro кода с ошибками, code review, Telegram WebApp интеграция.
---

# Frontend Skill

## Identity & Style

Senior frontend разработчик. Основной стек: React + TypeScript + Vite, Svelte 5 для
новых проектов. Пишу компоненты которые легко читать, тестировать и переиспользовать.

**Правила:**
- Русский, кратко
- Указывай когда что-то плохо для производительности (лишние ре-рендеры, тяжёлые импорты)
- Accessibility не игнорируем — семантика, aria, keyboard nav
- Адаптивный стиль: "дай компонент" → код; "как организовать" → архитектура

---

## React — паттерны

### Компонент с правильной типизацией

```typescript
import { type FC, type ReactNode, useState, useCallback } from 'react'

interface ButtonProps {
  children: ReactNode
  onClick?: () => void | Promise<void>
  variant?: 'primary' | 'secondary' | 'danger'
  disabled?: boolean
  loading?: boolean
  className?: string
}

export const Button: FC<ButtonProps> = ({
  children,
  onClick,
  variant = 'primary',
  disabled = false,
  loading = false,
  className,
}) => {
  const handleClick = useCallback(async () => {
    if (disabled || loading || !onClick) return
    await onClick()
  }, [onClick, disabled, loading])

  return (
    <button
      onClick={handleClick}
      disabled={disabled || loading}
      aria-busy={loading}
      className={cn(buttonVariants({ variant }), className)}
    >
      {loading ? <Spinner size="sm" /> : children}
    </button>
  )
}
```

### Custom hooks

```typescript
// Data fetching с loading/error state
function useUser(userId: string) {
  const [state, setState] = useState<{
    data: User | null
    loading: boolean
    error: Error | null
  }>({ data: null, loading: true, error: null })

  useEffect(() => {
    let cancelled = false

    async function fetch() {
      try {
        setState(s => ({ ...s, loading: true, error: null }))
        const user = await userService.getById(userId)
        if (!cancelled) setState({ data: user, loading: false, error: null })
      } catch (err) {
        if (!cancelled) setState({ data: null, loading: false, error: err as Error })
      }
    }

    fetch()
    return () => { cancelled = true }
  }, [userId])

  return state
}

// Или через TanStack Query (предпочтительно)
function useUser(userId: string) {
  return useQuery({
    queryKey: ['users', userId],
    queryFn: () => userService.getById(userId),
    staleTime: 5 * 60 * 1000,
  })
}
```

### Избегаем лишних ре-рендеров

```typescript
// Мемоизация дорогих вычислений
const sortedItems = useMemo(
  () => items.sort((a, b) => a.price - b.price),
  [items],
)

// Стабильные callback
const handleDelete = useCallback(
  (id: string) => deleteItem(id),
  [deleteItem],  // только если deleteItem из пропсов
)

// memo для чистых компонентов
const ItemCard = memo(({ item, onDelete }: ItemCardProps) => {
  return <div onClick={() => onDelete(item.id)}>{item.name}</div>
})

// Когда НЕ нужна мемоизация:
// - Простые компоненты с примитивными пропсами
// - Компоненты которые всегда ре-рендерятся (меняется контекст)
// - Везде подряд — это антипаттерн
```

### Context — правильный способ

```typescript
// Разделяй state и dispatch — разные контексты
const UserStateContext = createContext<User | null>(null)
const UserDispatchContext = createContext<Dispatch<UserAction> | null>(null)

export function UserProvider({ children }: { children: ReactNode }) {
  const [user, dispatch] = useReducer(userReducer, null)

  return (
    <UserStateContext value={user}>
      <UserDispatchContext value={dispatch}>
        {children}
      </UserDispatchContext>
    </UserStateContext>
  )
}

// Типизированные хуки
export function useUser() {
  const user = useContext(UserStateContext)
  if (user === undefined) throw new Error('useUser must be inside UserProvider')
  return user
}
```

---

## Svelte 5 — runes

```svelte
<script lang="ts">
  // $state — реактивное состояние
  let count = $state(0)
  let user = $state<User | null>(null)

  // $derived — вычисляемые значения (замена $: и stores)
  let doubled = $derived(count * 2)
  let isAdmin = $derived(user?.role === 'admin')

  // $effect — side effects (замена reactive statements)
  $effect(() => {
    console.log('count changed:', count)
    // cleanup (опционально)
    return () => console.log('cleanup')
  })

  // Props через $props
  let { title, onClose }: { title: string; onClose: () => void } = $props()

  // Async
  async function loadUser(id: string) {
    user = await userService.getById(id)
  }
</script>

<button onclick={() => count++}>
  Count: {count} (doubled: {doubled})
</button>
```

### Svelte 5 — компоненты и snippets

```svelte
<!-- Snippet — переиспользуемый шаблон внутри компонента -->
{#snippet userCard(user: User)}
  <div class="card">
    <h3>{user.name}</h3>
    <p>{user.email}</p>
  </div>
{/snippet}

{#each users as user}
  {@render userCard(user)}
{/each}

<!-- Передача snippet в дочерний компонент -->
<Modal>
  {#snippet header()}
    <h2>Заголовок</h2>
  {/snippet}
  Содержимое
</Modal>
```

---

## TanStack Query

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'

// Query
function UsersList() {
  const { data: users, isLoading, error } = useQuery({
    queryKey: ['users'],
    queryFn: () => api.getUsers(),
    staleTime: 5 * 60 * 1000,   // данные свежие 5 минут
    gcTime: 10 * 60 * 1000,      // в кэше 10 минут
  })

  if (isLoading) return <Skeleton />
  if (error) return <ErrorMessage error={error} />
  return <>{users?.map(u => <UserCard key={u.id} user={u} />)}</>
}

// Mutation с оптимистичным обновлением
function useDeleteUser() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: (userId: string) => api.deleteUser(userId),

    onMutate: async (userId) => {
      await queryClient.cancelQueries({ queryKey: ['users'] })
      const prev = queryClient.getQueryData<User[]>(['users'])
      queryClient.setQueryData<User[]>(['users'], old => old?.filter(u => u.id !== userId))
      return { prev }  // rollback data
    },

    onError: (err, userId, ctx) => {
      queryClient.setQueryData(['users'], ctx?.prev)  // rollback
    },

    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['users'] })
    },
  })
}
```

---

## Telegram Mini App

```typescript
import WebApp from '@twa-dev/sdk'

// Инициализация
WebApp.ready()
WebApp.expand()

// Получить данные пользователя
const user = WebApp.initDataUnsafe.user
// user.id, user.first_name, user.username

// Верификация на бэке (обязательно!)
// initData отправляется на сервер и проверяется через HMAC
async function verifyUser(initData: string): Promise<boolean> {
  const response = await fetch('/api/auth/telegram', {
    method: 'POST',
    body: JSON.stringify({ initData }),
  })
  return response.ok
}

// Тема Telegram
const colors = WebApp.themeParams
document.documentElement.style.setProperty('--bg', colors.bg_color)

// Main Button
WebApp.MainButton.setText('Оплатить 199₽')
WebApp.MainButton.show()
WebApp.MainButton.onClick(() => {
  handlePayment()
})

// Закрыть приложение
WebApp.close()

// Haptic
WebApp.HapticFeedback.impactOccurred('medium')
WebApp.HapticFeedback.notificationOccurred('success')
```

```typescript
// Верификация initData на бэке (Hono)
app.post('/api/auth/telegram', async (c) => {
  const { initData } = await c.req.json()

  const urlParams = new URLSearchParams(initData)
  const hash = urlParams.get('hash')
  urlParams.delete('hash')

  const dataCheckString = [...urlParams.entries()]
    .sort(([a], [b]) => a.localeCompare(b))
    .map(([k, v]) => `${k}=${v}`)
    .join('\n')

  const secretKey = await crypto.subtle.importKey(
    'raw',
    new TextEncoder().encode('WebAppData'),
    { name: 'HMAC', hash: 'SHA-256' },
    false, ['sign'],
  )

  const hmacKey = await crypto.subtle.sign(
    'HMAC',
    secretKey,
    new TextEncoder().encode(config.botToken),
  )

  const key = await crypto.subtle.importKey('raw', hmacKey, { name: 'HMAC', hash: 'SHA-256' }, false, ['sign'])
  const signature = await crypto.subtle.sign('HMAC', key, new TextEncoder().encode(dataCheckString))
  const expected = Buffer.from(signature).toString('hex')

  if (expected !== hash) throw new UnauthorizedError()

  const user = JSON.parse(urlParams.get('user') ?? '{}')
  return c.json({ token: await signToken({ sub: String(user.id) }) })
})
```

---

## TailwindCSS — паттерны

```typescript
// cn() — условные классы
import { clsx, type ClassValue } from 'clsx'
import { twMerge } from 'tailwind-merge'

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}

// Использование
<div className={cn(
  'base-class p-4 rounded',
  isActive && 'bg-blue-500',
  isDisabled && 'opacity-50 cursor-not-allowed',
  className,
)} />

// cva() — варианты компонента
import { cva } from 'class-variance-authority'

const buttonVariants = cva(
  'inline-flex items-center justify-center rounded font-medium transition-colors',
  {
    variants: {
      variant: {
        primary: 'bg-blue-600 text-white hover:bg-blue-700',
        secondary: 'bg-gray-100 text-gray-900 hover:bg-gray-200',
        danger: 'bg-red-600 text-white hover:bg-red-700',
      },
      size: {
        sm: 'h-8 px-3 text-sm',
        md: 'h-10 px-4',
        lg: 'h-12 px-6 text-lg',
      },
    },
    defaultVariants: { variant: 'primary', size: 'md' },
  },
)
```

---

## Next.js — App Router

```typescript
// app/users/[id]/page.tsx — Server Component по умолчанию
interface Props { params: Promise<{ id: string }> }

export default async function UserPage({ params }: Props) {
  const { id } = await params
  const user = await db.getUser(id)  // прямой вызов БД, не нужен API
  if (!user) notFound()
  return <UserProfile user={user} />
}

export async function generateMetadata({ params }: Props) {
  const { id } = await params
  const user = await db.getUser(id)
  return { title: user?.name ?? 'User' }
}

// Статическая генерация
export async function generateStaticParams() {
  const users = await db.getAllUsers()
  return users.map(u => ({ id: u.id }))
}
```

### Server Actions

```typescript
'use server'
import { revalidatePath } from 'next/cache'
import { redirect } from 'next/navigation'

export async function createUser(formData: FormData) {
  const parsed = schema.safeParse(Object.fromEntries(formData))
  if (!parsed.success) return { error: parsed.error.flatten().fieldErrors }

  await db.users.create(parsed.data)
  revalidatePath('/users')
  redirect('/users')
}

// Client Component
'use client'
import { useActionState } from 'react'

export function CreateUserForm() {
  const [state, action, isPending] = useActionState(createUser, null)
  return (
    <form action={action}>
      <input name="name" />
      <button disabled={isPending}>{isPending ? 'Создаём...' : 'Создать'}</button>
      {state?.error && <p>{JSON.stringify(state.error)}</p>}
    </form>
  )
}
```

### Кэширование и ISR

```typescript
// fetch с ISR
const data = await fetch('https://api.example.com/data', {
  next: { revalidate: 3600 },  // обновлять раз в час
})

// no-store — всегда свежие данные (SSR)
const live = await fetch('https://api.example.com/live', { cache: 'no-store' })

// unstable_cache — для не-fetch источников (pgx, drizzle)
import { unstable_cache } from 'next/cache'

const getCachedUser = unstable_cache(
  async (id: string) => db.getUser(id),
  ['user'],
  { revalidate: 300, tags: ['users'] },
)

// Инвалидация в Server Action
import { revalidateTag } from 'next/cache'
revalidateTag('users')
```

### RSC vs Client Component

```
Server Component (default)       Client Component ('use client')
─────────────────────────        ──────────────────────────────
✅ Прямой доступ к БД            ✅ useState, useEffect, hooks
✅ Секреты не утекают            ✅ onClick, event handlers
✅ Меньше JS в бандле            ✅ Browser APIs (localStorage)
❌ Нет hooks, нет событий        ❌ Нет доступа к БД напрямую
```

### Middleware

```typescript
// middleware.ts
import { NextRequest, NextResponse } from 'next/server'

export function middleware(req: NextRequest) {
  const token = req.cookies.get('token')?.value
  if (!token && req.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', req.url))
  }
  return NextResponse.next()
}

export const config = { matcher: ['/dashboard/:path*'] }
```

---

## Astro

```astro
---
// src/pages/blog/[slug].astro
import { getCollection } from 'astro:content'
import Layout from '@/layouts/Layout.astro'

export async function getStaticPaths() {
  const posts = await getCollection('blog')
  return posts.map(post => ({ params: { slug: post.slug }, props: { post } }))
}

const { post } = Astro.props
const { Content } = await post.render()
---

<Layout title={post.data.title}>
  <article>
    <h1>{post.data.title}</h1>
    <Content />
  </article>
</Layout>
```

### Content Collections

```typescript
// src/content/config.ts
import { defineCollection, z } from 'astro:content'

const blog = defineCollection({
  type: 'content',
  schema: z.object({
    title: z.string(),
    date: z.date(),
    tags: z.array(z.string()).default([]),
    draft: z.boolean().default(false),
  }),
})

export const collections = { blog }
```

### Islands Architecture

```astro
---
import HeavyChart from '@/components/Chart.tsx'
import Counter from '@/components/Counter.svelte'
---

<!-- Статичный HTML — 0 JS -->
<h1>Заголовок</h1>

<!-- client:load — сразу -->
<HeavyChart client:load />

<!-- client:visible — когда попадает в viewport -->
<HeavyChart client:visible />

<!-- client:idle — когда браузер не занят -->
<Counter client:idle />

<!-- client:only — только клиент, без SSR -->
<MapComponent client:only="react" />
```

### Когда Astro, когда Next.js

| Сценарий | Astro | Next.js |
|----------|-------|---------|
| Блог, документация, лендинг | ✅ Идеально | Избыточно |
| Максимальный Lighthouse | ✅ | Сложнее |
| Разные фреймворки (React + Svelte) | ✅ Уникально | ❌ |
| Fullstack: Auth, БД, API | Можно | ✅ Лучше |
| Сложный интерактивный UI | Можно (islands) | ✅ Лучше |

---

## Bun — frontend tooling

```bash
# Bun вместо Node в любом frontend проекте
bun create vite my-app --template react-ts
bun install     # в разы быстрее npm
bun run dev
bun run build

# Bun bundler (для простых проектов без Vite)
bun build ./src/index.tsx --outdir ./dist --minify --sourcemap
```

```toml
# bunfig.toml
[test]
environment = "happy-dom"  # DOM API в тестах без jsdom
preload = ["./test/setup.ts"]
```

```typescript
// test/setup.ts — DOM моки для Bun test
import { mock } from 'bun:test'

global.localStorage = {
  getItem: mock(() => null),
  setItem: mock(() => {}),
  removeItem: mock(() => {}),
  clear: mock(() => {}),
  length: 0,
  key: mock(() => null),
}

Object.defineProperty(window, 'matchMedia', {
  value: mock((query: string) => ({
    matches: false,
    media: query,
    addEventListener: mock(() => {}),
    removeEventListener: mock(() => {}),
  })),
})
```

---

## Reference Files

- `references/performance.md` — bundle optimization, code splitting, виртуализация, Core Web Vitals, изображения, шрифты
- `references/forms.md` — React Hook Form, Zod валидация, controlled/uncontrolled