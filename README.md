# QueCocinoIA Backend

Technical showcase for the Cloudflare Workers backend that powers authentication, ingredient inventory, and AI recipe generation.

## Overview

This service is a serverless API built with Hono and deployed on Cloudflare Workers. It combines:
- JWT-based auth with refresh token rotation,
- user-scoped inventory persistence in Cloudflare D1 via Prisma,
- recipe generation through Workers AI with strict post-validation and fallback logic.

## Core capabilities

- Local auth: register, login, refresh, logout, profile update, password change, account deletion.
- OAuth auth: Google and Facebook with state validation and redirect allowlist.
- Ingredient CRUD constrained per authenticated user (`usuarioId` scoping).
- Recipe generation pipeline:
  - AI prompt from current stock,
  - strict validation against allowed ingredients (except `agua`),
  - ranking of valid results,
  - deterministic fallback generation if AI fails.
- Cook endpoint that decrements stock safely with optimistic concurrency checks.
- Rate limits:
  - auth flows by client IP/scope (in-memory rolling window),
  - recipe generation limited to 1 attempt per hour per user (unless test mode bypass is active).

## Stack

- Cloudflare Workers runtime
- Hono (routing/middleware)
- Prisma ORM + `@prisma/adapter-d1`
- Cloudflare D1 (SQLite)
- Cloudflare Workers AI
- `jose` (JWT), `bcryptjs` (password hashing)

## Architecture

```text
src/
  index.js                  # App wiring, CORS, security headers, route mounting, global error handler
  routes/
    auth.js                 # Auth + OAuth + session endpoints
    ingredientes.js         # Ingredient routes (auth required)
    recetas.js              # Recipe routes (auth required)
  controllers/
    ingredientesController.js
    recetasController.js
  services/
    workersAiClient.js      # Workers AI execution + JSON extraction
    cocinarService.js       # Stock subtraction + concurrency-safe updates
    fallbackRecetas.js      # Deterministic recipe fallback
  middlewares/
    auth.js                 # Bearer token verification
    authRateLimit.js        # Auth anti-bruteforce limiter
    limitPorHora.js         # Recipe generation limiter
    testMode.js             # Test bypass control
  utils/
    normalize.js            # Text/unit normalization
    validaRecetas.js        # AI recipe guardrails
    rankRecetas.js          # Quality scoring and sort
    promptBuilder.js        # Constrained AI prompt generator
    jwt.js, cookies.js      # Token and cookie helpers
  models/
    index.js                # Prisma client per D1 binding
```

## Recipe generation pipeline

1. Resolve ingredient source:
  - request body ingredients, or
  - stored user ingredients if body does not include an array.
2. Build a constrained prompt (`promptBuilder`) with max quantities per ingredient.
3. Call Workers AI (up to 2 attempts with different temperature settings).
4. Validate recipes (`validaRecetas`) so output cannot use out-of-stock or unknown ingredients (except `agua`).
5. Rank recipes (`rankRecetas`) and persist results into `Consulta`.
6. If no valid AI output exists, generate deterministic fallback recipes and validate/rank again.

## Security model

- CORS allowlist from `CORS_ORIGIN`.
- OAuth redirect allowlist from `OAUTH_ALLOWED_REDIRECT_ORIGINS`.
- Global response security headers (`X-Frame-Options`, `X-Content-Type-Options`, strict CSP, etc.).
- Access token in `Authorization: Bearer ...`.
- Refresh token in HttpOnly cookie (`qc_refresh`) with optional `SameSite`/`Secure` tuning.
- Refresh token rotation with hashed token storage in DB.

## Data model (Prisma)

| Entity | Purpose | Key constraints |
| --- | --- | --- |
| `Usuario` | Auth/account/preferences | `email` unique, `provider+providerId` unique |
| `Ingrediente` | User inventory item | unique (`usuarioId`, `nombreNorm`) |
| `Consulta` | Stored recipe generation result | indexed by (`usuarioId`, `createdAt`) and (`usuarioId`, `fecha`) |

Schema file: `prisma/schema.prisma`.

## Environment and bindings

### Cloudflare bindings

- `DB` (D1 binding) - required
- `AI` (Workers AI binding) - required

### Runtime vars (`.dev.vars` for local worker runtime)

