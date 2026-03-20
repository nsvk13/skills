---
name: frontend
description: >
  Expert frontend development skill. Use for ANY frontend question: React (hooks,
  context, suspense, server components), Svelte 5 (runes, snippets, stores),
  TypeScript на фронте, Vite, TailwindCSS, CSS (animations, grid, flexbox),
  state management (Zustand, Jotai, Svelte stores), routing (TanStack Router,
  React Router, SvelteKit), data fetching (TanStack Query, SWR), формы (React Hook Form,
  Svelte), Telegram Mini Apps (WebApp API), SSR/SSG, производительность фронта,
  бандл-оптимизация, accessibility. Trigger на: написание компонентов, вопросы про
  state, "как сделать анимацию", paste JSX/Svelte кода с ошибками, code review
  фронтенда, архитектура фронт-приложения, Telegram WebApp интеграция.
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

## Reference Files

- `references/performance.md` — bundle optimization, lazy loading, Core Web Vitals
- `references/forms.md` — React Hook Form, Zod валидация, controlled/uncontrolled