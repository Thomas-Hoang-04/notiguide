# Transmitter Dispatch — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Three changes to the transmitter dispatch path (backend-heavy, plus one small admin-dashboard string):
- **Part A — Dispatch device name:** attach the target receiver's human-readable `assigned_name` to the full-payload `cmd/transmit` packet so the firmware OLED can show it (display-only, unsigned).
- **Part B — Case B ack-timeout reconciliation:** close the deadlock where a `CALL` dispatch whose hub dies *after publish* is never acknowledged — today nothing fires, the `device:busy` lock lingers for 12h, and the dashboard never learns the dispatch was lost.
- **Part C — Dashboard timeout message:** give the new `ack_timeout` dispatch-failure reason its own bilingual toast on the admin queue page (today it falls through to the generic "system issue" toast).

**Architecture:**
- *Part A* adds an optional `device_name` field to the `TransmitEnvelope` JSON, set from `receiver.assignedName`. It is deliberately **outside** the signed canonical (`transmit-v1` unchanged, `schema_version` stays `1`), so it is fully backward/forward compatible and requires no lockstep deploy.
- *Part B* makes the absence of an ack observable. On each successful **CALL** publish the service arms a short-TTL timer key `dispatch:pending-ack:{dispatchId}` next to the existing `dispatch:tracking:{dispatchId}` record. When the hub acks, both keys are cleared; when the timer key's TTL lapses, the existing Redis keyspace-expiry listener (the same one that drives ticket no-show cleanup) fires a `DispatchReconciliationService` that **atomically claims** the tracking record (GET then DEL-with-count-check — exactly one racer wins), releases the `device:busy` lock **if it still belongs to this ticket**, and emits `DEVICE_DISPATCH_FAILED(reason=ack_timeout)`. The rejection and timeout paths are consolidated into one `failDispatch` method (DRY).
- *Part C* adds one branch to the admin dashboard's SSE handler so the new `ack_timeout` reason shows a purpose-built bilingual toast instead of the generic infrastructure error.

**Tech Stack:** Kotlin / Spring Boot 3.5 (WebFlux, coroutines) / Jackson (`jackson-module-kotlin`) / Spring Data Redis (reactive `ReactiveRedisTemplate`) / JUnit 5 + AssertJ + MockK + `kotlinx-coroutines-test` (backend). Next.js 16 / next-intl / Biome (admin web).

**Source design:** approved in the brainstorming session for this plan (Option 1 — keyspace-expiry trigger). The earlier display-name spec is `docs/spec/Transmitter Dispatch Device Name Spec.md`.

---

## Status: firmware already landed; scope is backend + one dashboard string

The **firmware side** of the original device-name plan (`transmitter/main/events/transmitter_events.h`, `dispatch/dispatch.c`, `display/display.c`) has been **implemented and reviewed**, so its tasks have been removed from this document. The firmware is backward-compatible: it tolerates the absence of `device_name` and falls back to `public_id` / `slot-N`. Therefore Part A (the backend producer of `device_name`) can land independently, after the firmware, with no lockstep requirement.

This plan covers the **`notiguide` backend** (Parts A & B) plus **one small admin-dashboard change** in the **`web` repo** (Part C — the `ack_timeout` toast).

---

## Conventions for this plan (project rules)

- **No commit steps.** Committing is the executor's decision — this plan never instructs `git commit`.
- **No build steps.** Do not run `./gradlew build`/`assemble` or `yarn build`. The separate audit flow owns full compilation/build verification. The only commands this plan runs are the targeted new/changed **test classes** (backend TDD loop) and **`yarn lint`** (frontend).
- **GitNexus impact.** Per the repo's GitNexus rules, run `gitnexus_impact({target, direction: "upstream"})` before editing each backend symbol and `gitnexus_detect_changes()` after edits to confirm scope. These are included as steps. The backend repo is indexed as `notiguide`.
- **Named imports only**, at the top of the file (no fully-qualified inline references) — matches the existing code style.
- **No isolated unit-test harness exists** for `TransmitterDispatchService`, `TransmitterOperationalListener`, or `RedisKeyExpirationListener` (they wire MQTT/keyspace subscriptions). Tasks touching those are **edit-only**; their behavior is verified by the audit/integration flow. The new pure logic (`RedisKeyManager` helpers, `DispatchReconciliationService`) **is** unit-tested via TDD.
- **Frontend (Part C):** edit-only — a single toast-reason branch plus two i18n strings, matching the existing (untested) dispatch-reason branches; verification is **`yarn lint`** (biome) only, no unit test, no build. New i18n keys must **mirror structurally** between `en.json` and `vi.json` per the project bilingual rule. The Vietnamese copy in Task C1 was proposed and approved in chat before inclusion.

---

## File Structure

| File | Repo | Responsibility | Change |
|---|---|---|---|
| `backend/src/main/kotlin/com/thomas/notiguide/domain/device/service/TransmitterDispatchService.kt` | notiguide | Builds & publishes `cmd/transmit`; writes dispatch tracking | **Part A:** add `device_name` to `TransmitEnvelope` (`private`→`internal`), set from `receiver.assignedName`. **Part B:** inject `DeviceTransmitterProperties`; extract a `trackDispatch(...)` helper that writes the tracking record (always) **and** arms the `pending-ack` timer (CALL only); call it from both dispatch paths. |
| `backend/src/test/kotlin/com/thomas/notiguide/domain/device/service/TransmitEnvelopeSerializationTest.kt` | notiguide | Verifies `TransmitEnvelope` wire shape | **Create** (Part A). |
| `backend/src/main/kotlin/com/thomas/notiguide/core/redis/RedisKeyManager.kt` | notiguide | Central Redis key vocabulary | **Part B:** add `dispatchPendingAck`, `isPendingAckKey`, `parsePendingAckKey`. |
| `backend/src/test/kotlin/com/thomas/notiguide/core/redis/RedisKeyManagerTest.kt` | notiguide | Key-format unit tests | **Modify** (Part B): add pending-ack tests. |
| `backend/src/main/kotlin/com/thomas/notiguide/core/device/DeviceTransmitterProperties.kt` | notiguide | Transmitter config | **Part B:** add `dispatchAckTimeoutSeconds: Long = 45` (+`require`). |
| `backend/src/main/resources/application.yaml` | notiguide | Default config | **Part B:** add `dispatch-ack-timeout-seconds: 45`. |
| `backend/src/main/kotlin/com/thomas/notiguide/domain/device/service/DispatchReconciliationService.kt` | notiguide | Atomic claim → ticket-scoped busy release + emit failure; shared by rejection & timeout | **Create** (Part B). |
| `backend/src/test/kotlin/com/thomas/notiguide/domain/device/service/DispatchReconciliationServiceTest.kt` | notiguide | Unit-tests the reconciler | **Create** (Part B). |
| `backend/src/main/kotlin/com/thomas/notiguide/domain/device/listener/TransmitterOperationalListener.kt` | notiguide | Handles hub heartbeats & acks | **Part B:** route applied/unchanged→`completeDispatch`, rejected→`failDispatch`; delete the now-redundant `handleTransmitRejection`; swap the `queueEventBroadcaster` dependency for `DispatchReconciliationService`. |
| `backend/src/main/kotlin/com/thomas/notiguide/core/redis/RedisKeyExpirationListener.kt` | notiguide | Reacts to Redis key expiry | **Part B:** on `pending-ack` key expiry, delegate to `DispatchReconciliationService.failDispatch(..., "ack_timeout")` via an `ObjectProvider`. |
| `web/src/app/[locale]/dashboard/queue/page.tsx` | web | Admin queue page; SSE handler for dispatch failures | **Part C:** add an `ack_timeout` branch to the `DEVICE_DISPATCH_FAILED` reason switch. |
| `web/src/messages/en.json` | web | English copy | **Part C:** add `queue.dispatch.errorAckTimeout`. |
| `web/src/messages/vi.json` | web | Vietnamese copy | **Part C:** add `queue.dispatch.errorAckTimeout` (mirrored). |
| `docs/CHANGELOGS.md` | (workspace) | Change log | **Part D:** dated entry for Parts A, B & C. |

**Task order:** A1 (independent) → B1 (keys, needed by B3/B4/B6) → B2 (config, needed by B4) → B3 (reconciler, needed by B5/B6) → B4 (arm timer) → B5 (wire acks) → B6 (fire on expiry) → C1 (dashboard toast; depends on B emitting `ack_timeout`) → D1 (CHANGELOGS).

