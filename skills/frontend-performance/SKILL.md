---
name: frontend-performance
description: >
  Frontend rendering and loading performance optimization for web applications.
  Use when: frontend performance, web performance, Core Web Vitals, LCP, FID, INP, CLS, TTFB,
  page load optimization, bundle size, code splitting, lazy loading, image optimization,
  render blocking resources, critical CSS, font loading, JavaScript performance, React performance,
  Vue performance, Angular performance, hydration, SSR performance, SSG, ISR, tree shaking,
  Lighthouse audit, WebPageTest, performance budget, resource hints, prefetch, preload, preconnect,
  service worker, HTTP/2, HTTP/3, compression, Brotli, gzip, CDN performance, edge caching,
  long task, main thread blocking, requestAnimationFrame, virtual scroll, layout thrashing,
  memory leak frontend, infinite scroll performance, webpack bundle analyzer, Vite build.
argument-hint: >
  Describe the framework (React, Vue, Next.js, etc.), the performance symptom (e.g., "LCP > 4s",
  "bundle is 3MB", "scroll is janky on mobile"), and whether SSR/SSG/SPA is in use.
  Include Lighthouse scores or WebPageTest results if available.
---

# Frontend Performance Specialist

## When to Use

Invoke this skill when you need to:
- Diagnose and improve Core Web Vitals (LCP, INP, CLS)
- Reduce JavaScript bundle size via code splitting and tree shaking
- Optimize image delivery and font loading
- Improve Time to First Byte (TTFB) through SSR, SSG, or edge caching
- Eliminate render-blocking resources
- Fix React/Vue/Angular-specific rendering performance issues

---

## Step 1 — Measure Before Optimizing

**Core Web Vitals targets (Google "Good" thresholds):**

| Metric | What It Measures | Good | Needs Improvement | Poor |
|---|---|---|---|---|
| **LCP** (Largest Contentful Paint) | Time to render the largest visible element | < 2.5s | 2.5s–4s | > 4s |
| **INP** (Interaction to Next Paint) | Response time of user interactions | < 200ms | 200ms–500ms | > 500ms |
| **CLS** (Cumulative Layout Shift) | Visual stability — unexpected layout shifts | < 0.1 | 0.1–0.25 | > 0.25 |
| **TTFB** (Time to First Byte) | Server response time | < 800ms | 800ms–1800ms | > 1800ms |
| **FCP** (First Contentful Paint) | First meaningful pixel | < 1.8s | 1.8s–3s | > 3s |

**Measurement tools:**

| Tool | Use For |
|---|---|
| Chrome DevTools → Lighthouse | Local audit; actionable recommendations |
| WebPageTest (webpagetest.org) | Real network conditions; film strip view |
| PageSpeed Insights | Field data (CrUX) + lab data for a URL |
| Chrome DevTools → Performance tab | Main thread profiling; long tasks |
| `web-vitals` JS library | Real-user monitoring (RUM) in production |

```js
// Monitor Core Web Vitals in production
import { onLCP, onINP, onCLS, onFCP, onTTFB } from 'web-vitals';

function sendToAnalytics({ name, value, id }) {
  fetch('/api/vitals', {
    method: 'POST',
    body: JSON.stringify({ metric: name, value, id, url: location.href }),
  });
}

onLCP(sendToAnalytics);
onINP(sendToAnalytics);
onCLS(sendToAnalytics);
```

Checklist:
- [ ] Lighthouse audit run against the production URL — not localhost
- [ ] Field data (CrUX) checked in PageSpeed Insights — real-user experience may differ from lab
- [ ] `web-vitals` library integrated — real-user metric data collected continuously
- [ ] Performance budget defined: LCP < 2.5s, INP < 200ms, bundle < 200KB per route

---

## Step 2 — Reduce JavaScript Bundle Size

A large JavaScript bundle is the most common LCP and INP killer.

