# Analytics Implementation Plan

## Overview

The analytics module captures queue lifecycle events, stores them in TimescaleDB, and surfaces actionable insights through both backend API endpoints and frontend dashboard pages. This is the **P0 backend gap** identified in `docs/planned/Backend Remaining Plan.md` — the `analytics_event` table and `analytics_event_type` enum already exist in the schema but no events are written or queried.

### What This Plan Covers

1. **Backend**: Analytics event write path, query endpoints, estimated wait time, continuous aggregates
2. **Admin Web**: Analytics dashboard pages (store-level + cross-store overview)
3. **Client Web**: Estimated wait time display (consuming existing `estimatedWaitTime` field)

### What This Plan Does NOT Cover

- Multi-counter / service type routing (Feature 6) — analytics will be enriched when that ships
- No-show rate tracking (Feature 5) — `TICKET_SKIPPED` events are reserved for no-show/grace-period scenarios but not emitted until Feature 5 is implemented. Voluntary cancellations emit `TICKET_CANCELLED` instead.
- Notifier device analytics (`DEVICE_TRIGGERED`) — deferred until the notifier device domain exists (P2)

---

## Existing Schema (Already in `schema.sql`)

```sql
CREATE TYPE analytics_event_type AS ENUM (
    'TICKET_ISSUED',
    'TICKET_CALLED',
    'TICKET_COMPLETED',
    'TICKET_SKIPPED',
    'DEVICE_TRIGGERED'
);

CREATE TABLE analytics_event (
    time           TIMESTAMP WITH TIME ZONE NOT NULL,
    store_id       UUID REFERENCES store(id) ON DELETE SET NULL,
    event_type     analytics_event_type NOT NULL,
    ticket_id      UUID,
    wait_duration_seconds INT,
    device_id      UUID REFERENCES notifier_device(id) ON DELETE SET NULL,
    metadata       JSONB
);

SELECT create_hypertable('analytics_event', 'time');

CREATE INDEX idx_analytics_store_time ON analytics_event(store_id, time DESC);
CREATE INDEX idx_analytics_event_type ON analytics_event(event_type, time DESC);
```

### Schema Notes

- `wait_duration_seconds`: For `TICKET_CALLED`, this is the time from issued → called (queue wait). For `TICKET_COMPLETED`, this is the time from called → completed (service duration). For `TICKET_SKIPPED`, this is the time from issued → skipped (total time in system).
- `metadata`: JSONB for extensible fields — `counter_id`, `ticket_number`, `previous_status` (for skipped/cancelled tickets that were in WAITING vs CALLED state).
- `device_id`: Only populated for `DEVICE_TRIGGERED` events (future use).
- TimescaleDB hypertable enables efficient time-range queries and continuous aggregates.

---

## Phase 1: Backend — Analytics Event Write Path

### 1.1 Domain Structure

Create the analytics domain under `domain/analytics/`:

```
domain/analytics/
├── entity/
│   └── AnalyticsEvent.kt
├── repository/
│   └── AnalyticsEventRepository.kt
└── service/
    └── AnalyticsEventService.kt
```

No controller in this phase — the write path is internal only (called from `QueueService`).

### 1.2 Entity

```kotlin
// AnalyticsEvent.kt
data class AnalyticsEvent(
    val time: OffsetDateTime,
    val storeId: UUID,
    val eventType: AnalyticsEventType,
    val ticketId: UUID?,
    val waitDurationSeconds: Int?,
    val deviceId: UUID?,
    val metadata: String?  // JSON string
)

enum class AnalyticsEventType {
    TICKET_ISSUED,
    TICKET_CALLED,
    TICKET_COMPLETED,
    TICKET_CANCELLED,
    TICKET_SKIPPED,
    DEVICE_TRIGGERED
}
```

**Schema migration required**: Add `'TICKET_CANCELLED'` to the `analytics_event_type` enum in `schema.sql`:

```sql
ALTER TYPE analytics_event_type ADD VALUE 'TICKET_CANCELLED';
```

**Event type semantics:**
- `TICKET_CANCELLED` — voluntary cancellation by customer or admin (used NOW)
- `TICKET_SKIPPED` — no-show/grace-period expiry (reserved for Feature 5)

This distinction is important: `skipRate` (no-show rate) must only count `TICKET_SKIPPED`, not voluntary cancellations. Conflating the two would make the metric meaningless when Feature 5 ships. The `cancelRate` (voluntary) is a separate metric: `CANCELLED / ISSUED`.

### 1.3 Repository

Use R2DBC `DatabaseClient` for raw INSERT (not Spring Data R2DBC `ReactiveCrudRepository`) since `analytics_event` has no primary key — it's a TimescaleDB hypertable optimized for append-only writes.

```kotlin
// AnalyticsEventRepository.kt
@Repository
class AnalyticsEventRepository(private val client: DatabaseClient) {

    suspend fun insert(event: AnalyticsEvent) { ... }

    // Batch insert for future use (e.g., backfill)
    suspend fun insertBatch(events: List<AnalyticsEvent>) { ... }
}
```

### 1.4 Service

```kotlin
// AnalyticsEventService.kt
@Service
class AnalyticsEventService(private val repository: AnalyticsEventRepository) {

    suspend fun recordTicketIssued(storeId: UUID, ticketId: UUID, ticketNumber: String)
    suspend fun recordTicketCalled(storeId: UUID, ticketId: UUID, ticketNumber: String, counterId: String?, issuedAt: Instant)
    suspend fun recordTicketCompleted(storeId: UUID, ticketId: UUID, ticketNumber: String, calledAt: Instant?)
    suspend fun recordTicketCancelled(storeId: UUID, ticketId: UUID, ticketNumber: String, issuedAt: Instant?, previousStatus: TicketStatus)
    // recordTicketSkipped reserved for Feature 5 (no-show/grace period expiry)
}
```

**Duration calculations:**
- `TICKET_CALLED`: `wait_duration_seconds` = `calledAt - issuedAt` (how long the customer waited in queue). Use the `calledAt` timestamp that was passed to the Lua script (available in the ticket hash after the script completes), NOT a separate `Instant.now()` call, to avoid clock drift between the Lua script execution and the analytics write.
- `TICKET_COMPLETED`: `wait_duration_seconds` = `now() - calledAt` (how long the service took)
- `TICKET_CANCELLED`: `wait_duration_seconds` = `now() - issuedAt` (total time in system before voluntary cancel)

**Metadata captured:**
- `counter_id` (on CALLED and COMPLETED, if available)
- `ticket_number` (on all events)
- `previous_status` (on CANCELLED — distinguishes cancelled-while-waiting from cancelled-while-called)

