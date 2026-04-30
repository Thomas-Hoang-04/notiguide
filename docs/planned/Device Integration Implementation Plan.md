# Device Integration Implementation Plan

Concrete backend and admin-web plan for integrating the unified `device` domain end-to-end: ESP-01 and ESP32-C3 **receiver** families, passive PT2272 receivers, **and** the ESP32-C3 **transmitter hub** that drives them. Both families share enrollment, approval, lifecycle, and admin UX through one Kotlin domain; firmware contracts live in their respective design guides and are referenced — never duplicated — here.

Authoritative firmware sources:

- Receivers — [RECEIVER_DESIGN_GUIDE.md](RECEIVER_DESIGN_GUIDE.md) (historical ESP-01 reference) and [RECEIVER_ESP32C3_DESIGN_GUIDE.md](RECEIVER_ESP32C3_DESIGN_GUIDE.md).
- Transmitter hub — [TRANSMITTER_ESP32C3_DESIGN_GUIDE.md](TRANSMITTER_ESP32C3_DESIGN_GUIDE.md). §A–§G describe the firmware; §H of that guide is the originating backend slice that has been merged into this plan and should now be treated as a duplicate, with this plan as the source of truth.

If this plan disagrees with the firmware guides on contract specifics (topics, canonical strings, RF-code shape), fix the guide first, then refresh this plan.

**Last updated:** 2026-04-29

---

## 1. Scope, Baseline, And Risk Map

### 1.0 Platform Consistency Principle

A device-bound ticket is a **regular queue ticket** that happens to have a `device_id` attached. It enters the same Redis queue (global + service-type sorted sets), holds the same ticket hash shape, participates in the same position-ordered FIFO, and flows through every admin operation identically to a mobile-web-client ticket. The table below maps each lifecycle step to what is shared and what differs:

| Lifecycle step | Shared (identical for both platforms) | Device-specific addition | Mobile-web-specific |
| --- | --- | --- | --- |
| **Issue** | `QueueService.issueTicket` creates the ticket hash, ZADD to queue, SSE `TICKET_ISSUED`, MQTT `TICKET_ISSUED`, counter increment, position tracking | `device_id` written to ticket hash; `device:busy:{deviceId}` key set; preflight hub election check (§D7a step 1) | FCM token registration (customer-initiated) |
| **Wait (WAITING)** | Ticket sits in queue sorted set with timestamp score; admin sees it in waiting list; position is ZRANK-based; TTL = 12 h; SSE updates flow to admin dashboard | `device:busy:{deviceId}` TTL mirrors `TICKET_WAITING` | Customer polls status; position alerts via FCM when near front |
| **Call Next / Jump Call** | `callNext` / `callSpecificTicket` Lua scripts pop from queue → serving set; status → CALLED; `called_at` set; TTL → 30 min; SSE `TICKET_CALLED`; MQTT `TICKET_CALLED`; proactive position alerts sent to *other* waiting tickets | `DEVICE_CALL_REQUESTED` event published → transmitter dispatch → RF broadcast to physical receiver; `device:busy:{deviceId}` stays bound to the ticket and is refreshed to `TICKET_CALLED`; FCM skipped for *this* ticket | FCM `TICKET_CALLED` notification sent to *this* ticket's registered token |
| **Serve** | `serveTicket` sets status → SERVED; TTL → 2 h; removes from serving set; SSE `TICKET_SERVED`; MQTT `TICKET_SERVED`; analytics `TICKET_COMPLETED` | `DEVICE_STOP_REQUESTED` event → RF stop broadcast. The busy key moves to a terminal TTL before dispatch and is deleted only after a successful stop publish; on dispatch failure it stays reserved fail-closed | FCM token removed |
| **Cancel** | `cancelTicket` sets status → CANCELLED; TTL → 2 h; removes from queue (if WAITING) or serving (if CALLED); SSE `TICKET_CANCELLED`; MQTT `TICKET_CANCELLED`; analytics `TICKET_CANCELLED` | If was CALLED: `DEVICE_STOP_REQUESTED` → RF stop broadcast. The busy key moves to a terminal TTL before dispatch and is deleted only after a successful stop publish; on dispatch failure it stays reserved fail-closed | FCM token removed |
| **No-show → Requeue** | `handleNoShow` executes `REQUEUE_TICKET_SCRIPT` Lua; ticket re-enters queue at offset position; status → REQUEUED → WAITING; requeue_count incremented; TTL → 12 h; SSE `TICKET_REQUEUED` | `DEVICE_STOP_REQUESTED` event → RF stop broadcast; `device_id` stays on ticket hash; `device:busy:{deviceId}` TTL returns to `TICKET_WAITING`, and a dispatch failure leaves that reservation in place | — |
| **No-show → Skip** | `handleNoShow` sets status → SKIPPED; TTL → 2 h; removes from serving; SSE `TICKET_SKIPPED`; analytics `TICKET_SKIPPED` | `DEVICE_STOP_REQUESTED` event → RF stop broadcast. The busy key moves to a terminal TTL before dispatch and is deleted only after a successful stop publish; on dispatch failure it stays reserved fail-closed | FCM token removed |
| **Transfer** | `transferTicket` moves WAITING ticket between service-type queues; score preserved; SSE `TICKET_TRANSFERRED` | `device_id` stays on ticket hash; `device:busy:{deviceId}` unchanged (device follows the ticket) | — |
| **Queue pause/resume** | `pauseQueue` / `resumeQueue` toggles `queue_state` | No difference — device-bound tickets honour queue state identically | No difference |
| **Cleanup** | `ServingSetCleanupScheduler` removes orphaned serving-set entries; Redis TTL auto-expires stale tickets | `device:busy:{deviceId}` TTL mirrors ticket TTL; key auto-expires alongside ticket | FCM token TTL (12 h) auto-expires alongside ticket |
| **Admin dashboard** | Ticket appears in waiting list, serving display, and queue stats identically; all SSE events fire | Device badge shown on ticket row (§D7b); links to device detail | — |

**The admin never needs to know whether a ticket was issued via a device or via the mobile web client to manage it.** Call Next, serve, cancel, no-show, transfer, pause/resume, and cleanup all work through the same `QueueService` / `QueueAdminController` methods regardless of origin. The device-specific additions are side effects that fire transparently based on the presence of `device_id` in the ticket hash.

Platform-specific features that **cannot** exist on the other platform (by nature of the medium, not by omission):

- **Device → no mobile-web equivalent:** RF broadcast to a physical buzzer/vibrator. The customer carries the device, not a phone — there is no browser to send FCM to.
- **Mobile web → no device equivalent:** FCM push notifications, client-side status polling, in-browser ticket card UI, haptic feedback, position alerts. The physical device has no screen and no return channel to the customer beyond vibration/buzzer.

These are inherent platform differences, not gaps to close. The backend treats both origins identically at the queue level; only the notification channel diverges.

### 1.1 Scope

In scope:

- Backend: Kotlin 2.3.20, Spring Boot 3.5.11, Java 21, WebFlux, coroutines, R2DBC, Redis, Paho MQTT v5 (`org.eclipse.paho.mqttv5.client:1.2.5`).
- Admin web: Next.js 16.2.1, React 19.2.4, next-intl 4.8.3, Tailwind 4, shadcn/ui, Biome 2.2.0.
- Backend/admin end-state for both **receiver** and **transmitter hub** families: enrollment, bootstrap, activation, RF-code (receivers only), lifecycle, heartbeat ingestion (hubs only), active-hub election, queue-side dispatch, and the admin UI that surfaces all of this.
- Docs: this plan and `docs/CHANGELOGS.md`.

Out of scope:

- Firmware in `receiver-esp32/`, `receiver-esp8266/`, and `transmitter/`. The transmitter design guide §A–§G covers the firmware target end state; §I covers firmware build config.
- Customer-facing `client-web/`.
- Web Serial bridge (provisioning helper).
- Per-device MQTT credentials and broker ACLs (v1 uses one shared MQTT credential; isolation rests on TLS, unguessable `public_id`s, and signed operational commands).
- `DEVICE_TRIGGERED` analytics emission (deferred — see §11).

### 1.2 Repo Invariants The Plan Must Preserve

- `SecurityConfig` permits only `/api/auth/**`, `/api/queue/public/**`, `/actuator/health`, `/actuator/info`. Device endpoints stay authenticated; no permit-all widening.
- `RateLimitFilter.resolveTier` currently matches strict only for `/api/queue/public/**`, auth for `/api/auth/**`, standard for everything else under `/api/**`. Method-specific strict routes require making this helper method-aware.
- `ReactiveRedisTemplate<String, String>` with `StringRedisSerializer`. Do not introduce JSON-wrapping serializers.
- Ticket hash key: `ticket:{storeId}:{ticketId}` via `RedisKeyManager.ticket(storeId, ticketId)`.
- `MqttConfig` is `@ConditionalOnProperty(prefix = "mqtt", name = ["broker"])`. New listeners and publishers must be `@ConditionalOnBean(MqttClientManager::class)`.
- `MqttPublisher` keeps the `${mqtt.topic-prefix}/store/{storeId}/queue` topic. Device-domain publishing uses a fresh wrapper (`DeviceMqttPublisher`) that publishes to top-level `receiver/...` and `transmitter/...` paths — separate from the prefixed queue topics.
- `MqttClientManager` currently exposes only `subscribe(topicFilter, qos)` and caches subscriptions as `topic → qos` for reconnect (`backend/src/main/kotlin/com/thomas/notiguide/core/mqtt/MqttClientManager.kt`). Phase F0 extends it with a subscription descriptor/overload that can preserve MQTT v5 `noLocal` semantics across reconnects for `transmitter/bootstrap/+`.
- `web/src/components/ui` does **not** ship `table`, `tabs`, or `accordion`. Existing admin/store tables use native `<table>` markup inside `glass-card`.
- Sidebar labels are a TS union in `web/src/components/layout/sidebar.tsx`; adding `devices` requires updating both the union and `navigation.devices` in `web/src/messages/{en,vi}.json`.

### 1.3 Receiver Risk Map (GitNexus, upstream — verified 2026-04-25)


| Symbol              | Risk     | d=1 dependents | Action                                                                                                                                                |
| ------------------- | -------- | -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `QueueService`      | LOW      | 4              | New methods only; no signature changes outside `issueTicket` (§D7a).                                                                                  |
| `RedisKeyManager`   | MEDIUM   | 9              | Add keys only; never rename existing ones.                                                                                                            |
| `MqttClientManager` | LOW      | 2              | Wrap; do not mutate the existing API. The v5 subscription overload (§F0) is additive.                                                                 |
| `SecurityConfig`    | LOW      | 0              | No changes.                                                                                                                                           |
| `RateLimitFilter`   | LOW      | 1              | Make tier resolution method-aware; only enrollment-token creation uses strict tier.                                                                   |
| `TicketDto`         | **HIGH** | 8              | `deviceId` / `deviceName` addition breaks 3 processes (`issueTicket`, `callNext`, `callSpecificTicket`); update every consumer in the same D7a slice. |


Phase F0's first task re-runs `gitnexus_impact` per the repo rule; treat the numbers above as a snapshot.

### 1.4 Transmitter-Specific Risk Map

The transmitter slice introduces no new HIGH-risk symbols beyond the receiver baseline. Net new additions are pure-additive listeners, services, and one `DeviceMqttPublisher` overload:


| Symbol                                                           | Risk   | Depends on                                             | Notes                                                                                                                                                                                                                   |
| ---------------------------------------------------------------- | ------ | ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `DeviceCanonical.transmitV1(...)`                                | LOW    | new helper                                             | Pure builder, golden-string unit-test only.                                                                                                                                                                             |
| `DeviceMqttPublisher.publishTransmit(...)`                       | LOW    | publisher                                              | Net-new method; QoS 1, **not retained**.                                                                                                                                                                                |
| `DeviceMqttPublisher.publishDeact(...)` (kind-aware route)       | LOW    | existing receiver method gains a `device.kind` switch  | Existing receiver path unchanged; hub variant adds the `transmitter/...` namespace.                                                                                                                                     |
| `DeviceRegistrationService.onRegister(...)`                      | MEDIUM | shared between receivers and hubs after generalisation | Phase D2 reshapes it to accept a `DeviceFamily` discriminator (Phase T1 is the second consumer). Receiver paths must keep refusing `TRANSMITTER_HUB`; transmitter paths must refuse anything else. Single-slice change. |
| `TransmitterDispatchService`                                     | LOW    | consumes `DeviceDispatchEventBroadcaster` from D7a     | New file. Must drop on slow subscriber so one stuck consumer never stalls dispatch.                                                                                                                                     |
| `TransmitterElectionService`                                     | LOW    | reads Redis liveness keys + `device` rows              | Lazy / on-dispatch; no scheduled work.                                                                                                                                                                                  |
| `TransmitterBootstrapListener`, `TransmitterOperationalListener` | LOW    | new MQTT subscriptions                                 | Both `@ConditionalOnProperty` on `device.transmitter.enabled`.                                                                                                                                                          |


The whole transmitter slice is gated behind `device.transmitter.enabled` (§6.2); flipping it off restores the receiver-only behaviour exactly.

---

## 2. Device Contract Summary

Authoritative sources: [RECEIVER_DESIGN_GUIDE.md](RECEIVER_DESIGN_GUIDE.md), [RECEIVER_ESP32C3_DESIGN_GUIDE.md](RECEIVER_ESP32C3_DESIGN_GUIDE.md), [TRANSMITTER_ESP32C3_DESIGN_GUIDE.md](TRANSMITTER_ESP32C3_DESIGN_GUIDE.md). If this summary disagrees with them, fix the guide first, then refresh this section.

### 2.1 MQTT Topics

**Receiver topics**


| Topic                                     | Direction        | QoS | Retained | Use                                                                |
| ----------------------------------------- | ---------------- | --- | -------- | ------------------------------------------------------------------ |
| `receiver/bootstrap/register`             | device → backend | 1   | no       | Registration payload                                               |
| `receiver/bootstrap/{challenge_id}`       | both             | 1   | no       | `pending`, `challenge`, `rejected`, `result`, `response` envelopes |
| `receiver/device/{public_id}/cmd/rf_code` | backend → device | 1   | yes      | Signed RF trigger code                                             |
| `receiver/device/{public_id}/cmd/deact`   | backend → device | 1   | yes      | Signed suspend / resume / decommission                             |
| `receiver/device/{public_id}/ack`         | device → backend | 1   | no       | Single ack stream, `ack_for` discriminator                         |


