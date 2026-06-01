# Remove `hardware_model` from Device Domain

Remove the `hardware_model` column, `device_hardware_model` enum type,
and all references across the backend and frontend.

**Date:** 2026-06-01

---

## 1. Problem

The `hardware_model` column on the `device` table is a denormalized
derivative of `kind`. After the local pairing migration, hub-paired
receivers are inserted by `RosterSyncListener` which *guesses* the
hardware model from the RF band — mapping `433M` → `PT2272` and
`2_4G` → `ESP32_C3`. Both mappings are incorrect: hub-paired 433M
receivers are ESP8266 or ESP32-C3 running ESP-NOW firmware, not PT2272
chips.

The column is `NOT NULL`, so every code path that inserts a device
must supply a value — even when the correct value is unknown.

## 2. Why `kind` is sufficient

| `kind` | Hardware (always) | Notes |
|---|---|---|
| `RECEIVER_433M_PASSIVE` | PT2272 | Only passive devices use this chip |
| `RECEIVER_433M` | ESP8266 or ESP32-C3 | Hub-paired; `hub_slot IS NOT NULL` |
| `RECEIVER_2_4G` | ESP32-C3 | Hub-paired; `hub_slot IS NOT NULL` |
| `TRANSMITTER_HUB` | ESP32-C3 | Always |

No business logic or dispatch path depends on `hardware_model`. The
frontend displays it in the device list table, device detail page, and
offers a hardware filter dropdown — all of which can be removed since
`kind` already conveys the same operational information more
accurately.

## 3. Changes

### 3.1 Schema

**`schema.sql`** — Remove `hardware_model device_hardware_model NOT NULL`
from `CREATE TABLE device`. Remove the `CREATE TYPE device_hardware_model`
enum declaration.

**`V2__add_hub_slot.sql`** — Append:

```sql
ALTER TABLE device DROP COLUMN hardware_model;
DROP TYPE device_hardware_model;
```

### 3.2 Backend — Delete

- `DeviceHardwareModel.kt` — Enum class deleted entirely.

### 3.3 Backend — Modify

| File | Change |
|---|---|
| `Device.kt` | Remove `hardwareModel` field, update `equals`/`hashCode` |
| `DeviceDto.kt` | Remove `hardwareModel` field and import |
| `DeviceDetailDto.kt` | Remove `hardwareModel` field and import |
| `R2DBCConfig.kt` | Remove `DeviceHardwareModelWriteConverter` object, remove from `getCustomConverters()` list, remove from `withEnum()` registration, remove import |
| `DeviceQueryService.kt` | Remove `d.hardware_model` from both SQL queries, remove `hardwareModel` from `mapRow()` |
| `DeviceQueryService.kt` (`toDetail`) | Remove `hardwareModel` from `DeviceDetailDto` construction |
| `RosterSyncListener.kt` | Remove `bandToHardwareModel()`, remove `hardwareModel` parameter from `upsertRosterReceiver()`, remove `hardware_model` from INSERT and UPDATE SQL, remove import |
| `DeviceRegistrationService.kt` | Remove `isLegalHardwarePair()` method. Remove `hardwareModel` from `ParsedRegistration` data class (and its `equals`/`hashCode`). Remove `hardwareModel` parsing and validation from `parseReceiverRegistration()` and `parseTransmitterRegistration()`. Remove `hardwareModel` from the `Device()` constructor call in `handleRegistration()`. Keep `hardware_model` in the JSON wire-format classes (`ReceiverBootstrapRegistration`, `TransmitterBootstrapRegistration`) so deserialization doesn't break — the field is simply ignored after parsing. |
| `PassiveDeviceRegistrationService.kt` | Remove `hardwareModel` check (`request.hardwareModel != DeviceHardwareModel.PT2272`). Remove `hardwareModel` from `Device()` constructor call. |
| `PassiveDeviceRegistrationRequest.kt` | Remove `hardwareModel` field and import. |

### 3.4 Frontend — Modify

| File | Change |
|---|---|
| `types/device.ts` | Remove `DeviceHardwareModel` type. Remove `hardwareModel` field from `DeviceDto`. Remove `hardwareModel` field from `PassiveDeviceRegistrationRequest`. |
| `devices/page.tsx` | Remove `hardwareFilter` state, remove hardware filter dropdown from UI, remove hardware filter logic from `filteredDevices`. |
| `devices/[id]/page.tsx` | Remove hardware model display row. |
| `device-list-table.tsx` | Remove hardware model column (`<td>` and `<th>`). |
| `passive-device-form-dialog.tsx` | Remove `hardwareModel: "PT2272"` from form submission payload. |

### 3.5 Not changed

- **MQTT wire-format** (`ReceiverBootstrapRegistration`,
  `TransmitterBootstrapRegistration`): Keep `hardware_model` in the
  JSON classes. The hub firmware still sends it in registration
  payloads. The backend deserializes but ignores the value. This avoids
  requiring a firmware update.
- **`device_rf_code_kind_check()` trigger**: References `device_kind`,
  not `device_hardware_model`. No change.

## 4. Migration safety

The `hardware_model` column has no foreign key references, no indexes,
and is not used in any trigger. `ALTER TABLE device DROP COLUMN` is a
metadata-only operation in PostgreSQL — it marks the column as dropped
without rewriting the table. The `DROP TYPE` succeeds because no
remaining column references the type.

No data migration is needed. The column values are discarded.
