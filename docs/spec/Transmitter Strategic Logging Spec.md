# Transmitter Strategic Logging Spec

**Date:** 2026-05-27
**Scope:** Add field-debugging logging to all transmitter modules, matching the RF submodule's existing style

## Context

The 433 MHz RF submodule (`rf/`) has solid lifecycle and diagnostic logging via `ESP_LOGI`, `ESP_LOGW`, and the `ESP_RETURN_ON_*` / `ESP_ERROR_CHECK` macros (which log on failure automatically). The remaining 11 modules have minimal or zero explicit logging, making field debugging difficult.

This spec adds strategic logging to every module — not verbose tracing, but enough to reconstruct device behavior from serial output when diagnosing issues. The NRF24 module receives detailed logging given its criticality and complex hardware interaction.

## API Conventions (verified against ESP-IDF v6.0 docs)

| Macro | Level | Usage |
|-------|-------|-------|
| `ESP_LOGI(TAG, ...)` | INFO | Lifecycle transitions, key decision points, init/deinit summaries |
| `ESP_LOGW(TAG, ...)` | WARN | Recoverable issues, retries, rejected operations |
| `ESP_LOGE(TAG, ...)` | ERROR | Failures, signing errors, hardware faults |
| `ESP_RETURN_ON_ERROR` | (auto) | Already logs on failure at ERROR level — do NOT duplicate with a preceding log |
| `ESP_RETURN_ON_FALSE` | (auto) | Already logs on failure at ERROR level — do NOT duplicate with a preceding log |
| `ESP_ERROR_CHECK` | (auto) | Logs + aborts on failure — do NOT duplicate |

**ISR safety:** `ESP_LOGx` must NOT be called from ISR context. The existing ISR callbacks (`rf_timer_callback`, `nrf24_irq_isr`) correctly have no logging and must remain that way.

**Tag convention:** Each file already defines `static const char *TAG = "..."` (or `#define RF_TAG "RF"` for the RF submodule). All new logs use the existing TAG — no new tags introduced.

## Module-by-Module Logging Plan

### 1. `network/wifi.c` (TAG: `"wifi"`)

Currently zero explicit log calls.

| Location | Level | Message | Rationale |
|----------|-------|---------|-----------|
| `wifi_start_sta` success (after `esp_wifi_start`) | INFO | `"STA starting, SSID=%.32s"` (from `sta_cfg.sta.ssid`) | Confirm which network the device is connecting to |
| `wifi_start_softap` success (after `esp_wifi_start`) | INFO | `"SoftAP started, SSID=%s"` (from `s_softap_ssid`) | Confirm recovery AP is up |
| `wifi_stop` (in `stop_if_started`, only when `s_wifi_started` was true) | INFO | `"WiFi stopped"` | Confirm clean teardown |
| `wifi_event_handler` — `WIFI_EVENT_STA_DISCONNECTED` with retry | WARN | `"STA disconnected, retry %u/%u"` | Track retry progression |
| `wifi_event_handler` — `WIFI_EVENT_STA_DISCONNECTED` retries exhausted | WARN | `"STA retries exhausted"` | Explain why the device enters recovery |
| `wifi_event_handler` — `IP_EVENT_STA_GOT_IP` | INFO | `"STA connected"` | Confirm station association |

**Note:** `wifi_start_sta` receives `cfg->wifi_ssid` but copies it into `sta_cfg.sta.ssid`. The log uses the copy to avoid logging stale config. The SSID is capped at 32 chars by the `wifi_config_t` struct — `%.32s` matches this bound.

### 2. `network/mqtt.c` (TAG: `"mqtt"`)

Currently 4 logs (errors only).

| Location | Level | Message | Rationale |
|----------|-------|---------|-----------|
| `mqtt_event_handler` — `MQTT_EVENT_CONNECTED` | INFO | `"MQTT connected"` | Confirm broker connectivity |
| `mqtt_event_handler` — `MQTT_EVENT_DISCONNECTED` | WARN | `"MQTT disconnected"` | Signal connectivity loss |
| `mqtt_subscribe_current_phase` — activated branch | INFO | `"Subscribed to operational topics"` | Confirm post-activation subscription |
| `mqtt_subscribe_current_phase` — bootstrap branch | INFO | `"Subscribed to bootstrap topics"` | Confirm bootstrap subscription |
| `mqtt_publish_register` success | INFO | `"Bootstrap register published"` | Confirm registration message sent |
| `handle_bootstrap_pending` — nonce matched | INFO | `"Bootstrap pending, challenge_id assigned"` | Track bootstrap progression |
| `handle_bootstrap_rejected` | WARN | `"Bootstrap rejected by backend"` | Explain activation failure |
| `handle_bootstrap_challenge` — response published | INFO | `"Bootstrap challenge response sent"` | Track bootstrap progression |
| `handle_bootstrap_result` — activation committed | INFO | `"Bootstrap activation confirmed"` | Confirm successful activation |
| `mqtt_start` success (after `mqtt_start_client` returns OK) | INFO | `"MQTT client started"` | Confirm client initialization |
| `mqtt_stop` (when `s_client != NULL`) | INFO | `"MQTT client stopped"` | Confirm clean teardown |

