# Backend Audit — Bugs, Logic Loopholes & Oversights

Comprehensive audit of all backend Kotlin source files. Issues are grouped by severity.

---

## Fixed Findings

| # | Severity | Finding | File(s) |
|---|----------|---------|---------|
| 1 | P0 | `callNextUntilSuccess` can loop infinitely | `QueueService.kt` |
| 2 | P0 | Admin creation does not validate `storeId` exists | `AdminService.kt` |
| 4 | P0 | Counter expiry race at midnight boundary | `RedisCounterRepository.kt` |
| 5 | P1 | Store deletion race condition (accepted limitation) | `StoreService.kt` |
| 6 | P1 | Ticket TTL expiry creates silent orphans in serving set | `RedisTTLPolicy.kt`, `RedisKeyExpirationListener.kt` |
| 7 | P1 | `UpdateStoreRequest` allows setting name to empty | `StoreService.kt` |
| 8 | P1 | `UpdateStoreRequest` cannot distinguish "not provided" from "set to null" | `StoreService.kt`, `UpdateStoreRequest.kt` |
| 9 | P1 | No `@Valid` on `CreateStoreRequest` in controller | `StoreController.kt`, `CreateStoreRequest.kt` |
| 10 | P1 | `JWTToPrincipal` silently returns null for deleted admins | `JWTToPrincipal.kt` |
| 11 | P1 | Public queue rate limiting implemented (deferred finding closed) | `RateLimitFilter.kt`, `RateLimiter.kt`, `SecurityConfig.kt`, `application-prod.yaml` |
| 12 | P2 | Ticket status is a raw string, not an enum | `TicketDto.kt`, `RedisTicketRepository.kt`, `QueueService.kt` |
| 13 | P2 | Hardcoded timezone in `RedisCounterRepository` | `RedisCounterRepository.kt`, `R2DBCConfig.kt` |
| 14 | P2 | No logging in queue operations | `QueueService.kt` |
| 15 | P2 | No idempotency on serve/cancel | `QueueService.kt` |
| 16 | P2 | No pagination on admin list | `AdminService.kt`, `AdminController.kt` |
| 17 | P2 | SecurityConfig public path mismatch with documentation | `SecurityConfig.kt` |
| 18 | P2 | Exception handler exposes internal class names | `ExceptionHandler.kt` |
| 20 | P3 | Timestamps use epoch seconds, not milliseconds | `RedisTicketRepository.kt` |
| 22 | P3 | `createdBy` not set on admin creation | `AdminService.kt`, `AdminController.kt` |

---

## Open / Deferred Findings

### 3. No Analytics Events Emitted Anywhere *(DEFERRED — analytics is planned for a later sprint)*
**Severity**: P0
**File**: `QueueService.kt:218, 242` (TODO comments)

The `analytics_event` table exists in the schema with a TimescaleDB hypertable, but **no code writes to it**. `serveTicket()` and `cancelTicket()` have TODO comments about emitting analytics events with `wait_duration_seconds`, but the service layer already reads the ticket data (including `issued_at` and `called_at`) before deletion — the data is available but discarded.

This means:
- No historical data accumulates for Feature 1 (Wait Time Estimation) or Feature 7 (Analytics Dashboards)
- The TimescaleDB hypertable sits empty indefinitely

> **Note**: Deferred to the analytics sprint. Revisit when analytics features are prioritized.

---

### 21. `markServed` and `markCancelled` Are Identical *(KEPT AS-IS)*
**Severity**: P3
**File**: `RedisTicketRepository.kt:14-20`

Both methods just `DEL` the ticket key. The distinction is semantic only — no different Redis operation. The TODO comments explain why (analytics should be captured first), but until analytics is implemented, these could be a single method.

> **Decision**: Keep the separate methods. The semantic distinction is intentional and will diverge once analytics events are emitted per status (deferred to analytics sprint).
