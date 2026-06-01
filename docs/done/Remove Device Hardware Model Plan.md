# Remove `hardware_model` from Device Domain — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove the `hardware_model` column, `device_hardware_model` enum type, and all backend/frontend references — the column is a denormalized derivative of `kind` that RosterSyncListener currently fills with incorrect values.

**Architecture:** Pure deletion across the stack: schema → entity → DTOs → services → frontend. No new code is introduced. MQTT wire-format classes retain the `hardware_model` field so hub firmware does not need a corresponding update.

**Tech Stack:** Kotlin 2.3 / Spring Boot 3.5 / R2DBC / Next.js 16 / React 19 / TypeScript 5

**Spec:** `docs/spec/Remove Device Hardware Model Spec.md`

---

## Existing Codebase Context

### Key files

| File | Role |
|---|---|
| `backend/.../db/schema.sql` | Master schema with `device_hardware_model` enum and `device` table |
| `backend/.../db/migration/V2__add_hub_slot.sql` | Existing migration file (add `hub_slot` + `last_roster_seq`) |
| `backend/.../types/DeviceHardwareModel.kt` | Enum: `ESP_01`, `ESP32_C3`, `PT2272` |
| `backend/.../entity/Device.kt` | Entity with `hardwareModel` field |
| `backend/.../dto/DeviceDto.kt` | List/dispatch DTO |
| `backend/.../dto/DeviceDetailDto.kt` | Detail page DTO |
| `backend/.../service/DeviceQueryService.kt` | SQL queries + `mapRow()` mapper for both DTOs |
| `backend/.../service/DeviceRegistrationService.kt` | MQTT registration with `isLegalHardwarePair()` + `ParsedRegistration` |
| `backend/.../service/PassiveDeviceRegistrationService.kt` | Admin dashboard passive device registration |
| `backend/.../request/PassiveDeviceRegistrationRequest.kt` | Request DTO for passive registration |
| `backend/.../listener/RosterSyncListener.kt` | Roster sync with `bandToHardwareModel()` |
| `backend/.../core/database/R2DBCConfig.kt` | R2DBC enum converter registration |
| `web/src/types/device.ts` | Frontend TypeScript types |
| `web/src/features/device/device-filter-bar.tsx` | Hardware filter dropdown component |
| `web/src/features/device/device-list-table.tsx` | Device list table with hardware column |
| `web/src/features/device/passive-device-form-dialog.tsx` | Passive device form |
| `web/src/app/[locale]/dashboard/devices/page.tsx` | Devices page with filter state |
| `web/src/app/[locale]/dashboard/devices/[id]/page.tsx` | Device detail page |
| `web/src/messages/en.json` | English i18n keys |
| `web/src/messages/vi.json` | Vietnamese i18n keys |

---

### Task 1: Schema migration

**Files:**

- Modify: `backend/src/main/resources/db/migration/V2__add_hub_slot.sql`
- Modify: `backend/src/main/resources/db/schema.sql`

- [ ] **Step 1: Append DROP statements to migration file**

Add to `backend/src/main/resources/db/migration/V2__add_hub_slot.sql`:

```sql
ALTER TABLE device DROP COLUMN IF EXISTS hardware_model;
DROP TYPE IF EXISTS device_hardware_model;
```

The file should now read:

```sql
ALTER TABLE device ADD COLUMN IF NOT EXISTS hub_slot SMALLINT;
ALTER TABLE device ADD COLUMN IF NOT EXISTS last_roster_seq INT;
ALTER TABLE device DROP COLUMN IF EXISTS hardware_model;
DROP TYPE IF EXISTS device_hardware_model;
```

- [ ] **Step 2: Remove `device_hardware_model` enum from schema.sql**

In `backend/src/main/resources/db/schema.sql`, delete this block:

```sql
CREATE TYPE device_hardware_model AS ENUM (
    'ESP_01',
    'ESP32_C3',
    'PT2272'
);
```

- [ ] **Step 3: Remove `hardware_model` column from `CREATE TABLE device`**

In the same file, remove this line from the `device` table definition:

