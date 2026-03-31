# Shortlink — Tasks

> OpenSpec artifact: Acceptance criteria → implementation task list
> Source phase: 09-test-strategy

---

## Implementation Order

Work through phases in order. Each phase has a "gate" — do not start the next phase until all gate items pass.

---

## Phase 0: Repo Bootstrap

**Gate:** `npm run dev` starts without errors; `/api/healthz` returns `{ status: "ok" }`.

- [ ] Scaffold Next.js 15 app with TypeScript (`create-next-app --typescript`)
- [ ] Install dependencies: `@auth0/nextjs-auth0`, `prisma`, `next-intl`, `zod`, `tailwindcss`
- [ ] Copy `.env.example` template; fill in local values
- [ ] Configure `next.config.ts` (`output: 'standalone'`, next-intl plugin)
- [ ] Configure `tailwind.config.ts`
- [ ] Implement `GET /api/healthz` → `{ status: "ok", db: "connected", version, timestamp }`
- [ ] Implement `lib/logger.ts` (structured JSON logger)
- [ ] Implement `lib/request-id.ts` + add `X-Request-Id` in `middleware.ts`
- [ ] Set up Vitest (`vitest.config.ts`); add first unit test (request ID format)
- [ ] Add Prisma schema with all 5 tables; run `prisma migrate dev --name init`
- [ ] Set up `docker-compose.yml` (postgres:15 + mock-oauth2-server stubs)
- [ ] Set up `stubs/README.md` with version table

---

## Phase 1: Auth

**Gate:** OAuth login works end-to-end locally; API key round-trip (create → use → revoke) works.

- [ ] Configure `@auth0/nextjs-auth0`: `AUTH0_*` env vars, `app/api/auth/[...auth0]/route.ts`
- [ ] Implement `lib/auth.ts`: `resolveAuth()` — JWT session + API key paths
- [ ] API key auth: SHA-256 lookup → bcrypt verify → resolve `owner_id`
- [ ] Apply auth middleware to all `/api/*` routes (except `/healthz`)
- [ ] `POST /api/keys`: create key (JWT only), SHA-256 + bcrypt storage, return raw key once
- [ ] `GET /api/keys`: list keys (prefix + metadata only, never full key)
- [ ] `DELETE /api/keys/{id}`: revoke key; write audit log entry
- [ ] Enforce max 5 active keys per user
- [ ] Integration tests: valid session, expired session, valid API key, revoked API key, no auth

---

## Phase 2: Link CRUD

**Gate:** All acceptance criteria for Create, Redirect, List, Delete pass.

- [ ] `POST /api/links`: create link, auto-slug (7-char `[A-Za-z0-9]`), reserved slug check
- [ ] `POST /api/links`: idempotency key support (`Idempotency-Key` header + `IdempotencyKey` table)
- [ ] `GET /{slug}`: redirect handler — 301 + `Cache-Control: public, max-age=86400`
- [ ] `GET /{slug}`: click count increment (fire-and-forget `$executeRaw`)
- [ ] `GET /api/links`: paginated list (`page`, `per_page`, owner-scoped)
- [ ] `GET /api/links/{id}`: single link (owner-scoped)
- [ ] `DELETE /api/links/{id}`: delete + audit log entry
- [ ] `lib/errors.ts`: `AppError` class + all error codes from `openapi.yaml`
- [ ] `lib/rate-limit.ts`: per-key (30/300) and per-user (10/60) limits; `X-RateLimit-*` headers
- [ ] Integration tests: all acceptance criteria scenarios in `09-test-strategy.md`

---

## Phase 3: Frontend

**Gate:** All 6 Playwright E2E flows pass; axe-core zero critical violations.

