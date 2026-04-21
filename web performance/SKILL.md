---
name: web-performance
description: >
  A comprehensive, opinionated playbook for making websites load fast — merging
  canonical web perf best practices with real production techniques from a
  React app with a 114 KB main bundle. Covers bundle diet, intent-based
  prefetching, multi-tier caching, AI streaming, WebSocket sharing, Service
  Workers, rendering strategies, and Core Web Vitals. Use when asked to "speed
  up a site", "improve LCP/CLS/INP", "make this load faster", "optimize
  performance", reduce bundle size, or improve Lighthouse scores.
license: MIT
---

# Web Performance SKILL

> Goal: Ship pages that feel instant — perceived performance beats raw
> milliseconds every time.
>
> Source: canonical web perf best practices + @brotzky's production thread
> (React app, 114 KB main bundle).

---

## Table of Contents

1.  [Mental Model](#1-mental-model)
2.  [Audit First, Fix Second](#2-audit-first-fix-second)
3.  [Bundle Diet — Kill the Libraries](#3-bundle-diet--kill-the-libraries)
4.  [Resource Hints](#4-resource-hints)
5.  [Intent-Based Prefetching](#5-intent-based-prefetching)
6.  [Images](#6-images)
7.  [Fonts](#7-fonts)
8.  [JavaScript](#8-javascript)
9.  [CSS](#9-css)
10. [Caching — Multi-Tier Strategy](#10-caching--multi-tier-strategy)
11. [Streaming & Parallelism](#11-streaming--parallelism)
12. [Rendering](#12-rendering)
13. [WebSocket — One Per App](#13-websocket--one-per-app)
14. [Service Worker](#14-service-worker)
15. [Server / Rendering Strategy](#15-server--rendering-strategy)
16. [Core Web Vitals Cheat Sheet](#16-core-web-vitals-cheat-sheet)
17. [Quick Wins Checklist](#17-quick-wins-checklist)
18. [Common Mistakes to Avoid](#18-common-mistakes-to-avoid)
19. [Reference Links](#19-reference-links)

---

## 1. Mental Model

Performance = **fewer bytes** × **closer to the user** × **in the right order**.

Every optimisation falls into one of three buckets:

| Bucket | Goal | Examples |
|---|---|---|
| **Reduce** | Less data to transfer | Kill libraries, compress images, purge CSS |
| **Parallelize** | Start more work earlier | Preload, preconnect, parallel API calls |
| **Prioritize** | Critical path first | Inline critical CSS, defer non-blocking JS |

Always measure **real-user data** (CrUX / RUM) before lab data (Lighthouse).
Lab scores reflect your machine on a fast connection; CrUX shows what users
actually experience.

---

## 2. Audit First, Fix Second

### Tools (in priority order)

1. **[PageSpeed Insights](https://pagespeed.web.dev/)** — real CrUX field data
   + Lighthouse lab score in one click.
2. **[WebPageTest](https://www.webpagetest.org/)** — waterfall analysis,
   filmstrip view, throttled connections.
3. **[DebugBear](https://www.debugbear.com/)** — continuous monitoring with
   regression alerts.
4. **Chrome DevTools → Performance tab** — flame charts for runtime jank and
   long tasks.
5. **Chrome DevTools → Network tab** — waterfall, blocking resources, bundle
   sizes.

### What to read in the waterfall

- **TTFB > 800ms?** → Fix the server before touching anything else.
- **Long bars before first byte?** → DNS/TCP/TLS; add `preconnect`.
- **Red render-blocking bars?** → JS or CSS blocking paint; defer or inline.
- **Late LCP image?** → Missing `fetchpriority="high"` or `loading="lazy"` on
  the hero.
- **Layout shifts after load?** → Missing `width`/`height` or dynamic content
  injection above the fold.

---

## 3. Bundle Diet — Kill the Libraries

> @brotzky: "killed almost all libraries — using custom versions tailored to
> what I need. Main JS bundle: **114 KB** for a React app."

This is the most impactful long-term lever. A lean bundle cold-loads fast on
any device and any network.

### Step 1 — Audit what you are shipping

```bash
# Next.js built-in bundle analyser
ANALYZE=true next build

# Vite
npx vite-bundle-visualizer

# Check npm package cost before installing
npx bundlephobia <package-name>
```

Look for: large packages used for a single function, duplicate polyfills,
moment.js, full lodash.

### Step 2 — Replace or delete each library

| Common offender | What to do |
|---|---|
| `moment` (330 KB) | Replace with `date-fns` or native `Intl.DateTimeFormat` |
| `lodash` full build | Import from `lodash-es` (tree-shakeable) or write the 5-line util yourself |
| `axios` | `fetch` + a small wrapper is usually enough |
| `framer-motion` full | Use `@motionone/dom` or CSS transitions for simple animations |
| `chart.js` full | Use `lightweight-charts`, `uplot`, or lazy-import only needed controllers |
| Large UI kits | Import individual components, not the entire library |

### Step 3 — Write the custom minimal version

If you use 10% of a library, write those 10%. Example: replacing a full
date-picker library with a 50-line component covering your two actual use cases.

### Step 4 — Cookie-based routing in middleware

Move routing logic to middleware so the correct bundle is served before any JS
runs on the client — eliminating the "check auth, then redirect" waterfall.

```ts
// middleware.ts (Next.js)
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const token = request.cookies.get('auth-token');
  const { pathname } = request.nextUrl;

  // Unauthenticated user on a protected route → redirect at the edge
  if (!token && pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  // Authenticated user hitting login → skip straight to app
  if (token && pathname === '/login') {
    return NextResponse.redirect(new URL('/dashboard', request.url));
  }

  return NextResponse.next();
}
```

---

## 4. Resource Hints

Resource hints are the **highest ROI** optimisation you can add in minutes.
Place them in `<head>` before any stylesheets.

### `preconnect` — warm TCP+TLS handshake

```html
<!-- Fonts -->
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />

<!-- Your API + WebSocket server -->
<link rel="preconnect" href="https://api.yourapp.com" crossorigin />
<link rel="preconnect" href="wss://ws.yourapp.com" crossorigin />
```

**Rule:** Only preconnect to origins used within the first 2 seconds. Too many
preconnects waste TCP sockets.

---

### `preload` — fetch a critical resource immediately

```html
<!-- Hero image — most common LCP fix -->
<link
  rel="preload"
  as="image"
  href="/hero.webp"
  imagesrcset="/hero-480.webp 480w, /hero-960.webp 960w"
  imagesizes="100vw"
/>

<!-- Primary web font -->
<link
  rel="preload"
  as="font"
  href="/fonts/inter-var.woff2"
  type="font/woff2"
  crossorigin
/>
```

Limit to 3 preloads maximum — competing preloads slow the real critical path.

---

### `modulepreload` — preload ES module chunks

Better than `<link rel="preload" as="script">` for modern JS. The browser
parses and compiles the module immediately.

```html
<!-- Preload critical route chunks as modules -->
<link rel="modulepreload" href="/assets/dashboard.chunk.js" />
<link rel="modulepreload" href="/assets/shared-vendor.chunk.js" />
```

---

### `prefetch` — low-priority load for the next page

```html
<link rel="prefetch" href="/dashboard" />
<link rel="prefetch" as="script" href="/assets/dashboard.chunk.js" />
```

---

### `dns-prefetch` — DNS only (cheap, broad)

```html
<link rel="dns-prefetch" href="https://analytics.example.com" />
```

---

### `fetchpriority` — nudge browser resource priority

```html
<!-- Hero image: boost priority -->
<img src="/hero.webp" fetchpriority="high" alt="Hero" />

<!-- Below-fold images: lower priority -->
<img src="/card.webp" fetchpriority="low" loading="lazy" alt="Card" />

<!-- Non-critical preload: don't steal LCP bandwidth -->
<link rel="preload" as="script" href="/analytics.js" fetchpriority="low" />
```

---

## 5. Intent-Based Prefetching

> @brotzky: "login intent preloads routes + APIs before session resolves.
> Preload 7 critical APIs + preconnect WS. Hover-prefetch on tickers, chat,
> agents, screener."

Don't wait for the user to navigate — predict what they need and fetch early.

### Login-intent prefetch

```tsx
// While the user is on the login page, warm up the authenticated shell.
// By the time they hit Enter, routes and APIs are already in cache.

useEffect(() => {
  // Preconnect WebSocket server
  const link = document.createElement('link');
  link.rel = 'preconnect';
  link.href = 'wss://ws.yourapp.com';
  document.head.appendChild(link);

  // Prefetch dashboard JS chunk
  import(/* webpackPrefetch: true */ '../pages/Dashboard');

  // Prime React Query cache with critical API calls
  queryClient.prefetchQuery({ queryKey: ['user-profile'],   queryFn: fetchProfile });
  queryClient.prefetchQuery({ queryKey: ['critical-data'],  queryFn: fetchCriticalData });
}, []);
```

### Mount-time parallel API fetch — never waterfall

```ts
// Bad: sequential — each call waits for the previous
const user   = await fetchUser();
const charts = await fetchCharts();
const news   = await fetchNews();

// Good: all three fire simultaneously
const [user, charts, news] = await Promise.all([
  fetchUser(),
  fetchCharts(),
  fetchNews(),
]);
```

### Hover-prefetch — load data on pointer intent

```tsx
// On hover, fetch detail data. On click, it's already in cache.

function TickerCard({ ticker }: { ticker: string }) {
  const queryClient = useQueryClient();

  function handleMouseEnter() {
    queryClient.prefetchQuery({
      queryKey: ['ticker', ticker],
      queryFn: () => fetchTickerDetails(ticker),
      staleTime: 30_000, // skip refetch if already fresh
    });
  }

  return <div onMouseEnter={handleMouseEnter}>{ticker}</div>;
}
```

### Prefetch on WebSocket connect

```ts
// When WS opens, immediately request data the user needs in the next few seconds
socket.on('connect', () => {
  socket.emit('prefetch', { routes: ['portfolio', 'watchlist'] });
});
```

---

## 6. Images

Images are usually **the largest contributor to page weight**.

### Use modern formats

| Format | Use case | Savings vs JPEG |
|---|---|---|
| **WebP** | Photos, illustrations | ~30% |
| **AVIF** | Photos (best compression) | ~50% |
| **SVG** | Icons, logos | N/A (vector) |

```html
<picture>
  <source srcset="/hero.avif" type="image/avif" />
  <source srcset="/hero.webp" type="image/webp" />
  <img src="/hero.jpg" alt="Hero" width="1200" height="630" fetchpriority="high" />
</picture>
```

### Always set width & height — prevents CLS

```html
<img src="/photo.webp" width="800" height="450" alt="Photo" />
```

### Lazy-load below-fold only — never the hero

```html
<!-- Below fold: safe to lazy-load -->
<img src="/card.webp" loading="lazy" width="400" height="300" alt="Card" />

<!-- Hero / LCP image: never lazy-load -->
<img src="/hero.webp" fetchpriority="high" alt="Hero" width="1200" height="630" />
```

### Responsive images

```html
<img
  src="/hero-960.webp"
  srcset="/hero-480.webp 480w, /hero-960.webp 960w, /hero-1440.webp 1440w"
  sizes="(max-width: 600px) 480px, (max-width: 1200px) 960px, 1440px"
  alt="Hero"
  width="960"
  height="540"
  fetchpriority="high"
/>
```

---

## 7. Fonts

### Self-host with a variable font

```css
/* One .woff2 file covers all weights — no Google Fonts round-trip */
@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter-var.woff2') format('woff2');
  font-weight: 100 900;
  font-style: normal;
  font-display: swap; /* show system font while loading — never invisible text */
}
```

### Preload the primary font

```html
<link
  rel="preload"
  as="font"
  href="/fonts/inter-var.woff2"
  type="font/woff2"
  crossorigin
/>
```

### Subset to used characters

```bash
pyftsubset inter.ttf \
  --output-file=inter-subset.woff2 \
  --flavor=woff2 \
  --unicodes="U+0020-007F,U+00A0-00FF"
```

Font caching (30 days) is covered in [Section 14 — Service Worker](#14-service-worker).

---

## 8. JavaScript

### Defer / async all scripts

```html
<!-- App bundle — runs after HTML is parsed, non-blocking -->
<script src="/app.js" defer></script>

<!-- Independent third-party scripts -->
<script src="/analytics.js" async></script>
```

Rule: `defer` for your own code, `async` for independent third-party scripts.

### Load third-party scripts after `load`

```html
<script>
  window.addEventListener('load', () => {
    const s = document.createElement('script');
    s.src = 'https://analytics.example.com/tracker.js';
    s.async = true;
    document.head.appendChild(s);
  });
</script>
```

Next.js:

```jsx
import Script from 'next/script';

// Safe default — fires after hydration
<Script src="https://widget.example.com" strategy="afterInteractive" />

// Non-critical — fires when browser is idle
<Script src="https://chat.example.com" strategy="lazyOnload" />
```

### Tree-shake & dynamic import

```js
// Import only what you use
import { debounce } from 'lodash-es';

// Lazy-load heavy routes
const Dashboard = React.lazy(() => import('./Dashboard'));

// Prefetch a chunk when browser is idle
const Chart = React.lazy(
  () => import(/* webpackPrefetch: true */ './HeavyChart')
);
```

### Break long tasks (> 50 ms)

```js
// Yield to the browser between expensive iterations
async function processItems(items) {
  for (const item of items) {
    processItem(item);
    await new Promise(r => setTimeout(r, 0)); // yield
  }
}

// Scheduler API — modern browsers
await scheduler.postTask(() => heavyWork(), { priority: 'background' });
```

---

## 9. CSS

### Inline critical CSS — load the rest async

```html
<head>
  <style>
    /* Only above-the-fold styles */
    body { margin: 0; font-family: Inter, sans-serif; }
    .hero { min-height: 100vh; background: #000; }
  </style>

  <link rel="preload" href="/styles.css" as="style"
        onload="this.rel='stylesheet'" />
  <noscript><link rel="stylesheet" href="/styles.css" /></noscript>
</head>
```

Tools: [Critical](https://github.com/addyosmani/critical), [Penthouse](https://github.com/pocketjoso/penthouse).

### Purge unused CSS

```bash
npx purgecss --css styles.css --content "**/*.html" --output purged/
```

Tailwind does this automatically in production.

### Avoid layout thrashing

```js
// Bad: read → write → read forces two reflows
const h = el.offsetHeight;   // read
el.style.height = h + 'px';  // write
const h2 = el.offsetHeight;  // read — triggers another reflow

// Good: batch reads then writes
const h = el.offsetHeight;
requestAnimationFrame(() => { el.style.height = h + 'px'; });
```

---

## 10. Caching — Multi-Tier Strategy

> @brotzky: "SWR cache tiers (30s–1d) headers. localStorage → React Query
> hydration (prices: 5 min). Gemini context cache (30 min KV). AI summaries +
> chart data cached/shared across users."

Design your cache as a stack: **memory → localStorage → HTTP → CDN → server**.

### Tier 1 — React Query in-memory cache

```ts
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 30_000,       // fresh for 30s — no refetch during this window
      gcTime:  5 * 60 * 1000,  // keep in memory 5 min after unmount
    },
  },
});
```

### Tier 2 — localStorage hydration (persist across reloads)

```ts
import { persistQueryClient } from '@tanstack/react-query-persist-client';
import { createSyncStoragePersister } from '@tanstack/query-sync-storage-persister';

// Hydrate React Query from localStorage on boot — instant paint on cache hit
persistQueryClient({
  queryClient,
  persister: createSyncStoragePersister({ storage: window.localStorage }),
  maxAge: 5 * 60 * 1000, // prices stay fresh for 5 minutes
});
```

### Tier 3 — HTTP SWR cache headers

```
# Real-time data (prices) — short cache, long background revalidation
Cache-Control: public, max-age=30, stale-while-revalidate=86400

# Semi-static data (charts, summaries)
Cache-Control: public, max-age=300, stale-while-revalidate=86400

# Immutable hashed assets — cache forever
Cache-Control: public, max-age=31536000, immutable

# HTML — always revalidate
Cache-Control: no-cache
```

### Tier 4 — Shared cache (all users share computed results)

AI summaries and chart data are expensive to generate. Compute once and serve
to every user from a shared KV store.

```ts
// Edge KV / Redis — keyed by content, not user
async function getTickerSummary(ticker: string) {
  const cacheKey = `summary:${ticker}`;
  const cached = await kv.get(cacheKey);
  if (cached) return JSON.parse(cached);

  const summary = await generateAISummary(ticker); // expensive
  await kv.set(cacheKey, JSON.stringify(summary), { ex: 1800 }); // 30 min TTL
  return summary;
}
```

### Tier 5 — LLM context cache

For AI features sending the same large system prompt on every request:

```ts
// Claude: cache_control marks the prefix as reusable
{
  model: 'claude-opus-4-5',
  system: [
    {
      type: 'text',
      text: LARGE_SYSTEM_PROMPT,           // cached — not re-billed
      cache_control: { type: 'ephemeral' } // 5-min TTL on Anthropic infra
    }
  ],
  messages: [{ role: 'user', content: userMessage }]
}
```

---

## 11. Streaming & Parallelism

> @brotzky: "real token streaming (no fake chunks), anti-buffering headers.
> Parallel tool calls + batched prompts. Client-side API batches over a single
> endpoint."

### Real token streaming — disable all buffering

```ts
// Next.js route handler — stream tokens to the client as they arrive
export async function POST(req: Request) {
  const stream = await anthropic.messages.stream({ /* ... */ });

  return new Response(stream.toReadableStream(), {
    headers: {
      'Content-Type':      'text/event-stream',
      'Cache-Control':     'no-cache, no-transform', // prevent proxy buffering
      'X-Accel-Buffering': 'no',                     // disable Nginx buffering
      'Connection':        'keep-alive',
    },
  });
}
```

### Parallel LLM tool calls

```ts
// Bad: sequential — each call waits for the previous
const summary   = await llm.call({ tool: 'summarize' });
const sentiment = await llm.call({ tool: 'sentiment' });

// Good: fire all independent calls simultaneously
const [summary, sentiment, keywords] = await Promise.all([
  llm.call({ tool: 'summarize' }),
  llm.call({ tool: 'sentiment' }),
  llm.call({ tool: 'keywords'  }),
]);
```

### Client-side API request batching

N parallel fetch calls = N round-trips. Batch them into one:

```ts
// Client — sends multiple requests in a single HTTP call
async function batchFetch(requests: BatchRequest[]) {
  const res = await fetch('/api/batch', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ requests }),
  });
  return res.json(); // returns array in same order as requests
}

// Usage — one round-trip for three data points
const [prices, news, chart] = await batchFetch([
  { endpoint: '/prices', params: { ticker: 'AAPL' } },
  { endpoint: '/news',   params: { ticker: 'AAPL' } },
  { endpoint: '/chart',  params: { ticker: 'AAPL', range: '1d' } },
]);
```

```ts
// Server — /api/batch route handler
export async function POST(req: Request) {
  const { requests } = await req.json();

  // Execute all sub-requests in parallel on the server
  const results = await Promise.all(
    requests.map(({ endpoint, params }) => internalFetch(endpoint, params))
  );

  return Response.json(results);
}
```

---

## 12. Rendering

> @brotzky: "lazy charts w/ reserved height (no layout shift). Tab-level lazy
> loading. Memoized list items. Instant paint on cache hits. Extremely
> lightweight DOM."

### Lazy charts with reserved height — zero CLS

```tsx
const PriceChart = lazy(() => import('./PriceChart'));

function StockPage() {
  return (
    // Reserve exact height before chart loads → zero layout shift
    <div style={{ minHeight: 320 }}>
      <Suspense fallback={<ChartSkeleton height={320} />}>
        <PriceChart />
      </Suspense>
    </div>
  );
}
```

### Tab-level lazy loading

```tsx
// Only mount the active tab — inactive tabs don't execute or render
const tabs = {
  overview:   lazy(() => import('./tabs/Overview')),
  financials: lazy(() => import('./tabs/Financials')),
  news:       lazy(() => import('./tabs/News')),
};

function StockTabs({ activeTab }) {
  const TabContent = tabs[activeTab];
  return (
    <Suspense fallback={<TabSkeleton />}>
      <TabContent />
    </Suspense>
  );
}
```

### Memoized list items

```tsx
// Prevent re-renders of every row when parent state changes
const TickerRow = React.memo(function TickerRow({ ticker, price, change }) {
  return (
    <tr>
      <td>{ticker}</td>
      <td>{price}</td>
      <td className={change >= 0 ? 'green' : 'red'}>{change}%</td>
    </tr>
  );
});
```

### Instant paint on cache hit

```tsx
// placeholderData shows stale data immediately — user never sees a blank screen
function Dashboard() {
  const { data } = useQuery({
    queryKey: ['portfolio'],
    queryFn: fetchPortfolio,
    staleTime: 30_000,
    placeholderData: (prev) => prev, // keep showing previous data during refetch
  });

  return <PortfolioTable data={data} />;
}
```

### Keep the DOM lean (< 1500 nodes) — virtualise long lists

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';

function TickerList({ tickers }) {
  const parentRef = useRef(null);
  const virtualizer = useVirtualizer({
    count: tickers.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 48, // row height in px
  });

  return (
    <div ref={parentRef} style={{ height: 600, overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize() }}>
        {virtualizer.getVirtualItems().map(virtualItem => (
          <TickerRow
            key={virtualItem.key}
            style={{ transform: `translateY(${virtualItem.start}px)` }}
            ticker={tickers[virtualItem.index]}
          />
        ))}
      </div>
    </div>
  );
}
```

---

## 13. WebSocket — One Per App

> @brotzky: "single shared WS across the app. Smart reconnect + visibility
> gating. Prefetch on connect."

One shared WS connection is dramatically cheaper than per-component connections.

```ts
// ws-manager.ts — singleton WebSocket shared across the entire app

let socket: WebSocket | null = null;
const listeners = new Map<string, Set<(data: unknown) => void>>();

/** Returns the open socket, or creates one. */
export function getSocket(): WebSocket {
  if (socket?.readyState === WebSocket.OPEN) return socket;
  return createSocket();
}

function createSocket(): WebSocket {
  socket = new WebSocket('wss://ws.yourapp.com');

  socket.addEventListener('open', () => {
    // Prefetch initial data on connect — don't wait for components to request
    socket!.send(JSON.stringify({ type: 'prefetch', topics: ['portfolio', 'watchlist'] }));
  });

  socket.addEventListener('message', (event) => {
    const { type, data } = JSON.parse(event.data);
    // Fan out to all registered listeners for this message type
    listeners.get(type)?.forEach(cb => cb(data));
  });

  socket.addEventListener('close', () => {
    // Visibility gating — don't reconnect in a hidden/background tab
    if (document.visibilityState === 'visible') {
      setTimeout(createSocket, 2000); // add exponential backoff in production
    } else {
      document.addEventListener('visibilitychange', onVisible, { once: true });
    }
  });

  return socket;
}

/** Resume reconnect when user returns to the tab. */
function onVisible() {
  if (document.visibilityState === 'visible') createSocket();
}

/** Subscribe to a message type. Returns an unsubscribe function. */
export function subscribe(type: string, cb: (data: unknown) => void) {
  if (!listeners.has(type)) listeners.set(type, new Set());
  listeners.get(type)!.add(cb);
  return () => listeners.get(type)?.delete(cb);
}
```

---

## 14. Service Worker

> @brotzky: "app shell precached. NetworkFirst nav (3s timeout). Fonts cached
> 30d. Idle registration."

### Register on idle — never block the first load

```ts
// Only register once the browser has nothing more important to do
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    requestIdleCallback(() => {
      navigator.serviceWorker.register('/sw.js');
    });
  });
}
```

### Caching strategies (Workbox)

```js
// sw.js

import { registerRoute }         from 'workbox-routing';
import { CacheFirst, NetworkFirst, StaleWhileRevalidate } from 'workbox-strategies';
import { precacheAndRoute }      from 'workbox-precaching';

// 1. App shell — precached on install, loads instantly offline
precacheAndRoute(self.__WB_MANIFEST);

// 2. Navigation — NetworkFirst with 3s timeout, fall back to cache
registerRoute(
  ({ request }) => request.mode === 'navigate',
  new NetworkFirst({ networkTimeoutSeconds: 3 })
);

// 3. Fonts — CacheFirst, 30-day TTL (never hit the network after first load)
registerRoute(
  ({ request }) => request.destination === 'font',
  new CacheFirst({
    cacheName: 'fonts',
    plugins: [{ cacheExpiration: { maxAgeSeconds: 30 * 24 * 60 * 60 } }],
  })
);

// 4. Static assets (JS / CSS / images) — serve from cache, refresh in background
registerRoute(
  ({ request }) => ['script', 'style', 'image'].includes(request.destination),
  new StaleWhileRevalidate({ cacheName: 'static-assets' })
);

// 5. API responses — NetworkFirst, short cache for freshness
registerRoute(
  ({ url }) => url.pathname.startsWith('/api/'),
  new NetworkFirst({ cacheName: 'api-cache' })
);
```

---

## 15. Server / Rendering Strategy

| Strategy | Best for | LCP | Dynamic |
|---|---|---|---|
| **SSG** | Marketing, docs, blogs | ⚡ Best | ❌ |
| **ISR** | E-commerce, news feeds | ⚡ Fast | ✅ Partial |
| **SSR** | Per-user dashboards | 🟡 Depends on TTFB | ✅ |
| **CSR** | Authenticated-only apps | 🔴 Worst | ✅ |

**Rule:** Default to SSG/ISR. Use SSR only when content must be per-user at
request time. Never CSR for public-facing pages.

### Reduce TTFB

- Host close to your users, or use edge (Vercel Edge, Cloudflare Workers).
- Enable HTTP/2 — multiplexes requests over one TCP connection.
- Cache expensive DB queries server-side.
- Fix N+1 queries — the #1 hidden TTFB killer.

### Enable Brotli on CDN

```
Content-Encoding: br   # ~15-20% smaller than gzip for text assets
```

---

## 16. Core Web Vitals Cheat Sheet

| Metric | Measures | Target | Top fixes |
|---|---|---|---|
| **LCP** | Largest element load time | < 2.5s | Preload hero + `fetchpriority="high"`, fast TTFB, no lazy-load on LCP |
| **INP** | Click/tap responsiveness | < 200ms | Break long tasks, defer JS, keep DOM < 1500 nodes, virtualise lists |
| **CLS** | Visual stability | < 0.1 | `width`/`height` on images, reserve chart height, `font-display: swap` |

### LCP debug

1. Is the LCP an image? → Preload with `fetchpriority="high"`.
2. Is it `loading="lazy"`? → Remove it.
3. URL only appears after JS runs? → Move it to the HTML.
4. TTFB > 800ms? → Fix the server first.

### INP debug

1. Long task > 50ms? → Break it up or move to a Web Worker.
2. Third-party script blocking? → Load after `window.load`.
3. DOM > 1500 nodes? → Virtualise long lists.

### CLS debug

1. Images without dimensions? → Add `width` and `height`.
2. Ads/banners injected dynamically? → Reserve space with `min-height`.
3. Font reflow? → `font-display: swap` + preload.
4. Chart loading without height reservation? → Always set `minHeight` on the container.

---

## 17. Quick Wins Checklist

```
BUNDLE
[ ] Run bundle analyser — identify the top 3 heaviest packages
[ ] Replace moment → date-fns or Intl.DateTimeFormat
[ ] Replace full lodash → lodash-es tree-shaking or custom utils
[ ] Add cookie-based routing in middleware (eliminate client-side auth redirect)
[ ] Add modulepreload for top 2-3 JS chunks

RESOURCE HINTS
[ ] preconnect to every third-party origin used above the fold
[ ] preconnect to your WebSocket server
[ ] preload the LCP image with fetchpriority="high"
[ ] preload the primary font file
[ ] Remove loading="lazy" from the LCP/hero image

INTENT PREFETCH
[ ] Prefetch authenticated routes + critical APIs on login page load
[ ] Add hover-prefetch to high-click-rate list items
[ ] Prefetch data on WebSocket connect

IMAGES
[ ] Convert hero and card images to WebP or AVIF
[ ] Add width & height to every <img> tag
[ ] Lazy-load all images below the fold

FONTS
[ ] Self-host with font-display: swap
[ ] Subset font to used Unicode ranges

JS / CSS
[ ] defer or async all <script> tags
[ ] Load analytics + tag manager after window.load
[ ] Inline critical above-fold CSS, load rest async
[ ] PurgeCSS / Tailwind purge unused styles

CACHING
[ ] SWR Cache-Control headers tuned per data type (30s–1d)
[ ] localStorage → React Query hydration for key data
[ ] Shared KV cache for AI summaries + chart data
[ ] LLM context cache for repeated large system prompts
[ ] Cache-Control: immutable on hashed static assets
[ ] Enable Brotli on CDN

STREAMING
[ ] Add X-Accel-Buffering: no + Cache-Control: no-transform on SSE routes
[ ] Parallel tool calls with Promise.all
[ ] Batch client API requests over a single endpoint

RENDERING
[ ] Reserve minHeight on lazy chart containers (prevent CLS)
[ ] Tab-level lazy loading — only mount the active tab
[ ] React.memo on frequently re-rendered list items
[ ] Virtualise lists with > 100 rows

WEBSOCKET
[ ] Single shared WS singleton across the entire app
[ ] Visibility gating — don't reconnect in background tabs
[ ] Prefetch on WS connect

SERVICE WORKER
[ ] Idle-register the service worker (requestIdleCallback)
[ ] Precache app shell
[ ] Fonts: CacheFirst with 30-day TTL
[ ] Navigation: NetworkFirst with 3s timeout
```

---

## 18. Common Mistakes to Avoid

| Mistake | Why it hurts | Fix |
|---|---|---|
| `loading="lazy"` on the hero | Delays LCP — browser defers fetch | Remove from above-fold images |
| Preloading > 3 resources | Resources compete; critical path slows | Preload ≤ 3; use `prefetch` for the rest |
| `preconnect` to unused origins | Wastes TCP sockets | Only origins used in first 2s |
| Importing full lodash / moment | Massive bundle bloat | lodash-es tree-shaking or write the util |
| Per-component WebSocket connections | N×TCP overhead, reconnect chaos | One shared WS singleton |
| Chart mounting without reserved height | CLS — content jumps on load | Always set `minHeight` on the container |
| Registering Service Worker eagerly | Competes with first load | Register in `requestIdleCallback` |
| Fake AI streaming (buffer then flush) | Terrible perceived latency | Real streaming + `X-Accel-Buffering: no` |
| Sequential API calls | Unnecessary added latency | `Promise.all` for independent calls |
| SSR for every page | High TTFB for static content | SSG/ISR by default; SSR only for per-user |
| No `width`/`height` on images | CLS layout jumps | Always specify dimensions |
| Lighthouse only, no real user data | Lab scores ≠ real experience | CrUX / PageSpeed Insights field data |
| Fixing everything at once | Can't tell what helped | One change at a time, measure each |

---

## 19. Reference Links

- [web.dev/explore/fast](https://web.dev/explore/fast) — Google's canonical performance guide
- [web.dev/articles/lcp](https://web.dev/articles/lcp) — LCP deep dive
- [web.dev/articles/inp](https://web.dev/articles/inp) — INP deep dive
- [web.dev/articles/cls](https://web.dev/articles/cls) — CLS deep dive
- [PageSpeed Insights](https://pagespeed.web.dev/) — Real user + lab score
- [WebPageTest](https://www.webpagetest.org/) — Waterfall analysis
- [DebugBear](https://www.debugbear.com/) — Continuous monitoring
- [Squoosh](https://squoosh.app/) — Image compression playground
- [Bundlephobia](https://bundlephobia.com/) — npm package cost checker
- [TanStack Query persist](https://tanstack.com/query/latest/docs/framework/react/plugins/persistQueryClient) — localStorage hydration
- [Workbox](https://developer.chrome.com/docs/workbox) — Service Worker tooling
- [@tanstack/react-virtual](https://tanstack.com/virtual/latest) — DOM virtualisation for long lists