**Not logged:** Individual `route_message` calls — these are high-frequency and the dispatch module logs the outcome. The `mqtt_ctl_task` restart loop already has an `ESP_LOGE` for restart failures.

### 3. `dispatch/dispatch.c` (TAG needed: `"dispatch"`)

Currently zero log calls and no TAG defined. Add `static const char *TAG = "dispatch";` at file scope.

| Location | Level | Message | Rationale |
|----------|-------|---------|-----------|
| `dispatch_handle_transmit_json` — parse failure | WARN | `"Transmit rejected: bad_shape"` | Diagnose malformed commands |
| `dispatch_handle_transmit_json` — width mismatch | WARN | `"Transmit rejected: width_mismatch, band=%s bits=%d"` | Diagnose payload issues |
| `dispatch_handle_transmit_json` — signature failed | WARN | `"Transmit rejected: signature_failed"` | Detect auth issues |
| `dispatch_handle_transmit_json` — duplicate (ring hit) | INFO | `"Transmit duplicate: dispatch_id=%.8s status=%s"` | Explain idempotency replay |
| `dispatch_handle_transmit_json` — device not active | WARN | `"Transmit rejected: %s"` (reason: device_suspended / device_decommissioned) | Explain operational state rejection |
| `dispatch_handle_transmit_json` — bad hex | WARN | `"Transmit rejected: bad_hex"` | Diagnose payload encoding |
| `dispatch_handle_transmit_json` — applied | INFO | `"Transmit applied: band=%s bits=%d"` | Confirm successful dispatch |
| `dispatch_handle_transmit_json` — tx_failed / no_2_4g_radio | WARN | `"Transmit rejected: %s, band=%s"` | Diagnose radio failures |
| `dispatch_handle_deact_json` — parse failure | WARN | `"Deact rejected: bad_shape"` | Diagnose malformed lifecycle commands |
| `dispatch_handle_deact_json` — signature failed | WARN | `"Deact rejected: signature_failed"` | Detect auth issues |
| `dispatch_handle_deact_json` — processed | INFO | `"Deact processed: action=%s status=%s"` | Confirm lifecycle transitions |

**Header include:** Add `#include "esp_log.h"` — the file currently includes neither `esp_log.h` nor `esp_check.h` directly.

### 4. `network/heartbeat.c` (TAG needed: `"heartbeat"`)

Currently zero log calls and no TAG defined. Add `static const char *TAG = "heartbeat";` at file scope.

| Location | Level | Message | Rationale |
|----------|-------|---------|-----------|
| `heartbeat_start` — task created | INFO | `"Heartbeat started"` | Confirm periodic reporting is active |
| `heartbeat_stop` — task signaled to stop | INFO | `"Heartbeat stopped"` | Confirm reporting is paused |

**Not logged:** Individual heartbeat publishes — these are periodic (every 10s) and would flood the log. The MQTT layer already confirms connectivity.

**Header include:** Add `#include "esp_log.h"` — the file currently does not include it.

### 5. `provision/http_server.c` (TAG: `"http_server"`)

Currently zero explicit log calls (has TAG already).

| Location | Level | Message | Rationale |
|----------|-------|---------|-----------|
| `provision_http_server_start` — after `httpd_start` | INFO | `"HTTP provisioning server started"` | Confirm SoftAP server is ready |
| `provision_http_server_stop` — after `httpd_stop` | INFO | `"HTTP provisioning server stopped"` | Confirm server teardown |
| `provision_post_handler` — provisioning saved | INFO | `"Provisioning saved via HTTP"` | Confirm provisioning source |
| `provision_post_handler` — validation failure | WARN | `"Provisioning rejected: invalid payload"` | Diagnose bad provisioning attempts |
| `retry_post_handler` | INFO | `"WiFi retry requested via HTTP"` | Track recovery attempts |
| `reset_post_handler` — factory reset | WARN | `"Factory reset requested via HTTP"` | Important lifecycle event, WARN since destructive |

