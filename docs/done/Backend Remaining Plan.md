# Backend Remaining Plan (Condensed)

Merged implementation backlog from:

- `docs/planned/Backend Recap.md` ("What's Remaining" + "Open TODOs")
- `docs/planned/Backend Audit.md` ("Open / Deferred Findings")
- Current backend source TODO scan (`backend/src`)

**Last updated**: 2026-06-10
**Status**: ✅ **Resolved** — all tracked items complete. No remaining backlog.

---

## Priority Backlog

✅ **Cleared.** The last open item — **P2 — Test Coverage** — is complete. It was scoped and delivered through the approved `docs/spec/Test Coverage Spec.md` (whose source is this plan's "P2 — Test Coverage") and `docs/done/Test Coverage Plan.md`: a DB-free unit-test suite (66 backend tests across 27 classes, 31 frontend tests) with Kover/Vitest V8 coverage reporting, all passing with no infrastructure running. See the Completed table below and `docs/CHANGELOGS.md` (2026-06-09).

---

## Completed (Previously Tracked Here)


| Item                                           | Status                | Reference                                                                                                                                                                          |
| ---------------------------------------------- | --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| P1 — Device Domain                             | **Done** (2026-05-11) | Full device lifecycle: 14 services, 4 controllers, 4 MQTT listeners, 2 entities, 5 Redis cache records, 7 core infra classes. 3 firmware codebases (transmitter ESP32, receiver ESP32, receiver ESP8266). |
| P3 — MQTT Integration                          | **Done** (2026-03-22) | MQTT v5 fully implemented: `core/mqtt/` package, publishes at 4 lifecycle points, conditional beans. See `CHANGELOGS.md` MQTT migration audit.                                     |
| P0 — Analytics Event Writes                    | **Done** (2026-03-23) | `AnalyticsEventService` writes events at all 5 lifecycle points (`TICKET_ISSUED`, `TICKET_CALLED`, `TICKET_COMPLETED`, `TICKET_CANCELLED`, `TICKET_SKIPPED`).                      |
| P0 — Analytics Query Endpoints                 | **Done** (2026-03-23) | `AnalyticsQueryService` + `AnalyticsController`: summary, peak hours, throughput, realtime stats, overview (super admin).                                                          |
| P0 — Estimated Wait Time                       | **Done** (2026-03-23) | `QueueService` populates `estimatedWaitTime` on ticket DTO using rolling avg service duration × position.                                                                          |
| P0 — Continuous Aggregates                     | **Done** (2026-03-23) | `analytics_hourly` and `analytics_daily` materialized views with TimescaleDB continuous aggregate policies.                                                                        |
| P0 — `TICKET_CANCELLED` Enum                   | **Done** (2026-03-23) | Added to `analytics_event_type` enum in `schema.sql`.                                                                                                                              |
| `RedisCounterRepository.getCurrentCount` usage | **Done** (2026-03-23) | Used in realtime stats endpoint.                                                                                                                                                   |
| `markServed`/`markCancelled` divergence        | **Done** (2026-03-23) | Diverged via `TICKET_COMPLETED` vs `TICKET_CANCELLED` event types.                                                                                                                 |
| Analytics Frontend Dashboard                   | **Done** (2026-03-23) | Store analytics page + super admin overview page. Components: `realtime-stats`, `summary-cards`, `peak-hours-chart`, `throughput-chart`, `store-ranking-table`, `period-selector`. |
| Jump-Call Feature                              | **Done** (2026-03-25) | `allowJumpCall` store field + `callSpecificTicket` endpoint + Lua script + stacked serving display.                                                                                |
| Features Implementation Plan                   | **Done** (2026-03-25) | All 7 features fully implemented (see breakdown below). Verified in full-repo audit (2026-03-25).                                                                                  |
| P2 — Remote Device Diagnostics                 | **Done** (2026-05-17) | Firmware heartbeat enrichment (heap, RSSI, uptime, dispatch counters, IP) + backend `HubDiagnosticsService` (MQTT ingest, USB relay, health summary) + frontend diagnostics panel and hub health card. See companion plans and joined audit in `CHANGELOGS.md`. |
| P2 — Test Coverage                             | **Done** (2026-06-09) | DB-free unit-test suite per approved `docs/spec/Test Coverage Spec.md` + `docs/done/Test Coverage Plan.md`: backend 27 classes / 66 tests (3 layers + JWT integration slice), `web` 22 tests, `client-web` 9 tests, Kover + Vitest V8 coverage reporting. All pass with no infra running. Integration/TestContainers, Redis Lua execution, and R2DBC `DatabaseClient` query tests are explicitly out of scope per spec §3.7. See `CHANGELOGS.md` 2026-06-09. |


### Features Implementation Plan — Breakdown

All features from `docs/planned/Features Implementation Plan.md` are now **fully implemented** across backend, admin-web, and client-web:


| #   | Feature                              | Status   | Completed                                                                                                                                                                                                                                    |
| --- | ------------------------------------ | -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | F9 Ph1 — Default counter ID          | **Done** | Frontend-only: localStorage persistence, queue page reads default, store settings tab.                                                                                                                                                       |
| 2   | F10 Ph1 — Login history              | **Done** | `login_history` table, entity/repo, `AuthController` records on login, `AdminController` paginated endpoint, cleanup scheduler (>90 days), security settings page.                                                                           |
| 3   | F9 Ph2 + F4 — Queue pausing/capping  | **Done** | `store_settings` table, `StoreSettings` entity/repo/service, queue state (ACTIVE/PAUSED) in Redis, `QueueStateToggle` component, `allowJumpCall` toggle, `maxQueueSize` enforcement, client-web paused/full banners.                         |
| 4   | F2 — Proactive FCM alerts            | **Done** | `FcmNotificationService.sendProactiveAlert()`, position-based alerts with dedup flags, `POSITION_ALERT` handler in service worker (both admin-web and client-web), bilingual notifications.                                                  |
| 5   | F9 Ph3 + F5 — No-show / grace period | **Done** | Grace period expiry keys in Redis, `handleNoShow()` with SKIP/REQUEUE actions, `maxRequeues` + `requeueOffset` settings, `markSkipped()`, `TICKET_SKIPPED` + `TICKET_REQUEUED` SSE events, serving display countdown timer + no-show button. |
| 6   | F10 Ph2 — Active sessions            | **Done** | `admin_session` table, `AdminSession` entity/repo, `SessionService`, token hash tracking, session rotation on refresh, revocation endpoint, session cleanup scheduler, security settings page with device/browser display.                   |
| 7   | F6 — Multi-counter / service types   | **Done** | `service_type` table, `ServiceType` entity/repo/service/controller, per-service-type Redis queues, cross-queue cleanup in `callNext`/`callSpecificTicket`, ticket transfer, service type selector in client-web, management UI in admin-web. |


---

## Current Backend Inventory (2026-05-11)

### Domains (5)

| Domain       | Entities                            | Services | Controllers | Storage                   |
| ------------ | ----------------------------------- | -------- | ----------- | ------------------------- |
| **Admin**    | Admin, AdminSession, LoginHistory   | 5        | 2           | PostgreSQL                |
| **Analytics**| AnalyticsEvent                      | 2        | 1           | TimescaleDB hypertable    |
| **Device**   | Device, DeviceRfCode                | 16       | 4           | PostgreSQL + Redis        |
| **Queue**    | (Redis-only)                        | 3        | 2           | Redis                     |
| **Store**    | Store, StoreSettings, ServiceType   | 2        | 2           | PostgreSQL                |

### Core Infrastructure (10 packages)

config, database, device, exception, firebase, jwt, mqtt, ratelimit, redis, security, sse

### Database (11 tables)

store, admin, device, device_rf_code, analytics_event, login_history, store_settings, admin_session, service_type, analytics_hourly (mat. view), analytics_daily (mat. view)

### Firmware (3 codebases)

- `transmitter/` — ESP32-C3, nRF24 driver, MQTT, dispatch, diagnostics, OLED, serial (39 files)
- `receiver-esp32/` — ESP32, nRF24 driver, MQTT, vibrator (23 files)
- `receiver-esp8266/` — ESP8266, nRF24 driver, MQTT, vibrator, recovery (22 files)

---

## Suggested Execution Order

✅ **Nothing remaining.** All items above are complete. Test coverage (P2) was delivered per the approved Test Coverage Spec — see the Completed table.

