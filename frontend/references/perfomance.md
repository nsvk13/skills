# Frontend Performance

## Bundle optimization

### Vite — анализ и оптимизация
```bash
# Анализ бандла
npm install --save-dev rollup-plugin-visualizer

# vite.config.ts
import { visualizer } from 'rollup-plugin-visualizer'

export default defineConfig({
  plugins: [
    visualizer({ open: true, gzip: true, filename: 'dist/stats.html' }),
  ],
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
          query: ['@tanstack/react-query'],
          router: ['@tanstack/react-router'],
        },
      },
    },
    chunkSizeWarningLimit: 500,
  },
})
```

### Code splitting — lazy loading
```typescript
import { lazy, Suspense } from 'react'

// Роут-based splitting (основное)
const Dashboard = lazy(() => import('./pages/Dashboard'))
const Settings = lazy(() => import('./pages/Settings'))

// С fallback
<Suspense fallback={<PageSkeleton />}>
  <Dashboard />
</Suspense>

// Условный импорт тяжёлой библиотеки
async function renderChart(data: ChartData) {
  const { Chart } = await import('chart.js')  // грузится только когда нужен
  // ...
}

// Preload на hover
function NavLink({ to, children }: { to: string; children: ReactNode }) {
  const preload = () => import(`./pages/${to}`)  // начать загрузку заранее
  return <a href={to} onMouseEnter={preload}>{children}</a>
}
```

### Tree shaking — что ломает его
```typescript
// ❌ Импорт всей библиотеки
import _ from 'lodash'
const result = _.groupBy(items, 'type')

// ✅ Именованный импорт
import { groupBy } from 'lodash-es'  // или отдельный пакет lodash.groupby

// ❌ Side-effect импорты без /*#__PURE__*/
import 'some-polyfill'  // может не убраться при tree shaking

// ✅ Явные named exports из своих модулей
export { UserCard } from './UserCard'
export { OrderList } from './OrderList'
// не export * from '...' — ломает tree shaking
```

---

## React производительность

### Профилирование
```typescript
// React DevTools Profiler — в браузере
// Или программно:
import { Profiler } from 'react'

<Profiler id="UserList" onRender={(id, phase, duration) => {
  if (duration > 16) console.warn(`${id} took ${duration}ms (${phase})`)
}}>
  <UserList />
</Profiler>
```

### Когда мемоизировать
```typescript
// ✅ memo — если компонент часто рендерится с теми же пропсами
const HeavyChart = memo(({ data }: { data: ChartData[] }) => {
  // дорогой рендер
})

// ✅ useMemo — дорогие вычисления (сортировка, фильтрация больших списков)
const filtered = useMemo(
  () => items.filter(i => i.category === selectedCategory).sort(byDate),
  [items, selectedCategory],
)

// ✅ useCallback — если передаётся в memo-компонент
const handleDelete = useCallback((id: string) => {
  setItems(prev => prev.filter(i => i.id !== id))
}, [])  // нет зависимостей — стабильная функция

// ❌ Не нужно:
const doubled = useMemo(() => count * 2, [count])     // тривиально
const onClick = useCallback(() => setOpen(true), [])  // компонент не memo
```

### Виртуализация длинных списков
```typescript
import { useVirtualizer } from '@tanstack/react-virtual'

function VirtualList({ items }: { items: Item[] }) {
  const parentRef = useRef<HTMLDivElement>(null)

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 64,  // высота элемента
    overscan: 5,
  })

  return (
    <div ref={parentRef} style={{ height: '600px', overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize() }}>
        {virtualizer.getVirtualItems().map(virtualItem => (
          <div
            key={virtualItem.key}
            style={{
              position: 'absolute',
              top: virtualItem.start,
              height: virtualItem.size,
            }}
          >
            <ItemRow item={items[virtualItem.index]!} />
          </div>
        ))}
      </div>
    </div>
  )
}
```