**Header include:** Add `#include "esp_log.h"` — the file currently only has `esp_check.h` (which may include `esp_log.h` transitively, but following the project convention of explicit includes).

### 6. `config/device_config.c` (TAG: `"device_config"`)

Currently 3 logs (NVS read warnings only).

| Location | Level | Message | Rationale |
|----------|-------|---------|-----------|
| `device_config_load` — end of function | INFO | `"Config loaded: provisioned=%s activated=%s op_state=%s"` | Startup state summary |
| `device_config_save_provisioning` — success | INFO | `"Provisioning config saved"` | Confirm NVS write |
| `device_config_commit_activation` — success | INFO | `"Activation committed: public_id=%s"` | Confirm device identity stored |
| `device_config_commit_lifecycle` — success | INFO | `"Lifecycle committed: op_state=%s"` (using `device_config_op_state_to_string`) | Confirm state transition persisted |
| `device_config_factory_reset` — success | WARN | `"Factory reset complete"` | Important destructive event |
| `device_config_clear_enroll_token` — success | INFO | `"Enrollment token cleared"` | Track bootstrap cleanup |

### 7. `security/device_identity.c` (TAG: `"device_identity"`)

Currently errors only (via `log_mbedtls_error` helper and `ESP_LOGE`).

| Location | Level | Message | Rationale |
|----------|-------|---------|-----------|
| `generate_device_keypair` — success | INFO | `"Device keypair generated and saved"` | Confirm first-boot key creation |
| `load_device_keypair_from_nvs` — success | INFO | `"Device keypair loaded from NVS"` | Confirm key recovery |
| `load_backend_public_key` — success | INFO | `"Backend public key loaded"` | Confirm signature verification ready |
| `device_identity_ensure` — new keypair generated (after `ESP_ERR_NOT_FOUND` branch) | INFO | `"No existing keypair, generating new identity"` | Distinguish new device from returning device |

### 8. `nrf24/nrf24_transmitter.c` (TAG: `"nrf24"`) — DETAILED

Currently has error-path logging only plus misleading `ESP_LOGI` in failure labels.

**Fix existing logs:** The `fail_irq`, `fail_device`, `fail_bus`, `fail` labels currently use `ESP_LOGI` — these should be `ESP_LOGW` since they indicate partial init failure (the device still boots, just without 2.4 GHz radio).

| Location | Level | Message | Rationale |
|----------|-------|---------|-----------|
| `nrf24_tx_init` — after SPI bus + GPIO setup | INFO | `"SPI: MOSI=%d MISO=%d SCK=%d CS=%d CE=%d IRQ=%d"` | Hardware wiring verification (one-time at init) |
| `nrf_probe_chip` — variant detected | INFO | `"Chip detected: %s"` (`"nRF24L01+"` or `"nRF24L01 (legacy)"`) | Confirm hardware variant |
| `nrf24_tx_init` — success (`h->ready = true`) | INFO | `"nRF24 initialized: %s, ch=%d, %s"` (variant, channel, data rate) | Init summary |
| `nrf24_tx_send` — before CE pulse | INFO | `"TX start: addr=%02x:%02x:%02x:%02x:%02x"` | Correlate sends with receiver addresses |
| `nrf24_tx_send` — success (TX_DS set) | INFO | `"TX success"` | Confirm delivery |
| `nrf24_tx_send` — MAX_RT | WARN | `"TX MAX_RT: auto-retransmit exhausted"` | Diagnose range/interference issues |
| `nrf24_tx_send` — timeout (no IRQ within 50ms) | WARN | `"TX timeout: no IRQ within 50ms"` | Diagnose hardware faults |
| `nrf24_tx_send` — `power_down` label, `nrf_rreg8` fails | WARN | `"Power-down read-back failed"` | Diagnose SPI issues post-TX (log in the else branch when `nrf_rreg8` returns non-OK) |
| `nrf24_tx_deinit` — success | INFO | `"nRF24 deinitialized"` | Confirm clean teardown |
| `fail_irq` label | WARN (fix from LOGI) | `"Init failed at IRQ handler"` | Accurate severity |
| `fail_device` label | WARN (fix from LOGI) | `"Init failed at SPI device"` | Accurate severity |
| `fail_bus` label | WARN (fix from LOGI) | `"Init failed at SPI bus"` | Accurate severity |
| `fail` label | WARN (fix from LOGI) | `"nRF24 init failed"` | Accurate severity |

### 9. `dispatch/radio_supervisor.c` (TAG: `"radio_supervisor"`)

