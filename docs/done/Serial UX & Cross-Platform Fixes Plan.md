# Serial UX & Cross-Platform Fixes — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix cross-platform flag rendering, add password toggles to provisioning dialog, fix ESP32-C3 provision timeout, fix transmitter firmware version copy-paste bug, add ESP8266 Wi-Fi debug logging, and extract a shared serial console component for the provisioning dialog.

**Architecture:** Six independent tasks touching `web/`, `receiver-esp32/`, `receiver-esp8266/`, and `transmitter/`. Web changes follow existing shadcn/ui + Tailwind patterns and bilingual i18n. Firmware changes are single-line or small-block edits.

**Tech Stack:** Next.js 16, React 19, TypeScript 5, Tailwind CSS 4, country-flag-icons, shadcn/ui, ESP-IDF v6.0 (C), ESP8266 RTOS SDK (C)

**Spec:** `docs/spec/Serial UX & Cross-Platform Fixes Spec.md`

---

### Task 1: Flag Icons — Replace Unicode Emoji with SVG Components

**Files:**
- Modify: `web/package.json`
- Modify: `web/src/components/layout/language-switcher.tsx`

- [ ] **Step 1: Install country-flag-icons**

```bash
cd web && yarn add country-flag-icons
```

- [ ] **Step 2: Rewrite language-switcher.tsx**

Replace the entire file content:

```tsx
"use client";

import { GB, VN } from "country-flag-icons/react/3x2";
import { useLocale } from "next-intl";
import type { FC, SVGProps } from "react";
import { useTransition } from "react";
import { Toggle } from "@/components/ui/toggle";
import { usePathname, useRouter } from "@/i18n/navigation";
import { cn } from "@/lib/utils";

const LOCALE_META: Record<
  string,
  { Flag: FC<SVGProps<SVGSVGElement>>; label: string }
> = {
  vi: { Flag: VN, label: "Tiếng Việt" },
  en: { Flag: GB, label: "English" },
};

type Locale = keyof typeof LOCALE_META;

export function LanguageSwitcher() {
  const locale = useLocale() as Locale;
  const pathname = usePathname();
  const router = useRouter();
  const [isPending, startTransition] = useTransition();

  const nextLocale: Locale = locale === "vi" ? "en" : "vi";
  const { Flag } = LOCALE_META[locale];

  function handlePress() {
    startTransition(() => {
      router.replace(pathname, { locale: nextLocale });
    });
  }

  return (
    <Toggle
      variant="outline"
      size="sm"
      pressed={false}
      onPressedChange={handlePress}
      aria-label={LOCALE_META[nextLocale].label}
      className={cn(
        "gap-1.5 rounded-full px-2.5",
        isPending && "pointer-events-none opacity-60",
      )}
    >
      <Flag aria-hidden="true" className="h-3.5 w-5 rounded-sm" />
      <span className="text-xs font-semibold">{locale.toUpperCase()}</span>
    </Toggle>
  );
}
```

Key changes:
- `country-flag-icons/react/3x2` exports `VN` and `GB` as React SVG components — tree-shaken, only these two are bundled (~2 KB).
- `Flag` component receives standard SVG props. Sized with `h-3.5 w-5` (3:2 aspect ratio matching the component library). `rounded-sm` clips flag corners to blend with the Toggle shape.
- `LOCALE_META.flag` emoji field removed; replaced with `Flag` component reference.
- No global CSS import needed.

- [ ] **Step 3: Verify in browser**

```bash
cd web && yarn dev
```

Open the dashboard in a Chromium browser on Windows (or any browser). Confirm:
- VN flag renders as a colored SVG, not as "VN" text
- GB flag renders as a colored SVG, not as "GB" text
- Toggle click switches language and flag
- Pending state (opacity transition) works
- Size looks proportionate inside the Toggle button

---

### Task 2: Password Show/Hide + Dialog Expansion

**Files:**
- Modify: `web/src/features/device/usb-provision-dialog.tsx`
- Modify: `web/src/messages/en.json`
- Modify: `web/src/messages/vi.json`

- [ ] **Step 1: Add i18n keys**

In `web/src/messages/en.json`, inside the `"devices" > "usb"` object, add after the `"mqtt_password"` key:

```json
"showPassword": "Show password",
"hidePassword": "Hide password",
```

In `web/src/messages/vi.json`, same location:

```json
"showPassword": "Hiện mật khẩu",
"hidePassword": "Ẩn mật khẩu",
```