| Variable | Required | Notes |
| --- | --- | --- |
| `JWT_SECRET` | Yes | Access token signing secret |
| `JWT_REFRESH_SECRET` | Recommended | Refresh signing secret (falls back to `JWT_SECRET`) |
| `APP_ENV` | Yes | Use `development` locally, `production` in prod |
| `CORS_ORIGIN` | Yes | Comma-separated allowed frontend origins |
| `OAUTH_ALLOWED_REDIRECT_ORIGINS` | Yes | Comma-separated OAuth redirect origins |
| `GOOGLE_CLIENT_ID` | OAuth | Required for Google OAuth |
| `GOOGLE_CLIENT_SECRET` | OAuth | Required for Google OAuth |
| `GOOGLE_CALLBACK_URL` | OAuth | Required for Google OAuth |
| `FACEBOOK_CLIENT_ID` | OAuth | Required for Facebook OAuth |
| `FACEBOOK_CLIENT_SECRET` | OAuth | Required for Facebook OAuth |
| `FACEBOOK_CALLBACK_URL` | OAuth | Required for Facebook OAuth |
| `FACEBOOK_GRAPH_VERSION` | No | Defaults to `v19.0` |
| `CLOUDFLARE_AI_MODEL` | No | Model override for Workers AI |
| `TEST_MODE` | No | Global test bypass in non-production |
| `TEST_MODE_TOKEN` | No | Per-request bypass via `x-test-mode` header |
| `COOKIE_SAMESITE` | No | `Lax` (default), `Strict`, or `None` |
| `COOKIE_SECURE` | No | Force `Secure` cookie flag |

### Prisma CLI vars (`.env`)

`DATABASE_URL` is required for local Prisma CLI commands only (not used by Worker runtime).

## Local setup

1. Install dependencies:
```bash
npm install
```
2. Create runtime vars:
```bash
copy .dev.vars.example .dev.vars
```
3. Create Prisma CLI vars:
```bash
copy .env.example .env
```
4. Generate Prisma client:
```bash
npm run prisma:generate
```
5. Apply local D1 migrations:
```bash
npm run d1:migrate:local
```
6. Start local worker:
```bash
npm run dev
```
7. Verify health:
```text
GET http://127.0.0.1:8787/health
```

## Scripts

- `npm run dev` - local Worker runtime
- `npm run dev:remote` - remote-mode Worker runtime
- `npm run deploy` - deploy Worker
- `npm run check:syntax` - syntax check for all `src/**/*.js`
- `npm run lint` - alias to syntax check
- `npm run test` - minimal Node-based smoke tests
- `npm run prisma:generate` - generate Prisma client
- `npm run prisma:migrate:dev` - run Prisma dev migration flow
- `npm run prisma:studio` - open Prisma Studio
- `npm run d1:migrate:local` - apply local D1 migrations
- `npm run d1:migrate:remote` - apply remote D1 migrations

## API surface

### Health
- `GET /health`

### Auth
- `POST /auth/register`
- `POST /auth/login`
- `POST /auth/refresh`
- `POST /auth/logout`
- `GET /auth/me`
- `PATCH /auth/me`
- `POST /auth/change-password`
- `DELETE /auth/me`
- `GET /auth/google`
- `GET /auth/google/callback`
- `GET /auth/facebook`
- `GET /auth/facebook/callback`
- `POST /auth/dev/token` (non-production + test mode only)

### Ingredients (auth required)
- `GET /api/ingredientes`
- `POST /api/ingredientes`
- `DELETE /api/ingredientes`
- `GET /api/ingredientes/:id`
- `PUT /api/ingredientes/:id`
- `DELETE /api/ingredientes/:id`

### Recipes (auth required)
- `POST /api/recetas`
- `POST /api/recetas/generar`
- `GET /api/recetas`
- `GET /api/recetas/:id`
- `DELETE /api/recetas/:id`
- `POST /api/recetas/cocinar`
- `POST /api/recetas/:id/cocinar`

## Deployment notes

1. Configure production secrets with Wrangler (`wrangler secret put`).
2. Apply remote D1 migrations:
```bash
npm run d1:migrate:remote
```
3. Deploy:
```bash
npm run deploy
```

`wrangler.toml` already defines:
- `DB` binding,
- `AI` binding,
- observability logs/traces enabled.