---

## Part A — Dispatch device name (full-payload path)

### Task A1: `device_name` on `TransmitEnvelope` (TDD)

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/service/TransmitterDispatchService.kt`
- Create: `backend/src/test/kotlin/com/thomas/notiguide/domain/device/service/TransmitEnvelopeSerializationTest.kt`

- [ ] **Step 1: Impact analysis (project rule)**

Run: `gitnexus_impact({target: "buildPayload", direction: "upstream"})`
Expected: LOW risk — `buildPayload` is a private helper used only by `TransmitterDispatchService.handle(...)`; `TransmitEnvelope` is a file-local data class with no external callers. If HIGH/CRITICAL, stop and report before editing.

- [ ] **Step 2: Write the failing test**

Create `backend/src/test/kotlin/com/thomas/notiguide/domain/device/service/TransmitEnvelopeSerializationTest.kt`:

```kotlin
package com.thomas.notiguide.domain.device.service

import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import java.util.UUID

class TransmitEnvelopeSerializationTest {

    private val mapper = jacksonObjectMapper()

    private fun envelope(deviceName: String?) = TransmitEnvelope(
        dispatchId = UUID.fromString("11111111-1111-1111-1111-111111111111"),
        receiverPublicId = "rx-abc123",
        band = "433M",
        rfCodeHex = "A1B2C3",
        rfCodeBits = 24,
        protoAny = false,
        issuedAt = "2026-06-09T10:00:00Z",
        deviceName = deviceName,
        signatureB64 = "c2ln"
    )

    @Test
    fun `device_name is serialized when assignedName is present`() {
        val node = mapper.readTree(mapper.writeValueAsString(envelope("Table 5")))
        assertThat(node.has("device_name")).isTrue()
        assertThat(node.get("device_name").asText()).isEqualTo("Table 5")
    }

    @Test
    fun `device_name key is omitted when assignedName is null`() {
        val node = mapper.readTree(mapper.writeValueAsString(envelope(null)))
        assertThat(node.has("device_name")).isFalse()
    }

    @Test
    fun `signed fields remain present with snake_case names`() {
        val node = mapper.readTree(mapper.writeValueAsString(envelope("Table 5")))
        assertThat(node.has("receiver_public_id")).isTrue()
        assertThat(node.has("rf_code_hex")).isTrue()
        assertThat(node.has("signature_b64")).isTrue()
        assertThat(node.get("schema_version").asInt()).isEqualTo(1)
    }
}
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.device.service.TransmitEnvelopeSerializationTest"`
Expected: **FAIL at compilation** — `TransmitEnvelope` is currently file-`private` and has no `deviceName` parameter (`Cannot access 'TransmitEnvelope': it is private` / `no value passed for parameter 'deviceName'`).

- [ ] **Step 4: Add the `JsonInclude` import**

In `TransmitterDispatchService.kt`, the file already imports `com.fasterxml.jackson.annotation.JsonProperty`. Add directly beneath it:

```kotlin
import com.fasterxml.jackson.annotation.JsonInclude
```

> `@field:JsonInclude(JsonInclude.Include.NON_NULL)` is verified-current Jackson API (`jackson-databind`, 2.x). When `assignedName` is null the key is omitted entirely; the firmware treats absence and empty string identically.

- [ ] **Step 5: Promote `TransmitEnvelope` to `internal` and add `deviceName`**

Find the data class near the bottom of the file:

```kotlin
private data class TransmitEnvelope(
    @field:JsonProperty("schema_version")
    val schemaVersion: Int = 1,
    @field:JsonProperty("dispatch_id")
    val dispatchId: UUID,
    @field:JsonProperty("receiver_public_id")
    val receiverPublicId: String,
    val band: String,
    @field:JsonProperty("rf_code_hex")
    val rfCodeHex: String,
    @field:JsonProperty("rf_code_bits")
    val rfCodeBits: Int,
    @field:JsonProperty("proto_any")
    val protoAny: Boolean,
    @field:JsonProperty("issued_at")
    val issuedAt: String,
    @field:JsonProperty("signature_b64")
    val signatureB64: String
)
```

Replace it with (changes `private`→`internal`, inserts `deviceName` before `signatureB64`):

```kotlin
internal data class TransmitEnvelope(
    @field:JsonProperty("schema_version")
    val schemaVersion: Int = 1,
    @field:JsonProperty("dispatch_id")
    val dispatchId: UUID,
    @field:JsonProperty("receiver_public_id")
    val receiverPublicId: String,
    val band: String,
    @field:JsonProperty("rf_code_hex")
    val rfCodeHex: String,
    @field:JsonProperty("rf_code_bits")
    val rfCodeBits: Int,
    @field:JsonProperty("proto_any")
    val protoAny: Boolean,
    @field:JsonProperty("issued_at")
    val issuedAt: String,
    @field:JsonProperty("device_name")
    @field:JsonInclude(JsonInclude.Include.NON_NULL)
    val deviceName: String?,
    @field:JsonProperty("signature_b64")
    val signatureB64: String
)
```

- [ ] **Step 6: Set `deviceName` from the receiver in `buildPayload`**

Find the `return TransmitEnvelope(...)` block inside `buildPayload(...)`:

```kotlin
        return TransmitEnvelope(
            dispatchId = dispatchId,
            receiverPublicId = requireNotNull(receiver.publicId),
            band = patched.band,
            rfCodeHex = rfCodeHex,
            rfCodeBits = patched.bits,
            protoAny = patched.protoAny,
            issuedAt = issuedAt.toInstant().toString(),
            signatureB64 = signer.sign(canonical)
        )
```

Replace with (adds the `deviceName` named argument only — `canonical` and `signer.sign(...)` are untouched):

```kotlin
        return TransmitEnvelope(
            dispatchId = dispatchId,
            receiverPublicId = requireNotNull(receiver.publicId),
            band = patched.band,
            rfCodeHex = rfCodeHex,
            rfCodeBits = patched.bits,
            protoAny = patched.protoAny,
            issuedAt = issuedAt.toInstant().toString(),
            deviceName = receiver.assignedName,
            signatureB64 = signer.sign(canonical)
        )
```

> Do **not** add `deviceName` to `DeviceCanonical.transmit(...)` and do **not** pass it to `signer.sign(...)`. The signed canonical must stay `transmit-v1|...|issued_at` exactly. (`SlotDispatchEnvelope` is unchanged — the firmware already holds receiver names locally in its roster, synced via the `cmd/label` topic.)

- [ ] **Step 7: Run the test to verify it passes**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.device.service.TransmitEnvelopeSerializationTest"`
Expected: **PASS** — 3 tests green.

- [ ] **Step 8: Confirm scope (project rule)**

Run: `gitnexus_detect_changes({scope: "all"})`
Expected: only `TransmitterDispatchService.kt` and the new test file reported as changed.

---

## Part B — Case B: dispatch ack-timeout reconciliation

### Design notes (read before B1–B6)