**Transmitter hub topics**


| Topic                                      | Direction     | QoS | Retained | Use                                                                |
| ------------------------------------------ | ------------- | --- | -------- | ------------------------------------------------------------------ |
| `transmitter/bootstrap/register`           | hub → backend | 1   | no       | Registration payload                                               |
| `transmitter/bootstrap/{challenge_id}`     | both          | 1   | no       | `pending`, `challenge`, `rejected`, `result`, `response` envelopes |
| `transmitter/hub/{public_id}/cmd/transmit` | backend → hub | 1   | **no**   | Signed dispatch (`transmit-v1`)                                    |
| `transmitter/hub/{public_id}/cmd/deact`    | backend → hub | 1   | yes      | Signed suspend / resume / decommission                             |
| `transmitter/hub/{public_id}/heartbeat`    | hub → backend | 0   | no       | 10 s cadence                                                       |
| `transmitter/hub/{public_id}/ack`          | hub → backend | 1   | no       | Single ack stream, `ack_for` discriminator                         |


`cmd/transmit` is intentionally **not retained**: the broker must never replay a stale broadcast on hub reconnect. `cmd/deact` is retained on both families so a device coming back online learns its current lifecycle state before processing fresh commands.

**Backend startup subscriptions** (via `MqttClientManager`):

- Always: `receiver/bootstrap/register`, `receiver/bootstrap/+`, `receiver/device/+/ack`.
- When `device.transmitter.enabled = true` (§6.2): also `transmitter/bootstrap/register`, `transmitter/bootstrap/+`, `transmitter/hub/+/ack`, `transmitter/hub/+/heartbeat`.

The receiver bootstrap listeners must keep explicit envelope-type dropping on `receiver/bootstrap/+`; the ESP-01 path remains MQTT 3.1.1-only, and the receiver flows do not rely on broker-side no-local semantics. For the transmitter path, the wildcard subscription preserves MQTT v5 `noLocal` behaviour across reconnects (§F0), and the explicit envelope-type drop stays in place as defensive validation if backend-owned types ever leak through.

### 2.2 On-Wire Names And Storage

The on-wire registration payload only ever appears for **MCU** receivers (ESP-01 / ESP32-C3) and the **transmitter hub** (ESP32-C3). Passive PT2272-class receivers have no firmware and no return channel — the admin enrolls them manually (§7, §D2.1).

MCU receiver registration payload uses:

- `hardware_model`: `ESP-01` or `ESP32-C3`
- `receiver_type`: `RECEIVER_433M` or `RECEIVER_2_4G`
- `firmware_version`: free-form string, persisted to `device.firmware_version`

Transmitter hub registration payload uses:

- `hardware_model`: `ESP32-C3`
- `kind`: literal `TRANSMITTER_HUB` (the discriminator field is `kind`, not `receiver_type`)
- `firmware_version`: free-form string, persisted to `device.firmware_version`

Backend storage:

- `device.hardware_model`: identifier-safe enum (`ESP_01`, `ESP32_C3`, `PT2272`) mapped from the payload value.
- `device.kind`: mapped from `receiver_type` for receivers, from the literal `kind` for hubs, or `RECEIVER_433M_PASSIVE` for passive registrations.

Legal pairs:


| Wire `hardware_model` | Stored enum | Legal `kind`                                        | Registered via                |
| --------------------- | ----------- | --------------------------------------------------- | ----------------------------- |
| `ESP-01`              | `ESP_01`    | `RECEIVER_433M`                                     | MQTT bootstrap (§D2)          |
| `ESP32-C3`            | `ESP32_C3`  | `RECEIVER_433M`, `RECEIVER_2_4G`, `TRANSMITTER_HUB` | MQTT bootstrap (§D2 / §T1)    |
| `PT2272`              | `PT2272`    | `RECEIVER_433M_PASSIVE`                             | Admin manual register (§D2.1) |


`PT2272` covers any pin-compatible non-MCU 433 MHz decoder family (HS1527, HX2272, etc.). A wholly different non-MCU chip family would get its own enum value and its own row here.

Do not persist dash-containing hardware labels directly: Kotlin enum constants and the repo's existing R2DBC `EnumCodec` pattern expect DB enum labels to match Kotlin enum names. DTO/request parsing owns the wire-label mapping.

### 2.3 Canonical Strings

UTF-8, no trailing newline, exact field order. Builders live in `core/device/DeviceCanonical.kt` and stay pure.

```text
activate-v1|<challenge_id>|<nonce>|<issued_at>|<expires_at>
rf-code-v1|<public_id>|<code_version>|<rf_code_hex>|<rf_code_bits>|<issued_at>
deact-v1|<public_id>|<command_id>|<action>|<issued_at>
transmit-v1|<hub_public_id>|<dispatch_id>|<receiver_public_id>|<band>|<rf_code_hex>|<rf_code_bits>|<issued_at>
```

Rules:

- `issued_at` / `expires_at` are ISO-8601 UTC ending in `Z`.
- `rf_code_hex` is uppercase hex.
- Integer fields (e.g. `rf_code_bits`) are base-10 without prefixes.
- `dispatch_id` is a UUID minted by the backend per dispatch event so the hub can dedupe replays.
- `band ∈ { "433M", "2_4G" }`. Populated from `device.kind` of the destination receiver:
  - `RECEIVER_433M` and `RECEIVER_433M_PASSIVE` → `"433M"`.
  - `RECEIVER_2_4G` → `"2_4G"`.

Each builder ships with a golden-string unit test pinned against the example payloads in the firmware guides (`activate-v1`/`rf-code-v1`/`deact-v1` in [RECEIVER_ESP32C3_DESIGN_GUIDE.md](RECEIVER_ESP32C3_DESIGN_GUIDE.md) §E.3; `transmit-v1` in [TRANSMITTER_ESP32C3_DESIGN_GUIDE.md](TRANSMITTER_ESP32C3_DESIGN_GUIDE.md) §E.3).

**Why `band` is on the wire even though `bits` is now disjoint per band.** The §2.4 width rules make `bits` disjoint (40 only on 2.4G; 1..32 only on 433M), so `band` is no longer load-bearing for disambiguation. It stays on the wire because (1) it lets the hub firmware route to the right radio without back-deriving from `bits`, (2) it makes the signed canonical string self-describing for audit and replay analysis, and (3) it leaves room for future widths without re-cutting the wire format. This is not a contract break against an existing firmware: no transmitter firmware is in production yet, so `band` lands as part of `transmit-v1` from day one rather than a future `transmit-v2`.

### 2.4 RF-Code Validation

Validate before signing or persisting. The bit-width rules reflect what each receiver family physically uses on the air:


| `kind`                  | `rf_code_bits` | Constraints                                     | Source of code                    | What the bytes mean on air                                                                   |
| ----------------------- | -------------- | ----------------------------------------------- | --------------------------------- | -------------------------------------------------------------------------------------------- |
| `RECEIVER_433M`         | 1..32          | `hex_len == 2 * ceil(bits/8)`                   | Backend `SecureRandom` (§D5.1)    | RC-Switch payload value matched at the receiver MCU                                          |
| `RECEIVER_433M_PASSIVE` | 1..24          | `hex_len == 2 * ceil(bits/8)`, **hardware cap** | Admin enters preset value (§D5.2) | PT2272 wire value matched in silicon                                                         |
| `RECEIVER_2_4G`         | **exactly 40** | `hex_len == 10`                                 | Backend `SecureRandom` (§D5.1)    | nRF24 5-byte `RX_ADDR_P1` — silicon-level identity, **not** an application-layer match value |


Defaults for backend-generated codes (auto-issued at activation) live in §6.1: `default-bits-433m = 32`, `default-bits-24g = 40`. PT2272 has no default — admins always supply the preset because it's hardware-fixed at the chip.

**PT2272 24-bit hardware guard.** The 24-bit cap on `RECEIVER_433M_PASSIVE` is a property of the chip itself: PT2272-class decoders encode 12 tri-state symbols, each represented by 2 bits, for a 24-bit total. The cap therefore must be enforced as an authoritative server-side checkpoint, not just a UI nicety: a request with `bits > 24` on a passive registration is rejected before any DB write with a dedicated `pt2272_hardware_cap_exceeded` error so the UI can show a hardware-specific message rather than the generic width error.

**PT2272-compatible 433 MHz protocol subset.** For passive PT2272 and pin-compatible decoder ICs, the matching transmitter-side subset is protocol IDs **1..4** from `transmitter/main/rf/rf_data.c`. All four entries keep the same non-inverted PT2272-style waveform (`sync 1:31`, `zero 1:3`, `one 3:1`) and differ only by base pulse length (`350`, `320`, `240`, `150` µs). Protocol 5+ remain available in the RF engine for other 433 MHz families but are outside the PT2272-compatible contract.

**RECEIVER_2_4G fixed at 40 bits.** nRF24L01 / nRF24L01+ silicon supports 3-, 4-, or 5-byte addresses via `SETUP_AW`, but 5 bytes is the chip default and the datasheet flags shorter widths as more noise-prone (PS v1.0 §7.3.2 and PS v2.0 §7.3.2). The plan locks `bits = 40` so the validator, the forbidden-set, and the `device_rf_code` schema CHECK can all stay simple. A request with any other `bits` for `RECEIVER_2_4G` is rejected before any DB write with `width_out_of_range`. The 40-bit value is written verbatim to the receiver's `RX_ADDR_P1` (LSByte first on the wire) at activation time and re-applied to the silicon on every rotation — see §D5.1.

**2.4G-only toggle payload.** The fixed payload `TOGGLE_MAGIC = 0xAA 0x55` exists only on the `RECEIVER_2_4G` path. There the transmitter hub sends the magic bytes to the receiver's nRF24 address, and `rf_trigger_on_packet` matches on that magic (defense-in-depth against improbable CRC-pass-by-noise) before toggling the vibrator. On the 433 MHz path there is no separate toggle payload: the transmitted RF frame itself is the `rf_code`, and the normal 433 MHz receiver-side check compares the decoded value against the stored `rf_code`. The magic is firmware-defined and never appears on the wire of any backend/admin surface outside the 2.4 GHz dispatch path.

Hubs never get a `device_rf_code` row — they have no per-device code; they execute whatever the dispatch event tells them to.

### 2.5 Transmitter Topology Rules

These rules are pinned in [TRANSMITTER_ESP32C3_DESIGN_GUIDE.md](TRANSMITTER_ESP32C3_DESIGN_GUIDE.md) §H.1 and are the load-bearing pre-conditions behind the implementation phases below. They live here so the receiver-side phases can reference them without re-stating.

- **MQTT-driven, not autonomous.** A hub does nothing on its own. It subscribes to `transmitter/hub/{public_id}/cmd/transmit` and broadcasts the contained RF code on receipt. No RF activity happens without a backend dispatch event.
- **Heartbeat-required.** Each hub publishes a periodic heartbeat (10 s interval, 30 s liveness window) on `transmitter/hub/{public_id}/heartbeat`. The backend updates `device.last_seen_at` and bumps a Redis liveness key (§T3 / §4).
- **Per-store registered cap: 3 (ceiling, not floor).** A store may operate with **1**, **2**, or **3** registered hubs. The cap matters at registration time only: don't accept a 4th. A store can run with one hub for a stretch — for example, while waiting for a redundancy unit to ship — and that single hub serves as both active and the entire pool. There is no minimum.
- **Per-store active cap: 1.** With 0 registered: dispatch fails with `no_active_transmitter`. With 1 registered and live: that hub is the active. With 2–3 registered: election picks the hub with the most recent live heartbeat (ties broken by lowest `device.id`). When *no* registered hub is heartbeating, dispatch fails — admin sees the failure and triages, no silent queueing.
- **Failover is automatic.** When the cached primary stops heartbeating, the next dispatch event re-elects from live hubs. No promote/demote endpoint needed; admin `suspend` / `decommission` removes a hub from the candidate pool.
- **No simultaneous broadcast.** Because only one hub transmits per dispatch event, RF collisions between peer hubs are not a concern and the hubs need no inter-hub coordination protocol.

---

## 3. Backend Data Model

### 3.1 Final Schema Shape

Update `backend/src/main/resources/db/schema.sql` for fresh installs. Add a one-shot operator-run migration at:

```text
backend/src/main/resources/db/migrations/2026-04-25-device-domain.sql
```

The repo has no Flyway/Liquibase, and `bootJar` excludes `db/**`, so the migration ships separately and is not packaged in the prod JAR.