Currently 1 error log.

| Location | Level | Message | Rationale |
|----------|-------|---------|-----------|
| `radio_supervisor_init` — success | INFO | `"Radio supervisor initialized: 433M=ready 2.4G=%s"` (`"ready"` or `"unavailable"`) | Init summary for both radios |
| `radio_tx_send` — after `xSemaphoreGive`, success | INFO | `"TX %s: %u bits"` (band string) | Confirm transmission parameters |
| `radio_tx_send` — after `xSemaphoreGive`, failure | WARN | `"TX failed: %s err=%s"` (band string, `esp_err_to_name`) | Diagnose radio failures |
| `radio_supervisor_deinit` | INFO | `"Radio supervisor deinitialized"` | Confirm teardown |

**Mutex placement:** All logging in `radio_tx_send` must occur AFTER `xSemaphoreGive(s_radio_mtx)` to minimize mutex hold time. Capture `err` and band info before releasing, then log after the give.

### 10. `serial/serial_protocol.c` (TAG: `"serial"`)

Currently 2 logs (1 error, 1 init).

| Location | Level | Message | Rationale |
|----------|-------|---------|-----------|
| `serial_handle_line` — after identifying command type | INFO | `"Serial cmd: %s"` (the `type` string) | Track command flow from host |
| `serial_handle_line` — unknown command | WARN | `"Serial unknown cmd: %s"` (the `type` string) | Diagnose protocol mismatches |
| `handle_provision` — restart | INFO | `"Serial provision received, restarting"` | Track provisioning source |
| `handle_factory_reset` — restart | WARN | `"Serial factory reset, restarting"` | Important lifecycle event |
| `handle_update_mqtt` — restart | INFO | `"Serial MQTT config updated, restarting"` | Track config changes |

**Not logged:** `handle_ping`, `handle_identify`, `handle_status` — these are read-only queries that would flood the log during active serial sessions.

### 11. `controls/controls.c` (TAG: `"controls"`)

Currently 1 init log.

| Location | Level | Message | Rationale |
|----------|-------|---------|-----------|
| `on_long_press_start` — recovery armed | WARN | `"Recovery mode armed (hold 5s more)"` | Track user-initiated recovery |
| `recovery_stage2_timer_cb` — recovery triggered | WARN | `"Recovery mode triggered, restarting"` | Critical lifecycle event |
| `on_long_press_release` — recovery cancelled | INFO | `"Recovery mode cancelled"` | Track user intent |

**Not logged:** Single click, double click — these are normal UI navigation. LED pattern changes — these are reactive to events already logged by other modules.

## Files Modified

| File | Changes |
|------|---------|
| `main/network/wifi.c` | Add 6 log calls (has `esp_log.h` + TAG already) |
| `main/network/mqtt.c` | Add 11 log calls (has `esp_log.h` + TAG already) |
| `main/dispatch/dispatch.c` | Add TAG `"dispatch"` + `#include "esp_log.h"` + 11 log calls |
| `main/network/heartbeat.c` | Add TAG `"heartbeat"` + `#include "esp_log.h"` + 2 log calls |
| `main/provision/http_server.c` | Add `#include "esp_log.h"` + 6 log calls (has TAG already) |
| `main/config/device_config.c` | Add 6 log calls (has `esp_log.h` + TAG already) |
| `main/security/device_identity.c` | Add 4 log calls (has `esp_log.h` + TAG already) |
| `main/nrf24/nrf24_transmitter.c` | Fix 4 existing LOGI→LOGW + add 9 log calls (has `esp_log.h` + TAG already) |
| `main/dispatch/radio_supervisor.c` | Add 4 log calls (has `esp_log.h` + TAG already); logs after mutex release |
| `main/serial/serial_protocol.c` | Add 5 log calls (has `esp_log.h` + TAG already) |
| `main/controls/controls.c` | Add 3 log calls (has `esp_log.h` + TAG already) |

**Total: 11 files, ~67 new log calls, 4 severity fixes**

## Constraints

- No logging from ISR context (`rf_timer_callback`, `nrf24_irq_isr`)
- No sensitive data in logs (no MQTT passwords, WiFi passwords, enrollment tokens, private keys, signature payloads)
- `ESP_RETURN_ON_ERROR` / `ESP_RETURN_ON_FALSE` already log on failure — do not add a redundant log before them
- Dispatch IDs truncated to 8 chars (`%.8s`) in logs to avoid flooding
- Public IDs are safe to log (they are public identifiers by design)
- SSID truncated to 32 chars (`%.32s`) matching `wifi_config_t` bounds