- **Two keys, both keyed only by `dispatchId`:** `dispatch:tracking:{id}` (the data record, 5-min TTL — already written today) and `dispatch:pending-ack:{id}` (a tiny `"1"` timer, TTL = `dispatchAckTimeoutSeconds` = 45 s). The timer's TTL must stay **well below** the tracking TTL so the record still exists when the timer fires (45 s ≪ 5 min ✓).
- **Why a separate timer key:** a Redis keyspace `expired` event delivers only the **key name**, not the value. Encoding everything in one key would force the ack path (which only knows `dispatchId`) to reconstruct a multi-field key to delete it. Two `dispatchId`-only keys keep both the ack path and the expiry path simple.
- **Atomic claim via GET → DEL-count (not `getAndDelete`):** `getAndDelete`/GETDEL was **not** confirmed available by Context7 for this Spring Data Redis version, so this plan deliberately uses only `opsForValue().get(...)`, `delete(...)`, and `opsForValue().set(..., Duration)` — all already used in this codebase. The arbiter is the **`delete` return count**: when two callers race (a late ack vs. the timeout), each reads the record but only the one whose `DEL` returns `1` performs the side-effects; the loser (count `0`) no-ops. This is a hardening of the exact GET-then-DEL shape the current `handleTransmitRejection` already uses.
- **Timer armed for CALL dispatches only.** Case B is a *call*-side deadlock: a lost `DEVICE_CALL_REQUESTED` leaves the device busy with no way to free it. `DEVICE_STOP_REQUESTED` dispatches still write the tracking record (so the rejection path keeps working) but do **not** arm a timer — a lost stop creates no new stuck-busy state, and arming one would (a) emit a `DEVICE_DISPATCH_FAILED` for a timed-out stop that today surfaces nothing, and (b) widen the cross-dispatch window described next.
- **Busy release is ticket-scoped.** `failDispatch` frees `device:busy:{deviceId}` only when the *current* busy record's `ticketId` matches the timed-out dispatch's ticket. Rationale: a `STOP`+`RELEASE` frees the device early, so a **different** ticket may re-reserve the same device before an earlier, still-armed `CALL` timer fires; an unguarded release would then wrongly free the new ticket's lock. The check is read-then-conditional-delete (a microsecond TOCTOU window remains, identical in size to the pre-existing rejection path — acceptable, not introduced here). This guard also hardens the rejection path, which previously released the lock unconditionally.
- **Disposition mirrors today's rejection path:** release the (matching) `device:busy` lock + emit one SSE `DEVICE_DISPATCH_FAILED`. The queue ticket itself is **not** cancelled or moved (identical to a hub rejection). On **success** the busy lock is intentionally **retained** (the receiver is now serving).
- **Degrades safely:** if the keyspace listener is down, both keys self-expire (timer at 45 s, tracking at 5 min) — worst case reverts to today's behavior, no leak.

---

### Task B1: `RedisKeyManager` pending-ack helpers (TDD)

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/core/redis/RedisKeyManager.kt`
- Test (modify): `backend/src/test/kotlin/com/thomas/notiguide/core/redis/RedisKeyManagerTest.kt`

- [ ] **Step 1: Impact analysis**

Run: `gitnexus_impact({target: "RedisKeyManager", direction: "upstream"})`
Expected: many read-only call sites; the change only **adds** functions, so existing callers are unaffected (additive, LOW risk).

- [ ] **Step 2: Write the failing tests**

In `RedisKeyManagerTest.kt`, add a `dispatchId` field next to the existing ids and append these tests inside the class:

```kotlin
    private val dispatchId = UUID.fromString("33333333-3333-3333-3333-333333333333")

    @Test
    fun `dispatchPendingAck key has expected format`() {
        assertThat(RedisKeyManager.dispatchPendingAck(dispatchId))
            .isEqualTo("dispatch:pending-ack:$dispatchId")
    }

    @Test
    fun `isPendingAckKey recognises pending-ack keys only`() {
        assertThat(RedisKeyManager.isPendingAckKey(RedisKeyManager.dispatchPendingAck(dispatchId))).isTrue()
        assertThat(RedisKeyManager.isPendingAckKey(RedisKeyManager.dispatchTracking(dispatchId))).isFalse()
        assertThat(RedisKeyManager.isPendingAckKey(RedisKeyManager.ticket(storeId, ticketId))).isFalse()
    }

    @Test
    fun `parsePendingAckKey round-trips a pending-ack key`() {
        val key = RedisKeyManager.dispatchPendingAck(dispatchId)
        assertThat(RedisKeyManager.parsePendingAckKey(key)).isEqualTo(dispatchId)
    }

    @Test
    fun `parsePendingAckKey returns null for a non-uuid suffix`() {
        assertThat(RedisKeyManager.parsePendingAckKey("dispatch:pending-ack:not-a-uuid")).isNull()
    }
```

- [ ] **Step 3: Run the tests to verify they fail**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.core.redis.RedisKeyManagerTest"`
Expected: **FAIL at compilation** — `dispatchPendingAck` / `isPendingAckKey` / `parsePendingAckKey` are unresolved references.

- [ ] **Step 4: Add the key builder**

In `RedisKeyManager.kt`, find:

```kotlin
    fun dispatchTracking(dispatchId: UUID): String = "dispatch:tracking:$dispatchId"
```

Add directly beneath it:

```kotlin
    fun dispatchPendingAck(dispatchId: UUID): String = "dispatch:pending-ack:$dispatchId"
```

- [ ] **Step 5: Add the predicate and parser**

Find:

```kotlin
    fun isTicketKey(key: String) = key.startsWith("ticket:")
    fun isGraceExpiryKey(key: String) = key.startsWith("grace:")
```

Add directly beneath them:

```kotlin
    fun isPendingAckKey(key: String) = key.startsWith("dispatch:pending-ack:")

    fun parsePendingAckKey(key: String): UUID? =
        try {
            UUID.fromString(key.removePrefix("dispatch:pending-ack:"))
        } catch (_: IllegalArgumentException) {
            null
        }
```

- [ ] **Step 6: Run the tests to verify they pass**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.core.redis.RedisKeyManagerTest"`
Expected: **PASS** — existing tests plus the 4 new ones green.

---

### Task B2: `dispatchAckTimeoutSeconds` property + yaml

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/core/device/DeviceTransmitterProperties.kt`
- Modify: `backend/src/main/resources/application.yaml`

*(Config-only task — no unit test; the `require` mirrors the existing validations.)*

- [ ] **Step 1: Add the property and its guard**

Find:

```kotlin
@ConfigurationProperties(prefix = "device.transmitter")
data class DeviceTransmitterProperties(
    val enabled: Boolean = true,
    val heartbeatIntervalSeconds: Long = 10,
    val heartbeatLivenessSeconds: Long = 30,
    val activeCacheSeconds: Long = 60,
    val maxRegisteredPerStore: Int = 3,
    val diagnosticsCacheSeconds: Long = 45
) {
    init {
        require(heartbeatIntervalSeconds > 0) { "Heartbeat intervals must be positive" }
        require(heartbeatLivenessSeconds > 0) { "Heartbeat keep-alive period must be positive" }
        require(activeCacheSeconds > 0) { "Cache TTL must be positive" }
        require(maxRegisteredPerStore > 0) { "Max active transmitter count must be positive" }
        require(diagnosticsCacheSeconds > heartbeatIntervalSeconds) { "Diagnostic cache must exceed heartbeat interval" }
    }
}
```

Replace with (adds `dispatchAckTimeoutSeconds` and a `require`):

```kotlin
@ConfigurationProperties(prefix = "device.transmitter")
data class DeviceTransmitterProperties(
    val enabled: Boolean = true,
    val heartbeatIntervalSeconds: Long = 10,
    val heartbeatLivenessSeconds: Long = 30,
    val activeCacheSeconds: Long = 60,
    val maxRegisteredPerStore: Int = 3,
    val diagnosticsCacheSeconds: Long = 45,
    val dispatchAckTimeoutSeconds: Long = 45
) {
    init {
        require(heartbeatIntervalSeconds > 0) { "Heartbeat intervals must be positive" }
        require(heartbeatLivenessSeconds > 0) { "Heartbeat keep-alive period must be positive" }
        require(activeCacheSeconds > 0) { "Cache TTL must be positive" }
        require(maxRegisteredPerStore > 0) { "Max active transmitter count must be positive" }
        require(diagnosticsCacheSeconds > heartbeatIntervalSeconds) { "Diagnostic cache must exceed heartbeat interval" }
        require(dispatchAckTimeoutSeconds > 0) { "Dispatch ack timeout must be positive" }
    }
}
```

- [ ] **Step 2: Add the default to `application.yaml`**

Find:

```yaml
  transmitter:
    enabled: ${DEVICE_TRANSMITTER_ENABLED:true}
    heartbeat-interval-seconds: 10
    heartbeat-liveness-seconds: 30
    active-cache-seconds: 60
    max-registered-per-store: 3
```

Replace with:

```yaml
  transmitter:
    enabled: ${DEVICE_TRANSMITTER_ENABLED:true}
    heartbeat-interval-seconds: 10
    heartbeat-liveness-seconds: 30
    active-cache-seconds: 60
    max-registered-per-store: 3
    dispatch-ack-timeout-seconds: 45
