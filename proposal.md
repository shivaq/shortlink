# Shortlink — Proposal

> OpenSpec artifact: Problem statement + scope
> Source phases: 00-ideation, 01-requirements

---

## Problem

A developer building side projects struggles with sharing long, ugly URLs in Slack, Notion, and README files — and has no self-hosted shortener that also works as an API for CI/CD automation.

## Target Users

**Human:** "Side-project Shibata" — a bilingual (EN/JA) developer who shortens links for daily sharing in Slack, Notion, and docs. High tech comfort; uses the product a few times per week.

**Agent:** CI/CD pipeline (GitHub Actions) that creates short links per PR/deploy and posts them to Slack. Expects deterministic JSON, idempotency on retry, and `Retry-After` on rate limits.

## Value Proposition

Self-hosted URL shortener with full data control, no third-party tracking, and a clean API designed for both human and AI agent consumers. Learning vehicle for Cloudflare + GCP multi-cloud architecture.

## Business Model

Free — personal learning project. Target cost: < $10/month (Cloud Run min-instances=0, Cloud SQL db-f1-micro).

## North Star Metric

Weekly active link creators (users or agents that create ≥1 link/week).

## MVP Scope

### In scope (v1)

- Create short link (custom slug optional)
- Redirect `short.example/<slug>` → original URL (301, CDN-cached)
- List / delete own links (web UI + API)
- OAuth login (Google or GitHub) for humans via Auth0
- API key auth for CI/CD agents
- Click count per link
- EN + JA locale support (URL prefix: `/en/`, `/ja/`)
- Cookie consent banner + privacy policy (EN/JA)
- Immutable audit log for destructive actions

### Out of scope (v1)

- Custom domains, detailed analytics, team/org features
- Cloudflare Workers / KV / D1
- Link expiration/TTL, QR codes, bulk creation
- Data export + account deletion (post-MVP, legally required under GDPR simulation)

## Key Risks

| Risk | Mitigation |
|------|-----------|
| Cloudflare CDN may not cache 301s effectively | Validate cache hit rate; Workers escape hatch if < 40% at 30 days |
| Cloud SQL cost creep | Monitor billing weekly; stop instance between dev sessions |
| Auth0 free tier limits | Verify before building |

## Compliance Posture

Simulating GDPR + CCPA + SOC 2. Drives: cookie consent, data deletion flow, privacy policy, immutable audit logs, vendor DPAs.
