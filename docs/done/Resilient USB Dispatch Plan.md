# Resilient USB Dispatch Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let the admin queue keep dispatching to receivers over a USB-connected hub when the MQTT hub is offline (Tier 1) or the backend is entirely unreachable (Tier 2), reconciling state when the cloud returns.

**Architecture:** A derived **dispatch mode** (`ONLINE_MQTT` / `ONLINE_SERIAL_FALLBACK` / `OFFLINE_SERIAL` / `DISABLED`) drives the queue UI. Tier 1 keeps the backend authoritative and only emits RF over serial when the MQTT path reports `DEVICE_DISPATCH_FAILED`. Tier 2 runs the existing queue locally from a `localStorage` snapshot, dispatches via a new firmware `transmit_slot` serial command (hub uses its own roster — RF codes never leave the hub), and replays terminal outcomes to a new idempotent reconcile endpoint on reconnect.

**Tech Stack:** Kotlin/Spring WebFlux + R2DBC + Redis (backend); ESP-IDF v6 / cJSON (firmware); Next.js 16 / React 19 / TypeScript / zustand 5 / next-intl 4 / shadcn-ui / Tailwind 4 (web). Tests: JUnit5 + MockK + AssertJ (backend); vitest node env (web).

**Spec:** `docs/spec/Resilient USB Dispatch.md`

## Global Constraints

- **No git-commit steps** in this plan — the executor decides when to commit (project convention).
- **No app builds** during implementation (`yarn build`, `idf.py build`, `./gradlew build` are forbidden) — a separate audit flow builds. Per-task verification uses **tests + lint only**.
- **Lint before tests** for web tasks: run `yarn lint` (Biome) before `yarn vitest run`.
- **Web tests run in node env** (`vitest.config.ts`: `environment: "node"`, files `src/**/*.test.ts`, coverage scope `src/lib/**` `src/store/**` `src/types/**`). No jsdom/JSX/`renderHook`. **All testable logic lives in `src/lib/` or `src/store/` as pure functions/stores;** hooks/components are thin wrappers verified by manual QA.
- **shadcn/ui only**; reuse existing `Button`/`Badge`/`Tooltip`. Token-based colors only (no hex).
- **Alert boxes** use the canonical Warning pattern verbatim: `rounded-xl border border-warning/40 bg-warning/15 text-warning dark:border-warning/50 dark:bg-warning/20`, icon `size-4 shrink-0`, `gap-2.5`, `px-3.5 py-3`. Use `text-warning` (never `text-warning-foreground`).
- **Bilingual:** every new UI string added to **both** `src/messages/en.json` and `src/messages/vi.json`, mirrored structurally. **Vietnamese copy in this plan is PROPOSED — pending native review before writing `vi.json`** (per `CLAUDE.md`).
- **Named imports only** (no fully-qualified imports). Match existing code style.
- **CHANGELOGS:** log every file change in `docs/CHANGELOGS.md`, including skipped items.
- **Firmware has no host-test harness** and builds are forbidden here, so firmware tasks (Phase 2) cannot run automated red-green tests; they document the on-device verification to be run during the audit flow.
- **API/lib usage verified via Context7** prior to writing this plan: zustand v5 curried `create<T>()`, manual storage persistence (mirroring `src/store/queue.ts`); next-intl `useTranslations` + `t('key', {arg})` interpolation. Web Serial / EventSource usage mirrors existing `src/lib/serial/*` and `src/hooks/use-queue-events.ts`.

---

## File Structure

**Backend (`backend/src/main/kotlin/com/thomas/notiguide`)**
- Modify `core/sse/QueueEventBroadcaster.kt` — add `deviceId`, `dispatchAction` to `QueueSseEvent`.
- Modify `domain/device/service/TransmitterDispatchService.kt` — populate new SSE fields in `emitDispatchFailed`.
- Modify `domain/queue/request/IssueDeviceTicketRequest.kt` — add `allowSerialFallback`.
- Modify `domain/device/service/DeviceDispatchService.kt` — honor `allowSerialFallback`.
- Modify `domain/queue/controller/QueueAdminController.kt` — thread the flag; add `reconcile-offline` route.
- Create `domain/queue/request/ReconcileOfflineRequest.kt`, `domain/queue/response/ReconcileOfflineResponse.kt`.
- Create `domain/queue/service/OfflineReconciliationService.kt`.
- Modify `domain/queue/service/QueueService.kt` — add `reconcileTerminalTransition` (suppressed-dispatch terminal write).
- Tests under `backend/src/test/kotlin/.../` mirroring `DispatchReconciliationServiceTest`.

**Firmware (`transmitter/main`)**
- Modify `dispatch/dispatch.c` + `dispatch/dispatch.h` — extract `dispatch_slot_execute`.
- Modify `serial/serial_protocol.c` — add `handle_transmit_slot` + dispatcher branch.

**Web (`web/src`)**
- Modify `types/queue.ts`, `lib/constants.ts`, `lib/serial/types.ts`, `features/queue/api.ts`, `features/device/api.ts`.
- Create `lib/dispatch/mode.ts`, `lib/dispatch/serial-command.ts`, `lib/dispatch/reachability.ts`, `lib/dispatch/dedupe.ts` (+ `.test.ts` each).
- Create `store/offline-dispatch.ts` (+ `.test.ts`).
- Modify `hooks/use-queue-events.ts`; create `hooks/use-backend-reachability.ts`, `hooks/use-dispatch-mode.ts`, `hooks/use-serial-dispatch.ts`.
- Create `features/queue/dispatch-mode-banner.tsx`; modify `features/queue/device-dispatch-panel.tsx`, `features/queue/serving-display.tsx`, `app/[locale]/dashboard/queue/page.tsx`.
- Modify `messages/en.json`, `messages/vi.json`.

---

# Phase 1 — Backend

### Task 1: Enrich `DEVICE_DISPATCH_FAILED` SSE with `deviceId` + `dispatchAction`

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/core/sse/QueueEventBroadcaster.kt:17-25`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/service/TransmitterDispatchService.kt:212-240`
- Test: `backend/src/test/kotlin/com/thomas/notiguide/domain/device/service/TransmitterDispatchFailedEventTest.kt` (new)

**Interfaces:**
- Produces: `QueueSseEvent(..., deviceId: UUID? = null, dispatchAction: String? = null)`; `emitDispatchFailed` sets `deviceId = event.deviceId` and `dispatchAction = "call"|"stop"` from `event.type`.

- [ ] **Step 1: Add fields to `QueueSseEvent`**

In `QueueEventBroadcaster.kt`, extend the data class (keep existing fields/order; append the two nullable fields before `timestamp`):

```kotlin
data class QueueSseEvent(
    val type: String,
    val storeId: UUID,
    val ticketId: UUID,
    val ticketNumber: String? = null,
    val counterId: String? = null,
    val reason: String? = null,
    val deviceId: UUID? = null,
    val dispatchAction: String? = null,
    val timestamp: Long = System.currentTimeMillis()
)
```

- [ ] **Step 2: Write the failing test**

Create `TransmitterDispatchFailedEventTest.kt`. Because `emitDispatchFailed` is private, assert through the public failure path is heavy; instead test the mapping helper. Add a private→internal seam: extract the action mapping into a top-level `fun dispatchActionOf(type: DeviceDispatchEventType): String` in `TransmitterDispatchService.kt` and test it.

```kotlin
package com.thomas.notiguide.domain.device.service

import com.thomas.notiguide.domain.device.types.DeviceDispatchEventType
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test

class TransmitterDispatchFailedEventTest {
    @Test
    fun `dispatchActionOf maps call and stop`() {
        assertThat(dispatchActionOf(DeviceDispatchEventType.DEVICE_CALL_REQUESTED)).isEqualTo("call")
        assertThat(dispatchActionOf(DeviceDispatchEventType.DEVICE_STOP_REQUESTED)).isEqualTo("stop")
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.device.service.TransmitterDispatchFailedEventTest"`
Expected: FAIL — `dispatchActionOf` unresolved.

- [ ] **Step 4: Implement the mapping + wire into `emitDispatchFailed`**

In `TransmitterDispatchService.kt`, add at file top-level (outside the class):

```kotlin
internal fun dispatchActionOf(type: DeviceDispatchEventType): String =
    when (type) {
        DeviceDispatchEventType.DEVICE_CALL_REQUESTED -> "call"
        DeviceDispatchEventType.DEVICE_STOP_REQUESTED -> "stop"
    }
```

Then in `emitDispatchFailed`, populate the new fields on the broadcast `QueueSseEvent`:

```kotlin
queueEventBroadcaster.broadcast(
    QueueSseEvent(
        type = QueueEventType.DEVICE_DISPATCH_FAILED.name,
        storeId = event.storeId,
        ticketId = event.ticketId,
        ticketNumber = event.ticketNumber,
        reason = reason,
        deviceId = event.deviceId,
        dispatchAction = dispatchActionOf(event.type)
    )
)
```

- [ ] **Step 5: Run test to verify it passes**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.device.service.TransmitterDispatchFailedEventTest"`
Expected: PASS.

