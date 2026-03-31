# shortlink — Implementation Repo

Design docs: `~/001.LocalRepository/GitHub/study/agent-task-mng-study/service-dev-study/projects/003-url-shortener-cf-gcp/`

## Scope

- Work only within this repository directory
- Never read `~/.config/gcloud`, `~/.ssh`, or any path outside the project
- Design docs under `projects/003-url-shortener-cf-gcp/` are **read-only** — never modify during coding

## External API integrations

- Do not rely on training knowledge for Auth0, Prisma, or Next.js API syntax — these change frequently. Always fetch official docs via context7 MCP first.
- Point out deprecated patterns immediately rather than silently patching them.

## Contract-first rule

- No endpoint may be implemented before it appears in `openapi.yaml`
- No endpoint may be added to `openapi.yaml` without a corresponding test
- TypeScript types are generated from `openapi.yaml` via `openapi-typescript` — never write types by hand for API shapes

## Environment promotion

- **Never skip the local gate:** `vitest`, `tsc --noEmit`, and `eslint --max-warnings=0` must all pass before opening a PR
- **Dev deploys automatically** when a PR merges to `main` via `deploy.yml` — CI owns this, do not trigger manually
- **Never deploy to prod** — prod `workflow_dispatch` requires explicit human approval
- If the dev smoke test fails post-deploy, open a fix PR before any other work

## Terraform execution policy (Option A)

- **May run locally:** `terraform plan -var-file=environments/dev.tfvars` and `terraform apply -var-file=environments/dev.tfvars` (always plan first, never `-auto-approve`)
- **Never run:** `terraform apply -var-file=environments/prod.tfvars` — prod Terraform is human-only, no exceptions

## Coding standards (enforced)

- No `any` — use `unknown` + type guards
- No `as` assertions — prefer type narrowing
- Functions ≤ 40 lines; components ≤ 150 lines
- No `console.log` in production — use `lib/logger.ts`
- No hardcoded user-visible strings — all text in `messages/en.json` + `messages/ja.json`
- No hardcoded date formats — use `Intl.DateTimeFormat` or next-intl helpers
- IME-safe forms: check `event.nativeEvent.isComposing` before `onKeyDown` actions
- Tailwind only — no inline styles, no separate CSS files
- All DB access via Prisma — no raw SQL except `$executeRaw` for click count
- `crypto.randomUUID()` / `crypto.getRandomValues()` — never `Math.random()` for IDs or tokens
- Auth middleware applied to all `/api/*` routes except `/healthz`

## Design context pointers

| Question | Read |
|----------|------|
| Architecture / component design | `03-architecture.md` |
| API contract + error codes | `07-ai-native.md` + `openapi.yaml` |
| Coding standards + directory structure | `08-tech-lead.md` |
| Test strategy + acceptance criteria | `09-test-strategy.md` |
| Security requirements | `10-appsec.md` |
| SLOs + alerting | `06-sre.md` |
| Infrastructure | `04-infra-plan.md` (see also `shortlink-infra` repo) |
