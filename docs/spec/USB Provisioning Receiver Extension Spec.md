# USB Provisioning — Receiver Extension Spec

Extends the existing USB provisioning flow to support both receiver variants — ESP8266 (ESP-01 modules via USB-serial adapter) and ESP32-C3 (native USB) — alongside the existing ESP32-C3 transmitter support.

## Context

Both receiver variants currently provision via SoftAP + HTTP server, which has proven unreliable on the ESP8266. This spec adds serial provisioning using the same JSON protocol the transmitter already speaks. The ESP8266 receiver replaces SoftAP entirely with serial (SoftAP unreliable). The ESP32-C3 receiver adds serial as a concurrent provisioning path alongside SoftAP (kept as backup). The existing Web Serial frontend is extended to handle all device types through a single auto-detecting provisioning dialog.

The companion firmware spec is `docs/planned/UART Provisioning Design.md`. This document supersedes the protocol format defined there — the receiver firmware must implement the transmitter-compatible protocol described below, not the `>`-prefix protocol from the original UART spec.

### Hardware Matrix

| Device | Chip | USB Interface | USB VID:PID | Reboot Behavior |
|--------|------|---------------|-------------|-----------------|
| Transmitter | ESP32-C3 | Native USB Serial/JTAG | `0x303a:0x1001` | Port re-enumerates |
| Receiver (ESP32) | ESP32-C3 | Native USB Serial/JTAG | `0x303a:0x1001` | Port re-enumerates |
| Receiver (ESP8266) | ESP8266 | UART0 via USB-serial adapter | Adapter-dependent (currently `0x0403:0x6001` FTDI) | Port stays open |

The USB-serial adapter VID:PID for ESP8266 receivers may change if a different adapter is used. The filter list in `types.ts` should be treated as extensible.

## Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Protocol format | Transmitter's UUID-correlated JSON protocol | Reuses `serial-protocol.ts` as-is |
| Device detection | `identify` command response (not USB vendor ID) | Future-proof; firmware declares its own kind |
| WiFi test command | Yes, receiver implements `provision.test_wifi` | Same safety net as transmitter provisioning |
| Command naming | `factory_reset` (not `reset`), `identify` (not `identity`) | Unified naming across device families |
| Provisioning scope | Provisioning flow only | Control panel for receivers is a separate task |
| Backend changes | None | Enrollment token is already device-kind-agnostic |

## Receiver Serial Protocol (Firmware)

### Transport

115200 8N1, JSON lines terminated by `\n`.

- **ESP32-C3 receiver:** Native USB Serial/JTAG — same driver as the transmitter (`usb_serial_jtag`). No external adapter needed.
- **ESP8266 receiver (ESP-01):** UART0 (GPIO1 TX, GPIO3 RX) through a USB-serial adapter (currently FTDI FT232RL, `0x0403:0x6001`).

### Message Format

Identical to the transmitter protocol defined in `transmitter/main/serial/serial_protocol.c`:

**Request** (host → device):
```json
{"id":"<uuid>","type":"<command>","payload":{...}}
```

**Response** (device → host):
```json
{"id":"<uuid>","type":"response","ok":true,"payload":{...}}
```

**Error response:**
```json
{"id":"<uuid>","type":"response","ok":false,"error":"<reason>"}
```

Log lines (ESP_LOG output) are not prefixed — the host distinguishes them by JSON parse failure, same as the transmitter.

### Command Set

#### `ping`

Identical to transmitter.

**Response payload:** `{ "uptime_ms": <number> }`

#### `identify`

Same as transmitter, with the addition of `device_kind`.

**Response payload:**
```json
{
  "public_id": "",
  "device_name": "",
  "device_kind": "RECEIVER_433M",
  "op_state": "NONE",
  "firmware_version": "f8a2469",
  "mac": "AB:CD:EF:12:34:56"
}
```

`device_kind` is one of: `"RECEIVER_433M"`, `"RECEIVER_2_4G"`. The transmitter firmware also adds this field with value `"TRANSMITTER_HUB"`.

#### `status`

Not implemented on the receiver. The serial connection is provisioning-only — the receiver does not support status monitoring over serial. The `identify` command provides the device info needed during provisioning.

#### `provision`

Same flat payload as transmitter.

**Request payload:**
```json
{
  "wifi_ssid": "MyNetwork",
  "wifi_pwd": "MyPassword",
  "mqtt_uri": "mqtts://broker.example.com:8883",
  "mqtt_user": "device_user",
  "mqtt_pwd": "device_pass",
  "enroll_token": "enroll-token-abc123"
}
```