### 1.5 Integration into QueueService

Modify `QueueService` to inject `AnalyticsEventService` (optional via `= null` like `MqttPublisher`) and emit events at each lifecycle point:

| Lifecycle Point | Method | Event Type | Duration Source |
|---|---|---|---|
| Ticket issued | `issueTicket()` | `TICKET_ISSUED` | — |
| Ticket called | `callNext()` (success branch) | `TICKET_CALLED` | `issuedAt` from ticket hash |
| Ticket served | `serveTicket()` | `TICKET_COMPLETED` | `calledAt` from ticket hash (read BEFORE `markServed` deletes the key) |
| Ticket cancelled | `cancelTicket()` | `TICKET_CANCELLED` | `issuedAt` from ticket hash (read BEFORE `markCancelled` deletes the key) |

**Critical ordering**: The ticket hash data (`issued_at`, `called_at`) must be read BEFORE `markServed`/`markCancelled` deletes the Redis key. The current code already does this — `getTicket()` is called at the top of both methods. The TODOs in `RedisTicketRepository` confirm this requirement.

**Fire-and-forget vs. inline**: Analytics writes should be **inline** (awaited), not fire-and-forget. Rationale:
- R2DBC INSERT is non-blocking and fast (~1-2ms for a single row)
- Lost analytics events are harder to diagnose than slightly slower queue operations
- If the DB is down, the write fails visibly rather than silently dropping events
- Queue operations are not high-frequency (admin-initiated, not customer-burst)

**Idempotent no-op paths**: When `serveTicket()` or `cancelTicket()` hit the idempotent no-op path (ticket hash is empty), do NOT emit an analytics event. The ticket was already handled — emitting would create duplicates.

### 1.6 Resolving Existing TODOs

After this phase, the following TODOs are resolved:
- `QueueService.kt:264` — `// TODO: emit analytics event with wait_duration_seconds` (serve)
- `QueueService.kt:304` — `// TODO: emit analytics event with wait_duration_seconds` (cancel)
- `RedisTicketRepository.kt:14` — `// TODO: Service layer must call getTicket() BEFORE this...` (markServed)
- `RedisTicketRepository.kt:18` — `// TODO: Service layer must call getTicket() BEFORE this...` (markCancelled)

The TODO comments should be removed after implementation. The `RedisTicketRepository` TODO comments are informational — the current code already follows this pattern correctly.

---

## Phase 2: Backend — Analytics Query Endpoints

### 2.1 Domain Structure (additions)

```
domain/analytics/
├── controller/
│   └── AnalyticsController.kt
├── dto/
│   ├── RealtimeStatsResponse.kt
│   ├── StoreSummaryResponse.kt
│   ├── PeakHoursResponse.kt
│   ├── DailyThroughputResponse.kt
│   └── OverviewResponse.kt
├── entity/
│   └── AnalyticsEvent.kt
├── repository/
│   └── AnalyticsEventRepository.kt
└── service/
    ├── AnalyticsEventService.kt      (Phase 1 — write path)
    └── AnalyticsQueryService.kt      (Phase 2 — read path)
```

### 2.2 Endpoints

All analytics endpoints require authentication. Access control follows the existing pattern: `StoreAccessUtil.requireStoreAccess()` for store-scoped endpoints, `ROLE_SUPER_ADMIN` check for cross-store overview.

#### Store-Scoped Endpoints (Admin + Super Admin)

| Method | Path | Description | Response |
|---|---|---|---|
| `GET` | `/api/analytics/{storeId}/realtime` | Live Redis snapshot | `RealtimeStatsResponse` |
| `GET` | `/api/analytics/{storeId}/summary?period={period}` | Aggregated stats | `StoreSummaryResponse` |
| `GET` | `/api/analytics/{storeId}/peak-hours?range={range}` | Hourly distribution | `PeakHoursResponse` |
| `GET` | `/api/analytics/{storeId}/throughput?range={range}` | Daily throughput | `DailyThroughputResponse` |

#### Cross-Store Endpoints (Super Admin Only)

| Method | Path | Description | Response |
|---|---|---|---|
| `GET` | `/api/analytics/overview/realtime` | Cross-store live Redis snapshot | `OverviewRealtimeResponse` |
| `GET` | `/api/analytics/overview?period={period}` | Cross-store summary | `OverviewResponse` |

#### Query Parameters

- `period`: `today`, `week`, `month`, `quarter` — controls the time range for aggregation
  - `today`: midnight local time → now
  - `week`: 7 days ago → now
  - `month`: 30 days ago → now
  - `quarter`: 90 days ago → now
- `range`: `7d`, `30d`, `90d` — controls the lookback for distribution/throughput queries

### 2.3 Response DTOs

```kotlin
// RealtimeStatsResponse.kt — sourced from Redis, no DB query
// "Active counters" metric (Feature 7 ideas) omitted — requires Feature 6 (Multi-Counter)
data class RealtimeStatsResponse(
    val currentQueueSize: Long,
    val currentServingCount: Long,
    val ticketsIssuedToday: Long,
    val estimatedAvgWaitMinutes: Double?
)

// StoreSummaryResponse.kt — sourced from TimescaleDB
data class StoreSummaryResponse(
    val period: String,
    val totalIssued: Long,
    val totalCompleted: Long,
    val totalCancelled: Long,
    val totalSkipped: Long,               // no-show skips only (Feature 5, 0 until then)
    val avgWaitSeconds: Double?,           // avg queue wait (issued → called)
    val avgServiceSeconds: Double?,        // avg service duration (called → completed)
    val medianWaitSeconds: Double?,        // median queue wait (computed from raw data, not continuous aggregates)
    val peakHour: Int?,                    // hour of day (0-23) with most tickets issued
    val cancelRate: Double?,              // cancelled / issued as percentage
    val skipRate: Double?                 // skipped / issued as percentage (0 until Feature 5)
)

// PeakHoursResponse.kt
data class PeakHoursResponse(
    val range: String,
    val hours: List<HourlyCount>
)
data class HourlyCount(
    val hour: Int,        // 0-23
    val avgTickets: Double // average tickets issued per day at this hour
)

// DailyThroughputResponse.kt
data class DailyThroughputResponse(
    val range: String,
    val days: List<DailyCount>
)
data class DailyCount(
    val date: LocalDate,
    val issued: Long,
    val completed: Long,
    val cancelled: Long,
    val skipped: Long
)

// OverviewRealtimeResponse.kt — Super Admin realtime snapshot (sourced from Redis, no DB query)
// Aggregates across all active stores by iterating RedisQueueRepository/RedisCounterRepository
data class OverviewRealtimeResponse(
    val activeStores: Long,
    val totalQueueSize: Long,
    val totalServingCount: Long,
    val totalIssuedToday: Long,
    val estimatedAvgWaitMinutes: Double?
)

// OverviewResponse.kt — Super Admin cross-store view
data class OverviewResponse(
    val period: String,
    val totalStores: Long,
    val totalIssued: Long,
    val totalCompleted: Long,
    val avgWaitSeconds: Double?,
    val stores: List<StoreAnalyticsSummary>
)
data class StoreAnalyticsSummary(
    val storeId: UUID,
    val storeName: String,
    val issued: Long,
    val completed: Long,
    val cancelled: Long,
    val skipped: Long,
    val avgWaitSeconds: Double?
)
```