- [ ] **Step 2: Add imports and state to usb-provision-dialog.tsx**

Add `Eye` and `EyeOff` to the `lucide-react` import (line 1):

```tsx
import { Check, Eye, EyeOff, Loader2, Usb, Wifi, WifiOff } from "lucide-react";
```

Add two state booleans after `const [errorMessage, setErrorMessage] = useState("");` (around line 118):

```tsx
const [showWifiPwd, setShowWifiPwd] = useState(false);
const [showMqttPwd, setShowMqttPwd] = useState(false);
```

- [ ] **Step 3: Expand dialog width**

Change the `DialogContent` className from `max-w-lg` to `max-w-2xl`:

```tsx
<DialogContent className="max-w-2xl">
```

- [ ] **Step 4: Wrap WiFi password field with toggle**

Replace the WiFi password field block (lines ~462-471):

```tsx
<div className="space-y-2">
  <Label htmlFor="usb-wifi-pwd">{tUsb("wifi_password")}</Label>
  <div className="relative">
    <Input
      id="usb-wifi-pwd"
      type={showWifiPwd ? "text" : "password"}
      value={wifiPwd}
      onChange={(e) => setWifiPwd(e.target.value)}
      maxLength={64}
    />
    <button
      type="button"
      onClick={() => setShowWifiPwd((v) => !v)}
      className="absolute top-1/2 right-3 -translate-y-1/2 text-muted-foreground hover:text-foreground"
      aria-label={
        showWifiPwd ? tUsb("hidePassword") : tUsb("showPassword")
      }
    >
      {showWifiPwd ? (
        <EyeOff aria-hidden="true" className="size-4" />
      ) : (
        <Eye aria-hidden="true" className="size-4" />
      )}
    </button>
  </div>
</div>
```

- [ ] **Step 5: Wrap MQTT password field with toggle**

Replace the MQTT password field block (lines ~503-516):

```tsx
<div className="space-y-2">
  <Label htmlFor="usb-mqtt-pwd">{tUsb("mqtt_password")}</Label>
  <div className="relative">
    <Input
      id="usb-mqtt-pwd"
      type={showMqttPwd ? "text" : "password"}
      value={mqttPwd}
      onChange={(e) => setMqttPwd(e.target.value)}
      maxLength={127}
      aria-invalid={!!errors.mqttPwd}
    />
    <button
      type="button"
      onClick={() => setShowMqttPwd((v) => !v)}
      className="absolute top-1/2 right-3 -translate-y-1/2 text-muted-foreground hover:text-foreground"
      aria-label={
        showMqttPwd ? tUsb("hidePassword") : tUsb("showPassword")
      }
    >
      {showMqttPwd ? (
        <EyeOff aria-hidden="true" className="size-4" />
      ) : (
        <Eye aria-hidden="true" className="size-4" />
      )}
    </button>
  </div>
  {errors.mqttPwd && (
    <InlineError message={errors.mqttPwd} className="mt-1" />
  )}
</div>
```

- [ ] **Step 6: Verify**

Open provisioning dialog. Confirm:
- Both password fields show Eye icon by default (password masked)
- Clicking Eye toggles to EyeOff (password visible) and back
- Dialog is wider (`max-w-2xl`) — form fields have more breathing room
- MQTT validation error still appears below the field
- aria-labels toggle correctly (inspect with dev tools)

---

### Task 3: ESP32-C3 Provision Timeout Fix

**Files:**
- Modify: `receiver-esp32/main/serial/serial_protocol.c`
- Modify: `web/src/features/device/usb-provision-dialog.tsx`

- [ ] **Step 1: Firmware — inline restart in handle_provision**

In `receiver-esp32/main/serial/serial_protocol.c`, modify `handle_provision` (line 140). After the `send_response` call and before `return true;`, add the inline restart and change the return:

Replace lines 183-186:
```c
    cJSON *resp = cJSON_CreateObject();
    cJSON_AddBoolToObject(resp, "restarting", true);
    send_response(id, true, resp, NULL);
    return true;
```

With:
```c
    cJSON *resp = cJSON_CreateObject();
    cJSON_AddBoolToObject(resp, "restarting", true);
    send_response(id, true, resp, NULL);

    usb_serial_jtag_wait_tx_done(pdMS_TO_TICKS(250));
    esp_restart();
    return false;
```

