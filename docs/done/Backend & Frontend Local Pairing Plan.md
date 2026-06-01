# Backend & Frontend — Local Pairing Integration

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Integrate hub-managed receiver pairing into the backend and frontend. Add: (1) `hub_slot` schema migration, (2) MQTT roster listener with signed ACK, (3) slot-based dispatch path in `TransmitterDispatchService`, (4) receiver label push, (5) unpair command, (6) frontend hub-paired device indicators.

**Architecture:** Hub-paired receivers are distinguished by `hub_slot IS NOT NULL` on the `device` table. Roster updates arrive from the hub via MQTT and are processed by a new `RosterSyncListener`. Dispatch for hub-paired receivers sends a lightweight `{slot, action}` payload on the existing `cmd/transmit` topic. Passive receivers continue using the full RF code payload unchanged.

**Tech Stack:** Kotlin 2.3 / Spring Boot 3.5 / WebFlux / R2DBC / TimescaleDB / Next.js 16 / React 19 / TypeScript 5

**Spec:** `docs/spec/Local Pairing & Hub-Managed Dispatch Spec.md` — sections 5, 6, 7, 8, 9

---

## Existing Codebase Context

### Key files


| File                                                     | Role                                                                                |
| -------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| `backend/.../entity/Device.kt`                           | Device entity — `id`, `publicId`, `kind`, `status`, `assignedName`, `storeId`, etc. |
| `backend/.../types/DeviceKind.kt`                        | `RECEIVER_433M`, `RECEIVER_433M_PASSIVE`, `RECEIVER_2_4G`, `TRANSMITTER_HUB`        |
| `backend/.../service/TransmitterDispatchService.kt`      | Builds signed RF payload, publishes `cmd/transmit` via MQTT                         |
| `backend/.../service/TransmitterElectionService.kt`      | Redis hub liveness election                                                         |
| `backend/.../service/DeviceDispatchService.kt`           | Pre-dispatch checks, busy key, ticket issuance                                      |
| `backend/.../listener/TransmitterOperationalListener.kt` | MQTT heartbeat/ack handler                                                          |
| `backend/.../core/device/DeviceMqttPublisher.kt`         | MQTT publish helpers (`publishTransmit`, `publishDeact`, etc.)                      |
| `backend/.../core/device/DeviceCanonical.kt`             | Signature canonical string builders                                                 |
| `backend/.../core/device/DeviceCommandSigner.kt`         | ECDSA P-256 `SHA256withECDSA` signer                                                |
| `backend/.../core/mqtt/MqttProperties.kt`                | `topicPrefix` = `"notiguide"`                                                       |
| `backend/.../db/schema.sql`                              | `device` + `device_rf_code` table definitions                                       |


### MQTT topic prefix

All topics use `MqttProperties.topicPrefix` (default: `notiguide`).

### Signature infrastructure

`DeviceCommandSigner.sign(canonical: String): String` — signs with EC P-256, SHA256withECDSA, returns base64. Existing canonicals: `transmit-v1|...`, `deact-v1|...`, `rf-code-v1|...`, `activate-v1|...`.

---

### Task 1: Schema migration

**Files:**

- Create: `backend/src/main/resources/db/migration/V2__add_hub_slot.sql` 
- **Step 1: Write migration SQL**

```sql
ALTER TABLE device ADD COLUMN IF NOT EXISTS hub_slot SMALLINT;
ALTER TABLE device ADD COLUMN IF NOT EXISTS last_roster_seq INT;
```

- **Step 2: Update `schema.sql` to include new columns**

Add the two columns to the `CREATE TABLE device` statement so fresh installs get them:

```sql
-- After the existing columns, before the closing );
    hub_slot SMALLINT,
    last_roster_seq INT,
```

---

### Task 2: Update Device entity

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/entity/Device.kt`
- **Step 1: Add `hubSlot` and `lastRosterSeq` fields**

Add two nullable fields after `activatedAt`:

```kotlin
@Column("hub_slot")
val hubSlot: Short? = null,

