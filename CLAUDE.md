# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**notiguide** is a queue management and notification system for stores. It consists of two separate repos in one workspace directory (each has its own `.git`):

- `backend/` — Kotlin/Spring Boot reactive API
- `web/` — Next.js admin dashboard (early stage, scaffolded with create-next-app)
- `docs/` — Design documents and implementation plans (not a code project)

## Backend (Kotlin/Spring Boot)

### Commands

```bash
cd backend

# Start infrastructure (PostgreSQL + Redis)
docker compose up -d

# Run the application (auto-connects to Docker Compose services)
./gradlew bootRun

# Run tests
./gradlew test

# Build
./gradlew build

# Run a single test class
./gradlew test --tests "com.thomas.notiguide.SomeTest"
```

### Tech Stack

- **Kotlin 2.3** / **Java 21** / **Spring Boot 3.5** with **WebFlux** (fully reactive, coroutine-based)
- **TimescaleDB** (PostgreSQL 17) via **R2DBC** (non-blocking database access)
- **Redis 8.6** (reactive Lettuce client, RESP3) for queue state and caching
- **RSA-512 JWT** auth (Auth0 java-jwt) + **Argon2** password hashing
- **Firebase Admin SDK** for push notifications (FCM)
- **Eclipse Paho MQTT v5** (dependency declared, not yet integrated)

### Architecture

Package structure: `com.thomas.notiguide`

- `core/` — Cross-cutting infrastructure:
  - `security/` — SecurityConfig (public: `/api/auth/**`, `/api/queue/public/**`; everything else authenticated), RSAKeyProperties, Argon2PwdEncoder
  - `jwt/` — JWTManager (issue/verify with RSA-512; verify uses public-key-only), JWTAuthFilter (CoWebFilter with catch-all error handling), JWTToPrincipal (authorities sourced from DB role, not JWT claims)
  - `redis/` — RedisConfig (StringRedisSerializer for all serializers — GenericJackson2JsonRedisSerializer must NOT be used as it wraps strings in JSON quotes breaking Lua ARGV), RedisKeyExpirationListener (cleans up expired tickets from queue/serving sets), RedisKeyManager, RedisTTLPolicy
  - `ratelimit/` — RateLimitFilter (CoWebFilter at FIRST order), RateLimiter (Redis Lua sliding-window), RateLimitProperties (strict/auth/standard tiers)
  - `config/` — AppProperties (timezone, CORS), CorsConfig (CorsConfigurationSource with exposedHeaders for rate-limit headers)
  - `database/` — R2DBCConfig
  - `firebase/` — FirebaseConfig (classpath in dev, env var path in prod; optional via @ConditionalOnBean)
  - `exception/` — Global ExceptionHandler (`@RestControllerAdvice`), HttpException, ErrorResponse
- `domain/` — Business domains, each with `entity/`, `repository/`, `service/`, `controller/`, `dto/`, `request/` subpackages:
  - `admin/` — Admin CRUD, auth (login), role-based access (SUPER_ADMIN vs ADMIN). SUPER_ADMIN has no store, ADMIN must have a store.
  - `queue/` — Redis-backed queue system. Three repositories (RedisQueueRepository, RedisTicketRepository, RedisCounterRepository), QueueService, ServingSetCleanupScheduler, QueuePublicController, QueueAdminController. Fully implemented with Lua scripts for atomic operations.
  - `store/` — Store CRUD (entity + repo + service + controller exist)
- `shared/principal/` — AdminPrincipal, AdminPrincipalAuthToken, StoreAccessUtil

### Key Patterns

- All controllers and services use Kotlin coroutines (suspend functions), not Mono/Flux
- Redis key conventions: `store:{storeId}:queue`, `store:{storeId}:serving`, `ticket:{storeId}:{ticketId}`, `store:{storeId}:counter:{date}`
- Ticket lifecycle TTLs: WAITING → 12h, CALLED → 30min, SERVED/CANCELLED → immediate deletion
- Docker Compose auto-starts via `spring-boot-docker-compose` dev dependency
- DB schema initialized from `src/main/resources/db/schema.sql` (mounted into Postgres init dir)

## Web Frontend (Next.js)

### Commands

```bash
cd web

# Dev server
yarn dev

# Build
yarn build

# Lint
yarn lint        # biome check

# Format
yarn format      # biome format --write
```

