# Serial UX & Cross-Platform Fixes Spec

Six independent fixes targeting the USB provisioning flow, firmware reporting, cross-platform rendering, and serial diagnostics.

---

## 1. Flag Icons — Windows Chromium Rendering

**Problem:** Unicode regional-indicator flag emoji (🇻🇳 🇬🇧) render as two-letter codes (VN, GB) on Windows Chromium because Windows lacks flag emoji glyphs.

**Fix:** Replace emoji with `country-flag-icons` React SVG components (tree-shakeable, only VN + GB bundled).

### Changes


| File                                              | Change                                                                                                                                                                                                                              |
| ------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `web/package.json`                                | Add `country-flag-icons` dependency                                                                                                                                                                                                 |
| `web/src/components/layout/language-switcher.tsx` | Import `{ VN, GB }` from `country-flag-icons/react/3x2`; replace `<span className="text-base leading-none">{LOCALE_META[locale].flag}</span>` with the corresponding React component. Map locale to component: `{ vi: VN, en: GB }` |


### Notes

- The `LOCALE_META` object can drop the `flag` field since flags are now React components.
- `country-flag-icons/react/3x2` exports each flag as an SVG React component. Only the two imported flags are bundled (~2KB).
- Size with Tailwind `className="h-4 w-5"` (3:2 aspect ratio) to match the current `text-base` emoji sizing.
- No global CSS import needed — these are self-contained SVG components.

---

## 2. Password Show/Hide + Dialog Expansion

**Problem:** WiFi and MQTT password fields in the USB provisioning dialog have no visibility toggle. Dialog is narrow (`max-w-lg`).

**Fix:** Add Eye/EyeOff toggle to both password fields following the existing pattern in `create-admin-dialog.tsx`. Widen dialog.

### Changes


| File                                               | Change                                                                                                                                             |
| -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `web/src/features/device/usb-provision-dialog.tsx` | Add `showWifiPwd` + `showMqttPwd` state; wrap password inputs in `<div className="relative">` with toggle button; change `max-w-lg` to `max-w-2xl` |
| `web/src/messages/en.json`                         | Add `showPassword` / `hidePassword` keys under the USB provisioning namespace                                                                      |
| `web/src/messages/vi.json`                         | Add Vietnamese equivalents                                                                                                                         |


### Pattern Reference

From `create-admin-dialog.tsx` lines 207-238:

```tsx
<div className="relative">
  <Input
    type={showPassword ? "text" : "password"}
    ...
  />
  <button
    type="button"
    onClick={() => setShowPassword((v) => !v)}
    className="absolute top-1/2 right-3 -translate-y-1/2 text-muted-foreground hover:text-foreground"
    aria-label={showPassword ? t("hidePassword") : t("showPassword")}
  >
    {showPassword ? <EyeOff className="size-4" /> : <Eye className="size-4" />}
  </button>
</div>
```

Apply to both `wifiPwd` and `mqttPwd` fields with independent state booleans.

---

## 3. ESP32-C3 Provision Serial Timeout

**Problem:** `serial_timeout: provision` error on the web side when provisioning ESP32-C3 receivers. The device sends the response, then restarts — but the USB disconnect can race with the browser reading the response, causing a 10s timeout.

**Root cause:** The receiver defers `esp_restart()` to `dispatch_command` (after `cJSON_Delete`), while the transmitter handles it inline. The response may not be fully read by the host before the USB device disappears.

### Firmware Fix (receiver-esp32)

Adapt the transmitter's inline restart pattern in `receiver-esp32/main/serial/serial_protocol.c`:

**Current (receiver):**

```c
// handle_provision returns true → dispatch_command calls restart
static bool handle_provision(const char *id, const cJSON *cmd_payload) {
    // ... NVS writes ...
    send_response(id, true, resp, NULL);
    return true;
}

// In dispatch_command:
if (needs_restart) {
    usb_serial_jtag_wait_tx_done(pdMS_TO_TICKS(250));
    esp_restart();
}
```

**Target (match transmitter pattern):**

```c
// handle_provision does restart inline, returns false (no deferred restart)
static bool handle_provision(const char *id, const cJSON *cmd_payload) {
    // ... NVS writes ...
    send_response(id, true, resp, NULL);
    usb_serial_jtag_wait_tx_done(pdMS_TO_TICKS(250));
    esp_restart();
    return false; // unreachable but satisfies signature
}
```

Keep the deferred restart pattern in `dispatch_command` for `factory_reset` and `retry`.

### Web Fix (provision timeout resilience)

In `usb-provision-dialog.tsx`, catch `serial_timeout` and `serial_disconnected` errors from the `provision` command specifically and treat them as "device is restarting" instead of a fatal error:

```tsx
// In handleProvision, around the sendCommand("provision", ...) call:
try {
  await sendCommand("provision", { ... });
} catch (err) {
  if (
    err instanceof Error &&
    (err.message.startsWith("serial_timeout") ||
     err.message === "serial_disconnected")
  ) {
    // Device likely restarted before we read the response — expected
  } else {
    throw err; // Re-throw unexpected errors
  }
}

setStep("restarting");
await waitForRestartReconnect();
```

---

## 4. Firmware Version "dev" in MQTT Registration — HAND-IMPLEMENTED

