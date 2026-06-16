# Scalar - Deployment Guide

Complete guide for deploying the Scalar monorepo to production on Vercel.

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Local Development Setup](#local-development-setup)
4. [Pre-Deployment Checklist](#pre-deployment-checklist)
5. [Vercel Deployment](#vercel-deployment)
6. [Environment Configuration](#environment-configuration)
7. [Monitoring & Performance](#monitoring--performance)
8. [Troubleshooting](#troubleshooting)
9. [Rollback Procedures](#rollback-procedures)

---

## Overview

**Scalar** is deployed as a monorepo on Vercel with the following characteristics:

- **Build System:** Turbo + pnpm
- **Framework:** Multi-framework (Vue, React, Next.js, Nuxt, Astro)
- **Deployment Target:** Vercel (prj_KeNpiNyFIW1BNdt0nKFMsGhvAgxh)
- **Build Time:** ~2-5 minutes (cached builds faster)
- **Node.js:** v24 LTS
- **Package Manager:** pnpm 10.16.1

---

## Prerequisites

### 1. System Requirements

```bash
# Verify Node.js version (v24 or later)
node --version
# Output: v24.14.1+

# Verify pnpm version
pnpm --version
# Output: 10.16.1+

# Verify Git
git --version
```

### 2. Vercel Account & CLI

```bash
# Install Vercel CLI
npm install -g vercel

# Login to Vercel
vercel login

# Link to project
vercel link
# Select: xeondesk/scalar
# Project ID: prj_KeNpiNyFIW1BNdt0nKFMsGhvAgxh
```

### 3. GitHub Access

- ✅ Repository access: xeondesk/scalar
- ✅ Branch permissions on `main`
- ✅ GitHub Actions enabled

---

## Local Development Setup

### 1. Clone Repository

```bash
git clone https://github.com/xeondesk/scalar.git
cd scalar
```

### 2. Install Dependencies

```bash
# Install using pnpm
pnpm install --frozen-lockfile

# Verify installation
pnpm list --depth=0 | head -20
```

### 3. Environment Setup

```bash
# Copy environment template
cp .env.example .env.development.local

# Edit environment file (add your values)
nano .env.development.local

# Required variables:
# - AI_GATEWAY_API_KEY (optional for local dev)
# - NODE_ENV=development
```

### 4. Verify Setup

```bash
# Type check
pnpm types:check

# Linting
pnpm lint:check

# Build packages
pnpm build:packages
```

---

## Pre-Deployment Checklist

### Code Quality

- [ ] All tests passing locally
  ```bash
  pnpm test
  ```

- [ ] No linting errors
  ```bash
  pnpm lint:check
  ```

- [ ] Type checking passes
  ```bash
  pnpm types:check
  ```

- [ ] No unused dependencies
  ```bash
  pnpm lint:knip
  ```

### Build Verification

- [ ] Build succeeds locally
  ```bash
  pnpm build:packages
  pnpm build:integrations
  ```

- [ ] No console errors/warnings during build

### Git & Commit

- [ ] All changes committed
  ```bash
  git status
  # Should show: working tree clean
  ```

- [ ] Branch is up to date with main
  ```bash
  git pull origin main
  git log --oneline | head -5
  ```

- [ ] No merge conflicts
  ```bash
  git diff --name-only origin/main
  ```

### Documentation

- [ ] CHANGELOG.md updated with changes
- [ ] README.md reflects current state
- [ ] Migration plan documented
- [ ] Any breaking changes documented

### Environment Variables

- [ ] All required vars set in `.env.development.local`
  ```bash
  cat .env.development.local | grep -v "^#" | grep -v "^$"
  ```

- [ ] No secrets in code
  ```bash
  git diff origin/main | grep -E "password|secret|key|token"
  # Should return: nothing
  ```

- [ ] Variables documented in `.env.example`

---

## Vercel Deployment

### Option 1: Automatic (Recommended)

Vercel automatically deploys when you push to `main`:

```bash
# Ensure you're on main branch
git checkout main

# Pull latest changes
git pull origin main

# Merge your feature branch (if applicable)
git merge feature/your-feature

# Push to main
git push origin main

# Vercel will automatically:
# 1. Detect the push
# 2. Trigger CI/CD pipeline
# 3. Run tests and linting
# 4. Build the monorepo
# 5. Deploy to production
```

**Deployment Status:** Monitor at https://vercel.com/dashboard/project/prj_KeNpiNyFIW1BNdt0nKFMsGhvAgxh

### Option 2: Manual via CLI

```bash
# Preview deployment (optional)
vercel --prod --prebuilt

# Or with environment override
VERCEL_FORCE_NO_BUILD_CACHE=1 vercel deploy --prod
```

### Option 3: GitHub Actions

The repository has automated CI/CD workflows:

- **Deploy Preview:** On every PR (`.github/workflows/deploy-previews.yml`)
- **Deploy Main:** On push to main (`.github/workflows/main.yml`)
- **Release:** On version tag (`.github/workflows/release.yml`)

**Status:** https://github.com/xeondesk/scalar/actions

---

## Environment Configuration

### Vercel Project Settings

Navigate to: https://vercel.com/dashboard/project/prj_KeNpiNyFIW1BNdt0nKFMsGhvAgxh/settings

#### 1. Environment Variables

Add these variables in **Settings → Environment Variables**:

| Variable | Value | Environment | Source |
|----------|-------|-------------|--------|
| `AI_GATEWAY_API_KEY` | Your API key | Production, Preview | Vercel Console |
| `VERCEL_WEB_ANALYTICS_ID` | Your ID | Production, Preview | Vercel Analytics |
| `NODE_ENV` | `production` | Production | Auto-configured |

#### 2. Build & Development Settings

**Build Command:**
```bash
pnpm run build:packages && pnpm run build:integrations
```

**Install Command:**
```bash
pnpm install --frozen-lockfile
```

**Output Directory:**
```
./
```

#### 3. Git Integration

- ✅ Connected to: xeondesk/scalar
- ✅ Branch: main (production)
- ✅ Preview deployments: All PR branches

#### 4. Domains

Configure custom domains in **Settings → Domains**:

- Primary domain: scalar.com (if applicable)
- Preview deployments: auto-generated

#### 5. Security Headers

These are configured in `vercel.json` and automatically applied:

```json
{
  "X-Content-Type-Options": "nosniff",
  "X-Frame-Options": "SAMEORIGIN",
  "X-XSS-Protection": "1; mode=block",
  "Referrer-Policy": "strict-origin-when-cross-origin",
  "Permissions-Policy": "geolocation=(), microphone=(), camera=()"
}
```

---

## Monitoring & Performance

### 1. Real-time Dashboard

Access at: https://vercel.com/dashboard/project/prj_KeNpiNyFIW1BNdt0nKFMsGhvAgxh

Monitor:
- ✅ Latest deployments
- ✅ Build times and status
- ✅ Environment variables
- ✅ Domain health

### 2. Analytics

Enable Web Vitals tracking:

1. Go to **Settings → Analytics**
2. Enable **Web Vitals**
3. View real-time metrics at: https://vercel.com/dashboard/project/prj_KeNpiNyFIW1BNdt0nKFMsGhvAgxh/analytics

Key Metrics:
- **Largest Contentful Paint (LCP)** - Target: < 2.5s
- **Interaction to Next Paint (INP)** - Target: < 200ms
- **Cumulative Layout Shift (CLS)** - Target: < 0.1

### 3. Build Performance

```bash
# View build analysis
vercel projects ls
vercel env ls

# Check cache efficiency
turbo run build --summarize
```

### 4. Function Invocations

Monitor API route performance in Vercel dashboard:
- Invocation count
- Response times
- Error rates
- Log insights

---

## Troubleshooting

### Build Failures

#### Issue: "pnpm: command not found"

**Solution:**
```bash
# Ensure Node.js version is correct
node --version  # Should be v24+

# Vercel automatically installs pnpm 10.16.1
# If manually deploying, install it:
npm install -g pnpm@10.16.1
```

#### Issue: "Cannot find module @scalar/*"

**Solution:**
```bash
# Clear build cache and reinstall
pnpm clean
pnpm install --frozen-lockfile
pnpm build:packages
```

#### Issue: "TypeScript compilation error"

**Solution:**
```bash
# Type check locally first
pnpm types:check

# Fix any TypeScript errors
# Then rebuild
pnpm clean && pnpm build
```

### Performance Issues

#### Slow Build Times (> 10 minutes)

**Diagnosis:**
```bash
# Check build time distribution
turbo run build --summarize

# Check cache efficiency
ls .turbo/cache | wc -l  # Should be > 100 items
```

**Solution:**
1. Clear Vercel build cache: **Settings → Git → Clear Build Cache**
2. Rebuild with `VERCEL_FORCE_NO_BUILD_CACHE=1 vercel deploy --prod`
3. Check for expensive operations in `turbo.json`

#### High JavaScript Bundle Size

**Diagnosis:**
```bash
# Analyze bundle size
pnpm build:packages
ls -lh packages/*/dist/*.js | sort -k5 -h
```

**Solution:**
1. Review tree-shaking in vite.config.ts
2. Remove unused dependencies: `pnpm lint:knip`
3. Check for duplicate dependencies in lock file

### Deployment Issues

#### Issue: "Function exceeds max duration (60s)"

**Solution:**
Increase timeout in `vercel.json`:
```json
{
  "functions": {
    "api/**": {
      "maxDuration": 120
    }
  }
}
```

#### Issue: "Memory limit exceeded"

**Solution:**
1. Optimize large operations into smaller functions
2. Use Vercel's serverless cron for background tasks
3. Consider Edge Runtime for lightweight handlers

#### Issue: "Domain DNS issues"

**Solution:**
1. Verify DNS settings in domain registrar
2. Check Vercel domain configuration
3. Use: `nslookup your-domain.com`

---

## Rollback Procedures

### Scenario 1: Rollback to Previous Version

#### Automatic via Vercel

```bash
# In Vercel Dashboard
1. Go to Deployments
2. Find stable deployment (green checkmark)
3. Click "Promote to Production"
4. Confirm action
```

#### Manual via Git + CLI

```bash
# Find the commit to rollback to
git log --oneline | head -10

# Reset to previous commit
git reset --hard <commit-hash>

# Force push to main (use with caution!)
git push origin main --force

# Vercel will redeploy the previous version
```

### Scenario 2: Emergency Hotfix

```bash
# Create hotfix branch from main
git checkout main
git pull origin main
git checkout -b hotfix/critical-issue

# Make minimal changes
# ... fix the issue ...

# Commit and push
git add .
git commit -m "fix: critical issue"
git push origin hotfix/critical-issue

# Create Pull Request and merge
# GitHub Actions will trigger automatic deployment
```

### Scenario 3: Revert Recent Commit

```bash
# Identify the commit to revert
git log --oneline -5

# Create revert commit (creates new commit)
git revert <commit-hash>

# Push the revert
git push origin main

# Vercel redeploys immediately
```

### Verification After Rollback

```bash
# Check deployment status
vercel deployments

# Verify application functionality
curl https://scalar.com/api/health

# Check error logs
vercel logs --tail
```

---

## Performance Optimization Tips

### 1. Cache Optimization

Configure in `vercel.json` for static assets:

```json
{
  "headers": [
    {
      "source": "/(.*)\\.(?:js|css|png|jpg|jpeg|gif|svg)$",
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

### 2. Edge Functions

For low-latency routes, convert to Edge Functions:

```typescript
// api/middleware.ts
export const config = {
  runtime: 'edge',
}

export default async function handler(req: Request) {
  // Low-latency execution at edge
  return new Response('Hello from Edge!')
}
```

### 3. Incremental Static Regeneration (ISR)

If using Next.js pages:

```typescript
export const revalidate = 3600 // revalidate every hour
```

### 4. Image Optimization

Let Vercel handle image optimization:

```html
<!-- Use Next.js Image component -->
<Image src="/image.jpg" alt="..." width={400} height={300} />
```

---

## Security Best Practices

### 1. Environment Variables

✅ Use Vercel's **Settings → Environment Variables**  
❌ Never commit `.env.development.local` with real secrets  
❌ Never use hardcoded API keys in code

### 2. CORS Configuration

Verify CORS headers are correct:

```bash
curl -i -X OPTIONS https://scalar.com/api/endpoint
# Should show: Access-Control-Allow-Origin
```

### 3. Secret Rotation

For sensitive credentials:

1. Generate new secret in source system
2. Add new value in Vercel Environment Variables
3. Update code to use new variable
4. Revoke old secret in source system

### 4. Audit Logging

Review all deployments at: https://vercel.com/dashboard/project/prj_KeNpiNyFIW1BNdt0nKFMsGhvAgxh/logs

---

## Post-Deployment Tasks

### 1. Smoke Testing

```bash
# Test critical endpoints
curl https://scalar.com/ --head
# Should return: 200 OK

curl https://scalar.com/api/health
# Should return: healthy response
```

### 2. Performance Verification

1. Visit https://vercel.com/dashboard → Analytics
2. Check Web Vitals (LCP, INP, CLS)
3. Compare to baseline

### 3. Error Monitoring

1. Check Vercel Logs for errors: `vercel logs --tail`
2. Verify no spike in 5xx errors
3. Review error tracking service (if enabled)

### 4. Notify Team

```bash
# Example notification
echo "✅ Deployed: <version> to production"
echo "📊 Analytics: https://vercel.com/dashboard/analytics"
echo "📝 Changes: https://github.com/xeondesk/scalar/compare/..."
```

---

## Support & Resources

### Vercel Documentation

- [Vercel Deployment Guide](https://vercel.com/docs/deployments/overview)
- [Environment Variables](https://vercel.com/docs/projects/environment-variables)
- [Security & Headers](https://vercel.com/docs/concepts/edge-network/headers)

### Scalar Documentation

- [Main README](./README.md)
- [Migration Plan](./MIGRATION_PLAN.md)
- [Contributing Guide](./CONTRIBUTING.md)

### Getting Help

- GitHub Issues: https://github.com/xeondesk/scalar/issues
- Scalar Discord: https://discord.gg/scalar
- Vercel Support: https://vercel.com/support

---

## Deployment Checklist Template

Copy this for each deployment:

```markdown
## Deployment: [Date] - [Release Version]

### Pre-Deployment
- [ ] Tests passing: `pnpm test`
- [ ] Linting passing: `pnpm lint:check`
- [ ] Build successful: `pnpm build`
- [ ] Type check passing: `pnpm types:check`
- [ ] All commits pushed: `git push origin main`

### Deployment
- [ ] Vercel deployment triggered
- [ ] Build completed successfully
- [ ] Tests ran in CI/CD
- [ ] Preview deployment verified

### Post-Deployment
- [ ] Production site loads
- [ ] Critical paths working
- [ ] No console errors
- [ ] Analytics updated
- [ ] Team notified

### Monitoring (24 hours)
- [ ] Error rate normal
- [ ] Performance metrics normal
- [ ] User reports reviewed
```

---

**Last Updated:** June 17, 2026  
**Maintained By:** Scalar Team  
**Status:** ✅ Production Ready