```sql
CREATE TYPE device_kind AS ENUM (
  'RECEIVER_433M',          -- MCU 433 MHz (ESP-01 / ESP32-C3); 1..32 bits
  'RECEIVER_433M_PASSIVE',  -- PT2272-class hardware decoder; 1..24 bits, admin-registered
  'RECEIVER_2_4G',          -- MCU 2.4 GHz (ESP32-C3); rf_code is the 5-byte nRF24 RX_ADDR_P1 (bits = 40)
  'TRANSMITTER_HUB'         -- ESP32-C3 hub, dual-radio (433 MHz + 2.4 GHz)
);

CREATE TYPE device_hardware_model AS ENUM (
  'ESP_01',
  'ESP32_C3',
  'PT2272'                  -- pin-compatible non-MCU 433 MHz decoder family
);

CREATE TYPE device_status AS ENUM (
  'PENDING',
  'PENDING_RF_CODE',
  'ACTIVE',
  'SUSPENDED',
  'DECOMMISSIONED',
  'REJECTED'
);

CREATE TYPE device_rf_ack_status AS ENUM (
  'PENDING',
  'APPLIED',
  'UNCHANGED',
  'REJECTED'
);

CREATE TABLE device (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  public_id             VARCHAR(32),
  public_key_der        BYTEA,
  hardware_model        device_hardware_model NOT NULL,
  kind                  device_kind NOT NULL,
  status                device_status NOT NULL DEFAULT 'PENDING',
  assigned_name         VARCHAR(100),
  store_id              UUID REFERENCES store(id) ON DELETE SET NULL,
  firmware_version      VARCHAR(32),
  last_seen_at          TIMESTAMPTZ,
  activated_at          TIMESTAMPTZ,
  created_at            TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at            TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE UNIQUE INDEX idx_device_public_id
  ON device(public_id) WHERE public_id IS NOT NULL;

CREATE UNIQUE INDEX idx_device_public_key_der
  ON device(public_key_der) WHERE public_key_der IS NOT NULL;

CREATE INDEX idx_device_store ON device(store_id);
CREATE INDEX idx_device_status ON device(status);
CREATE INDEX idx_device_kind ON device(kind);

-- Plaintext payload encoding (firmware/backend MUST agree). Only
-- ciphertext is stored; plaintext meaning by kind:
--   RECEIVER_433M / RECEIVER_433M_PASSIVE: big-endian unsigned int,
--     left-padded to byte_len = ceil(bits/8); only the bottom `bits`
--     LSBs are significant. Width: bits ∈ [1, 32] (433M) or [1, 24]
--     (433M_PASSIVE).
--   RECEIVER_2_4G: 5-byte nRF24 address (SETUP_AW = 11). bits = 40,
--     byte_len = 5 enforced by the kind-aware CHECK below. Bytes go on
--     the wire LSByte first (PS v1.0 §8.3.1).
-- TRANSMITTER_HUB never gets a device_rf_code row — hubs have no
-- per-device code; the dispatch event carries it per-broadcast.
CREATE TABLE device_rf_code (
  device_id      UUID PRIMARY KEY REFERENCES device(id) ON DELETE CASCADE,
  payload        BYTEA    NOT NULL,                       -- pgp_sym_encrypt_bytea(plaintext, device.rf-code.encryption-key)
  bits           SMALLINT NOT NULL,                       -- see kind-aware constraint below
  byte_len       SMALLINT NOT NULL,                       -- 1..32
  version        INT      NOT NULL,
  issued_at      TIMESTAMPTZ NOT NULL,
  ack            device_rf_ack_status NOT NULL DEFAULT 'PENDING',
  ack_at         TIMESTAMPTZ,

  CONSTRAINT chk_rf_bits      CHECK (bits     BETWEEN 1 AND 256),
  CONSTRAINT chk_rf_byte_len  CHECK (byte_len BETWEEN 1 AND 32),
  CONSTRAINT chk_rf_byte_math CHECK (byte_len = (bits + 7) / 8)
);

-- Width constraint scoped by device kind. RECEIVER_2_4G stores a 5-byte
-- nRF24 address (bits = 40) and nothing else; 433M variants keep their
-- existing 1..32 / 1..24 ranges. TRANSMITTER_HUB is rejected outright —
-- no row of this table should ever describe a hub. The trigger fires on
-- insert and update; the device_rf_code row is created in the same
-- transaction as the device row (D2.1) or after activation (D5.1), so
-- the kind lookup is always satisfiable.
CREATE OR REPLACE FUNCTION device_rf_code_kind_check() RETURNS TRIGGER AS $$
DECLARE
  k device_kind;
BEGIN
  SELECT kind INTO k FROM device WHERE id = NEW.device_id;
  IF k = 'TRANSMITTER_HUB' THEN
    RAISE EXCEPTION 'TRANSMITTER_HUB never owns a device_rf_code row';
  ELSIF k = 'RECEIVER_2_4G' AND (NEW.bits <> 40 OR NEW.byte_len <> 5) THEN
    RAISE EXCEPTION 'RECEIVER_2_4G requires bits = 40 and byte_len = 5';
  ELSIF k = 'RECEIVER_433M' AND NEW.bits NOT BETWEEN 1 AND 32 THEN
    RAISE EXCEPTION 'RECEIVER_433M requires bits BETWEEN 1 AND 32';
  ELSIF k = 'RECEIVER_433M_PASSIVE' AND NEW.bits NOT BETWEEN 1 AND 24 THEN
    RAISE EXCEPTION 'RECEIVER_433M_PASSIVE requires bits BETWEEN 1 AND 24';
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_device_rf_code_kind_check
  BEFORE INSERT OR UPDATE ON device_rf_code
  FOR EACH ROW EXECUTE FUNCTION device_rf_code_kind_check();
```

Rules:

- Root `device` table holds durable identity, assignment, status, timestamps. Nothing transient.
- The `(hardware_model, kind)` pairing is enforced in the application validator, not the DB — the DB enum constrains hardware families; per-pair legality varies with deployment intent.
- Schema-level `chk_rf_*` constraints enforce the universal envelope (1..256 bits, byte math). Per-kind bit-width rules from §2.4 live in `RfCodeValidator` because they need `device.kind`. Hub rejection lives in the trigger.
- Both 433 MHz and 2.4 GHz codes share the same row shape; `bits` / `byte_len` differ. Splitting into two tables would duplicate every read path and join for no behavioural gain.
- Update `R2DBCConfig` with `EnumCodec` entries and `EnumWriteSupport` converters for `device_kind`, `device_hardware_model`, `device_status`, and `device_rf_ack_status`, matching the existing `admin_role` pattern.
- Update `analytics_event.device_id` in `schema.sql` to reference `device(id) ON DELETE SET NULL`; it currently references the legacy `notifier_device(id)`.
- **Transmitter hubs add no new tables.** Heartbeat liveness lives in Redis (§4); election cache lives in Redis. The only transmitter-specific durable column is `device.last_seen_at`, which already exists.

### 3.2 Migration Rules

The deployed `notifier_device` table is an old FCM-token-shaped placeholder. Kotlin code does not use it today; `analytics_event.device_id` is its only known FK.

Migration steps (one BEGIN / COMMIT, idempotent — every step must be safe to rerun without data loss after passive devices or hubs exist):

1. Wrap in `BEGIN` / `COMMIT`.
2. Create enum types with duplicate-object guards (`DO $$ BEGIN ... EXCEPTION WHEN duplicate_object ...`).
3. Rename `notifier_device` → `device` only if `device` does not already exist.
4. Drop legacy FCM columns with `ALTER TABLE device DROP COLUMN IF EXISTS device_token, DROP COLUMN IF EXISTS battery_level, DROP COLUMN IF EXISTS last_ping, DROP COLUMN IF EXISTS is_active;` so a rerun after the columns are gone is a no-op.
5. Rename `name` → `assigned_name` only if `name` exists and `assigned_name` does not (guard with an `information_schema.columns` `DO` block).
6. Add new columns with `ADD COLUMN IF NOT EXISTS` (nullable / defaulted): `public_id`, `public_key_der`, `hardware_model`, `kind`, `status`, `firmware_version`, `last_seen_at`, `activated_at`, `updated_at`.
7. Drop legacy rows that cannot be represented as receivers or hubs. Use the **new** required columns as the legacy discriminator — passive PT2272 rows legitimately have `public_key_der IS NULL`, so that column must not appear in this filter:
  ```sql
   DELETE FROM device
   WHERE kind IS NULL OR hardware_model IS NULL;
  ```
   On rerun, every surviving row already has both columns populated, so this DELETE matches nothing.
8. Replace the old `store_id` FK (`ON DELETE CASCADE`) with `ON DELETE SET NULL` — the old action would delete device audit history when a store is removed. Drop the old constraint with `IF EXISTS` before recreating.
9. Recreate `analytics_event.device_id` so it references `device(id) ON DELETE SET NULL` after the rename. Drop the old constraint with `IF EXISTS` first; this step runs unconditionally so a rerun against a `device`-renamed schema still repairs the FK.
10. Set `hardware_model` and `kind` to `NOT NULL`. Wrap in a `DO` block that checks `is_nullable` in `information_schema.columns` so the rerun is a no-op.
11. Create `device_rf_code` with `CREATE TABLE IF NOT EXISTS`.
12. Create the `device_rf_code_kind_check` trigger function with `CREATE OR REPLACE FUNCTION` and the trigger with `DROP TRIGGER IF EXISTS … ; CREATE TRIGGER …` so the hub-rejection clause can be patched in on rerun without orphaning the old version.
13. Recreate indexes with `IF NOT EXISTS`.

Operator command:

```bash
psql "$DATABASE_URL" -f backend/src/main/resources/db/migrations/2026-04-25-device-domain.sql
```

Rollback is snapshot-based: the migration intentionally drops FCM-shaped columns that have no current consumers.

---

## 4. Redis Keyspace

Add only new keys to `RedisKeyManager`; do not rename existing ones.

```kotlin
// Receiver-domain keys (Phase F0)
fun enrollmentToken(sha256Hex: String) = "enroll:$sha256Hex"
fun enrollmentTokenPattern() = "enroll:*"
fun deviceActivation(challengeId: UUID) = "device:activation:$challengeId"
fun deviceActivationByDevice(deviceId: UUID) = "device:activation-by-device:$deviceId"
fun deviceLifecycleCommand(deviceId: UUID) = "device:lifecycle:$deviceId"
fun deviceBusy(deviceId: UUID) = "device:busy:$deviceId"

// Transmitter-domain keys (Phases T3 / T4)
fun deviceHubAlive(deviceId: UUID) = "device:hub:alive:$deviceId"
fun storeTransmitterActive(storeId: UUID) = "store:$storeId:transmitter:active"
```

Values are JSON strings (Redis is `ReactiveRedisTemplate<String, String>`).


| Key                                      | Value                                                                                     | TTL                                                         | Owner phase |
| ---------------------------------------- | ----------------------------------------------------------------------------------------- | ----------------------------------------------------------- | ----------- |
| `enroll:{sha256}`                        | `{storeId, issuedByAdminId, issuedAt, expiresAt}`                                         | `device.enrollment.token-ttl-seconds`, default 3600         | F1          |
| `device:activation:{challengeId}`        | `{deviceId, publicKeyFingerprint, registrationNonce, nonce, issuedAt, expiresAt, status}` | 15 minutes                                                  | F2 / T1     |
| `device:activation-by-device:{deviceId}` | `{challengeId}`                                                                           | matches activation key                                      | F2 / T1     |
| `device:lifecycle:{deviceId}`            | `{commandId, action, issuedAt, ackStatus}`                                                | 30 minutes                                                  | F6 / T6     |
| `device:busy:{deviceId}`                 | `{storeId, ticketId, boundAt}`                                                            | mirrors ticket lifecycle (§D7a)                             | D7a         |
| `device:hub:alive:{deviceId}`            | `"1"`                                                                                     | `device.transmitter.heartbeat-liveness-seconds`, default 30 | T3          |
| `store:{storeId}:transmitter:active`     | `{deviceId, electedAt}`                                                                   | `device.transmitter.active-cache-seconds`, default 60       | T4          |


Token rules:

- Generate 128 bits with `SecureRandom`, encode as base64url without padding.
- Hash the exact trimmed token using SHA-256 hex (no case folding).
- Issue with `SET ... EX ... NX`.
- Consume with `GETDEL`.
- Plaintext tokens never enter Postgres; admin web shows them once.

Transmitter Redis rules:

- `device:hub:alive:{deviceId}` is bumped on every heartbeat ingest in `TransmitterOperationalListener` (§T3); its TTL of 30 s is exactly 3× the firmware-side 10 s heartbeat cadence (§E.5 of the transmitter design guide), so a single missed packet doesn't drop a hub from the candidate pool.
- `store:{storeId}:transmitter:active` is written by `TransmitterElectionService` (§T4) only when a fresh election runs. Suspending or decommissioning the cached primary deletes this key so the next dispatch re-elects.

Ticket binding rule: add `device_id` to the existing ticket hash at `RedisKeyManager.ticket(storeId, ticketId)` (§D7a). Do not introduce a parallel ticket key format.

---

## 5. Backend Package Layout

```text
backend/src/main/kotlin/com/thomas/notiguide/
├── core/device/
│   ├── DeviceCanonical.kt                # activate-v1, rf-code-v1, deact-v1, transmit-v1 builders
│   ├── DeviceCommandSigner.kt
│   ├── DeviceCommandSigningConfig.kt
│   ├── DeviceCommandSigningProperties.kt
│   ├── DeviceMqttPublisher.kt            # publishes to receiver/... and transmitter/... topics
│   └── DevicePublicIdMinter.kt
└── domain/device/
    ├── controller/
    │   ├── DeviceAdminController.kt          # /api/devices/** except enrollment-tokens
    │   └── EnrollmentTokenController.kt      # /api/devices/enrollment-tokens/**
    ├── dto/
    ├── entity/
    │   ├── Device.kt
    │   └── DeviceRfCode.kt
    ├── listener/
    │   ├── DeviceBootstrapListener.kt        # receiver/bootstrap/{register,+}
    │   ├── DeviceOperationalListener.kt      # receiver/device/+/ack
    │   ├── TransmitterBootstrapListener.kt   # transmitter/bootstrap/{register,+}     (§T1)
    │   └── TransmitterOperationalListener.kt # transmitter/hub/+/{heartbeat,ack}      (§T3)
    ├── repository/
    │   ├── DeviceRepository.kt
    │   └── DeviceRfCodeRepository.kt
    ├── request/
    ├── service/
    │   ├── DeviceActivationService.kt
    │   ├── DeviceApprovalService.kt
    │   ├── DeviceDispatchService.kt          # producer for in-process dispatch events (§D7a)
    │   ├── DeviceLifecycleService.kt
    │   ├── DeviceRegistrationService.kt      # MCU bootstrap intake; family-aware after Phase D2
    │   ├── PassiveDeviceRegistrationService.kt   # PT2272 manual register (§D2.1)
    │   ├── EnrollmentTokenService.kt
    │   ├── RfCodeForbiddenSet.kt             # §D5.0
    │   ├── RfCodeService.kt
    │   ├── TransmitterDispatchService.kt     # broadcaster consumer → cmd/transmit    (§T5)
    │   └── TransmitterElectionService.kt     # active-hub election, Redis-cached      (§T4)
    └── types/
        ├── DeviceFamily.kt                   # RECEIVER | TRANSMITTER (§D2 / §T1)
        ├── DeviceHardwareModel.kt
        ├── DeviceKind.kt
        ├── DeviceLifecycleAckStatus.kt
        ├── DeviceRfAckStatus.kt
        └── DeviceStatus.kt
```

