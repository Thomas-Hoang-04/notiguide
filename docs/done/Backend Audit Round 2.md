# Backend Audit Round 2 — 2026-03-12

Full codebase status quo check. 51 source files reviewed.

---

## Current State Summary

| Domain | Status | Notes |
|---|---|---|
| Admin (auth, CRUD, roles) | **Complete** | Login, create, verify, delete, password update, paginated listing |
| Store (CRUD) | **Complete** | Create, update, soft-delete with guard checks |
| Queue (Redis) | **Complete** | Issue, call-next, serve, cancel, cleanup scheduler, public + admin endpoints |
| Notifier Device | **Not started** | Schema exists (`notifier_device` table), no Kotlin code |
| Analytics | **Not started** | Schema exists (`analytics_event` hypertable), TODO comments throughout |
| MQTT Integration | **Not started** | Paho v5 dependency declared, no code |
| Rate Limiting | **Not started** | Config property exists (`rate-limit.enabled`), no implementation |
| Test Suite | **Missing** | Only placeholder `contextLoads()` test |

---

## Findings

### P0 — Critical

| # | Finding | File(s) | Detail |
|---|---|---|---|
| 1 | ~~**Store deletion leaves store inactive on failure**~~ | `StoreService.kt` | ✅ FIXED — Moved queue ticket check before deactivation. Failed deletes no longer leave store in broken `isActive=false` state. |

### P1 — High

| # | Finding | File(s) | Detail |
|---|---|---|---|
| 2 | **No rate limiting on public queue endpoints** | `QueuePublicController.kt` | `/api/queue/public/{storeId}/tickets` (POST) has no auth and no rate limit. A single client can issue unlimited tickets, filling Redis. Config key `rate-limit.enabled` exists but no implementation. _(Previously deferred — still open)_ |
| 3 | ~~**FirebaseConfig fails hard in prod without credentials**~~ | `FirebaseConfig.kt` | ✅ FIXED — `firebaseApp()` now returns `null` with warning log if credentials missing. `firebaseMessaging` bean gated with `@ConditionalOnBean(FirebaseApp::class)`. |
| 4 | ~~**RedisKeyExpirationListener silently swallows startup errors**~~ | `RedisKeyExpirationListener.kt` | ✅ FIXED — Added retry with exponential backoff (5 attempts: 1s, 2s, 5s, 10s, 30s). Logs escalate from WARN to ERROR on final failure. Exposed `isListening` flag for health observability. Subscription errors reset state so retries get a clean container. |

### P2 — Medium

| # | Finding | File(s) | Detail |
|---|---|---|---|
| 5 | ~~**No pagination on store listing**~~ | `StoreRepository.kt`, `StoreController.kt`, `ServingSetCleanupScheduler.kt` | ✅ FIXED — Added `GET /api/stores?page&size` with paginated query. Cleanup scheduler now uses `findAllActive()` to skip inactive stores. |
| 6 | ~~**Weak password policy**~~ | `CreateAdminRequest.kt`, `UpdatePasswordRequest.kt` | ✅ FIXED — Added `@Pattern` requiring at least one uppercase, one lowercase, and one digit. |
| 7 | **Timestamp format inconsistency across layers** | Multiple files | Entities use `OffsetDateTime` (timezone-aware), Redis stores epoch millis as strings, DTOs use `Instant` (UTC). Functionally correct but confusing for maintainers — three different temporal representations for the same data. |
| 8 | **Validation logic duplicated** | Request classes + Service classes | `@NotBlank`/`@Size` annotations on requests AND `require()` calls in services for the same fields. _(Previously accepted as defense-in-depth — noting for awareness)_ |

### P3 — Low / Informational

| # | Finding | File(s) | Detail |
|---|---|---|---|
| 9 | **AdminPrincipal hardcodes account status methods** | `AdminPrincipal.kt:21-23` | `isAccountNonExpired()`, `isAccountNonLocked()`, `isCredentialsNonExpired()` all return `true`. Fine for now but worth noting if account lockout is ever needed. |
| 10 | **UpdateStoreRequest uses mutable `var` properties** | `UpdateStoreRequest.kt` | Mutable with custom `@JsonSetter` for `addressProvided` tracking. Functional but idiomatic Kotlin would prefer immutable approach. _(Previously accepted — noting for awareness)_ |
| 11 | **Firebase credentials in classpath for dev** | `firebase/notiguide-firebase.json` | Lives in `src/main/resources`. Should verify it's in `.gitignore` to avoid leaking credentials in version control. |
| 12 | **No service-to-service auth pattern** | Architecture-wide | If internal services are added (analytics, notifications), there's no internal auth mechanism. Future concern only. |
| 13 | **`callNext` Lua script hardcodes iteration limit** | `QueueService.kt:68` | `local maxIterations = 100` inside Lua script is a magic number not tied to the Kotlin-side `CALL_NEXT_MAX_RETRIES = 10`. Should be a named constant or passed as ARGV. |
| 14 | **MQTT dependency unused** | `build.gradle.kts` | Eclipse Paho MQTT v5 dependency declared but no integration code exists. Dead dependency adds build size and potential attack surface. |

---

## Retracted Findings (from initial draft)

| Original # | Claim | Reason Retracted |
|---|---|---|
| 1 | "Prod Redis config missing host" | **False.** `application-prod.yaml` line 14 sets `host: redis`. |
| 5 | "RedisKeyExpirationListener uses Dispatchers.Default" | **False.** Line 26 already uses `Dispatchers.IO`. |
| 8 | "R2DBC URL parsing uses string operations" | **Partially false.** `R2DBCConfig.kt` line 62 uses `URI()` constructor, not raw string splitting. Manual but not fragile. |
| 9 | "CLAUDE.md says queue not yet built" | **Already fixed** in this audit session. |
| 10 | "Security path inconsistency in CLAUDE.md" | **Already fixed** in this audit session. |

---

## Actionable Recommendations (Priority Order)

1. ~~**Fix store deletion atomicity**~~ — ✅ Done
2. **Implement rate limiting** — At minimum per-IP throttle on public ticket issuance
3. ~~**Make Firebase optional**~~ — ✅ Done
4. ~~**Harden RedisKeyExpirationListener**~~ — ✅ Done
5. ~~**Add store pagination**~~ — ✅ Done
6. **Remove or integrate MQTT dep** — Dead dependencies are tech debt
7. **Add tests** — Still zero meaningful tests _(deferred multiple times — growing risk)_

---

## Files Reviewed (51 total)

All `.kt` files under `src/main/kotlin/com/thomas/notiguide/`, plus:
- `build.gradle.kts`, `settings.gradle.kts`
- `compose.yaml`
- `application.yaml`, `application-dev.yaml`, `application-prod.yaml`
- `db/schema.sql`, `redis/redis.conf`
- `src/test/kotlin/.../NotiguideApplicationTests.kt`
