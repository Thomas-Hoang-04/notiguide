# Transmitter OLED Header & Clock Rework Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Declutter the transmitter's OLED header into an icon-based status bar with a real-time NTP clock, and slim the Network Info screen so nothing is clipped.

**Architecture:** Wire the existing `main/time/time_sync.c` SNTP sample into the build; it sets the ESP32 system clock on Wi-Fi got-IP. The existing `time_utils` already reads the system clock first, so dispatch timestamps converge automatically; a backend ISO-8601 timestamp also bootstraps the clock (guarded). The status bar reads the system clock directly for an `HH:MM` local-time readout, switches MQTT/WiFi/USB to a consistent icon+word layout (style C′), and the Network Info screen drops the duplicated `Hub:` row (style N1).

**Tech Stack:** ESP-IDF v6.0, LVGL 8.3.11 (SSD1306 128×64, 1-bpp), lwIP SNTP, newlib time (`setenv`/`tzset`/`localtime_r`).

**Spec:** `docs/spec/Transmitter OLED Header & Clock Rework Spec.md`

---

## Working conventions for this plan (read first)

These deviate from the generic writing-plans template, deliberately, to match this repo's rules:

- **No git-commit steps.** Per project convention the executor decides when to commit; this plan never commits.
- **No build/flash steps.** Building and on-device flashing are handled by the project's separate audit/flash flow. Do **not** run `idf.py build`. The final task lists the on-device acceptance checks to be performed in that flow.
- **No host unit tests (no TDD red-green).** The transmitter firmware has no host test harness, and the changed code is LVGL/ESP-IDF-coupled (`gettimeofday`, `lv_label_set_text`, panel I/O) that cannot run off-target. "Verification" per task is therefore: (a) **GitNexus impact analysis before editing any symbol** (mandatory repo rule), (b) inspection against the exact code below, and (c) **`gitnexus_detect_changes` at the end** to confirm scope. On-device behavior is validated in Task 7.
- **GitNexus is mandatory.** `transmitter/CLAUDE.md` requires `gitnexus_impact` before editing a symbol and `gitnexus_detect_changes` before finishing. If any tool reports the index is stale, run `npx gitnexus analyze` in `transmitter/` first.
- **All paths are relative to `transmitter/`** unless they start with `docs/` (repo root). Run GitNexus/CLI commands from inside `transmitter/`.
- **Do not restructure** existing files; follow the surrounding C style (4-space indent, `(void)` on ignored returns, `snprintf`).

---

## File structure

| File | Responsibility | Change |
|---|---|---|
| `main/CMakeLists.txt` | Component build | Add `time/time_sync.c` source; add `lwip` dependency (provides `esp_sntp.h`). |
| `main/Kconfig.projbuild` | Build-time config | New `TRANSMITTER_DEVICE_TIMEZONE` string (POSIX TZ). |
| `main/time/time_sync.{h,c}` | SNTP driver (dropped-in sample) | Guard rename; TZ macro rename; add missing libc includes. |
| `main/network/wifi.c` | Wi-Fi STA/AP lifecycle | Start SNTP once on first got-IP. |
| `main/utils/time_utils.c` | Epoch/clock helpers | Guarded `settimeofday()` bootstrap from backend timestamp. |
| `main/display/display_screens.h` | OLED widget structs | `status_bar_t.lbl_uptime`→`lbl_clock`; drop `screen_network_t.lbl_hub_id`. |
| `main/display/display_screens.c` | OLED widget builders | Status bar: WiFi/USB → icons, clock right-aligned. Network: drop Hub row, widen row spacing. |
| `main/display/display.c` | OLED update logic + lifecycle | TZ at init; `format_status_clock()`; rewrite `update_status_bar()` (C′); drop hub line from `update_screen_network()`. |
| `docs/CHANGELOGS.md` | Change log | Record all of the above. |