```

> `application-prod.yaml` has **no** `device.transmitter` block, so it inherits this default via Spring profile merging — **no prod-yaml change is needed.** (The tracking-record TTL stays the hardcoded 5 minutes in `TransmitterDispatchService`; the invariant `dispatchAckTimeoutSeconds (45s) < tracking TTL (5m)` holds with wide margin. An over-large override fails safe: the timer outlives the record, so the reconciler reads `null` and no-ops.)

---

### Task B3: `DispatchReconciliationService` (TDD)

**Files:**
- Create: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/service/DispatchReconciliationService.kt`
- Create: `backend/src/test/kotlin/com/thomas/notiguide/domain/device/service/DispatchReconciliationServiceTest.kt`

- [ ] **Step 1: Write the failing test**

Create `backend/src/test/kotlin/com/thomas/notiguide/domain/device/service/DispatchReconciliationServiceTest.kt`:

```kotlin
package com.thomas.notiguide.domain.device.service

import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import com.thomas.notiguide.core.redis.RedisKeyManager
import com.thomas.notiguide.core.sse.QueueEventBroadcaster
import com.thomas.notiguide.core.sse.QueueSseEvent
import com.thomas.notiguide.domain.device.dto.DispatchTrackingRecord
import com.thomas.notiguide.domain.device.redis.DeviceBusyRecord
import io.mockk.every
import io.mockk.mockk
import io.mockk.slot
import io.mockk.verify
import kotlinx.coroutines.test.runTest
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.data.redis.core.ReactiveRedisTemplate
import org.springframework.data.redis.core.ReactiveValueOperations
import reactor.core.publisher.Mono
import java.time.OffsetDateTime
import java.time.ZoneOffset
import java.util.UUID

class DispatchReconciliationServiceTest {

    private val redis = mockk<ReactiveRedisTemplate<String, String>>()
    private val valueOps = mockk<ReactiveValueOperations<String, String>>()
    private val broadcaster = mockk<QueueEventBroadcaster>(relaxed = true)

    // findAndRegisterModules() pulls in JavaTimeModule so DeviceBusyRecord's OffsetDateTime round-trips,
    // matching the Spring-configured ObjectMapper the service receives in production.
    private val objectMapper = jacksonObjectMapper().findAndRegisterModules()
    private val service = DispatchReconciliationService(redis, objectMapper, broadcaster)

    private val dispatchId = UUID.fromString("33333333-3333-3333-3333-333333333333")
    private val deviceId = UUID.fromString("44444444-4444-4444-4444-444444444444")
    private val storeId = UUID.fromString("55555555-5555-5555-5555-555555555555")
    private val ticketId = UUID.fromString("66666666-6666-6666-6666-666666666666")

    private val trackingKey = RedisKeyManager.dispatchTracking(dispatchId)
    private val pendingAckKey = RedisKeyManager.dispatchPendingAck(dispatchId)
    private val busyKey = RedisKeyManager.deviceBusy(deviceId)

    private fun trackingJson(): String = objectMapper.writeValueAsString(
        DispatchTrackingRecord(
            deviceId = deviceId,
            storeId = storeId,
            ticketId = ticketId,
            ticketNumber = "A001"
        )
    )

    private fun busyJson(ticket: UUID): String = objectMapper.writeValueAsString(
        DeviceBusyRecord(
            storeId = storeId,
            ticketId = ticket,
            boundAt = OffsetDateTime.of(2026, 6, 10, 10, 0, 0, 0, ZoneOffset.UTC)
        )
    )

    @Test
    fun `failDispatch claims, releases the matching busy lock, and emits DEVICE_DISPATCH_FAILED`() = runTest {
        every { redis.opsForValue() } returns valueOps
        every { valueOps.get(trackingKey) } returns Mono.just(trackingJson())
        every { redis.delete(trackingKey) } returns Mono.just(1L)
        every { redis.delete(pendingAckKey) } returns Mono.just(1L)
        every { valueOps.get(busyKey) } returns Mono.just(busyJson(ticketId))
        every { redis.delete(busyKey) } returns Mono.just(1L)

        service.failDispatch(dispatchId, "ack_timeout")

        val event = slot<QueueSseEvent>()
        verify(exactly = 1) { redis.delete(busyKey) }
        verify(exactly = 1) { broadcaster.broadcast(capture(event)) }
        assertThat(event.captured.type).isEqualTo("DEVICE_DISPATCH_FAILED")
        assertThat(event.captured.reason).isEqualTo("ack_timeout")
        assertThat(event.captured.ticketId).isEqualTo(ticketId)
        assertThat(event.captured.ticketNumber).isEqualTo("A001")
    }

    @Test
    fun `failDispatch does not release the busy lock when it belongs to a newer ticket`() = runTest {
        val otherTicket = UUID.fromString("77777777-7777-7777-7777-777777777777")
        every { redis.opsForValue() } returns valueOps
        every { valueOps.get(trackingKey) } returns Mono.just(trackingJson())
        every { redis.delete(trackingKey) } returns Mono.just(1L)
        every { redis.delete(pendingAckKey) } returns Mono.just(1L)
        every { valueOps.get(busyKey) } returns Mono.just(busyJson(otherTicket))

        service.failDispatch(dispatchId, "ack_timeout")

        verify(exactly = 0) { redis.delete(busyKey) }
        verify(exactly = 1) { broadcaster.broadcast(any()) }
    }

    @Test
    fun `failDispatch is idempotent when the record is already gone`() = runTest {
        every { redis.opsForValue() } returns valueOps
        every { valueOps.get(trackingKey) } returns Mono.empty()

        service.failDispatch(dispatchId, "ack_timeout")

        verify(exactly = 0) { broadcaster.broadcast(any()) }
        verify(exactly = 0) { redis.delete(busyKey) }
    }

    @Test
    fun `failDispatch no-ops when it loses the delete race`() = runTest {
        every { redis.opsForValue() } returns valueOps
        every { valueOps.get(trackingKey) } returns Mono.just(trackingJson())
        every { redis.delete(trackingKey) } returns Mono.just(0L)

        service.failDispatch(dispatchId, "ack_timeout")

        verify(exactly = 0) { broadcaster.broadcast(any()) }
        verify(exactly = 0) { redis.delete(busyKey) }
    }

    @Test
    fun `completeDispatch clears both keys without emitting or releasing busy`() = runTest {
        every { redis.delete(pendingAckKey) } returns Mono.just(1L)
        every { redis.delete(trackingKey) } returns Mono.just(1L)

        service.completeDispatch(dispatchId)

        verify(exactly = 1) { redis.delete(trackingKey) }
        verify(exactly = 1) { redis.delete(pendingAckKey) }
        verify(exactly = 0) { broadcaster.broadcast(any()) }
        verify(exactly = 0) { redis.delete(busyKey) }
    }
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.device.service.DispatchReconciliationServiceTest"`
Expected: **FAIL at compilation** — `DispatchReconciliationService` does not exist yet.

- [ ] **Step 3: Create the service**

Create `backend/src/main/kotlin/com/thomas/notiguide/domain/device/service/DispatchReconciliationService.kt`:

```kotlin
package com.thomas.notiguide.domain.device.service

import com.fasterxml.jackson.databind.ObjectMapper
import com.thomas.notiguide.core.mqtt.MqttPublisher.QueueEventType
import com.thomas.notiguide.core.redis.RedisKeyManager
import com.thomas.notiguide.core.sse.QueueEventBroadcaster
import com.thomas.notiguide.core.sse.QueueSseEvent
import com.thomas.notiguide.domain.device.dto.DispatchTrackingRecord
import com.thomas.notiguide.domain.device.redis.DeviceBusyRecord
import kotlinx.coroutines.reactor.awaitSingleOrNull
import org.slf4j.LoggerFactory
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty
import org.springframework.data.redis.core.ReactiveRedisTemplate
import org.springframework.stereotype.Service
import java.util.UUID

/**
 * Reconciles dispatches that ended without a successful ack — either rejected by the hub or never
 * acknowledged (the hub died after publish). Shared by [TransmitterOperationalListener] (rejection
 * path) and [com.thomas.notiguide.core.redis.RedisKeyExpirationListener] (ack-timeout path).
 */
@Service
@ConditionalOnProperty(prefix = "device.transmitter", name = ["enabled"], havingValue = "true")
class DispatchReconciliationService(
    private val redis: ReactiveRedisTemplate<String, String>,
    private val objectMapper: ObjectMapper,
    private val queueEventBroadcaster: QueueEventBroadcaster
) {

    private val log = LoggerFactory.getLogger(this::class.java)

    /**
     * Atomically claims a dispatch and marks it failed: releases the device-busy lock (only if it still
     * belongs to this dispatch's ticket) and emits a DEVICE_DISPATCH_FAILED SSE event. Idempotent — only
     * the caller whose DEL removes the tracking record performs the side-effects, so a racing ack and
     * timeout can never both fire.
     */
    suspend fun failDispatch(dispatchId: UUID, reason: String) {
        val trackingKey = RedisKeyManager.dispatchTracking(dispatchId)

        val json = runCatching {
            redis.opsForValue().get(trackingKey).awaitSingleOrNull()
        }.getOrElse { ex ->
            log.warn("dispatch_tracking_read_failed dispatch={}", dispatchId, ex)
            return
        } ?: return // already reconciled — idempotent no-op

        val claimed = runCatching {
            redis.delete(trackingKey).awaitSingleOrNull()
        }.getOrElse { ex ->
            log.warn("dispatch_tracking_delete_failed dispatch={}", dispatchId, ex)
            return
        } ?: 0L
        if (claimed == 0L) return // lost the race to a concurrent ack/timeout

        // Best-effort: disarm the timer so it does not fire a no-op expiry later.
        runCatching {
            redis.delete(RedisKeyManager.dispatchPendingAck(dispatchId)).awaitSingleOrNull()
        }

        val tracking = runCatching {
            objectMapper.readValue(json, DispatchTrackingRecord::class.java)
        }.getOrElse { ex ->
            log.warn("dispatch_tracking_malformed dispatch={}", dispatchId, ex)
            return
        }

        releaseBusyIfOwned(tracking.deviceId, tracking.ticketId)

        queueEventBroadcaster.broadcast(
            QueueSseEvent(
                type = QueueEventType.DEVICE_DISPATCH_FAILED.name,
                storeId = tracking.storeId,
                ticketId = tracking.ticketId,
                ticketNumber = tracking.ticketNumber,
                reason = reason
            )
        )
        log.warn(
            "dispatch_failed_reconciled dispatch={} reason={} device={}",
            dispatchId, reason, tracking.deviceId
        )
    }

    /**
     * Marks a dispatch as successfully applied: disarms the timer and drops the tracking record so the
     * timeout path cannot fire. Emits nothing and intentionally retains the device-busy lock (the
     * receiver is now serving).
     */
    suspend fun completeDispatch(dispatchId: UUID) {
        runCatching {
            redis.delete(RedisKeyManager.dispatchPendingAck(dispatchId)).awaitSingleOrNull()
            redis.delete(RedisKeyManager.dispatchTracking(dispatchId)).awaitSingleOrNull()
        }.onFailure { ex ->
            log.warn("dispatch_complete_cleanup_failed dispatch={}", dispatchId, ex)
        }
    }

    /**
     * Frees `device:busy:{deviceId}` only when the current busy record's ticket matches [ticketId]. A
     * later dispatch may have re-reserved the same device (e.g. after a STOP+RELEASE); we must not free
     * its lock. Absent busy key ⇒ nothing to release.
     */
    private suspend fun releaseBusyIfOwned(deviceId: UUID, ticketId: UUID) {
        runCatching {
            val busyKey = RedisKeyManager.deviceBusy(deviceId)
            val busyJson = redis.opsForValue().get(busyKey).awaitSingleOrNull() ?: return
            val busy = runCatching {
                objectMapper.readValue(busyJson, DeviceBusyRecord::class.java)
            }.getOrNull()
            if (busy?.ticketId == ticketId) {
                redis.delete(busyKey).awaitSingleOrNull()
            } else {
                log.info("dispatch_busy_release_skipped device={} reason=ticket_mismatch", deviceId)
            }
        }.onFailure { ex ->
            log.warn("dispatch_busy_release_failed device={}", deviceId, ex)
        }
    }
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.device.service.DispatchReconciliationServiceTest"`
Expected: **PASS** — 5 tests green.

- [ ] **Step 5: Confirm scope**

Run: `gitnexus_detect_changes({scope: "all"})`
Expected: only the new service + its test reported as changed.

---