### 2.4 Repository Query Methods

Add query methods to `AnalyticsEventRepository` using R2DBC `DatabaseClient` with parameterized queries:

```kotlin
suspend fun getSummary(storeId: UUID, from: OffsetDateTime, to: OffsetDateTime): StoreSummaryRow
suspend fun getHourlyDistribution(storeId: UUID, from: OffsetDateTime, to: OffsetDateTime): List<HourlyCountRow>
suspend fun getDailyThroughput(storeId: UUID, from: OffsetDateTime, to: OffsetDateTime): List<DailyCountRow>
suspend fun getOverview(from: OffsetDateTime, to: OffsetDateTime): List<StoreOverviewRow>
```

**TimescaleDB-specific queries:**
- Use `time_bucket('1 hour', time)` for hourly grouping
- Use `time_bucket('1 day', time)` for daily grouping
- **Median calculation**: `percentile_cont(0.5) WITHIN GROUP (ORDER BY wait_duration_seconds)` — computed at query time against the raw `analytics_event` table, NOT from continuous aggregates (TimescaleDB does not support ordered-set aggregates in continuous aggregates)
- All time calculations respect the store timezone configured in `AppProperties.timezone`
- **Per-counter metrics**: `counter_id` is stored in `metadata` JSONB. Counter-grouped queries use `metadata->>'counter_id'` — consider a GIN index on `metadata` if per-counter queries become frequent

### 2.5 Security Configuration

Add `/api/analytics/**` to the authenticated routes in `SecurityConfig`. These are NOT public endpoints — they require a valid JWT.

---

## Phase 3: Backend — Estimated Wait Time

This resolves the P1 item from `Backend Remaining Plan.md` and the TODO in `TicketDto.kt:24`.

### 3.1 Estimation Strategy

```
estimated_wait = tickets_ahead × avg_service_duration
```

Where:
- `tickets_ahead` = current position in queue (from Redis `ZRANK`)
- `avg_service_duration` = rolling average of `wait_duration_seconds` from the last 20 `TICKET_COMPLETED` events for this store (from TimescaleDB)

### 3.2 Refinements (from Feature 1 in Features Ideas)

- **Trimmed mean**: Exclude the top and bottom 10% of service durations before averaging, to prevent outliers (e.g., a 45-minute complex case) from skewing the estimate
- **Cold start fallback**: If fewer than 5 completed tickets exist for the store, return `null` (the frontend already handles this as "Estimating...")
- **Caching**: Cache the computed `avg_service_duration` per store in Redis (`store:{storeId}:avg_service_seconds`) with a 5-minute TTL. Recalculate on cache miss. This avoids hitting TimescaleDB on every ticket status check.

### 3.3 Implementation

Add to `AnalyticsQueryService`:

```kotlin
suspend fun getEstimatedWaitMinutes(storeId: UUID, positionInQueue: Long): Long?
```

Returns **minutes** (rounded, minimum 1). The internal calculation uses seconds, but the return value is converted to minutes because:
- The client-web `TicketCard` already displays `estimatedWaitTime` as minutes (`${value} min`)
- Minute-level granularity is appropriate for customer-facing estimates
- The existing `TicketStatusResponse.estimatedWaitTime` field semantics become "estimated wait in minutes"

Modify `QueueService.getTicketStatus()` to call `getEstimatedWaitMinutes()` and populate the `estimatedWaitTime` field in `TicketStatusResponse`.

### 3.4 New Redis Key

Add to `RedisKeyManager`:
```kotlin
fun avgServiceDuration(storeId: UUID) = "store:$storeId:avg_service_seconds"
```

TTL: 5 minutes (via `RedisTTLPolicy`).

### 3.5 Resolving Existing TODO

After this phase, the following TODO is resolved:
- `TicketDto.kt:24` — `// TODO: populate from analytics domain when average service duration data is available`

---

## Phase 4: Backend — TimescaleDB Continuous Aggregates

Continuous aggregates pre-compute rollups so dashboard queries are instant even over months of data.

### 4.1 Schema Additions (append to `schema.sql`)

```sql
-- Hourly rollup: ticket counts and average wait times per store per hour
-- Note: PERCENTILE_CONT (ordered-set aggregate) is NOT supported in TimescaleDB
-- continuous aggregates. Median is computed at query time from raw data instead.
CREATE MATERIALIZED VIEW analytics_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    store_id,
    event_type,
    COUNT(*) AS event_count,
    AVG(wait_duration_seconds) FILTER (WHERE wait_duration_seconds IS NOT NULL) AS avg_wait_seconds
FROM analytics_event
GROUP BY bucket, store_id, event_type
WITH NO DATA;

-- Refresh policy: refresh the last 3 hours every 30 minutes
SELECT add_continuous_aggregate_policy('analytics_hourly',
    start_offset => INTERVAL '3 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '30 minutes'
);

-- Daily rollup: same metrics at day granularity
CREATE MATERIALIZED VIEW analytics_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS bucket,
    store_id,
    event_type,
    COUNT(*) AS event_count,
    AVG(wait_duration_seconds) FILTER (WHERE wait_duration_seconds IS NOT NULL) AS avg_wait_seconds
FROM analytics_event
GROUP BY bucket, store_id, event_type
WITH NO DATA;

SELECT add_continuous_aggregate_policy('analytics_daily',
    start_offset => INTERVAL '3 days',
    end_offset => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 hour'
);

-- Data retention: drop raw events older than 90 days, keep rollups indefinitely
SELECT add_retention_policy('analytics_event', INTERVAL '90 days');
```

### 4.2 Query Strategy