- [ ] **Step 6: Log to CHANGELOGS** — add an entry under a new "Resilient USB Dispatch" heading in `docs/CHANGELOGS.md` describing the `QueueSseEvent` field additions.

---

### Task 2: `allowSerialFallback` on device-ticket issuance

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/queue/request/IssueDeviceTicketRequest.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/service/DeviceDispatchService.kt:36-43`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/queue/controller/QueueAdminController.kt:76-89`
- Test: `backend/src/test/kotlin/com/thomas/notiguide/domain/device/service/DeviceDispatchIssueFallbackTest.kt` (new)

**Interfaces:**
- Consumes: `TransmitterElectionService` (via `ObjectProvider`), `QueueService.issueTicket`.
- Produces: `DeviceDispatchService.issueDeviceTicket(storeId, deviceId, serviceTypeId, allowSerialFallback: Boolean = false)`; `IssueDeviceTicketRequest(deviceId, serviceTypeId, allowSerialFallback: Boolean = false)`.

- [ ] **Step 1: Add the request field**

`IssueDeviceTicketRequest.kt`:

```kotlin
data class IssueDeviceTicketRequest(
    @field:NotNull
    val deviceId: UUID,
    val serviceTypeId: UUID? = null,
    val allowSerialFallback: Boolean = false
)
```

- [ ] **Step 2: Write the failing test**

`DeviceDispatchIssueFallbackTest.kt` — verify that with `allowSerialFallback=true` and **no** elected transmitter, issuance proceeds (does not throw); with `false` it throws `DeviceConflictEnvelopeException("no_active_transmitter")`. Mirror MockK style from `DispatchReconciliationServiceTest`.

```kotlin
package com.thomas.notiguide.domain.device.service

import com.thomas.notiguide.domain.device.controller.DeviceConflictEnvelopeException
import com.thomas.notiguide.domain.queue.service.QueueService
import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import io.mockk.coEvery
import io.mockk.every
import io.mockk.mockk
import kotlinx.coroutines.test.runTest
import org.assertj.core.api.Assertions.assertThatThrownBy
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.ObjectProvider
import org.springframework.data.redis.core.ReactiveRedisTemplate
import java.util.UUID

class DeviceDispatchIssueFallbackTest {
    private val deviceQueryService = mockk<DeviceQueryService>(relaxed = true)
    private val queueService = mockk<QueueService>(relaxed = true)
    private val redis = mockk<ReactiveRedisTemplate<String, String>>(relaxed = true)
    private val electionProvider = mockk<ObjectProvider<TransmitterElectionService>>()
    private val propsProvider = mockk<ObjectProvider<com.thomas.notiguide.core.device.DeviceTransmitterProperties>>(relaxed = true)
    private val service = DeviceDispatchService(
        deviceQueryService, queueService, redis, jacksonObjectMapper().findAndRegisterModules(),
        electionProvider, propsProvider
    )
    private val storeId = UUID.randomUUID()
    private val deviceId = UUID.randomUUID()

    @Test
    fun `issuance without fallback throws when no active transmitter`() = runTest {
        every { electionProvider.ifAvailable } returns mockk {
            coEvery { electActive(storeId) } returns null
        }
        assertThatThrownBy {
            kotlinx.coroutines.runBlocking {
                service.issueDeviceTicket(storeId, deviceId, null, allowSerialFallback = false)
            }
        }.isInstanceOf(DeviceConflictEnvelopeException::class.java)
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cd backend && ./gradlew test --tests "*DeviceDispatchIssueFallbackTest"`
Expected: FAIL — `issueDeviceTicket` has no `allowSerialFallback` parameter.

- [ ] **Step 4: Implement**

In `DeviceDispatchService.kt`, change the signature and the guard (`:36-42`):

```kotlin
suspend fun issueDeviceTicket(
    storeId: UUID,
    deviceId: UUID,
    serviceTypeId: UUID?,
    allowSerialFallback: Boolean = false
): TicketDto {
    val elected = transmitterElectionServiceProvider.ifAvailable?.electActive(storeId)
    if (elected == null && !allowSerialFallback) {
        throw DeviceConflictEnvelopeException("no_active_transmitter")
    }
    val device = loadDispatchableDevice(storeId, deviceId)
    // ... rest unchanged ...
```

In `QueueAdminController.kt` (`:83-87`), pass the flag through:

```kotlin
val ticket = deviceDispatchService.issueDeviceTicket(
    storeId = storeId,
    deviceId = request.deviceId,
    serviceTypeId = request.serviceTypeId,
    allowSerialFallback = request.allowSerialFallback
)
```

- [ ] **Step 5: Run test to verify it passes**

Run: `cd backend && ./gradlew test --tests "*DeviceDispatchIssueFallbackTest"`
Expected: PASS.

- [ ] **Step 6: Log to CHANGELOGS.**

---

### Task 3: `OfflineReconciliationService` + reconcile endpoint

**Files:**
- Create: `backend/src/main/kotlin/com/thomas/notiguide/domain/queue/request/ReconcileOfflineRequest.kt`
- Create: `backend/src/main/kotlin/com/thomas/notiguide/domain/queue/response/ReconcileOfflineResponse.kt`
- Create: `backend/src/main/kotlin/com/thomas/notiguide/domain/queue/service/OfflineReconciliationService.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/queue/service/QueueService.kt` (add `reconcileTerminalTransition`)
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/queue/controller/QueueAdminController.kt`
- Test: `backend/src/test/kotlin/com/thomas/notiguide/domain/queue/service/OfflineReconciliationServiceTest.kt` (new)

**Interfaces:**
- Produces:
  - `ReconcileOfflineRequest(transitions: List<OfflineTransition>)`, `OfflineTransition(ticketId: UUID, action: OfflineAction, at: String)`, `enum OfflineAction { SERVE, CANCEL, NO_SHOW }`.
  - `ReconcileOfflineResponse(results: List<ReconcileItemResult>)`, `ReconcileItemResult(ticketId: UUID, result: String)` where result ∈ `applied|superseded|gone`.
  - `OfflineReconciliationService.reconcile(storeId, request, principal): ReconcileOfflineResponse`.
  - `QueueService.reconcileTerminalTransition(storeId, ticketId, action): String` — applies terminal state **without** emitting `DEVICE_*_REQUESTED`, clears `device:busy`, broadcasts the normal `TICKET_SERVED`/`TICKET_CANCELLED` SSE; returns `applied|superseded|gone`.

- [ ] **Step 1: Create request/response DTOs**

`ReconcileOfflineRequest.kt`:

```kotlin
package com.thomas.notiguide.domain.queue.request

import jakarta.validation.constraints.NotNull
import jakarta.validation.constraints.Size
import java.util.UUID

data class ReconcileOfflineRequest(
    @field:NotNull
    @field:Size(max = 500)
    val transitions: List<OfflineTransition> = emptyList()
)

data class OfflineTransition(
    @field:NotNull val ticketId: UUID,
    @field:NotNull val action: OfflineAction,
    @field:Size(max = 40) val at: String? = null
)

enum class OfflineAction { SERVE, CANCEL, NO_SHOW }
```

`ReconcileOfflineResponse.kt`:

```kotlin
package com.thomas.notiguide.domain.queue.response

import java.util.UUID

data class ReconcileOfflineResponse(val results: List<ReconcileItemResult>)
data class ReconcileItemResult(val ticketId: UUID, val result: String)
```

- [ ] **Step 2: Write the failing test for `OfflineReconciliationService`**

`OfflineReconciliationServiceTest.kt` — MockK style. Assert: (a) each transition calls `queueService.reconcileTerminalTransition` and the per-item result is propagated; (b) a `QueueService` returning `gone` yields a `gone` item; (c) store access is required.

```kotlin
package com.thomas.notiguide.domain.queue.service

import com.thomas.notiguide.domain.queue.request.OfflineAction
import com.thomas.notiguide.domain.queue.request.OfflineTransition
import com.thomas.notiguide.domain.queue.request.ReconcileOfflineRequest
import com.thomas.notiguide.shared.principal.AdminPrincipal
import com.thomas.notiguide.shared.principal.StoreAccessService
import io.mockk.coEvery
import io.mockk.coVerify
import io.mockk.mockk
import kotlinx.coroutines.test.runTest
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import java.util.UUID

class OfflineReconciliationServiceTest {
    private val queueService = mockk<QueueService>()
    private val storeAccess = mockk<StoreAccessService>(relaxed = true)
    private val service = OfflineReconciliationService(queueService, storeAccess)
    private val storeId = UUID.randomUUID()
    private val principal = mockk<AdminPrincipal>(relaxed = true)
    private val t1 = UUID.randomUUID()
    private val t2 = UUID.randomUUID()