@Column("last_roster_seq")
val lastRosterSeq: Int? = null,
```

- **Step 2: Update `equals` and `hashCode`**

Add `hubSlot` and `lastRosterSeq` to both methods following the existing pattern.

---

### Task 3: Add `DeviceCanonical.rosterAck()`

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/core/device/DeviceCanonical.kt`
- **Step 1: Add roster ACK canonical builder**

```kotlin
fun rosterAck(
    hubPublicId: String,
    seq: Int,
    issuedAt: OffsetDateTime
): String = "roster-ack-v1|$hubPublicId|$seq|${issuedAt.toInstant()}"
```

---

### Task 4: Add `DeviceCanonical.slotDispatch()`

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/core/device/DeviceCanonical.kt`
- **Step 1: Add slot dispatch canonical builder**

```kotlin
fun slotDispatch(
    hubPublicId: String,
    dispatchId: UUID,
    slot: Int,
    action: String,
    issuedAt: OffsetDateTime
): String = "dispatch-v1|$hubPublicId|$dispatchId|$slot|$action|${issuedAt.toInstant()}"
```

---

### Task 5: Add MQTT publish methods to `DeviceMqttPublisher`

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/core/device/DeviceMqttPublisher.kt`
- **Step 1: Add `publishRosterAck` method**

```kotlin
suspend fun publishRosterAck(
    hubPublicId: String,
    payload: Any
) {
    publishJson(
        topic = "${mqttProperties.topicPrefix}/transmitter/hub/$hubPublicId/roster/ack",
        payload = payload,
        retained = false
    )
}
```

- **Step 2: Add `publishLabel` method**

```kotlin
suspend fun publishLabel(
    hubPublicId: String,
    payload: Any
) {
    publishJson(
        topic = "${mqttProperties.topicPrefix}/transmitter/hub/$hubPublicId/cmd/label",
        payload = payload,
        retained = false
    )
}
```

- **Step 3: Add `publishUnpair` method**

```kotlin
suspend fun publishUnpair(
    hubPublicId: String,
    payload: Any
) {
    publishJson(
        topic = "${mqttProperties.topicPrefix}/transmitter/hub/$hubPublicId/cmd/unpair",
        payload = payload,
        retained = false
    )
}
```

---

### Task 6: Create `RosterSyncListener`

**Files:**

- Create: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/listener/RosterSyncListener.kt`
- **Step 1: Write the listener**

This component subscribes to `{prefix}/transmitter/hub/+/roster/update`, processes roster payloads, upserts device records, and publishes signed ACKs.

```kotlin
@Component
@ConditionalOnBean(MqttClientManager::class)
@ConditionalOnProperty(prefix = "device.transmitter", name = ["enabled"], havingValue = "true")
class RosterSyncListener(
    private val mqttClientManager: MqttClientManager,
    private val mqttProperties: MqttProperties,
    private val objectMapper: ObjectMapper,
    private val deviceRepository: DeviceRepository,
    private val client: DatabaseClient,
    private val publisherProvider: ObjectProvider<DeviceMqttPublisher>,
    private val signerProvider: ObjectProvider<DeviceCommandSigner>
) : SmartInitializingSingleton {

    private val log = LoggerFactory.getLogger(this::class.java)
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
    private val rosterTopic by lazy {
        "${mqttProperties.topicPrefix}/transmitter/hub/+/roster/update"
    }

    // ... handler registration, message parsing, upsert logic
}
```

The handler must:

1. Extract `publicId` from the topic
2. Parse the JSON roster payload (`schema_version`, `seq`, `receivers[]`)
3. Look up the hub device by `publicId`
4. Check `last_roster_seq` for idempotency — if `seq <= lastRosterSeq`, re-ACK without processing
5. For each receiver in roster: upsert device record with `hubSlot`, `kind` from band, `status = ACTIVE`, `storeId` from hub
6. Delete hub-paired devices whose `hubSlot` is not in the roster (unpaired)
7. Update `last_roster_seq` on the hub device
8. Sign and publish ACK with canonical `roster-ack-v1|{hubPublicId}|{seq}|{issuedAt}`
9. Include `schema_version = 1` in every roster ACK payload; the hub rejects ACKs without it

ACK payload:

```kotlin
private data class RosterAckEnvelope(
    @field:JsonProperty("schema_version")
    val schemaVersion: Int = 1,
    val seq: Int,
    @field:JsonProperty("issued_at")
    val issuedAt: String,
    @field:JsonProperty("signature_b64")
    val signatureB64: String
)
```

**Note:** Roster updates are only triggered by pair/unpair events on the hub. Label changes (`cmd/label`) update the hub's NVS locally without incrementing the roster sequence — the backend will not receive a roster update after pushing a label. The `receivers[].label` field in roster updates reflects the hub's current NVS state at the time of the next pair/unpair event.

---

### Task 7: Modify `TransmitterDispatchService` for slot-based dispatch

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/service/TransmitterDispatchService.kt`

