# NotiGuide — Future Feature Ideas

Detailed idea-level plans for 8 features. No code, just concepts, logic, and dependencies.

Features 9–10 cover the **Settings page expansion** (Store Preferences, Security). Features 2, 4, and 5 are surfaced to admins through these settings tabs.

---

## Feature 1: Dynamic Wait Time Estimation

> **Implementation plan**: See `docs/done/Analytics Implementation Plan.md` Phase 3 — **DONE** (2026-03-23). Estimated wait time is populated on ticket issuance using rolling avg service duration × queue position.

**Goal:** Tell the customer "Your estimated wait is ~12 minutes" when they receive a ticket.

### How It Works
- Track the **average service duration** per store: the time between `TICKET_CALLED` and `TICKET_COMPLETED` (already captured in `analytics_event.wait_duration_seconds`)
- When a new ticket is issued, calculate: `estimated_wait = tickets_ahead × avg_service_duration`
- Refine over time using a **rolling average** of the last N completed tickets (e.g., last 20) rather than all-time averages, so it adapts to current staffing levels and rush hours

### Edge Cases
- **Cold start:** No historical data for a brand-new store → show "Estimating..." or use a system-wide default (e.g., 5 min/ticket)
- **Multiple counters:** If a store has 3 active counters processing simultaneously, divide the estimate by the active counter count: `estimated_wait = (tickets_ahead × avg_duration) / active_counters`
- **Outliers:** One ticket that took 45 minutes (complex issue) shouldn't skew the avg → use **median** or trim the top/bottom 10% before averaging

### Data Source
- TimescaleDB `analytics_event` table — query recent `TICKET_COMPLETED` events for `wait_duration_seconds`
- Redis queue — `ZCARD` for current `tickets_ahead` count

### Output
- Returned as part of the ticket issuance response: `{ "ticketNumber": "A-042", "estimatedWaitMinutes": 12 }`
- Can also be refreshed on-demand via a `GET /api/queue/{storeId}/estimate/{ticketId}` endpoint

### Dependencies
- Requires Feature 7 (Analytics) to be generating data
- Accuracy improves with Feature 6 (Multi-Counter) for the `active_counters` divisor

---

## Feature 2: "You're Next" Proactive Alerts (FCM)

> **Settings integration:** Customer-facing alert thresholds are configured per-store in Feature 9 (Store Preferences).

**Goal:** Push a notification to the customer's device when they're about to be called (e.g., 2 spots away).

### How It Works
- When an admin calls the **next ticket** (`TICKET_CALLED` event), the system checks who is now at position 1 and 2 in the queue
- For those customers, trigger an FCM push notification:
  - Position 2: *"Almost there! You're 2nd in line at [Store Name]."*
  - Position 1: *"You're next! Please head to the counter."*
- When the ticket is actually called: *"It's your turn! Counter #3 is ready for you."*

### Alert Thresholds
- Configurable per store (some stores may want alerts at position 3, others only at 1)
- Default: alert at position **2** (upcoming) and position **0** (your turn)

### FCM Integration
- Customer devices register their FCM token when they join the queue (stored in Redis alongside ticket data, or in a `customer_device` transient mapping)
- Server-side: use the Firebase Admin SDK to send targeted notifications
- Supports Android, iOS, and Web (via service workers)

### Preventing Spam
- Each position threshold triggers **one** notification per ticket — no repeats
- Track which alerts have been sent in the Redis ticket hash (e.g., `alert_position_2: true`)

### Dependencies
- Firebase project setup + Admin SDK on the backend
- Client apps must request notification permission and register FCM tokens
- Ties into the existing `notifier_device` table or a new transient Redis mapping for customer tokens

---

## Feature 4: Queue Pausing / Capping

> **Settings integration:** Queue cap is configured per-store in Feature 9 (Store Preferences), Phase 2. Pause/resume is a real-time action on the queue page, not a setting.

**Goal:** Let admins temporarily stop accepting new tickets without disrupting the existing queue.

