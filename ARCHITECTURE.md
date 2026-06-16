# Architecture & Design

Complete architectural documentation for the Scalar monorepo.

## Table of Contents

1. [Overview](#overview)
2. [Monorepo Structure](#monorepo-structure)
3. [Package Organization](#package-organization)
4. [Build Pipeline](#build-pipeline)
5. [Deployment Architecture](#deployment-architecture)
6. [Technology Stack](#technology-stack)
7. [Design Patterns](#design-patterns)

---

## Overview

**Scalar** is a production-ready monorepo containing:

1. **Core Libraries** (packages/) - Reusable npm packages
2. **Framework Integrations** (integrations/) - 17+ framework support
3. **Examples** (examples/) - Reference implementations
4. **Standalone Projects** (projects/) - Independent applications
5. **Build Tools** (tooling/) - Internal automation

### Architecture Principles

- ✅ **Modular**: Each package has single responsibility
- ✅ **Scalable**: Supports 30+ packages and growing
- ✅ **Type-Safe**: TypeScript strict mode throughout
- ✅ **Testable**: Comprehensive test coverage
- ✅ **Deployable**: Optimized for Vercel
- ✅ **Maintainable**: Clear conventions and documentation

---

## Monorepo Structure

### Top-Level Organization

```
scalar/
├── packages/                  # Published npm libraries (30+)
├── integrations/              # Framework integrations (17+)
├── examples/                  # Reference implementations (10+)
├── projects/                  # Standalone applications
├── documentation/             # Docs site
├── tooling/                   # Build & internal tools
├── .github/                   # GitHub Actions workflows
│   ├── workflows/             # 21 CI/CD workflows
│   └── ...
├── .vercel/                   # Vercel configuration
├── biome.json                 # Code formatting & linting
├── eslint.config.mjs          # ESLint configuration
├── turbo.json                 # Turbo monorepo config
├── tsconfig.json              # TypeScript configuration
├── pnpm-workspace.yaml        # pnpm workspace definition
├── package.json               # Root dependencies
├── vercel.json                # Deployment config ✨ NEW
├── .env.example               # Environment template ✨ NEW
├── MIGRATION_PLAN.md          # Architecture migration ✨ NEW
├── DEPLOYMENT.md              # Deployment guide ✨ NEW
├── PRODUCTION_SECURITY.md     # Security hardening ✨ NEW
├── PERFORMANCE.md             # Performance optimization ✨ NEW
├── ARCHITECTURE.md            # This file ✨ NEW
└── README.md                  # Main documentation
```

### Workspace Categories

#### 1. Packages (/packages)

Core libraries published to npm registry.

```
packages/
├── api-reference/             # Interactive API reference UI
├── api-client/                # Standalone API client
├── api-client-react/          # React wrapper for API client
├── api-reference-react/       # React wrapper for reference
├── components/                # Reusable UI components
├── core/                      # Core utilities
├── openapi-parser/            # OpenAPI parsing
├── openapi-upgrader/          # OpenAPI version upgrading
├── mock-server/               # Mock API server
├── code-highlight/            # Code syntax highlighting
├── helpers/                   # General utilities
├── icons/                     # SVG icon library
└── [20+ more]
```

**Characteristics:**
- Individually versioned and published
- Proper package.json with exports
- TypeScript source code
- Unit tests (Vitest)
- ESM and CJS builds

#### 2. Integrations (/integrations)

Framework-specific implementations.

```
integrations/
├── next.js/                   # Next.js integration
├── nuxt/                      # Nuxt integration
├── astro/                     # Astro integration
├── react/                     # React integration
├── vue/                       # Vue integration
├── fastify/                   # Fastify integration
├── express/                   # Express integration
├── nestjs/                    # NestJS integration
├── hono/                      # Hono integration
├── sveltekit/                 # SvelteKit integration
├── django/                    # Django integration
├── fastapi/                   # FastAPI integration
└── [5+ more]
```

**Characteristics:**
- Framework-specific adapters
- Reuse core libraries
- Example setups
- Framework's testing patterns
- Framework's build tools

#### 3. Examples (/examples)

Reference implementations showing best practices.

```
examples/
├── nextjs-api-reference/      # Next.js example
├── nuxt/                      # Nuxt example
├── react/                     # React example
├── web/                       # Vanilla JavaScript example
├── ssg/                       # Static site generation
└── [5+ more]
```

#### 4. Projects (/projects)

Standalone applications.

```
projects/
├── [Application A]/
├── [Application B]/
└── [Application C]/
```

#### 5. Tooling (/tooling)

Internal build scripts and automation.

```
tooling/
├── scripts/                   # Build and release scripts
├── changelog-generator/       # Changelog automation
└── [Internal tools]
```

---

## Package Organization

### Typical Package Structure

```
packages/core/
├── src/
│   ├── index.ts               # Public API
│   ├── types.ts               # Type definitions
│   ├── utils.ts               # Utilities
│   ├── composables.ts         # Vue composables
│   └── components/
│       ├── Button.vue         # Vue component
│       └── ...
├── tests/
│   ├── unit/
│   ├── integration/
│   └── ...
├── dist/                      # Build output (generated)
├── package.json               # Package metadata
├── tsconfig.json              # TypeScript config
├── vitest.config.ts           # Test config
├── README.md                  # Package documentation
├── CHANGELOG.md               # Version history
└── LICENSE                    # Licensing
```

### Package.json Structure

```json
{
  "name": "@scalar/core",
  "version": "1.0.0",
  "description": "Core utilities for Scalar",
  "type": "module",
  "main": "./dist/index.cjs",
  "module": "./dist/index.mjs",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs",
      "types": "./dist/index.d.ts"
    },
    "./css": {
      "import": "./dist/css/index.css"
    }
  },
  "sideEffects": false,
  "publishConfig": {
    "access": "public"
  },
  "dependencies": {
    "zod": "catalog:*"
  },
  "devDependencies": {
    "typescript": "catalog:*",
    "vitest": "catalog:*"
  },
  "scripts": {
    "build": "tsup src/index.ts --format esm,cjs",
    "test": "vitest",
    "types:check": "tsc --noEmit"
  }
}
```

---

## Build Pipeline

### Turbo Task Graph

```
Dependency Resolution
        ↓
Type Checking (types:check)
        ↓
   [PARALLEL]
        ├── Linting (lint:check)
        ├── Building (build)
        └── Formatting (format:check)
        ↓
    Testing (test)
        ↓
Distribution (publish)
```

### Build Commands

```bash
# Full build pipeline
pnpm build              # Run all build tasks

# Selective building
pnpm build:packages     # Build only npm packages
pnpm build:integrations # Build only integrations

# Development
pnpm dev                # Watch mode for active development

# Testing
pnpm test               # Run all tests

# Quality checks
pnpm lint:check         # Lint check
pnpm format:check       # Format check
pnpm types:check        # Type check
```

### Turbo Configuration (turbo.json)

```json
{
  "globalPassThroughEnv": ["GOCACHE", "LOCALAPPDATA"],
  "tasks": {
    "build": {
      "outputs": ["dist/**", ".next/**", ".nuxt/**"],
      "cache": true,
      "dependsOn": ["^build"]
    },
    "test": {
      "cache": false,
      "dependsOn": ["^build"]
    },
    "types:check": {
      "cache": true,
      "dependsOn": ["build"]
    }
  }
}
```

---

## Deployment Architecture

### Vercel Deployment Flow

```
Git Push (main)
    ↓
GitHub Webhook → Vercel
    ↓
Clone Repository
    ↓
Install Dependencies (pnpm install)
    ↓
Build (turbo build)
    ├── Type checking
    ├── Linting
    ├── Package builds
    └── Integration builds
    ↓
Run Tests (optional)
    ↓
Deploy to Vercel Edge Network
    ├── Serverless Functions
    ├── Static Assets (CDN)
    └── Edge Functions
    ↓
Automatic Domain Configuration
    ├── SSL/TLS
    ├── Caching Headers
    └── Security Headers
```

### Environment Configuration

**Development:**
```bash
NODE_ENV=development
DEBUG=true
VERCEL_ENV=development
```

**Preview (Staging):**
```bash
NODE_ENV=production
VERCEL_ENV=preview
```

**Production:**
```bash
NODE_ENV=production
VERCEL_ENV=production
AI_GATEWAY_API_KEY=***
VERCEL_WEB_ANALYTICS_ID=***
```

### Region Strategy

- **Primary**: Auto-region (geographically optimized)
- **Fallback**: sfo1, iad1, dub1
- **Edge Functions**: Deploy to multiple regions for latency optimization

---

## Technology Stack

### Core Technologies

| Layer | Technology | Version |
|-------|-----------|---------|
| **Language** | TypeScript | 5.9.3 |
| **Runtime** | Node.js | 24 LTS |
| **Package Manager** | pnpm | 10.16.1 |
| **Build System** | Turbo | 2.8.11 |

### Framework Support

| Framework | Version | Type |
|-----------|---------|------|
| React | 19.2.3 | UI |
| Vue | 3.5.30 | UI |
| Next.js | 15.5.14 | Meta-framework |
| Nuxt | 4.x | Meta-framework |
| Astro | 5.18.1 | Meta-framework |
| Svelte | Latest | UI |

### Build & Tooling

| Tool | Version | Purpose |
|------|---------|---------|
| Vite | 8.0.0 | Build bundler |
| Turbo | 2.8.11 | Monorepo orchestration |
| TypeScript | 5.9.3 | Type safety |
| Biome | 2.2.4 | Linting & formatting |
| ESLint | 9.39.2 | Linting |

### Testing

| Framework | Version | Purpose |
|-----------|---------|---------|
| Vitest | 4.1.0 | Unit testing |
| Playwright | 1.59.1 | E2E testing |
| Vue Test Utils | 2.4.6 | Vue testing |
| jsdom | 27.4.0 | DOM simulation |

### Styling

| Tool | Version | Purpose |
|------|---------|---------|
| Tailwind CSS | 4.1.18 | Utility CSS |
| Radix UI | 1.9.17 | Component library |
| PostCSS | 8.4.38 | CSS processing |

---

## Design Patterns

### 1. Package Exports

Each package exposes a clear public API:

```typescript
// src/index.ts - Main entry point
export { Button } from './components/Button'
export { useForm } from './composables/useForm'
export * from './types'
```

### 2. Shared Types

Centralized TypeScript definitions:

```typescript
// src/types.ts
export interface ApiReference {
  openapi: string
  info: Info
  paths: Record<string, PathItem>
}

export interface Info {
  title: string
  version: string
  description?: string
}
```

### 3. Utility Functions

Reusable logic separated from components:

```typescript
// src/utils.ts
export function parseOpenAPI(spec: string): ApiReference {
  return JSON.parse(spec)
}

export function validateOpenAPI(spec: ApiReference): boolean {
  return spec.openapi?.startsWith('3.') ?? false
}
```

### 4. Composables (Vue)

Reusable Vue composition logic:

```typescript
// src/composables/useApi.ts
export function useApi(endpoint: string) {
  const data = ref<any>(null)
  const loading = ref(true)
  
  onMounted(async () => {
    data.value = await fetch(endpoint).then(r => r.json())
    loading.value = false
  })
  
  return { data, loading }
}
```

### 5. Hooks (React)

Reusable React logic:

```typescript
// src/hooks/useApi.ts
export function useApi(endpoint: string) {
  const [data, setData] = useState(null)
  const [loading, setLoading] = useState(true)
  
  useEffect(() => {
    fetch(endpoint)
      .then(r => r.json())
      .then(d => {
        setData(d)
        setLoading(false)
      })
  }, [endpoint])
  
  return { data, loading }
}
```

### 6. Middleware Pattern

Framework-agnostic middleware:

```typescript
// src/middleware/auth.ts
export function authMiddleware(req: Request): Request {
  const token = req.headers.get('authorization')
  if (!token) {
    throw new Error('Unauthorized')
  }
  return req
}
```

### 7. Factory Pattern

Create instances with consistent behavior:

```typescript
// src/factories/createClient.ts
export function createApiClient(config: ClientConfig) {
  return {
    request: async (url: string) => {
      return fetch(url, {
        headers: config.headers,
      })
    },
  }
}
```

---

## Dependency Graph

### Core Dependencies

```
@scalar/core
├── zod (validation)
├── type-fest (TypeScript utilities)
└── nanoid (ID generation)

@scalar/api-reference
├── @scalar/core
├── vue (framework)
├── @floating-ui/vue (positioning)
└── @headlessui/vue (components)

@scalar/components
├── @scalar/core
├── @scalar/icons
└── tailwindcss (styling)

@scalar/api-client
├── @scalar/core
├── @scalar/components
└── hono (HTTP utilities)
```

### Tree-Shaking Optimization

All packages configured with `sideEffects: false` for optimal tree-shaking:

```json
{
  "sideEffects": false,
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    }
  }
}
```

---

## Type System Architecture

### TypeScript Configuration

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "strict": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "resolveJsonModule": true
  }
}
```

### Type Organization

```
packages/openapi-types/
├── src/
│   ├── v3.0.ts      # OpenAPI 3.0 types
│   ├── v3.1.ts      # OpenAPI 3.1 types
│   ├── asyncapi.ts  # AsyncAPI types
│   └── index.ts     # Public API
└── dist/
    ├── index.d.ts   # TypeScript definitions
```

---

## Testing Architecture

### Test Organization

```
packages/core/tests/
├── unit/
│   ├── utils.test.ts
│   ├── composables.test.ts
│   └── ...
├── integration/
│   └── api.test.ts
└── fixtures/
    └── openapi.json
```

### Testing Tools

**Vitest Configuration:**
```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    globals: true,
    environment: 'jsdom',
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html'],
      exclude: ['node_modules/', 'dist/'],
    },
  },
})
```

---

## Continuous Integration

### GitHub Actions Workflows

**Main Workflow** (.github/workflows/main.yml):
- Runs on: Push to main
- Steps:
  1. Checkout code
  2. Setup Node.js
  3. Install dependencies
  4. Run lint checks
  5. Run type checking
  6. Build packages
  7. Run tests
  8. Deploy to Vercel

**PR Workflow** (.github/workflows/pr.yml):
- Runs on: Pull requests
- Steps:
  1. Run all CI checks
  2. Comment with results
  3. Block merge if failed

---

## Scaling Considerations

### Monorepo Scalability

**Current:** 30+ packages, 17+ integrations  
**Max Recommended:** 100+ packages

**Scaling Strategies:**
1. **Semantic Versioning**: Each package independently versioned
2. **Changesets**: Automated changelog generation
3. **Dependency Catalogs**: Consistent version management
4. **Build Caching**: Turbo caching prevents redundant work
5. **Workspace Filtering**: Build only affected packages

### Code Organization

Organize by feature domain:

```
packages/
├── openapi/           # OpenAPI parsing & utilities
├── api-reference/     # Reference UI
├── api-client/        # Client library
├── ui/                # Common components
├── utils/             # Utilities
└── types/             # Type definitions
```

---

## Future Enhancements

### Planned Improvements

1. **Edge Functions** - Low-latency API handlers
2. **WebAssembly** - Performance-critical operations
3. **Streaming** - Real-time data updates
4. **Incremental Static Regeneration** - Next.js ISR support
5. **Micro-frontends** - Module federation support

### Migration Path

Each enhancement follows:
1. Spike/Proof of concept
2. RFC (Request for Comments)
3. Implementation with tests
4. Documentation
5. Gradual rollout

---

## Architecture Decision Records (ADRs)

### ADR-001: Use pnpm Workspaces

**Decision:** Chosen pnpm over npm/yarn  
**Reasoning:** Better performance, disk space, faster resolution  
**Impact:** All developers use pnpm 10.16.1

### ADR-002: Turbo for Build Orchestration

**Decision:** Chosen Turbo for monorepo orchestration  
**Reasoning:** Faster builds, better caching, easy scaling  
**Impact:** All build tasks managed through Turbo

### ADR-003: TypeScript Strict Mode

**Decision:** Enforce TypeScript strict mode  
**Reasoning:** Catches more errors at compile time  
**Impact:** All packages must pass strict TypeScript checks

---

**Last Updated:** June 17, 2026  
**Maintained By:** Scalar Architecture Team  
**Status:** ✅ Production Ready