### Избегаем лишних ре-рендеров — паттерны
```typescript
// ❌ Новый объект на каждый рендер
<Child style={{ padding: 16 }} />  // новая ссылка каждый раз

// ✅ Вынести константу
const childStyle = { padding: 16 }
<Child style={childStyle} />

// ❌ Инлайн функция в пропсах memo-компонента
<MemoChild onClick={() => handleClick(item.id)} />

// ✅ Стабильная функция
const onClick = useCallback(() => handleClick(item.id), [item.id])
<MemoChild onClick={onClick} />

// Паттерн "children как функция" — изолирует изменения
function StateWrapper({ children }: { children: (count: number) => ReactNode }) {
  const [count, setCount] = useState(0)
  return <>{children(count)}</>
}
// Родитель не ре-рендерится при изменении count
```

---

## Core Web Vitals

| Метрика | Хорошо | Плохо | Что влияет |
|---------|--------|-------|------------|
| LCP (Largest Contentful Paint) | < 2.5s | > 4s | Размер изображений, TTFB, render-blocking ресурсы |
| INP (Interaction to Next Paint) | < 200ms | > 500ms | Тяжёлые обработчики, долгие задачи в main thread |
| CLS (Cumulative Layout Shift) | < 0.1 | > 0.25 | Изображения без размеров, динамический контент |

### LCP — быстрый первый экран
```html
<!-- Preload главного изображения -->
<link rel="preload" as="image" href="/hero.webp" fetchpriority="high">

<!-- Preconnect к важным доменам -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="dns-prefetch" href="https://api.example.com">
```

```typescript
// Next.js — priority для hero изображения
<Image src="/hero.webp" priority alt="Hero" width={1200} height={600} />

// Не lazy load above-the-fold контент
<img src="/logo.png" loading="eager" />  // не lazy!
```

### CLS — избегаем прыжков
```css
/* Всегда указывай размеры изображений */
img { aspect-ratio: 16/9; width: 100%; }

/* Skeleton с точными размерами */
.skeleton { width: 100%; height: 48px; }

/* Резервируй место для динамического контента */
.ad-slot { min-height: 250px; }
```

### INP — отзывчивый UI
```typescript
// Тяжёлые вычисления — в Web Worker
const worker = new Worker(new URL('./heavy.worker.ts', import.meta.url))
worker.postMessage({ data: largeDataset })
worker.onmessage = (e) => setResult(e.data)

// Или defer через scheduler
function handleClick() {
  // Срочное — сразу
  setLoading(true)

  // Несрочное — после paint
  scheduler.postTask(() => {
    processHeavyData()
  }, { priority: 'background' })
}

// Debounce для поиска
const debouncedSearch = useMemo(
  () => debounce((query: string) => search(query), 300),
  [],
)
```

---

## Изображения

```typescript
// Форматы по приоритету: AVIF > WebP > JPEG/PNG
// <picture> для fallback
<picture>
  <source srcSet="/img.avif" type="image/avif" />
  <source srcSet="/img.webp" type="image/webp" />
  <img src="/img.jpg" alt="..." width={800} height={600} loading="lazy" />
</picture>

// Next.js Image — автоматически
<Image
  src="/photo.jpg"
  alt="Photo"
  width={800}
  height={600}
  loading="lazy"       // по умолчанию
  placeholder="blur"   // показывает размытый placeholder
  blurDataURL="data:image/jpeg;base64,..."
/>
```

---

## Шрифты

```css
/* font-display: swap — текст виден сразу, шрифт подменяется */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom.woff2') format('woff2');
  font-display: swap;
  font-weight: 400;
}

/* Preload критичных шрифтов */
/* <link rel="preload" as="font" href="/fonts/custom.woff2" crossorigin> */
```

```typescript
// Next.js — встроенная оптимизация шрифтов
import { JetBrains_Mono } from 'next/font/google'

const mono = JetBrains_Mono({
  subsets: ['latin', 'cyrillic'],
  variable: '--font-mono',
  display: 'swap',
})
```