**Validation:** same as transmitter — `wifi_ssid`, `mqtt_uri`, `mqtt_user`, `mqtt_pwd`, `enroll_token` required; `wifi_pwd` optional; `mqtt_uri` must start with `mqtts://`.

**Success response:** `{ "restarting": true }`

Device writes to NVS, flushes UART, waits 250ms, reboots.

**Errors:** `missing_payload`, `missing_fields`, `nvs_write_failed`

#### `provision.test_wifi`

Identical to transmitter.

**Request payload:** `{ "wifi_ssid": "...", "wifi_pwd": "..." }` (`wifi_pwd` optional)

**Success response:** `{ "connected": true, "rssi": -55, "ip": "192.168.1.50" }`

**Failure response:** `{ "connected": false }`

**Errors:** `already_connected`, `missing_fields`, `wifi_start_failed`

Timeout: 15 seconds.

#### `factory_reset`

Identical to transmitter.

**Success response:** `{ "restarting": true }`

Erases `device_cfg` NVS namespace, flushes UART, reboots.

#### `retry` (receiver-only)

Re-attempts the boot sequence using existing NVS credentials. Only meaningful when the device has provisioning data but failed to connect.

**Request payload:** none

**Success response:** `{ "restarting": true }`

Device reboots.

**Error:** `not_provisioned` (if no NVS config exists)

### Session Management

**ESP8266 receiver:** Synchronous blocking in the main task. No background task, no session tracking. When provisioning is needed, the main task enters a blocking read loop. This saves ~4KB of RAM versus a background task — significant on the ESP8266's ~40KB usable heap. During normal operation, serial is not active.

**ESP32-C3 receiver:** Asynchronous background task, similar to the transmitter. The serial read loop runs in a dedicated FreeRTOS task alongside the existing SoftAP + HTTP provisioning. This allows either provisioning path (serial or WiFi) to succeed. Unlike the transmitter, the serial task is torn down once the device enters operational state — it is only needed during provisioning/recovery. API: `serial_protocol_init()` starts the task, `serial_protocol_stop()` deletes it.

## Transmitter Firmware Change

Add `device_kind` to the `identify` response:

In `transmitter/main/serial/serial_protocol.c`, the `handle_identify()` function adds one field:

```c
cJSON_AddStringToObject(payload, "device_kind", "TRANSMITTER_HUB");
```

No other transmitter changes.

## Frontend Changes

### `web/src/lib/serial/types.ts`

**New constant:**
```typescript
export const FTDI_USB_FILTER = {
  usbVendorId: 0x0403,
  usbProductId: 0x6001,
} as const;

export const ALL_DEVICE_FILTERS = [
  ESP32_C3_USB_FILTER,
  FTDI_USB_FILTER,
] as const;
```

**Extended `IdentifyPayload`:**
```typescript
export interface IdentifyPayload {
  public_id: string;
  device_name: string;
  device_kind: DeviceKind;
  op_state: string;
  firmware_version: string;
  mac: string;
}
```

**New command entry:**
```typescript
export interface SerialCommandMap {
  // ... existing entries ...
  retry: { payload: undefined; response: RestartResult };
}
```

**Type guard** (import `DeviceKind` from `@/types/device` — avoids redeclaring the existing type):
```typescript
import type { DeviceKind } from "@/types/device";
export type { DeviceKind };

export function isReceiverKind(kind: DeviceKind): boolean {
  return kind !== "TRANSMITTER_HUB";
}
```

### `web/src/lib/serial/use-serial.ts`

Six changes:

1. **`connect()`** — replace `ESP32_C3_USB_FILTER` with `ALL_DEVICE_FILTERS` in `requestPort()`:
   ```typescript
   const port = await navigator.serial.requestPort({
     filters: [...ALL_DEVICE_FILTERS],
   });
   ```

2. **`connect()` re-enumeration fallback** — search for any known filter, not just ESP32-C3:
   ```typescript
   const freshPort = ports.find((p) => {
     const info = p.getInfo();
     return ALL_DEVICE_FILTERS.some(
       (f) => info.usbVendorId === f.usbVendorId && info.usbProductId === f.usbProductId
     );
   });
   ```

3. **`reconnectKnownPort()`** — same filter expansion.

4. **`handleSerialConnect` event listener** — same filter expansion.

5. **New exposed state** — `deviceKind`:
   ```typescript
   const [deviceKind, setDeviceKind] = useState<DeviceKind | null>(null);
   ```
   Cleared on disconnect.