    @Test
    fun `reconcile applies each transition and propagates per-item results`() = runTest {
        coEvery { queueService.reconcileTerminalTransition(storeId, t1, OfflineAction.SERVE) } returns "applied"
        coEvery { queueService.reconcileTerminalTransition(storeId, t2, OfflineAction.CANCEL) } returns "gone"

        val res = service.reconcile(
            storeId,
            ReconcileOfflineRequest(
                listOf(
                    OfflineTransition(t1, OfflineAction.SERVE, null),
                    OfflineTransition(t2, OfflineAction.CANCEL, null)
                )
            ),
            principal
        )

        assertThat(res.results.map { it.result }).containsExactly("applied", "gone")
        coVerify(exactly = 1) { storeAccess.requireStoreAccess(principal, storeId) }
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cd backend && ./gradlew test --tests "*OfflineReconciliationServiceTest"`
Expected: FAIL — `OfflineReconciliationService` does not exist.

- [ ] **Step 4: Implement `OfflineReconciliationService`**

```kotlin
package com.thomas.notiguide.domain.queue.service

import com.thomas.notiguide.domain.queue.request.ReconcileOfflineRequest
import com.thomas.notiguide.domain.queue.response.ReconcileItemResult
import com.thomas.notiguide.domain.queue.response.ReconcileOfflineResponse
import com.thomas.notiguide.shared.principal.AdminPrincipal
import com.thomas.notiguide.shared.principal.StoreAccessService
import org.springframework.stereotype.Service
import java.util.UUID

@Service
class OfflineReconciliationService(
    private val queueService: QueueService,
    private val storeAccess: StoreAccessService
) {
    suspend fun reconcile(
        storeId: UUID,
        request: ReconcileOfflineRequest,
        principal: AdminPrincipal
    ): ReconcileOfflineResponse {
        storeAccess.requireStoreAccess(principal, storeId)
        val results = request.transitions.map { transition ->
            val result = runCatching {
                queueService.reconcileTerminalTransition(storeId, transition.ticketId, transition.action)
            }.getOrDefault("gone")
            ReconcileItemResult(transition.ticketId, result)
        }
        return ReconcileOfflineResponse(results)
    }
}
```

- [ ] **Step 5: Implement `QueueService.reconcileTerminalTransition`**

Add to `QueueService.kt`. It reuses the existing terminal write but **suppresses** device dispatch. Read the current terminal ticket state; if absent → `gone`; if already terminal → `superseded`; else write the terminal status to the ticket hash, remove from serving set, clear `device:busy`, and broadcast the queue SSE — **without** calling `emitDeviceDispatchEvent`.

```kotlin
suspend fun reconcileTerminalTransition(
    storeId: UUID,
    ticketId: UUID,
    action: com.thomas.notiguide.domain.queue.request.OfflineAction
): String {
    val ticketKey = RedisKeyManager.ticket(storeId, ticketId)
    val ticketData = redis.opsForHash<String, String>().entries(ticketKey)
        .collectMap({ it.key }, { it.value }).awaitSingleOrNull().orEmpty()
    if (ticketData.isEmpty()) return "gone"

    val current = ticketData["status"]?.let { runCatching { TicketStatus.valueOf(it) }.getOrNull() }
    if (current == TicketStatus.SERVED || current == TicketStatus.CANCELLED) return "superseded"

    val terminalStatus = when (action) {
        com.thomas.notiguide.domain.queue.request.OfflineAction.SERVE -> TicketStatus.SERVED
        com.thomas.notiguide.domain.queue.request.OfflineAction.CANCEL,
        com.thomas.notiguide.domain.queue.request.OfflineAction.NO_SHOW -> TicketStatus.CANCELLED
    }

    // Terminal write: status + serving-set removal + busy cleanup, NO device dispatch emission.
    redis.opsForHash<String, String>().put(ticketKey, "status", terminalStatus.name).awaitSingleOrNull()
    redis.opsForZSet().remove(RedisKeyManager.serving(storeId), ticketId.toString()).awaitSingleOrNull()
    parseUuid(ticketData["device_id"])?.let { deviceId ->
        redis.delete(RedisKeyManager.deviceBusy(deviceId)).awaitSingleOrNull()
    }
    redis.expire(ticketKey, RedisTTLPolicy.TICKET_TERMINAL).awaitSingleOrNull()

    queueEventBroadcaster.broadcast(
        QueueSseEvent(
            type = (if (terminalStatus == TicketStatus.SERVED) QueueEventType.TICKET_SERVED
                    else QueueEventType.TICKET_CANCELLED).name,
            storeId = storeId,
            ticketId = ticketId,
            ticketNumber = ticketData["number"]
        )
    )
    return "applied"
}
```

> **Implementer note:** confirm the exact `RedisKeyManager` accessor names against `QueueService.kt`'s existing terminal handlers (`serving(...)`, `ticket(...)`, `deviceBusy(...)`), the `QueueEventType` import already used in the file, and that `parseUuid`/`RedisTTLPolicy.TICKET_TERMINAL` are the in-file helpers used by `handleTerminalDeviceDispatch`. Run `gitnexus_impact({target: "serveTicket"})` to confirm no terminal helper is bypassed.

- [ ] **Step 6: Add the controller endpoint**

In `QueueAdminController.kt`, inject `OfflineReconciliationService` and add:

```kotlin
@PostMapping("/reconcile-offline")
suspend fun reconcileOffline(
    @PathVariable storeId: UUID,
    @Valid @RequestBody request: ReconcileOfflineRequest,
    @AuthenticationPrincipal principal: AdminPrincipal
): ResponseEntity<ReconcileOfflineResponse> =
    ResponseEntity.ok(offlineReconciliationService.reconcile(storeId, request, principal))
```

Add imports for `ReconcileOfflineRequest`, `ReconcileOfflineResponse`, `OfflineReconciliationService`, and add the constructor parameter `private val offlineReconciliationService: OfflineReconciliationService`.

- [ ] **Step 7: Run tests**

Run: `cd backend && ./gradlew test --tests "*OfflineReconciliationServiceTest"`
Expected: PASS.

- [ ] **Step 8: Log to CHANGELOGS.**

---

# Phase 2 — Firmware

> **No automated tests / no build here** (Global Constraints). Each task lists the on-device verification to run in the audit flow. Run `gitnexus_impact({target: "dispatch_handle_slot"})` before Task 4 and report the blast radius.

### Task 4: Extract `dispatch_slot_execute` shared core

**Files:**
- Modify: `transmitter/main/dispatch/dispatch.c:383-478`
- Modify: `transmitter/main/dispatch/dispatch.h`

**Interfaces:**
- Produces: `typedef struct { char status[16]; char reason[32]; int64_t applied_at_ms; } slot_exec_result_t;` and `void dispatch_slot_execute(uint8_t slot, const char *action, slot_exec_result_t *out);` (declared in `dispatch.h`). `status` is `"applied"`/`"rejected"`; `reason` empty on success.

- [ ] **Step 1: Declare the result type + function in `dispatch.h`**

```c
typedef struct {
    char status[16];      // "applied" | "rejected"
    char reason[32];      // empty on success; else slot_not_found|device_suspended|...
    int64_t applied_at_ms;
} slot_exec_result_t;

// Executes a slot dispatch from the active roster. No signature check, no ack channel,
// no dedup ring — callers own those. Gates on OP_STATE_ACTIVE and roster presence.
void dispatch_slot_execute(uint8_t slot, const char *action, slot_exec_result_t *out);
```

- [ ] **Step 2: Implement `dispatch_slot_execute` in `dispatch.c`** (lift the op-state gate, roster lookup, and RF build/transmit from the current `dispatch_handle_slot` body at `:422-477`):

```c
void dispatch_slot_execute(uint8_t slot, const char *action, slot_exec_result_t *out)
{
    memset(out, 0, sizeof(*out));

    const device_config_t *cfg = device_config_get();
    if (cfg->op_state != OP_STATE_ACTIVE) {
        strlcpy(out->status, "rejected", sizeof(out->status));
        strlcpy(out->reason,
                cfg->op_state == OP_STATE_SUSPENDED ? "device_suspended" : "device_decommissioned",
                sizeof(out->reason));
        return;
    }

    roster_t *roster = roster_get_active();
    const roster_entry_t *entry = roster_find_slot(roster, slot);
    if (entry == NULL) {
        strlcpy(out->status, "rejected", sizeof(out->status));
        strlcpy(out->reason, "slot_not_found", sizeof(out->reason));
        return;
    }

    uint8_t tx_code[ROSTER_MAX_CODE_LEN];
    memcpy(tx_code, entry->rf_code, entry->rf_code_len);
    const uint8_t radio_action = strcmp(action, "stop") == 0 ? RADIO_ACTION_STOP : RADIO_ACTION_START;
    if (entry->band == RADIO_BAND_433M) {
        tx_code[0] &= 0x7F;
        if (radio_action == RADIO_ACTION_STOP) {
            tx_code[0] |= 0x80;
        }
    }

    const esp_err_t tx_result = radio_tx_send((radio_band_t)entry->band, tx_code,
                                              entry->rf_code_len, entry->rf_bits, true, radio_action);
    if (tx_result == ESP_OK) {
        strlcpy(out->status, "applied", sizeof(out->status));
        out->applied_at_ms = time_utils_now_epoch_ms();
    } else {
        strlcpy(out->status, "rejected", sizeof(out->status));
        strlcpy(out->reason, tx_result == ESP_ERR_NOT_SUPPORTED ? "no_2_4g_radio" : "tx_failed",
                sizeof(out->reason));
    }
}
```

- [ ] **Step 3: Refactor `dispatch_handle_slot` to delegate** — keep `parse_slot_from_root`, `verify_slot_signature`, action validation, and the dedup ring, but replace the inline op-state/roster/transmit block (`:422-477`) with a call to `dispatch_slot_execute`, then map the result to `dispatch_ring_remember` + `publish_transmit_ack` + `post_dispatch_event` exactly as before (status/reason now come from `out`). Preserve the `slot_to_transmit_cmd(&slot_cmd, entry, &tx_cmd)` call for building the ack envelope — fetch `entry` for the ack via `roster_find_slot` once, or pass status/reason through. Behavior must be identical for the MQTT path.

- [ ] **Step 4: Verification (audit flow — document, do not run a build here)**

On-device: publish an MQTT slot dispatch and confirm `transmit ack` and RF emission are unchanged for `call` and `stop` on both bands; confirm a suspended hub still rejects with `device_suspended`, and an empty slot rejects with `slot_not_found`.

- [ ] **Step 5: Log to CHANGELOGS** (note the extraction + that MQTT slot behavior is unchanged).

---

### Task 5: Add `transmit_slot` serial command

**Files:**
- Modify: `transmitter/main/serial/serial_protocol.c` (new `handle_transmit_slot` + dispatcher branch at `:756`)

**Interfaces:**
- Consumes: `dispatch_slot_execute` (Task 4).
- Produces: serial command `transmit_slot` with payload `{ "slot": <1..MAX>, "action": "call"|"stop" }`, response `{ "status": "applied"|"rejected", "reason"?: string }`.

- [ ] **Step 1: Implement the handler** (place near `handle_transmit`, reuse `json_get_string`/`serial_send_error`/`serial_send_response` and the `OP_STATE_ACTIVE` reasoning that `dispatch_slot_execute` already enforces):

```c
static void handle_transmit_slot(const char *id, const cJSON *payload)
{
    if (payload == NULL) {
        serial_send_error(id, "missing_payload");
        return;
    }
    const cJSON *slot_item = cJSON_GetObjectItem(payload, "slot");
    const char *action = json_get_string(payload, "action");
    if (!cJSON_IsNumber(slot_item) || action == NULL) {
        serial_send_error(id, "missing_fields");
        return;
    }
    const int slot_val = (int)slot_item->valuedouble;
    if (slot_val < 1 || slot_val > CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS) {
        serial_send_error(id, "invalid_slot");
        return;
    }
    if (strcmp(action, "call") != 0 && strcmp(action, "stop") != 0) {
        serial_send_error(id, "invalid_action");
        return;
    }

    slot_exec_result_t result;
    dispatch_slot_execute((uint8_t)slot_val, action, &result);

    cJSON *resp = cJSON_CreateObject();
    cJSON_AddStringToObject(resp, "status", result.status);
    if (strcmp(result.status, "applied") == 0) {
        cJSON_AddNumberToObject(resp, "applied_at_ms", (double)result.applied_at_ms);
    } else {
        cJSON_AddStringToObject(resp, "reason", result.reason);
    }
    serial_send_response(id, true, resp, NULL);
}
```

Add `#include "dispatch/dispatch.h"` if not already present.

- [ ] **Step 2: Route the command** — in the dispatcher chain (`serial_protocol.c:756`, beside `transmit`):

```c
} else if (strcmp(type, "transmit_slot") == 0) {
    serial_mark_session_active();
    handle_transmit_slot(id, payload);
```

- [ ] **Step 3: Verification (audit flow — document)** — over Web Serial send `{type:"transmit_slot", payload:{slot:1, action:"call"}}`; expect `{status:"applied"}` + RF; `slot:99` → `invalid_slot`; an unpaired slot → `slot_not_found`; suspended hub → `device_suspended`.

- [ ] **Step 4: Log to CHANGELOGS.**

---

# Phase 3 — Frontend pure logic (node-tested)

### Task 6: Types, constants, API functions

**Files:**
- Modify: `web/src/types/queue.ts`, `web/src/lib/serial/types.ts`, `web/src/lib/constants.ts`, `web/src/features/queue/api.ts`

**Interfaces:**
- Produces: `QueueSseEvent.deviceId?/dispatchAction?`; `IssueDeviceTicketRequest.allowSerialFallback?`; `TransmitSlotPayload`; `SerialCommandMap.transmit_slot`; `API_ROUTES.QUEUE.RECONCILE_OFFLINE`; `reconcileOffline(storeId, body)`; `ReconcileOfflineRequest`/`ReconcileOfflineResponse` TS types.

- [ ] **Step 1: Extend `types/queue.ts`**

```ts
export interface QueueSseEvent {
  type: QueueEventType;
  storeId: string;
  ticketId: string;
  ticketNumber: string | null;
  counterId: string | null;
  reason: string | null;
  deviceId?: string | null;
  dispatchAction?: "call" | "stop" | null;
  timestamp: number;
}

export interface IssueDeviceTicketRequest {
  deviceId: string;
  serviceTypeId?: string | null;
  allowSerialFallback?: boolean;
}

export type OfflineAction = "SERVE" | "CANCEL" | "NO_SHOW";
export interface OfflineTransition {
  ticketId: string;
  action: OfflineAction;
  at?: string;
}
export interface ReconcileOfflineRequest {
  transitions: OfflineTransition[];
}
export interface ReconcileItemResult {
  ticketId: string;
  result: "applied" | "superseded" | "gone";
}
export interface ReconcileOfflineResponse {
  results: ReconcileItemResult[];
}
```

- [ ] **Step 2: Extend `lib/serial/types.ts`** — add the payload type and map entry:

```ts
export interface TransmitSlotPayload {
  slot: number;
  action: "call" | "stop";
}
```

Add to `SerialCommandMap` (after `transmit`):

```ts
  transmit_slot: { payload: TransmitSlotPayload; response: TransmitResult };
```

- [ ] **Step 3: Add the route + API fn** — in `lib/constants.ts` under `QUEUE`:

```ts
    RECONCILE_OFFLINE: (storeId: string) =>
      `/api/queue/admin/${storeId}/reconcile-offline`,
```

In `features/queue/api.ts`:

```ts
export function reconcileOffline(storeId: string, body: ReconcileOfflineRequest) {
  return post<ReconcileOfflineResponse>(
    API_ROUTES.QUEUE.RECONCILE_OFFLINE(storeId),
    body,
  );
}
```

Add `ReconcileOfflineRequest`, `ReconcileOfflineResponse` to the type import block.

- [ ] **Step 4: Verify lint** — Run: `cd web && yarn lint`
Expected: no errors. (Types-only change; covered by downstream tests.)

- [ ] **Step 5: Log to CHANGELOGS.**

---

### Task 7: `deriveDispatchMode` pure function

**Files:**
- Create: `web/src/lib/dispatch/mode.ts`
- Test: `web/src/lib/dispatch/mode.test.ts`

**Interfaces:**
- Produces: `type DispatchMode = "ONLINE_MQTT" | "ONLINE_SERIAL_FALLBACK" | "OFFLINE_SERIAL" | "DISABLED";` and `deriveDispatchMode(signals: DispatchSignals): DispatchMode` where `DispatchSignals = { backendReachable: boolean; dispatchReady: boolean; hubConnectedForStore: boolean }`.

- [ ] **Step 1: Write the failing test** (`mode.test.ts`) — full truth table from spec §3.1:

```ts
import { describe, expect, it } from "vitest";
import { deriveDispatchMode } from "@/lib/dispatch/mode";

describe("deriveDispatchMode", () => {
  it("offline + hub → OFFLINE_SERIAL", () => {
    expect(deriveDispatchMode({ backendReachable: false, dispatchReady: false, hubConnectedForStore: true })).toBe("OFFLINE_SERIAL");
  });
  it("offline + no hub → DISABLED", () => {
    expect(deriveDispatchMode({ backendReachable: false, dispatchReady: true, hubConnectedForStore: false })).toBe("DISABLED");
  });
  it("online + dispatchReady → ONLINE_MQTT (hub irrelevant)", () => {
    expect(deriveDispatchMode({ backendReachable: true, dispatchReady: true, hubConnectedForStore: true })).toBe("ONLINE_MQTT");
    expect(deriveDispatchMode({ backendReachable: true, dispatchReady: true, hubConnectedForStore: false })).toBe("ONLINE_MQTT");
  });
  it("online + not ready + hub → ONLINE_SERIAL_FALLBACK", () => {
    expect(deriveDispatchMode({ backendReachable: true, dispatchReady: false, hubConnectedForStore: true })).toBe("ONLINE_SERIAL_FALLBACK");
  });
  it("online + not ready + no hub → DISABLED", () => {
    expect(deriveDispatchMode({ backendReachable: true, dispatchReady: false, hubConnectedForStore: false })).toBe("DISABLED");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd web && yarn vitest run src/lib/dispatch/mode.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement `mode.ts`**

```ts
export type DispatchMode =
  | "ONLINE_MQTT"
  | "ONLINE_SERIAL_FALLBACK"
  | "OFFLINE_SERIAL"
  | "DISABLED";

export interface DispatchSignals {
  backendReachable: boolean;
  dispatchReady: boolean;
  hubConnectedForStore: boolean;
}

export function deriveDispatchMode(signals: DispatchSignals): DispatchMode {
  const { backendReachable, dispatchReady, hubConnectedForStore } = signals;
  if (!backendReachable) {
    return hubConnectedForStore ? "OFFLINE_SERIAL" : "DISABLED";
  }
  if (dispatchReady) return "ONLINE_MQTT";
  return hubConnectedForStore ? "ONLINE_SERIAL_FALLBACK" : "DISABLED";
}
```

- [ ] **Step 4: Run lint + test**

Run: `cd web && yarn lint && yarn vitest run src/lib/dispatch/mode.test.ts`
Expected: PASS.

- [ ] **Step 5: Log to CHANGELOGS.**

---

### Task 8: `buildSerialDispatch` pure function (slot vs full-payload)

**Files:**
- Create: `web/src/lib/dispatch/serial-command.ts`
- Test: `web/src/lib/dispatch/serial-command.test.ts`

**Interfaces:**
- Produces: `buildSerialDispatch(device: { hubSlot: number | null }, action: "call" | "stop"): SerialDispatchPlan` where
  `SerialDispatchPlan = { kind: "slot"; slot: number; action } | { kind: "payload"; action }`.
  `slot` plan → caller sends `transmit_slot`; `payload` plan → caller fetches `getUsbDispatchPayload` then sends `transmit`.

- [ ] **Step 1: Write the failing test**

```ts
import { describe, expect, it } from "vitest";
import { buildSerialDispatch } from "@/lib/dispatch/serial-command";

describe("buildSerialDispatch", () => {
  it("hub-paired receiver → slot plan", () => {
    expect(buildSerialDispatch({ hubSlot: 3 }, "call")).toEqual({ kind: "slot", slot: 3, action: "call" });
  });
  it("standalone receiver → payload plan", () => {
    expect(buildSerialDispatch({ hubSlot: null }, "stop")).toEqual({ kind: "payload", action: "stop" });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd web && yarn vitest run src/lib/dispatch/serial-command.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement**

```ts
export type SerialDispatchAction = "call" | "stop";

export type SerialDispatchPlan =
  | { kind: "slot"; slot: number; action: SerialDispatchAction }
  | { kind: "payload"; action: SerialDispatchAction };

export function buildSerialDispatch(
  device: { hubSlot: number | null },
  action: SerialDispatchAction,
): SerialDispatchPlan {
  if (device.hubSlot != null) {
    return { kind: "slot", slot: device.hubSlot, action };
  }
  return { kind: "payload", action };
}
```

- [ ] **Step 4: Run lint + test**

Run: `cd web && yarn lint && yarn vitest run src/lib/dispatch/serial-command.test.ts`
Expected: PASS.

- [ ] **Step 5: Log to CHANGELOGS.**

---

### Task 9: `reachabilityReducer` debounce logic

**Files:**
- Create: `web/src/lib/dispatch/reachability.ts`
- Test: `web/src/lib/dispatch/reachability.test.ts`

**Interfaces:**
- Produces: `INITIAL_REACHABILITY: ReachabilityState`; `reachabilityReducer(state, event): ReachabilityState`; `isReachable(state): boolean`. Events: `{ type: "sse_open" } | { type: "sse_error" } | { type: "api_error" }`. State declares offline only after `FAILURE_THRESHOLD` (2) consecutive failures; any `sse_open` resets to reachable immediately.

- [ ] **Step 1: Write the failing test**

```ts
import { describe, expect, it } from "vitest";
import { INITIAL_REACHABILITY, isReachable, reachabilityReducer } from "@/lib/dispatch/reachability";

describe("reachabilityReducer", () => {
  it("starts reachable", () => {
    expect(isReachable(INITIAL_REACHABILITY)).toBe(true);
  });
  it("stays reachable after a single failure (debounce)", () => {
    const s = reachabilityReducer(INITIAL_REACHABILITY, { type: "sse_error" });
    expect(isReachable(s)).toBe(true);
  });
  it("goes unreachable after 2 consecutive failures", () => {
    let s = reachabilityReducer(INITIAL_REACHABILITY, { type: "sse_error" });
    s = reachabilityReducer(s, { type: "api_error" });
    expect(isReachable(s)).toBe(false);
  });
  it("recovers immediately on sse_open", () => {
    let s = reachabilityReducer(INITIAL_REACHABILITY, { type: "sse_error" });
    s = reachabilityReducer(s, { type: "sse_error" });
    s = reachabilityReducer(s, { type: "sse_open" });
    expect(isReachable(s)).toBe(true);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd web && yarn vitest run src/lib/dispatch/reachability.test.ts`
Expected: FAIL.

- [ ] **Step 3: Implement**

```ts
export const FAILURE_THRESHOLD = 2;

export interface ReachabilityState {
  consecutiveFailures: number;
  reachable: boolean;
}

export const INITIAL_REACHABILITY: ReachabilityState = {
  consecutiveFailures: 0,
  reachable: true,
};

export type ReachabilityEvent =
  | { type: "sse_open" }
  | { type: "sse_error" }
  | { type: "api_error" };

export function reachabilityReducer(
  state: ReachabilityState,
  event: ReachabilityEvent,
): ReachabilityState {
  if (event.type === "sse_open") {
    return { consecutiveFailures: 0, reachable: true };
  }
  const consecutiveFailures = state.consecutiveFailures + 1;
  return {
    consecutiveFailures,
    reachable: consecutiveFailures < FAILURE_THRESHOLD,
  };
}

export function isReachable(state: ReachabilityState): boolean {
  return state.reachable;
}
```

- [ ] **Step 4: Run lint + test**

Run: `cd web && yarn lint && yarn vitest run src/lib/dispatch/reachability.test.ts`
Expected: PASS.

- [ ] **Step 5: Log to CHANGELOGS.**

---

### Task 10: `createDispatchDedupe` (Tier-1 SSE de-dup)

**Files:**
- Create: `web/src/lib/dispatch/dedupe.ts`
- Test: `web/src/lib/dispatch/dedupe.test.ts`

**Interfaces:**
- Produces: `createDispatchDedupe(ttlMs?: number)` returning `{ seen(key: string, now?: number): boolean }` — returns `true` the first time a key is seen within `ttlMs`, `false` on repeats.

- [ ] **Step 1: Write the failing test**

```ts
import { describe, expect, it } from "vitest";
import { createDispatchDedupe } from "@/lib/dispatch/dedupe";

describe("createDispatchDedupe", () => {
  it("admits a key once, rejects repeats within ttl", () => {
    const d = createDispatchDedupe(1000);
    expect(d.seen("t1:call:5", 0)).toBe(true);
    expect(d.seen("t1:call:5", 500)).toBe(false);
  });
  it("re-admits after ttl elapses", () => {
    const d = createDispatchDedupe(1000);
    expect(d.seen("k", 0)).toBe(true);
    expect(d.seen("k", 1500)).toBe(true);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd web && yarn vitest run src/lib/dispatch/dedupe.test.ts`
Expected: FAIL.

- [ ] **Step 3: Implement**

```ts
export function createDispatchDedupe(ttlMs = 5000) {
  const seenAt = new Map<string, number>();
  return {
    seen(key: string, now: number = Date.now()): boolean {
      const last = seenAt.get(key);
      if (last !== undefined && now - last < ttlMs) return false;
      seenAt.set(key, now);
      return true;
    },
  };
}
```

- [ ] **Step 4: Run lint + test**

Run: `cd web && yarn lint && yarn vitest run src/lib/dispatch/dedupe.test.ts`
Expected: PASS.

- [ ] **Step 5: Log to CHANGELOGS.**

---

### Task 11: Offline dispatch store (snapshot + outbox)

**Files:**
- Create: `web/src/store/offline-dispatch.ts`
- Test: `web/src/store/offline-dispatch.test.ts`

**Interfaces:**
- Produces: `useOfflineDispatchStore` (zustand) with state `{ slotByDevice: Record<string, number>; waitingDeviceTickets: SnapshotTicket[]; outbox: OutboxEntry[] }` and actions `setSlot(deviceId, hubSlot)`, `slotFor(deviceId): number | null`, `setWaitingSnapshot(tickets)`, `shiftWaitingDeviceTicket(): SnapshotTicket | null`, `appendOutbox(entry)`, `clearOutbox(ticketIds)`, `hydrate()`. `OutboxEntry = { ticketId: string; action: OfflineAction; at: string }`; `SnapshotTicket = { ticketId: string; number: string; deviceId: string; hubSlot: number | null }`. The waiting snapshot is the **only** offline data source for "call next" (the live waiting list is backend-fetched and unavailable offline). Persists to `localStorage` (key `offlineDispatch`) via a manual helper mirroring `store/queue.ts`.

- [ ] **Step 1: Write the failing test** (localStorage is shimmed by `vitest.setup.ts`)

```ts
import { describe, expect, it } from "vitest";
import { useOfflineDispatchStore } from "@/store/offline-dispatch";

describe("offline-dispatch store", () => {
  it("records slot mapping and resolves it", () => {
    useOfflineDispatchStore.getState().setSlot("dev-1", 4);
    expect(useOfflineDispatchStore.getState().slotFor("dev-1")).toBe(4);
    expect(useOfflineDispatchStore.getState().slotFor("missing")).toBeNull();
  });
  it("appends to outbox, persists, and clears by ticketId", () => {
    useOfflineDispatchStore.getState().appendOutbox({ ticketId: "t1", action: "SERVE", at: "2026-06-24T00:00:00Z" });
    expect(JSON.parse(localStorage.getItem("offlineDispatch") ?? "{}").outbox).toHaveLength(1);
    useOfflineDispatchStore.getState().clearOutbox(["t1"]);
    expect(useOfflineDispatchStore.getState().outbox).toHaveLength(0);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd web && yarn vitest run src/store/offline-dispatch.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement** (manual persistence, mirroring `store/queue.ts`)

```ts
"use client";

import { create } from "zustand";
import type { OfflineAction } from "@/types/queue";

export interface OutboxEntry {
  ticketId: string;
  action: OfflineAction;
  at: string;
}

export interface SnapshotTicket {
  ticketId: string;
  number: string;
  deviceId: string;
  hubSlot: number | null;
}

interface PersistShape {
  slotByDevice: Record<string, number>;
  waitingDeviceTickets: SnapshotTicket[];
  outbox: OutboxEntry[];
}

interface OfflineDispatchState extends PersistShape {
  setSlot: (deviceId: string, hubSlot: number) => void;
  slotFor: (deviceId: string) => number | null;
  setWaitingSnapshot: (tickets: SnapshotTicket[]) => void;
  shiftWaitingDeviceTicket: () => SnapshotTicket | null;
  appendOutbox: (entry: OutboxEntry) => void;
  clearOutbox: (ticketIds: string[]) => void;
  hydrate: () => void;
}

const STORAGE_KEY = "offlineDispatch";

export const useOfflineDispatchStore = create<OfflineDispatchState>()((set, get) => {
  const persist = () => {
    const { slotByDevice, waitingDeviceTickets, outbox } = get();
    localStorage.setItem(STORAGE_KEY, JSON.stringify({ slotByDevice, waitingDeviceTickets, outbox }));
  };
  return {
    slotByDevice: {},
    waitingDeviceTickets: [],
    outbox: [],

    setSlot: (deviceId, hubSlot) => {
      set({ slotByDevice: { ...get().slotByDevice, [deviceId]: hubSlot } });
      persist();
    },

    slotFor: (deviceId) => get().slotByDevice[deviceId] ?? null,

    setWaitingSnapshot: (tickets) => {
      set({ waitingDeviceTickets: tickets });
      persist();
    },

    shiftWaitingDeviceTicket: () => {
      const [head, ...rest] = get().waitingDeviceTickets;
      if (!head) return null;
      set({ waitingDeviceTickets: rest });
      persist();
      return head;
    },

    appendOutbox: (entry) => {
      set({ outbox: [...get().outbox, entry] });
      persist();
    },

    clearOutbox: (ticketIds) => {
      const remove = new Set(ticketIds);
      set({ outbox: get().outbox.filter((e) => !remove.has(e.ticketId)) });
      persist();
    },

    hydrate: () => {
      const raw = localStorage.getItem(STORAGE_KEY);
      if (!raw) return;
      try {
        const parsed = JSON.parse(raw) as Partial<PersistShape>;
        set({
          slotByDevice: parsed.slotByDevice ?? {},
          waitingDeviceTickets: parsed.waitingDeviceTickets ?? [],
          outbox: parsed.outbox ?? [],
        });
      } catch {
        // ignore corrupt data
      }
    },
  };
});
```

- [ ] **Step 4: Run lint + test**

Run: `cd web && yarn lint && yarn vitest run src/store/offline-dispatch.test.ts`
Expected: PASS.

- [ ] **Step 5: Log to CHANGELOGS.**

---

# Phase 4 — Frontend integration (hooks + components; manual QA)

> These tasks wire the Phase-3 pure logic into React. They are **not** node-unit-tested (no jsdom). Verification per task: `yarn lint` passes, and the **manual QA** steps listed. The full type-check/build runs in the audit flow.

### Task 12: SSE connection state + `useBackendReachability`

**Files:**
- Modify: `web/src/hooks/use-queue-events.ts`
- Create: `web/src/hooks/use-backend-reachability.ts`

**Interfaces:**
- Consumes: `reachabilityReducer`, `INITIAL_REACHABILITY`, `isReachable` (Task 9).
- Produces: `useQueueEvents(storeId, onEvent, onConnectionChange?)` where `onConnectionChange?: (state: "open" | "closed") => void`; `useBackendReachability(storeId): { reachable: boolean; onSseEvent: () => void; onApiError: () => void }` — actually exposes `reachable` and feeds an internal reducer from SSE state.

- [ ] **Step 1: Add connection callback to `useQueueEvents`**

```ts
export function useQueueEvents(
  storeId: string | null,
  onEvent: QueueEventHandler,
  onConnectionChange?: (state: "open" | "closed") => void,
) {
  const onEventRef = useRef(onEvent);
  onEventRef.current = onEvent;
  const onConnRef = useRef(onConnectionChange);
  onConnRef.current = onConnectionChange;

  useEffect(() => {
    if (!storeId) return;
    const url = `${API_BASE_URL}${API_ROUTES.QUEUE.EVENTS(storeId)}`;
    const eventSource = new EventSource(url, { withCredentials: true });
    eventSource.onopen = () => onConnRef.current?.("open");
    eventSource.onerror = () => onConnRef.current?.("closed");
    // ... existing addEventListener loop unchanged ...
    return () => eventSource.close();
  }, [storeId]);
}
```

- [ ] **Step 2: Implement `use-backend-reachability.ts`**

```ts
"use client";

import { useCallback, useReducer } from "react";
import {
  INITIAL_REACHABILITY,
  isReachable,
  reachabilityReducer,
} from "@/lib/dispatch/reachability";

export function useBackendReachability() {
  const [state, dispatch] = useReducer(reachabilityReducer, INITIAL_REACHABILITY);
  const onConnectionChange = useCallback((s: "open" | "closed") => {
    dispatch({ type: s === "open" ? "sse_open" : "sse_error" });
  }, []);
  const onApiError = useCallback(() => dispatch({ type: "api_error" }), []);
  return { reachable: isReachable(state), onConnectionChange, onApiError };
}
```

- [ ] **Step 3: Verify lint** — Run: `cd web && yarn lint` → no errors.

- [ ] **Step 4: Manual QA (audit flow)** — with the queue page open, kill the backend; confirm `reachable` flips to false after ~2 SSE errors and back to true on reconnect.

- [ ] **Step 5: Log to CHANGELOGS.**

---

### Task 13: `useDispatchMode` + `useSerialDispatch`

**Files:**
- Create: `web/src/hooks/use-dispatch-mode.ts`
- Create: `web/src/hooks/use-serial-dispatch.ts`

**Interfaces:**
- Consumes: `deriveDispatchMode` (Task 7), `buildSerialDispatch` (Task 8), `useSerial` (existing), `getUsbDispatchPayload` (existing), `useBackendReachability` (Task 12).
- Produces:
  - `useDispatchMode({ reachable, dispatchReady, serial }): DispatchMode` — computes `hubConnectedForStore` from `serial.portState === "open"` && `serial.identifyPayload?.device_kind === "TRANSMITTER_HUB"` && `serial.deviceState?.op_state === "ACTIVE"`.
  - `useSerialDispatch(storeId, serial)` → `dispatch(target: { id: string; hubSlot: number | null }, action): Promise<TransmitResult>` that resolves the `SerialDispatchPlan` and calls `serial.sendCommand("transmit_slot", ...)` or `getUsbDispatchPayload` + `serial.sendCommand("transmit", ...)`. The minimal `{ id, hubSlot }` shape (not full `DeviceDto`) is deliberate: at CALL time the device is busy and absent from `getAvailableDevices`, so callers supply `id` from the SSE event/serving ticket and `hubSlot` from the persisted snapshot.

- [ ] **Step 1: Implement `use-dispatch-mode.ts`**

```ts
"use client";

import { useMemo } from "react";
import { deriveDispatchMode, type DispatchMode } from "@/lib/dispatch/mode";
import type { UseSerialReturn } from "@/lib/serial/use-serial";

export function useDispatchMode(args: {
  reachable: boolean;
  dispatchReady: boolean;
  serial: UseSerialReturn;
}): DispatchMode {
  const { reachable, dispatchReady, serial } = args;
  const hubConnectedForStore =
    serial.portState === "open" &&
    serial.identifyPayload?.device_kind === "TRANSMITTER_HUB" &&
    serial.deviceState?.op_state === "ACTIVE";
  return useMemo(
    () => deriveDispatchMode({ backendReachable: reachable, dispatchReady, hubConnectedForStore: !!hubConnectedForStore }),
    [reachable, dispatchReady, hubConnectedForStore],
  );
}
```

- [ ] **Step 2: Implement `use-serial-dispatch.ts`**

```ts
"use client";

import { useCallback } from "react";
import { buildSerialDispatch, type SerialDispatchAction } from "@/lib/dispatch/serial-command";
import { getUsbDispatchPayload } from "@/features/device/api";
import type { UseSerialReturn } from "@/lib/serial/use-serial";
import type { TransmitResult } from "@/lib/serial/types";

export function useSerialDispatch(storeId: string, serial: UseSerialReturn) {
  return useCallback(
    async (
      target: { id: string; hubSlot: number | null },
      action: SerialDispatchAction,
    ): Promise<TransmitResult> => {
      const plan = buildSerialDispatch({ hubSlot: target.hubSlot }, action);
      if (plan.kind === "slot") {
        return serial.sendCommand("transmit_slot", { slot: plan.slot, action: plan.action });
      }
      const payload = await getUsbDispatchPayload({ storeId, deviceId: target.id, action: plan.action });
      return serial.sendCommand("transmit", {
        receiver_public_id: payload.receiverPublicId,
        band: payload.band,
        rf_code_hex: payload.rfCodeHex,
        rf_code_bits: payload.rfCodeBits,
        proto_any: payload.protoAny,
      });
    },
    [storeId, serial],
  );
}
```

> **Implementer note:** confirm `UseSerialReturn` is exported from `lib/serial/use-serial.ts` (it is). Keep the `sendCommand` generic call shape identical to `usb-dispatch-dialog.tsx`.

- [ ] **Step 3: Verify lint** — Run: `cd web && yarn lint` → no errors.

- [ ] **Step 4: Log to CHANGELOGS.**

---

### Task 14: `DispatchModeBanner` + i18n keys

**Files:**
- Create: `web/src/features/queue/dispatch-mode-banner.tsx`
- Modify: `web/src/messages/en.json`, `web/src/messages/vi.json`

**Interfaces:**
- Consumes: `DispatchMode`.
- Produces: `<DispatchModeBanner mode={DispatchMode} />` — renders only for `ONLINE_SERIAL_FALLBACK` / `OFFLINE_SERIAL`.

- [ ] **Step 1: Add i18n keys** — in `en.json` `queue.dispatch` (after `nextPage`), add:

```json
      "banner": {
        "serialFallback": "Transmitter offline — dispatching over USB.",
        "offline": "No connection — running the queue locally over USB. Changes sync when you're back online."
      },
      "offlineIssueDisabled": "Can't add new pagers while offline. You can still call and finish current ones.",
      "rePageUsb": "Re-page over USB",
      "errorReceiverNotOnHub": "Receiver isn't paired to this hub.",
      "syncedToast": "Synced {count} offline updates."
```

In `vi.json` `queue.dispatch`, the **PROPOSED** mirror (pending native review — do not finalize without sign-off):

```json
      "banner": {
        "serialFallback": "Bộ phát ngoại tuyến — đang phát tín hiệu qua USB.",
        "offline": "Mất kết nối — đang chạy hàng đợi cục bộ qua USB. Thay đổi sẽ đồng bộ khi có mạng trở lại."
      },
      "offlineIssueDisabled": "Không thể cấp thiết bị mới khi ngoại tuyến. Vẫn có thể gọi và phục vụ khách hiện tại.",
      "rePageUsb": "Gọi lại qua USB",
      "errorReceiverNotOnHub": "Bộ thu chưa ghép nối với bộ phát này.",
      "syncedToast": "Đã đồng bộ {count} cập nhật ngoại tuyến."
```

- [ ] **Step 2: Implement the banner** (canonical Warning alert box; `Usb` icon from lucide-react; `text-warning`, `rounded-xl`, dark overrides, icon `shrink-0`, `gap-2.5`, `px-3.5 py-3`):

```tsx
"use client";

import { Usb } from "lucide-react";
import { useTranslations } from "next-intl";
import type { DispatchMode } from "@/lib/dispatch/mode";

export function DispatchModeBanner({ mode }: { mode: DispatchMode }) {
  const tQueue = useTranslations("queue");
  if (mode !== "ONLINE_SERIAL_FALLBACK" && mode !== "OFFLINE_SERIAL") return null;
  const key = mode === "OFFLINE_SERIAL" ? "dispatch.banner.offline" : "dispatch.banner.serialFallback";
  return (
    <div className="flex items-center gap-2.5 rounded-xl border border-warning/40 bg-warning/15 px-3.5 py-3 text-sm text-warning dark:border-warning/50 dark:bg-warning/20">
      <Usb aria-hidden="true" className="size-4 shrink-0" />
      <span>{tQueue(key)}</span>
    </div>
  );
}
```

- [ ] **Step 3: Verify lint** — Run: `cd web && yarn lint` → no errors.

- [ ] **Step 4: Manual QA** — render in both modes, light + dark; confirm the box matches the existing paused banner styling.

- [ ] **Step 5: Log to CHANGELOGS.**

---

### Task 15: Mode-aware `DeviceDispatchPanel`

**Files:**
- Modify: `web/src/features/queue/device-dispatch-panel.tsx`

**Interfaces:**
- Consumes: `DispatchMode` (passed as a prop from the page), `issueDeviceTicket` with `allowSerialFallback`.
- Produces: panel that enables dispatch when `mode !== "DISABLED"`, sends `allowSerialFallback: mode !== "ONLINE_MQTT"` on issue, disables new-ticket issuance in `OFFLINE_SERIAL` with the `offlineIssueDisabled` note.

- [ ] **Step 1: Accept a `mode` prop and derive readiness** — change `DeviceDispatchPanelProps` to add `mode: DispatchMode`; compute `const canDispatch = mode !== "DISABLED" && mode !== "OFFLINE_SERIAL";` for issuance (issuance is online-only per spec §2), and pass `dispatchReady={canDispatch}` to each `DeviceDispatchCard`. **Remove the panel's internal no-hub warning** (`device-dispatch-panel.tsx:176-184`, the `!loading && !dispatchReady` block) — the page now renders `DispatchModeBanner` above the panel, so keeping the internal one would double the warning. The panel keeps its own `getAvailableDevices` fetch for the device list but no longer uses its internal `dispatchReady` for card enablement (mode supersedes it).

- [ ] **Step 2: Thread `allowSerialFallback` into the issue call** — in `handleDispatch`:

```ts
const ticket = await issueDeviceTicket(storeId, {
  deviceId,
  allowSerialFallback: mode === "ONLINE_SERIAL_FALLBACK",
});
```

- [ ] **Step 3: Render the offline note** — when `mode === "OFFLINE_SERIAL"`, replace the device list with the `offlineIssueDisabled` message (reuse the existing empty-state `<p className="...text-muted-foreground">` style; do not introduce new components).

- [ ] **Step 4: Verify lint** — Run: `cd web && yarn lint` → no errors.

- [ ] **Step 5: Manual QA** — in `ONLINE_SERIAL_FALLBACK` the Dispatch button is enabled and issuance succeeds; in `OFFLINE_SERIAL` issuance is replaced by the note.

- [ ] **Step 6: Log to CHANGELOGS.**

---

### Task 16: Wire the page — mode, banner, Tier-1 retry, reconcile-on-reconnect

**Files:**
- Modify: `web/src/app/[locale]/dashboard/queue/page.tsx`

**Interfaces:**
- Consumes: `useBackendReachability` (12), `useSerial` (existing), `useDispatchMode` (13), `useSerialDispatch` (13), `getAvailableDevices` (existing), `DispatchModeBanner` (14), `createDispatchDedupe` (10), `useOfflineDispatchStore` (11), `reconcileOffline` (6).

- [ ] **Step 1: Add hooks + a page-owned availability fetch** — instantiate `const serial = useSerial();` and `const { reachable, onConnectionChange } = useBackendReachability();`. Add `const [dispatchReady, setDispatchReady] = useState(false);` fed by a page-owned `getAvailableDevices(storeId)` effect (keyed on `storeId` + `deviceRefreshSignal`). **This is a separate fetch from `DeviceDispatchPanel`'s** (the panel keeps its own for its paginated list); the minor double-fetch of the same cheap endpoint is accepted to keep the two concerns decoupled. In this effect, for every returned device with `hubSlot != null` call `useOfflineDispatchStore.getState().setSlot(device.id, device.hubSlot)` — because `setSlot` never removes, slots captured while a device is available remain resolvable after it becomes busy. Compute `const mode = useDispatchMode({ reachable, dispatchReady, serial });` and `const serialDispatch = useSerialDispatch(storeId, serial);`.

- [ ] **Step 2: Pass `onConnectionChange` to SSE** — `useQueueEvents(storeId, handler, onConnectionChange)`.

- [ ] **Step 3: Hydrate + maintain the waiting snapshot** — call `useOfflineDispatchStore.getState().hydrate()` once on mount. In the same availability effect (and on `statsRefreshSignal`), when `reachable`, fetch `listWaitingTickets(storeId)`, keep only device-bound tickets (`t.deviceId != null`), map each to `{ ticketId: t.id, number: t.number, deviceId: t.deviceId, hubSlot: slotFor(t.deviceId) }`, and call `setWaitingSnapshot(...)`. This is the only data source for offline call-next (Task 17).

- [ ] **Step 4: Tier-1 retry in the SSE handler** — replace the current `no_active_transmitter` branch (`queue/page.tsx:129-130`) with: if `mode === "ONLINE_SERIAL_FALLBACK"` and `event.deviceId` and `event.dispatchAction` and the dedupe admits `${event.ticketId}:${event.dispatchAction}:${event.deviceId}`, resolve `const hubSlot = useOfflineDispatchStore.getState().slotFor(event.deviceId);` and `const result = await serialDispatch({ id: event.deviceId, hubSlot }, event.dispatchAction);`; on `result.status === "applied"` show nothing, else `toast.error(tQueue("dispatch.errorNoActiveTransmitter"))`. Otherwise keep the existing toast. Create one `const dedupe = useRef(createDispatchDedupe()).current;`.

- [ ] **Step 5: Reconcile on reconnect** — add an effect keyed on `reachable`, using a `prevReachableRef` to detect the `false → true` rising edge. On that edge, read `const transitions = useOfflineDispatchStore.getState().outbox.map(({ ticketId, action, at }) => ({ ticketId, action, at }));` if non-empty: `await reconcileOffline(storeId, { transitions })`. **On a resolved response, clear every sent item** — `clearOutbox(transitions.map((t) => t.ticketId))` — because `applied`, `superseded`, and `gone` are all terminal outcomes (nothing to retry). **Only if the call itself rejects** (network) do items remain for the next rising edge. Then `toast.success(tQueue("dispatch.syncedToast", { count: transitions.length }))` and bump `deviceRefreshSignal` to re-fetch the now-authoritative queue.

- [ ] **Step 6: Render the banner** — add `<DispatchModeBanner mode={mode} />` directly under the paused banner block (`queue/page.tsx:249`). Pass `mode` to `<DeviceDispatchPanel ... mode={mode} />`.

- [ ] **Step 7: Verify lint** — Run: `cd web && yarn lint` → no errors.

- [ ] **Step 8: Manual QA** — Tier-1 drill: stop the MQTT hub, connect USB hub, call a device ticket → RF over serial, no error toast. Reconnect drill in Task 17.

- [ ] **Step 9: Log to CHANGELOGS.**

---

### Task 17: Offline calls in `ServingDisplay` + call-next + re-page button

**Files:**
- Modify: `web/src/features/queue/serving-display.tsx`
- Modify: `web/src/app/[locale]/dashboard/queue/page.tsx` (call-next offline branch)

**Interfaces:**
- Consumes: `useOfflineDispatchStore` (11), `useSerialDispatch` (13), `DispatchMode`.
- Produces: serve/cancel/no-show and call-next that, when `mode === "OFFLINE_SERIAL"`, dispatch `transmit_slot` over serial + append to the outbox + mutate the local serving store, instead of hitting the backend; a "Re-page over USB" button on serving device tickets in either serial mode.

- [ ] **Step 1: Pass `mode`, `serialDispatch`, and the bound `DeviceDto` resolver into `ServingDisplay`** — add props `mode: DispatchMode` and `dispatchSerial: (device: DeviceDto, action: "call" | "stop") => Promise<TransmitResult>`. (The serving ticket carries `deviceId`; resolve `hubSlot` via `useOfflineDispatchStore.getState().slotFor(deviceId)` and build a minimal `{ id: deviceId, hubSlot } as DeviceDto`-shaped arg for `buildSerialDispatch` — or call a thin `dispatchByDeviceId(deviceId, action)` helper added to `useSerialDispatch`.)

- [ ] **Step 2: Branch the terminal handlers** — in `handleServe`/`handleCancel`/`handleNoShow`, when `mode === "OFFLINE_SERIAL"`:

```ts
const store = useOfflineDispatchStore.getState();
// Non-device tickets can't be progressed offline (public display is backend-driven).
if (!ticket.deviceId) {
  toast.warning(tQueue("dispatch.offlineIssueDisabled"));
  return;
}
// Best-effort STOP RF: only hub-paired (slot) receivers are reachable offline; a standalone
// receiver (no cached slot) can't be paged offline, but the terminal transition is still recorded.
const hubSlot = store.slotFor(ticket.deviceId);
if (hubSlot != null) {
  const res = await dispatchSerial({ id: ticket.deviceId, hubSlot }, "stop");
  if (res.status !== "applied" && res.reason === "slot_not_found") {
    toast.error(tQueue("dispatch.errorReceiverNotOnHub"));
  }
}
store.appendOutbox({
  ticketId: ticket.id,
  action: "SERVE", // or "CANCEL" / "NO_SHOW" per handler
  at: new Date().toISOString(),
});
removeServingTicket(ticket.id);
return;
```

- [ ] **Step 3: Offline call-next** — in `page.tsx` `handleCallNext`, when `mode === "OFFLINE_SERIAL"`: pull the next snapshot ticket via `const next = useOfflineDispatchStore.getState().shiftWaitingDeviceTicket();` (returns `null` when none — show `tQueue("queueEmpty")`). If `next.hubSlot != null`, `await serialDispatch({ id: next.deviceId, hubSlot: next.hubSlot }, "call")` (on `slot_not_found` toast `errorReceiverNotOnHub`). Then `addServingTicket` with a reconstructed `TicketDto` (`{ id: next.ticketId, number: next.number, status: "CALLED", calledAt: new Date().toISOString(), issuedAt: null, position: null, deviceId: next.deviceId, deviceName: null }`). Do **not** call the backend, and do **not** append to the outbox — a still-serving offline call carries no terminal entry yet (spec §6 "in-progress at reconnect"). New tickets are never issued offline; this only advances pagers already in the snapshot.

- [ ] **Step 4: Add the re-page button** — on each serving card with `ticket.deviceId`, when `mode` is a serial mode, render a shadcn `Button variant="outline" size="sm"` with a `Radio` icon and label `tQueue("dispatch.rePageUsb")` that calls `dispatchSerial(device, "call")`; on `rejected` `slot_not_found`, `toast.error(tQueue("dispatch.errorReceiverNotOnHub"))`.

- [ ] **Step 5: Verify lint** — Run: `cd web && yarn lint` → no errors.

- [ ] **Step 6: Manual QA — Tier-2 drill** — kill the backend; serve/cancel current device tickets → `transmit_slot` over serial, outbox grows; restore backend → reconcile POST fires, toast shows synced count, **no double-page**, queue refreshes.

- [ ] **Step 7: Log to CHANGELOGS.**

---

# Phase 5 — Integration & documentation

### Task 18: Cross-layer drill checklist + CHANGELOGS finalization

**Files:**
- Modify: `docs/CHANGELOGS.md`

- [ ] **Step 1: Record the manual integration drills** (to be executed in the audit/build flow) under the CHANGELOGS "Resilient USB Dispatch" heading: Tier-1 drill (Task 16 Step 8), Tier-2 + reconcile drill (Task 17 Step 6), firmware on-device checks (Tasks 4–5), and the **double-page guard** assertion (reconcile must not emit `DEVICE_STOP_REQUESTED`).
- [ ] **Step 2: Confirm every skipped/deferred item is logged** — standalone-receiver offline gap, non-device offline boundary, long-outage TTL fidelity (spec §11), and the firmware no-host-test gap.
- [ ] **Step 3: Note the pending Vietnamese copy review** — `vi.json` strings added in Task 14 remain provisional until native sign-off.

---

## Self-Review (run before handoff)

1. **Spec coverage:** Tier-1 fallback (Tasks 1,2,13,15,16) ✓; Tier-2 offline (Tasks 4,5,11,17) ✓; reconcile (Tasks 3,16,17) ✓; mode machine (Task 7) ✓; banner/UI rules (Task 14) ✓; slot-vs-payload branch (Task 8,13) ✓; firmware `transmit_slot` (Tasks 4,5) ✓; debounce/dedupe (Tasks 9,10) ✓; i18n bilingual (Task 14) ✓.
2. **Placeholder scan:** every code step shows real code; firmware verification is explicitly manual (constraint-driven), not a placeholder.
3. **Type consistency:** `DispatchMode`, `SerialDispatchPlan`, `OfflineAction`, `OutboxEntry`, `slot_exec_result_t`, `ReconcileItemResult.result` values (`applied|superseded|gone`) are used consistently across tasks; `transmit_slot` payload/response match `SerialCommandMap`.