**Status:** Hand-implemented by user. Receiver-ESP32 verified correct. **Transmitter has copy-paste bug**: uses `CONFIG_RECEIVER_FIRMWARE_VERSION` (undefined in transmitter sdkconfig) instead of `CONFIG_TRANSMITTER_FIRMWARE_VERSION`.

### Verification

| File                                 | Line | Status | Issue |
| ------------------------------------ | ---- | ------ | ----- |
| `receiver-esp32/main/network/mqtt.c` | 211  | OK     | Correctly uses `CONFIG_RECEIVER_FIRMWARE_VERSION` |
| `transmitter/main/network/mqtt.c`    | 84   | **BUG** | Uses `CONFIG_RECEIVER_FIRMWARE_VERSION` — must be `CONFIG_TRANSMITTER_FIRMWARE_VERSION` |

ESP8266 already passes firmware version via `mqtt_receiver_init()` parameter — no change needed.

---

## 5. ESP8266 Wi-Fi Test Diagnostics

~~5a. PMF Mismatch — hand-fixed by user (synced test vs prod PMF config).~~
~~5b. Baud rate reduction — dropped; keeping 115200.~~

Wi-Fi test failure root cause still unknown. Baud rate stays at 115200. Instead, add diagnostic logging to help investigate via the serial console (section 6).

### 5c. Debug Logging for Wi-Fi Test

Add an `ESP_LOGI` call in the ESP8266 `handle_test_wifi` handler to log the received password size immediately after parsing. This helps confirm whether the password is arriving intact over serial.

| File | Change |
|------|--------|
| `receiver-esp8266/main/serial/serial_protocol.c` | In `handle_test_wifi`, after extracting `pwd` from the JSON payload, add: `ESP_LOGI(TAG, "wifi test: ssid=%s pwd_len=%d", ssid, pwd ? (int)strlen(pwd) : 0);` |

This log line will appear in the serial console during provisioning (section 6), since ESP log output shares UART0 with the JSON protocol. The web side's `SerialProtocol` routes non-JSON lines to the `console.log` event automatically.

---

## 6. Serial Console in Provisioning Dialog

**Problem:** During provisioning, device logs (Wi-Fi connection attempts, MQTT errors, boot messages) are invisible. Debugging requires a separate serial monitor, which conflicts with the Web Serial session.

**Solution:** The `SerialProtocol` already dispatches `console.log` events for all non-JSON serial output (ESP log lines like `I (1234) WIFI: connected`). Add a console panel to the provisioning dialog that subscribes to these events.

### Extract `SerialConsole` Component

Extract the console rendering logic from `UsbControlPanel` into a reusable component:

| File | Purpose |
|------|---------|
| `web/src/features/device/serial-console.tsx` | New shared component |
| `web/src/features/device/usb-control-panel.tsx` | Refactor to use `SerialConsole` |
| `web/src/features/device/usb-provision-dialog.tsx` | Add `SerialConsole` |

**Component interface:**

```tsx
interface SerialConsoleProps {
  protocol: SerialProtocol;
}
```

**Extracted logic (from `UsbControlPanel`):**
- `consoleLogs` state with `ConsoleEntry[]` type (`{ text, level, timestamp }`)
- `detectLogLevel()` function (parses ESP log prefixes: E, W, I, D)
- `console.log` event listener on `protocol.events`
- 500-line buffer cap (`slice(-500)`)
- Auto-scroll via `ref` + `requestAnimationFrame` + `scrollIntoView`
- Pause/Resume toggle button
- Collapsible wrapper (shadcn `Collapsible`)
- Reuses existing `usb-console.css` styles (monospace, level-based colors)

### Integration in Provisioning Dialog

- Place `<SerialConsole protocol={protocol} />` below the form section and stepper
- Visible during all dialog steps — logs accumulate across the entire session
- The `protocol` instance comes from the existing `useSerial` hook (already available in the dialog via `sendCommand` / `protocolRef`)
- The `useSerial` hook needs to expose `protocolRef.current` (or the `events` EventTarget) so the console can subscribe

### Refactor `UsbControlPanel`

Replace the inline console code (~60 lines: state, event listener, render) with `<SerialConsole protocol={protocol} />`. The event log panel (wifi_connected, mqtt_connected, etc.) stays in `UsbControlPanel` since it's specific to the transmitter control flow.

---

## Scope Summary


| #   | Item                          | Repos Touched               | Risk                                            |
| --- | ----------------------------- | --------------------------- | ----------------------------------------------- |
| 1   | Flag icons                    | web                         | Low — cosmetic                                  |
| 2   | Password toggle + dialog size | web                         | Low — UI only                                   |
| 3   | Provision timeout             | receiver-esp32, web         | Medium — firmware restart path + error handling |
| 4   | Firmware version (done)       | transmitter (bug remaining) | Low — copy-paste fix                            |
| 5a  | PMF fix (done)                | —                           | Hand-fixed by user                              |
| 5b  | Baud rate (dropped)           | —                           | Keeping 115200                                  |
| 5c  | Wi-Fi debug logging           | receiver-esp8266            | Low — one ESP_LOGI call                         |
| 6   | Serial console in provisioning | web                        | Low — extract + reuse existing pattern          |