**Important:** The `hubSlot` branch must be inserted BEFORE the existing `deviceRfCodeRepository.findDecryptedPayload()` call. Hub-paired receivers have no entry in `device_rf_code` — their RF codes live only on the hub NVS. Hitting the RF code lookup for a hub-paired device would fail or return null.

**Ordering preservation requirements:**

- Keep hub election, receiver lookup, and the null checks for `receiver`, `hubPublicId`, and `receiver.publicId` before any dispatch branch.
- Check `receiver.hubSlot != null` immediately after those null checks, before resolving `signer`, `publisher`, decrypted RF payloads, or any full-payload canonical string.
- Return immediately after `handleSlotDispatch(...)` so hub-paired receivers never call `deviceRfCodeRepository.findDecryptedPayload()`, `patchRfCode(...)`, or `DeviceCanonical.transmit(...)`.
- Keep the existing full-payload path below the slot branch unchanged for passive/server-managed receivers where `hubSlot == null`.

- **Step 1: Restructure `handle()` to branch on `hubSlot` before RF code lookup**

After the existing null checks on `receiver`, `hubPublicId`, and `receiver.publicId`, insert the slot dispatch branch. The existing RF code lookup and full-payload path remain below, unchanged:

```kotlin
private suspend fun handle(event: DeviceDispatchEvent) {
    val hub = transmitterElectionService.electActive(event.storeId)
    if (hub == null) {
        log.warn(
            "transmitter_dispatch_election_lost store={} ticket={} device={}",
            event.storeId, event.ticketId, event.deviceId
        )
        emitDispatchFailed(event, "no_active_transmitter")
        return
    }

    val receiver = deviceRepository.findById(event.deviceId)
    val hubPublicId = hub.publicId
    if (receiver == null || hubPublicId == null || receiver.publicId == null) {
        log.warn(
            "transmitter_dispatch_failed_missing_device store={} ticket={} device={} hub={}",
            event.storeId, event.ticketId, event.deviceId, hub.id
        )
        emitDispatchFailed(event, "device_not_found")
        return
    }

    // Hub-paired receivers: slot-based dispatch (no RF code lookup needed)
    if (receiver.hubSlot != null) {
        handleSlotDispatch(hub, receiver, event)
        return
    }

    // --- Existing full-payload dispatch for passive/server-managed receivers ---
    // (everything below this line is the existing code, unchanged)
    val signer = signerProvider.ifAvailable
    val publisher = publisherProvider.ifAvailable
    val decrypted = runCatching {
        deviceRfCodeRepository.findDecryptedPayload(receiver.id!!)
    }.getOrElse {
        // ... existing error handling
    }
    // ... rest of existing handle() body unchanged
}
```

- **Step 2: Add `handleSlotDispatch` method**

This method mirrors the existing dispatch flow but skips RF code lookup — the hub resolves slot → RF config locally from its NVS roster. The busy key is managed by `DeviceDispatchService` upstream (set on CALL, cleared here on STOP+RELEASE — same pattern as the existing full-payload path).