6. **`openPort()` becomes device-kind-aware** — after opening the port, it sends `identify` first to determine device kind. Then conditionally sends `status` and starts the 30-second poll **only for transmitters**. Receivers skip both since their serial connection is provisioning-only.

   ```typescript
   async (port: SerialPort) => {
     updatePortState("opening");
     await port.open({ baudRate: SERIAL_BAUD_RATE });
     await protocolRef.current.connect(port);
     portRef.current = port;
     updatePortState("open");

     const id = await protocolRef.current.send("identify");
     const kind = id.device_kind as DeviceKind;
     setDeviceKind(kind);

     if (!isReceiverKind(kind)) {
       const status = await protocolRef.current.send("status");
       setDeviceState(status);
       startStatusPoll();
     }
   }
   ```

### `web/src/features/device/usb-provision-dialog.tsx`

This is the main UI change. The dialog becomes device-kind-aware:

#### Detection step

After connecting, the dialog already sends `identify` (line 161). The `identify` response now includes `device_kind`. The dialog stores this and branches the form:

```typescript
const [deviceKind, setDeviceKind] = useState<DeviceKind | null>(null);

// In the identify effect:
const id = await sendCommand("identify");
setIdentity(id);
setDeviceKind(id.device_kind);
setStep("form");
```

#### Form branching

The "Hub name" (`assignedName`) field is shown only for transmitters:

```typescript
{!isReceiverKind(deviceKind) && (
  <div className="space-y-2">
    <Label htmlFor="usb-name">{tUsb("assigned_name")}</Label>
    <Input ... />
  </div>
)}
```

Validation skips `assignedName` for receivers.

#### Device info card

Shows `device_kind` label (translated) alongside MAC and firmware version.

#### Restart detection branching

The `waitForRestartReconnect()` function currently handles USB re-enumeration (transmitter behavior). The restart behavior depends on the **USB interface type**, not the device kind:

- **Native USB (Espressif `0x303a`):** Port re-enumerates on reboot — both transmitter and ESP32-C3 receiver. Requires `reconnectKnownPort()`.
- **USB-serial adapter (FTDI, etc.):** Port stays open through reboot — ESP8266 receiver. Poll `ping` on the existing connection.

The branching uses the port's vendor ID:

```typescript
function isNativeUsbPort(port: SerialPort): boolean {
  return port.getInfo().usbVendorId === ESP32_C3_USB_FILTER.usbVendorId;
}

async function waitForRestartReconnect(): Promise<void> {
  const nativeUsb = portRef.current && isNativeUsbPort(portRef.current);

  if (nativeUsb) {
    // USB Serial/JTAG re-enumerates on reboot (transmitter + ESP32-C3 receiver)
    for (let attempt = 0; attempt < 20; attempt++) {
      await sleep(2_000);
      try {
        await reconnectKnownPort();
        await sendCommand("ping");
        return;
      } catch {}
    }
  } else {
    // USB-serial adapter stays open through reboot (ESP8266 receiver)
    for (let attempt = 0; attempt < 20; attempt++) {
      await sleep(2_000);
      try {
        await sendCommand("ping");
        return;
      } catch {
        // device still rebooting
      }
    }
  }
}
```

#### Pending device polling

`listPendingHubIds()` currently queries `listDevices("TRANSMITTER_HUB", storeId)`. For receivers, it needs to query the correct kind:

```typescript
async function listPendingDeviceIds(): Promise<Set<string>> {
  const kind = deviceKind && isReceiverKind(deviceKind)
    ? deviceKind                  // "RECEIVER_433M" or "RECEIVER_2_4G"
    : "TRANSMITTER_HUB";
  const result = await listDevices(kind, storeId);
  return new Set(
    result.devices
      .filter((d) => d.status === "PENDING")
      .map((d) => d.id),
  );
}
```

#### Approval request

For receivers, `assignedName` is not collected in the form. The approval request auto-generates a name from the device MAC:

```typescript
await approveDevice(pending[0].id, {
  assignedName: deviceKind && isReceiverKind(deviceKind)
    ? `Receiver ${identity?.mac.slice(-5).replace(":", "")}`  // e.g. "Receiver 3456"
    : assignedName.trim(),
  storeId,
});
```

### `web/src/lib/serial/serial-protocol.ts`

No changes. The protocol layer is already device-agnostic.

### `web/src/lib/serial/support.ts`

No changes.

### `web/src/features/device/api.ts`