**Analyze the bundle:**
```bash
# Vite
npx vite build --mode production
npx vite-bundle-visualizer

# Webpack
npx webpack-bundle-analyzer dist/stats.json

# Next.js
ANALYZE=true next build  # requires @next/bundle-analyzer
```

**Code splitting — load only what the current route needs:**
```js
// ❌ Imports the entire library on page load
import { Chart } from 'chart.js';

// ✅ Route-level code splitting (React)
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Chart = lazy(() => import('./components/Chart'));

// ✅ Dynamic import on user interaction
button.addEventListener('click', async () => {
  const { default: Modal } = await import('./Modal');
  Modal.open();
});
```

**Tree shaking — import only what you use:**
```js
// ❌ Imports entire lodash (~70KB)
import _ from 'lodash';
_.debounce(fn, 300);

// ✅ Import only the function needed (~1KB)
import debounce from 'lodash/debounce';
// or use the lodash-es package with named imports
import { debounce } from 'lodash-es';
```

**Replace heavy libraries:**

| Library (heavy) | Lightweight alternative |
|---|---|
| Moment.js (67KB) | `date-fns` (tree-shakeable) or `Temporal` API |
| Lodash full (70KB) | Native JS methods + `lodash-es` selective imports |
| Axios (13KB) | Native `fetch` API |
| jQuery (87KB) | Vanilla JS |
| Chart.js full (200KB) | Chart.js selective import or `lightweight-charts` |

Checklist:
- [ ] Bundle visualizer run — top 5 contributors to bundle size identified
- [ ] Route-level code splitting applied — no single JS bundle > 200KB for initial load
- [ ] Dynamic imports used for modals, charts, and other interaction-triggered features
- [ ] Tree-shakeable imports used for all utility libraries
- [ ] `node_modules` scanned for accidental full-library imports

---

## Step 3 — Optimize Images

Images are typically the largest resource on a page and the primary LCP element.

**Image optimization checklist:**

| Practice | How |
|---|---|
| Modern formats | Use WebP or AVIF — 25–50% smaller than JPEG/PNG |
| Responsive sizes | `srcset` and `sizes` — serve the right size per viewport |
| Lazy loading | `loading="lazy"` on below-the-fold images |
| Prioritize LCP image | `fetchpriority="high"` on the hero/LCP image |
| Explicit dimensions | Always set `width` and `height` — prevents CLS |
| CDN delivery | Serve from a CDN with edge caching; use image transformation APIs |

```html
<!-- LCP image — load eagerly, highest priority -->
<img
  src="/hero.webp"
  srcset="/hero-400.webp 400w, /hero-800.webp 800w, /hero-1200.webp 1200w"
  sizes="(max-width: 600px) 400px, (max-width: 1200px) 800px, 1200px"
  width="1200"
  height="630"
  fetchpriority="high"
  alt="Product hero image"
/>

<!-- Below-the-fold image — lazy load -->
<img
  src="/product.webp"
  width="400" height="300"
  loading="lazy"
  alt="Product detail"
/>
```

**Next.js `<Image>` — handles all of the above automatically:**
```jsx
import Image from 'next/image';

<Image
  src="/hero.jpg"
  width={1200}
  height={630}
  priority          // LCP image — sets fetchpriority="high" and preloads
  alt="Hero image"
/>
```

Checklist:
- [ ] Images served in WebP or AVIF format
- [ ] LCP image identified and marked with `fetchpriority="high"` or `priority` prop
- [ ] `loading="lazy"` on all below-the-fold images
- [ ] All images have explicit `width` and `height` attributes — prevents CLS
- [ ] Images served from a CDN with appropriate cache headers (`Cache-Control: public, max-age=31536000, immutable`)

---

## Step 4 — Eliminate Render-Blocking Resources

Render-blocking CSS and JS delay FCP and LCP.