```kotlin
private suspend fun handleSlotDispatch(
    hub: Device,
    receiver: Device,
    event: DeviceDispatchEvent
) {
    val hubPublicId = hub.publicId ?: return
    val signer = signerProvider.ifAvailable ?: run {
        emitDispatchFailed(event, "infrastructure_unavailable")
        return
    }
    val publisher = publisherProvider.ifAvailable ?: run {
        emitDispatchFailed(event, "infrastructure_unavailable")
        return
    }

    val dispatchId = UUID.randomUUID()
    val issuedAt = OffsetDateTime.now(ZoneOffset.UTC)
    val action = when (event.type) {
        DeviceDispatchEventType.DEVICE_CALL_REQUESTED -> "call"
        DeviceDispatchEventType.DEVICE_STOP_REQUESTED -> "stop"
    }

    val canonical = DeviceCanonical.slotDispatch(
        hubPublicId = hubPublicId,
        dispatchId = dispatchId,
        slot = receiver.hubSlot!!.toInt(),
        action = action,
        issuedAt = issuedAt
    )

    val envelope = SlotDispatchEnvelope(
        dispatchId = dispatchId,
        slot = receiver.hubSlot!!.toInt(),
        action = action,
        issuedAt = issuedAt.toInstant().toString(),
        signatureB64 = signer.sign(canonical)
    )

    runCatching {
        publisher.publishTransmit(hubPublicId, envelope)
    }.onFailure { ex ->
        log.warn(
            "transmitter_slot_dispatch_failed_publish store={} ticket={} slot={}",
            event.storeId, event.ticketId, receiver.hubSlot, ex
        )
        emitDispatchFailed(event, "publish_failed")
        return
    }

    if (event.type == DeviceDispatchEventType.DEVICE_STOP_REQUESTED &&
        event.disposition == DeviceDispatchStopDisposition.RELEASE
    ) {
        runCatching {
            redis.delete(RedisKeyManager.deviceBusy(event.deviceId)).awaitSingleOrNull()
        }.onFailure { ex ->
            log.warn(
                "transmitter_slot_dispatch_busy_release_failed store={} ticket={} device={}",
                event.storeId, event.ticketId, event.deviceId, ex
            )
        }
    }
}
```

- **Step 3: Add `SlotDispatchEnvelope` data class**

```kotlin
private data class SlotDispatchEnvelope(
    @field:JsonProperty("schema_version")
    val schemaVersion: Int = 1,
    @field:JsonProperty("dispatch_id")
    val dispatchId: UUID,
    val slot: Int,
    val action: String,
    @field:JsonProperty("issued_at")
    val issuedAt: String,
    @field:JsonProperty("signature_b64")
    val signatureB64: String
)
```

---

### Task 8: Update `DeviceDispatchService` availability filter for hub-paired devices

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/service/DeviceDispatchService.kt`

**Context:** `isDispatchableDevice()` currently requires `device.rfCode != null`. Hub-paired receivers have no entry in `device_rf_code` — their RF codes live only on the hub NVS. Without this fix, hub-paired devices would never appear in `getAvailableDevices()` and the "Send via device" dialog would not list them.

- **Step 1: Update `isDispatchableDevice` to accept hub-paired devices**

Change the predicate to accept devices that either have an RF code (server-managed) or a hub slot (hub-paired):

```kotlin
private fun isDispatchableDevice(
    device: DeviceDto,
    storeId: UUID
): Boolean =
    device.storeId == storeId &&
        !device.kind.isHub() &&
        device.status == DeviceStatus.ACTIVE &&
        (device.rfCode != null || device.hubSlot != null)
```

---

### Task 9: Write dispatch tracking Redis key on publish

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/service/TransmitterDispatchService.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/core/redis/RedisKeyManager.kt`

**Context:** When the hub rejects a dispatch (RF send failure, slot not found, etc.), it publishes a transmit ACK with `status = "rejected"`. The backend currently only logs these rejection ACKs (Task 10 adds the reaction). To map `dispatchId` back to `deviceId`, `storeId`, `ticketId`, and `ticketNumber`, this task writes a transient Redis key at dispatch publish time. The key has a 5-minute TTL — long enough for the hub to ACK, short enough to self-clean.

- **Step 1: Add `dispatchTracking` key to `RedisKeyManager`**

```kotlin
fun dispatchTracking(dispatchId: UUID): String = "dispatch:tracking:$dispatchId"
```

- **Step 2: Add `DispatchTrackingRecord` data class**

Place in a new file or alongside `TransmitterDispatchService`:

```kotlin
data class DispatchTrackingRecord(
    @field:JsonProperty("device_id")
    val deviceId: UUID,
    @field:JsonProperty("store_id")
    val storeId: UUID,
    @field:JsonProperty("ticket_id")
    val ticketId: UUID,
    @field:JsonProperty("ticket_number")
    val ticketNumber: String? = null
)
```

- **Step 3: Write tracking key in `handle()` after successful publish (full-payload path)**

After the existing `publisher.publishTransmit(hubPublicId, payload)` succeeds (around line 154), before the stop/release busy-key block:

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

- **Step 4: Write tracking key in `handleSlotDispatch()` after successful publish**

Same pattern as Step 3, after `publisher.publishTransmit(hubPublicId, envelope)` succeeds.

---

### Task 10: React to transmit ACK rejections

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/listener/TransmitterOperationalListener.kt`

**Context:** When `handleAck()` receives a transmit ACK with `status` other than `"applied"` or `"unchanged"`, the hub has rejected the dispatch (RF send failure, slot not found, etc.). The backend should look up the dispatch tracking key (written in Task 9), emit `DEVICE_DISPATCH_FAILED` so the operator sees the failure, and release the device busy key so the device becomes available for re-dispatch.

**Spec:** `docs/spec/Pairing & Dispatch Robustness Fixes Spec.md` — §9

- **Step 1: Add `QueueEventBroadcaster` dependency**

Add to the constructor:

```kotlin
private val queueEventBroadcaster: QueueEventBroadcaster,
```

- **Step 2: Add rejection branch in `handleAck()` transmit case**

In the `ack.ackFor == "transmit"` branch, after the existing `log.info(...)`, add:

```kotlin
if (ack.status != "applied" && ack.status != "unchanged") {
    handleTransmitRejection(ack.dispatchId, publicId, ack.status, ack.reason)
}
```

- **Step 3: Add `handleTransmitRejection` method**

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

- **Step 4: Add `DispatchTrackingRecord` data class locally**

If `DispatchTrackingRecord` was made package-private in `TransmitterDispatchService`, add a matching deserialization class here:

```kotlin
private data class DispatchTrackingRecord(
    @field:JsonProperty("device_id")
    val deviceId: UUID,
    @field:JsonProperty("store_id")
    val storeId: UUID,
    @field:JsonProperty("ticket_id")
    val ticketId: UUID,
    @field:JsonProperty("ticket_number")
    val ticketNumber: String? = null
)
```

If `DispatchTrackingRecord` was placed in a shared `dto` package in Task 9, import it instead.

---

### Task 11: Add label push on device rename

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/service/DeviceQueryService.kt` (or wherever rename logic lives)

**Firmware coordination note:** The hub's `roster_set_name()` updates NVS locally but does NOT increment the roster sequence or mark the roster as pending sync. This means the backend will NOT receive a roster update in response to a label push — the label is a one-way backend → hub command. This is by design: the label originated from the backend, so no round-trip sync is needed.

- **Step 1: After updating `assignedName` in DB, publish label to hub if hub-paired**

When the admin renames a device (via the existing rename/update endpoint):

```kotlin
if (device.hubSlot != null) {
    val hub = deviceRepository.findActiveHubByStore(device.storeId!!)
    if (hub?.publicId != null) {
        val publisher = publisherProvider.ifAvailable
        publisher?.publishLabel(
            hub.publicId,
            LabelEnvelope(
                schemaVersion = 1,
                slot = device.hubSlot!!.toInt(),
                label = newName
            )
        )
    }
}
```

- **Step 2: Add `LabelEnvelope` data class**

```kotlin
data class LabelEnvelope(
    @field:JsonProperty("schema_version")
    val schemaVersion: Int = 1,
    val slot: Int,
    val label: String
)
```

---