Tasks are ordered so the time foundation lands first, then the display consumers. Files compile as a set; the build is validated in the separate flash flow after Task 7 (see conventions above).

---

## Task 1: Build wiring, Kconfig, and SNTP sample adaptation

**Files:**
- Modify: `main/CMakeLists.txt`
- Modify: `main/Kconfig.projbuild`
- Modify: `main/time/time_sync.h`
- Modify: `main/time/time_sync.c`

This task has no existing-symbol edits, so no impact analysis is needed (new source file + config + the sample's own internals).

- [ ] **Step 1: Add the SNTP source to the build**

In `main/CMakeLists.txt`, add `time/time_sync.c` to `SRCS` (right after the existing `time_utils.c` line):

```cmake
    "utils/time_utils.c"
    "time/time_sync.c"
    "rf/rf_data.c"
```

- [ ] **Step 2: Add the `lwip` component dependency**

`esp_sntp.h` (the legacy SNTP API the sample uses) lives in the `lwip` component. Add it to `PRIV_DEPS`:

```cmake
set(PRIV_DEPS
    nvs_flash esp_wifi esp_netif esp_event esp_http_server lwip
    esp_driver_gpio esp_driver_spi esp_timer esp_hw_support
    mbedtls espressif__mqtt espressif__cjson esp_driver_gptimer
    espressif__button esp_driver_usb_serial_jtag
)
```

- [ ] **Step 3: Add the timezone Kconfig symbol**

In `main/Kconfig.projbuild`, inside the existing `menu "Transmitter Hub Firmware"`, insert the block below right after the `TRANSMITTER_BACKEND_PUBKEY_B64` config (between its `help` text and `config TRANSMITTER_OLED_SDA_GPIO`):

```kconfig
config TRANSMITTER_DEVICE_TIMEZONE
    string "Device POSIX timezone (TZ) for the OLED clock"
    default "ICT-7"
    help
        POSIX TZ string used to render local time on the OLED status-bar
        clock. Default "ICT-7" is Vietnam Indochina Time (UTC+7, no DST).
        POSIX offsets are sign-inverted: UTC+7 is written "-7".
```

- [ ] **Step 4: Rename the sample's include guard**

In `main/time/time_sync.h`, rename the copy-paste guard. Change the opening:

```c
#ifndef TRANSMITTER_TIME_SYNC_H
#define TRANSMITTER_TIME_SYNC_H
```

and the closing line at the bottom of the file:

```c
#endif //TRANSMITTER_TIME_SYNC_H
```

- [ ] **Step 5: Rename the TZ macro and add missing libc includes in `time_sync.c`**

In `main/time/time_sync.c`, add the two libc headers the sample uses but does not include (`memset` → `string.h`, `setenv` → `stdlib.h`). Change the include block:

```c
#include "time_sync.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_sntp.h"
#include "esp_log.h"

#include <stdlib.h>
#include <string.h>
#include <sys/time.h>
```

Then update the timezone branch to read the new Kconfig symbol:

```c
#ifdef CONFIG_TRANSMITTER_DEVICE_TIMEZONE
    setenv("TZ", CONFIG_TRANSMITTER_DEVICE_TIMEZONE, 1);
#else
    setenv("TZ", "UTC", 1);
#endif
```

- [ ] **Step 6: Verify**

Run from `transmitter/`:

```bash
grep -n "time/time_sync.c" main/CMakeLists.txt && grep -n "lwip" main/CMakeLists.txt
grep -n "TRANSMITTER_DEVICE_TIMEZONE" main/Kconfig.projbuild main/time/time_sync.c
grep -n "TRANSMITTER_TIME_SYNC_H" main/time/time_sync.h
grep -nE "DOORBELL_TIME_SYNC_H|CONFIG_DEVICE_TIMEZONE" main/time/time_sync.*
```
Expected: the first three commands print matching lines; the last prints **nothing** (no stale names remain).

---

## Task 2: Start SNTP on first got-IP

**Files:**
- Modify: `main/network/wifi.c`

- [ ] **Step 1: Impact analysis**

Run from `transmitter/`:

```bash
# via GitNexus MCP
gitnexus_impact({target: "wifi_event_handler", direction: "upstream"})
```
Report the blast radius. This is an internal static handler; expect LOW/MEDIUM. Proceed unless HIGH/CRITICAL (if so, surface to the user first).

- [ ] **Step 2: Include the time-sync header**

In `main/network/wifi.c`, add the include after `config/device_config.h`:

```c
#include "config/device_config.h"
#include "time/time_sync.h"
```

- [ ] **Step 3: Add a one-shot guard flag**

Add the static flag alongside the other file-scope booleans:

```c
static bool s_wifi_initialized;
static bool s_wifi_started;
static bool s_sntp_started;
static bool s_sta_connected;
```

- [ ] **Step 4: Kick off SNTP in the got-IP branch**

In `wifi_event_handler`, the `IP_EVENT_STA_GOT_IP` branch becomes:

```c
    } else if (event_base == IP_EVENT && event_id == IP_EVENT_STA_GOT_IP) {
        s_sta_connected = true;
        s_sta_retry_count = 0;
        ESP_LOGI(TAG, "STA connected");
        if (!s_sntp_started) {
            time_sync_init();   /* sets TZ + starts SNTP polling, non-blocking */
            s_sntp_started = true;
        }
        xEventGroupSetBits(s_wifi_events, WIFI_BIT_STA_CONNECTED);
        (void)esp_event_post(TRANSMITTER_EVENTS, TX_EVENT_WIFI_CONNECTED, NULL, 0, 0);
    }
```

- [ ] **Step 5: Verify**

```bash
grep -n "time/time_sync.h\|s_sntp_started\|time_sync_init" main/network/wifi.c
```
Expected: the include, the static flag declaration, and the guarded `time_sync_init()` call all present.

---

## Task 3: Guarded system-clock bootstrap from the backend

**Files:**
- Modify: `main/utils/time_utils.c`

- [ ] **Step 1: Impact analysis**

```bash
gitnexus_impact({target: "time_utils_sync_from_iso8601", direction: "upstream"})
```
Expected callers: `mqtt.c` (bootstrap) and `dispatch.c` (per-command `issued_at`). The change is purely additive (only sets the clock when it is currently unset), so risk is LOW even though there are several callers. Surface to the user only if HIGH/CRITICAL.

- [ ] **Step 2: Add the guarded `settimeofday` bootstrap**

In `main/utils/time_utils.c`, `time_utils_sync_from_iso8601()` currently ends with the offset assignment and `return ESP_OK;`. Replace that tail:

```c
    s_backend_epoch_offset_us = epoch_ms * 1000LL - esp_timer_get_time();
    s_have_backend_reference = true;

    /* Bootstrap the system clock from the backend ONLY if it is not already
       set (year < 2024). NTP owns the clock once it syncs; never override it. */
    struct timeval tv_now = { 0 };
    if (gettimeofday(&tv_now, NULL) != 0 || tv_now.tv_sec < 1704067200LL) {
        struct timeval tv = {
            .tv_sec  = (time_t)(epoch_ms / 1000LL),
            .tv_usec = (suseconds_t)((epoch_ms % 1000LL) * 1000LL),
        };
        (void)settimeofday(&tv, NULL);
    }
    return ESP_OK;
```

(`<sys/time.h>` is already included in this file, so `gettimeofday`/`settimeofday`/`struct timeval`/`suseconds_t` are available.)

- [ ] **Step 3: Verify**

```bash
grep -n "settimeofday\|1704067200LL" main/utils/time_utils.c
```
Expected: the guarded `settimeofday` block present; the threshold matches the one already used in `time_utils_now_epoch_ms()`.

---

## Task 4: Status-bar struct + builder (icons + clock)

**Files:**
- Modify: `main/display/display_screens.h`
- Modify: `main/display/display_screens.c`

- [ ] **Step 1: Impact analysis**

```bash
gitnexus_impact({target: "screens_create_status_bar", direction: "upstream"})
```
Expected: called only from `create_operational_screens` in `display.c`. The struct-field rename (`lbl_uptime`→`lbl_clock`) is consumed only in `display.c` (updated in Task 5). Proceed unless HIGH/CRITICAL.

- [ ] **Step 2: Rename the status-bar clock field**

In `main/display/display_screens.h`, change the `status_bar_t` member (this is the **status bar's** field — leave `screen_dashboard_t.lbl_uptime` untouched):

```c
typedef struct {
    lv_obj_t *lbl_mqtt;
    lv_obj_t *lbl_wifi;
    lv_obj_t *lbl_usb;
    lv_obj_t *lbl_state;
    lv_obj_t *lbl_clock;
} status_bar_t;
```

- [ ] **Step 3: Rebuild the status-bar widgets as icons + right-aligned clock**

In `main/display/display_screens.c`, `screens_create_status_bar()`, replace the five label-creation blocks (from `bar->lbl_mqtt = ...` through the old uptime label) with:

```c
    bar->lbl_mqtt = lv_label_create(container);
    lv_obj_set_style_text_color(bar->lbl_mqtt, DISPLAY_WHITE, 0);
    lv_obj_set_pos(bar->lbl_mqtt, 0, 0);
    lv_label_set_text(bar->lbl_mqtt, "o MQTT");

    bar->lbl_wifi = lv_label_create(container);
    lv_obj_set_style_text_color(bar->lbl_wifi, DISPLAY_WHITE, 0);
    lv_obj_set_pos(bar->lbl_wifi, 38, 0);
    lv_label_set_text(bar->lbl_wifi, LV_SYMBOL_CLOSE);

    bar->lbl_usb = lv_label_create(container);
    lv_obj_set_style_text_color(bar->lbl_usb, DISPLAY_WHITE, 0);
    lv_obj_set_pos(bar->lbl_usb, 54, 0);
    lv_label_set_text(bar->lbl_usb, LV_SYMBOL_USB);
    lv_obj_add_flag(bar->lbl_usb, LV_OBJ_FLAG_HIDDEN);

    bar->lbl_state = lv_label_create(container);
    lv_obj_set_style_text_color(bar->lbl_state, DISPLAY_WHITE, 0);
    lv_obj_set_pos(bar->lbl_state, 58, 0);
    lv_label_set_text(bar->lbl_state, "BOOT");

    bar->lbl_clock = lv_label_create(container);
    lv_obj_set_style_text_color(bar->lbl_clock, DISPLAY_WHITE, 0);
    lv_obj_align(bar->lbl_clock, LV_ALIGN_TOP_RIGHT, -2, 0);
    lv_label_set_text(bar->lbl_clock, "--:--");
```

Notes: `LV_SYMBOL_WIFI`/`LV_SYMBOL_USB`/`LV_SYMBOL_CLOSE` come from `font/lv_symbol_def.h` (already included) and render via the `proggy_clean_12 → lv_font_montserrat_10` fallback. The clock alignment is set **once** here (flush-right, persists across text changes); positions for wifi/usb/state are starting values that `update_status_bar()` re-sets per mode (Task 5).

- [ ] **Step 4: Verify**

```bash
grep -n "lbl_clock\|LV_SYMBOL_USB\|LV_ALIGN_TOP_RIGHT" main/display/display_screens.c
grep -n "lbl_clock\|lbl_uptime" main/display/display_screens.h
```
Expected: `display_screens.c` shows the new clock/icon lines; `display_screens.h` shows `lbl_clock` in `status_bar_t` (and `lbl_uptime` still present only in `screen_dashboard_t`).

---

## Task 5: Status-bar update logic, clock formatter, and timezone init

**Files:**
- Modify: `main/display/display.c`

- [ ] **Step 1: Impact analysis**

```bash
gitnexus_impact({target: "update_status_bar", direction: "upstream"})
```
Expected: called from `update_active_screen`, `splash_exit_to_dashboard`, and the MQTT/WiFi/lifecycle event branches in `display_event_handler` — all within `display.c`. Proceed unless HIGH/CRITICAL.

- [ ] **Step 2: Add the libc includes**

In `main/display/display.c`, extend the top includes:

```c
#include <stdint.h>
#include <stdlib.h>
#include <string.h>
#include <stdio.h>
#include <time.h>
#include <sys/time.h>
```

- [ ] **Step 3: Apply the timezone once, early in `display_init()`**

In `display_init()`, immediately after the hardware-init line, add the TZ setup (before the refresh timer is created and before any local-time rendering):

```c
    ESP_RETURN_ON_ERROR(display_hw_init(), TAG, "hw init");

    /* Apply the configured timezone once, before any local-time rendering, so
       the status-bar clock is correctly zoned regardless of which source sets
       the system clock (NTP on got-IP, or a backend timestamp). */
    setenv("TZ", CONFIG_TRANSMITTER_DEVICE_TIMEZONE, 1);
    tzset();

    s_last_user_activity_ms = esp_timer_get_time() / 1000;
```

- [ ] **Step 4: Swap the forward declaration**

Replace the `format_status_uptime` forward declaration (in the forward-declarations block near the top) with the clock formatter:

```c
static void format_uptime(char *buf, size_t sz);
static void format_status_clock(char *buf, size_t sz);
```

- [ ] **Step 5: Replace `format_status_uptime()` with `format_status_clock()`**

Replace the entire `format_status_uptime()` definition with:

```c
static void format_status_clock(char *buf, size_t sz)
{
    struct timeval tv;
    if (gettimeofday(&tv, NULL) == 0 && tv.tv_sec >= 1704067200LL) { /* >= 2024-01-01 UTC */
        struct tm local_tm;
        localtime_r(&tv.tv_sec, &local_tm);
        (void)snprintf(buf, sz, "%02d:%02d", local_tm.tm_hour, local_tm.tm_min);
    } else {
        (void)snprintf(buf, sz, "--:--");
    }
}
```

(`format_uptime()` — the longer one used by the Dashboard `Uptime:` row — is **left unchanged**.)

- [ ] **Step 6: Rewrite `update_status_bar()` for the C′ layout**

Keep the early `if (s_splash_state != SPLASH_DONE) return;` and the `const device_config_t *cfg = ...` / `state_str` switch. Replace **everything from `char uptime_buf[12];` to the end of the function** — delete this existing block:

```c
    char uptime_buf[12];
    format_status_uptime(uptime_buf, sizeof(uptime_buf));

    bool wired = serial_protocol_is_host_connected();
    if (wired) {
        lv_label_set_text(s_status_bar.lbl_mqtt, s_mqtt_connected ? LV_SYMBOL_BULLET : "o");
        lv_label_set_text(s_status_bar.lbl_wifi, s_wifi_connected ? LV_SYMBOL_UP : LV_SYMBOL_CLOSE);
        lv_obj_set_pos(s_status_bar.lbl_wifi, 8, 0);
        lv_obj_set_pos(s_status_bar.lbl_usb, 16, 0);
        lv_obj_clear_flag(s_status_bar.lbl_usb, LV_OBJ_FLAG_HIDDEN);
        lv_obj_set_pos(s_status_bar.lbl_state, 48, 0);
        lv_obj_set_pos(s_status_bar.lbl_uptime, 84, 0);
    } else {
        lv_label_set_text(s_status_bar.lbl_mqtt, s_mqtt_connected ? LV_SYMBOL_BULLET " MQTT" : "o MQTT");
        lv_label_set_text(s_status_bar.lbl_wifi, s_wifi_connected ? LV_SYMBOL_UP " WiFi" : LV_SYMBOL_CLOSE " WiFi");
        lv_obj_set_pos(s_status_bar.lbl_wifi, 38, 0);
        lv_obj_add_flag(s_status_bar.lbl_usb, LV_OBJ_FLAG_HIDDEN);
        lv_obj_set_pos(s_status_bar.lbl_state, 76, 0);
        lv_obj_set_pos(s_status_bar.lbl_uptime, 104, 0);
    }
    lv_label_set_text(s_status_bar.lbl_state, state_str);
    lv_label_set_text(s_status_bar.lbl_uptime, uptime_buf);
```

and replace it with:

```c
    char clock_buf[12];
    format_status_clock(clock_buf, sizeof(clock_buf));

    /* MQTT keeps its word; WiFi/USB are icons (via the montserrat fallback).
       The clock is right-aligned at creation, so only its text changes here. */
    lv_label_set_text(s_status_bar.lbl_mqtt, s_mqtt_connected ? LV_SYMBOL_BULLET " MQTT" : "o MQTT");
    lv_label_set_text(s_status_bar.lbl_wifi, s_wifi_connected ? LV_SYMBOL_WIFI : LV_SYMBOL_CLOSE);
    lv_obj_set_pos(s_status_bar.lbl_wifi, 38, 0);

    if (serial_protocol_is_host_connected()) {
        lv_obj_set_pos(s_status_bar.lbl_usb, 54, 0);
        lv_obj_clear_flag(s_status_bar.lbl_usb, LV_OBJ_FLAG_HIDDEN);
        lv_obj_set_pos(s_status_bar.lbl_state, 72, 0);
    } else {
        lv_obj_add_flag(s_status_bar.lbl_usb, LV_OBJ_FLAG_HIDDEN);
        lv_obj_set_pos(s_status_bar.lbl_state, 58, 0);
    }

    lv_label_set_text(s_status_bar.lbl_state, state_str);
    lv_label_set_text(s_status_bar.lbl_clock, clock_buf);
```

The X positions (38 / 54 / 58 / 72) are starting values; the separate on-device flash flow may nudge them a few pixels (the Montserrat icons are slightly wider than the old text). The clock is never repositioned here — its right alignment was fixed in Task 4.

- [ ] **Step 7: Verify**

```bash
grep -n "format_status_uptime" main/display/display.c          # expect: NO matches
grep -n "format_status_clock\|lbl_clock\|LV_SYMBOL_WIFI\|setenv(\"TZ\"" main/display/display.c
```
Expected: no `format_status_uptime` remains (neither definition nor forward decl); the clock formatter, `lbl_clock` updates, `LV_SYMBOL_WIFI`, and the TZ setup are all present.

---

## Task 6: Slim the Network Info screen (drop Hub, add breathing room)

**Files:**
- Modify: `main/display/display_screens.h`
- Modify: `main/display/display_screens.c`
- Modify: `main/display/display.c`

- [ ] **Step 1: Impact analysis**

```bash
gitnexus_impact({target: "update_screen_network", direction: "upstream"})
gitnexus_impact({target: "screens_create_network_info", direction: "upstream"})
```
Expected: builder called only from `create_operational_screens`; updater only from `update_active_screen`. The `lbl_hub_id` field is referenced only in these two functions. Proceed unless HIGH/CRITICAL.

- [ ] **Step 2: Remove the `lbl_hub_id` field**

In `main/display/display_screens.h`, `screen_network_t` becomes:

```c
typedef struct {
    lv_obj_t *lbl_ip;
    lv_obj_t *lbl_rssi;
    lv_obj_t *lbl_ssid;
    lv_obj_t *lbl_usb;
} screen_network_t;
```

- [ ] **Step 3: Drop the Hub row and widen row spacing in the builder**

In `main/display/display_screens.c`, `screens_create_network_info()`:

First, increase the row gap (this anchor — ending in `net->lbl_ip` — is unique to this builder, so it won't touch the other screens' `pad_row`):

```c
    lv_obj_set_style_pad_row(cont, 3, 0);
    lv_obj_set_style_text_font(cont, &proggy_clean_12, 0);

    net->lbl_ip = lv_label_create(cont);
```

Then delete the three-line `lbl_hub_id` creation block so the SSID label is followed directly by the wired label:

```c
    net->lbl_ssid = lv_label_create(cont);
    lv_obj_set_style_text_color(net->lbl_ssid, DISPLAY_WHITE, 0);
    lv_label_set_text(net->lbl_ssid, "AP:   N/A");

    net->lbl_usb = lv_label_create(cont);
    lv_obj_set_style_text_color(net->lbl_usb, DISPLAY_WHITE, 0);
    lv_label_set_text(net->lbl_usb, "Wired: ---");
```

(That is: the block `net->lbl_hub_id = lv_label_create(cont); ... lv_label_set_text(net->lbl_hub_id, "Hub:  N/A");` is removed entirely.)

- [ ] **Step 4: Drop the Hub update from the updater**

In `main/display/display.c`, `update_screen_network()`, remove the hub block (which also removes the now-unused local `cfg`). The SSID block should flow straight into the wired-status block:

```c
    lv_label_set_text(s_network_scr.lbl_ssid, buf);

    if (serial_protocol_is_host_connected()) {
        (void)snprintf(buf, sizeof(buf), "Wired: Session active");
    } else if (serial_protocol_is_usb_host_connected()) {
        (void)snprintf(buf, sizeof(buf), "Wired: No session");
    } else {
        (void)snprintf(buf, sizeof(buf), "Wired: ---");
    }
    lv_label_set_text(s_network_scr.lbl_usb, buf);
```

(The deleted lines are `const device_config_t *cfg = device_config_get();`, the `"Hub:  %s"` `snprintf`, and `lv_label_set_text(s_network_scr.lbl_hub_id, buf);`. `device_config_t`/`device_config_get` remain used by other functions in the file — only this local use goes away.)

- [ ] **Step 5: Verify**

```bash
grep -n "lbl_hub_id" main/display/display_screens.h main/display/display_screens.c main/display/display.c   # expect: NO matches
grep -n "pad_row(cont, 3" main/display/display_screens.c
```
Expected: `lbl_hub_id` is gone everywhere; the network screen's `pad_row` is `3`.

---

## Task 7: Changelog, scope check, and on-device acceptance

**Files:**
- Modify: `docs/CHANGELOGS.md`

- [ ] **Step 1: Record the change in the changelog**

`docs/CHANGELOGS.md` is reverse-chronological with `## YYYY-MM-DD — Title` entries. Insert this block immediately after the `# Changelogs` heading (above the current top entry):

```markdown
## 2026-06-09 — Transmitter OLED header & clock rework

Reworked the transmitter OLED status bar and Network Info screen, and replaced the status-bar
uptime readout with real NTP-synced local time. Implements
`docs/spec/Transmitter OLED Header & Clock Rework Spec.md`. All SNTP/TZ/LVGL APIs were verified
against Context7 (ESP-IDF v6.0, LVGL 8.4 docs) before implementation.

**Time (`main/time/time_sync.c`, `main/network/wifi.c`, `main/utils/time_utils.c`, `main/Kconfig.projbuild`, `main/CMakeLists.txt`):**
- Wired the dropped-in SNTP sample into the build (`time/time_sync.c` source + `lwip` in `PRIV_REQUIRES`, which provides `esp_sntp.h`); renamed its include guard and pointed it at `CONFIG_TRANSMITTER_DEVICE_TIMEZONE`.
- Added Kconfig `TRANSMITTER_DEVICE_TIMEZONE` (default `ICT-7`, UTC+7 no DST).
- `wifi.c` starts SNTP once on the first got-IP (static guard).
- `time_utils_sync_from_iso8601()` now also bootstraps the system clock via `settimeofday()`, guarded to fire only when the clock is unset (year < 2024) so it never overrides NTP.

**Display (`main/display/display.c`, `display_screens.{c,h}`):**
- Header style C′: `MQTT` keeps its word; WiFi → `LV_SYMBOL_WIFI`, wired → `LV_SYMBOL_USB` (icons via the existing `proggy_clean_12 → montserrat_10` fallback); state retained; clock right-aligned. `status_bar_t.lbl_uptime` → `lbl_clock`; `format_status_uptime()` → `format_status_clock()` (reads the system clock; `--:--` until set, then `HH:MM` local). TZ applied once in `display_init()`.
- Network Info style N1: removed the duplicated `Hub:` row (`screen_network_t.lbl_hub_id`) and widened row spacing (`pad_row` 1 → 3) so the `Wired:` row is no longer clipped.

**Skipped (per spec scope):** no web/backend changes; no 12-hour or seconds format; timezone is build-time (Kconfig), not runtime-provisioned.
```

- [ ] **Step 2: Confirm change scope with GitNexus**

```bash
# from transmitter/
gitnexus_detect_changes({scope: "all"})
```
Expected affected symbols/files only: `wifi_event_handler`, `time_utils_sync_from_iso8601`, `screens_create_status_bar`, `screens_create_network_info`, `update_status_bar`, `update_screen_network`, `format_status_clock` (new), plus `display_init`, and the two structs / build files. Investigate anything outside this set before proceeding.

- [ ] **Step 3: Refresh the GitNexus index**

The transmitter index lives at `transmitter/.gitnexus/` and currently has `"embeddings": 0` (per its `meta.json`), so a plain re-analyze is correct — there are no embeddings to preserve:

```bash
# from transmitter/
npx gitnexus analyze
```

(The repo's PostToolUse hook normally re-analyzes after a commit/merge; since this plan does not commit, run it manually here.)

- [ ] **Step 4: On-device acceptance (separate flash flow)**

Flash via the project's normal flow (not part of this plan), then confirm:

1. Header clock shows `--:--` right after boot, then real local time (`HH:MM`, ICT) within seconds of Wi-Fi connecting.
2. Header is uncluttered with **nothing clipped** in **both** Wi-Fi mode and wired-host mode; WiFi and USB render as icons; the `MQTT` word and the state text are present; the clock sits flush-right.
3. If any icon shows as a placeholder box, the Montserrat-10 build lacks that glyph — fall back to regenerating `proggy_clean_12` with the symbols merged (`--range 0x20-0x7F,0x2022,0xF00D,0xF077,0xF1EB,0xF287`). Not expected, since `↑`/`✕` already render via this path.
4. Network Info shows four spaced rows — `IP` / `RSSI` / `AP+Ch` / `Wired: <status>` — with **no clipping**; the `Hub:` row is gone.
5. Device Info still shows the hub `public_id`; the Dashboard still shows the `Uptime:` row.
6. If header glyphs overlap or the clock crowds the state on hardware, nudge the X positions in `update_status_bar()` (Task 5, Step 6) by a few pixels.

---

## Self-review / audit

Audit performed against the spec and codebase (spec compliance, UI-rules compliance, logical errors, flow inconsistency, integration). Findings amended inline before finalizing — see the "Audit notes" the assistant reports alongside this plan. The plan covers every spec section: §3 time architecture (Tasks 1–3, 5-Step-3), §4 header C′ (Tasks 4–5), §5 network N1 (Task 6), §6 files-touched (all tasks), §7 edge cases (Task 5 TZ-early; Task 3 guard), §8 verification (Task 7). Web-frontend UI rules are out of scope (firmware OLED); applicable doc conventions (Mermaid in the spec, `docs/planned/` location, no embedded commit steps) are met.
