# Security Hardening Guide

Production security best practices and configurations for the Scalar monorepo.

## Table of Contents

1. [Overview](#overview)
2. [Security Headers](#security-headers)
3. [Authentication & Authorization](#authentication--authorization)
4. [Input Validation](#input-validation)
5. [Dependency Management](#dependency-management)
6. [Secrets Management](#secrets-management)
7. [Error Handling](#error-handling)
8. [Logging & Monitoring](#logging--monitoring)
9. [CORS & Origin Policies](#cors--origin-policies)
10. [Security Checklist](#security-checklist)

---

## Overview

### Current Security Posture

✅ **Strengths:**
- Automated dependency updates via Renovate
- TypeScript strict mode (type safety)
- ESLint with security rules
- Biome linting
- Git hooks for code quality
- Knip for unused dependency detection
- OIDC for Vercel authentication

⚠️ **Areas for Enhancement:**
- Explicit CORS policies in code
- Request validation schemas
- Rate limiting configuration
- Secret rotation procedures
- Audit logging

---

## Security Headers

### Configured in vercel.json

All responses include security headers:

```json
{
  "X-Content-Type-Options": "nosniff",
  "X-Frame-Options": "SAMEORIGIN",
  "X-XSS-Protection": "1; mode=block",
  "Referrer-Policy": "strict-origin-when-cross-origin",
  "Permissions-Policy": "geolocation=(), microphone=(), camera=()"
}
```

### Content Security Policy (CSP)

Add to `vercel.json` for stricter CSP:

```json
{
  "Content-Security-Policy": "default-src 'self'; script-src 'self' 'unsafe-inline' https://cdn.jsdelivr.net; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' data: https://fonts.googleapis.com; connect-src 'self' https://api.scalar.com"
}
```

### HSTS (HTTP Strict Transport Security)

```json
{
  "Strict-Transport-Security": "max-age=31536000; includeSubDomains; preload"
}
```

---

## Authentication & Authorization

### 1. API Authentication

All API routes should validate authentication:

```typescript
// api/protected-route.ts
import { verifyAuth } from '@scalar/auth'

export default async function handler(req: Request) {
  const token = req.headers.get('authorization')?.replace('Bearer ', '')
  
  if (!token) {
    return new Response('Unauthorized', { status: 401 })
  }
  
  try {
    const user = await verifyAuth(token)
    // Process request
  } catch {
    return new Response('Forbidden', { status: 403 })
  }
}
```

### 2. Session Management

```typescript
// Secure session configuration
const sessionConfig = {
  secure: true,           // HTTPS only
  httpOnly: true,         // JavaScript inaccessible
  sameSite: 'Strict',     // CSRF protection
  maxAge: 3600,           // 1 hour expiration
}
```

### 3. Role-Based Access Control (RBAC)

```typescript
export async function requireRole(role: string) {
  return async (req: Request) => {
    const user = await getUser(req)
    
    if (!user || !user.roles.includes(role)) {
      throw new Error(`Requires role: ${role}`)
    }
    
    return user
  }
}
```

---

## Input Validation

### 1. Request Validation

Use Zod for runtime validation:

```typescript
import { z } from 'zod'

const createPostSchema = z.object({
  title: z.string().min(1).max(200),
  content: z.string().min(10),
  tags: z.array(z.string()).max(10),
})

export async function POST(req: Request) {
  const body = await req.json()
  const data = createPostSchema.parse(body)
  // Process validated data
}
```

### 2. SQL Injection Prevention

Always use parameterized queries:

```typescript
// ❌ WRONG - vulnerable
const query = `SELECT * FROM users WHERE id = ${userId}`

// ✅ CORRECT - safe
const query = 'SELECT * FROM users WHERE id = $1'
const result = await db.query(query, [userId])
```

### 3. XSS Prevention

```typescript
// ✅ Use framework's built-in escaping
import { escapeHtml } from '@scalar/helpers'

export function UserCard({ user }: { user: User }) {
  return <div>{escapeHtml(user.bio)}</div>
}
```

### 4. URL Validation

```typescript
function isValidUrl(url: string): boolean {
  try {
    new URL(url)
    return true
  } catch {
    return false
  }
}

// Only redirect to validated URLs
if (isValidUrl(redirectUrl)) {
  return redirect(redirectUrl)
}
```

---

## Dependency Management

### 1. Security Updates

Renovate automatically creates PRs for security updates:

```yaml
# renovate.json
{
  "extends": [
    "config:base",
    ":semanticCommits"
  ],
  "vulnerabilityAlerts": {
    "enabled": true,
    "automerge": true
  },
  "ignoreDeps": []
}
```

### 2. Verify Dependency Integrity

```bash
# Check for vulnerabilities
pnpm audit

# Check for unused dependencies
pnpm lint:knip

# Verify lock file integrity
pnpm install --frozen-lockfile --audit
```

### 3. Dependency Pinning

Critical security updates are pinned:

```yaml
# pnpm-workspace.yaml
overrides:
  '@types/node': 'catalog:*'
  'lodash': '4.18.1'
  'yaml': '2.8.3'
  'picomatch': '4.0.4'
```

### 4. Supply Chain Security

```bash
# Verify package signatures (if supported)
npm info @scalar/core | grep dist.integrity

# Check for malicious packages
pnpm list | grep -E "(npm-audit|snyk)"
```

---

## Secrets Management

### 1. Environment Variables

✅ **DO:**
- Store in Vercel Environment Variables
- Use different values per environment
- Rotate secrets regularly
- Document all required variables in `.env.example`

```bash
# ✅ CORRECT
# .env.example (no values)
AI_GATEWAY_API_KEY=
DATABASE_URL=

# Vercel Settings → Environment Variables
# Add actual values there
```

❌ **DON'T:**
- Commit real values to `.env.example`
- Log environment variables
- Expose secrets in error messages
- Share secrets in communication channels

### 2. Secret Rotation

Monthly rotation for production secrets:

```bash
# Step 1: Generate new secret in source system
# Step 2: Add to Vercel Environment Variables
# Step 3: Update code if needed
# Step 4: Deploy
# Step 5: Wait for all instances to pick up new value
# Step 6: Revoke old secret
# Step 7: Monitor for errors
```

### 3. Access Control

Vercel OIDC for secure CI/CD:

```typescript
// Automatically available in GitHub Actions
process.env.VERCEL_OIDC_TOKEN

// Use for authentication
const response = await fetch('https://api.vercel.com/...', {
  headers: {
    'Authorization': `Bearer ${process.env.VERCEL_OIDC_TOKEN}`
  }
})
```

---

## Error Handling

### 1. Error Responses

Never expose sensitive information:

```typescript
// ❌ WRONG - exposes internal details
export async function handler(req: Request) {
  try {
    // ...
  } catch (error) {
    return Response.json({
      error: error.message,  // Might expose database structure
      stack: error.stack     // Exposes code paths
    }, { status: 500 })
  }
}

// ✅ CORRECT - generic error to user
export async function handler(req: Request) {
  try {
    // ...
  } catch (error) {
    console.error('Unexpected error:', error)  // Log internally
    return Response.json({
      error: 'Internal server error'  // Generic message
    }, { status: 500 })
  }
}
```

### 2. Error Logging

```typescript
import { captureException } from '@scalar/monitoring'

export async function handler(req: Request) {
  try {
    // ...
  } catch (error) {
    // Log full error internally for debugging
    captureException(error, {
      request: req,
      context: { userId: user.id }
    })
    
    // Return generic error to user
    return Response.json({
      error: 'Something went wrong'
    }, { status: 500 })
  }
}
```

### 3. Rate Limiting

Implement rate limiting for APIs:

```typescript
import { Ratelimit } from '@upstash/ratelimit'

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, '1 h'),
})

export async function handler(req: Request) {
  const ip = req.headers.get('x-forwarded-for')
  const { limit, remaining } = await ratelimit.limit(ip)
  
  if (!limit) {
    return new Response('Too many requests', { status: 429 })
  }
  
  // Process request
}
```

---

## Logging & Monitoring

### 1. Audit Logging

Log critical actions:

```typescript
async function logAuditEvent(event: {
  action: string
  userId: string
  resource: string
  timestamp: Date
  ipAddress: string
  result: 'success' | 'failure'
}) {
  await db.auditLog.create({
    ...event,
    // Don't log sensitive data like passwords
  })
}

// Usage
await logAuditEvent({
  action: 'user_login',
  userId: user.id,
  resource: 'auth',
  timestamp: new Date(),
  ipAddress: req.ip,
  result: 'success'
})
```

### 2. Security Monitoring

```typescript
// Flag suspicious activity
if (failedLogins > 5) {
  await notifySecurityTeam({
    alert: 'Brute force attempt detected',
    ip: ipAddress,
    userId: userId,
  })
}
```

### 3. Web Vitals for Security

Monitor Core Web Vitals to detect performance attacks:

```typescript
// Automatically collected by Vercel
// Check dashboard at: vercel.com/dashboard → Analytics
// Anomalies may indicate security issues
```

---

## CORS & Origin Policies

### 1. CORS Configuration

```typescript
// api/cors-handler.ts
export function setCorsHeaders(
  response: Response,
  origin: string
) {
  const allowedOrigins = [
    'https://scalar.com',
    'https://app.scalar.com',
    'https://api.scalar.com',
  ]
  
  if (allowedOrigins.includes(origin)) {
    response.headers.set('Access-Control-Allow-Origin', origin)
    response.headers.set(
      'Access-Control-Allow-Methods',
      'GET, POST, PUT, DELETE, OPTIONS'
    )
    response.headers.set(
      'Access-Control-Allow-Headers',
      'Content-Type, Authorization'
    )
    response.headers.set(
      'Access-Control-Allow-Credentials',
      'true'
    )
    response.headers.set('Access-Control-Max-Age', '86400')
  }
  
  return response
}
```

### 2. CSRF Protection

```typescript
import { csrf } from '@scalar/csrf'

export async function POST(req: Request) {
  const token = req.headers.get('x-csrf-token')
  
  if (!csrf.verify(token, req)) {
    return new Response('CSRF token invalid', { status: 403 })
  }
  
  // Process POST request
}
```

### 3. Origin Validation

```typescript
export function isOriginAllowed(origin: string): boolean {
  const allowed = process.env.ALLOWED_ORIGINS?.split(',') || []
  return allowed.includes(origin)
}

export async function handler(req: Request) {
  const origin = req.headers.get('origin')
  
  if (!isOriginAllowed(origin)) {
    return new Response('Origin not allowed', { status: 403 })
  }
  
  // Process request
}
```

---

## Security Checklist

### Pre-Deployment Security Review

Use this checklist before every production deployment:

#### Code Security
- [ ] No hardcoded secrets or API keys
- [ ] No sensitive data in logs
- [ ] Input validation on all endpoints
- [ ] Output encoding to prevent XSS
- [ ] SQL injection prevention (parameterized queries)
- [ ] CSRF tokens on state-changing requests

#### Dependency Security
- [ ] `pnpm audit` shows no vulnerabilities
- [ ] All packages are up-to-date
- [ ] Security patches applied
- [ ] No abandoned dependencies
- [ ] `pnpm lint:knip` shows no unused imports

#### Configuration Security
- [ ] Environment variables set in Vercel
- [ ] `.env.example` has no real values
- [ ] Security headers configured in `vercel.json`
- [ ] CORS policies configured
- [ ] Rate limiting configured

#### Authentication & Authorization
- [ ] User authentication required for protected routes
- [ ] Role-based access control implemented
- [ ] Session timeouts configured
- [ ] Secure cookie settings (httpOnly, secure, sameSite)
- [ ] Password requirements enforced

#### Error Handling
- [ ] No sensitive info in error messages
- [ ] All errors logged internally
- [ ] Proper HTTP status codes
- [ ] User-friendly error messages
- [ ] Error tracking service active

#### Monitoring
- [ ] Audit logging enabled
- [ ] Security alerts configured
- [ ] Error monitoring active
- [ ] Performance baseline established
- [ ] Log retention policy set

#### Deployment
- [ ] Tests passing locally and in CI/CD
- [ ] Type checking passes
- [ ] Linting passes
- [ ] Build successful
- [ ] Preview deployment tested
- [ ] Performance baseline acceptable

### Post-Deployment Security Verification

After deploying to production:

- [ ] Application loads without errors
- [ ] Security headers present: `curl -I https://scalar.com`
- [ ] No console errors in browser developer tools
- [ ] Authentication working correctly
- [ ] Protected routes require authentication
- [ ] Audit logs recording events
- [ ] Error monitoring active
- [ ] Performance metrics acceptable
- [ ] No security alerts triggered

---

## Common Vulnerabilities & Mitigations

### SQL Injection

```typescript
// ❌ Vulnerable
const query = `SELECT * FROM users WHERE email = '${email}'`

// ✅ Safe
const query = 'SELECT * FROM users WHERE email = $1'
await db.query(query, [email])
```

### Cross-Site Scripting (XSS)

```typescript
// ❌ Vulnerable
<div>{userInput}</div>

// ✅ Safe
<div>{escapeHtml(userInput)}</div>
```

### Cross-Site Request Forgery (CSRF)

```typescript
// ✅ Implement CSRF tokens
export async function POST(req: Request) {
  const token = req.headers.get('x-csrf-token')
  if (!csrf.verify(token, req.origin)) {
    return new Response('Unauthorized', { status: 403 })
  }
}
```

### Insecure Deserialization

```typescript
// ❌ Vulnerable
const data = JSON.parse(userInput)  // Without validation

// ✅ Safe
const schema = z.object({ /* ... */ })
const data = schema.parse(JSON.parse(userInput))
```

### Sensitive Data Exposure

```typescript
// ❌ Wrong - logging sensitive data
console.log('User:', { ...user, password })

// ✅ Correct - redacting sensitive fields
const { password, ...safeUser } = user
console.log('User:', safeUser)
```

---

## Security Resources

### External Security Tools

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Node.js Security Best Practices](https://nodejs.org/en/docs/guides/security/)
- [npm Audit](https://docs.npmjs.com/cli/v10/commands/npm-audit)
- [Snyk Security Scanning](https://snyk.io/)

### Vercel Security Features

- [Vercel Security Overview](https://vercel.com/security)
- [Vercel OIDC](https://vercel.com/docs/deployments/secure-authentication)
- [Vercel Environment Variables](https://vercel.com/docs/projects/environment-variables)

### Internal Resources

- [CONTRIBUTING.md](./CONTRIBUTING.md) - Code standards
- [DEPLOYMENT.md](./DEPLOYMENT.md) - Deployment guide
- [MIGRATION_PLAN.md](./MIGRATION_PLAN.md) - Architecture details

---

## Incident Response

### If You Suspect a Security Issue

1. **Stop and Assess**
   - Do not ignore potential security issues
   - Assess severity and impact

2. **Notify Immediately**
   - Contact security team
   - Create private security issue (GitHub Security tab)
   - Do NOT discuss publicly

3. **Remediate**
   - Create fix in private branch
   - Test thoroughly
   - Request security review

4. **Deploy Fix**
   - Deploy as soon as approved
   - Monitor closely for issues
   - Document incident

5. **Post-Incident**
   - Update security documentation
   - Add test coverage for issue
   - Share learnings with team

### Security Contact

For security vulnerabilities:
- Email: security@scalar.com (or your organization's security email)
- Do NOT create public GitHub issues for vulnerabilities

---

## Regular Security Maintenance

### Weekly
- Monitor security alerts from Renovate
- Review error logs for anomalies
- Check Web Vitals dashboard

### Monthly
- Run full security audit: `pnpm audit`
- Review dependency updates
- Rotate secrets (if applicable)
- Update security documentation

### Quarterly
- Security code review
- Penetration testing (if applicable)
- Update security checklist
- Team security training

---

**Last Updated:** June 17, 2026  
**Maintained By:** Scalar Security Team  
**Status:** ✅ Active Monitoring