This matches the transmitter pattern: restart inline after sending response, so `dispatch_command` never reaches its own restart block for this command.

- [ ] **Step 2: Web — make provision timeout non-fatal**

In `web/src/features/device/usb-provision-dialog.tsx`, modify the `handleProvision` function (line 230). Replace the bare `await sendCommand("provision", ...)` with a try-catch that tolerates timeout/disconnect:

Replace lines 244-253:
```tsx
      setStep("provisioning");

      await sendCommand("provision", {
        wifi_ssid: wifiSsid,
        wifi_pwd: wifiPwd || undefined,
        mqtt_uri: mqttUri,
        mqtt_user: mqttUser,
        mqtt_pwd: mqttPwd,
        enroll_token: tokenResult.token,
      });
```

With:
```tsx
      setStep("provisioning");

      try {
        await sendCommand("provision", {
          wifi_ssid: wifiSsid,
          wifi_pwd: wifiPwd || undefined,
          mqtt_uri: mqttUri,
          mqtt_user: mqttUser,
          mqtt_pwd: mqttPwd,
          enroll_token: tokenResult.token,
        });
      } catch (provisionErr) {
        const msg =
          provisionErr instanceof Error ? provisionErr.message : "";
        if (
          msg.startsWith("serial_timeout") ||
          msg === "serial_disconnected"
        ) {
          // Device restarted before response arrived — expected
        } else {
          throw provisionErr;
        }
      }
```

The `setStep("restarting")` and `waitForRestartReconnect()` on the next lines remain unchanged — the flow continues whether the response arrived or the device already restarted.

- [ ] **Step 3: Verify**

Flash the updated receiver-esp32 firmware and test provisioning from the web dashboard. Confirm:
- Provisioning completes without `serial_timeout` error
- The stepper progresses through all steps to "Done"
- If the response does arrive before restart, the flow works identically to before

---

### Task 4: Transmitter Firmware Version Copy-Paste Bug

**Files:**
- Modify: `transmitter/main/network/mqtt.c:84`

- [ ] **Step 1: Fix the macro name**

In `transmitter/main/network/mqtt.c`, line 84, replace:

```c
cJSON_AddStringToObject(root, "firmware_version", CONFIG_RECEIVER_FIRMWARE_VERSION);
```

With:

```c
cJSON_AddStringToObject(root, "firmware_version", CONFIG_TRANSMITTER_FIRMWARE_VERSION);
```

- [ ] **Step 2: Verify compilation**

```bash
cd transmitter && idf.py build
```

Confirm build succeeds with no warnings about undefined macros. The `CONFIG_TRANSMITTER_FIRMWARE_VERSION` macro is defined in `transmitter/sdkconfig` as `"v1.0"`.

---

### Task 5: ESP8266 Wi-Fi Test Debug Logging

**Files:**
- Modify: `receiver-esp8266/main/serial/serial_protocol.c`

- [ ] **Step 1: Add ESP_LOGI after password extraction**

In `receiver-esp8266/main/serial/serial_protocol.c`, in `handle_test_wifi` (line 173), add a log line after `const char *pwd = json_get_string(...)` (line 185) and before the `wifi_start_sta_test` call (line 187):

Insert after line 185:
```c
    ESP_LOGI(TAG, "wifi test: ssid=%s pwd_len=%d", ssid, pwd ? (int)strlen(pwd) : 0);
```

The resulting block should read:

```c
    const char *pwd = json_get_string(cmd_payload, "wifi_pwd");

    ESP_LOGI(TAG, "wifi test: ssid=%s pwd_len=%d", ssid, pwd ? (int)strlen(pwd) : 0);

    esp_err_t err = wifi_start_sta_test(ssid, pwd, pdMS_TO_TICKS(15000));
```

This log goes to UART0 as `I (xxxx) SERIAL: wifi test: ssid=MyNetwork pwd_len=12` and will be visible in the serial console (Task 6).

- [ ] **Step 2: Verify the TAG macro exists**

Confirm `TAG` is defined at the top of the file (it should be `static const char *TAG = "SERIAL";` or similar). If not, check the exact tag name and use it.

```bash
grep -n "TAG" receiver-esp8266/main/serial/serial_protocol.c | head -5
```

- [ ] **Step 3: Build**

```bash
cd receiver-esp8266 && idf.py build
```

---

### Task 6: Serial Console — Extract Component + Add to Provisioning Dialog