### Tech Stack

- **Next.js 16** / **React 19** / **TypeScript 5**
- **Tailwind CSS 4** (via PostCSS)
- **Biome 2.2** for linting and formatting (2-space indent, organized imports)
- **Yarn** (node-modules linker)
- **React Compiler** enabled (babel-plugin-react-compiler)

Currently scaffolded with default create-next-app structure; implementation plan is in `docs/Web Implementation Plan.md`.


## Notes:
- All file changes should be specify in `docs/CHANGELOGS.md`, even what is skipped from the implementation steps.
- Do not attempt to build anything after implementation. There is a specific audit flow for that
- Use named imports at the top of the file only. Do not use full qualified imports
- CSS should be written to seperate files if they are too complex and lengthy. Avoid lengthy Tailwind class inline
- When building the frontend, make sure to keep the pages most modularized as possible. Do not create a single large page.
- When writing code, make sure to follow the existing code style and naming conventions.
- Do not use any other UI framework than shadcn/ui.
- From then on, all frontend code should be written with bilingual support (English and Vietnamese)
- The frontend should be written in a way that is easy to understand and maintain.
- The frontend UI should be modern, coherent to current design (refer strictly to `docs/walkthrough/Web Styles.md`), with breathability.
- The physical device domain (`receiver`) has been set up using ESP-IDF v6.0. You can always refer the source code of ESP-IDF v6.0 in '/home/thomas/esp/v6.0/esp-idf'.
- Only use the UI components existing in the project. Do not create new UI components. Only crawl new components from shadcn/ui if necessary.

### Vietnamese copy rules (i18n)

`vi.json` must read like text written by a native Vietnamese speaker, not a translation of `en.json`. Treat the EN copy as the source of meaning, never the source of phrasing.

- **Don't over-qualify nouns.** Use plain `cookie` — not `cookie phiên`, not `một cookie`. Drop the determiner `một` ("a/one") unless cardinality matters; Vietnamese rarely requires it.
- **Action confirmations stay terse.** Don't add the verb back when context implies it: `Đã xong`, not `Đã bật xong`.
- **Avoid `bấm` for committing a primary action button.** Prefer `xác nhận` (e.g., `Xác nhận Cho phép`). `bấm` is acceptable for taps on icons/links.
- **Avoid `hộp xác nhận`** — sounds technical. Describe behavior instead (e.g., `trình duyệt sẽ hỏi quyền truy cập`).
- **Prefer `kích hoạt` over `bật`** for "enable/turn on" in feature/permission contexts.
- **Don't frame requests as "you change your browser, we don't change anything."** In a security/permission context users may misread it as evasive. Focus on the user's choice, scope (this site only), and reversibility instead.

**Structural mirroring (both directions):** `en.json` and `vi.json` must mirror each other structurally — same keys, same number of paragraphs (`<p>` blocks), same rich-text tags (`<bold>`, `<shield>`, etc.) in equivalent positions. When you change one side's structure, change the other immediately. Wording is per-language; structure must match.

When unsure about Vietnamese phrasing, propose the copy in plain text in chat and get approval before writing to `vi.json`. Do not iterate edits blindly.

<!-- gitnexus:start -->
# GitNexus — Code Intelligence

This project is indexed by GitNexus as **notiguide** (14505 symbols, 25760 relationships, 300 execution flows). Use the GitNexus MCP tools to understand code, assess impact, and navigate safely.

> If any GitNexus tool warns the index is stale, run `npx gitnexus analyze` in terminal first.

## Always Do

- **MUST run impact analysis before editing any symbol.** Before modifying a function, class, or method, run `gitnexus_impact({target: "symbolName", direction: "upstream"})` and report the blast radius (direct callers, affected processes, risk level) to the user.
- **MUST run `gitnexus_detect_changes()` before committing** to verify your changes only affect expected symbols and execution flows.
- **MUST warn the user** if impact analysis returns HIGH or CRITICAL risk before proceeding with edits.
- When exploring unfamiliar code, use `gitnexus_query({query: "concept"})` to find execution flows instead of grepping. It returns process-grouped results ranked by relevance.
- When you need full context on a specific symbol — callers, callees, which execution flows it participates in — use `gitnexus_context({name: "symbolName"})`.

## When Debugging