| Query Type | Time Range | Source |
|---|---|---|
| Realtime stats | Now | Redis |
| Today's summary | Midnight → now | Raw `analytics_event` table (today's data isn't in the continuous aggregate yet) |
| Weekly summary | 7 days | `analytics_hourly` view + raw tail |
| Monthly/quarterly | 30-90 days | `analytics_daily` view + raw tail |
| Peak hours | 7-90 days | `analytics_hourly` view + raw tail |
| Daily throughput | 7-90 days | `analytics_daily` view + raw tail |

**Refresh gap handling**: Continuous aggregates have an `end_offset` — the most recent 1 hour (hourly view) or 1 day (daily view) isn't materialized. Queries that need up-to-the-minute accuracy must UNION the continuous aggregate with raw `analytics_event` data for the trailing gap. The repository layer handles this transparently — each query method fetches from the aggregate for the bulk of the range, then supplements with a raw query for the unmaterialized tail. This is transparent to the controller.

---

## Phase 5: Admin Web — Analytics Dashboard

### 5.0 Two-Tier Architecture: Overview + Analytics

**Rationale**: The current empty Overview page redirects to Queue. This plan replaces it with a lightweight analytics summary, and creates a separate detailed Analytics section:

- **Overview** (`/dashboard/page.tsx`): Minimal realtime metrics (4 cards) for quick dashboard context
  - Admin: Store-level snapshot (Queue Now, Serving Now, Issued Today, Avg Wait)
  - Super Admin: Cross-store aggregated snapshot + Active Stores count

- **Analytics** (`/dashboard/analytics/page.tsx`): Full-featured dashboard with charts, detailed summaries, period selection
  - Admin: Store-level deep dive (peak hours chart, daily throughput, summary stats)
  - Super Admin: Cross-store overview with store rankings table and drill-down links

**Benefits**:
- Faster dashboard load (Overview fetches only 4 realtime metrics)
- Clear information hierarchy (get-at-a-glance vs. deep analysis)
- Sidebar navigation groups naturally (Overview → Queue → Analytics → Stores)

### 5.1 Overview Page Changes (Replaces Current Empty Dashboard)

**File**: `src/app/[locale]/dashboard/page.tsx`

Instead of redirecting to `/dashboard/queue`, this page now displays:

#### 5.1.1 Admin View (auto-scoped to assigned store)

```
Page Title: "{Store Name}" (fetched from auth context)

┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Queue Now    │ │ Serving Now  │ │ Issued Today │ │ Avg Wait     │
│     12       │ │      3       │ │     87       │ │   ~8 min     │
└──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘

[Optional: Mini sparklines showing 7-day trend]

[View full analytics →]  link to /dashboard/analytics
```

#### 5.1.2 Super Admin View (cross-store aggregated)

```
Page Title: "Dashboard"

┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Active Stores│ │ Total Issued │ │ Total Done   │ │ Avg Wait     │
│      5       │ │    1,247     │ │   1,130      │ │   ~9 min     │
└──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘

[Optional: Mini sparklines showing 7-day trend across all stores]

[View full analytics →]  link to /dashboard/analytics
```

**Components**: Reuse `realtime-stats.tsx` (from Phase 5.3 module structure) with conditional layout based on role.

**Polling**: Use the existing `POLLING_INTERVAL` constant (10 seconds, defined in `src/lib/constants.ts`) for realtime stats only. No charts or period-based queries on the Overview page.

**Page gradient**: Set `bg-gradient-page` (the default cool gradient) via `useLayoutStore.setPageGradientClass()` — the warm gradient is reserved for the Queue page.