**CSS — inline critical, defer the rest:**
```html
<!-- Critical CSS inlined in <head> — renders above-the-fold content immediately -->
<style>
  /* Only above-the-fold styles */
  body { margin: 0; font-family: system-ui; }
  .hero { background: #000; color: #fff; padding: 2rem; }
</style>

<!-- Non-critical CSS loaded asynchronously -->
<link rel="preload" href="/styles.css" as="style" onload="this.rel='stylesheet'">
<noscript><link rel="stylesheet" href="/styles.css"></noscript>
```

**JavaScript — defer or async:**
```html
<!-- ❌ Render-blocking — parser stops here -->
<script src="/app.js"></script>

<!-- ✅ Defer — executes after HTML parsed; maintains order -->
<script src="/app.js" defer></script>

<!-- ✅ Async — executes as soon as downloaded; no order guarantee -->
<script src="/analytics.js" async></script>
```

**Resource hints — start connections and downloads early:**
```html
<!-- Preconnect: establish TCP+TLS to critical third-party origins -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://cdn.myapp.com" crossorigin>

<!-- Preload: download a resource early (use sparingly — competes for bandwidth) -->
<link rel="preload" href="/hero.webp" as="image" fetchpriority="high">
<link rel="preload" href="/inter-var.woff2" as="font" type="font/woff2" crossorigin>

<!-- Prefetch: hint for next navigation (low priority) -->
<link rel="prefetch" href="/checkout">
```

**Font loading — minimize layout shift:**
```css
/* Use font-display: swap — show fallback font; replace when ready */
@font-face {
    font-family: 'Inter';
    src: url('/fonts/inter-var.woff2') format('woff2');
    font-display: swap;
    font-weight: 100 900;
}

/* Size-adjust to reduce CLS from font-swap */
@font-face {
    font-family: 'Inter-fallback';
    src: local('Arial');
    size-adjust: 107%;
    ascent-override: 90%;
}
```

Checklist:
- [ ] No render-blocking `<script>` tags in `<head>` without `defer` or `async`
- [ ] Critical CSS inlined; non-critical CSS loaded asynchronously
- [ ] `<link rel="preconnect">` added for all critical third-party origins (fonts, CDN, analytics)
- [ ] LCP image preloaded via `<link rel="preload">`
- [ ] Web fonts use `font-display: swap` or `font-display: optional`

---

## Step 5 — Fix INP (Interaction Responsiveness)

INP measures how quickly the page responds to all user interactions. Long tasks on the main thread cause poor INP.

**Find long tasks:**
```js
// Long Task Observer — logs tasks > 50ms
const observer = new PerformanceObserver((list) => {
    list.getEntries().forEach(entry => {
        console.warn('Long task detected', {
            duration: entry.duration,
            startTime: entry.startTime,
        });
    });
});
observer.observe({ type: 'longtask', buffered: true });
```

**Break up long tasks:**
```js
// ❌ Blocks main thread for 500ms
function processLargeList(items) {
    items.forEach(item => heavyComputation(item));
}

// ✅ Yield to the browser between chunks
async function processLargeList(items) {
    const CHUNK_SIZE = 100;
    for (let i = 0; i < items.length; i += CHUNK_SIZE) {
        const chunk = items.slice(i, i + CHUNK_SIZE);
        chunk.forEach(item => heavyComputation(item));
        await yieldToMain(); // let the browser handle user input
    }
}

function yieldToMain() {
    return new Promise(resolve => setTimeout(resolve, 0));
}
```

**React — avoid unnecessary re-renders:**
```jsx
// ❌ Re-renders on every parent render
function ExpensiveList({ items }) {
    return items.map(item => <Item key={item.id} data={item} />);
}

// ✅ Memoize component — only re-renders when items changes
const ExpensiveList = memo(function ExpensiveList({ items }) {
    return items.map(item => <Item key={item.id} data={item} />);
});

// ✅ Memoize expensive computation
const sortedItems = useMemo(() => items.sort(compare), [items]);

// ✅ Stable callback reference
const handleClick = useCallback(() => doSomething(id), [id]);
```