### Task 12: Add unpair functionality

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/controller/DeviceAdminController.kt`
- Modify or create: unpair service logic
- **Step 1: Add unpair endpoint or extend delete endpoint**

When a hub-paired device is deleted from the admin dashboard, publish an unpair command to the hub:

```kotlin
if (device.hubSlot != null) {
    val hub = deviceRepository.findActiveHubByStore(device.storeId!!)
    if (hub?.publicId != null) {
        val publisher = publisherProvider.ifAvailable
        publisher?.publishUnpair(
            hub.publicId,
            UnpairEnvelope(schemaVersion = 1, slot = device.hubSlot!!.toInt())
        )
    }
}
```

- **Step 2: Add `UnpairEnvelope` data class**

```kotlin
data class UnpairEnvelope(
    @field:JsonProperty("schema_version")
    val schemaVersion: Int = 1,
    val slot: Int
)
```

---

### Task 13: Add `findHubForStore` repository method

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/repository/DeviceRepository.kt`
- **Step 1: Add query method**

```kotlin
@Query("""
    SELECT * FROM device
    WHERE store_id = :storeId
      AND kind = 'TRANSMITTER_HUB'
      AND status = 'ACTIVE'
    LIMIT 1
""")
suspend fun findActiveHubByStore(storeId: UUID): Device?
```

---

### Task 14: Masked RF collision check for 433M codes

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/repository/DeviceRfCodeRepository.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/service/RfCodeService.kt` (calls `findCollision` at line 274)

**Context:** The current 433M collision query compares decrypted payloads with exact byte equality (`pgp_sym_decrypt_bytea(...) = :plaintext`). But receivers match on masked low bits — a 32-bit code `0xABCD1234` and a 16-bit code `0x1234` have overlapping low bits and could cross-trigger within the same store. This task changes the collision check to compare the overlapping bit region.

**Spec:** `docs/spec/Pairing & Dispatch Robustness Fixes Spec.md` — §9

- **Step 1: Add `findAll433MDecryptedInStore` query**

Add a method that returns all 433M codes in a store with their decrypted payloads and bit widths:

```kotlin
suspend fun findAll433MDecryptedInStore(storeId: UUID): List<Decrypted433MRecord> =
    client.sql(
        """
        SELECT d.id, d.public_id, r.bits,
               pgp_sym_decrypt_bytea(r.payload, :encryptionKey) AS plaintext
        FROM device_rf_code r
        JOIN device d ON d.id = r.device_id
        WHERE d.kind IN ('RECEIVER_433M', 'RECEIVER_433M_PASSIVE')
          AND d.store_id = :storeId
          AND d.status IN ('PENDING_RF_CODE', 'ACTIVE', 'SUSPENDED')
        """
    )
        .bind("storeId", storeId)
        .bind("encryptionKey", encryptionKey)
        .map { row ->
            Decrypted433MRecord(
                deviceId = row.get("id", UUID::class.java)!!,
                publicId = row.get("public_id", String::class.java),
                bits = row.get("bits", Integer::class.java)!!.toInt(),
                plaintext = row.get("plaintext", ByteArray::class.java)!!
            )
        }
        .all()
        .collectList()
        .awaitSingle()

data class Decrypted433MRecord(
    val deviceId: UUID,
    val publicId: String?,
    val bits: Int,
    val plaintext: ByteArray
)
```

- **Step 2: Add masked collision check method**

In the service that calls collision checks, add a method that compares the overlapping low bits:

```kotlin
suspend fun findMaskedCollision(
    storeId: UUID,
    candidatePlaintext: ByteArray,
    candidateBits: Int,
    excludeDeviceId: UUID? = null
): DeviceCodeCollision? {
    val existing = deviceRfCodeRepository.findAll433MDecryptedInStore(storeId)
    return existing
        .filter { it.deviceId != excludeDeviceId }
        .firstOrNull { record ->
            val overlapBits = minOf(candidateBits, record.bits)
            val mask = if (overlapBits >= 32) 0xFFFFFFFF.toInt()
                       else (1 shl overlapBits) - 1
            val candidateInt = candidatePlaintext.toBigEndianInt() and mask
            val existingInt = record.plaintext.toBigEndianInt() and mask
            candidateInt == existingInt
        }
        ?.let { DeviceCodeCollision(it.deviceId, it.publicId) }
}