**Glass styling**: Realtime stats cards use `glass-card` (no context variant — neutral analytics context, distinct from Queue's `glass-context-action`).

### 5.2 Analytics Route Structure

```
src/app/[locale]/dashboard/
├── page.tsx                  # MODIFIED: Overview (realtime cards, replaces old redirect)
├── analytics/
│   ├── page.tsx              # Analytics detail: store-scoped (Admin) / overview (Super Admin)
│   └── [storeId]/
│       └── page.tsx          # Store detail analytics (Super Admin drill-down)
```

### 5.3 Sidebar Addition

Update the sidebar `navItems` array in `src/components/layout/sidebar.tsx`:

Add "Analytics" nav item. Icon: `BarChart3` from Lucide. Visible to **all authenticated roles** (no `superAdminOnly` or `adminOnly` flag) — both roles need deep analytics, just with different views (store-level vs overview).

Updated sidebar order:
```
- Dashboard (all)             ← Overview (realtime cards, NO Queue management)
- Queue (adminOnly)           ← Queue management operations
- Analytics (all)             ← Deep analytics (NEW)
- Stores (superAdminOnly)
- Admins (all)
- Settings (all)
```

**Implementation**:
```typescript
// 1. Update NavItem label union type to include "analytics":
//    label: "overview" | "queue" | "analytics" | "stores" | "admins" | "settings"

// 2. Add to navItems array (after Queue, before Stores):
{
  href: "/dashboard/analytics",
  label: "analytics",
  icon: <BarChart3 aria-hidden="true" className="size-5" />,
}
```

Add translation key `"analytics": "Analytics"` / `"Phân tích"` to `navigation` object in both `en.json` and `vi.json`.

The Analytics nav item links to `/dashboard/analytics`. The page renders conditionally:
- **Admin**: Auto-scoped to their assigned store (same as queue page)
- **Super Admin**: Shows the cross-store overview with drill-down to individual stores

### 5.4 Feature Module Structure (Analytics Pages)

```
src/features/analytics/
├── api.ts                    # API client functions
├── types.ts                  # Response DTOs
├── analytics-overview.tsx    # Super Admin: cross-store summary
├── store-analytics.tsx       # Single-store analytics view
├── realtime-stats.tsx        # Live Redis stats cards
├── summary-cards.tsx         # Period summary (issued, completed, avg wait, etc.)
├── peak-hours-chart.tsx      # Hourly distribution bar chart
├── throughput-chart.tsx      # Daily throughput line chart
├── period-selector.tsx       # today / week / month / quarter toggle
└── store-ranking-table.tsx   # Super Admin: stores ranked by metrics
```

### 5.5 Page Layout — Analytics Detail (Admin / Super Admin drill-down)

**Page gradient**: Use default `bg-gradient-page` (no warm override — analytics is informational, not action-oriented).

**Glass styling**:
- Realtime stats cards: `glass-card` (neutral context)
- Summary panel: `glass-card` with rounded-xl
- Chart panels: `glass-card` with rounded-xl
- Period selector: `glass-panel` (lighter frosted effect for toolbar-like controls)
- Store ranking table: `glass-card` with rounded-xl

```
Page: "Analytics — {Store Name}"

┌─────────────────────────────────────────────────────────────────┐
│ Period Selector: [Today] [This Week] [This Month] [Quarter]    │
└─────────────────────────────────────────────────────────────────┘

┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Queue Now    │ │ Serving Now  │ │ Issued Today │ │ Avg Wait     │
│     12       │ │      3       │ │     87       │ │   ~8 min     │
└──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Summary                                                         │
│ Total Issued: 342 │ Completed: 310 │ Cancelled: 32             │
│ Avg Queue Wait: 7m 42s │ Avg Service Time: 4m 15s              │
│ Median Wait: 6m 30s │ Cancel Rate: 9.4%                        │
│ Peak Hour: 11:00 AM                                             │
└─────────────────────────────────────────────────────────────────┘

┌────────────────────────────────┐ ┌──────────────────────────────┐
│ Peak Hours                     │ │ Daily Throughput              │
│ (bar chart, 24 hours)          │ │ (line chart, issued/completed)│
│                                │ │                              │
│  ██                            │ │         ╱\                   │
│  ██  ██                        │ │    ╱\  ╱  \                  │
│  ██  ██  ██                    │ │   ╱  ╲╱    \                 │
│  ██  ██  ██  ██                │ │  ╱          \                │
│  6  9  12  15  18              │ │ Mon Tue Wed Thu Fri          │
└────────────────────────────────┘ └──────────────────────────────┘
```

### 5.6 Page Layout — Analytics Overview (Super Admin)

```
Page: "Analytics Overview"

┌─────────────────────────────────────────────────────────────────┐
│ Period Selector: [Today] [This Week] [This Month] [Quarter]    │
└─────────────────────────────────────────────────────────────────┘

┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Active Stores│ │ Total Issued │ │ Total Done   │ │ Avg Wait     │
│      5       │ │    1,247     │ │   1,130      │ │   ~9 min     │
└──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Store Rankings                                                   │
│                                                                  │
│ Store Name     │ Issued │ Done │ Cancelled │ Avg Wait │ Cancel %  │
│ ─────────────  │ ────── │ ──── │ ───────── │ ──────── │ ───────── │
│ MegaMart       │    342 │  310 │        32 │  7m 42s  │    9.4%   │
│ QuickStop      │    287 │  275 │        12 │  4m 15s  │    4.2%   │
│ FreshMart      │    198 │  180 │        18 │ 11m 30s  │    9.1%   │
│ ...            │        │      │         │          │           │
│                                                   [View →]      │
└─────────────────────────────────────────────────────────────────┘
```

Clicking a store row navigates to `/dashboard/analytics/{storeId}` for the detail view.

### 5.7 Charting Library & Tech Stack

**Library**: **Recharts**

Recharts is the recommended charting library for this project. It's lightweight, composable, React-native, and integrates seamlessly with shadcn/ui.

**Chart components**:
- `BarChart` for peak hours (hourly distribution)
- `LineChart` for daily throughput (issued/completed trend)

**Styling**:
- Primary series color: `--chart-accent` (`#4F46E5` light theme / `#818CF8` dark theme)
- Secondary series (completed, cancelled): `--primary` (`#0D9488` light / `#2DD4BE` dark)
- Chart text/labels: Use existing `--foreground` tokens for contrast
- Font: Inter for chart axis labels, legends, and data labels. Inter is mapped to `--font-mono` in the Tailwind theme (`globals.css` line 23: `--font-mono: var(--font-inter)`). Use `font-mono` class on chart wrapper elements to apply Inter.

**Responsive design**:
- Charts use `ResponsiveContainer` wrapper for fluid sizing
- Minimum width: 300px; stack charts vertically below `l:` breakpoint (600px). **Note**: This project uses custom breakpoints (`s:`, `l:`, `xl:`, etc.) — standard Tailwind `sm`/`md`/`lg` are disabled. Use `l:grid-cols-2` for side-by-side charts, single column by default.
- Tooltip: Show on hover using design system tokens: `--card` background + `--card-foreground` text + `--border` border (light: white card, dark: `#28354A`). This ensures tooltips match the glass design system rather than using arbitrary opacity values.

**Recharts dependency**: `recharts` is **NOT currently in `web/package.json`** — must be installed (`yarn add recharts`) at the start of Phase 5 frontend work.

### 5.8 Bilingual Support (i18n)

All visible text must use `next-intl` translation keys. Add analytics-specific keys to both `src/messages/en.json` and `src/messages/vi.json`:

```json
{
  "analytics": {
    "title": "Analytics",
    "overview": "Analytics Overview",
    "storeAnalytics": "Store Analytics",
    "period": {
      "today": "Today",
      "week": "This Week",
      "month": "This Month",
      "quarter": "This Quarter"
    },
    "realtime": {
      "queueNow": "Queue Now",
      "servingNow": "Serving Now",
      "issuedToday": "Issued Today",
      "avgWait": "Avg Wait"
    },
    "summary": {
      "totalIssued": "Total Issued",
      "totalCompleted": "Completed",
      "totalCancelled": "Cancelled",
      "totalSkipped": "Skipped",
      "avgQueueWait": "Avg Queue Wait",
      "avgServiceTime": "Avg Service Time",
      "medianWait": "Median Wait",
      "cancelRate": "Cancel Rate",
      "skipRate": "Skip Rate",
      "peakHour": "Peak Hour"
    },
    "charts": {
      "peakHours": "Peak Hours",
      "dailyThroughput": "Daily Throughput",
      "issued": "Issued",
      "completed": "Completed"
    },
    "storeRanking": {
      "title": "Store Rankings",
      "storeName": "Store Name",
      "view": "View"
    },
    "noData": "Not enough data yet",
    "estimating": "Estimating..."
  }
}
```

### 5.9 Store Selector (Super Admin)

On the store analytics page, Super Admins see the same store selector as the queue page (reuse `features/queue/store-selector.tsx` or extract to a shared component). Admin users are auto-locked to their store.

### 5.10 Polling & Data Freshness

- **Realtime stats cards**: Use `POLLING_INTERVAL` from `src/lib/constants.ts` (10 seconds). Follow the queue-stats pattern: `setInterval` + `visibilitychange` listener to pause when tab is hidden.
- **Summary/charts**: Fetch on page load and on period change. No auto-refresh — the data doesn't change fast enough to warrant polling.
- Use the existing `api.ts` fetch wrapper with auth headers.

---

## Phase 6: Client Web — Estimated Wait Time Display

### 6.1 Current State

The client-web **already implements** estimated wait time display:

- `client-web/src/types/queue.ts` — `TicketStatusResponse` includes `estimatedWaitTime: number | null`
- `client-web/src/components/queue/ticket-card.tsx` — renders `estimatedWaitTime` when non-null and status is `WAITING`, with a `waitUnavailable` fallback when null
- i18n keys `queue.estimatedWait` and `queue.waitUnavailable` already exist in both `en.json` and `vi.json`

The field is currently always `null` from the backend. Once Phase 3 populates it (as minutes), the client-web will display it automatically.

### 6.2 Changes Required

**Minimal** — verify and adjust only:

1. **Verify display format**: The current `TicketCard` displays `${status.estimatedWaitTime} min`. Since Phase 3 now returns minutes (not seconds), this works correctly without conversion. Verify the display reads naturally (e.g., "12 min" not "0 min" for very short waits — Phase 3 enforces minimum of 1).
2. **Cold start**: Already handled — `waitUnavailable` text shows when `estimatedWaitTime` is null.
3. **Optional enhancement**: Consider showing `~` prefix for the estimate (e.g., "~12 min") to convey approximation. This is a minor i18n string tweak, not a code change.

---

## Implementation Order

| Step | Phase | Scope | Backend Work | Frontend Work | Dependencies |
|---|---|---|---|---|---|
| 1 | Phase 1 | Analytics write path | Entity + Repository + Service + QueueService integration | None | None |
| 2 | Phase 4 | Continuous aggregates | Schema migration (SQL only) | None | Phase 1 (needs data to aggregate) |
| 3 | Phase 2 | Query endpoints | Controller + AnalyticsQueryService + Repository queries | None | Phase 1, Phase 4 |
| 4 | Phase 3 | Estimated wait time | AnalyticsQueryService addition + QueueService integration | None | Phase 1 (needs TICKET_COMPLETED events) |
| 5 | Phase 5 | Admin dashboard | Sidebar + Overview + Analytics pages + charts + i18n | None | Phase 2, Phase 3 |
| 6 | Phase 6 | Client wait display | None | Ticket card update + i18n | Phase 3 |

**Frontend dependency order** (Phase 5):
- 5.1: Update Overview page (dashboard realtime cards) — depends on Phase 2 (realtime endpoints)
- 5.2: Update Sidebar (add Analytics nav item) — no dependencies, pure UI/translation
- 5.3-5.6: Build Analytics detail pages (charts, summaries) — depends on Phase 2, Phase 3 (all query endpoints)

All frontend work can be completed in Phase 5. Steps 1-4 are backend-only and should complete first.

---

## Security Considerations

- **No public analytics endpoints**: All analytics data requires authentication. The estimated wait time is the only analytics-derived value exposed through a public endpoint (via the existing `TicketStatusResponse`).
- **Store access control**: Store-scoped analytics endpoints use `StoreAccessUtil.requireStoreAccess()` — an Admin can only see their own store's analytics, Super Admins can see all.
- **Rate limiting**: Analytics endpoints fall under the existing `standard` rate limit tier. No special rate limiting needed — these are low-frequency dashboard queries, not high-throughput APIs.
- **Data retention**: The 90-day retention policy on raw events prevents unbounded table growth. Continuous aggregates (rollups) are retained indefinitely.

## Performance Considerations

- **Write path impact**: A single R2DBC INSERT per lifecycle event adds ~1-2ms. Queue operations are admin-initiated (not customer-burst), so this is negligible.
- **Read path**: Dashboard queries hit continuous aggregates (pre-computed), not raw events. Even without aggregates, the hypertable indexes (`store_id, time DESC`) ensure efficient range scans.
- **Estimated wait cache**: The 5-minute Redis cache on `avg_service_duration` prevents repeated TimescaleDB queries on every ticket status check (which IS high-frequency from customer polling).
- **No background job complexity**: Analytics writes are synchronous within the request lifecycle. No message queue, no async worker — keeps the architecture simple for the current scale.

## Testing Considerations

Per `Backend Remaining Plan.md` P1, analytics emission behavior needs test coverage:

- **Event emission**: Verify that each queue lifecycle method emits the correct event type with correct duration
- **Idempotent paths**: Verify that serve/cancel no-op paths do NOT emit duplicate events
- **Duration accuracy**: Verify `wait_duration_seconds` calculation from stored timestamps
- **Cold start**: Verify estimated wait returns `null` when insufficient data exists
- **Trimmed mean**: Verify outlier exclusion in average calculation
- **Access control**: Verify Admin cannot access other stores' analytics, Super Admin can access all

---

## Comprehensive Plan Audit (Opus re-audit, 2026-03-23)

Verified against actual codebase state. All claims cross-referenced with source files.

### A. Logical Consistency

| # | Check | Status |
|---|-------|--------|
| 1 | **Phase Dependencies**: All phases have explicit dependencies in Implementation Order table; no circular deps. Phase 4 (aggregates) correctly depends on Phase 1 (needs data). | ✅ PASS |
| 2 | **Event Type Semantics**: TICKET_CANCELLED (voluntary) vs TICKET_SKIPPED (no-show, Feature 5) clearly distinguished. cancelRate and skipRate won't conflate. TICKET_CANCELLED requires ALTER TYPE migration (confirmed not yet in schema.sql). | ✅ PASS |
| 3 | **Duration Calculations**: Each event type has explicit formula and timestamp source. TICKET_CALLED uses calledAt from Lua script (not separate Instant.now()). TICKET_COMPLETED uses calledAt read before markServed deletes key. Verified: QueueService.serveTicket() calls getTicket() before markServed. | ✅ PASS |
| 4 | **Idempotent Paths**: serve/cancel no-op paths (empty ticket hash) explicitly skip analytics writes. Both methods return early when `ticket.isEmpty()` before analytics TODO locations. | ✅ PASS |
| 5 | **Realtime Stats Source**: Redis-only for realtime (no DB query per request). Polling uses existing `POLLING_INTERVAL` constant (10s). | ✅ PASS |
| 6 | **Cold Start Handling**: Estimated wait returns null with <5 completed events. client-web TicketCard handles null via `waitUnavailable` fallback (verified at ticket-card.tsx lines 59-71). | ✅ PASS |
| 7 | **TTL Management**: avg_service_duration cache: 5min TTL. analytics_event raw data: 90-day retention. Continuous aggregates: infinite. No conflict with existing RedisTTLPolicy entries. | ✅ PASS |
| 8 | **Rate Limiting**: Analytics endpoints fall under existing standard tier. Low-frequency dashboard queries — no special tier needed. | ✅ PASS |
| 9 | **Two-Tier Architecture**: Overview (lightweight realtime cards) vs Analytics (full charts/summaries) properly separated. Overview replaces current redirect-to-queue. Link to full analytics documented. | ✅ PASS |
| 10 | **Store Access Control**: Admin auto-scoped via storeId from auth store. Super Admin sees all via StoreAccessUtil (verified: SUPER_ADMIN gets blanket access). Drill-down route requires SUPER_ADMIN role check. | ✅ PASS |
| 11 | **Super Admin Overview Realtime**: `GET /api/analytics/overview/realtime` endpoint provides cross-store realtime snapshot. Without this, Super Admin Overview cards would have no appropriate API (store-scoped `/realtime` only covers one store). | ✅ PASS |

### B. UI Schema Compliance (verified against `docs/walkthrough/Web Styles.md` and `globals.css`)

| # | Rule | Implementation | Status |
|---|------|----------------|--------|
| 1 | **Component Library**: Only shadcn/ui (CLAUDE.md rule) | Recharts for charts (data visualization, not UI framework); all UI components use shadcn/ui | ✅ PASS |
| 2 | **Charting Library**: Recharts selected | BarChart (peak hours), LineChart (throughput), ResponsiveContainer wrapper. Note: Recharts is a plan-level decision, not mentioned in Web Styles.md. | ✅ PASS |
| 3 | **Colors**: CSS tokens from design system | `--chart-accent` (light: #4F46E5, dark: #818CF8) verified in globals.css. Secondary series uses `--primary`. All tokens exist in both themes. | ✅ PASS |
| 4 | **Border Radius**: Token system | Plan specifies `rounded-xl` on glass-card elements (maps to `--radius-xl`). Tailwind classes, not hardcoded px values. | ✅ PASS |
| 5 | **Fonts**: Be Vietnam Pro (body), Inter (charts) | Inter mapped to `--font-mono` in globals.css line 23. Plan documents that `font-mono` class applies Inter to chart wrappers. Both fonts loaded in layout.tsx. | ✅ PASS |
| 6 | **Glass Design**: Apple Liquid Glass principles | Plan specifies glass class assignments: `glass-card` for stats/summary/chart panels, `glass-panel` for period selector toolbar. Matches existing queue page pattern. | ✅ PASS |
| 7 | **Responsive Design**: Custom breakpoints | Plan uses `l:` breakpoint (600px) for chart stacking. Standard `sm:` (640px) is disabled in this project. Custom breakpoints verified in globals.css lines 8-20. | ✅ PASS |
| 8 | **Accessibility**: WCAG AA contrast | ARIA labels for chart tooltips and legends to be added during Phase 5 implementation. | ⚠️ IMPL |
| 9 | **Bilingual Text**: next-intl keys | Full i18n key tree in Section 5.8. Navigation key "analytics" specified with English and Vietnamese values. | ✅ PASS |
| 10 | **Tooltip Styling**: Design system tokens | Recharts tooltips use `--card` bg + `--card-foreground` text + `--border` border. Consistent with shadcn token pattern. | ✅ PASS |

### C. Loopholes & Oversights

| # | Issue | Resolution | Status |
|---|-------|-----------|--------|
| 1 | **Realtime cards on Analytics page** | Section 5.5 shows realtime cards at top of Analytics detail page | ✅ COVERED |
| 2 | **Failed analytics writes** | Inline writes surface through ExceptionHandler. Queue operation fails visibly, not silently. Acceptable for admin-frequency operations. | ✅ COVERED |
| 3 | **Store rankings pagination** | Acceptable for MVP (stores are admin-created, unlikely >50). Add pagination if needed in Phase 2 review. | ⚠️ DEFERRED |
| 4 | **Query parameter validation** | Use Kotlin enum class `Period { TODAY, WEEK, MONTH, QUARTER }` and `Range { D7, D30, D90 }` — Spring auto-rejects invalid values with 400. | ⚠️ IMPL |
| 5 | **Timezone in time_bucket** | Implementation must use `time_bucket('1 hour', time AT TIME ZONE ...)` to align hourly buckets with local midnight. | ⚠️ IMPL |
| 6 | **Continuous aggregate refresh gap** | Section 4.2 explicitly handles this: UNION aggregate + raw tail query. Repository layer abstracts this. | ✅ COVERED |
| 7 | **Drill-down route auth** | AnalyticsController uses StoreAccessUtil — Admin can only see own store, Super Admin can see all. | ✅ COVERED |
| 8 | **Estimated wait minimum** | `max(1, roundedMinutes)` in `getEstimatedWaitMinutes()` return. Implement in Phase 3. | ⚠️ IMPL |
| 9 | **Metadata JSON injection** | counter_id validated with `@Size(max = 100)` on controller (verified). ticket_number is system-generated. Low risk. | ✅ COVERED |
| 10 | **Sidebar active link** | `pathname.startsWith("/dashboard/analytics")` correctly highlights for both `/analytics` and `/analytics/{storeId}`. Verified in sidebar.tsx line 111. | ✅ COVERED |
| 11 | **Overview → Analytics link** | Admin links to `/dashboard/analytics` (auto-scoped). Super Admin links to `/dashboard/analytics` (overview). No broken link. | ✅ COVERED |
| 12 | **NavItem TypeScript type** | Plan documents updating the label union to include `"analytics"`. Without this, TypeScript rejects the new nav item. | ✅ COVERED |
| 13 | **Sparklines** | Explicitly marked "(Optional)" in wireframes. Not part of MVP scope. | ✅ SCOPED |
| 14 | **Recharts dependency** | Plan states `yarn add recharts` required at Phase 5 start. Not currently in package.json (confirmed). | ✅ COVERED |

### D. Schema & API Contract Completeness

| # | Item | Status |
|---|------|--------|
| 1 | **analytics_event table**: All columns match schema.sql exactly. Verified line-by-line. | ✅ PASS |
| 2 | **analytics_event_type enum**: 5 values in schema.sql. Plan adds TICKET_CANCELLED via ALTER TYPE. All 6 values have clear semantics. | ✅ PASS |
| 3 | **Continuous aggregates**: analytics_hourly + analytics_daily. PERCENTILE_CONT exclusion documented (not supported — median from raw data). | ✅ PASS |
| 4 | **Retention policy**: 90-day raw events. Aggregates retained indefinitely. | ✅ PASS |
| 5 | **RealtimeStatsResponse**: 4 fields, all sourceable from Redis without DB query. | ✅ PASS |
| 6 | **OverviewRealtimeResponse**: 5 fields for Super Admin cross-store realtime. New endpoint. | ✅ PASS |
| 7 | **StoreSummaryResponse**: 11 fields. medianWaitSeconds computed from raw data (not aggregates). | ✅ PASS |
| 8 | **OverviewResponse**: Cross-store summary with per-store breakdown list. | ✅ PASS |
| 9 | **Redis key**: `store:{storeId}:avg_service_seconds` with 5-min TTL. No collision with existing keys. | ✅ PASS |
| 10 | **Authentication**: All analytics endpoints require JWT. Not in public path list. Verified SecurityConfig. | ✅ PASS |

### E. Frontend Implementation Readiness

| # | Item | Status |
|---|------|--------|
| 1 | **Overview page**: Replaces current redirect-to-queue in `/dashboard/page.tsx`. Verified current page is pure redirect. | ✅ PASS |
| 2 | **Analytics routes**: `/dashboard/analytics/page.tsx` + `/dashboard/analytics/[storeId]/page.tsx`. Next.js dynamic route pattern. | ✅ PASS |
| 3 | **Sidebar**: navItems array + NavItem type union update + "analytics" translation key in both language files. | ✅ PASS |
| 4 | **Feature module**: `src/features/analytics/` (does not exist yet — confirmed). 10 files planned. | ✅ PASS |
| 5 | **Component isolation**: Matches existing queue feature pattern. Each chart/card/table is separate. | ✅ PASS |
| 6 | **i18n keys**: Full key tree in Section 5.8. analytics.*, navigation.analytics. | ✅ PASS |
| 7 | **Role-based rendering**: `useAuthStore().isSuperAdmin` (verified: exists, derived from `admin.role === ROLES.SUPER_ADMIN`). | ✅ PASS |
| 8 | **Polling**: `POLLING_INTERVAL` (10s) from `src/lib/constants.ts`. Follows queue-stats pattern: setInterval + visibilitychange pause. | ✅ PASS |
| 9 | **Error handling**: Use existing `ApiError` + `translateCommonApiError()` pattern (verified). Toast fallback via sonner. | ✅ PASS |
| 10 | **Store selector reuse**: `src/features/queue/store-selector.tsx` exists with clean API. `useStoreName` hook also reusable. | ✅ PASS |
| 11 | **Glass styling**: Specified per component type (glass-card for panels, glass-panel for toolbar). Verified in `src/styles/glass.css`. | ✅ PASS |
| 12 | **Page gradient**: Default `bg-gradient-page` (cool) for analytics. Warm gradient reserved for queue. Pattern via `useLayoutStore`. | ✅ PASS |

### F. Backend Implementation Readiness

| # | Item | Status |
|---|------|--------|
| 1 | **Entity**: AnalyticsEvent data class. Fields map 1:1 to schema.sql. metadata as String (JSON). | ✅ PASS |
| 2 | **Repository**: R2DBC DatabaseClient (not ReactiveCrudRepository — no PK on hypertable). | ✅ PASS |
| 3 | **Service (write)**: AnalyticsEventService with 4 record* methods. Optional injection (`= null`) matching MqttPublisher/FcmNotificationService pattern (verified in QueueService). | ✅ PASS |
| 4 | **QueueService integration**: 4 lifecycle points mapped. getTicket() ordering verified correct in both serveTicket() and cancelTicket(). | ✅ PASS |
| 5 | **Service (read)**: AnalyticsQueryService separate from write service. | ✅ PASS |
| 6 | **Controller**: 6 endpoints (4 store-scoped + 2 cross-store). Role/store access via StoreAccessUtil. | ✅ PASS |
| 7 | **Estimated wait**: Trimmed mean, cold start guard, min 1 minute, Redis cache 5-min TTL. | ✅ PASS |
| 8 | **Continuous aggregates**: Hourly + daily views with refresh policies. Refresh gap handled by UNION pattern. | ✅ PASS |
| 9 | **Security**: `/api/analytics/**` requires JWT. StoreAccessUtil enforces store-level access. | ✅ PASS |
| 10 | **Existing TODOs**: QueueService lines 264, 304. RedisTicketRepository lines 14, 18. TicketDto line 24. All verified in codebase. | ✅ PASS |

### G. Known Gaps (Non-Blocking)

| # | Item | Severity | Resolution |
|---|------|----------|-----------|
| 1 | Sparklines on Overview | Optional | Deferred to post-MVP. Marked "(Optional)" in wireframes. |
| 2 | Store rankings pagination | Low | Acceptable for MVP. Add if store count exceeds ~100. |
| 3 | Timezone hint in chart UI | Low | Consider tooltip like "(UTC+7)" in Phase 5 review. |
| 4 | ARIA labels for Recharts | Medium | Add `role`, `aria-label` to chart containers during Phase 5. |
| 5 | Period/range enum validation | Medium | Use Kotlin enum binding — Spring auto-rejects invalid values. Phase 2. |
| 6 | time_bucket AT TIME ZONE | Medium | Verify during Phase 2/4 SQL implementation. |
| 7 | Estimated wait min guard | Low | `max(1, roundedMinutes)` in Phase 3. |

---

## Cross-References

| Document | Relevant Section | Status After This Plan |
|---|---|---|
| `docs/planned/Backend Remaining Plan.md` | P0 — Analytics Event Writes | Resolved by Phase 1 |
| `docs/planned/Backend Remaining Plan.md` | P1 — Populate `estimatedWaitTime` | Resolved by Phase 3 |
| `docs/planned/Backend Remaining Plan.md` | `RedisCounterRepository.getCurrentCount` pending usage | Used in Phase 2 (realtime stats) |
| `docs/planned/Backend Remaining Plan.md` | Deferred Watchlist #1 (markServed/markCancelled split) | Divergence implemented in Phase 1 (different event types) |
| `docs/planned/Features Ideas.md` | Feature 1 — Dynamic Wait Time Estimation | Implemented by Phase 3 |
| `docs/planned/Features Ideas.md` | Feature 7 — Operational Analytics Dashboards | Implemented by Phases 2, 4, 5 |
| `docs/done/Web Implementation Plan.md` | `TicketStatusResponse.estimatedWaitTime` always null | Resolved by Phase 3 |
| `docs/done/Client Web Implementation Plan.md` | Estimated wait time deferred | Resolved by Phase 6 |
| `QueueService.kt` TODOs | Lines 264, 304 | Resolved by Phase 1 |
| `TicketDto.kt` TODO | Line 24 | Resolved by Phase 3 |
| `RedisTicketRepository.kt` TODOs | Lines 14, 18 | Informational — already correct, comments removed |
