# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is an AI API gateway/proxy built with Go. It aggregates 40+ upstream AI providers (OpenAI, Claude, Gemini, Azure, AWS Bedrock, etc.) behind a unified API, with user management, billing, rate limiting, and an admin dashboard.

## Tech Stack

- **Backend**: Go 1.22+, Gin web framework, GORM v2 ORM
- **Frontend**: React 19, TypeScript, Rsbuild, Base UI, Tailwind CSS
- **Databases**: SQLite, MySQL, PostgreSQL (all three must be supported)
- **Cache**: Redis (go-redis) + in-memory cache
- **Auth**: JWT, WebAuthn/Passkeys, OAuth (GitHub, Discord, OIDC, etc.)
- **Frontend package manager**: Bun (preferred over npm/yarn/pnpm)

## Build & Development Commands

### Backend (Go)

```bash
# Run backend directly
go run main.go

# Run all tests
go test ./...

# Run tests in a specific package
go test ./service/...

# Run a single test file
go test -v ./service/text_quota_test.go

# Run tests with match pattern
go test -v -run TestFunctionName ./path/to/package
```

### Frontend (web/default)

```bash
# Install dependencies
bun install

# Development server (proxies API to backend on port 3000)
bun run dev

# Production build
bun run build

# Type check
bun run typecheck

# Lint
bun run lint

# Format
bun run format

# Sync i18n translations
bun run i18n:sync
```

### Docker Development Stack

```bash
# Start full dev stack (PostgreSQL + Redis + backend)
make dev-api

# Start frontend dev server
make dev-web

# Start both (full development)
make dev

# Rebuild backend after Go code changes
make dev-api-rebuild

# Reset setup wizard state (clear root user, return to setup screen)
make reset-setup
```

### Full Build

```bash
# Build all frontends and start backend
make all

# Build default frontend only
make build-frontend

# Build classic frontend only
make build-frontend-classic
```

## Architecture

Layered architecture: Router -> Controller -> Service -> Model

```
router/        — HTTP routing (API, relay, dashboard, web)
controller/    — Request handlers
service/       — Business logic
model/         — Data models and DB access (GORM)
relay/         — AI API relay/proxy with provider adapters
  relay/channel/ — Provider-specific adapters (openai/, claude/, gemini/, aws/, etc.)
middleware/    — Auth, rate limiting, CORS, logging, distribution
setting/       — Configuration management (ratio, model, operation, system, performance)
common/        — Shared utilities (JSON, crypto, Redis, env, rate-limit, etc.)
dto/           — Data transfer objects (request/response structs)
constant/      — Constants (API types, channel types, context keys)
types/         — Type definitions (relay formats, file sources, errors)
i18n/          — Backend internationalization (go-i18n, en/zh)
oauth/         — OAuth provider implementations
pkg/           — Internal packages (cachex, ionet, billingexpr)
web/           — Frontend themes container
  web/default/ — Default frontend (React 19, Rsbuild, Base UI, Tailwind)
  web/classic/ — Classic frontend (React 18, Vite, Semi Design)
```

### Relay System (Key Path for AI API Requests)

The relay system handles all AI API requests:

1. `router/relay-router.go` routes requests to handlers
2. `relay/*.go` handlers process specific API formats (OpenAI, Claude, Gemini, etc.)
3. `relay/channel/*.go` provider-specific adapters transform requests/responses
4. `relay/relay_adaptor.go` coordinates the relay flow

When adding a new provider:
1. Create adapter in `relay/channel/<provider>/`
2. Register in `relay/channel/adapter.go`
3. Add channel type constant in `constant/channel.go`
4. Check `streamSupportedChannels` for StreamOptions support

## Internationalization (i18n)

### Backend (`i18n/`)
- Library: `nicksnyder/go-i18n/v2`
- Languages: en, zh

### Frontend (`web/default/src/i18n/`)
- Library: `i18next` + `react-i18next` + `i18next-browser-languagedetector`
- Languages: en (base), zh (fallback), fr, ru, ja, vi
- Translation files: `web/default/src/i18n/locales/{lang}.json` — flat JSON, keys are English source strings
- Usage: `useTranslation()` hook, call `t('English key')` in components

## Rules

### Rule 1: JSON Package — Use `common/json.go`

All JSON marshal/unmarshal operations MUST use the wrapper functions in `common/json.go`:

- `common.Marshal(v any) ([]byte, error)`
- `common.Unmarshal(data []byte, v any) error`
- `common.UnmarshalJsonStr(data string, v any) error`
- `common.DecodeJson(reader io.Reader, v any) error`
- `common.GetJsonType(data json.RawMessage) string`

Do NOT directly import or call `encoding/json` in business code. These wrappers exist for consistency and future extensibility.

Note: `json.RawMessage`, `json.Number`, and other type definitions from `encoding/json` may still be referenced as types, but actual marshal/unmarshal calls must go through `common.*`.

### Rule 2: Database Compatibility — SQLite, MySQL >= 5.7.8, PostgreSQL >= 9.6

All database code MUST be fully compatible with all three databases simultaneously.

**Use GORM abstractions:**
- Prefer GORM methods (`Create`, `Find`, `Where`, `Updates`, etc.) over raw SQL.
- Let GORM handle primary key generation — do not use `AUTO_INCREMENT` or `SERIAL` directly.

**When raw SQL is unavoidable:**
- Column quoting differs: PostgreSQL uses `"column"`, MySQL/SQLite uses `` `column` ``.
- Use `commonGroupCol`, `commonKeyCol` variables from `model/main.go` for reserved-word columns like `group` and `key`.
- Boolean values differ: PostgreSQL uses `true`/`false`, MySQL/SQLite uses `1`/`0`. Use `commonTrueVal`/`commonFalseVal`.
- Use `common.UsingPostgreSQL`, `common.UsingSQLite`, `common.UsingMySQL` flags to branch DB-specific logic.

**Forbidden without cross-DB fallback:**
- MySQL-only functions (e.g., `GROUP_CONCAT` without PostgreSQL `STRING_AGG` equivalent)
- PostgreSQL-only operators (e.g., `@>`, `?`, `JSONB` operators)
- `ALTER COLUMN` in SQLite (unsupported — use column-add workaround)
- Database-specific column types without fallback — use `TEXT` instead of `JSONB` for JSON storage

### Rule 3: Frontend — Prefer Bun

Use `bun` as the preferred package manager and script runner for the frontend (`web/default/` directory).

### Rule 4: New Channel StreamOptions Support

When implementing a new channel:
- Confirm whether the provider supports `StreamOptions`.
- If supported, add the channel to `streamSupportedChannels`.

### Rule 5: Upstream Relay Request DTOs — Preserve Explicit Zero Values

For request structs parsed from client JSON and re-marshaled to upstream providers:

- Optional scalar fields MUST use pointer types with `omitempty` (e.g. `*int`, `*uint`, `*float64`, `*bool`), not non-pointer scalars.
- Semantics:
  - field absent in client JSON => `nil` => omitted on marshal
  - field explicitly set to zero/false => non-`nil` pointer => must still be sent upstream

### Rule 6: Billing Expression System — Read `pkg/billingexpr/expr.md`

When working on tiered/dynamic billing (expression-based pricing), you MUST read `pkg/billingexpr/expr.md` first. It documents the design philosophy, expression language, full system architecture, token normalization rules, quota conversion, and expression versioning.