private fun ByteArray.toBigEndianInt(): Int {
    var result = 0
    for (b in this) {
        result = (result shl 8) or (b.toInt() and 0xFF)
    }
    return result
}
```

- **Step 3: Replace the 433M collision call site in `RfCodeService`**

In `RfCodeService.kt` at line 274, replace the single `deviceRfCodeRepository.findCollision(device.kind, device.storeId, plaintext)` call with a branch: for 433M bands use `findMaskedCollision(storeId, plaintext, bits, excludeDeviceId)`, for 2.4G keep the existing `deviceRfCodeRepository.findCollision(device.kind, null, plaintext)` (exact byte match, since all 2.4G codes have the same 40-bit width).

---

### Task 15: Update `DeviceDto` to include `hubSlot`

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/dto/DeviceDto.kt`
- **Step 1: Add `hubSlot` field**

```kotlin
val hubSlot: Short? = null,
```

- **Step 2: Update the mapper in `DeviceQueryService`**

Wherever `DeviceDto` is constructed from a database row, add:

```kotlin
hubSlot = row.get("hub_slot", java.lang.Short::class.java)?.toShort(),
```

---

### Task 16: Frontend — Hub-paired device indicator

**Files:**

- Modify: `web/src/app/[locale]/dashboard/devices/[id]/page.tsx` (or device list component)
- Modify: `web/src/types/device.ts`
- **Step 1: Add `hubSlot` to `DeviceDto` TypeScript type**

```typescript
export interface DeviceDto {
  // ... existing fields
  hubSlot: number | null;
}
```

- **Step 2: Add hub-paired badge in device list/detail**

Where device cards are rendered, add a visual indicator:

```tsx
{device.hubSlot != null && (
  <span className="text-xs text-muted-foreground">
    {tDevices("hubPaired")} · Slot {device.hubSlot}
  </span>
)}
```

- **Step 3: Disable RF code editing for hub-paired devices**

In the device detail page, conditionally hide or disable the RF code rotation/edit controls when `device.hubSlot != null`.

- **Step 4: Add i18n keys**

`en.json`:

```json
{
  "devices": {
    "hubPaired": "Hub-paired"
  }
}
```

`vi.json`:

```json
{
  "devices": {
    "hubPaired": "Ghép cục bộ"
  }
}
```

---

### Task 17: Frontend — Label push on rename

**Files:**

- The existing device rename flow should work without frontend changes — the backend handles the MQTT label push after updating the DB (Task 11). No frontend change needed unless the rename UI doesn't exist for all device types.

---

### Task 18: Subscribe hub to roster/update topic

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/listener/TransmitterOperationalListener.kt`

Or use the new `RosterSyncListener` from Task 6 (which handles its own subscription).

- **Step 1: Verify `RosterSyncListener` subscribes independently**

The `RosterSyncListener` from Task 6 registers its own handler with `mqttClientManager` and subscribes to the `roster/update` wildcard topic. No changes needed to `TransmitterOperationalListener`.

---

### Task 19: Build & lint verification

- **Step 1: Run backend build**

```bash
cd backend && ./gradlew build
```

- **Step 2: Run frontend lint and build**

```bash
cd web && yarn lint && yarn build
```

---

### Task 20: Update CHANGELOGS.md

- **Step 1: Add changelog entry**

Add entry to `docs/CHANGELOGS.md` documenting:

- Schema: added `hub_slot` and `last_roster_seq` columns to `device` table
- Backend: `RosterSyncListener` processes hub roster updates with signed ACK
- Backend: `TransmitterDispatchService` supports slot-based dispatch for hub-paired receivers
- Backend: `DeviceDispatchService.isDispatchableDevice()` updated to accept hub-paired devices without RF code rows
- Backend: Label push on device rename for hub-paired devices
- Backend: Unpair command published when hub-paired device is deleted
- Backend: New canonicals: `roster-ack-v1`, `dispatch-v1` (slot variant)
- Backend: Dispatch tracking Redis key enables ACK rejection reaction (busy key release + SSE event)
- Backend: `TransmitterOperationalListener` reacts to hub transmit rejections with `DEVICE_DISPATCH_FAILED`
- Backend: 433M RF collision check uses masked bit comparison instead of exact byte equality
- Frontend: Hub-paired badge on device list, RF code editing disabled for hub-paired devices
- Frontend: Bilingual i18n keys added
