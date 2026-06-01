# USB Provisioning — Receiver Extension Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend the existing USB provisioning flow to support both ESP32-C3 and ESP8266 receiver modules alongside the ESP32-C3 transmitter, using a single auto-detecting provisioning dialog.

**Architecture:** The frontend serial stack gains multi-device awareness: expanded USB filters, device-kind detection via the `identify` command, and a provisioning dialog that branches by device type. The ESP32-C3 receiver adds serial provisioning as an async background task alongside the existing SoftAP + HTTP (kept as backup), then tears it down after reaching operational state. The ESP8266 receiver replaces SoftAP + HTTP with a synchronous blocking serial protocol (saves RAM). The transmitter gets one new field in its identify response.

**Tech Stack:** TypeScript/React (Next.js 16), ESP-IDF v6.0 (C), ESP8266 RTOS SDK (C), Web Serial API, cJSON

**Spec:** `docs/planned/USB Provisioning Receiver Extension Spec.md`

---

## Phase 1 — Frontend

### Task 1: Serial Types

**Files:**
- Modify: `web/src/lib/serial/types.ts`

- [ ] **Step 1: Add FTDI USB filter and combined filter array**

After the existing `ESP32_C3_USB_FILTER` constant (line 117), add:

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

- [ ] **Step 2: Add `device_kind` to `IdentifyPayload`**

Import `DeviceKind` from the existing device types (avoids re-declaring the type that `web/src/types/device.ts` already defines):

```typescript
import type { DeviceKind } from "@/types/device";
```

Add the field to the existing `IdentifyPayload` interface (line 69):

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

Re-export `DeviceKind` for convenience:

```typescript
export type { DeviceKind };
```

- [ ] **Step 3: Add `isReceiverKind` helper**

After the `IdentifyPayload` interface:

```typescript
export function isReceiverKind(kind: DeviceKind): boolean {
  return kind !== "TRANSMITTER_HUB";
}
```

- [ ] **Step 4: Add `retry` command to `SerialCommandMap`**

Add after the `lifecycle` entry in `SerialCommandMap` (line 112):

```typescript
export interface SerialCommandMap {
  ping: { payload: undefined; response: PingResult };
  identify: { payload: undefined; response: IdentifyPayload };
  status: { payload: undefined; response: StatusPayload };
  provision: { payload: ProvisionPayload; response: RestartResult };
  "provision.test_wifi": { payload: TestWifiPayload; response: TestWifiResult };
  update_mqtt: { payload: UpdateMqttPayload; response: RestartResult };
  factory_reset: { payload: undefined; response: RestartResult };
  transmit: { payload: TransmitPayload; response: TransmitResult };
  lifecycle: { payload: LifecyclePayload; response: LifecycleResult };
  retry: { payload: undefined; response: RestartResult };
}
```

---

### Task 2: Serial Hook — Multi-Device Awareness

**Files:**
- Modify: `web/src/lib/serial/use-serial.ts`

- [ ] **Step 1: Update imports**

Replace the current imports from `./types` (lines 6-11):

```typescript
import type { DeviceKind, SerialCommandMap, StatusPayload } from "./types";
import {
  ALL_DEVICE_FILTERS,
  ESP32_C3_USB_FILTER,
  SERIAL_BAUD_RATE,
  SERIAL_SESSION_POLL_INTERVAL_MS,
  isReceiverKind,
} from "./types";
```

- [ ] **Step 2: Add `deviceKind` to return interface and state**

Extend `UseSerialReturn` (line 14) to include `deviceKind`:

```typescript
export interface UseSerialReturn {
  canUseSerial: boolean;
  portState: PortState;
  identifyPayload: IdentifyPayload | null;
  deviceKind: DeviceKind | null;
  isNativeUsbPort: boolean;
  connect: () => Promise<void>;
  reconnectKnownPort: () => Promise<void>;
  disconnect: () => Promise<void>;
  refreshStatus: () => Promise<StatusPayload | null>;
  sendCommand: <K extends keyof SerialCommandMap>(
    type: K,
    ...args: SerialCommandMap[K]["payload"] extends undefined
      ? []
      : [payload: SerialCommandMap[K]["payload"]]
  ) => Promise<SerialCommandMap[K]["response"]>;
  deviceState: StatusPayload | null;
  events: EventTarget;
}
```

Add the `IdentifyPayload` import (already in the type import line from Step 1):

```typescript
import type { DeviceKind, IdentifyPayload, SerialCommandMap, StatusPayload } from "./types";
```

Add state declarations after `deviceState` state (line 35):

```typescript
const [identifyPayload, setIdentifyPayload] = useState<IdentifyPayload | null>(null);
const [isNativeUsbPort, setIsNativeUsbPort] = useState(true);
```

Derive `deviceKind` as a computed value (no separate state needed):

```typescript
const deviceKind = identifyPayload?.device_kind ?? null;
```

Include in the return object (line 320):

```typescript
return {
  canUseSerial,
  portState,
  identifyPayload,
  deviceKind,
  isNativeUsbPort,
  connect,
  reconnectKnownPort,
  disconnect,
  refreshStatus,
  sendCommand,
  deviceState,
  events: protocolRef.current.events,
};
```

- [ ] **Step 3: Make `openPort()` device-kind-aware**

Replace the `openPort` callback (lines 115-141). After opening the port and connecting the protocol, it now sends `identify` first to detect device kind, then only sends `status` and starts polling for transmitters:

```typescript
const openPort = useCallback(
  async (port: SerialPort) => {
    updatePortState("opening");
    try {
      await port.open({ baudRate: SERIAL_BAUD_RATE });
      await protocolRef.current.connect(port);
      portRef.current = port;
      const portInfo = port.getInfo();
      setIsNativeUsbPort(portInfo.usbVendorId === ESP32_C3_USB_FILTER.usbVendorId);
      updatePortState("open");

      const id = await protocolRef.current.send("identify");
      setIdentifyPayload(id);

      if (!isReceiverKind(id.device_kind as DeviceKind)) {
        const status = await protocolRef.current.send("status");
        setDeviceState(status);
        startStatusPoll();
      }
    } catch (error) {
      stopStatusPoll();
      await protocolRef.current.disconnect();
      try {
        await port.close();
      } catch {
        // ignore close errors after a partially opened port
      }
      portRef.current = null;
      setDeviceKind(null);
      setIsNativeUsbPort(true);
      updatePortState("closed");
      throw error;
    }
  },
  [startStatusPoll, stopStatusPoll, updatePortState],
);
```

- [ ] **Step 4: Expand `connect()` filters**

In the `connect` callback, replace `requestPort` filter (line 149-151):

```typescript
const port = await navigator.serial.requestPort({
  filters: [...ALL_DEVICE_FILTERS],
});
```

Replace the re-enumeration fallback `ports.find` block (lines 166-172):

```typescript
const freshPort = ports.find((p) => {
  const info = p.getInfo();
  return ALL_DEVICE_FILTERS.some(
    (f) =>
      info.usbVendorId === f.usbVendorId &&
      info.usbProductId === f.usbProductId,
  );
});
```

- [ ] **Step 5: Expand `reconnectKnownPort()` filters**

Replace the `ports.find` block in `reconnectKnownPort` (lines 187-193):

```typescript
const knownPort = ports.find((p) => {
  const info = p.getInfo();
  return ALL_DEVICE_FILTERS.some(
    (f) =>
      info.usbVendorId === f.usbVendorId &&
      info.usbProductId === f.usbProductId,
  );
});
if (knownPort) {
  await openPort(knownPort);
}
```

- [ ] **Step 6: Expand `handleSerialConnect` auto-connect filter**

In the serial connect event listener (lines 282-296), replace the ESP32-specific check:

```typescript
const handleSerialConnect = (e: Event) => {
  if (manualConnectRef.current) return;
  if (portStateRef.current !== "closed") return;
  const port = e.target as SerialPort;
  const info = port.getInfo();
  const isKnown = ALL_DEVICE_FILTERS.some(
    (f) =>
      info.usbVendorId === f.usbVendorId &&
      info.usbProductId === f.usbProductId,
  );
  if (isKnown) {
    void openPort(port).catch(() => {
      portRef.current = null;
      updatePortState("closed");
    });
  }
};
```

- [ ] **Step 7: Clear `deviceKind` on disconnect**

In the `disconnect` callback (lines 199-217), add `setDeviceKind(null)` after `setDeviceState(null)`:

```typescript
const disconnect = useCallback(async () => {
  if (portStateRef.current !== "open") return;
  updatePortState("closing");
  stopStatusPoll();
  setDeviceState(null);
  setIdentifyPayload(null);
  setIsNativeUsbPort(true);

  await protocolRef.current.disconnect();

  if (portRef.current) {
    try {
      await portRef.current.close();
    } catch {
      // ignore close errors
    }
    portRef.current = null;
  }

  updatePortState("closed");
}, [stopStatusPoll, updatePortState]);
```

Also clear in the `handleStreamClosed` handler (lines 252-257):

```typescript
const handleStreamClosed = () => {
  stopStatusPoll();
  setDeviceState(null);
  setIdentifyPayload(null);
  setIsNativeUsbPort(true);
  portRef.current = null;
  updatePortState("closed");
};
```

And in the `handleSerialDisconnect` handler (lines 272-279):

```typescript
const handleSerialDisconnect = (e: Event) => {
  if (portRef.current && e.target === portRef.current) {
    stopStatusPoll();
    void protocolRef.current.disconnect();
    setDeviceState(null);
    setDeviceKind(null);
    setIsNativeUsbPort(true);
    portRef.current = null;
    updatePortState("closed");
  }
};
```

---

### Task 3: Provisioning Dialog — Device-Kind Branching

**Files:**
- Modify: `web/src/features/device/usb-provision-dialog.tsx`

- [ ] **Step 1: Update imports**

Add device-kind imports. Replace the serial types import (line 24):

```typescript
import type { DeviceKind, IdentifyPayload, TestWifiResult } from "@/lib/serial/types";
import { ESP32_C3_USB_FILTER, isReceiverKind } from "@/lib/serial/types";
```

- [ ] **Step 2: Destructure hook values and derive identity**

Update the `useSerial()` destructure (lines 90-99). The hook now exposes the full `identifyPayload`, so the dialog no longer needs its own `identify` state or command:

```typescript
const {
  canUseSerial,
  portState,
  identifyPayload,
  deviceKind,
  isNativeUsbPort,
  connect,
  reconnectKnownPort,
  disconnect,
  sendCommand,
  deviceState,
} = useSerial();
```