**Files:**
- Create: `web/src/features/device/serial-console.tsx`
- Modify: `web/src/features/device/usb-control-panel.tsx`
- Modify: `web/src/features/device/usb-provision-dialog.tsx`
- Modify: `web/src/lib/serial/use-serial.ts`

- [ ] **Step 1: Expose protocol events from useSerial**

The `useSerial` hook already exposes `events: EventTarget` in its return type (line 39 of `use-serial.ts`). Verify this is wired through — the provisioning dialog destructures `useSerial()` at line 91 but does NOT currently destructure `events`. It will need to.

No file changes needed for this step — just confirm `events` is available. It is: `UseSerialReturn` includes `events: EventTarget` at line 39.

- [ ] **Step 2: Create serial-console.tsx**

Create `web/src/features/device/serial-console.tsx`:

```tsx
"use client";

import { ChevronDown, Pause, Play, Terminal } from "lucide-react";
import { useTranslations } from "next-intl";
import { useCallback, useEffect, useRef, useState } from "react";
import { Button } from "@/components/ui/button";
import {
  Collapsible,
  CollapsibleContent,
  CollapsibleTrigger,
} from "@/components/ui/collapsible";
import "@/styles/usb-console.css";

interface ConsoleEntry {
  timestamp: number;
  text: string;
  level: string;
}

function detectLogLevel(line: string): string {
  if (line.startsWith("E ") || line.startsWith("E (")) return "E";
  if (line.startsWith("W ") || line.startsWith("W (")) return "W";
  if (line.startsWith("I ") || line.startsWith("I (")) return "I";
  if (line.startsWith("D ") || line.startsWith("D (")) return "D";
  return "";
}

interface SerialConsoleProps {
  events: EventTarget;
}

export function SerialConsole({ events }: SerialConsoleProps) {
  const tUsb = useTranslations("devices.usb");

  const [consoleLogs, setConsoleLogs] = useState<ConsoleEntry[]>([]);
  const [consolePaused, setConsolePaused] = useState(false);
  const [consoleOpen, setConsoleOpen] = useState(false);
  const consoleEndRef = useRef<HTMLDivElement>(null);

  const addConsoleLog = useCallback(
    (text: string) => {
      if (consolePaused) return;
      setConsoleLogs((prev) => {
        const next = [
          ...prev,
          { timestamp: Date.now(), text, level: detectLogLevel(text) },
        ];
        return next.length > 500 ? next.slice(-500) : next;
      });
      requestAnimationFrame(() => {
        consoleEndRef.current?.scrollIntoView({ behavior: "smooth" });
      });
    },
    [consolePaused],
  );

  useEffect(() => {
    const handleConsole = (e: Event) => {
      addConsoleLog((e as CustomEvent).detail);
    };
    events.addEventListener("console.log", handleConsole);
    return () => events.removeEventListener("console.log", handleConsole);
  }, [events, addConsoleLog]);

  useEffect(() => {
    if (!consolePaused) {
      consoleEndRef.current?.scrollIntoView({ behavior: "smooth" });
    }
  }, [consolePaused]);

  return (
    <Collapsible open={consoleOpen} onOpenChange={setConsoleOpen}>
      <CollapsibleTrigger className="flex items-center gap-2 text-sm text-muted-foreground hover:text-foreground">
        <Terminal aria-hidden="true" className="size-4" />
        <span>{tUsb("console_log")}</span>
        <ChevronDown
          aria-hidden="true"
          className="size-3.5 transition-transform [[data-panel-open]_&]:rotate-180"
        />
      </CollapsibleTrigger>
      <CollapsibleContent>
        <div className="mt-2">
          <div className="mb-1 flex items-center justify-end">
            <Button
              variant="ghost"
              size="sm"
              onClick={() => setConsolePaused(!consolePaused)}
              className="h-6 px-2 text-xs"
            >
              {consolePaused ? (
                <Play aria-hidden="true" className="mr-1 size-3" />
              ) : (
                <Pause aria-hidden="true" className="mr-1 size-3" />
              )}
              {consolePaused
                ? tUsb("console_resume")
                : tUsb("console_pause")}
            </Button>
          </div>
          <div className="usb-console">
            {consoleLogs.map((entry) => (
              <div
                key={entry.timestamp}
                className="usb-console-line"
                data-level={entry.level}
              >
                {entry.text}
              </div>
            ))}
            <div ref={consoleEndRef} />
          </div>
        </div>
      </CollapsibleContent>
    </Collapsible>
  );
}
```

