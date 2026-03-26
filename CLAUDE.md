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
