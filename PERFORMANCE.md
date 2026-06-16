# Performance Optimization Guide

Comprehensive guide for optimizing Scalar's build performance, runtime performance, and deployment efficiency.

## Table of Contents

1. [Build Performance](#build-performance)
2. [Runtime Performance](#runtime-performance)
3. [Bundle Optimization](#bundle-optimization)
4. [Deployment Optimization](#deployment-optimization)
5. [Monitoring & Metrics](#monitoring--metrics)
6. [Performance Checklist](#performance-checklist)

---

## Build Performance

### Current Status

**Build Time:** 2-5 minutes (Vercel optimized)  
**Cache Efficiency:** Turbo caching enabled  
**Parallelization:** ✅ Optimized  

### Optimization Strategies

#### 1. Turbo Caching

Turbo caches build outputs automatically. Verify it's working:

```bash
# Check cache hits/misses
turbo run build --summarize

# Output should show cache efficiency
# Example: "4 cached, 26 total" means 4 were cache hits
```

**Optimization Tips:**

```bash
# Skip cache and rebuild everything (debugging)
turbo run build --force --no-cache

# Use local only caching (faster for dev)
TURBO_CACHE_MODE=local pnpm build:packages

# Enable distributed caching for team
export TURBO_API="https://api.vercel.com"
export TURBO_TOKEN="your-token"
export TURBO_TEAM="your-team"
pnpm build:packages
```

#### 2. Incremental Builds

Only rebuild affected packages:

```bash
# Build only changed packages
pnpm build:packages --filter @scalar/core

# Build changed packages and dependents
turbo run build --only="packages" --affected
```

#### 3. Parallel Build Optimization

Configure in `turbo.json`:

```json
{
  "tasks": {
    "build": {
      "outputs": ["dist/**"],
      "cache": true,
      "concurrency": 100
    }
  }
}
```

**Current Setting:** `--concurrency=100%` allows maximum parallelization

#### 4. Dependency Optimization

```bash
# Find slow dependencies
turbo run build --summarize | sort -k2 -nr | head -20

# Optimize slow packages
# - Remove unnecessary devDependencies
# - Replace heavy packages with lighter alternatives
# - Use tree-shaking friendly packages
```

#### 5. TypeScript Compilation Optimization

```json
// tsconfig.json
{
  "compilerOptions": {
    "skipLibCheck": true,           // Skip type checking of declaration files
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "moduleResolution": "node",
    "target": "ES2020",             // Modern target = less transpilation
    "module": "ESNext"              // ESNext = better tree-shaking
  }
}
```

### Build Performance Baseline

Create a baseline for measuring improvements:

```bash
# Run build and measure time
time pnpm build:packages

# Output example:
# real    2m34s
# user    8m45s
# sys     0m12s
```

Track baseline in `PERFORMANCE_BASELINE.md`:
- Date: June 17, 2026
- Build Time: 2m 34s
- CI Build Time: ~3m 00s (including setup)
- Cache Efficiency: 40-60% hit rate

---

## Runtime Performance

### Core Web Vitals

Monitor in Vercel dashboard: https://vercel.com/dashboard → Analytics

**Targets:**
- **LCP (Largest Contentful Paint):** < 2.5s
- **INP (Interaction to Next Paint):** < 200ms
- **CLS (Cumulative Layout Shift):** < 0.1

### Performance Monitoring

#### 1. Web Vitals Integration

Automatically tracked by Vercel. View at:
- Dashboard Analytics: https://vercel.com/dashboard/project/prj_KeNpiNyFIW1BNdt0nKFMsGhvAgxh/analytics
- Real-time data: Updated continuously
- Historical data: Available for trend analysis

#### 2. Runtime Metrics

Monitor key metrics:

```typescript
// pages/api/health.ts
export default async function handler(req: Request) {
  const startTime = performance.now()
  
  // Do work
  const result = await getAnalytics()
  
  const duration = performance.now() - startTime
  
  return Response.json({
    status: 'healthy',
    responseTime: duration,
    timestamp: new Date(),
  })
}
```

#### 3. Error Rate Monitoring

```typescript
// Monitor error rates
async function trackError(error: Error, context: Record<string, any>) {
  const errorRate = (await getErrorCount()) / (await getTotalRequests())
  
  if (errorRate > 0.01) {  // > 1% error rate
    await alertOps({
      severity: 'warning',
      message: 'Error rate elevated',
      rate: errorRate,
    })
  }
}
```

---

## Bundle Optimization

### Current Bundle Analysis

Analyze bundle size:

```bash
# Build and analyze
pnpm build:packages

# Check individual package sizes
du -sh packages/*/dist/

# Example output:
# 245K    packages/api-reference/dist
# 1.2M    packages/api-client/dist
# 89K     packages/components/dist
```

### Optimization Techniques

#### 1. Tree-Shaking

Ensure packages support tree-shaking:

```json
// package.json
{
  "main": "./dist/index.cjs",
  "module": "./dist/index.mjs",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    }
  },
  "sideEffects": false  // Enable tree-shaking
}
```

#### 2. Code Splitting

For large components, use dynamic imports:

```typescript
// ✅ Dynamic import (lazy loaded)
const HeavyComponent = dynamic(() => import('./HeavyComponent'), {
  loading: () => <div>Loading...</div>,
})

export function Page() {
  return <HeavyComponent />
}
```

#### 3. Dependency Deduplication

```bash
# Check for duplicate dependencies
pnpm list --all | grep duplicates

# Remove duplicates
pnpm dedupe --lockfile
```

#### 4. Vendor Bundle Optimization

Configure in Vite:

```typescript
// vite.config.ts
import { defineConfig } from 'vite'

export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'vendor': [
            'vue',
            'react',
            'react-dom',
          ],
          'ui': [
            '@scalar/components',
            '@scalar/icons',
          ],
        },
      },
    },
  },
})
```

#### 5. Minification

Vite automatically minifies. Verify:

```bash
# Check minified output
ls -lh packages/api-reference/dist/*.js | tail -5

# Should see .min.js files or similar
```

---

## Deployment Optimization

### Vercel Deployment Configuration

Configured in `vercel.json`:

```json
{
  "buildCommand": "pnpm run build:packages && pnpm run build:integrations",
  "installCommand": "pnpm install --frozen-lockfile",
  "outputDirectory": "./"
}
```

### Optimization Strategies

#### 1. Caching Headers

Configure aggressive caching for static assets:

```json
{
  "headers": [
    {
      "source": "/(.*)\\.(?:js|css|png|jpg|jpeg|gif|svg|ico|woff|woff2)$",
      "headers": [
        {
          "key": "Cache-Control",
          "value": "public, max-age=31536000, immutable"
        }
      ]
    }
  ]
}
```

**Benefits:**
- Browser caches assets for 1 year
- CDN caches indefinitely (immutable hash)
- Zero requests for unchanged assets

#### 2. Compression

Vercel automatically compresses with Brotli:

```bash
# Verify compression in response headers
curl -I https://scalar.com/api/data

# Look for:
# content-encoding: br  (Brotli)
# content-encoding: gzip (fallback)
```

#### 3. Edge Caching

Use Vercel Edge Cache for frequently accessed data:

```typescript
// api/cached-data.ts
export const config = {
  cache: 'verified',
}

export default async function handler(req: Request) {
  const response = new Response(JSON.stringify({ /* data */ }))
  
  response.headers.set(
    'Cache-Control',
    'public, max-age=3600, s-maxage=86400'
  )
  
  return response
}
```

#### 4. Edge Functions

Convert performance-critical handlers to Edge Functions:

```typescript
// api/edge-handler.ts
export const config = {
  runtime: 'edge',
  regions: ['sfo1', 'iad1', 'dub1'],  // Deploy to multiple regions
}

export default async function handler(req: Request) {
  // Runs on Vercel Edge Network
  return new Response('Low-latency response')
}
```

---

## Monitoring & Metrics

### Setup Performance Monitoring

#### 1. Real User Monitoring (RUM)

Track actual user performance:

```typescript
// Automatically enabled by Vercel Web Analytics
// No configuration needed

// Optional: custom metrics
if ('PerformanceObserver' in window) {
  const observer = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      console.log(`${entry.name}: ${entry.duration}ms`)
    }
  })
  
  observer.observe({
    entryTypes: ['measure', 'navigation', 'resource'],
  })
}
```

#### 2. Synthetic Monitoring

Track from fixed locations:

```bash
# Create uptime check in Vercel
# Dashboard → Monitoring → Uptime checks

# Or use external services:
# - Pingdom
# - StatusCake
# - Synthetic Monitoring
```

#### 3. Error Rate Tracking

```typescript
// Monitor error rate
const errorCount = await getErrorCount()
const totalRequests = await getTotalRequests()
const errorRate = errorCount / totalRequests

console.log(`Error rate: ${(errorRate * 100).toFixed(2)}%`)

// Alert if threshold exceeded
if (errorRate > 0.01) {
  await sendAlert('High error rate detected')
}
```

#### 4. Performance Budget

Define acceptable performance limits:

```typescript
// performance-budget.json
{
  "budgets": [
    {
      "type": "bundle",
      "name": "main",
      "baseline": "1.2mb",
      "maxSize": "1.3mb"
    },
    {
      "type": "timings",
      "name": "loadTime",
      "baseline": "2.5s",
      "maxSize": "3s"
    }
  ]
}
```

### Dashboards & Reporting

#### Vercel Analytics Dashboard

Access at: https://vercel.com/dashboard/project/prj_KeNpiNyFIW1BNdt0nKFMsGhvAgxh/analytics

Monitor:
- **Web Vitals** - LCP, INP, CLS, FID, TTFB
- **Traffic** - Page views, unique visitors
- **Performance** - Response times by endpoint
- **Errors** - Error rate trends

---

## Performance Checklist

### Pre-Optimization

- [ ] Establish baseline metrics
  ```bash
  turbo run build --summarize
  ```
  - Record build times
  - Note cache efficiency

- [ ] Identify slow packages
  ```bash
  turbo run build --summarize | sort -k2 -nr | head -10
  ```

- [ ] Check bundle sizes
  ```bash
  du -sh packages/*/dist/
  ```

### Optimization Phase

#### Build Optimization
- [ ] Enable Turbo caching
- [ ] Configure incremental builds
- [ ] Update TypeScript target to ES2020
- [ ] Set `skipLibCheck: true` if appropriate
- [ ] Remove unnecessary polyfills

#### Bundle Optimization
- [ ] Verify tree-shaking enabled (sideEffects: false)
- [ ] Implement code splitting for large components
- [ ] Analyze and optimize vendor bundle
- [ ] Remove unused dependencies
- [ ] Replace heavy packages with lighter alternatives

#### Runtime Optimization
- [ ] Lazy load non-critical components
- [ ] Optimize images (use Vercel Image Optimization)
- [ ] Implement caching headers
- [ ] Enable compression (Vercel automatic)
- [ ] Consider Edge Functions for latency-critical paths

#### Deployment Optimization
- [ ] Configure cache headers in vercel.json
- [ ] Set up Web Vitals monitoring
- [ ] Create performance budget
- [ ] Establish error tracking
- [ ] Set up alerting for anomalies

### Post-Optimization Verification

- [ ] Build time improved by target %
- [ ] Bundle size reduced
- [ ] Web Vitals improved (LCP, INP, CLS)
- [ ] Error rate within acceptable range
- [ ] All tests passing
- [ ] No regressions in functionality

### Continuous Monitoring

- [ ] Weekly: Check Web Vitals dashboard
- [ ] Weekly: Review error rates
- [ ] Monthly: Analyze bundle size trends
- [ ] Monthly: Review performance budget compliance
- [ ] Quarterly: Full performance audit

---

## Performance Tuning Examples

### Example 1: Optimize Heavy Component

```typescript
// ❌ Before - loads entire library upfront
import HeavyChart from './HeavyChart'

export function Dashboard() {
  return (
    <div>
      {showChart && <HeavyChart data={data} />}
    </div>
  )
}
```

```typescript
// ✅ After - lazy load heavy component
import dynamic from 'next/dynamic'

const HeavyChart = dynamic(
  () => import('./HeavyChart'),
  { loading: () => <Skeleton /> }
)

export function Dashboard() {
  return (
    <div>
      {showChart && <HeavyChart data={data} />}
    </div>
  )
}
```

### Example 2: Cache API Response

```typescript
// ❌ Before - no caching
export async function getServerSideProps() {
  const data = await fetch('https://api.scalar.com/data')
  return { props: { data } }
}
```

```typescript
// ✅ After - with caching
export async function getStaticProps() {
  const data = await fetch('https://api.scalar.com/data')
  return {
    props: { data },
    revalidate: 3600  // Cache for 1 hour
  }
}
```

### Example 3: Image Optimization

```typescript
// ❌ Before
<img src="/image.jpg" alt="..." />

// ✅ After - Next.js Image optimization
import Image from 'next/image'

<Image
  src="/image.jpg"
  alt="..."
  width={400}
  height={300}
  priority={false}
  loading="lazy"
/>
```

---

## Performance Goals

### Build Performance Targets

| Metric | Current | Target | Timeline |
|--------|---------|--------|----------|
| Full Build | 2m 34s | 2m 00s | Q3 2026 |
| Incremental Build | 30s | 15s | Q3 2026 |
| Cache Hit Rate | 45% | 70% | Q3 2026 |

### Runtime Performance Targets

| Metric | Target | Actual |
|--------|--------|--------|
| LCP | < 2.5s | Monitor |
| INP | < 200ms | Monitor |
| CLS | < 0.1 | Monitor |
| TTFB | < 600ms | Monitor |

### Bundle Size Targets

| Package | Current | Target |
|---------|---------|--------|
| api-reference | ~245KB | 200KB |
| api-client | ~1.2MB | 1.0MB |
| components | ~89KB | 80KB |

---

## Tools & Resources

### Performance Testing Tools

- [WebPageTest](https://www.webpagetest.org/) - Detailed performance analysis
- [Lighthouse](https://developers.google.com/web/tools/lighthouse) - Automated auditing
- [Bundle Analyzer](https://esbuild.github.io/analyze/) - Bundle composition
- [Import Cost](https://marketplace.visualstudio.com/items?itemName=wix.vscode-import-cost) - Import size in VS Code

### Vercel Performance Tools

- [Vercel Analytics](https://vercel.com/docs/analytics) - Real user monitoring
- [Vercel Insights](https://vercel.com/docs/insights) - Web Vitals tracking
- [Vercel Logs](https://vercel.com/docs/logs) - Performance logs

### NPM Package Alternatives

When a package is too heavy, consider:

| Heavy Package | Lighter Alternative |
|---------------|-------------------|
| lodash | lodash-es (tree-shakable) |
| moment | date-fns or dayjs |
| axios | node-fetch or native fetch |
| validator | zod (with built-in validation) |

---

## Common Performance Issues & Solutions

### Issue: Slow Build Times

**Diagnosis:**
```bash
turbo run build --summarize | grep -E "⚠|✓"
```

**Solution:**
1. Identify slowest packages
2. Check for unnecessary dependencies
3. Enable incremental builds
4. Use build cache: `turbo build --cache-all`

### Issue: Large Bundle Size

**Diagnosis:**
```bash
npm run build && npm run analyze
```

**Solution:**
1. Remove unused dependencies: `pnpm lint:knip`
2. Dynamic import large components
3. Tree-shake unused exports: `sideEffects: false`
4. Replace heavy packages

### Issue: Slow Page Load

**Diagnosis:**
- Check Vercel Analytics for LCP/INP/CLS
- Use Chrome DevTools Performance tab
- Check Network tab for slow resources

**Solution:**
1. Optimize images
2. Lazy load non-critical components
3. Implement caching
4. Use CDN for static assets

---

**Last Updated:** June 17, 2026  
**Maintained By:** Scalar Performance Team  
**Status:** ✅ Actively Monitored