### How It Works
- A store has a new state: `ACTIVE` (default), `PAUSED`, or `CAPPED`
- **Paused:** No new tickets can be issued. Existing tickets continue being processed normally. Customer-facing message: *"This queue is temporarily paused. Please check back shortly."*
- **Capped:** A maximum queue size is set (e.g., 50 tickets). Once reached, new tickets are rejected with: *"Queue is full. Please try again later."*

### Admin Controls
| Action | Who Can Do It | Effect |
|---|---|---|
| Pause queue | Admin (own store) or SuperAdmin | Sets store queue state to `PAUSED` |
| Resume queue | Admin (own store) or SuperAdmin | Sets store queue state to `ACTIVE` |
| Set cap | Admin (own store) or SuperAdmin | Sets max queue size (e.g., 50) |
| Remove cap | Admin (own store) or SuperAdmin | Removes the max limit |

### Where State Lives
- Redis key: `store:{storeId}:queue_state` → value: `ACTIVE` / `PAUSED` / `CAPPED:50`
- Checked at ticket issuance time — if paused/capped, reject immediately with a clear error
- Persisting to Postgres is optional (for audit/history), but the real-time check is Redis-only for speed

### Use Cases
- Store closing in 30 minutes → admin pauses queue, finishes remaining tickets
- Lunch rush → admin caps at 30 to prevent overload
- Emergency → SuperAdmin pauses all queues for a specific store

---

## Feature 5: No-Show / Grace Period Handling

> **Settings integration:** Grace period, no-show action, max re-queues, and re-queue offset are configured per-store in Feature 9 (Store Preferences), Phase 3.

**Goal:** Handle customers who don't show up when their ticket is called.

### Grace Period is Optional

The grace period feature is **opt-in per store**. Not all businesses need automatic no-show handling:
- A **hospital** or **government office** may want a 3-minute grace period — customers might be in a waiting room and need time to reach the counter.
- A **coffee shop** or **fast-food counter** expects the customer to be present when called — no grace period needed. The admin simply serves or manually skips.

When grace period is disabled (`grace_period_sec = 0`), no timer is set, no automatic action fires. The admin handles the flow entirely through manual "Serve" / "Cancel" actions as they do today.

### Flow (When Grace Period Is Enabled)
1. Admin calls ticket `A-042` → status becomes `CALLED`
2. Timer starts: **grace period** (e.g., 3 minutes, configurable per store)
3. If customer arrives → Admin marks `COMPLETED` → normal flow
4. If timer expires and customer hasn't arrived:
   - **Option A — Skip:** Mark as `SKIPPED`. Customer is removed from the queue entirely. They can re-join at the back.
   - **Option B — Re-queue:** Move the ticket back into the queue, but N positions behind the current front (e.g., 3 spots back). Status goes to `REQUEUED`. Max re-queues: 1.

### Flow (When Grace Period Is Disabled)
1. Admin calls ticket `A-042` → status becomes `CALLED`
2. No timer. Admin serves the customer or manually cancels/skips if they don't show.
3. Identical to today's behavior — zero disruption for stores that don't need this feature.