- [ ] Set up next-intl: `messages/en.json`, `messages/ja.json`, locale middleware, `[locale]/layout.tsx`
- [ ] S1 Landing page (`/[locale]/`): URL input, create button, result display
- [ ] S2 Login page (Auth0 Universal Login redirect)
- [ ] S3 Dashboard (`/[locale]/dashboard`): link table with click counts, delete button
- [ ] S4 API Key Management (`/[locale]/dashboard/api-keys`): generate key (show-once), revoke
- [ ] S5 Redirect 404 page (locale-aware)
- [ ] S6 Privacy Policy (`/[locale]/privacy`): EN + JA content
- [ ] Cookie consent banner (stores consent in localStorage / cookie)
- [ ] Copy-to-clipboard on link creation result
- [ ] Locale switcher: `/en/` ↔ `/ja/` with `<html lang={locale}>`
- [ ] `Accept-Language` detection on first visit (middleware → 307 to `/[locale]/`)
- [ ] All UI text in message catalogs — no hardcoded strings
- [ ] All interactive elements keyboard-reachable; modals trap focus; Escape closes
- [ ] All form fields have `<label>`; errors via `aria-describedby`; live regions for toasts

---

## Phase 4: Observability

**Gate:** Cloud Logging shows structured JSON; Cloud Monitoring shows latency/error metrics; one alert fires correctly in a test.

- [ ] Structured log on every API request: `method, path, status, latency_ms, owner_id (hashed), request_id`
- [ ] OpenTelemetry Node.js SDK → Cloud Trace for redirect path + write path
- [ ] GCP uptime check on `/healthz` (every 5 min, 3 regions)
- [ ] Alert: error rate > 5% for 5 min → Slack
- [ ] Alert: uptime check fails 2x consecutively → Slack
- [ ] Alert: CDN cache hit ratio < 40% at 30 days → Slack (use Cloudflare Analytics API)
- [ ] Alert: DB connections > 20 → Slack

---

## Phase 5: CI + Deploy

**Gate:** CI green on first PR; dev deploy succeeds; smoke test passes.

- [ ] `.github/workflows/ci.yml`: lint + typecheck + unit tests + integration tests (see `08-tech-lead.md §CI Workflow`)
- [ ] `.github/workflows/deploy.yml`: WIF auth → npm ci → test → Docker build → Artifact Registry → prisma migrate deploy → Cloud Run deploy → smoke test
- [ ] `.github/workflows/terraform.yml`: fmt-check + validate + plan on PR; apply on merge to main
- [ ] Dockerfile (multi-stage, `output: 'standalone'`)
- [ ] Branch protection: require PR + CI green before merge to `main`
- [ ] First real push to `main`: verify dev auto-deploy + smoke test passes

---

## Definition of Done (per feature)

- [ ] All acceptance criteria scenarios pass (automated)
- [ ] `openapi.yaml` updated if API surface changed
- [ ] Unit tests written (≥90% coverage on new logic)
- [ ] Integration tests written (happy path + error paths)
- [ ] CI pipeline green (lint + typecheck + test)
- [ ] Auth applied to all protected endpoints
- [ ] Input validated server-side (Zod schemas)
- [ ] No secrets in code or logs
- [ ] Structured log entry for key operations
- [ ] No hardcoded user-visible strings
- [ ] `messages/ja.json` 100% complete vs `messages/en.json`
- [ ] PR review checklist from `design.md §Coding Standards` completed

---

## Top 5 Edge Cases to Test Early

| # | Edge case | Risk |
|---|-----------|------|
| 1 | Concurrent auto-slug generation (UNIQUE collision) | Two instances generate same 7-char slug; one request fails without retry logic |
| 2 | DB connection pool exhaustion under spike | db-f1-micro: 25 max connections; 3 instances × 5 pool = 15; migration job adds more |
| 3 | Stale CDN cache after link deletion | 301 cached with max-age=86400; deleted link still redirects for up to 24h |
| 4 | API key used immediately after revocation | Race condition between revoked_at write and auth middleware read |
| 5 | Auth0 JWKS endpoint unavailable | All session auth fails if JWKS cache expires during Auth0 outage |