`DeviceHardwareModel` is shared with the transmitter hub (which reuses `ESP32-C3`); no extra hardware-model enum value is needed for that family. Its enum constants are `ESP_01`, `ESP32_C3`, and `PT2272`; request/response DTOs map these to the wire labels `ESP-01`, `ESP32-C3`, and `PT2272`.

`DeviceFamily` is a discriminator inferred from the bootstrap topic namespace (`receiver/...` → `RECEIVER`, `transmitter/...` → `TRANSMITTER`). It exists primarily so `DeviceRegistrationService.onRegister` can refuse cross-family payloads (e.g. a `kind: TRANSMITTER_HUB` arriving on `receiver/bootstrap/register` is rejected, and vice versa). The firmware never sees the discriminator on the wire.

---

## 6. Backend Configuration

### 6.1 Receiver / Shared Properties

Add to `application.yaml`. Five knobs, each with one job:

```yaml
device:
  command-signing:
    # PEM-encoded EC P-256 private key for SHA256withECDSA. Spring
    # resource location: `classpath:...` for dev or `file:/abs/path.pem`
    # for prod-mounted secrets. Prefix with `base64:` to inline base64-
    # encoded PEM contents (useful for env-only platforms). Shared by
    # the receiver and transmitter fleets — one signing key covers both.
    pk: ${DEVICE_CMD_SIGNING_PK:}
  rf-code:
    # Symmetric passphrase for `pgp_sym_encrypt_bytea` over plaintext RF
    # payload bytes. Required whenever any receiver is active.
    encryption-key: ${DEVICE_RF_CODE_ENCRYPTION_KEY:}
    # Auto-generated initial code widths.
    # 433M defaults at its 32-bit cap (max entropy on the band).
    # 2.4G is fixed at 40 bits — the nRF24 RX_ADDR_P1 width. The rf_code
    # for a 2.4G receiver IS its silicon-level address; 5 bytes is the
    # chip default (SETUP_AW = 11) and shorter widths are noise-prone
    # per the datasheet (§2.4). The width is intentionally not
    # config-tunable.
    default-bits-433m: 32
    default-bits-24g: 40
  enrollment:
    token-ttl-seconds: ${DEVICE_ENROLLMENT_TTL_SECONDS:3600}
```

Dev profile may use `pk: classpath:device/cmd_signing_priv.pem`.

Implementation rules:

- Bind `DeviceCommandSigningProperties` and RF-code properties unconditionally, but do not fail app startup just because `pk` or `encryption-key` is blank. APIs that need signing, MQTT publishing, or RF-code encryption return `503 Service Unavailable` with a structured error when the required dependency/config is absent; read-only list/detail endpoints still work.
- Create `DeviceCommandSigner` only when `pk` is non-blank, then fail fast if that provided key cannot be loaded as an EC P-256 private key. Do not stack multi-name `@ConditionalOnProperty` on the config — Spring Boot 3.5 evaluates multiple `name` entries with AND semantics ("all properties must pass the test for the condition to match", per the [`@ConditionalOnProperty` reference](https://docs.spring.io/spring-boot/3.5/api/java/org/springframework/boot/autoconfigure/condition/ConditionalOnProperty.html)). Use a single annotation per condition.
- Resolve `pk` via `ResourceLoader.getResource(pk).inputStream` then `PemContent.load(inputStream).getPrivateKey()`. Use explicit resource prefixes: `classpath:` for dev resources and `file:` for mounted files. Bare absolute paths are not part of the contract. If the value starts with `base64:`, decode the body to text and use `PemContent.of(text).getPrivateKey()`.
- Add `**/resources/device/**` to `backend/.gitignore` and `"device/**"` to the existing `bootJar.exclude(...)` list so a dev key never ships in the prod JAR.
- `DeviceMqttPublisher` and MQTT listeners are `@ConditionalOnBean(MqttClientManager::class)`. If MQTT or signing is missing, admin APIs that need them return `503 Service Unavailable` with a clear error code; token listing and device listing still work.

RF-code encryption boundary:

- `DeviceRfCodeRepository` encrypts binary payloads via `pgp_sym_encrypt_bytea(plaintext, <device.rf-code.encryption-key>)` on write and decrypts via `pgp_sym_decrypt_bytea(...)` only for operational publish/dispatch paths.
- Plaintext may exist only in short-lived service memory while generating/signing/publishing an RF code or while dispatching through a transmitter hub. It never appears in admin DTOs, logs, MQTT acks, or SSE events.
- Admin DTOs may surface `bits`, `byte_len`, `version`, `ack`, `issued_at` — never the ciphertext or plaintext.

### 6.2 Transmitter Properties

Add to `application.yaml`, alongside the `device:` block:

```yaml
device:
  transmitter:
    enabled: ${DEVICE_TRANSMITTER_ENABLED:false}     # registers transmitter listeners + dispatch consumer when true
    heartbeat-interval-seconds: 10                   # informational; firmware default
    heartbeat-liveness-seconds: 30                   # Redis TTL for device:hub:alive:*
    active-cache-seconds: 60                         # Redis TTL for store:{id}:transmitter:active
    max-registered-per-store: 3                      # ceiling, not floor (§2.5)
```

Bind via `DeviceTransmitterProperties` `@ConfigurationProperties` bean.

`enabled = false` keeps the transmitter listeners and dispatch consumer **unregistered**, so the backend never subscribes to transmitter topics and the queue-side dispatch service surfaces `no_active_transmitter` immediately. `enabled = true` registers those beans on startup.

`heartbeat-`* and `active-cache-*` are deliberate hardcoded ceilings, config-readable for ops visibility but not per-store overridable. Bending them requires firmware coordination, so they belong in YAML, not in `store_settings`.

All transmitter listeners and `TransmitterDispatchService` are annotated:

```kotlin
@ConditionalOnProperty(prefix = "device.transmitter", name = ["enabled"], havingValue = "true")
```

This uses a single `name` entry, so the Spring Boot 3.5 AND/OR distinction is informational rather than load-bearing here.

---

## 7. Backend API

All routes require an authenticated admin. SUPER_ADMIN can act across stores; ADMIN is store-scoped via `StoreAccessUtil` and `principal.storeId`.

`DeviceAdminController` (`/api/devices/**` except enrollment tokens):

```text
GET    /api/devices                          # supports ?kind=... &storeId=... filters; includes registered count for the requested kind/store (§T7)
GET    /api/devices/{id}
POST   /api/devices/passive                # MCU-less PT2272 manual register (§D2.1)
POST   /api/devices/{id}/approve
POST   /api/devices/{id}/reject
POST   /api/devices/{id}/rf-code            # rotation only; 400 for RECEIVER_433M_PASSIVE and TRANSMITTER_HUB
POST   /api/devices/{id}/lifecycle          # body: {action: "suspend"|"resume"|"decommission"}
POST   /api/devices/{id}/reprovision        # 404 for RECEIVER_433M_PASSIVE
```

`EnrollmentTokenController` (`/api/devices/enrollment-tokens/**`):

```text
POST   /api/devices/enrollment-tokens
GET    /api/devices/enrollment-tokens
DELETE /api/devices/enrollment-tokens/{sha256}
```

Per-kind behaviour notes:

- `POST /api/devices/{id}/rf-code` returns `400 Bad Request {"error":"hardware_fixed_code"}` for `RECEIVER_433M_PASSIVE` (§D5.2) and `400 Bad Request {"error":"hub_no_rf_code"}` for `TRANSMITTER_HUB` (hubs own no per-device code).
- `POST /api/devices/{id}/reprovision` returns `404 Not Found` for `RECEIVER_433M_PASSIVE`. For MCU receivers and transmitter hubs it resets backend operational state so the device's next bootstrap is treated as a fresh activation: clear `device.public_id`, `activated_at`, and (receivers only) the `device_rf_code` row; set `status = PENDING`. The retained-topic cleanup that prevents stale broadcasts replaying against a brand-new `public_id` happens at the *next* activation transaction (see §D4 / §T2 step 9 — `priorPublicId != null` triggers `clearRetained` on the appropriate topic family). The reprovision endpoint itself does not publish over MQTT.
- `GET /api/devices` accepts a `kind` filter; when the caller passes `kind=TRANSMITTER_HUB&storeId=X` the response body includes a `registered` count (across `status NOT IN ('DECOMMISSIONED', 'REJECTED')`) so the admin web can render `2 / 3 hubs registered` (§T7). The count is computed in the same query the listing already runs.

`POST /api/devices/passive` body:

```json
{
  "hardwareModel": "PT2272",
  "kind": "RECEIVER_433M_PASSIVE",
  "assignedName": "Front gate fob #3",
  "storeId": "<UUID>",
  "rfCodeHex": "0A1B2C",
  "rfCodeBits": 24
}
```

Server flow (§D2.1 spells out the full sequence): validate the `(hardwareModel, kind)` pair, validate width per §2.4, parse hex to plaintext, run §D5.0 forbidden-set + uniqueness checks, transactionally INSERT `device` (`status = ACTIVE`, synthetic `pas-XXXXX` `public_id`) and `device_rf_code` (`version = 1`, `ack = APPLIED`). No MQTT, no signing, no enrollment token, no approval — admin authority *is* the approval. Returns the new `DeviceDto`.

Queue dispatch lives under existing queue admin routes (handled by `DeviceDispatchService` but exposed via `QueueAdminController`):

```text
POST /api/queue/admin/{storeId}/device-tickets
GET  /api/queue/admin/{storeId}/available-devices
```

`GET /api/queue/admin/{storeId}/available-devices` returns a preflight envelope rather than a bare array:

```json
{
  "devices": [],
  "dispatchReady": false,
  "error": "no_active_transmitter"
}
```

`devices` contains only receiver rows that are `ACTIVE`, store-owned by the requested store, have a current `device_rf_code` row, and have no `device:busy:{deviceId}` key. Hubs are excluded by kind filter. `dispatchReady` tells the queue page whether a live hub is currently available; `error` is omitted when dispatch is ready. The endpoint still returns `200 OK` when `dispatchReady = false` so the UI can render the disabled reason without forcing a failing POST round-trip.

Update `API_ROUTES` in admin web accordingly (§9.5).

Rate limiting: change `RateLimitFilter.resolveTier` to accept both `method` and `path`, then route only `POST /api/devices/enrollment-tokens` to the strict tier. Token listing/revoke and all other device endpoints inherit the standard `/api/**` tier.

---

## 8. Backend Services

The plan groups work into `D` phases (receiver-shaped) and `T` phases (transmitter-shaped). Phases that can land together — because they share a slice of the codebase or unblock each other — are flagged below; the combined execution order lives in §10.

### Phase F0 — Foundation

1. Run GitNexus impact for every touched symbol before editing it (§1.3 / §1.4).
2. Update schema and add the migration (§3).
3. Add Redis keys (no renames) — both receiver and transmitter buckets (§4).
4. Add device config properties for both receiver (§6.1) and transmitter (§6.2). Bind both `@ConfigurationProperties` beans.
5. Add signing config (`DeviceCommandSigner`, `DeviceCommandSigningProperties`) and canonical helpers (`activate-v1`, `rf-code-v1`, `deact-v1`, `transmit-v1`) in `core/device/DeviceCanonical.kt`. Each builder ships with a golden-string unit test pinned against the firmware-guide examples.
6. Add `DevicePublicIdMinter`. Rules:
  - MCU receivers: `rcv-XXXXX` (Crockford base32).
  - Passive PT2272: `pas-XXXXX` (same alphabet, distinct prefix so the admin UI can tag "passive" from the ID alone).
  - Transmitter hub: `hub-XXXXX`.
  - Collision check is global over `device.public_id`, regardless of kind.
7. Add `DeviceMqttPublisher` with the receiver methods (`publishRfCode`, `publishDeact`, `clearRetained`) and the transmitter additions:
  - `publishTransmit(hubPublicId, payload)` — `transmitter/hub/{publicId}/cmd/transmit`, QoS 1, **not retained**.
  - `publishDeact(...)` becomes kind-aware: route to `transmitter/hub/{publicId}/cmd/deact` when `device.kind = TRANSMITTER_HUB`, else the existing receiver path.
  - `clearRetained(publicId, kind)` gains a hub variant that issues a zero-length retained publish on `transmitter/hub/{publicId}/cmd/deact` on decommission and on reactivation that mints a new `public_id`.
8. Extend `MqttClientManager` with a stored subscription descriptor/overload so `transmitter/bootstrap/+` can preserve MQTT v5 `noLocal` behaviour across reconnects. The current wrapper only exposes `subscribe(topicFilter, qos)`, so reconnect replay is not expressive enough yet.

### Phase D1 — Enrollment Tokens

An enrollment token is the authorization link between an admin and a physical device. When an admin issues a token, they are granting permission for exactly one device to register against their store. The person who provisions the hardware pastes the token into the device's SoftAP setup page; the device then presents it during MQTT bootstrap registration, and the backend consumes it atomically. This proves the bootstrap request was initiated by someone with admin-granted physical access — without the token, a device on the network cannot begin registration. Tokens are single-use, short-lived, and opaque to firmware.

`EnrollmentTokenService`:

- Issue with `SET enroll:{hash} ... EX ... NX`.
- Return `{token, tokenHash, expiresAt}` once.
- List metadata only: `{tokenHash, storeId, issuedAt, expiresAt}`.
- Revoke by hash.

Authority: SUPER_ADMIN may issue for any store or none; ADMIN may only issue for `principal.storeId` and is rejected if absent. Tokens are family-agnostic — the same token can activate either a receiver or a hub, since the bootstrap topic the firmware publishes on tells the backend which family the device is.

### Phase D2 — Bootstrap Registration (MCU receivers)

`DeviceBootstrapListener` routes:

- `receiver/bootstrap/register` → `DeviceRegistrationService.onRegister(payload, family = RECEIVER)`.
- `receiver/bootstrap/{cid}` with `type == "response"` → `DeviceActivationService.onResponse`.
- Backend-owned envelope types arriving on `receiver/bootstrap/+` are ignored (self-echo guard).

`DeviceRegistrationService.onRegister(payload, family)` is generalised at this phase so the §T1 transmitter listener can reuse it. The `family` discriminator is inferred from the topic namespace by the listener and is not on the wire. Receiver registration:

1. Validate `schema_version`, required fields, public-key DER/base64 shape, wire `hardware_model`, `receiver_type`, and `firmware_version`. Reject any payload whose `kind` resolves to `TRANSMITTER_HUB` (cross-family guard) or `RECEIVER_433M_PASSIVE` (passive devices never come in over MQTT).
2. Map wire `hardware_model` → `DeviceHardwareModel` and `receiver_type` → `DeviceKind`.
3. Consume the enrollment token via `GETDEL enroll:{hash}`.
4. If the token is missing, publish `rejected` with `invalid_token`.
5. Upsert by `public_key_der`. Persist `firmware_version` from the payload.
6. Set status to `PENDING`; on reprovision, clear operational fields and the old `device_rf_code` row.
7. Write `device:activation:{challengeId}` and `device:activation-by-device:{deviceId}`.
8. Publish `pending` on `receiver/bootstrap/{challengeId}`.

### Phase D2.1 — Manual Registration (Passive PT2272)

`PassiveDeviceRegistrationService.register(req, adminId)` for `POST /api/devices/passive`:

1. Authority check: SUPER_ADMIN may target any store; ADMIN must target `principal.storeId`.
2. Validate the `(hardwareModel, kind)` pair (§2.2). Reject anything that isn't `(PT2272, RECEIVER_433M_PASSIVE)`.
3. **PT2272 hardware-cap guard**: reject `bits > 24` with `400 Bad Request {"error":"pt2272_hardware_cap_exceeded","limit":24}` *before* any other validation — the cap is a hardware-physical property of the chip, not just a configuration. Then run the residual width checks: `bits >= 1` and `hex_len == 2 * ceil(bits/8)`. Both surface as `400 Bad Request {"error":"width_out_of_range"}`.
4. Parse `rfCodeHex` to plaintext bytes; run §D5.0 forbidden-set + uniqueness guards. Surface uniqueness collisions as `409 Conflict` with the colliding device's `public_id` (helps the admin diagnose without exposing the colliding code).
5. Mint a `pas-XXXXX` `public_id`.
6. Transactionally INSERT `device` (`status = ACTIVE`, `activated_at = now()`) and `device_rf_code` (`version = 1`, `ack = APPLIED`, `ack_at = now()` — no future ack is coming).
7. Return the new `DeviceDto`.

No MQTT, no enrollment token, no approval, no signing, no challenge. Skips D2 / D3 / D4 entirely.

### Phase T1 — Bootstrap Registration (Transmitter Hubs)

`TransmitterBootstrapListener` is `@ConditionalOnProperty` on `device.transmitter.enabled` (§6.2). Routes:

- `transmitter/bootstrap/register` → `DeviceRegistrationService.onRegister(payload, family = TRANSMITTER)`.
- `transmitter/bootstrap/{cid}` with `type == "response"` → `DeviceActivationService.onResponse` (the activation path is family-agnostic — see §T2).
- Backend-owned envelope types arriving on `transmitter/bootstrap/+` are ignored. The wildcard subscription preserves MQTT v5 `noLocal` behaviour (§F0) so the broker shouldn't loop the backend's own publishes back, but the explicit drop stays as defense-in-depth.

Transmitter-family registration in the generalised `onRegister`:

1. Validate `schema_version`, required fields, public-key DER/base64 shape, wire `hardware_model`, the literal `kind == "TRANSMITTER_HUB"`, and `firmware_version`. Reject anything else (including all `RECEIVER_`* values) — cross-family guard, mirror of the receiver-side check.
2. Map wire `hardware_model` → `DeviceHardwareModel` (only `ESP32-C3` is legal for hubs in v1).
3. Consume the enrollment token via `GETDEL enroll:{hash}`. Capture the token's `storeId` — it is the cap-check store for step 5 and the device's pre-assigned store for step 6.
4. If the token is missing, publish `rejected` with `invalid_token`.
5. **Per-store cap check before upsert.** When the token's `storeId` is non-null, count rows where:
  ```sql
   kind = 'TRANSMITTER_HUB'
     AND store_id = :tokenStoreId
     AND status NOT IN ('DECOMMISSIONED', 'REJECTED')
     AND public_key_der IS DISTINCT FROM :incomingPublicKeyDer
  ```
   The `IS DISTINCT FROM` clause excludes the device being re-registered (same key, different boot) so a legitimate re-bootstrap of an already-registered hub doesn't trip the cap. If `count >= device.transmitter.max-registered-per-store`, publish `rejected` with `reason: "hub_cap_reached"` and stop. Decommissioning a hub frees a slot.
   When the token's `storeId` is null (SUPER_ADMIN-issued, deliberately unscoped), skip the cap check here and defer it to admin approval (§D3 below — the approval handler re-runs this query against the targeted store before assigning).
6. Upsert by `public_key_der`. Persist `firmware_version` from the payload. **If the token carried a `storeId`, write it onto `device.store_id` at this step** so subsequent registrations that target the same store can see this PENDING row in step 5's count.
7. Set status to `PENDING`.
8. Write `device:activation:{challengeId}` and `device:activation-by-device:{deviceId}`.
9. Publish `pending` on `transmitter/bootstrap/{challengeId}`.

Pre-assigning `store_id` at step 6 (unlike receivers, which defer to D3) is required so step 5's cap check sees PENDING hubs targeting the same store. ADMIN tokens always carry the admin's store; SUPER_ADMIN tokens may be unscoped (`store_id = NULL`), in which case the cap check defers to D3's approval handler.

### Phase D3 — Admin Approval

Approve:

- Validate admin store access.
- Set `store_id` and `assigned_name`. For receivers and for transmitter hubs whose enrollment token was issued *without* a `storeId` (SUPER_ADMIN unscoped path — see §T1 step 5), this is the moment `store_id` is committed.
- For `TRANSMITTER_HUB` rows, re-run the per-store cap check from §T1 step 5 against the just-assigned `store_id` (with the same `IS DISTINCT FROM :selfPublicKeyDer` self-exclusion). Reject with `409 Conflict {"error":"hub_cap_reached"}` if the slot is gone — this catches both the deliberately-unscoped path *and* the race where a different hub took the last slot between this hub's registration and approval. If the hub's `store_id` was already pre-assigned at §T1 step 6 and the admin overrides to a different store, run the cap check against the new store too.
- Load the activation challenge via `device:activation-by-device:{deviceId}`. Do not scan `device:activation:`*.
- Generate a ≥128-bit challenge nonce.
- Set challenge status `ISSUED`, `issuedAt`, `expiresAt`.
- Publish `challenge` on the appropriate bootstrap topic namespace (receiver vs hub) — `DeviceMqttPublisher` infers the namespace from `device.kind`.

Reject:

- Publish `rejected` on the appropriate namespace, delete both activation keys, set `device.status = REJECTED`.
- Do not refund the consumed enrollment token.

### Phase D4 — Activation (receivers)

`DeviceActivationService.onResponse`:

1. Load challenge and device. Capture `priorPublicId = device.public_id` (may be null on first activation; non-null on reprovision) — step 6 overwrites the column, so the value must be held in a local before the transaction.
2. Reject if missing, expired, or not `ISSUED`.
3. Rebuild `activate-v1|...` byte-for-byte.
4. Verify ECDSA P-256 signature against `device.public_key_der`.
5. Mint a fresh `public_id`.
6. Transactionally set `public_id` (the new one), `status = PENDING_RF_CODE`, `activated_at`.
7. Delete activation keys.
8. Publish `result` with the new `public_id` and assigned name on `receiver/bootstrap/{challengeId}`.
9. If `priorPublicId != null`, clear retained `cmd/rf_code` and `cmd/deact` on `receiver/device/{priorPublicId}/...` so a stale broadcast can't replay against the new identity.
10. Issue the initial RF code via `RfCodeService.autoIssue(deviceId)`. When the receiver acks the current version as `APPLIED` or `UNCHANGED`, promote status to `ACTIVE`.

If the initial publish cannot happen because MQTT or signing is unavailable, leave status as `PENDING_RF_CODE` and expose a retry action in the RF-code panel.

### Phase T2 — Activation (transmitter hubs)

`DeviceActivationService.onResponse` is family-agnostic — the canonical `activate-v1` string and the ECDSA verify path don't care which family the device belongs to. Hubs differ only in:

- `DevicePublicIdMinter` mints `hub-XXXXX` (driven by `device.kind == TRANSMITTER_HUB`).
- After signature-verify, the activation transaction sets `status = ACTIVE` directly. **There is no `PENDING_RF_CODE` for hubs** — they own no per-device code; `RfCodeService.autoIssue` is **not** called.
- `DeviceMqttPublisher` publishes `result` on `transmitter/bootstrap/{challengeId}` based on `device.kind`.
- Reprovision-side retained-cleanup runs against `transmitter/hub/{priorPublicId}/cmd/deact` only — there's no `cmd/rf_code` on the hub topic family.

Phases D4 and T2 share one implementation; the in-source kind switch (`when (device.kind)` after the verify step) is the only family-aware logic.

### Phase D5 — RF Code (receivers only)

`RfCodeValidator` and the `device_rf_code` write path are shared across receiver kinds; what differs is who picks the plaintext and whether MQTT is involved. **Hubs have no path through this phase** — they own no `device_rf_code` row.

#### D5.0 — Forbidden-set + uniqueness guards (all receiver kinds)

Run on every plaintext value, backend-generated or admin-supplied:

- **Forbidden, 433M family** (`RECEIVER_433M`, `RECEIVER_433M_PASSIVE`): reject low-entropy values where every nibble of the bottom `bits` is identical, including `0x00…0` and `0xFF…F`. Add firmware-documented sentinel patterns only when the firmware decoder actually reserves them.
- **Forbidden, 2.4G** (the rf_code is the 5-byte `RX_ADDR_P1`, so these guards protect against nRF24-specific address pathologies the datasheets call out, not generic payload entropy):
  1. Reject `0x0000000000` and `0xFFFFFFFFFF` — addresses with zero level transitions.
  2. Reject continuous-toggle patterns `0x5555555555` and `0xAAAAAAAAAA` — they look like the preamble and "raise the Packet-Error-Rate" per PS v1.0 §7.3.2 / PS v2.0 §7.3.2.
  3. Reject any address with **at most one level shift across the 5 bytes** (e.g., `0x00FFFFFFFF`, `0x000000FFFF`, `0xFFFF000000`). The datasheet warns these "can often be detected in noise and can give a false detection."
  4. Reject any address whose first byte (LSByte on the wire — the byte that gets matched against the preamble first) has fewer than two `0→1`/`1→0` transitions, on the same noise-detection rationale.
- **Uniqueness, kind-aware:**
  - **433M band** (groups `RECEIVER_433M` with `RECEIVER_433M_PASSIVE`): scoped to **same store**, among `device.status ∈ {PENDING_RF_CODE, ACTIVE, SUSPENDED}`. One hub broadcast hits every co-store 433M receiver, but stores are RF-isolated by deployment — and the PT2272 hardware-fixed code space is small enough that cross-store reuse is the realistic norm, not the exception.
  - **2.4G band**: scoped **globally** (across all stores), same status set. The address space (~10¹² values for 5 bytes minus the forbidden set above) is enormous, so global uniqueness costs nothing to enforce, and it eliminates the entire class of "two adjacent stores collided over the air" mystery bugs at mint time. The query is `WHERE kind = 'RECEIVER_2_4G' AND status IN (...)` — no `store_id` clause.
- On collision: backend-generated codes (D5.1) regenerate up to 8 times before failing with `RF code space exhausted` (only realistic at extreme density on 433M; effectively impossible on 2.4G). Admin-supplied codes (D5.2) fail immediately with `409 Conflict` and the colliding device's `public_id`.

#### D5.1 — Backend auto-issue and rotation (MCU receivers)

Applies to `RECEIVER_433M` and `RECEIVER_2_4G`. `RfCodeService.autoIssue(deviceId)` and `RfCodeService.rotate(deviceId, request, adminId)`:

- Pick `bits`:
  - `RECEIVER_433M`: rotation requests carry an explicit `bits` (admin choice within §2.4); auto-issue uses `default-bits-433m`.
  - `RECEIVER_2_4G`: width is **fixed at 40** (§2.4). Rotation requests for 2.4G must omit `bits` or pass `40`; any other value is a `400 Bad Request {"error":"width_out_of_range"}`. Auto-issue ignores `default-bits-24g` if it is ever set to anything other than 40.
- Generate plaintext via `SecureRandom`, sized to `byte_len = ceil(bits/8)`:
  - `RECEIVER_433M`: random integer; high `(8·byte_len − bits)` bits masked to 0 so the on-wire value matches `bits`.
  - `RECEIVER_2_4G`: 5 random bytes verbatim. The first generated byte goes on the wire as the LSByte (datasheet §8.3.1) and is therefore the byte the receiver's preamble matcher sees first; if it fails the §D5.0 first-byte-transition guard, regenerate.
- Run `RfCodeValidator.validate(device.kind, bits, plaintext)` (§2.4) and §D5.0 guards.
- `INSERT` on first issue, `UPDATE` with `version = version + 1` on rotation. Same `(payload, bits, byte_len)` cluster in one transaction.
- Encrypt plaintext via `pgp_sym_encrypt_bytea` inside `DeviceRfCodeRepository`. Plaintext is not returned to admin surfaces; it is available only to the operational publish/dispatch path.
- Sign `rf-code-v1|public_id|version|HEX(plaintext)|bits|issued_at` and publish a retained `cmd/rf_code`.
- Set `ack = PENDING`, `ack_at = NULL` on every write.
- Ack handling: normalise lower-case MQTT statuses to uppercase `DeviceRfAckStatus`. Update `device_rf_code` only when `ack.code_version == device_rf_code.version`. Update `device.last_seen_at`. If current status is `PENDING_RF_CODE` and the ack is `APPLIED` or `UNCHANGED`, promote to `ACTIVE`. Out-of-order old-version acks are ignored without user-facing noise.

**Receiver-side rotation, 2.4G specific.** On `RECEIVER_2_4G`, applying a new rf_code requires rewriting `RX_ADDR_P1` in silicon (not just updating an in-RAM matcher). During the rewrite window plus MQTT round-trip, a dispatch to the new address can be missed until the receiver acks `APPLIED`. Acceptable because rotation is admin-driven and rare; the admin UI surfaces `ack = PENDING` so the operator knows not to dispatch through this device until it lands. Firmware implementation details live in the receiver design guide.

#### D5.2 — Admin-set code (passive PT2272)

Applies to `RECEIVER_433M_PASSIVE`, used at registration (D2.1) and **only** at registration — rotation is rejected because the chip is hardware-fixed.

- `POST /api/devices/{id}/rf-code` returns `400 Bad Request` for `RECEIVER_433M_PASSIVE` with body `{"error":"hardware_fixed_code","message":"PT2272 codes are set at the chip and cannot rotate. Replace the device or update the chip's DIP switches and re-register."}`.
- The registration path supplies plaintext directly: `RfCodeValidator.validate` plus §D5.0 guards run, then `INSERT` with `version = 1`, `ack = APPLIED`, `ack_at = now()` — recording the truthful state immediately because no acknowledgement is ever coming.
- Plaintext is encrypted on write with the same `encryption-key`. No MQTT publish, no signing. The transmitter reads the row at dispatch time exactly like an MCU code.
- Passive devices have no acks; `ack` stays `APPLIED` from the moment of registration.

### Phase D6 — Lifecycle (receivers)

`DeviceLifecycleService.issue(deviceId, action, adminId)`:

- Allowed actions: `suspend`, `resume`, `decommission`.
- Reject any command for a `DECOMMISSIONED` device.
- **Passive devices (`RECEIVER_433M_PASSIVE`)**: skip the MQTT path entirely. Flip `device.status` directly (`suspend → SUSPENDED`, `resume → ACTIVE`, `decommission → DECOMMISSIONED`) and return synchronously. Dispatch (§D7a) consults `device.status` before binding, so a SUSPENDED passive device is unreachable at the queue level even though the chip itself never stopped listening — that's the trade-off for owning a hardware decoder.
- **MCU receivers**: sign `deact-v1|public_id|command_id|action|issued_at`, publish retained `cmd/deact` on `receiver/device/{publicId}/cmd/deact`, write `device:lifecycle:{deviceId}` with `{commandId, action, issuedAt, ackStatus: "PENDING"}` and a 30-minute TTL. No lifecycle-command columns on the root `device` table.

Ack handling (MCU receivers only):

- Normalise lower-case MQTT statuses to uppercase `DeviceLifecycleAckStatus`.
- Load `device:lifecycle:{deviceId}` by public-id lookup. If missing, update `last_seen_at`, log, and stop.
- On `OK`: `suspend → SUSPENDED`, `resume → ACTIVE`, `decommission → DECOMMISSIONED`, then delete the lifecycle key.
- On `IGNORED` or `REJECTED`: update the lifecycle key with the ack status until TTL expiry so the admin detail page can show the recent result without making it durable schema.
- On decommission `OK`, clear retained `cmd/rf_code` and `cmd/deact` for the receiver topic.

### Phase T6 — Lifecycle (transmitter hubs)

`DeviceLifecycleService` reuses the same `deact-v1` canonical and the same Redis lifecycle key shape for hubs. Differences:

- `DeviceMqttPublisher.publishDeact` routes to `transmitter/hub/{publicId}/cmd/deact` when `device.kind = TRANSMITTER_HUB`.
- On `suspend` ack `OK`: in addition to the standard status flip, **invalidate the Redis election cache** `store:{storeId}:transmitter:active` so the next dispatch re-elects from live hubs.
- On `decommission` ack `OK`: status flip, delete the lifecycle key, clear retained `cmd/deact` on the hub topic, and invalidate the election cache. Decommissioning a hub also frees a slot in the per-store cap (§T1 step 5 counts non-terminal status only).
- Hubs respond to `cmd/deact` while in any of `ACTIVE`, `SUSPENDED`, `DECOMMISSIONED` (the firmware keeps the MQTT session up under suspend so a `resume` can land), so the receiver-side rule about rejecting commands to `DECOMMISSIONED` devices applies symmetrically: the backend refuses to issue any new command against a `DECOMMISSIONED` row.

The receiver-side ack handling code path is reused verbatim, keyed by `command_id` / `public_id`; the only branch is the cache invalidation above.

### Phase T3 — Operational Topic Ingestion (transmitter hubs)

`TransmitterOperationalListener` (`@ConditionalOnProperty` on `device.transmitter.enabled`) subscribes to:

- `transmitter/hub/+/heartbeat`
- `transmitter/hub/+/ack`

For each heartbeat frame:

1. Update `device.last_seen_at` for the matching `public_id` (one-row UPDATE; non-blocking, runs in a coroutine).
2. Bump the Redis liveness key:
  ```text
   device:hub:alive:{deviceId} → "1"  EX  device.transmitter.heartbeat-liveness-seconds
  ```
   The 30 s default is exactly 3× the firmware's 10 s heartbeat cadence (§E.5 of the transmitter design guide). A single missed packet does not drop a hub from the candidate pool; sustained silence does.

For each `ack_for = "transmit"` frame: update `device.last_seen_at`, match on `dispatch_id`, and log/drop stale or out-of-order acks. v1 keeps no durable transmit-ack history; the structured log line is sufficient for admin triage.

For each `ack_for = "deact"` frame: hand off to the shared lifecycle-ack handler defined in §D6 and extended in §T6 (the cache invalidation in §T6 is the only hub-specific branch). The handler is keyed by `command_id` / `public_id`.

Heartbeat QoS is 0 (§E.3 / §E.5 of the firmware guide) — liveness is a sliding window, not a fact, and QoS 1 here would amplify broker write load without making the 30 s TTL more accurate. The listener tolerates dropped frames silently; only sustained silence has operational meaning.

### Phase T4 — Active-Hub Election

`TransmitterElectionService.electActive(storeId)` returns the elected hub:

1. Read cached `store:{storeId}:transmitter:active`. If present **and** the cached hub still has a live `device:hub:alive:`* key, return it. (The cache TTL is shorter than the alive-key TTL, but a hub can drop its alive-key between cache writes if it crashes — re-validating is the cheap correct move.)
2. Otherwise scan candidates: `kind = 'TRANSMITTER_HUB' AND store_id = X AND status = 'ACTIVE'`, filter by live `device:hub:alive:*` key, pick the one with the highest `last_seen_at` (ties broken by lowest `device.id` for determinism), cache the result with `device.transmitter.active-cache-seconds` TTL, and return.
3. If no candidate, return `null` — the dispatch service translates that into `409 Conflict {"error":"no_active_transmitter"}`.

Election runs **lazily, only on dispatch**, so heartbeat traffic doesn't pay election cost. Suspending or decommissioning the cached primary deletes the cache (§T6), so the next dispatch re-elects within one round-trip.

### Phase D7a — Queue Dispatch Backend Plumbing

`DeviceDispatchService`, `DeviceDispatchEventBroadcaster`, `TicketDto` widening, `QueueService.issueTicket` extra-fields path, no-show / serve / cancel hooks, `device:busy:{deviceId}` lifecycle, and the admin endpoints (`POST /device-tickets`, `GET /available-devices`).

**Platform-consistency invariant (§1.0).** Device tickets are regular queue tickets. `DeviceDispatchService.issueDeviceTicket` calls `QueueService.issueTicket` — the same method the public controller uses for mobile-web customers. Once issued, the ticket enters the same Redis sorted set, gets the same position score, and is visible in the same admin waiting list. Every subsequent admin operation (`callNext`, `callSpecificTicket`, `serveTicket`, `cancelTicket`, `handleNoShow`, `transferTicket`, `pauseQueue`, `resumeQueue`, `cleanupServingSet`) runs its existing logic unchanged. The device-specific hooks below are **transparent side effects** that fire only when `ticketData["device_id"]` is present — they do not alter the core ticket state machine or skip any shared step.

GitNexus warning: `TicketDto` is HIGH risk (8 d=1 dependents). Land D7a only after F0–D6/T6 are stable, and update all consumers in one slice: `QueueService.issueTicket`, `callNext`, `callSpecificTicket`, `listWaitingTickets`, `getTicketDto`, `CallNextResult`, `QueueAdminController`, and the admin-web queue types/components.

`TicketDto` gains:

```kotlin
val deviceId: UUID? = null
val deviceName: String? = null
```

Update the matching `web/src/types/queue.ts`.

`QueueService.issueTicket` gains an optional internal parameter:

```kotlin
suspend fun issueTicket(
  storeId: UUID,
  serviceTypeId: UUID? = null,
  extraFields: Map<String, String> = emptyMap()
): TicketDto
```

`extraFields` is written to the ticket hash *after* the `ISSUE_TICKET_SCRIPT` Lua call returns, using the same `redis.opsForHash().put(...)` pattern that already writes `service_type_id`. The Lua script itself is **not** modified — the post-script HSET preserves the existing TTL set by the script.

`DeviceDispatchService.issueDeviceTicket(storeId, deviceId, adminId, serviceTypeId)`:

1. Preflight dispatch availability. `DeviceDispatchService` is **always** registered (it owns the receiver-side queue plumbing), but `TransmitterElectionService` is `@ConditionalOnProperty` on `device.transmitter.enabled` — so the dispatch service must inject the election service as `Optional<TransmitterElectionService>` (or `ObjectProvider<TransmitterElectionService>`) and treat absent-bean exactly like `electActive(...) → null`. Concretely:
  ```kotlin
   val hub = transmitterElectionService.orElse(null)?.electActive(storeId)
   if (hub == null) {
       throw HttpException.conflict("no_active_transmitter")
   }
  ```
   This collapses three cases into one error path: `device.transmitter.enabled = false` (bean absent), no hub registered for the store, or no registered hub heartbeating. Fail before writing Redis with `409 Conflict {"error":"no_active_transmitter"}`.
2. Load device; require `status = ACTIVE`, same store, receiver kind (not `TRANSMITTER_HUB`), no `device:busy:{deviceId}` key, and a `device_rf_code` row.
3. Set `device:busy:{deviceId}` with `NX` and `RedisTTLPolicy.TICKET_WAITING`.
4. Call `QueueService.issueTicket(..., extraFields = mapOf("device_id" to deviceId.toString()))`.
5. If ticket issuance or extra-field persistence fails, delete the busy key before rethrowing. If the failure happens after the Lua issue script wrote a ticket, remove that ticket from both global/service queues and delete its ticket hash before returning an error.
6. Return the resulting `TicketDto` with device fields populated.

`GET /available-devices` reuses the same preflight state as step 1 and returns `QueueDispatchAvailabilityResponse { devices, dispatchReady, error? }`. `devices` contains only receiver devices that are `ACTIVE`, store-owned by the requested store, have a current `device_rf_code` row, and have no `device:busy:{deviceId}` key. Hubs are excluded by kind filter. `dispatchReady = false, error = "no_active_transmitter"` covers all three fail-closed cases from step 1: feature gate off, no hub registered, or no registered hub heartbeating.

Queue-operation hooks — shared-then-branch pattern (see §1.0 lifecycle table for the full picture):

All hooks inspect `ticketData["device_id"]` after the shared `QueueService` logic completes. The shared flow (Lua scripts, status/TTL, SSE, MQTT, analytics, position alerts) runs unchanged regardless of ticket origin.

**Call** (`callNext` / `callSpecificTicket`): if `device_id` absent → `FcmNotificationService.sendTicketCalledNotification`. If present → skip FCM, publish `DEVICE_CALL_REQUESTED` via `DeviceDispatchEventBroadcaster`, keep the device reservation bound to the ticket, and refresh `device:busy:{deviceId}` to `TICKET_CALLED`.

**Serve / Cancel**: if `device_id` absent → remove FCM token. If present → publish `DEVICE_STOP_REQUESTED(mode = RELEASE)` (only if prior status was CALLED — a WAITING cancel has no active RF broadcast to stop). Before emitting the stop event, rewrite `device:busy:{deviceId}` to a terminal TTL so a failed stop does not free the device early; `TransmitterDispatchService` deletes the key only after a successful stop publish.

**No-show — Requeue**: if `device_id` present → publish `DEVICE_STOP_REQUESTED(mode = REQUEUE)`, keep `device_id` on ticket hash (same buzzer fires on next call), reset busy TTL to `TICKET_WAITING`.

**No-show — Skip**: if `device_id` present → publish `DEVICE_STOP_REQUESTED(mode = RELEASE)`, rewrite the busy key to a terminal TTL, and let `TransmitterDispatchService` delete it only after a successful stop publish. If absent → remove FCM token.

**Transfer**: `device_id` and `device:busy:{deviceId}` stay untouched — the device follows the ticket across service types.

`DeviceDispatchEventBroadcaster` uses `Sinks.many().multicast().directBestEffort()` — drops `onNext` only for the specific slow subscriber, so one stuck consumer never stalls dispatch for others.

### Phase T5 — Transmitter Dispatch Consumer

`TransmitterDispatchService` (`@ConditionalOnProperty` on `device.transmitter.enabled`) subscribes to `DeviceDispatchEventBroadcaster` from §D7a.

On `DEVICE_CALL_REQUESTED(storeId, ticketId, deviceId)`:

1. `electActive(storeId)`. The §D7a preflight already proved a hub was elected at ticket-issue time, but the elected hub may have stopped heartbeating between then and the call-next event — this is the only race the consumer needs to handle. If `null`:
  - Log a structured warning `transmitter_dispatch_election_lost` with `storeId`, `ticketId`, `deviceId`.
  - Publish `DEVICE_DISPATCH_FAILED` through the queue-event path used elsewhere in the repo: widen `MqttPublisher.QueueEventType`, emit the MQTT queue event, and also `broadcast(QueueSseEvent(...))` locally so SSE stays coherent across single- and multi-instance deployments. §D7b widens the frontend `QueueEventType`, `QueueSseEvent`, and `useQueueEvents` listener list to receive it.
  - Stop. Do not auto-retry, auto-revert, or auto-release the device reservation. The ticket stays in its current queue state and the failure is an operator-facing signal.
2. Load the receiver row and decrypt its RF code via `pgp_sym_decrypt_bytea` — this is the only point in the system that does so on the dispatch hot path. Decrypted plaintext lives only in service-local memory for the duration of the publish.
3. Build `transmit-v1|hub_public_id|dispatch_id|receiver_public_id|band|rf_code_hex|rf_code_bits|issued_at`, where `band` is derived from the receiver row's `kind` (§2.3):
  - `RECEIVER_433M`, `RECEIVER_433M_PASSIVE` → `"433M"`.
  - `RECEIVER_2_4G` → `"2_4G"`.
   No new DB roundtrip — `kind` is already on the receiver row loaded in step 2.
4. Sign with the existing command-signing key.
5. Publish on `transmitter/hub/{hubPublicId}/cmd/transmit` (QoS 1, **not retained**) via `DeviceMqttPublisher.publishTransmit`.

On `DEVICE_STOP_REQUESTED(storeId, ticketId, deviceId)`:

- "Stop" is another `transmit-v1` with the same RF code (firmware toggles vibrator on each matching frame). Re-elect, decrypt, sign, publish.
- If re-election fails, emit `DEVICE_DISPATCH_FAILED` and structured warning. No silent fallback — the admin needs the signal because the receiver may keep vibrating until a hub resends the stop frame.

Busy-key ownership for hub-published signals:

- D7a writes or rewrites `device:busy:{deviceId}` before it emits the dispatch event.
- `DEVICE_STOP_REQUESTED` carries the post-stop disposition the consumer needs: `RELEASE` for serve / cancel / skip, `REQUEUE` for no-show requeue.
- T5 is the only place that releases the reservation: on successful `DEVICE_CALL_REQUESTED`, keep the `TICKET_CALLED` TTL; on successful `DEVICE_STOP_REQUESTED`, honour the event disposition and either delete the key (`RELEASE`) or keep it at `TICKET_WAITING` (`REQUEUE`).
- On `DEVICE_DISPATCH_FAILED`, keep the current busy-key value and TTL in place. This is a deliberate fail-closed rule: the device remains unavailable for new dispatches while its physical signal state is uncertain.

Transmit-ack ingestion lives in §T3, not here — keeping the consumer single-direction (events → MQTT).

### Phase D7b — Queue Dispatch UI

The queue page gains a dispatch action. Lands together with §T5 because they are mutually load-bearing: §T5 makes dispatch end-to-end, and §D7b is the admin-visible surface that exercises it.

- Header action button on `web/src/app/[locale]/dashboard/queue/page.tsx`: "Dispatch via device" (i18n key `queue.dispatch.action`). Disabled when `GET /available-devices` returns either an empty `devices` list or `dispatchReady = false`. The disabled-state tooltip distinguishes the two reasons.
- Device-selection `Dialog` showing the available receivers for the store. Only receiver kinds — never hubs.
- In-row `Radio` ticket badge linking to `/dashboard/devices/{deviceId}` for tickets bound to a device.
- Queue-page i18n keys: `queue.dispatch.`*. Vietnamese copy follows the user's natural / concise style — no English fallbacks in `vi.json`.
- Queue route helpers: extend `API_ROUTES.QUEUE` with `DEVICE_TICKETS(storeId)` and `AVAILABLE_DEVICES(storeId)` constants.
- Queue SSE typing/listener update: extend `web/src/types/queue.ts` `QueueEventType` with `DEVICE_DISPATCH_FAILED`, widen `QueueSseEvent`, add the new event name to `web/src/hooks/use-queue-events.ts`, and widen backend `MqttPublisher.QueueEventType` so the failure event also survives the repo's MQTT-backed cross-instance fan-out path.

Device-bound tickets appear inline in the existing queue UI — same waiting list, same serving display, same admin controls (§1.0). The only visual addition is a device badge on the ticket row (§9.7). Existing Call Next, serve, cancel, no-show, transfer, and queue-state controls stay unchanged.

### Phase T7 — Cap Enforcement Surface

The cap check in §T1 step 5 is the only enforcement point; T7 is purely about admin-web visibility:

- `GET /api/devices?kind=TRANSMITTER_HUB&storeId=X` returns the current registered count alongside the list (§7).
- The device list page renders a "2 / 3 hubs registered" badge for stores with at least one hub. For SUPER_ADMINs viewing the all-stores list, the badge appears next to the store header in the grouped view.
- A 4th-hub registration attempt over MQTT fails with the `hub_cap_reached` envelope; firmware shows a clear local error and goes back into provisioning. No backend follow-up needed — the rejection is durable in admin-side bootstrap logs.

---

## 9. Admin Web Plan

### 9.1 Routes

```text
web/src/app/[locale]/dashboard/devices/page.tsx
web/src/app/[locale]/dashboard/devices/pending/page.tsx
web/src/app/[locale]/dashboard/devices/tokens/page.tsx
web/src/app/[locale]/dashboard/devices/[id]/page.tsx
```

For `[id]`, prefer an async server wrapper:

```tsx
export default async function DeviceDetailPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;
  return <DeviceDetailContent deviceId={id} />;
}
```

Use React `use(params)` only when the page itself must be a client component; `useParams` from `next/navigation` is also fine inside nested client components.

The device detail page renders both receiver and transmitter rows; panels are conditional on `device.kind` (§9.8 for passive PT2272 specifics; §9.9 for transmitter hub specifics).

### 9.2 Sidebar

Update the `NavItem.label` union in `web/src/components/layout/sidebar.tsx`:

```ts
label:
  | "overview"
  | "queue"
  | "analytics"
  | "stores"
  | "admins"
  | "settings"
  | "devices";
```

Append a `devices` item using the lucide `Radio` icon. Add `navigation.devices` to both message files. Do not hide the nav item by role — backend scoping controls data visibility.

### 9.3 Feature Folder

Add `web/src/features/device/`:

```text
api.ts
types.ts
device-list-table.tsx
device-status-badge.tsx
device-filter-bar.tsx
pending-review-card.tsx
approve-dialog.tsx
reject-dialog.tsx
rf-code-editor.tsx                 # rotation editor (MCU receivers); read-only render for passive devices (§9.8); not rendered for hubs (§9.9)
lifecycle-panel.tsx
enrollment-token-dialog.tsx
enrollment-token-table.tsx
passive-device-form-dialog.tsx     # PT2272 manual register dialog (§9.8)
hub-heartbeat-panel.tsx            # transmitter hub liveness + election state (§9.9)
hub-cap-badge.tsx                  # "2 / 3 hubs registered" (§9.9)
dispatched-ticket-panel.tsx
use-device-ack-poll.ts
```

Use the existing repo pattern:

- API calls via `get`, `post`, `put`, `del` from `@/lib/api`; routes from `API_ROUTES`.
- Tables: native `<table>` markup inside `glass-card`, matching the existing admin/store pattern.
- Reuse already-present shadcn primitives: `Button`, `Badge`, `Dialog`, `AlertDialog`, `Input`, `Label`, `Select`, `Switch`, `Skeleton`, `Tooltip`, `InlineError`, `Sonner`.
- If a missing primitive is genuinely needed, add it via the shadcn CLI/source pattern and log it in `CHANGELOGS.md`. Do not add another UI framework.

### 9.4 UI Rules

Follow `docs/walkthrough/Web Styles.md` and current admin-web conventions:

- Use `glass-card` / `glass-panel` surfaces consistently with queue, store, and admin pages. Never nest cards (the design system explicitly forbids glass-on-glass); a detail page uses sibling panels.
- Use theme tokens (`primary`, `action`, `success`, `warning`, `destructive`, `muted`, `border`) — no arbitrary new colors.
- Move long Tailwind class lists to `web/src/styles/device.css`.
- Action icons (`Radio`, `Copy`, `RotateCcw`, `Power`, `Ban`, `Trash2`, `Check`, `X`, `Loader2`, `Activity` for heartbeat) need accessible labels and tooltips when icon-only.
- `AlertDialog` for reject, decommission, reprovision, and token-revoke confirmations. `Decommission` uses destructive styling.
- One-time token display lives in a dialog with a copy action and a clear warning. The plaintext token never returns once the dialog closes.
- Native tables wrap in `overflow-x-auto` with stable min widths.
- No marketing hero, no explanatory feature panels, no duplicate edit affordances.
- All visible strings flow through `en.json` and `vi.json`. Both are valid JSON — no comments.

### 9.5 API Constants

Append to `web/src/lib/constants.ts`:

```ts
DEVICES: {
  BASE: "/api/devices",
  BY_ID: (id: string) => `/api/devices/${id}`,
  PASSIVE: "/api/devices/passive",
  APPROVE: (id: string) => `/api/devices/${id}/approve`,
  REJECT: (id: string) => `/api/devices/${id}/reject`,
  RF_CODE: (id: string) => `/api/devices/${id}/rf-code`,
  LIFECYCLE: (id: string) => `/api/devices/${id}/lifecycle`,
  REPROVISION: (id: string) => `/api/devices/${id}/reprovision`,
  TOKENS: "/api/devices/enrollment-tokens",
  TOKEN_BY_HASH: (hash: string) => `/api/devices/enrollment-tokens/${hash}`,
},
```

Extend `API_ROUTES.QUEUE` with the dispatch helpers landed alongside §D7b:

```ts
QUEUE: {
  // ...existing keys
  DEVICE_TICKETS: (storeId: string) => `/api/queue/admin/${storeId}/device-tickets`,
  AVAILABLE_DEVICES: (storeId: string) => `/api/queue/admin/${storeId}/available-devices`,
},
```

### 9.6 i18n Keys

Add valid JSON entries to both `en.json` and `vi.json`. Vietnamese copy follows the user's natural / concise style — no English fallbacks left in `vi.json`.

- `navigation.devices`
- `devices.title`, `devices.pendingTab`, `devices.tokensTab`, `devices.listTitle`, `devices.emptyState`
- `devices.groupByStore`, `devices.filterStatus`, `devices.filterKind`, `devices.filterHardware`, `devices.filterStore`
- `devices.status.{PENDING, PENDING_RF_CODE, ACTIVE, SUSPENDED, DECOMMISSIONED, REJECTED}`
- `devices.kind.{RECEIVER_433M, RECEIVER_433M_PASSIVE, RECEIVER_2_4G, TRANSMITTER_HUB}`
- `devices.pending.*`, `devices.rfCode.*`, `devices.lifecycle.*`, `devices.dispatch.*`, `devices.tokens.*`
- `devices.passive.*` — full list in §9.8
- `devices.hub.*` — full list in §9.9
- `queue.dispatch.*` — list in §9.7

For any UI that displays `device.kind` (badge, chip, table cell, filter option, dialog field, detail metadata), use this visible-label map:


| Internal value          | i18n key                             | English label      | Vietnamese label   |
| ----------------------- | ------------------------------------ | ------------------ | ------------------ |
| `RECEIVER_433M`         | `devices.kind.RECEIVER_433M`         | `433 MHz Receiver` | `Bộ thu 433 MHz`   |
| `RECEIVER_433M_PASSIVE` | `devices.kind.RECEIVER_433M_PASSIVE` | `433 MHz Receiver` | `Bộ thu 433 MHz`   |
| `RECEIVER_2_4G`         | `devices.kind.RECEIVER_2_4G`         | `2.4 GHz Receiver` | `Bộ thu 2.4 GHz`   |
| `TRANSMITTER_HUB`       | `devices.kind.TRANSMITTER_HUB`       | `Transmitter Hub`  | `Bộ phát tín hiệu` |


The visible UI must show the mapped label value, never the backend enum literal (`RECEIVER_433M`, etc.) and never the i18n key name (`devices.kind.RECEIVER_433M`, etc.). `device.kind` remains an API/internal discriminator only.

### 9.7 Queue UI (D7b)

**Unified queue view (§1.0).** Device-bound tickets and mobile-web tickets appear in the same `waiting-list.tsx` and `serving-display.tsx` components, interleaved by position order. The admin does not need to distinguish origin to manage the queue. Every existing queue control — Call Next, serve, cancel, no-show, transfer, pause/resume, cleanup — works identically on device-bound tickets. The widened `TicketDto` carries `deviceId` and `deviceName` (nullable), so the UI conditionally renders a device badge without branching the ticket management logic.

The queue page gains:

- Header action button: `queue.dispatch.action` (label), `queue.dispatch.disabledNoDevice` and `queue.dispatch.disabledNoHub` (tooltip variants for the two disabled reasons).
- Device-selection dialog: `queue.dispatch.dialogTitle`, `queue.dispatch.dialogDescription`, `queue.dispatch.deviceLabel`, `queue.dispatch.confirm`, `queue.dispatch.cancel`.
- Per-row device indicator: when `ticket.deviceId` is non-null, render a `Radio` icon badge with the `ticket.deviceName` via `queue.dispatch.badgeLabel` (interpolated). Clicking the badge navigates to `/dashboard/devices/{deviceId}`. When `ticket.deviceId` is null (mobile-web ticket), no badge is rendered — the row looks exactly as it does today.
- Error toasts: `queue.dispatch.errorNoActiveTransmitter` (mapped from backend `no_active_transmitter`), `queue.dispatch.errorDeviceBusy`, `queue.dispatch.errorGeneric`.
- Queue SSE typing/listener update: extend `web/src/types/queue.ts` `QueueEventType` with `DEVICE_DISPATCH_FAILED`, widen `QueueSseEvent`, add the new event name to `web/src/hooks/use-queue-events.ts`, and widen backend `MqttPublisher.QueueEventType` so the failure event also survives the repo's MQTT-backed cross-instance fan-out path.

**Serving display.** When a device-bound ticket is called (via Call Next or jump call), it appears in the serving display exactly like a mobile-web ticket — same counter assignment, same "Serve" / "Cancel" / "No-show" action buttons. The device badge is shown inline so the admin knows a physical buzzer was triggered, but the interaction model is identical.

Existing Call Next, serve, cancel, no-show, transfer, and queue-state controls are unchanged. The dispatch button is enabled only when `GET /available-devices` returns `dispatchReady = true` and a non-empty `devices` array. When the page receives `DEVICE_DISPATCH_FAILED`, it shows the transmitter-unavailable toast, refreshes the preflight envelope, and leaves the ticket state otherwise untouched. v1 does not auto-retry or auto-revert the queue state; the event is a fail-closed operator signal and the later queue-side resend affordance remains deferred (§11).

### 9.8 Passive PT2272 Manual Registration UI

Passive PT2272 receivers have no firmware and cannot self-enroll. The admin reads the chip's preset code (often from DIP switches or a label) and registers manually. The UI uses only primitives already shipped in `web/src/components/ui` (`alert-dialog`, `badge`, `button`, `card`, `dialog`, `dropdown-menu`, `inline-error`, `input`, `label`, `popover`, `select`, `separator`, `skeleton`, `sonner`, `switch`, `tooltip`).

#### 9.8.1 Entry point and dialog

In `web/src/app/[locale]/dashboard/devices/page.tsx`, next to the **Issue enrollment token** button:

- Lucide `Plus` icon (existing add-action convention in the repo).
- Label key: `devices.passive.registerAction`.
- Disabled when ADMIN has no `principal.storeId`; tooltip uses `devices.passive.disabledNoStore`.
- Click opens `<PassiveDeviceFormDialog />`. Dialog-only flow — no separate route — matching the precedent of `store-form-dialog.tsx` and `service-type-form-dialog.tsx`.

The button is **not** role-gated in the UI; backend `StoreAccessUtil` enforces authority. The disabled state is purely a UX hint.

`passive-device-form-dialog.tsx` submits to `API_ROUTES.DEVICES.PASSIVE` via `post` from `@/lib/api`. Body shape per §7. Form fields:


| Field          | Component                | Notes                                                                                                                                                                                                                              |
| -------------- | ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Kind           | `Badge`                  | Read-only, shows the kind-label-map value for `RECEIVER_433M_PASSIVE` (`433 MHz Receiver` / `Bộ thu 433 MHz` per §9.6). The passive nature is conveyed by the adjacent `PT2272` hardware-model badge, not the kind label. Never render the enum literal. Hardcoded in payload.      |
| Hardware model | `Badge`                  | Read-only, `PT2272`. Hardcoded in payload.                                                                                                                                                                                         |
| Store          | `Select`                 | SUPER_ADMIN: reuse the repo's current selector pattern (`listStores(0, 100)` from the store feature) rather than coupling this dialog to the store-management table state. ADMIN: single read-only entry forced to `principal.storeId`. Required. |
| Assigned name  | `Input`                  | Required, `maxLength={100}`.                                                                                                                                                                                                       |
| Bit width      | `Select`                 | Options 1..24; default 24. Selecting a width re-derives the hex field's `maxLength`. Helper text via `devices.passive.bitsHelp` explains the 24-bit cap is a PT2272 hardware limit.                                                |
| RF code hex    | `Input` with `font-mono` | `[0-9A-Fa-f]+`, uppercased on blur. `maxLength = 2 * Math.ceil(bits / 8)`.                                                                                                                                                         |


Field errors render via `InlineError`; field labels via `Label`; helper text uses the existing muted-text pattern from `store-general-fields.tsx`. Footer: primary `Button` (`devices.passive.submit`) with a `Loader2` spinner during submit (matching `create-admin-dialog.tsx`); secondary `Button variant="outline"` (`common.cancel`, existing key). `Dialog` already provides its surface — do not nest it inside a `glass-card`.

On success: close, `toast.success` with `devices.passive.successToast` (interpolated with the new device's `public_id`), and re-fetch the device list (parent owns the query state, same pattern as `delete-admin-dialog.tsx`).

#### 9.8.2 Validation

Client-side (blocks submit; mirrors §2.4 + §D5.0):

- `assignedName.trim().length > 0`.
- `storeId` present.
- `rfCodeBits ∈ [1, 24]`.
- `rfCodeHex` matches `/^[0-9A-F]+$/i` and `rfCodeHex.length === 2 * Math.ceil(rfCodeBits / 8)`.

Server-side (surface via `toast` from `sonner` plus `InlineError` next to the relevant field):

- `409 Conflict` on uniqueness clash → `devices.passive.errorConflict`. The response body includes the colliding `public_id`; render it as plain text in the toast (no link). Never render the colliding code itself.
- `400 Bad Request` with structured `error` field (`pt2272_hardware_cap_exceeded`, `forbidden_pattern`, `width_out_of_range`) → keys under `devices.passive.errors.`*. The hardware-cap variant is the authoritative server-side echo of the `Select` cap, in case a client bypasses the form.
- `403 Forbidden` from `StoreAccessUtil` → `devices.passive.errorForbidden`.

#### 9.8.3 Detail page consistency for passive devices

`/dashboard/devices/{id}` renders panels conditionally on `device.kind`:

- `rf-code-editor.tsx` (`kind === "RECEIVER_433M_PASSIVE"`): render read-only. Show `bits` and a `Badge` reading `••  ••  ••` via key `devices.rfCode.maskedValue`. Hide the **Rotate** button. Replace the panel description with `devices.rfCode.lockedNote` — a frontend-i18n string that conveys the same meaning as the server's `hardware_fixed_code` 400 (§D5.2). Do not parrot the server's English message; the server returns the structured `error` code, the UI provides the user-facing copy.
- `lifecycle-panel.tsx`: behaviour is unchanged (backend handles passive lifecycle as a status flip per §D6). Add `devices.lifecycle.passiveSubtitle` so admins understand the chip itself does not stop listening.
- The **Reprovision** action is hidden for passive devices (the server returns 404 anyway per §7).

#### 9.8.4 i18n keys (`devices.passive.`*)

Add to both `en.json` and `vi.json`:

- `devices.passive.registerAction`
- `devices.passive.disabledNoStore`
- `devices.passive.dialogTitle`, `devices.passive.dialogDescription`
- `devices.passive.kindLabel`, `devices.passive.hardwareLabel`, `devices.passive.storeLabel`, `devices.passive.nameLabel`, `devices.passive.bitsLabel`, `devices.passive.bitsHelp`, `devices.passive.hexLabel`, `devices.passive.hexHelp`
- `devices.passive.hexErrorFormat`, `devices.passive.hexErrorLength`
- `devices.passive.submit`, `devices.passive.successToast`
- `devices.passive.errorConflict`, `devices.passive.errorForbidden`
- `devices.passive.errors.pt2272_hardware_cap_exceeded`, `devices.passive.errors.forbidden_pattern`, `devices.passive.errors.width_out_of_range`
- `devices.rfCode.maskedValue`, `devices.rfCode.lockedNote`
- `devices.lifecycle.passiveSubtitle`

### 9.9 Transmitter Hub UI

Transmitter hubs surface in the same `/dashboard/devices` flow as receivers. No separate route. Panels and badges that are hub-only are conditional on `device.kind === "TRANSMITTER_HUB"`.

#### 9.9.1 Device list

- Hubs render in the same list as receivers, with the kind-label-map value for `TRANSMITTER_HUB` (`Transmitter Hub` / `Bộ phát tín hiệu` per §9.6) displayed in the kind column. Do not expose `TRANSMITTER_HUB` or `devices.kind.TRANSMITTER_HUB` in visible UI.
- For each store with at least one registered hub, a `<HubCapBadge />` renders inline ("2 / 3 hubs registered", i18n `devices.hub.capBadge`). The count comes from the `registered` field of `GET /api/devices?kind=TRANSMITTER_HUB&storeId=X` (§7 / §T7). Tooltip via `devices.hub.capTooltip` explains the cap is a backend-enforced ceiling.
- Filtering by `kind=TRANSMITTER_HUB` is supported via `device-filter-bar.tsx` — no new component, just an additional option in the existing kind selector.

#### 9.9.2 Detail page panels

`/dashboard/devices/{id}` for `kind === "TRANSMITTER_HUB"` renders:

- **Identity panel** (shared with receivers): `public_id`, `assigned_name`, `firmware_version`, `activated_at`, `created_at`.
- **Heartbeat panel** (`hub-heartbeat-panel.tsx`, hub-only):
  - "Last heartbeat" line — `device.last_seen_at` formatted with the project's locale-agnostic time helper.
  - Liveness `Badge` derived from whether `device:hub:alive:`* is live; the backend exposes this as `liveness: "alive" | "stale"` on the hub detail response. i18n: `devices.hub.liveness.alive`, `devices.hub.liveness.stale`.
  - Election state line: `devices.hub.election.elected` when the hub is the cached primary for its store; `devices.hub.election.standby` otherwise. The cached primary comes from the same hub detail response (`activeHub: boolean`).
  - The detail page polls every 15 seconds (current firmware default heartbeat 10 seconds + 5-second margin) to catch liveness flips without ack-stream SSE (deferred, §11). The frontend does not read backend YAML at runtime, so this is an intentional client constant aligned with the documented defaults.
- **Lifecycle panel** (`lifecycle-panel.tsx`, shared): the same suspend / resume / decommission actions. Add `devices.lifecycle.hubSubtitle` explaining that suspending the active hub forces the next dispatch to re-elect.
- **No RF-code panel.** `rf-code-editor.tsx` is **not rendered** for hubs. The server returns `400 Bad Request {"error":"hub_no_rf_code"}` for any rotation attempt (§7).
- **Dispatched ticket panel** (`dispatched-ticket-panel.tsx`, shared): for hubs, this surfaces only "this hub is currently elected for store X — recent dispatches go through it" (i18n `devices.hub.recentDispatches.empty` / `devices.hub.recentDispatches.note`). v1 does not render a transmit-ack history (deferred).
- **Reprovision** is offered (calls `POST /api/devices/{id}/reprovision`); the action surfaces a confirmation `AlertDialog` (`devices.hub.reprovisionConfirm`) since it forces the hub back through SoftAP provisioning.

#### 9.9.3 Pending review

Hubs in `status === "PENDING"` show alongside receivers in the pending-review list. The approval flow is the same `<ApproveDialog />`; the dialog gains a hub-specific subtitle when the device kind is `TRANSMITTER_HUB` (i18n `devices.hub.approveSubtitle`) explaining that approving consumes a slot in the per-store cap.

#### 9.9.4 i18n keys (`devices.hub.`*)

Add to both `en.json` and `vi.json`:

- `devices.hub.capBadge` (interpolated `{registered} / {max}`), `devices.hub.capTooltip`
- `devices.hub.liveness.alive`, `devices.hub.liveness.stale`
- `devices.hub.election.elected`, `devices.hub.election.standby`
- `devices.hub.recentDispatches.note`, `devices.hub.recentDispatches.empty`
- `devices.hub.reprovisionConfirm`
- `devices.hub.approveSubtitle`
- `devices.lifecycle.hubSubtitle`

---

## 10. Execution Order

Phases that can ship together (because they share a slice or unblock each other) are grouped. Each step updates `docs/CHANGELOGS.md` with every file touched and any skipped item.

1. **F0** — Foundation: impact checks, schema/migration (§3), config, signing, canonical helpers (`activate-v1`, `rf-code-v1`, `deact-v1`, `transmit-v1`), `DevicePublicIdMinter` (with `hub-` prefix from day one), `DeviceMqttPublisher` with both receiver and hub publish methods, `MqttClientManager` v5 subscription overload, both Redis-key buckets.
2. **D1** — Enrollment token backend + admin-web token screen.
3. **D2 + T1** — Bootstrap registration for both families, `DeviceFamily` discriminator, per-store hub cap pre-check. Admin-web pending-review surface renders both kinds.
4. **D2.1** — PT2272 manual register backend + admin-web dialog.
5. **D3** — Approval (shared, kind-aware retained-cleanup hand-off in §D4 / §T2).
6. **D4 + T2** — Activation: shared signature-verify path; receiver path issues initial RF code, hub path goes straight to `ACTIVE` with `hub-XXXXX` `public_id`.
7. **D5** — RF code (receivers only): forbidden-set + uniqueness, auto-issue / rotation / admin-set, ack handling, RF-code editor.
8. **D6 + T6** — Lifecycle: shared `cmd/deact` path; hub path adds election-cache invalidation. Lifecycle panel handles both.
9. **T3 + T4** — Transmitter heartbeat ingestion + active-hub election service. Heartbeat panel + cap badge in admin web.
10. **D7a + T5** — Queue dispatch backend plumbing (broadcaster, `TicketDto` widening, `device:busy` lifecycle, admin endpoints) + transmitter dispatch consumer that turns events into `cmd/transmit`. The two land together because §T5 is what makes §D7a's preflight pass.
11. **D7b + T7** — Queue dispatch UI + cap-counter surface. Last because both depend on the dispatch path being end-to-end live.

After all phases ship, mark the notifier/device-domain row complete in `docs/planned/Backend Remaining Plan.md` with a link back here.

---

## 11. Deferred

Items intentionally out of scope for this plan. Each is independent of the work above and can land as a follow-up.

- Transmitter hub firmware (this plan covers backend and admin web only; firmware target end state lives in [TRANSMITTER_ESP32C3_DESIGN_GUIDE.md](TRANSMITTER_ESP32C3_DESIGN_GUIDE.md) §A–§G, §I).
- Web Serial bridge (provisioning helper).
- Per-device MQTT credentials and broker ACLs.
- SSE for device acks; v1 polls detail pages while ack is pending. The hub heartbeat panel polls on a fixed interval too.
- Queue-side resend affordance for a failed `DEVICE_CALL_REQUESTED` / `DEVICE_STOP_REQUESTED`. v1 surfaces `DEVICE_DISPATCH_FAILED`, keeps the device reservation fail-closed, and requires operator triage instead of auto-retrying or auto-reverting ticket state.
- Surfacing nRF24 probe failures from hubs into the heartbeat envelope as a capability bitmap so admin can flag a 433-only hub before it tries to dispatch a 2.4 G code.
- Transmit-ack analytics emission (how many dispatches actually fired vs. failed silently). Pairs with the `DEVICE_TRIGGERED` analytics gap below.
- RF-code history beyond the current version.
- `DEVICE_TRIGGERED` analytics emission.
- In-field rotation of the backend command-signing key.
- A dedicated `device_event` audit table.
- A dedicated transmitter admin screen with hub map / heartbeat trace beyond the per-device panels in §9.9.