### Grace Period Tracking
- When `CALLED`, set a Redis key `ticket:{storeId}:{ticketId}:grace_expiry` with TTL = grace period duration **only if `grace_period_sec > 0`**
- When this key expires (via Redis keyspace notification — you already have [RedisKeyExpirationListener](file:///home/thomas/Coding/Java%20%28Kotlin%29/notiguide/src/main/kotlin/com/thomas/notiguide/core/redis/RedisKeyExpirationListener.kt#17-54)), trigger the configured no-show action
- Alternatively, admins can manually trigger "No-Show" before the timer expires

### Customer Notification (ties into Feature 2)
- On `CALLED`: *"It's your turn! Head to Counter #3."*
- 1 minute before grace expiry (only if grace period enabled): *"Reminder: Please proceed to Counter #3 or your ticket will be skipped."*
- On `SKIPPED`: *"Your ticket A-042 has been skipped. You may re-join the queue."*

### Configurable Per Store
| Setting | Default | Description |
|---|---|---|
| `grace_period_seconds` | 0 (disabled) | How long to wait before no-show action. 0 = no automatic handling. |
| `no_show_action` | `SKIP` | `SKIP` or `REQUEUE` (only relevant when grace period > 0) |
| `max_requeues` | 1 | How many times a ticket can be re-queued (only relevant when grace period > 0) |
| `requeue_offset` | 3 | How many positions behind the front to re-insert (only relevant when grace period > 0) |

### Dependencies
- Leverages existing [RedisKeyExpirationListener](file:///home/thomas/Coding/Java%20%28Kotlin%29/notiguide/src/main/kotlin/com/thomas/notiguide/core/redis/RedisKeyExpirationListener.kt#17-54) for automatic grace period expiry
- Feature 2 (FCM alerts) for notifying the customer

---

## Feature 6: Multi-Counter / Service Type Routing

**Goal:** Allow a single store to have multiple distinct queues for different service types.

### Concept
Instead of one flat queue per store, a store can define **service types** (also called "queue lanes"):

| Store: "MegaMart" | Queue Lanes |
|---|---|
| Lane A | New Purchases |
| Lane B | Returns & Exchanges |
| Lane C | Customer Service |

Each lane has its own independent queue, ticket numbering prefix, and counter assignments.

### How Routing Works
- When a customer requests a ticket, they select a **service type** (via the app or a kiosk)
- The ticket is added to the corresponding lane's ZSet in Redis: `queue:{storeId}:{serviceTypeId}`
- Counters are assigned to one or more lanes. Counter #1 might serve Lane A only, while Counter #2 serves both Lane B and Lane C
- When an admin at Counter #2 clicks "Call Next", the system pulls from the oldest ticket across their assigned lanes

### Ticket Transfer
- An admin can **transfer** a ticket from one lane to another without cancelling it
- The ticket maintains its original `issued_at` timestamp for fair wait-time tracking
- Customer gets an FCM notification: *"Your ticket has been transferred to the Returns queue."*

### Redis Key Structure
- Queue: `queue:{storeId}:{serviceTypeId}` (ZSet — same as current, but scoped by service type)
- Serving: `serving:{storeId}:{serviceTypeId}` (Set)
- Counter assignment: `counter:{storeId}:{counterId}:lanes` (Set of serviceTypeIds)

### Database Impact
- New `service_type` table: [id](file:///home/thomas/Coding/Java%20%28Kotlin%29/ESP-Doorbell/src/main/kotlin/com/thomas/espdoorbell/doorbell/user/entity/Users.kt#50-62), `store_id`, [name](file:///home/thomas/Coding/Java%20%28Kotlin%29/ESP-Doorbell/src/main/kotlin/com/thomas/espdoorbell/doorbell/shared/principal/UserPrincipal.kt#21-22), `prefix` (e.g., "A", "B", "C"), `is_active`
- Current single-queue stores would have one default service type auto-created

### Backward Compatibility
- Stores with a single service type behave exactly like they do today — no breaking changes
- The "default" lane is created automatically when a store is set up

---

## Feature 7: Operational Analytics Dashboards

> **Implementation plan**: See `docs/done/Analytics Implementation Plan.md` — **DONE** (2026-03-23). Covers write path (Phase 1), query endpoints (Phase 2), estimated wait time (Phase 3), continuous aggregates (Phase 4), and admin dashboard (Phase 5–6).

**Goal:** Surface actionable insights from your TimescaleDB data for store admins and super admins.

### Metrics to Expose

**Real-time (from Redis):**
| Metric | Source |
|---|---|
| Current queue depth | `ZCARD queue:{storeId}` |
| Currently serving count | `SCARD serving:{storeId}` |
| Active counters | Count of counters with recent activity |

**Historical (from TimescaleDB):**
| Metric | Query Basis |
|---|---|
| Avg wait time (today / this week / this month) | `AVG(wait_duration_seconds)` grouped by time bucket |
| Peak hours | `COUNT(*)` of `TICKET_ISSUED` grouped by hour-of-day |
| Tickets processed per counter | `COUNT(*)` of `TICKET_COMPLETED` grouped by `counter_id` |
| No-show rate | `COUNT(SKIPPED) / COUNT(ISSUED)` as percentage |
| Service type breakdown | Ticket counts grouped by service type (requires Feature 6) |
| Daily/weekly throughput | `COUNT(TICKET_COMPLETED)` over time buckets |

### Access Control
| Role | What They See |
|---|---|
| SuperAdmin | All stores, cross-store comparisons, system-wide trends |
| Admin | Own store only |

### API Endpoints (idea-level)
- `GET /api/analytics/{storeId}/realtime` → live Redis snapshot
- `GET /api/analytics/{storeId}/summary?period=today` → aggregated TimescaleDB stats
- `GET /api/analytics/{storeId}/peak-hours?range=7d` → hourly distribution
- `GET /api/analytics/overview` → SuperAdmin cross-store summary

### TimescaleDB Advantages
- **Continuous aggregates:** Pre-compute hourly/daily rollups so queries are instant even over months of data
- **Data retention policies:** Auto-drop raw events older than 90 days, keep rollups forever
- **Time-bucket functions:** `time_bucket('1 hour', time)` for clean groupings

### Dependencies
- Requires analytics events to be consistently written (they're already in the schema)
- Feature 6 enriches analytics with service type breakdowns
- Feature 5 enriches analytics with no-show rates

---

## Feature 9: Store Preferences

**Goal:** Let store admins configure queue behavior defaults, queue limits, and no-show policies from a dedicated "Store" settings tab.

### Tab: Settings → Store (hidden for ROLE_SUPER_ADMIN)

Super Admins manage stores from the Stores page. This tab is only for `ROLE_ADMIN` admins and shows settings for their assigned store.

### Phase 1: Default Counter ID (Frontend Only)

No backend changes. Stored in `localStorage` scoped by store ID.

| Setting | Type | Default | Description |
|---|---|---|---|
| Default counter ID | Text input | Empty | Pre-fills the counter ID field on the queue page |

#### How It Works
- Store value in `localStorage` under key `store:{storeId}:defaultCounterId`
- Queue page reads this as initial state for counter ID input (admin can still override per-call)
- Validation: max 100 characters (matches existing constraint)

#### UI Layout
```
Card: "Queue Defaults"
  └─ Input: Default counter ID
     Caption: "Pre-fills the counter field when calling the next ticket"
```

### Phase 2: Queue Pausing & Capping (Backend Required)

*This is the settings surface for Feature 4 (Queue Pausing/Capping).*

#### Queue Cap Setting

| Setting | Type | Default | Stored In | Description |
|---|---|---|---|---|
| Max queue size | Number input | 0 (unlimited) | Database + Redis | Reject new tickets when queue reaches this size |

Pause/resume is **not** a persistent setting — it's a real-time toggle button on the queue page. The cap is configured here as a persistent store preference.

#### Backend Work
- **New table: `store_settings`**
  ```sql
  CREATE TABLE store_settings (
    store_id          UUID PRIMARY KEY REFERENCES store(id) ON DELETE CASCADE,
    max_queue_size    INT NOT NULL DEFAULT 0,
    grace_period_sec  INT NOT NULL DEFAULT 0,
    no_show_action    VARCHAR(10) NOT NULL DEFAULT 'SKIP',
    max_requeues      INT NOT NULL DEFAULT 1,
    requeue_offset    INT NOT NULL DEFAULT 3,
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
  );
  ```
- **Domain**: `StoreSettings` entity + `StoreSettingsRepository` in `domain/store/`
- **New endpoints** under `/api/stores/{id}/settings`:
  - `GET` — get store settings (Admin of store or Super Admin)
  - `PUT` — update store settings (Admin of store or Super Admin)
- **Queue state endpoints** under `/api/queue/admin/{storeId}`:
  - `POST /pause` — set queue state to `PAUSED`
  - `POST /resume` — set queue state to `ACTIVE`
- **Auto-create default settings row** when a store is created (in `StoreService.createStore`)
- **QueueService.issueTicket()** changes:
  - Read `store:{storeId}:queue_state` from Redis
  - If `PAUSED` → reject with 409
  - If capped → compare `ZCARD` against `max_queue_size`, reject if at cap
- **Redis cache**: `store:{storeId}:settings` hash — avoids DB lookup on every ticket issuance. Invalidate on settings PUT.
- **RedisKeyManager**: add `queueState(storeId)` and `storeSettings(storeId)` key patterns

### Phase 3: No-Show & Grace Period (Backend Required)

*This is the settings surface for Feature 5 (No-Show/Grace Period Handling).*

#### Settings

| Setting | Type | Default | Description |
|---|---|---|---|
| Grace period | Number input (seconds) | 180 | Time before a called ticket is marked no-show |
| No-show action | Select | SKIP | `SKIP` (remove) or `REQUEUE` (move back) |
| Max re-queues | Number input | 1 | Times a ticket can be re-queued before final skip |
| Re-queue offset | Number input | 3 | Positions behind front to re-insert |

These columns are already in the `store_settings` table from Phase 2.

#### Backend Work
- **QueueService.callNext()** changes: after setting ticket to `CALLED`, set grace expiry TTL key (`ticket:{storeId}:{ticketId}:grace_expiry`) if `grace_period_sec > 0`
- **RedisKeyExpirationListener** changes: detect `grace_expiry` key pattern → look up store's `no_show_action` → execute skip or requeue
- **New Lua script for requeue**: atomically remove from serving set, re-insert into queue ZSet at calculated score (position = front + offset)
- **New ticket statuses**: `SKIPPED` and `REQUEUED` added to ticket status enum
- **QueueAdminController**: add `POST /tickets/{ticketId}/no-show` for manual no-show trigger
- **Track requeue count**: store `requeue_count` in ticket hash, compare against `max_requeues`

#### UI Layout (full Store Preferences tab after Phases 1–3)
```
Card: "Queue Defaults"
  └─ Input: Default counter ID (Phase 1, localStorage)

Card: "Queue Limits"
  └─ Input: Max queue size (0 = unlimited)
     Caption: "New tickets are rejected when the queue reaches this size"

Card: "No-Show Handling"
  ├─ Toggle: Enable grace period
  │  Caption: "Automatically handle no-shows after a timeout. Disable for stores where customers are expected to be present immediately."
  ├─ Input: Grace period (seconds, range 30–600, visible if enabled)
  │  Caption: "How long to wait for a called customer before marking no-show"
  ├─ Separator (visible if enabled)
  ├─ Select: No-show action (Skip / Re-queue) (visible if enabled)
  ├─ Separator (visible if Re-queue)
  ├─ Input: Max re-queues (visible if Re-queue)
  └─ Input: Re-queue offset (visible if Re-queue)
```

### Phase 4: Operating Hours (Future)

- Per-store open/close hours per day of week
- Auto-pause queue outside operating hours
- Requires `store_schedule` table or extension to `store_settings`
- Timezone handling per store

No detailed spec — depends on multi-timezone support decisions.

### Dependencies
- Phase 1: none (frontend-only)
- Phase 2: Feature 4 (Queue Pausing/Capping) backend logic
- Phase 3: Feature 5 (No-Show/Grace Period) backend logic, extends `RedisKeyExpirationListener`
- Phase 4: TBD

---

## Feature 10: Security Settings

**Goal:** Give admins visibility into their login history and active sessions, with the ability to revoke sessions from other devices.

### Tab: Settings → Security (visible to all admins)

Each admin sees only their own sessions and login history.

### Phase 1: Login History (Backend Required)

#### What It Shows

A table of the last 20 login attempts (successful and failed) for the current admin.

| Column | Description |
|---|---|
| Date & time | When the login occurred |
| IP address | Source IP (from `X-Forwarded-For` or remote address) |
| Status | Success / Failed |

#### Backend Work
- **New table: `login_history`**
  ```sql
  CREATE TABLE login_history (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    admin_id    UUID NOT NULL REFERENCES admin(id) ON DELETE CASCADE,
    ip_address  VARCHAR(45) NOT NULL,
    success     BOOLEAN NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
  );

  CREATE INDEX idx_login_history_admin_created
    ON login_history (admin_id, created_at DESC);
  ```
- **Domain**: `LoginHistory` entity + repository in `domain/admin/`
- **Record on login**: in `AdminAuthService.login()`, insert a row after authentication (both success and failure). Extract IP from `ServerHttpRequest` (same pattern as `RateLimitFilter`)
- **New endpoint**: `GET /api/admins/{id}/login-history?limit=20` — returns recent logins, own-admin-only access
- **Retention**: scheduled task or DB policy to purge entries older than 90 days

#### UI Layout
```
Card: "Login History"
  └─ Table: Date | IP Address | Status
     └─ "Load more" button (if > 20 entries exist)
```

### Phase 2: Active Sessions (Backend Required)

Requires session tracking infrastructure — the largest backend change across all settings features.

#### What It Shows

A list of currently active sessions for the current admin with revocation.

| Column | Description |
|---|---|
| Device / Browser | User-Agent parsed to readable format |
| IP address | Source IP at login time |
| Last active | Last API request timestamp |
| Current | Badge indicating "This device" |
| Action | "Revoke" button (disabled for current session) |

#### Backend Work
- **New table: `admin_session`**
  ```sql
  CREATE TABLE admin_session (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    admin_id      UUID NOT NULL REFERENCES admin(id) ON DELETE CASCADE,
    token_hash    VARCHAR(64) NOT NULL UNIQUE,
    ip_address    VARCHAR(45) NOT NULL,
    user_agent    TEXT,
    last_active   TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
  );

  CREATE INDEX idx_admin_session_admin ON admin_session (admin_id);
  CREATE INDEX idx_admin_session_token_hash ON admin_session (token_hash);
  ```
- **Session lifecycle**:
  - **On login**: hash the issued JWT (SHA-256), insert row with IP + User-Agent. Return session ID in login response.
  - **On each authenticated request**: update `last_active` (throttled — at most once per 5 minutes via Redis key `session:{hash}:last_update` with 5min TTL, only write to DB when key is absent)
  - **On logout**: delete the session row for the current token
  - **On revoke**: delete the session row, add token hash to Redis blacklist (`revoked:{hash}` with TTL = remaining JWT lifetime)
- **JWTAuthFilter change**: after verifying JWT signature and expiry, check if token hash exists in Redis blacklist → reject if revoked
- **New endpoints** under `/api/admins/{id}/sessions`:
  - `GET` — list active sessions for own admin
  - `DELETE /{sessionId}` — revoke a session (own admin only, cannot revoke current)
- **Cleanup**: scheduled task to delete session rows where `created_at` is older than the JWT max lifetime

#### Frontend Work
- Parse User-Agent into readable device/browser labels (simple regex map)
- "This device" badge on current session (match by session ID stored in Zustand after login)
- "Revoke" button with confirmation dialog
- Refresh session list after revocation

#### UI Layout (full Security tab after Phases 1–2)
```
Card: "Active Sessions"
  └─ List of session cards:
     ├─ Device/Browser label + "This device" badge (if current)
     ├─ IP address · Last active "2 hours ago"
     └─ [Revoke] button (hidden for current session)

Card: "Login History"
  └─ Table: Date | IP Address | Status
     └─ "Load more" button
```

### Dependencies
- Phase 1: none beyond existing auth flow
- Phase 2: Redis blacklist infrastructure, JWTAuthFilter modification

---

## Settings Tab Summary

After all settings features are implemented:

```
Account | Password | Store | Security
```

- **Store** tab hidden for `ROLE_SUPER_ADMIN`
- All other tabs visible to all admins

## Implementation Order

| Step | Feature | Scope | Backend Work | Effort |
|---|---|---|---|---|
| 1 | Feature 9 Phase 1 | Default counter ID | None | Small |
| 2 | Feature 10 Phase 1 | Login history | `login_history` table + endpoint | Medium |
| 3 | Feature 9 Phase 2 | Queue pausing & capping | `store_settings` table + queue state Redis keys | Medium |
| 4 | Feature 9 Phase 3 | No-show & grace period | QueueService + RedisKeyExpirationListener + Lua script | Large |
| 5 | Feature 10 Phase 2 | Active sessions | `admin_session` table + JWT blacklist | Large |
| 6 | Feature 9 Phase 4 | Operating hours | TBD | Medium |

Step 1 is frontend-only and can ship immediately. Steps 2–5 each require backend work. Step 6 depends on features not yet designed.