**Virtualize long lists:**
```jsx
import { useVirtualizer } from '@tanstack/react-virtual';

// Only render visible rows — not all 10,000
function VirtualList({ items }) {
    const parentRef = useRef(null);
    const virtualizer = useVirtualizer({
        count: items.length,
        getScrollElement: () => parentRef.current,
        estimateSize: () => 48,
    });
    return (
        <div ref={parentRef} style={{ overflow: 'auto', height: '600px' }}>
            <div style={{ height: virtualizer.getTotalSize() }}>
                {virtualizer.getVirtualItems().map(row => (
                    <div key={row.key} style={{ transform: `translateY(${row.start}px)` }}>
                        {items[row.index].name}
                    </div>
                ))}
            </div>
        </div>
    );
}
```

Checklist:
- [ ] Long tasks (> 50ms) identified via Performance tab or PerformanceObserver
- [ ] Expensive computations moved off the main thread (Web Worker) or broken into async chunks
- [ ] React `memo`, `useMemo`, `useCallback` applied to expensive components and computations
- [ ] Lists with > 100 items virtualized — `@tanstack/react-virtual` or `react-window`

---

## Step 6 — Fix CLS (Layout Stability)

CLS is caused by elements appearing or resizing unexpectedly after initial paint.

**Common CLS causes and fixes:**

| Cause | Fix |
|---|---|
| Images without dimensions | Always set `width` and `height` on `<img>` |
| Fonts swapping | Use `font-display: swap` + size-adjust fallback |
| Dynamic content injected above existing content | Reserve space with `min-height`; insert below the fold |
| Ad slots without reserved space | Set explicit dimensions on the container |
| Animation using `top`/`left`/`margin` | Animate `transform` only — GPU-composited, no layout |

```css
/* ❌ Animating layout properties — triggers layout recalculation */
.slide-in { transition: margin-left 0.3s; margin-left: -100%; }
.slide-in.active { margin-left: 0; }

/* ✅ Animating transform — GPU layer, no CLS impact */
.slide-in { transition: transform 0.3s; transform: translateX(-100%); }
.slide-in.active { transform: translateX(0); }
```

Checklist:
- [ ] All images have explicit `width` and `height` set in HTML
- [ ] Ad and embed slots have reserved dimensions — no layout shifts when content loads
- [ ] Animations use `transform` and `opacity` only — no `margin`, `top`, `left`, `width` transitions
- [ ] Dynamic content (toasts, banners) inserted below existing content or in reserved space

---

## Output Report

### Critical
- LCP > 4s on mobile — most users abandon; significant SEO and conversion impact
- Entire application bundle loaded on first page — 3MB+ JavaScript blocking interaction
- LCP image not prioritized — browser discovers it late; LCP delayed by 2–3 seconds

### High
- Code splitting absent — every route loads the full bundle; initial LCP delayed
- Images served as JPEG/PNG without WebP fallback — 30–50% larger than necessary
- Render-blocking scripts in `<head>` — FCP delayed until all scripts download and execute

### Medium
- No `loading="lazy"` on below-the-fold images — bandwidth wasted on images users may not scroll to
- React components not memoized — unnecessary re-renders degrade INP on interaction
- Lists of > 200 items rendered in the DOM — scroll performance and memory usage degrade

### Low
- No `font-display: swap` on web fonts — invisible text during font download (FOIT)
- `<link rel="preconnect">` missing for CDN and font origins — extra DNS + TCP round trip on first request
- No real-user performance monitoring — lab data only; field performance unknown

### Passed
- Core Web Vitals measured via Lighthouse and `web-vitals` RUM; all metrics in "Good" range
- Route-level code splitting applied; initial JS bundle < 200KB; dynamic imports for interaction-triggered features
- LCP image identified, preloaded, and marked `fetchpriority="high"`; all images in WebP with explicit dimensions
- Render-blocking scripts use `defer`; critical CSS inlined; resource hints configured
- Long tasks broken into async chunks; expensive React components memoized; long lists virtualized
- All animations use `transform`/`opacity`; image dimensions set; font-display configured — CLS < 0.1