```sql
    hardware_model device_hardware_model NOT NULL,
```

---

### Task 2: Delete DeviceHardwareModel enum

**Files:**

- Delete: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/types/DeviceHardwareModel.kt`

- [ ] **Step 1: Delete the file**

Delete `backend/src/main/kotlin/com/thomas/notiguide/domain/device/types/DeviceHardwareModel.kt` entirely.

---

### Task 3: Update Device entity

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/entity/Device.kt`

- [ ] **Step 1: Remove `hardwareModel` field from constructor**

Remove the import:

```kotlin
import com.thomas.notiguide.domain.device.types.DeviceHardwareModel
```

Remove this field from the `Device` data class constructor:

```kotlin
    @Column("hardware_model")
    val hardwareModel: DeviceHardwareModel,
```

- [ ] **Step 2: Remove from `equals()`**

Remove this line from the `equals()` method:

```kotlin
        if (hardwareModel != other.hardwareModel) return false
```

- [ ] **Step 3: Remove from `hashCode()`**

Remove this line from the `hashCode()` method:

```kotlin
        result = 31 * result + hardwareModel.hashCode()
```

---

### Task 4: Update DTOs

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/dto/DeviceDto.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/dto/DeviceDetailDto.kt`

- [ ] **Step 1: Update `DeviceDto`**

Remove the import:

```kotlin
import com.thomas.notiguide.domain.device.types.DeviceHardwareModel
```

Remove this field:

```kotlin
    val hardwareModel: DeviceHardwareModel,
```

- [ ] **Step 2: Update `DeviceDetailDto`**

Remove the import:

```kotlin
import com.thomas.notiguide.domain.device.types.DeviceHardwareModel
```

Remove this field:

```kotlin
    val hardwareModel: DeviceHardwareModel,
```

---

### Task 5: Update R2DBCConfig

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/core/database/R2DBCConfig.kt`

- [ ] **Step 1: Remove enum codec registration**

Remove the import:

```kotlin
import com.thomas.notiguide.domain.device.types.DeviceHardwareModel
```

Remove this line from the `codecRegistrar` builder chain:

```kotlin
                    .withEnum("device_hardware_model", DeviceHardwareModel::class.java)
```

- [ ] **Step 2: Remove write converter from `getCustomConverters()` list**

Remove `DeviceHardwareModelWriteConverter` from the list:

```kotlin
        DeviceHardwareModelWriteConverter,
```

- [ ] **Step 3: Delete the converter object**

Remove this declaration:

```kotlin
    @WritingConverter
    object DeviceHardwareModelWriteConverter : EnumWriteSupport<DeviceHardwareModel>()
```

---

### Task 6: Update DeviceQueryService

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/service/DeviceQueryService.kt`

- [ ] **Step 1: Remove `d.hardware_model` from `listDevices()` SQL**

In the `listDevices()` method, remove this line from the SELECT clause:

```sql
                d.hardware_model,
```

- [ ] **Step 2: Remove `d.hardware_model` from `findDeviceById()` SQL**

In the `findDeviceById()` method, remove this line from the SELECT clause:

```sql
                d.hardware_model,
```

- [ ] **Step 3: Remove `hardwareModel` from `mapRow()`**

In the `mapRow()` function, remove this line from the `DeviceDto` constructor:

```kotlin
            hardwareModel = row.get("hardware_model", DeviceHardwareModel::class.java)!!,
```

Remove the import:

```kotlin
import com.thomas.notiguide.domain.device.types.DeviceHardwareModel
```

- [ ] **Step 4: Remove `hardwareModel` from `toDetail()` extension**

In the `DeviceDto.toDetail()` extension function, remove this line from the `DeviceDetailDto` constructor:

```kotlin
        hardwareModel = hardwareModel,
```

---

### Task 7: Update RosterSyncListener

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/listener/RosterSyncListener.kt`

- [ ] **Step 1: Remove `bandToHardwareModel()` method**

Delete this method entirely:

```kotlin
    private fun bandToHardwareModel(band: String): DeviceHardwareModel? = when (band) {
        "433M" -> DeviceHardwareModel.PT2272
        "2_4G" -> DeviceHardwareModel.ESP32_C3
        else -> null
    }
```

Remove the import:

```kotlin
import com.thomas.notiguide.domain.device.types.DeviceHardwareModel
```

- [ ] **Step 2: Remove `hardwareModel` from `handleRosterUpdate()` loop**

In the `handleRosterUpdate()` method, simplify the receiver processing loop. Change:

```kotlin
        for (receiver in roster.receivers) {
            val slot = receiver.slot.toShort()
            val kind = bandToKind(receiver.band)
            val hardwareModel = bandToHardwareModel(receiver.band)

            if (kind == null || hardwareModel == null) {
                log.warn("Skipping roster receiver with unknown band={} slot={} for hub {}", receiver.band, receiver.slot, publicId)
                continue
            }

            upsertRosterReceiver(
                storeId = hub.storeId,
                hubSlot = slot,
                kind = kind,
                hardwareModel = hardwareModel,
                label = receiver.label
            )
            activeSlots.add(slot)
        }
```

To:

```kotlin
        for (receiver in roster.receivers) {
            val slot = receiver.slot.toShort()
            val kind = bandToKind(receiver.band)

            if (kind == null) {
                log.warn("Skipping roster receiver with unknown band={} slot={} for hub {}", receiver.band, receiver.slot, publicId)
                continue
            }

            upsertRosterReceiver(
                storeId = hub.storeId,
                hubSlot = slot,
                kind = kind,
                label = receiver.label
            )
            activeSlots.add(slot)
        }
```

- [ ] **Step 3: Remove `hardwareModel` from `upsertRosterReceiver()`**

Change the method signature from:

```kotlin
    private suspend fun upsertRosterReceiver(
        storeId: UUID?,
        hubSlot: Short,
        kind: DeviceKind,
        hardwareModel: DeviceHardwareModel,
        label: String?
    ) {
```

To:

```kotlin
    private suspend fun upsertRosterReceiver(
        storeId: UUID?,
        hubSlot: Short,
        kind: DeviceKind,
        label: String?
    ) {
```

- [ ] **Step 4: Remove `hardware_model` from UPDATE SQL**

Change the UPDATE statement from:

```kotlin
            client.sql(
                """
                UPDATE device
                SET kind = :kind::device_kind,
                    hardware_model = :hardwareModel::device_hardware_model,
                    status = 'ACTIVE'::device_status,
                    assigned_name = :label,
                    updated_at = CURRENT_TIMESTAMP
                WHERE id = :id
                """
            )
                .bind("id", existing)
                .bind("kind", kind.name)
                .bind("hardwareModel", hardwareModel.name)
                .bind("label", label ?: "Slot $hubSlot")
```

To:

```kotlin
            client.sql(
                """
                UPDATE device
                SET kind = :kind::device_kind,
                    status = 'ACTIVE'::device_status,
                    assigned_name = :label,
                    updated_at = CURRENT_TIMESTAMP
                WHERE id = :id
                """
            )
                .bind("id", existing)
                .bind("kind", kind.name)
                .bind("label", label ?: "Slot $hubSlot")
```

- [ ] **Step 5: Remove `hardware_model` from INSERT SQL**

Change the INSERT statement from:

```kotlin
            client.sql(
                """
                INSERT INTO device (hub_slot, kind, hardware_model, status, assigned_name, store_id)
                VALUES (:hubSlot, :kind::device_kind, :hardwareModel::device_hardware_model, 'ACTIVE'::device_status, :label, :storeId)
                """
            )
                .bind("hubSlot", hubSlot)
                .bind("kind", kind.name)
                .bind("hardwareModel", hardwareModel.name)
                .bind("label", label ?: "Slot $hubSlot")
                .bind("storeId", storeId)
```

To:

```kotlin
            client.sql(
                """
                INSERT INTO device (hub_slot, kind, status, assigned_name, store_id)
                VALUES (:hubSlot, :kind::device_kind, 'ACTIVE'::device_status, :label, :storeId)
                """
            )
                .bind("hubSlot", hubSlot)
                .bind("kind", kind.name)
                .bind("label", label ?: "Slot $hubSlot")
                .bind("storeId", storeId)
```