1. `gitnexus_query({query: "<error or symptom>"})` — find execution flows related to the issue
2. `gitnexus_context({name: "<suspect function>"})` — see all callers, callees, and process participation
3. `READ gitnexus://repo/notiguide/process/{processName}` — trace the full execution flow step by step
4. For regressions: `gitnexus_detect_changes({scope: "compare", base_ref: "main"})` — see what your branch changed

## When Refactoring

- **Renaming**: MUST use `gitnexus_rename({symbol_name: "old", new_name: "new", dry_run: true})` first. Review the preview — graph edits are safe, text_search edits need manual review. Then run with `dry_run: false`.
- **Extracting/Splitting**: MUST run `gitnexus_context({name: "target"})` to see all incoming/outgoing refs, then `gitnexus_impact({target: "target", direction: "upstream"})` to find all external callers before moving code.
- After any refactor: run `gitnexus_detect_changes({scope: "all"})` to verify only expected files changed.

## Never Do

- NEVER edit a function, class, or method without first running `gitnexus_impact` on it.
- NEVER ignore HIGH or CRITICAL risk warnings from impact analysis.
- NEVER rename symbols with find-and-replace — use `gitnexus_rename` which understands the call graph.
- NEVER commit changes without running `gitnexus_detect_changes()` to check affected scope.

## Tools Quick Reference

| Tool | When to use | Command |
|------|-------------|---------|
| `query` | Find code by concept | `gitnexus_query({query: "auth validation"})` |
| `context` | 360-degree view of one symbol | `gitnexus_context({name: "validateUser"})` |
| `impact` | Blast radius before editing | `gitnexus_impact({target: "X", direction: "upstream"})` |
| `detect_changes` | Pre-commit scope check | `gitnexus_detect_changes({scope: "staged"})` |
| `rename` | Safe multi-file rename | `gitnexus_rename({symbol_name: "old", new_name: "new", dry_run: true})` |
| `cypher` | Custom graph queries | `gitnexus_cypher({query: "MATCH ..."})` |

## Impact Risk Levels

| Depth | Meaning | Action |
|-------|---------|--------|
| d=1 | WILL BREAK — direct callers/importers | MUST update these |
| d=2 | LIKELY AFFECTED — indirect deps | Should test |
| d=3 | MAY NEED TESTING — transitive | Test if critical path |

## Resources

| Resource | Use for |
|----------|---------|
| `gitnexus://repo/notiguide/context` | Codebase overview, check index freshness |
| `gitnexus://repo/notiguide/clusters` | All functional areas |
| `gitnexus://repo/notiguide/processes` | All execution flows |
| `gitnexus://repo/notiguide/process/{name}` | Step-by-step execution trace |

## Self-Check Before Finishing

Before completing any code modification task, verify:
1. `gitnexus_impact` was run for all modified symbols
2. No HIGH/CRITICAL risk warnings were ignored
3. `gitnexus_detect_changes()` confirms changes match expected scope
4. All d=1 (WILL BREAK) dependents were updated

## Keeping the Index Fresh

After committing code changes, the GitNexus index becomes stale. Re-run analyze to update it:

```bash
npx gitnexus analyze
```

If the index previously included embeddings, preserve them by adding `--embeddings`:

```bash
npx gitnexus analyze --embeddings
```

To check whether embeddings exist, inspect `.gitnexus/meta.json` — the `stats.embeddings` field shows the count (0 means no embeddings). **Running analyze without `--embeddings` will delete any previously generated embeddings.**

> Claude Code users: A PostToolUse hook handles this automatically after `git commit` and `git merge`.

## CLI

| Task | Read this skill file |
|------|---------------------|
| Understand architecture / "How does X work?" | `.claude/skills/gitnexus/gitnexus-exploring/SKILL.md` |
| Blast radius / "What breaks if I change X?" | `.claude/skills/gitnexus/gitnexus-impact-analysis/SKILL.md` |
| Trace bugs / "Why is X failing?" | `.claude/skills/gitnexus/gitnexus-debugging/SKILL.md` |
| Rename / extract / split / refactor | `.claude/skills/gitnexus/gitnexus-refactoring/SKILL.md` |
| Tools, resources, schema reference | `.claude/skills/gitnexus/gitnexus-guide/SKILL.md` |
| Index, status, clean, wiki CLI commands | `.claude/skills/gitnexus/gitnexus-cli/SKILL.md` |

<!-- gitnexus:end -->