No changes. `listDevices()` already accepts arbitrary `kind` string. `issueEnrollmentToken()` only requires `storeId`.

### Page-level files

No changes to `devices/page.tsx` or `devices/[id]/page.tsx`. The "Provision via USB" button already opens the dialog, which now handles both device types.

## i18n

New keys under `devices.usb`:

| Key | EN | VI |
|-----|----|----|
| `detecting` | Detecting device... | Đang nhận diện thiết bị... |
| `device_type` | Device type | Loại thiết bị |
| `type_transmitter_hub` | Transmitter Hub | Hub phát |
| `type_receiver_433m` | Receiver (433 MHz) | Bộ nhận (433 MHz) |
| `type_receiver_2_4g` | Receiver (2.4 GHz) | Bộ nhận (2.4 GHz) |

Existing keys (`provision_title`, `provision_desc`, step labels, etc.) are already device-agnostic — no changes needed.

## Receiver-ESP8266 Firmware Changes

### Removed files

| File | Reason |
|------|--------|
| `main/provision/http_server.c` / `.h` | Replaced by serial protocol |
| `main/provision/recovery.c` / `.h` | Recovery logic moves into serial provisioning loop |
| `main/provision/index.html` / `index.html.gz` | No more embedded web UI |
| `main/network/wifi.c` → `wifi_start_softap()` | SoftAP no longer needed |
| `main/Kconfig.projbuild` → `RECEIVER_AP_PASSWORD`, `RECEIVER_AP_CHANNEL` | SoftAP config removed |

### New files

| File | Purpose |
|------|---------|
| `main/serial/serial_protocol.h` | Public API: `serial_protocol_init()`, `serial_protocol_run_blocking()` |
| `main/serial/serial_protocol.c` | UART driver setup, line reader, JSON command dispatcher |

Uses the ESP8266 UART driver (`uart_driver_install` on UART0) instead of the ESP32-C3's USB Serial/JTAG driver. Otherwise follows the same structure as the transmitter's `serial_protocol.c`.

### Modified files

| File | Change |
|------|--------|
| `main/main.c` | Replace `provision_run_recovery_mode()` calls with `serial_protocol_run_blocking()` |
| `sdkconfig` | `CONFIG_ESP_CONSOLE_UART_BAUDRATE` → 115200, `CONFIG_ESPTOOLPY_MONITOR_BAUD` → 115200 |

## Receiver-ESP32 Firmware Changes

The ESP32-C3 receiver uses the same chip and SDK as the transmitter. SoftAP + HTTP provisioning is **kept** as a backup. Serial provisioning is added as a concurrent path using a background FreeRTOS task (same pattern as the transmitter). The serial task is torn down once the device reaches operational state.

### New files

| File | Purpose |
|------|---------|
| `main/serial/serial_protocol.h` | Public API: `serial_protocol_init()`, `serial_protocol_stop()` |
| `main/serial/serial_protocol.c` | USB Serial/JTAG driver, background read task, JSON command dispatcher |

Uses `usb_serial_jtag_driver_install()` — same driver as the transmitter. The command handlers implement the receiver command set (no `transmit`, `lifecycle`, `update_mqtt`, or `status`). The `identify` response derives `device_kind` from the existing `RECEIVER_RADIO_VARIANT` Kconfig choice via `device_config_receiver_type_string(device_config_compiled_receiver_type())`. The `provision` handler saves to NVS and reboots directly (same as transmitter), bypassing the HTTP event group.

### Modified files

| File | Change |
|------|--------|
| `main/main.c` | Add `serial_protocol_init()` early in `app_main()`, add `serial_protocol_stop()` before entering operational idle loop |

### Kept unchanged

All existing provisioning files remain: `http_server.c/h`, `recovery.c/h`, `index.html.gz`, `wifi_start_softap()`, SoftAP Kconfig entries.

## Out of Scope

- Receiver control panel (status monitoring, console log, reset/retry UI when connected to a provisioned receiver)
- Receiver dispatch controls in the USB dispatch dialog
- Any changes to `usb-control-panel.tsx` or `usb-dispatch-dialog.tsx`
- Backend enrollment token or device registration changes (already supports receivers)
- Receiver operational-mode serial listening (ESP32-C3 serial task torn down after provisioning; ESP8266 serial is blocking provisioning-only)
- Supporting additional USB-serial adapter VID:PIDs beyond the current FTDI `0x0403:0x6001` (the filter list is extensible; new adapters can be added trivially when needed)
