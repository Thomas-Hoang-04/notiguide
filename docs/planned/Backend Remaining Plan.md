# Backend Remaining Plan (Condensed)

Merged implementation backlog from:

- `docs/planned/Backend Recap.md` ("What's Remaining" + "Open TODOs")
- `docs/planned/Backend Audit.md` ("Open / Deferred Findings")
- Current backend source TODO scan (`backend/src`)

**Last updated**: 2026-03-25

---

## Priority Backlog

### P1 — Implement Notifier Device Domain

**Why now**

- `notifier_device` table exists but domain implementation is still missing.
- Needed for device-triggered analytics and future notifications/device management.
- MQTT infrastructure is already in place (`core/mqtt/` package) — devices will consume MQTT messages.

**Implementation targets**

- Create entity/repository/service/controller for notifier devices.
- Support basic registration, activation/deactivation, and heartbeat/battery updates.
- Align emitted analytics metadata for `DEVICE_TRIGGERED` where applicable.

**Primary sources**

- Recap remaining: notifier device domain not yet implemented.
- ESP-IDF v6.0 receiver codebase at `/home/thomas/esp/v6.0/esp-idf`.

---

## Completed (Previously Tracked Here)


| Item                                           | Status                | Reference                                                                                                                                                                          |
| ---------------------------------------------- | --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
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

## Suggested Execution Order

1. Implement P1 notifier device domain (physical device management via MQTT).
2. Add test coverage (P2) around queue, analytics, estimation, and auth flows.