This is a direct extraction from `usb-control-panel.tsx` lines 46-64 (types + detectLogLevel), 89-96 (state), 102-103 (ref), 131-164 (addConsoleLog + event listener + scroll effect), and 488-530 (JSX). The interface takes `events: EventTarget` from the `useSerial` hook.

- [ ] **Step 3: Refactor usb-control-panel.tsx to use SerialConsole**

In `web/src/features/device/usb-control-panel.tsx`:

**3a. Add import:**
```tsx
import { SerialConsole } from "./serial-console";
```

**3b. Remove extracted code:**
- Remove the `ConsoleEntry` interface (lines 46-50) — now in `serial-console.tsx`
- Remove `detectLogLevel` function (lines 58-64) — now in `serial-console.tsx`
- Remove state: `consoleLogs`, `consolePaused`, `consoleOpen` (lines 89, 91, 92)
- Remove `consoleEndRef` (line 102)
- Remove `addConsoleLog` callback (lines 131-146)
- Remove the `console.log` event listener `useEffect` (lines 158-164)
- Remove the `consolePaused` scroll `useEffect` (lines 192-196)
- Remove the `@/styles/usb-console.css` import (line 37) — now imported by `serial-console.tsx`

**3c. Replace the Collapsible console JSX** (lines 488-530) with:

```tsx
<SerialConsole events={events} />
```

Keep the `EventEntry` interface and all event-log related code (`eventLog`, `addEvent`, event listener useEffect, events Card JSX) — those are transmitter-specific and stay in `UsbControlPanel`.

- [ ] **Step 4: Add SerialConsole to usb-provision-dialog.tsx**

**4a. Add imports:**

```tsx
import { SerialConsole } from "./serial-console";
```

**4b. Destructure `events` from useSerial:**

Change line 91-102 to include `events`:

```tsx
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
    events,
  } = useSerial();
```

**4c. Add SerialConsole below the form/stepper content:**

Place it after the stepper section and before the closing `</DialogContent>`. Find the closing `</DialogContent>` tag and insert before it:

```tsx
        {portState === "open" && (
          <div className="border-t pt-4">
            <SerialConsole events={events} />
          </div>
        )}
```

This shows the console only when a serial connection is active, separated from the form content by a top border.

- [ ] **Step 5: Verify**

Open the provisioning dialog, connect a device. Confirm:
- Console Output collapsible appears below the form
- ESP log lines (boot messages, WiFi status) appear with color coding
- Pause/Resume buttons work
- Auto-scroll follows new entries
- The console survives through all provisioning steps (form → stepper → done)
- The transmitter device detail page's console still works (refactored to use same component)

---

## Verification Checklist

After all tasks, verify:

- [ ] `cd web && yarn lint` passes
- [ ] `cd web && yarn build` passes (type-check + build)
- [ ] Flag icons render on Windows Chromium (no "VN"/"GB" text)
- [ ] Password toggle works on both WiFi and MQTT fields
- [ ] Provisioning dialog is wider and has good spacing
- [ ] ESP32-C3 provisioning completes without timeout errors
- [ ] Serial console shows ESP log lines during provisioning
- [ ] Transmitter device detail console still works after refactor
- [ ] Transmitter firmware version registers as "v1.0" (not "dev") via MQTT
- [ ] ESP8266 logs password length during WiFi test

## File Change Summary

| File | Action | Task |
|------|--------|------|
| `web/package.json` | Add `country-flag-icons` | 1 |
| `web/src/components/layout/language-switcher.tsx` | Rewrite (emoji → SVG) | 1 |
| `web/src/messages/en.json` | Add 2 keys in `devices.usb` | 2 |
| `web/src/messages/vi.json` | Add 2 keys in `devices.usb` | 2 |
| `web/src/features/device/usb-provision-dialog.tsx` | Password toggles, dialog width, provision timeout resilience, serial console | 2, 3, 6 |
| `web/src/features/device/serial-console.tsx` | Create (extracted from control panel) | 6 |
| `web/src/features/device/usb-control-panel.tsx` | Refactor (use SerialConsole) | 6 |
| `receiver-esp32/main/serial/serial_protocol.c` | Inline restart in handle_provision | 3 |
| `transmitter/main/network/mqtt.c` | Fix CONFIG macro name | 4 |
| `receiver-esp8266/main/serial/serial_protocol.c` | Add ESP_LOGI for password length | 5 |
