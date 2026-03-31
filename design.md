# Shortlink — Design

> OpenSpec artifact: Architecture + API contract
> Source phases: 03-architecture, 07-ai-native, 08-tech-lead

---

## System Architecture

### Component overview

| Component | Technology | Responsibility |
|-----------|-----------|----------------|
| DNS + CDN | Cloudflare Free Tier | DNS, TLS 1.3, 301 caching, IP rate limiting |
| Application | Next.js 15 (App Router, TypeScript) on GCP Cloud Run | SSR pages, API routes, redirect handler, auth middleware |
| Database | Cloud SQL PostgreSQL 15 (db-f1-micro, asia-northeast1) | Users, links, API keys, audit log, click counts |
| DB proxy | Cloud SQL Auth Proxy (sidecar) | IAM-based auth, no public IP |
| Human auth | Auth0 (free tier) + `@auth0/nextjs-auth0` | OAuth 2.0 (Google + GitHub), RS256 JWT, session cookies |
| Secrets | GCP Secret Manager | Auth0 client secret, issuer URL, session key |
| Observability | GCP Cloud Logging + Monitoring + Trace (OpenTelemetry) | Structured logs, metrics, traces, uptime checks, Slack alerts |

**Key decision:** Single Cloud Run service — Next.js serves everything (frontend, API, redirects). No separate backend. Simplifies deployment and cost.

### System diagram

```
Browser / CI Agent
        │ HTTPS
        ▼
Cloudflare (DNS · TLS · CDN · Rate Limiting)
        │ HTTPS (cache miss)
        ▼
GCP Cloud Run — Next.js 15
  ├── middleware.ts         (locale detection, auth, rate limiting, request ID)
  ├── app/[locale]/         (SSR pages: landing, dashboard, settings, privacy)
  ├── app/[...path]/        (catch-all: slug redirect or 404)
  └── app/api/              (links CRUD, keys management, healthz)
        │                          │
        ▼                          ▼
Cloud SQL Auth Proxy          Auth0 JWKS
Cloud SQL PostgreSQL 15
        │
        ▼
GCP Observability (Logging · Monitoring · Trace)
        │
        ▼
Slack alerts
```

---

## Data Model

### `users`
```sql
id          UUID  PK
auth0_sub   TEXT  UNIQUE NOT NULL   -- Auth0 subject claim
email       TEXT  NOT NULL
name        TEXT
locale      VARCHAR(5) DEFAULT 'en'
created_at  TIMESTAMPTZ
updated_at  TIMESTAMPTZ
```

### `links`
```sql
id           UUID  PK
slug         VARCHAR(50) UNIQUE NOT NULL  -- 3-50 chars, alphanumeric+hyphens
original_url TEXT  NOT NULL
owner_id     UUID  FK users.id ON DELETE CASCADE
click_count  BIGINT DEFAULT 0
created_at   TIMESTAMPTZ
-- Indexes: idx_links_slug (hot path), idx_links_owner, idx_links_owner_created
```

**Auto-slug:** 7-char random `[A-Za-z0-9]` via `crypto.randomBytes()`. Retry on collision (max 3).
**Reserved slugs:** `api`, `en`, `ja`, `admin`, `settings`, `privacy`, `healthz`, `favicon.ico`, `robots.txt`, `sitemap.xml`

### `api_keys`
```sql
id              UUID  PK
name            VARCHAR(100)
key_hash        TEXT  NOT NULL   -- bcrypt (brute-force resistance)
key_lookup_hash TEXT  NOT NULL   -- SHA-256 (O(1) lookup)
key_prefix      VARCHAR(12)      -- "sk_...xxxx" for display
owner_id        UUID  FK users.id ON DELETE CASCADE
created_at      TIMESTAMPTZ
last_used_at    TIMESTAMPTZ
revoked_at      TIMESTAMPTZ      -- NULL = active
-- Partial index on key_lookup_hash WHERE revoked_at IS NULL
```

**Rules:** Max 5 active keys/user. No expiration (revoke-only). Links survive key revocation.
**Auth flow:** `SHA256(key)` → lookup → `bcrypt.compare(key, hash)` → resolve `owner_id`.

### `audit_log`
```sql
id          UUID  PK
actor_id    UUID  FK users.id
action      VARCHAR(50)   -- "delete_link", "revoke_key", "delete_account"
target_type VARCHAR(50)   -- "link", "api_key", "account"
target_id   UUID
metadata    JSONB
created_at  TIMESTAMPTZ
-- No UPDATE/DELETE — immutable. Retention: 1 year (SOC 2).
```

### `idempotency_keys`
```sql
key        TEXT  PK
response   JSONB
status     INT
created_at TIMESTAMPTZ
expires_at TIMESTAMPTZ   -- TTL 24h; cleaned up periodically
-- Index on expires_at for cleanup
```

---

## API Contract

Full spec: `openapi.yaml` (repo root). Types generated via `openapi-typescript`.

### Endpoint summary

| Method | Path | Auth | Response |
|--------|------|------|---------|
| GET | `/healthz` | None | `{ status, db, version, timestamp }` |
| GET | `/{slug}` | None | 301 redirect (or 404) |
| GET | `/api/links` | JWT or API key | `{ items[], total, page, per_page }` |
| GET | `/api/links/{id}` | JWT or API key | `Link` |
| POST | `/api/links` | JWT or API key | 201 `Link` |
| DELETE | `/api/links/{id}` | JWT or API key | 204 |
| GET | `/api/keys` | JWT only | `ApiKey[]` |
| POST | `/api/keys` | JWT only | 201 `{ id, key, key_prefix, name, created_at }` |
| DELETE | `/api/keys/{id}` | JWT only | 204 |