Remove the `identity` state declaration (line 102) and the `refreshStatus` destructure (no longer needed — receivers don't use it, and the restart flow now uses `ping`).

Use `identifyPayload` wherever the dialog previously used `identity`. For example, the existing identify effect (lines 154-178) that sends `identify` after connection — **remove it entirely**. The hook already sends `identify` in `openPort()`. Instead, add an effect that transitions to the `form` step when `identifyPayload` arrives:

```typescript
useEffect(() => {
  if (!open || step !== "connect" || !identifyPayload) return;
  setStep("form");
}, [open, step, identifyPayload]);
```

- [ ] **Step 3: Add device-kind display to the info card**

In the device info card (lines 375-388), replace `identity` with `identifyPayload` and add a device type row after the MAC row:

```typescript
<Card className="rounded-lg p-3">
  <div className="grid grid-cols-2 gap-2 text-sm">
    <span className="text-muted-foreground">{tUsb("mac")}</span>
    <span className="font-mono">{identifyPayload.mac}</span>
    <span className="text-muted-foreground">{tUsb("device_type")}</span>
    <span>
      {deviceKind === "TRANSMITTER_HUB"
        ? tUsb("type_transmitter_hub")
        : deviceKind === "RECEIVER_433M"
          ? tUsb("type_receiver_433m")
          : deviceKind === "RECEIVER_2_4G"
            ? tUsb("type_receiver_2_4g")
            : deviceKind}
    </span>
    <span className="text-muted-foreground">
      {tUsb("firmware")}
    </span>
    <span>{identifyPayload.firmware_version}</span>
    <span className="text-muted-foreground">
      {tDevices("detail.status")}
    </span>
    <span>{identifyPayload.op_state}</span>
  </div>
</Card>
```

- [ ] **Step 4: Conditionally hide hub name field for receivers**

Wrap the assigned name field block (lines 416-428) in a conditional:

```typescript
{deviceKind && !isReceiverKind(deviceKind) && (
  <div className="space-y-2">
    <Label htmlFor="usb-name">{tUsb("assigned_name")}</Label>
    <Input
      id="usb-name"
      value={assignedName}
      onChange={(e) => setAssignedName(e.target.value)}
      maxLength={100}
      aria-invalid={!!errors.assignedName}
    />
    {errors.assignedName && (
      <InlineError message={errors.assignedName} className="mt-1" />
    )}
  </div>
)}
```

- [ ] **Step 5: Update validation to skip name for receivers**

In the `validate()` function (lines 180-192), make the `assignedName` check conditional:

```typescript
function validate(): boolean {
  const errs: Record<string, string> = {};
  if (deviceKind && !isReceiverKind(deviceKind) && !assignedName.trim())
    errs.assignedName = tDevices("pending.approveNameRequired");
  if (!storeId) errs.storeId = tDevices("pending.approveStoreRequired");
  if (!wifiSsid.trim()) errs.wifiSsid = tDevices("usb.wifi_fail");
  if (!mqttUri.trim() || !mqttUri.startsWith("mqtts://"))
    errs.mqttUri = tErrors("badRequest");
  if (!mqttUser.trim()) errs.mqttUser = tErrors("badRequest");
  if (!mqttPwd.trim()) errs.mqttPwd = tErrors("badRequest");
  setErrors(errs);
  return Object.keys(errs).length === 0;
}
```

- [ ] **Step 6: Replace `listPendingHubIds` with device-kind-aware version**

Rename and update the function (lines 214-221):

```typescript
async function listPendingDeviceIds(): Promise<Set<string>> {
  const kind =
    deviceKind && isReceiverKind(deviceKind) ? deviceKind : "TRANSMITTER_HUB";
  const result = await listDevices(kind, storeId);
  return new Set(
    result.devices
      .filter((device) => device.status === "PENDING")
      .map((device) => device.id),
  );
}
```

Update the call site in `handleProvision` (line 242):

```typescript
const existingPendingIds = await listPendingDeviceIds();
```

- [ ] **Step 7: Replace `waitForRestartReconnect` with port-aware branching**

The restart behavior depends on the USB interface, not the device kind. Native USB ports (Espressif `0x303a`) re-enumerate on reboot; USB-serial adapters (FTDI etc.) stay open.

Replace the function (lines 223-234). The `isNativeUsbPort` flag is already set by the hook in Task 2 Step 3 (`openPort`) and exposed via `UseSerialReturn`:

```typescript
const {
  // ... existing destructures
  isNativeUsbPort,
} = useSerial();

async function waitForRestartReconnect(): Promise<void> {
  if (isNativeUsbPort) {
    for (let attempt = 0; attempt < 20; attempt++) {
      await sleep(2_000);
      try {
        await reconnectKnownPort();
        await sendCommand("ping");
        return;
      } catch {
        // waiting for USB re-enumeration
      }
    }
  } else {
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

- [ ] **Step 8: Update `pollForPendingDevice` to use correct kind**

Replace the hardcoded `"TRANSMITTER_HUB"` in `pollForPendingDevice` (line 297):

```typescript
const kind =
  deviceKind && isReceiverKind(deviceKind) ? deviceKind : "TRANSMITTER_HUB";
const result = await listDevices(kind, storeId);
```

- [ ] **Step 9: Update approval to auto-name receivers**

Replace the `approveDevice` call (lines 305-308):

```typescript
await approveDevice(pending[0].id, {
  assignedName:
    deviceKind && isReceiverKind(deviceKind)
      ? `Receiver ${identifyPayload?.mac.slice(-5).replace(":", "")}`
      : assignedName.trim(),
  storeId,
});
```

- [ ] **Step 10: Reset `deviceKind` state on dialog open**

The `identifyPayload` comes from the hook and is cleared on disconnect. The dialog already disconnects on close (line 142-146). No additional reset needed.

Update the form guard condition (line 373) from `identity &&` to `identifyPayload &&`:

```typescript
{(step === "form" || step === "testing_wifi") && identifyPayload && (
```

Update the error retry button (line 629) — replace `identity ? "form" : "connect"` with `identifyPayload ? "form" : "connect"`.

The `already_provisioned` warning (`deviceState?.provisioned`, line 390) will be `null` for receivers since they don't get `deviceState` (no `status` command). This is correct — receivers in provisioning mode are by definition un-provisioned or in recovery.

---

### Task 4: i18n

**Files:**
- Modify: `web/src/messages/en.json`
- Modify: `web/src/messages/vi.json`

- [ ] **Step 1: Add new English keys**

Inside the `devices.usb` object, add these keys (insert after `"connecting"` key):

```json
"detecting": "Detecting device...",
"device_type": "Device type",
"type_transmitter_hub": "Transmitter Hub",
"type_receiver_433m": "Receiver (433 MHz)",
"type_receiver_2_4g": "Receiver (2.4 GHz)",
```

- [ ] **Step 2: Add new Vietnamese keys**

Inside the `devices.usb` object in `vi.json`, add at the same position:

```json
"detecting": "Đang nhận diện thiết bị...",
"device_type": "Loại thiết bị",
"type_transmitter_hub": "Hub phát",
"type_receiver_433m": "Bộ nhận (433 MHz)",
"type_receiver_2_4g": "Bộ nhận (2.4 GHz)",
```

- [ ] **Step 3: Update `provision_desc` for device-generic wording**

The current `provision_desc` in `en.json` says "transmitter hub". Update to be device-generic:

```json
"provision_desc": "Connect a device via USB to send WiFi and MQTT settings from the dashboard. The dashboard generates the enrollment token; pending approval still applies."
```

Vietnamese:

```json
"provision_desc": "Kết nối thiết bị qua USB để gửi cấu hình WiFi và MQTT từ bảng điều khiển. Bảng điều khiển tự tạo mã đăng ký; thiết bị vẫn cần được duyệt."
```

- [ ] **Step 4: Verify structural mirroring**

Verify both JSON files have the same keys in the same order. Count the keys in `devices.usb` in both files — they must match.

Run: `cd web && yarn lint`

---

## Phase 2 — Transmitter Firmware

### Task 5: Add `device_kind` to Transmitter `identify` Response

**Files:**
- Modify: `transmitter/main/serial/serial_protocol.c:176-198`

- [ ] **Step 1: Add `device_kind` field to `handle_identify()`**

In the `handle_identify` function, add one line after the `firmware_version` field (before `serial_send_response`):

```c
static void handle_identify(const char *id)
{
    const device_config_t *cfg = device_config_get();
    uint8_t mac[6];
    esp_read_mac(mac, ESP_MAC_WIFI_STA);

    char mac_str[18];
    snprintf(mac_str, sizeof(mac_str), "%02X:%02X:%02X:%02X:%02X:%02X",
             mac[0], mac[1], mac[2], mac[3], mac[4], mac[5]);

    cJSON *payload = cJSON_CreateObject();
    cJSON_AddStringToObject(payload, "public_id",
                            cfg->has_public_id ? cfg->public_id : "");
    cJSON_AddStringToObject(payload, "device_name",
                            cfg->has_device_name ? cfg->device_name : "");
    cJSON_AddStringToObject(payload, "device_kind", "TRANSMITTER_HUB");
    cJSON_AddStringToObject(payload, "op_state",
                            cfg->has_op_state
                                ? device_config_op_state_to_string(cfg->op_state)
                                : "INVALID");
    cJSON_AddStringToObject(payload, "firmware_version", CONFIG_TRANSMITTER_FIRMWARE_VERSION);
    cJSON_AddStringToObject(payload, "mac", mac_str);
    serial_send_response(id, true, payload, NULL);
}
```

The only change is adding `cJSON_AddStringToObject(payload, "device_kind", "TRANSMITTER_HUB");`.

---

## Receiver API Differences from Transmitter

The command handler code in Tasks 6 and 8 is modeled on the transmitter, but the receiver codebases have different API names. The executor must adapt:

| Operation | Transmitter API | ESP32 Receiver API | ESP8266 Receiver API |
|-----------|----------------|--------------------|-----------------------|
| Get config | `device_config_get()` (global pointer) | `device_config_load(&cfg)` (load/free) | `device_config_load(&cfg)` (load/free) |
| Save provisioning | `device_config_save_*()` (individual args) | `device_config_save_provisioning(cfg, &prov)` | `device_config_store_provisioning(&prov)` |
| Factory reset | `device_config_reprovision()` | `device_config_reprovision()` | `device_config_erase_runtime()` |
| Op state to string | `device_config_op_state_to_string()` | `device_config_state_name(&cfg)` | `device_config_state_name(&cfg)` |
| Provisioned check | (inline) | `device_config_is_provisioned(&cfg)` | `device_config_is_provisioned(&cfg)` |
| WiFi test | `wifi_test_sta(ssid, pwd, timeout_ms)` | **Does not exist — must be implemented** | **Does not exist — must be implemented** |

Both receivers use a `device_provisioning_t` struct for saving credentials:

```c
// ESP32: char arrays with fixed max lengths
device_provisioning_t prov;
strncpy(prov.wifi_ssid, ssid->valuestring, sizeof(prov.wifi_ssid));
// ...
device_config_save_provisioning(&g_cfg, &prov);

// ESP8266: const char* pointers
device_provisioning_t prov = {
    .wifi_ssid = ssid->valuestring,
    .wifi_pwd  = pwd_str,
    .mqtt_uri  = uri->valuestring,
    // ...
};
device_config_store_provisioning(&prov);
```

Neither receiver has `device_config_get()`. The receivers use a load/free pattern — the `serial_protocol_run_blocking()` function receives the config context from `main.c` (either as a global or parameter). Use the same pattern the existing HTTP server uses.

The `wifi_test_sta()` function (used by `provision.test_wifi`) does not exist on either receiver. It must be added to `network/wifi.c` — model it on the transmitter's implementation. It temporarily connects to a given SSID, reports RSSI/IP, then disconnects. If this is too complex, `provision.test_wifi` can be deferred and the command handler can return `{"ok":false,"error":"not_supported"}`.

---

## Phase 3 — Receiver-ESP32 Firmware

### Task 6: Create Serial Protocol for Receiver-ESP32

**Files:**
- Create: `receiver-esp32/main/serial/serial_protocol.h`
- Create: `receiver-esp32/main/serial/serial_protocol.c`
- Modify: `receiver-esp32/main/CMakeLists.txt` (add new source)

Reference: The implementation mirrors `transmitter/main/serial/serial_protocol.c` with these key differences:
- **Asynchronous background task** (same as transmitter), runs alongside existing SoftAP + HTTP provisioning
- Receiver command set only: `ping`, `identify`, `provision`, `provision.test_wifi`, `factory_reset`, `retry`
- No `status`, `transmit`, `lifecycle`, `update_mqtt` commands
- `identify` response returns `device_kind` as `"RECEIVER_433M"` (or appropriate variant based on Kconfig)
- Uses `usb_serial_jtag_driver_install()` + `usb_serial_jtag_vfs_use_driver()` (ESP-IDF v6.0 API)
- The `provision` handler saves to NVS and reboots directly (same as transmitter) — it does not go through the HTTP event group
- API exposes `serial_protocol_stop()` to tear down the background task once the device enters operational state

- [ ] **Step 1: Create the header file**

```c
/* receiver-esp32/main/serial/serial_protocol.h */
#ifndef RECEIVER_SERIAL_PROTOCOL_H
#define RECEIVER_SERIAL_PROTOCOL_H

#include "esp_err.h"

/**
 * Install USB Serial/JTAG driver and start background read task.
 * Call early in app_main(), before provisioning/recovery checks.
 */
esp_err_t serial_protocol_init(void);

/**
 * Stop the background serial read task and release resources.
 * Call when entering operational state (serial no longer needed).
 */
void serial_protocol_stop(void);

#endif
```

- [ ] **Step 2: Create the implementation — USB driver and helpers**

Create `receiver-esp32/main/serial/serial_protocol.c`. Start with includes, constants, and infrastructure:

```c
#include "serial/serial_protocol.h"
#include "driver/usb_serial_jtag.h"
#include "driver/usb_serial_jtag_vfs.h"
#include "cJSON.h"
#include "config/device_config.h"
#include "security/device_identity.h"
#include "network/wifi.h"
#include "esp_log.h"
#include "esp_mac.h"
#include "esp_system.h"
#include "esp_timer.h"
#include "esp_wifi.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "sdkconfig.h"
#include <string.h>
#include <stdio.h>

static const char *TAG = "serial_prov";

#define LINE_BUF_SIZE 1024
#define TX_BUF_SIZE   4096
#define RX_BUF_SIZE   4096

static char s_line_buf[LINE_BUF_SIZE];
static int  s_line_pos;

static void send_json(cJSON *root)
{
    char *str = cJSON_PrintUnformatted(root);
    if (str) {
        printf("%s\n", str);
        fflush(stdout);
        cJSON_free(str);
    }
    cJSON_Delete(root);
}

static void send_response(const char *id, bool ok, cJSON *payload, const char *error)
{
    cJSON *root = cJSON_CreateObject();
    cJSON_AddStringToObject(root, "id", id);
    cJSON_AddStringToObject(root, "type", "response");
    cJSON_AddBoolToObject(root, "ok", ok);
    if (ok && payload) {
        cJSON_AddItemToObject(root, "payload", payload);
    } else {
        cJSON_Delete(payload);
    }
    if (!ok && error) {
        cJSON_AddStringToObject(root, "error", error);
    }
    send_json(root);
}

```

- [ ] **Step 3: Implement command handlers**

Add after the helpers in the same file. Each handler matches the transmitter's request/response format.

```c
static void handle_ping(const char *id)
{
    cJSON *payload = cJSON_CreateObject();
    cJSON_AddNumberToObject(payload, "uptime_ms",
                            (double)(esp_timer_get_time() / 1000));
    send_response(id, true, payload, NULL);
}

static void handle_identify(const char *id, const device_config_t *cfg)
{
    uint8_t mac[6];
    esp_read_mac(mac, ESP_MAC_WIFI_STA);
    char mac_str[18];
    snprintf(mac_str, sizeof(mac_str), "%02X:%02X:%02X:%02X:%02X:%02X",
             mac[0], mac[1], mac[2], mac[3], mac[4], mac[5]);

    cJSON *payload = cJSON_CreateObject();
    cJSON_AddStringToObject(payload, "public_id",
                            cfg->has_public_id ? cfg->public_id : "");
    cJSON_AddStringToObject(payload, "device_name",
                            cfg->has_device_name ? cfg->device_name : "");
    cJSON_AddStringToObject(payload, "device_kind",
                            device_config_receiver_type_string(
                                device_config_compiled_receiver_type()));
    cJSON_AddStringToObject(payload, "op_state",
                            cfg->has_op_state
                                ? device_config_state_name(cfg)
                                : "NONE");
    cJSON_AddStringToObject(payload, "firmware_version", CONFIG_RECEIVER_FIRMWARE_VERSION);
    cJSON_AddStringToObject(payload, "mac", mac_str);
    send_response(id, true, payload, NULL);
}

static void handle_provision(const char *id, cJSON *req_payload)
{
    if (!req_payload) {
        send_response(id, false, NULL, "missing_payload");
        return;
    }

    cJSON *ssid  = cJSON_GetObjectItem(req_payload, "wifi_ssid");
    cJSON *pwd   = cJSON_GetObjectItem(req_payload, "wifi_pwd");
    cJSON *uri   = cJSON_GetObjectItem(req_payload, "mqtt_uri");
    cJSON *muser = cJSON_GetObjectItem(req_payload, "mqtt_user");
    cJSON *mpwd  = cJSON_GetObjectItem(req_payload, "mqtt_pwd");
    cJSON *token = cJSON_GetObjectItem(req_payload, "enroll_token");

    if (!cJSON_IsString(ssid)  || !cJSON_IsString(uri) ||
        !cJSON_IsString(muser) || !cJSON_IsString(mpwd) ||
        !cJSON_IsString(token)) {
        send_response(id, false, NULL, "missing_fields");
        return;
    }

    /* ESP32 receiver uses struct with fixed-size char arrays.
       ESP8266 uses const char* pointers — adapt field assignment. See
       "Receiver API Differences" section above. */
    device_provisioning_t prov = { 0 };
    strncpy(prov.wifi_ssid, ssid->valuestring, sizeof(prov.wifi_ssid) - 1);
    strncpy(prov.wifi_pwd,
            (pwd && cJSON_IsString(pwd)) ? pwd->valuestring : "",
            sizeof(prov.wifi_pwd) - 1);
    strncpy(prov.mqtt_uri, uri->valuestring, sizeof(prov.mqtt_uri) - 1);
    strncpy(prov.mqtt_user, muser->valuestring, sizeof(prov.mqtt_user) - 1);
    strncpy(prov.mqtt_pwd, mpwd->valuestring, sizeof(prov.mqtt_pwd) - 1);
    strncpy(prov.enroll_token, token->valuestring, sizeof(prov.enroll_token) - 1);

    /* ESP32: device_config_save_provisioning(&g_cfg, &prov)
       ESP8266: device_config_store_provisioning(&prov) */
    esp_err_t err = device_config_save_provisioning(&g_cfg, &prov);

    if (err != ESP_OK) {
        send_response(id, false, NULL, "nvs_write_failed");
        return;
    }

    cJSON *p = cJSON_CreateObject();
    cJSON_AddBoolToObject(p, "restarting", 1);
    send_response(id, true, p, NULL);
}

static void handle_test_wifi(const char *id, cJSON *req_payload)
{
    if (!req_payload) {
        send_response(id, false, NULL, "missing_payload");
        return;
    }

    cJSON *ssid = cJSON_GetObjectItem(req_payload, "wifi_ssid");
    cJSON *pwd  = cJSON_GetObjectItem(req_payload, "wifi_pwd");

    if (!cJSON_IsString(ssid)) {
        send_response(id, false, NULL, "missing_fields");
        return;
    }

    esp_err_t err = wifi_test_sta(
        ssid->valuestring,
        (pwd && cJSON_IsString(pwd)) ? pwd->valuestring : NULL,
        15000);

    cJSON *p = cJSON_CreateObject();
    if (err == ESP_OK) {
        cJSON_AddBoolToObject(p, "connected", 1);
        wifi_sta_ip_info_t info;
        if (wifi_get_sta_info(&info) == ESP_OK) {
            cJSON_AddNumberToObject(p, "rssi", info.rssi);
            cJSON_AddStringToObject(p, "ip", info.ip_str);
        }
    } else {
        cJSON_AddBoolToObject(p, "connected", 0);
    }
    wifi_stop_test_sta();
    send_response(id, true, p, NULL);
}

static void handle_factory_reset(const char *id)
{
    esp_err_t err = device_config_reprovision();
    if (err != ESP_OK) {
        send_response(id, false, NULL, "reset_failed");
        return;
    }
    cJSON *p = cJSON_CreateObject();
    cJSON_AddBoolToObject(p, "restarting", 1);
    send_response(id, true, p, NULL);
}

static void handle_retry(const char *id, const device_config_t *cfg)
{
    if (!device_config_is_provisioned(cfg)) {
        send_response(id, false, NULL, "not_provisioned");
        return;
    }
    cJSON *p = cJSON_CreateObject();
    cJSON_AddBoolToObject(p, "restarting", 1);
    send_response(id, true, p, NULL);
}
```

**Important:** See the "Receiver API Differences" section above for the correct function names. The `wifi_test_sta()` / `wifi_get_sta_info()` / `wifi_stop_test_sta()` functions do not exist on either receiver — they must be added to `network/wifi.c` (model on the transmitter's implementation), or the `handle_test_wifi` handler can return `{"ok":false,"error":"not_supported"}` as a deferral. The `CONFIG_RECEIVER_DEVICE_KIND` and `CONFIG_RECEIVER_FIRMWARE_VERSION` macros must be defined in `Kconfig.projbuild`.

- [ ] **Step 4: Implement the init, background task, and stop**

Add at the bottom of `serial_protocol.c`. This mirrors the transmitter's async pattern — the read loop runs in a dedicated FreeRTOS task. The `provision`, `factory_reset`, and `retry` handlers save to NVS and call `esp_restart()` directly (same as transmitter).

```c
static TaskHandle_t s_serial_task;

static void dispatch_command(const char *id, const char *type, cJSON *payload)
{
    if (strcmp(type, "ping") == 0) {
        handle_ping(id);
    } else if (strcmp(type, "identify") == 0) {
        device_config_t cfg;
        device_config_load(&cfg);
        handle_identify(id, &cfg);
    } else if (strcmp(type, "provision") == 0) {
        handle_provision(id, payload);
        fflush(stdout);
        vTaskDelay(pdMS_TO_TICKS(250));
        esp_restart();
    } else if (strcmp(type, "provision.test_wifi") == 0) {
        handle_test_wifi(id, payload);
    } else if (strcmp(type, "factory_reset") == 0) {
        handle_factory_reset(id);
        fflush(stdout);
        vTaskDelay(pdMS_TO_TICKS(250));
        esp_restart();
    } else if (strcmp(type, "retry") == 0) {
        device_config_t cfg;
        device_config_load(&cfg);
        handle_retry(id, &cfg);
        fflush(stdout);
        vTaskDelay(pdMS_TO_TICKS(250));
        esp_restart();
    } else {
        send_response(id, false, NULL, "unknown_command");
    }
}

static void serial_read_task(void *arg)
{
    s_line_pos = 0;

    for (;;) {
        int c = fgetc(stdin);
        if (c == EOF) {
            vTaskDelay(pdMS_TO_TICKS(100));
            continue;
        }
        if (c == '\r') continue;
        if (c == '\n') {
            if (s_line_pos == 0) continue;
            s_line_buf[s_line_pos] = '\0';
            s_line_pos = 0;

            cJSON *root = cJSON_Parse(s_line_buf);
            if (!root) continue;

            cJSON *id_item   = cJSON_GetObjectItem(root, "id");
            cJSON *type_item = cJSON_GetObjectItem(root, "type");

            if (!cJSON_IsString(id_item) || !cJSON_IsString(type_item)) {
                cJSON_Delete(root);
                continue;
            }

            cJSON *payload = cJSON_GetObjectItem(root, "payload");
            dispatch_command(id_item->valuestring, type_item->valuestring, payload);
            cJSON_Delete(root);
            continue;
        }

        if (s_line_pos < LINE_BUF_SIZE - 1) {
            s_line_buf[s_line_pos++] = (char)c;
        } else {
            ESP_LOGW(TAG, "Line buffer overflow, discarding");
            s_line_pos = 0;
        }
    }
}

esp_err_t serial_protocol_init(void)
{
    usb_serial_jtag_driver_config_t drv_cfg = {
        .tx_buffer_size = TX_BUF_SIZE,
        .rx_buffer_size = RX_BUF_SIZE,
    };
    esp_err_t err = usb_serial_jtag_driver_install(&drv_cfg);
    if (err != ESP_OK) return err;

    usb_serial_jtag_vfs_use_driver();

    BaseType_t ret = xTaskCreate(serial_read_task, "serial_prov",
                                 4096, NULL, 5, &s_serial_task);
    return (ret == pdPASS) ? ESP_OK : ESP_ERR_NO_MEM;
}

void serial_protocol_stop(void)
{
    if (s_serial_task) {
        vTaskDelete(s_serial_task);
        s_serial_task = NULL;
    }
}
```

- [ ] **Step 5: Add Kconfig entry for firmware version**

The receiver type is already determined by the existing `RECEIVER_RADIO_VARIANT` Kconfig choice. The `handle_identify` handler uses `device_config_receiver_type_string(device_config_compiled_receiver_type())` which returns `"RECEIVER_433M"` or `"RECEIVER_2_4G"` based on the compiled variant.

Only `CONFIG_RECEIVER_FIRMWARE_VERSION` is new. In `receiver-esp32/main/Kconfig.projbuild`, add inside the `menu "Receiver Firmware"` block:

```kconfig
config RECEIVER_FIRMWARE_VERSION
    string "Firmware version string"
    default "dev"
```

- [ ] **Step 6: Register new source file in CMakeLists.txt**

Add `serial/serial_protocol.c` to the `SRCS` list in `receiver-esp32/main/CMakeLists.txt`.

---

### Task 7: Receiver-ESP32 Main Integration (Additive — No Deletions)

**Files:**
- Modify: `receiver-esp32/main/main.c`

All existing provisioning files are kept (SoftAP + HTTP as backup). This task only adds the serial protocol alongside.

- [ ] **Step 1: Add serial include**

Add to main.c includes:

```c
#include "serial/serial_protocol.h"
```

Keep the existing `#include "provision/http_server.h"` — the HTTP server stays.

- [ ] **Step 2: Add `serial_protocol_init()` early in `app_main`**

Add after `device_config_load`, before the boot state switch:

```c
ESP_ERROR_CHECK(device_config_load(&g_cfg));

g_provision_events = xEventGroupCreate();
ESP_ERROR_CHECK(g_provision_events != NULL ? ESP_OK : ESP_ERR_NO_MEM);

ESP_ERROR_CHECK(serial_protocol_init());
```

The serial background task is now running. It will handle `provision`/`factory_reset`/`retry` by saving to NVS and rebooting directly. SoftAP + HTTP continues to work in parallel via the existing `run_provisioning_mode()` flow.

- [ ] **Step 3: Add `serial_protocol_stop()` before operational idle loop**

At the end of `app_main`, before the idle `for (;;)` loop, tear down the serial task since it's no longer needed:

```c
    restore_trigger_state();
    dispatch_operational_state();

    serial_protocol_stop();

    ESP_LOGI(APP_TAG, "Operational, idle loop running");
    for (;;) {
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
```

- [ ] **Step 4: No other changes**

The existing `run_provisioning_mode()` function, `g_provision_events`, SoftAP, HTTP server, and all provisioning files remain untouched. The serial task runs concurrently — whichever path receives a `provision` command first wins (serial reboots directly; HTTP signals the event group).

---

## Phase 4 — Receiver-ESP8266 Firmware

### Task 8: Create Serial Protocol for Receiver-ESP8266

**Files:**
- Create: `receiver-esp8266/main/serial/serial_protocol.h`
- Create: `receiver-esp8266/main/serial/serial_protocol.c`
- Modify: `receiver-esp8266/main/CMakeLists.txt` (add new source)

The ESP8266 version is functionally identical to Task 6 but uses the ESP8266 RTOS SDK's UART driver instead of USB Serial/JTAG.

- [ ] **Step 1: Create the header file**

Same as Task 6 Step 1 — identical `serial_protocol.h` (same struct, same API).

```c
/* receiver-esp8266/main/serial/serial_protocol.h */
#ifndef RECEIVER_SERIAL_PROTOCOL_H
#define RECEIVER_SERIAL_PROTOCOL_H

#include "config/device_config.h"
#include "esp_err.h"

typedef enum {
    SERIAL_PROV_RESULT_PROVISIONED,
    SERIAL_PROV_RESULT_RETRY,
    SERIAL_PROV_RESULT_RESET,
    SERIAL_PROV_RESULT_ERROR,
} serial_prov_result_t;

esp_err_t serial_protocol_init(void);

serial_prov_result_t serial_protocol_run_blocking(const device_config_t *cfg,
                                                  const char *recovery_reason);

#endif
```

- [ ] **Step 2: Create the implementation**

The implementation is the same as Task 6 Steps 2-4 with one difference: the `serial_protocol_init()` function uses the UART driver instead of USB Serial/JTAG.

Replace the init function:

```c
#include "driver/uart.h"

esp_err_t serial_protocol_init(void)
{
    uart_config_t uart_cfg = {
        .baud_rate = 115200,
        .data_bits = UART_DATA_8_BITS,
        .parity    = UART_PARITY_DISABLE,
        .stop_bits = UART_STOP_BITS_1,
        .flow_ctrl = UART_HW_FLOWCTRL_DISABLE,
    };
    esp_err_t err = uart_param_config(UART_NUM_0, &uart_cfg);
    if (err != ESP_OK) return err;

    return uart_driver_install(UART_NUM_0, 2048, 0, 0, NULL, 0);
}
```

The blocking read loop also changes — instead of `fgetc(stdin)`, read from UART:

```c
serial_prov_result_t serial_protocol_run_blocking(const device_config_t *cfg,
                                                  const char *recovery_reason)
{
    serial_prov_result_t result = SERIAL_PROV_RESULT_ERROR;
    s_line_pos = 0;
    uint8_t byte;

    for (;;) {
        int len = uart_read_bytes(UART_NUM_0, &byte, 1, pdMS_TO_TICKS(100));
        if (len <= 0) continue;

        char c = (char)byte;
        if (c == '\r') continue;
        if (c == '\n') {
            if (s_line_pos == 0) continue;
            s_line_buf[s_line_pos] = '\0';
            s_line_pos = 0;

            /* Parse and dispatch — identical to ESP32 version */
            // ... (identical JSON parse + dispatch code) ...
        }

        if (s_line_pos < LINE_BUF_SIZE - 1) {
            s_line_buf[s_line_pos++] = c;
        } else {
            ESP_LOGW(TAG, "Line buffer overflow, discarding");
            s_line_pos = 0;
        }
    }

    return result;
}
```

For sending responses, use `uart_write_bytes` instead of `printf`:

```c
static void send_json(cJSON *root)
{
    char *str = cJSON_PrintUnformatted(root);
    if (str) {
        size_t len = strlen(str);
        uart_write_bytes(UART_NUM_0, str, len);
        uart_write_bytes(UART_NUM_0, "\n", 1);
        uart_wait_tx_done(UART_NUM_0, pdMS_TO_TICKS(500));
        cJSON_free(str);
    }
    cJSON_Delete(root);
}
```

Copy all command handlers from Task 6 Step 3 verbatim — they use the same `device_config` and `cJSON` APIs. Adapt function names if the ESP8266 receiver has different config API names (check `receiver-esp8266/main/config/device_config.h` for exact signatures).

- [ ] **Step 3: Register new source file in CMakeLists.txt**

Add `serial/serial_protocol.c` to the `SRCS` list in `receiver-esp8266/main/CMakeLists.txt`.

---

### Task 9: Receiver-ESP8266 Main Integration and Cleanup

**Files:**
- Modify: `receiver-esp8266/main/main.c`
- Delete: `receiver-esp8266/main/provision/http_server.c`, `http_server.h`
- Delete: `receiver-esp8266/main/provision/recovery.c`, `recovery.h`
- Delete: `receiver-esp8266/main/provision/index.html`, `index.html.gz`
- Modify: `receiver-esp8266/main/network/wifi.h` (remove `wifi_start_softap`)
- Modify: `receiver-esp8266/main/network/wifi.c` (remove `wifi_start_softap`)
- Modify: `receiver-esp8266/main/CMakeLists.txt` (remove deleted sources)
- Modify: `receiver-esp8266/sdkconfig`

- [ ] **Step 1: Update main.c includes**

Replace:
```c
#include "provision/recovery.h"
```

With:
```c
#include "serial/serial_protocol.h"
```

- [ ] **Step 2: Replace `handle_recovery_result` and recovery calls**

Remove the `handle_recovery_result` function (lines 181-186). Replace all `provision_run_recovery_mode()` calls with the serial protocol pattern.

Create a replacement function:

```c
static void run_serial_provisioning(const char *reason)
{
    ESP_LOGW(MAIN_TAG, "Entering serial provisioning: %s", reason);

    serial_prov_result_t result = serial_protocol_run_blocking(&cfg, reason);

    if (result == SERIAL_PROV_RESULT_RESET) {
        device_config_erase_runtime();
    }

    restart_after_response_flush();
}
```

Then replace each recovery call in `app_main`. For example:

```c
if (!device_config_is_provisioned(&cfg)) {
    ESP_LOGW(MAIN_TAG, "Not provisioned, entering serial provisioning");
    run_serial_provisioning("UNPROVISIONED");
    device_config_free(&cfg);
    continue;
}
```

Apply the same pattern to all five `provision_run_recovery_mode()` call sites (UNPROVISIONED, MISSING_ENROLL_TOKEN, WIFI_FAILED, MQTT_FAILED, BOOTSTRAP_FAILED).

- [ ] **Step 3: Add `serial_protocol_init()` call**

In `app_main`, add after `init_platform_once()`:

```c
ESP_ERROR_CHECK(init_platform_once());
ESP_ERROR_CHECK(serial_protocol_init());
ESP_LOGI(MAIN_TAG, "Serial provisioning initialized");
```

- [ ] **Step 4: Remove deleted files**

Delete these files:
- `receiver-esp8266/main/provision/http_server.c`
- `receiver-esp8266/main/provision/http_server.h`
- `receiver-esp8266/main/provision/recovery.c`
- `receiver-esp8266/main/provision/recovery.h`
- `receiver-esp8266/main/provision/index.html`
- `receiver-esp8266/main/provision/index.html.gz`

Remove `provision/http_server.c` and `provision/recovery.c` from `CMakeLists.txt` SRCS list.

- [ ] **Step 5: Remove `wifi_start_softap`**

In `receiver-esp8266/main/network/wifi.h`, remove the `wifi_start_softap()` declaration.
In `receiver-esp8266/main/network/wifi.c`, remove the `wifi_start_softap()` function.
In `receiver-esp8266/main/Kconfig.projbuild`, remove `RECEIVER_AP_CHANNEL` if present.

- [ ] **Step 6: Update sdkconfig baud rates**

In `receiver-esp8266/sdkconfig`, set:

```
CONFIG_ESP_CONSOLE_UART_BAUDRATE=115200
CONFIG_ESPTOOLPY_MONITOR_BAUD=115200
```

---

## Phase 5 — Verification

### Task 10: Frontend Lint and Type Check

- [ ] **Step 1: Run lint**

```bash
cd web && yarn lint
```

Fix any issues found.

- [ ] **Step 2: Log changes**

Add a summary of all changes to `docs/CHANGELOGS.md`.