---

### Task 8: Update DeviceRegistrationService

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/service/DeviceRegistrationService.kt`

- [ ] **Step 1: Remove import**

Remove:

```kotlin
import com.thomas.notiguide.domain.device.types.DeviceHardwareModel
```

- [ ] **Step 2: Remove `hardwareModel` from Device constructor in `handleRegistration()`**

In the `handleRegistration()` method (~line 82-99), remove this line from the `Device()` constructor call:

```kotlin
                hardwareModel = registration.hardwareModel,
```

- [ ] **Step 3: Simplify `parseReceiverRegistration()` validation**

Replace the hardwareModel parsing + validation block. Change:

```kotlin
        val hardwareModel = runCatching { DeviceHardwareModel.fromWireValue(request.hardwareModel) }.getOrNull()
        val kind = runCatching { DeviceKind.valueOf(request.receiverType) }.getOrNull()
        if (hardwareModel == null || kind == null || kind == DeviceKind.TRANSMITTER_HUB || kind == DeviceKind.RECEIVER_433M_PASSIVE) {
            deviceMqttPublisher.publishRejected(DeviceFamily.RECEIVER, challengeId, "model_radio_mismatch")
            return null
        }
        if (!isLegalHardwarePair(hardwareModel, kind)) {
            deviceMqttPublisher.publishRejected(DeviceFamily.RECEIVER, challengeId, "model_radio_mismatch")
            return null
        }
```

To:

```kotlin
        val kind = runCatching { DeviceKind.valueOf(request.receiverType) }.getOrNull()
        if (kind == null || kind == DeviceKind.TRANSMITTER_HUB || kind == DeviceKind.RECEIVER_433M_PASSIVE) {
            deviceMqttPublisher.publishRejected(DeviceFamily.RECEIVER, challengeId, "model_radio_mismatch")
            return null
        }
```

And remove `hardwareModel` from the returned `ParsedRegistration`:

```kotlin
        return ParsedRegistration(
            kind = kind,
            firmwareVersion = request.firmwareVersion.trim(),
            publicKeyDer = publicKeyDer,
            enrollmentToken = request.enrollmentToken,
            registrationNonce = request.registrationNonce
        )
```

- [ ] **Step 4: Simplify `parseTransmitterRegistration()` validation**

Replace the hardwareModel parsing + validation block. Change:

```kotlin
        val hardwareModel = runCatching { DeviceHardwareModel.fromWireValue(request.hardwareModel) }.getOrNull()
        val kind = runCatching { DeviceKind.valueOf(request.kind) }.getOrNull()
        if (hardwareModel == null || kind != DeviceKind.TRANSMITTER_HUB || !isLegalHardwarePair(hardwareModel, DeviceKind.TRANSMITTER_HUB)) {
            deviceMqttPublisher.publishRejected(DeviceFamily.TRANSMITTER, challengeId, "model_radio_mismatch")
            return null
        }
```

To:

```kotlin
        val kind = runCatching { DeviceKind.valueOf(request.kind) }.getOrNull()
        if (kind != DeviceKind.TRANSMITTER_HUB) {
            deviceMqttPublisher.publishRejected(DeviceFamily.TRANSMITTER, challengeId, "model_radio_mismatch")
            return null
        }
```

And remove `hardwareModel` from the returned `ParsedRegistration`:

```kotlin
        return ParsedRegistration(
            kind = DeviceKind.TRANSMITTER_HUB,
            firmwareVersion = request.firmwareVersion.trim(),
            publicKeyDer = publicKeyDer,
            enrollmentToken = request.enrollmentToken,
            registrationNonce = request.registrationNonce
        )
