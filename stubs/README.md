# Local Service Stubs

Run `docker compose up` from the repo root to start the full local stack.
Set `DATABASE_URL` and `AUTH0_*` in `.env.local` to point at these stubs (not the cloud).

| Service | Stub tool | API / SDK version stubbed | Created | Last verified |
|---------|-----------|--------------------------|---------|--------------|
| Auth0 | mock-oauth2-server (latest) | OIDC Core 1.0 / @auth0/nextjs-auth0@x.x.x | 2026-04-01 | 2026-04-01 |
| PostgreSQL | postgres:15 (docker) | PostgreSQL 15 / prisma@x.x.x | 2026-04-01 | 2026-04-01 |

> Fill in the `x.x.x` version fields after running `npm install` and checking `package.json`.

## Local .env.local values for stubs

```bash
# Auth0 stub (mock-oauth2-server on port 8080)
AUTH0_ISSUER_BASE_URL=http://localhost:8080/shortlink-dev
AUTH0_CLIENT_ID=shortlink-local
AUTH0_CLIENT_SECRET=any-secret-works-with-stub
AUTH0_BASE_URL=http://localhost:3000
AUTH0_SECRET=dev-session-secret-32-chars-minimum

# PostgreSQL stub (docker on port 5432)
DATABASE_URL=postgresql://shortlink:shortlink@localhost:5432/shortlink
```

## Updating a stub

1. Update stub tool version in `docker-compose.yml`
2. Update `Last verified` and `API / SDK version` in this table
3. Run `docker compose up` and confirm `npx vitest run` still passes

## When to suspect a stale stub

- A test fails locally after upgrading `@auth0/nextjs-auth0` or `prisma`
- `Last verified` is older than the relevant entry in `package.json`