### Task B4: Arm the timer in `TransmitterDispatchService` (both paths, CALL only)

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/service/TransmitterDispatchService.kt`

*(Edit-only — no isolated harness for this class; verified by the audit/integration flow.)*

- [ ] **Step 1: Impact analysis**

Run: `gitnexus_impact({target: "TransmitterDispatchService", direction: "upstream"})`
Expected: it is a `SmartInitializingSingleton` driven by the dispatch event stream; adding a constructor dependency and a private helper is LOW risk. Confirm no caller constructs it manually (Spring DI only). If HIGH/CRITICAL, stop and report.

- [ ] **Step 2: Import `DeviceTransmitterProperties`**

The file already imports `com.thomas.notiguide.core.device.DeviceCommandSigningProperties` and uses `DeviceDispatchEventType` (in `patchRfCode` and the STOP+RELEASE check), so that enum is already imported. Add (named import, alongside the other `core.device` imports):

```kotlin
import com.thomas.notiguide.core.device.DeviceTransmitterProperties
```

- [ ] **Step 3: Inject `DeviceTransmitterProperties` into the constructor**

Find the tail of the constructor:

```kotlin
    private val objectMapper: ObjectMapper,
    private val mqttPublisher: MqttPublisher? = null
) : SmartInitializingSingleton {
```

Replace with (inserts `transmitterProperties` before the trailing nullable param):

```kotlin
    private val objectMapper: ObjectMapper,
    private val transmitterProperties: DeviceTransmitterProperties,
    private val mqttPublisher: MqttPublisher? = null
) : SmartInitializingSingleton {
```

- [ ] **Step 4: Add a `trackDispatch` helper (always tracks; arms the timer for CALL only)**

Add this private method to the class (e.g. directly above `private fun buildPayload(`):

```kotlin
    private suspend fun trackDispatch(dispatchId: UUID, event: DeviceDispatchEvent) {
        runCatching {
            val trackingPayload = objectMapper.writeValueAsString(
                DispatchTrackingRecord(
                    deviceId = event.deviceId,
                    storeId = event.storeId,
                    ticketId = event.ticketId,
                    ticketNumber = event.ticketNumber
                )
            )
            redis.opsForValue()
                .set(
                    RedisKeyManager.dispatchTracking(dispatchId),
                    trackingPayload,
                    Duration.ofMinutes(5)
                )
                .awaitSingleOrNull()
            // Arm the ack-timeout timer only for CALL dispatches (the call-side deadlock of Case B), and
            // only AFTER the record exists so a timeout always finds it. STOP dispatches keep the tracking
            // record for the rejection path but arm no timer.
            if (event.type == DeviceDispatchEventType.DEVICE_CALL_REQUESTED) {
                redis.opsForValue()
                    .set(
                        RedisKeyManager.dispatchPendingAck(dispatchId),
                        "1",
                        Duration.ofSeconds(transmitterProperties.dispatchAckTimeoutSeconds)
                    )
                    .awaitSingleOrNull()
            }
        }.onFailure { ex ->
            log.warn("dispatch_tracking_write_failed dispatch={}", dispatchId, ex)
        }
    }
```

- [ ] **Step 5: Replace the inline tracking-write in the full-payload path with the helper**

In `handle(...)`, find this block (it appears **twice** in the file — this is the first occurrence, in the full-payload path, right after `publisher.publishTransmit(...)`):

```kotlin
        runCatching {
            val trackingPayload = objectMapper.writeValueAsString(
                DispatchTrackingRecord(
                    deviceId = event.deviceId,
                    storeId = event.storeId,
                    ticketId = event.ticketId,
                    ticketNumber = event.ticketNumber
                )
            )
            redis.opsForValue()
                .set(
                    RedisKeyManager.dispatchTracking(dispatchId),
                    trackingPayload,
                    Duration.ofMinutes(5)
                )
                .awaitSingleOrNull()
        }.onFailure { ex ->
            log.warn("dispatch_tracking_write_failed dispatch={}", dispatchId, ex)
        }
```

Replace it with:

```kotlin
        trackDispatch(dispatchId, event)
```

- [ ] **Step 6: Replace the same block in the slot path**

In `handleSlotDispatch(...)`, the **second** (identical) occurrence of that `runCatching { ... DispatchTrackingRecord ... dispatch_tracking_write_failed ... }` block appears right after `publisher.publishTransmit(hubPublicId, envelope)`. Replace that occurrence with the same call:

```kotlin
        trackDispatch(dispatchId, event)
```

> Both dispatch paths now track through one helper, so they cannot diverge. The existing `DEVICE_STOP_REQUESTED` + `RELEASE` busy-release blocks that follow each tracking write are **unchanged**.

- [ ] **Step 7: Confirm scope**

Run: `gitnexus_detect_changes({scope: "all"})`
Expected: only `TransmitterDispatchService.kt` reported as changed.

---

### Task B5: Wire ack outcomes in `TransmitterOperationalListener`

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/listener/TransmitterOperationalListener.kt`

*(Edit-only — no isolated harness for this class.)*

- [ ] **Step 1: Impact analysis**

Run: `gitnexus_impact({target: "handleTransmitRejection", direction: "upstream"})`
Expected: a single caller, `handleAck` (same file). Confirm `handleTransmitRejection` is file-local and that removing it + the `queueEventBroadcaster` constructor param affects nothing outside this class (Spring DI only). If HIGH/CRITICAL, stop and report.

- [ ] **Step 2: Swap imports**

Remove these now-unused imports:

```kotlin
import com.thomas.notiguide.core.mqtt.MqttPublisher.QueueEventType
import com.thomas.notiguide.core.sse.QueueEventBroadcaster
import com.thomas.notiguide.core.sse.QueueSseEvent
import com.thomas.notiguide.domain.device.dto.DispatchTrackingRecord
```

Add this import (alongside the other `domain.device.service` imports):

```kotlin
import com.thomas.notiguide.domain.device.service.DispatchReconciliationService
```

> `import com.thomas.notiguide.core.redis.RedisKeyManager` stays (still used by `handleHeartbeat`); `awaitSingleOrNull` stays (used by `touchHub`); `Duration` stays (used by `handleHeartbeat`).

- [ ] **Step 3: Replace the `queueEventBroadcaster` dependency with the reconciler**

Find the tail of the constructor:

```kotlin
    private val electionService: TransmitterElectionService,
    private val queueEventBroadcaster: QueueEventBroadcaster
) : SmartInitializingSingleton {
```

Replace with:

```kotlin
    private val electionService: TransmitterElectionService,
    private val dispatchReconciliationService: DispatchReconciliationService
) : SmartInitializingSingleton {
```

> Both this listener and `DispatchReconciliationService` are gated by `@ConditionalOnProperty(device.transmitter.enabled=true)`, so whenever the listener bean exists the reconciler bean exists too — direct (non-optional) injection is safe.

- [ ] **Step 4: Route ack outcomes to the reconciler**

In `handleAck(...)`, find the `transmit` branch:

```kotlin
            ack.ackFor == "transmit" -> {
                val seenAt = ack.appliedAt ?: OffsetDateTime.now(ZoneOffset.UTC)
                touchHub(publicId, seenAt)
                if (ack.dispatchId == null) {
                    log.warn("Dropping transmitter ack without dispatch_id for {}", publicId)
                    return
                }
                log.info(
                    "Transmitter dispatch ack: publicId={} dispatchId={} status={} reason={}",
                    publicId,
                    ack.dispatchId,
                    ack.status,
                    ack.reason
                )
                if (ack.status != "applied" && ack.status != "unchanged") {
                    handleTransmitRejection(ack.dispatchId, publicId, ack.status, ack.reason)
                }
            }
```

Replace with:

```kotlin
            ack.ackFor == "transmit" -> {
                val seenAt = ack.appliedAt ?: OffsetDateTime.now(ZoneOffset.UTC)
                touchHub(publicId, seenAt)
                val dispatchId = ack.dispatchId
                if (dispatchId == null) {
                    log.warn("Dropping transmitter ack without dispatch_id for {}", publicId)
                    return
                }
                log.info(
                    "Transmitter dispatch ack: publicId={} dispatchId={} status={} reason={}",
                    publicId,
                    dispatchId,
                    ack.status,
                    ack.reason
                )
                if (ack.status == "applied" || ack.status == "unchanged") {
                    dispatchReconciliationService.completeDispatch(dispatchId)
                } else {
                    dispatchReconciliationService.failDispatch(
                        dispatchId,
                        "transmit_rejected:${ack.reason ?: ack.status}"
                    )
                }
            }
```

> This preserves the existing rejection reason string (`transmit_rejected:<reason|status>`) and now also calls `completeDispatch` on success so the timer is disarmed.

- [ ] **Step 5: Delete the now-redundant `handleTransmitRejection`**

Remove the entire method (its busy-release + SSE-emit logic now lives in `DispatchReconciliationService.failDispatch`):

```kotlin
    private suspend fun handleTransmitRejection(
        dispatchId: UUID?,
        publicId: String,
        status: String,
        reason: String?
    ) {
        if (dispatchId == null) return

        val trackingKey = RedisKeyManager.dispatchTracking(dispatchId)
        val trackingJson = redis.opsForValue()
            .get(trackingKey)
            .awaitSingleOrNull() ?: run {
            log.warn("dispatch_rejection_no_tracking dispatch={}", dispatchId)
            return
        }

        val tracking = runCatching {
            objectMapper.readValue(trackingJson, DispatchTrackingRecord::class.java)
        }.getOrElse {
            log.warn("dispatch_rejection_malformed_tracking dispatch={}", dispatchId, it)
            return
        }

        runCatching {
            redis.delete(RedisKeyManager.deviceBusy(tracking.deviceId)).awaitSingleOrNull()
        }.onFailure { ex ->
            log.warn(
                "dispatch_rejection_busy_release_failed device={}",
                tracking.deviceId, ex
            )
        }

        queueEventBroadcaster.broadcast(
            QueueSseEvent(
                type = QueueEventType.DEVICE_DISPATCH_FAILED.name,
                storeId = tracking.storeId,
                ticketId = tracking.ticketId,
                ticketNumber = tracking.ticketNumber,
                reason = "transmit_rejected:${reason ?: status}"
            )
        )

        runCatching { redis.delete(trackingKey).awaitSingleOrNull() }

        log.warn(
            "dispatch_rejected_by_hub publicId={} dispatch={} status={} reason={} device={}",
            publicId, dispatchId, status, reason, tracking.deviceId
        )
    }
```

> After deletion, verify `redis`, `objectMapper`, `RedisKeyManager`, `awaitSingleOrNull`, and `Duration` are still referenced elsewhere in the file (they are — by `handleHeartbeat`/`touchHub`) and that `queueEventBroadcaster`, `QueueSseEvent`, `QueueEventType`, and `DispatchTrackingRecord` are no longer referenced (they aren't).
>
> **Behavior note (intentional):** the rejection path now releases the busy lock only when it still belongs to the rejected dispatch's ticket (via `failDispatch` → `releaseBusyIfOwned`), where it previously released unconditionally. This is a strict correctness improvement — see the Part B "Busy release is ticket-scoped" design note.

- [ ] **Step 6: Confirm scope**

Run: `gitnexus_detect_changes({scope: "all"})`
Expected: only `TransmitterOperationalListener.kt` reported as changed.

---

### Task B6: Fire reconciliation on keyspace expiry in `RedisKeyExpirationListener`

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/core/redis/RedisKeyExpirationListener.kt`

*(Edit-only — keyspace-subscription infra; verified by the audit/integration flow. Redis `notify-keyspace-events Ex` is already enabled in both dev and prod `redis.conf`.)*

- [ ] **Step 1: Impact analysis**

Run: `gitnexus_impact({target: "RedisKeyExpirationListener", direction: "upstream"})`
Expected: a `@Component` started by `ApplicationReadyEvent`; adding an optional constructor dependency and one branch is LOW risk. If HIGH/CRITICAL, stop and report.

- [ ] **Step 2: Add imports**

Add (alongside the existing imports):

```kotlin
import com.thomas.notiguide.domain.device.service.DispatchReconciliationService
import org.springframework.beans.factory.ObjectProvider
```

> `RedisKeyExpirationListener` already injects `domain.queue` collaborators, so a `domain.device` collaborator follows the same established (core→domain) wiring pattern. It is injected via `ObjectProvider` because the reconciler bean is conditional (absent when the transmitter feature is disabled).

- [ ] **Step 3: Inject the reconciler provider**

Find the constructor:

```kotlin
class RedisKeyExpirationListener(
    private val connectionFactory: ReactiveRedisConnectionFactory,
    private val queueRepo: RedisQueueRepository,
    private val queueService: QueueService
) : DisposableBean {
```

Replace with:

```kotlin
class RedisKeyExpirationListener(
    private val connectionFactory: ReactiveRedisConnectionFactory,
    private val queueRepo: RedisQueueRepository,
    private val queueService: QueueService,
    private val dispatchReconcilerProvider: ObjectProvider<DispatchReconciliationService>
) : DisposableBean {
```

- [ ] **Step 4: Handle pending-ack expiry in the collect block**

Find (inside `subscribe()`) the grace block followed by the ticket check:

```kotlin
                    if (RedisKeyManager.isGraceExpiryKey(expiredKey)) {
                        val (storeId, ticketId) = RedisKeyManager.parseGraceExpiryKey(expiredKey) ?: return@collect
                        log.info("Grace period expired: store={} ticket={}", storeId, ticketId)
                        try {
                            queueService.handleNoShow(storeId, ticketId)
                        } catch (e: Exception) {
                            log.error("Failed to handle no-show: store={} ticket={}", storeId, ticketId, e)
                        }
                        return@collect
                    }

                    if (!RedisKeyManager.isTicketKey(expiredKey)) return@collect
```

Replace with (inserts the pending-ack branch between the grace block and the ticket check):

```kotlin
                    if (RedisKeyManager.isGraceExpiryKey(expiredKey)) {
                        val (storeId, ticketId) = RedisKeyManager.parseGraceExpiryKey(expiredKey) ?: return@collect
                        log.info("Grace period expired: store={} ticket={}", storeId, ticketId)
                        try {
                            queueService.handleNoShow(storeId, ticketId)
                        } catch (e: Exception) {
                            log.error("Failed to handle no-show: store={} ticket={}", storeId, ticketId, e)
                        }
                        return@collect
                    }

                    if (RedisKeyManager.isPendingAckKey(expiredKey)) {
                        val dispatchId = RedisKeyManager.parsePendingAckKey(expiredKey) ?: return@collect
                        val reconciler = dispatchReconcilerProvider.ifAvailable ?: return@collect
                        log.info("Dispatch ack timed out: dispatch={}", dispatchId)
                        try {
                            reconciler.failDispatch(dispatchId, "ack_timeout")
                        } catch (e: Exception) {
                            log.error("Failed to reconcile timed-out dispatch: dispatch={}", dispatchId, e)
                        }
                        return@collect
                    }

                    if (!RedisKeyManager.isTicketKey(expiredKey)) return@collect
```

> `failDispatch` is a `suspend` function; the `.asFlow().collect { ... }` lambda runs inside the `private suspend fun subscribe()` context (the same place the existing `queueService.handleNoShow(...)` suspend call is made), so calling it directly is correct.

- [ ] **Step 5: Confirm scope**

Run: `gitnexus_detect_changes({scope: "all"})`
Expected: only `RedisKeyExpirationListener.kt` reported as changed.

---

## Part C — Frontend (admin dashboard)

### Task C1: `ack_timeout` dashboard toast

**Files:**
- Modify: `web/src/messages/en.json`
- Modify: `web/src/messages/vi.json`
- Modify: `web/src/app/[locale]/dashboard/queue/page.tsx`

*(Edit-only — matches the existing untested dispatch-reason branches. Verified by `yarn lint` + manual observation; no build per project rule.)*

**Context:** the queue page's SSE handler maps `DEVICE_DISPATCH_FAILED` reasons to toasts. `no_active_transmitter` and `device_not_found` have dedicated messages; every other reason (including the new `ack_timeout`) currently falls through to the generic `errorInfrastructure` toast. This task gives `ack_timeout` its own bilingual message. The translation namespace is `queue` (`const tQueue = useTranslations("queue")`), so the key is `dispatch.errorAckTimeout`.

- [ ] **Step 1: Add the English string**

In `web/src/messages/en.json`, find (inside `queue.dispatch`):

```json
      "errorDeviceNotFound": "Target device could not be found or is no longer available.",
      "errorInfrastructure": "Dispatch failed due to a system issue. Please try again.",
```

Replace with (inserts `errorAckTimeout` between them — keeping the generic catch-all last):

```json
      "errorDeviceNotFound": "Target device could not be found or is no longer available.",
      "errorAckTimeout": "The device didn't respond in time — it may be offline. Please try again.",
      "errorInfrastructure": "Dispatch failed due to a system issue. Please try again.",
```

- [ ] **Step 2: Add the mirrored Vietnamese string**

In `web/src/messages/vi.json`, find (inside `queue.dispatch`, the same position):

```json
      "errorDeviceNotFound": "Không tìm thấy thiết bị hoặc thiết bị không còn khả dụng.",
      "errorInfrastructure": "Gửi tín hiệu thất bại do lỗi hệ thống. Vui lòng thử lại.",
```

Replace with:

```json
      "errorDeviceNotFound": "Không tìm thấy thiết bị hoặc thiết bị không còn khả dụng.",
      "errorAckTimeout": "Thiết bị không phản hồi kịp thời — có thể đã mất kết nối. Vui lòng thử lại.",
      "errorInfrastructure": "Gửi tín hiệu thất bại do lỗi hệ thống. Vui lòng thử lại.",
```

> Structural mirror of the EN string (single sentence, em-dash, same `Vui lòng thử lại.` closer as `errorInfrastructure`). Copy approved in chat per the project's Vietnamese-copy rule.

- [ ] **Step 3: Add the handler branch**

In `web/src/app/[locale]/dashboard/queue/page.tsx`, find the `DEVICE_DISPATCH_FAILED` reason switch:

```tsx
      if (reason === "no_active_transmitter") {
        toast.error(tQueue("dispatch.errorNoActiveTransmitter"));
      } else if (reason === "device_not_found") {
        toast.error(tQueue("dispatch.errorDeviceNotFound"));
      } else {
        toast.error(tQueue("dispatch.errorInfrastructure"));
      }
```

Replace with (adds the `ack_timeout` branch before the catch-all):

```tsx
      if (reason === "no_active_transmitter") {
        toast.error(tQueue("dispatch.errorNoActiveTransmitter"));
      } else if (reason === "device_not_found") {
        toast.error(tQueue("dispatch.errorDeviceNotFound"));
      } else if (reason === "ack_timeout") {
        toast.error(tQueue("dispatch.errorAckTimeout"));
      } else {
        toast.error(tQueue("dispatch.errorInfrastructure"));
      }
```

- [ ] **Step 4: Lint**

Run: `cd web && yarn lint`
Expected: PASS — Biome reports no errors for the edited files (correct JSON, no formatting drift).

> Manual check (audit/integration flow, not run here): dispatch a `CALL` to a device whose hub then goes offline; after `dispatchAckTimeoutSeconds` (~45s) the queue page shows the new timeout toast instead of the generic system-issue toast.

---

## Part D — Documentation

### Task D1: CHANGELOGS entry

**Files:**
- Modify: `docs/CHANGELOGS.md`

- [ ] **Step 1: Prepend a dated entry**

Add this section directly under the top `# Changelogs` heading (above the most recent dated section), matching the file's existing format:

```markdown
## 2026-06-10 — Transmitter dispatch: device name, Case B ack-timeout reconciliation, dashboard toast

Backend dispatch work plus one admin-dashboard string. The firmware side of the device-name feature
had already landed and been reviewed; this entry covers the backend and web changes.

**Part A — Dispatch device name (full-payload path).** `TransmitEnvelope` now carries the target
receiver's `assigned_name` so the firmware OLED can show it (display-only, **unsigned**: the
`transmit-v1` canonical and signature verification are unchanged, `schema_version` stays `1`, fully
backward/forward compatible). Jackson `@JsonInclude(NON_NULL)` usage verified current.
- Modified `domain/device/service/TransmitterDispatchService.kt`: `TransmitEnvelope` promoted
  `private`→`internal`; added optional `@field:JsonProperty("device_name") @field:JsonInclude(NON_NULL)
  deviceName: String?`; `buildPayload(...)` sets `deviceName = receiver.assignedName`. Canonical/signer
  untouched.
- Created `domain/device/service/TransmitEnvelopeSerializationTest.kt`.
- **Skipped by design:** the slot dispatch wire format (`SlotDispatchEnvelope`) — the firmware already
  holds receiver names locally in its roster (synced via the `cmd/label` topic).

**Part B — Case B ack-timeout reconciliation.** Closed the deadlock where a CALL dispatch whose hub
dies after publish was never acknowledged: the `device:busy` lock lingered for 12h and no failure
reached the dashboard. On each successful **CALL** publish the service now arms a short-TTL
`dispatch:pending-ack:{id}` timer alongside the existing tracking record; its Redis keyspace expiry
drives a reconciler that atomically claims the tracking record (GET → DEL-with-count-check), releases
the busy lock **only if it still belongs to that ticket**, and emits
`DEVICE_DISPATCH_FAILED(reason=ack_timeout)`. Spring Data Redis `getAndDelete`/GETDEL could not be
confirmed for this version via Context7, so the claim uses only `get`/`delete`/`set(Duration)` (already
used in this codebase); the `delete` return-count is the race arbiter.
- Added `core/redis/RedisKeyManager.kt`: `dispatchPendingAck`, `isPendingAckKey`, `parsePendingAckKey`
  (+ tests in `RedisKeyManagerTest.kt`).
- Added `core/device/DeviceTransmitterProperties.kt`: `dispatchAckTimeoutSeconds: Long = 45` (+`require`);
  `application.yaml`: `dispatch-ack-timeout-seconds: 45` (prod inherits via profile merge).
- Created `domain/device/service/DispatchReconciliationService.kt` (`failDispatch` / `completeDispatch`,
  atomic claim, ticket-scoped busy release, idempotent) + `DispatchReconciliationServiceTest.kt`.
- Modified `domain/device/service/TransmitterDispatchService.kt`: inject `DeviceTransmitterProperties`;
  extracted a `trackDispatch(...)` helper that always writes tracking and arms the timer for CALL
  dispatches only; called from both dispatch paths.
- Modified `domain/device/listener/TransmitterOperationalListener.kt`: route applied/unchanged→
  `completeDispatch`, rejected→`failDispatch`; deleted the now-redundant `handleTransmitRejection`;
  swapped the `queueEventBroadcaster` dependency for `DispatchReconciliationService`. The rejection path's
  busy release is now ticket-scoped (was unconditional) — a strict correctness improvement.
- Modified `core/redis/RedisKeyExpirationListener.kt`: on `pending-ack` key expiry, delegate to
  `DispatchReconciliationService.failDispatch(..., "ack_timeout")` via `ObjectProvider`.

**Part C — Dashboard timeout toast (web).** The admin queue page now shows a purpose-built bilingual
toast for the new `ack_timeout` dispatch-failure reason instead of the generic system-issue toast.
- Modified `web/src/app/[locale]/dashboard/queue/page.tsx`: added an `ack_timeout` branch to the
  `DEVICE_DISPATCH_FAILED` reason switch.
- Modified `web/src/messages/en.json` and `web/src/messages/vi.json`: added the mirrored
  `queue.dispatch.errorAckTimeout` key.
```

---

## Self-Review

**1. Spec / design coverage**

| Design requirement | Task |
|---|---|
| Full-payload: backend adds optional `device_name` from `assignedName`, excluded from canonical/signer | A1 (Steps 5–6) |
| `@JsonInclude(NON_NULL)` omits null; `internal` promotion for testability | A1 (Steps 4–5) + test Step 2 |
| Two `dispatchId`-keyed keys (tracking + pending-ack); timer TTL ≪ tracking TTL | B1 (keys) + B2 (timeout=45s) + B4 (arm after tracking) |
| Atomic claim via GET → DEL-count (no `getAndDelete`) | B3 (`failDispatch`) |
| Timer armed for CALL dispatches only | B4 (`trackDispatch` type guard) |
| Busy release ticket-scoped (no cross-dispatch free) | B3 (`releaseBusyIfOwned`) |
| Disposition = release matching busy + emit `DEVICE_DISPATCH_FAILED`; ticket untouched; success retains busy | B3 (`failDispatch`/`completeDispatch`) |
| Rejection + timeout consolidated into one path (DRY) | B3 + B5 |
| Timeout triggered by keyspace expiry, gated by transmitter-enabled | B6 (`ObjectProvider`) |
| Both dispatch paths track via one helper | B4 (shared `trackDispatch`) |
| New `ack_timeout` reason has a dedicated bilingual dashboard toast | C1 |
| Backend-only firmware already landed | header + status note |

No design requirement is left without a task.

**2. Placeholder scan:** No `TBD`/`TODO`/"add error handling"/"similar to Task N". Every code step shows full before/after or full file content. ✓

**3. Type/identifier consistency:**
- Key name `dispatch:pending-ack:{id}` consistent across `RedisKeyManager` (B1), the arm site (B4), and the expiry parse (B6). ✓
- `dispatchAckTimeoutSeconds` (B2 property) ↔ `dispatch-ack-timeout-seconds` (B2 yaml) ↔ `transmitterProperties.dispatchAckTimeoutSeconds` (B4). ✓
- `failDispatch(dispatchId, reason)` / `completeDispatch(dispatchId)` signatures identical across B3 (definition), B5 (callers), B6 (caller). ✓
- `QueueSseEvent(type, storeId, ticketId, ticketNumber, reason)` matches the real data class fields. ✓
- `DispatchTrackingRecord(deviceId, storeId, ticketId, ticketNumber)` and `DeviceBusyRecord(storeId, ticketId, boundAt)` match the real data classes. ✓
- `DeviceDispatchEventType.DEVICE_CALL_REQUESTED` matches the enum (only `DEVICE_CALL_REQUESTED` / `DEVICE_STOP_REQUESTED` exist). ✓
- Backend `reason="ack_timeout"` (B3/B6) ↔ frontend `reason === "ack_timeout"` (C1) ↔ i18n key `queue.dispatch.errorAckTimeout` (C1). ✓

**4. UI-rules compliance (Part C):** The web change is a single toast string surfaced via existing `toast.error(...)` + next-intl — no new component, no alert box, no hyperlink, so the shadcn/ui, alert-box, and hyperlink rules do not apply. The **bilingual rule does** apply and is satisfied: a new `queue.dispatch.errorAckTimeout` key is added to **both** `en.json` and `vi.json` in the same position (structural mirror), and the Vietnamese copy was proposed and approved in chat before inclusion per the project's vi-copy rule. ✓

**5. Codebase integration (verified against source):**
- The full-payload and slot tracking-write blocks are byte-identical in the current source, so the single `trackDispatch` helper (B4) correctly replaces both occurrences. ✓
- `RedisKeyExpirationListener.subscribe()` is a `suspend` function and already makes a suspend call (`queueService.handleNoShow`) in the same `collect` lambda, so the suspend `failDispatch` call is valid there. ✓
- Both `TransmitterOperationalListener` and `DispatchReconciliationService` carry `@ConditionalOnProperty(device.transmitter.enabled=true)`, so direct injection (B5) cannot produce a missing-bean error; the core listener uses `ObjectProvider` (B6) because it is unconditional. ✓
- Removing `handleTransmitRejection` frees `queueEventBroadcaster`, `QueueSseEvent`, `QueueEventType`, and `DispatchTrackingRecord` from the listener (B5 Step 5 check); `RedisKeyManager`/`Duration`/`awaitSingleOrNull` remain used. ✓
- Cross-dispatch safety: for a `CALL`, `device:busy` is held by the same ticket for the whole timeout window; `releaseBusyIfOwned` covers the rare `CALL`-to-dead-hub + manual `STOP`+`RELEASE` + re-dispatch interleaving. ✓
- Part C: `queue.dispatch.errorNoActiveTransmitter`/`errorDeviceNotFound`/`errorInfrastructure` exist today at the same `en.json`/`vi.json` positions; `tQueue = useTranslations("queue")` confirmed at `page.tsx:43`, so `dispatch.errorAckTimeout` resolves. The catch-all `else` remains for any other reason. ✓

---

## Execution Handoff

**Plan complete and saved to `docs/planned/Transmitter Dispatch Device Name Plan.md` (backend Parts A & B, web Part C, docs Part D). Two execution options:**

**1. Subagent-Driven (recommended)** — a fresh subagent per task, reviewed between tasks, fast iteration.

**2. Inline Execution** — execute tasks in this session using executing-plans, batched with checkpoints.

**Which approach?**