```

- [ ] **Step 5: Delete `isLegalHardwarePair()` method**

Delete this entire method:

```kotlin
    private fun isLegalHardwarePair(
        hardwareModel: DeviceHardwareModel,
        kind: DeviceKind
    ): Boolean = when (hardwareModel) {
        DeviceHardwareModel.ESP_01 -> kind == DeviceKind.RECEIVER_433M
        DeviceHardwareModel.ESP32_C3 -> kind in setOf(
            DeviceKind.RECEIVER_433M,
            DeviceKind.RECEIVER_2_4G,
            DeviceKind.TRANSMITTER_HUB
        )
        DeviceHardwareModel.PT2272 -> kind == DeviceKind.RECEIVER_433M_PASSIVE
    }
```

- [ ] **Step 6: Update `ParsedRegistration` data class**

Remove `hardwareModel` from the data class. Change:

```kotlin
private data class ParsedRegistration(
    val hardwareModel: DeviceHardwareModel,
    val kind: DeviceKind,
    val firmwareVersion: String,
    val publicKeyDer: ByteArray,
    val enrollmentToken: String,
    val registrationNonce: String
) {
    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        if (javaClass != other?.javaClass) return false

        other as ParsedRegistration

        if (hardwareModel != other.hardwareModel) return false
        if (kind != other.kind) return false
        if (firmwareVersion != other.firmwareVersion) return false
        if (!publicKeyDer.contentEquals(other.publicKeyDer)) return false
        if (enrollmentToken != other.enrollmentToken) return false
        if (registrationNonce != other.registrationNonce) return false

        return true
    }

    override fun hashCode(): Int {
        var result = hardwareModel.hashCode()
        result = 31 * result + kind.hashCode()
        result = 31 * result + firmwareVersion.hashCode()
        result = 31 * result + publicKeyDer.contentHashCode()
        result = 31 * result + enrollmentToken.hashCode()
        result = 31 * result + registrationNonce.hashCode()
        return result
    }
}
```

To:

```kotlin
private data class ParsedRegistration(
    val kind: DeviceKind,
    val firmwareVersion: String,
    val publicKeyDer: ByteArray,
    val enrollmentToken: String,
    val registrationNonce: String
) {
    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        if (javaClass != other?.javaClass) return false

        other as ParsedRegistration

        if (kind != other.kind) return false
        if (firmwareVersion != other.firmwareVersion) return false
        if (!publicKeyDer.contentEquals(other.publicKeyDer)) return false
        if (enrollmentToken != other.enrollmentToken) return false
        if (registrationNonce != other.registrationNonce) return false

        return true
    }

    override fun hashCode(): Int {
        var result = kind.hashCode()
        result = 31 * result + firmwareVersion.hashCode()
        result = 31 * result + publicKeyDer.contentHashCode()
        result = 31 * result + enrollmentToken.hashCode()
        result = 31 * result + registrationNonce.hashCode()
        return result
    }
}
```

**Note:** Keep `hardware_model` in `ReceiverBootstrapRegistration` and `TransmitterBootstrapRegistration` wire-format classes — the hub firmware still sends this field. It is deserialized but ignored.

---

### Task 9: Update PassiveDeviceRegistrationService and request

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/request/PassiveDeviceRegistrationRequest.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/service/PassiveDeviceRegistrationService.kt`

- [ ] **Step 1: Remove `hardwareModel` from request DTO**

In `PassiveDeviceRegistrationRequest.kt`, remove the imports:

```kotlin
import com.thomas.notiguide.domain.device.types.DeviceHardwareModel
```

And remove this field:

```kotlin
    @field:NotNull
    val hardwareModel: DeviceHardwareModel,
```

- [ ] **Step 2: Simplify validation in `PassiveDeviceRegistrationService.register()`**

Change the combined check:

```kotlin
        if (request.hardwareModel != DeviceHardwareModel.PT2272 || request.kind != DeviceKind.RECEIVER_433M_PASSIVE) {
            throw IllegalArgumentException("Passive device registration only supports the (PT2272, RECEIVER_433M_PASSIVE) pairing")
        }
```

To:

```kotlin
        if (request.kind != DeviceKind.RECEIVER_433M_PASSIVE) {
            throw IllegalArgumentException("Passive device registration only supports RECEIVER_433M_PASSIVE")
        }
```

And remove the import if present:

```kotlin
import com.thomas.notiguide.domain.device.types.DeviceHardwareModel
```

- [ ] **Step 3: Remove `hardwareModel` from Device constructor call**

In `PassiveDeviceRegistrationService.register()`, remove this line from the `Device()` constructor call:

```kotlin
                hardwareModel = DeviceHardwareModel.PT2272,
```

---

### Task 10: Update frontend types

**Files:**

- Modify: `web/src/types/device.ts`

- [ ] **Step 1: Remove `DeviceHardwareModel` type**

Delete this line:

```typescript
export type DeviceHardwareModel = "ESP-01" | "ESP32-C3" | "PT2272";
```

- [ ] **Step 2: Remove `hardwareModel` from `DeviceDto`**

Remove this field from the `DeviceDto` interface:

```typescript
  hardwareModel: DeviceHardwareModel;
```

- [ ] **Step 3: Remove `hardwareModel` from `PassiveDeviceRegistrationRequest`**

Remove this field from the `PassiveDeviceRegistrationRequest` interface:

```typescript
  hardwareModel: "PT2272";
```

---

### Task 11: Update DeviceFilterBar

**Files:**

- Modify: `web/src/features/device/device-filter-bar.tsx`

- [ ] **Step 1: Remove hardware filter props and implementation**

From the `DeviceFilterBarProps` interface, remove:

```typescript
  hardwareFilter: string;
  onHardwareChange: (v: string) => void;
```

From the destructured props in the function signature, remove `hardwareFilter` and `onHardwareChange`.

Delete the `HARDWARE_OPTIONS` constant:

```typescript
const HARDWARE_OPTIONS = ["all", "ESP-01", "ESP32-C3", "PT2272"] as const;
```

Delete the `getHardwareLabel` function:

```typescript
  function getHardwareLabel(value: string) {
    if (value === "all") return tDevices("filterHardware");
    return value;
  }
```

Delete the entire hardware `<Select>` block (the third `<Select>` in the JSX):

```tsx
      <Select
        value={hardwareFilter}
        onValueChange={(v) => v && onHardwareChange(v)}
      >
        <SelectTrigger className="h-9 w-full gap-2 px-3 text-sm s:w-36">
          <span className="truncate">{getHardwareLabel(hardwareFilter)}</span>
        </SelectTrigger>
        <SelectContent
          align="start"
          alignItemWithTrigger={false}
          className="p-1.5"
        >
          {HARDWARE_OPTIONS.map((opt) => (
            <SelectItem key={opt} value={opt} className="py-2">
              {getHardwareLabel(opt)}
            </SelectItem>
          ))}
        </SelectContent>
      </Select>
```

---

### Task 12: Update devices page

**Files:**

- Modify: `web/src/app/[locale]/dashboard/devices/page.tsx`

- [ ] **Step 1: Remove hardware filter state**

Delete this line:

```typescript
  const [hardwareFilter, setHardwareFilter] = useState("all");
```

- [ ] **Step 2: Remove hardware filter from `filteredDevices`**

Change:

```typescript
  const filteredDevices = devices?.filter((d) => {
    if (statusFilter !== "all" && d.status !== statusFilter) return false;
    return !(hardwareFilter !== "all" && d.hardwareModel !== hardwareFilter);
  });
```

To:

```typescript
  const filteredDevices = devices?.filter((d) => {
    if (statusFilter !== "all" && d.status !== statusFilter) return false;
    return true;
  });
```

- [ ] **Step 3: Remove hardware props from `<DeviceFilterBar>`**

Remove these two props from the `<DeviceFilterBar>` component:

```tsx
        hardwareFilter={hardwareFilter}
        onHardwareChange={setHardwareFilter}
```

---

### Task 13: Update device list table

**Files:**

- Modify: `web/src/features/device/device-list-table.tsx`

- [ ] **Step 1: Remove hardware column header**

Delete this `<th>` block:

```tsx
              <th
                scope="col"
                className="hidden px-4 py-3 font-medium text-muted-foreground s:table-cell"
              >
                {tDevices("columnHardware")}
              </th>
```