### Error format (always JSON, never HTML)

```json
{
  "error": "invalid_url",
  "message": "The provided URL is not valid",
  "request_id": "req_abc123def456",
  "details": [{ "field": "url", "reason": "must start with http:// or https://" }]
}
```

`error` is always English (machine-readable). `message` is locale-aware via `Accept-Language`.

### Rate limits

| Auth | Write | Read |
|------|-------|------|
| IP (Cloudflare L1) | 10/IP/min | 100/IP/min |
| API key (app L2) | 30/key/min | 300/key/min |
| JWT session (app L2) | 10/user/min | 60/user/min |

Every response includes: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `X-Request-Id`.

### Idempotency

`POST /api/links` supports `Idempotency-Key: <uuid>` header. Same key + same body = cached response (24h TTL).

---

## Directory Structure

```
shortlink/
├── app/                          # Next.js 15 root
│   ├── app/
│   │   ├── [...path]/page.tsx    # Catch-all: redirect or locale dispatch
│   │   ├── [locale]/             # SSR pages (landing, dashboard, settings, privacy)
│   │   ├── api/
│   │   │   ├── auth/[...auth0]/  # Auth0 SDK handler
│   │   │   ├── links/            # GET+POST / GET+DELETE by id
│   │   │   ├── keys/             # GET+POST / DELETE by id
│   │   │   └── healthz/          # Health check
│   │   └── layout.tsx
│   ├── components/
│   │   ├── ui/                   # Button, Input, Modal, Card, Toast
│   │   └── features/             # LinkTable, CreateLinkForm, ApiKeyPanel
│   ├── lib/
│   │   ├── prisma.ts             # Prisma client singleton
│   │   ├── auth.ts               # resolveAuth() — JWT + API key
│   │   ├── errors.ts             # AppError class + errorResponse()
│   │   ├── logger.ts             # Structured JSON logger
│   │   ├── rate-limit.ts         # Per-key/per-user rate limiter
│   │   ├── idempotency.ts        # Idempotency-Key middleware
│   │   └── request-id.ts         # X-Request-Id generation
│   ├── middleware.ts              # Locale, auth, rate limit, request ID
│   ├── messages/
│   │   ├── en.json
│   │   └── ja.json
│   ├── prisma/
│   │   ├── schema.prisma
│   │   └── migrations/
│   ├── tests/
│   │   ├── unit/
│   │   ├── integration/
│   │   └── e2e/
│   ├── .env.example
│   ├── next.config.ts
│   ├── Dockerfile
│   └── package.json
├── infra/ → see shortlink-infra repo
├── .github/workflows/
│   ├── ci.yml
│   ├── deploy.yml
│   └── terraform.yml
└── openapi.yaml
```

---

## Technology Stack

| Concern | Choice | Key reason |
|---------|--------|-----------|
| Framework | Next.js 15 (App Router) | SSR + API routes in one service; native i18n support |
| Runtime | Node.js 22 LTS on Cloud Run | Managed, scale-to-zero, container-native |
| Database | Cloud SQL PostgreSQL 15 | Relational, referential integrity, Prisma support |
| ORM | Prisma 6 | Type-safe queries, migrations, schema formatting |
| Auth | Auth0 + `@auth0/nextjs-auth0` | Managed OAuth, PKCE, RS256 JWT, free tier |
| i18n | next-intl | App Router native, URL prefix routing, type-safe keys |
| Styling | Tailwind CSS | Utility-first, no CSS files, consistent design |
| Testing | Vitest (unit+integration) + Playwright (E2E) | Fast, TS-native, Prisma-compatible |
| IaC | Terraform | Industry standard, GCS state backend |
| CI/CD | GitHub Actions + WIF (no long-lived SA keys) | OIDC-based auth to GCP, secure |

---

## Coding Standards (enforced)

- No `any` — use `unknown` + type guards
- Functions ≤ 40 lines; components ≤ 150 lines
- No `console.log` in production — use `lib/logger.ts`
- No hardcoded strings — all text in `messages/en.json` + `messages/ja.json`
- No hardcoded date formats — use `Intl.DateTimeFormat` or next-intl helpers
- IME-safe: check `event.nativeEvent.isComposing` before `onKeyDown` actions
- Tailwind only — no inline styles, no separate CSS files
- All DB access via Prisma — no raw SQL except `$executeRaw` for click count
- `crypto.randomUUID()` / `crypto.getRandomValues()` — never `Math.random()` for IDs

---

## Environment Promotion

```
Local (npm run dev)
  Gate: vitest + tsc --noEmit + eslint --max-warnings=0 + healthz → ok
  ▼
Dev (GCP: shortlink-dev, auto on merge to main)
  Gate: healthz 200 + no error spike (15min) + redirect flow end-to-end
  ▼
Prod (GCP: shortlink-prod, manual workflow_dispatch only)
```

Agent rules: never trigger prod deploy. If dev smoke test fails, fix before any other work.
Terraform: Option A (local apply to dev OK, never -auto-approve, never prod.tfvars).