- [ ] **Step 2: Remove hardware skeleton cell**

In the skeleton loading rows, delete this `<td>`:

```tsx
                  <td className="hidden px-4 py-3 s:table-cell">
                    <Skeleton className="h-4 w-20" />
                  </td>
```

- [ ] **Step 3: Remove hardware data cell**

In the device rows, delete this `<td>`:

```tsx
                <td className="hidden px-4 py-3 text-muted-foreground s:table-cell">
                  {device.hardwareModel}
                </td>
```

- [ ] **Step 4: Update `colSpan` on empty state**

Change the colSpan from 6 to 5:

```tsx
                  colSpan={5}
```

---

### Task 14: Update device detail page

**Files:**

- Modify: `web/src/app/[locale]/dashboard/devices/[id]/page.tsx`

- [ ] **Step 1: Remove hardware model display block**

Delete this block from the device info section:

```tsx
          <div>
            <p className="text-xs text-muted-foreground">
              {tDevices("detail.hardware")}
            </p>
            <p className="text-sm">{device.hardwareModel}</p>
          </div>
```

---

### Task 15: Update passive device form

**Files:**

- Modify: `web/src/features/device/passive-device-form-dialog.tsx`

- [ ] **Step 1: Remove `hardwareModel` from form submission**

In the `handleSubmit` function, remove `hardwareModel: "PT2272"` from the `registerPassiveDevice()` call. Change:

```typescript
      const result = await registerPassiveDevice({
        hardwareModel: "PT2272",
        kind: "RECEIVER_433M_PASSIVE",
        assignedName: assignedName.trim(),
        storeId,
        rfCodeHex: rfCodeHex.toUpperCase(),
        rfCodeBits: 16,
      });
```

To:

```typescript
      const result = await registerPassiveDevice({
        kind: "RECEIVER_433M_PASSIVE",
        assignedName: assignedName.trim(),
        storeId,
        rfCodeHex: rfCodeHex.toUpperCase(),
        rfCodeBits: 16,
      });
```

---

### Task 16: Remove i18n keys

**Files:**

- Modify: `web/src/messages/en.json`
- Modify: `web/src/messages/vi.json`

- [ ] **Step 1: Remove hardware keys from `en.json`**

Delete these keys from the `devices` section:

```json
    "columnHardware": "Hardware",
    "filterHardware": "All Hardware",
```

And delete this key from the `devices.detail` section:

```json
      "hardware": "Hardware",
```

- [ ] **Step 2: Remove hardware keys from `vi.json`**

Delete the matching keys:

```json
    "columnHardware": "Phần cứng",
    "filterHardware": "Tất cả phần cứng",
```

And from `devices.detail`:

```json
      "hardware": "Phần cứng",
```

---

### Task 17: Update CHANGELOGS.md

**Files:**

- Modify: `docs/CHANGELOGS.md`

- [ ] **Step 1: Add changelog entry**

Add entry documenting:

- Schema: dropped `hardware_model` column from `device` table, dropped `device_hardware_model` enum type
- Backend: deleted `DeviceHardwareModel.kt` enum class
- Backend: removed `hardwareModel` from `Device` entity, `DeviceDto`, `DeviceDetailDto`
- Backend: removed `isLegalHardwarePair()` from `DeviceRegistrationService`
- Backend: removed `hardwareModel` from `ParsedRegistration`, `PassiveDeviceRegistrationRequest`
- Backend: removed `bandToHardwareModel()` from `RosterSyncListener`, simplified upsert SQL
- Backend: removed `DeviceHardwareModelWriteConverter` from `R2DBCConfig`
- Backend: removed `hardware_model` from `DeviceQueryService` SQL queries and mapper
- Frontend: removed `DeviceHardwareModel` type and `hardwareModel` from `DeviceDto`
- Frontend: removed hardware filter from device list page and filter bar
- Frontend: removed hardware column from device list table
- Frontend: removed hardware display from device detail page
- Frontend: removed `hardwareModel` from passive device form submission
- Frontend: removed `columnHardware`, `filterHardware`, `detail.hardware` i18n keys
