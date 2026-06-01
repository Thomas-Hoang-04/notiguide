# WiFi Connection Test for Receivers

Implements the `provision.test_wifi` serial command on both receiver platforms. Currently both return `not_supported` — this adds a one-shot WiFi connection test that validates credentials before provisioning.

## Transmitter Reference

The transmitter's `handle_test_wifi` (serial_protocol.c:307-362) uses this pattern:

```c
device_config_t test_cfg = { 0 };
strlcpy(test_cfg.wifi_ssid, ssid, sizeof(test_cfg.wifi_ssid));
if (pwd != NULL) strlcpy(test_cfg.wifi_pwd, pwd, sizeof(test_cfg.wifi_pwd));

esp_err_t err = wifi_start_sta(&test_cfg);       // async start
err = wifi_wait_for_sta_ready(pdMS_TO_TICKS(15000));  // blocking wait with timeout

// on success: esp_wifi_sta_get_rssi(&rssi) + esp_netif_get_ip_info()
wifi_stop();
```

Key: `wifi_start_sta` is async (returns after `esp_wifi_start`), and `wifi_wait_for_sta_ready` is a separate blocking wait on an event group with a caller-specified timeout.

## Design

### ESP32-C3 Receiver — Match the Transmitter

The receiver's current `wifi_start_sta()` blocks with a hardcoded 30s timeout (combines start + wait). To match the transmitter pattern without breaking existing callers, add one new function:

```c
esp_err_t wifi_start_sta_test(const char *ssid, const char *password, TickType_t timeout);
```

This function:
1. Initializes WiFi if needed (reuses `wifi_init_common()` + STA netif creation)
2. Configures STA with raw SSID/password strings (no `device_config_t` needed)
3. Starts WiFi and waits on the existing event group with the caller's `timeout`
4. Returns `ESP_OK` on connected, `ESP_FAIL` on auth failure/timeout

The existing `wifi_start_sta(const device_config_t *cfg)` stays unchanged. Cleanup uses the existing `wifi_stop()`.

The serial handler becomes nearly identical to the transmitter's:

```c
static void handle_test_wifi(const char *id, const cJSON *cmd_payload)
{
    if (cmd_payload == NULL) { send_error(id, "missing_payload"); return; }
    if (wifi_is_sta_connected()) { send_error(id, "already_connected"); return; }

    const char *ssid = json_get_string(cmd_payload, "wifi_ssid");
    if (ssid == NULL) { send_error(id, "missing_fields"); return; }
    const char *pwd = json_get_string(cmd_payload, "wifi_pwd");

    esp_err_t err = wifi_start_sta_test(ssid, pwd, pdMS_TO_TICKS(15000));

    cJSON *resp = cJSON_CreateObject();
    cJSON_AddBoolToObject(resp, "connected", err == ESP_OK);

    if (err == ESP_OK) {
        int rssi = 0;
        esp_wifi_sta_get_rssi(&rssi);
        cJSON_AddNumberToObject(resp, "rssi", rssi);

        esp_netif_t *netif = esp_netif_get_handle_from_ifkey("WIFI_STA_DEF");
        if (netif != NULL) {
            esp_netif_ip_info_t ip_info;
            if (esp_netif_get_ip_info(netif, &ip_info) == ESP_OK) {
                char ip_str[16];
                snprintf(ip_str, sizeof(ip_str), IPSTR, IP2STR(&ip_info.ip));
                cJSON_AddStringToObject(resp, "ip", ip_str);
            }
        }
    }

    send_response(id, true, resp, NULL);
    wifi_stop();
}
```

### ESP8266 Receiver — Adapted for RTOS SDK

The ESP8266's `wifi_start_sta(ssid, pwd)` also blocks (with `portMAX_DELAY`). Same approach — add a test variant:

```c
esp_err_t wifi_start_sta_test(const char *ssid, const char *password, TickType_t timeout);
```

Internally identical to `wifi_start_sta` but:
- Takes raw string params instead of extracted from the already-started WiFi
- Uses the caller's `timeout` instead of `portMAX_DELAY`
- Handles handler registration/retry reset

The serial handler uses the same structure but ESP8266-specific IP retrieval:

```c
static void handle_test_wifi(const char *id, const cJSON *payload)
{
    if (payload == NULL) { send_error(id, "missing_payload"); return; }

    const char *ssid = json_get_string(payload, "wifi_ssid");
    if (ssid == NULL) { send_error(id, "missing_fields"); return; }
    const char *pwd = json_get_string(payload, "wifi_pwd");

    esp_err_t err = wifi_start_sta_test(ssid, pwd, pdMS_TO_TICKS(15000));

    cJSON *resp = cJSON_CreateObject();
    cJSON_AddBoolToObject(resp, "connected", err == ESP_OK);

    if (err == ESP_OK) {
        wifi_ap_record_t ap;
        if (esp_wifi_sta_get_ap_info(&ap) == ESP_OK) {
            cJSON_AddNumberToObject(resp, "rssi", ap.rssi);
        }
        tcpip_adapter_ip_info_t ip_info;
        if (tcpip_adapter_get_ip_info(TCPIP_ADAPTER_IF_STA, &ip_info) == ESP_OK) {
            char ip_str[16];
            snprintf(ip_str, sizeof(ip_str), IPSTR, IP2STR(&ip_info.ip));
            cJSON_AddStringToObject(resp, "ip", ip_str);
        }
    }

    send_response(id, true, resp, NULL);
    wifi_stop();
}
```

## Platform-Specific APIs (verified with Context7)

| Operation | ESP-IDF v6.0 (ESP32-C3) | ESP8266 RTOS SDK |
|-----------|-------------------------|------------------|
| Get RSSI | `esp_wifi_sta_get_rssi(&rssi)` | `esp_wifi_sta_get_ap_info(&ap)` → `ap.rssi` |
| Get IP | `esp_netif_get_ip_info(netif, &ip)` | `tcpip_adapter_get_ip_info(IF_STA, &ip)` |
| Format IP | `IPSTR` + `IP2STR()` | `IPSTR` + `IP2STR()` |
| Cleanup | `wifi_stop()` (existing) | `wifi_stop()` (existing) |

Note: ESP-IDF v6.0 has `esp_wifi_sta_get_rssi()` for direct RSSI access. The ESP8266 SDK does not — use `esp_wifi_sta_get_ap_info()` instead.

## Changes Summary

### Receiver-ESP32

| File | Change |
|------|--------|
| `main/network/wifi.h` | Add `wifi_start_sta_test()` declaration |
| `main/network/wifi.c` | Implement `wifi_start_sta_test()` — reuses `wifi_init_common()`, event group, event handler; takes raw SSID/pwd + timeout |
| `main/serial/serial_protocol.c` | Replace `handle_test_wifi` stub with real implementation |

### Receiver-ESP8266

| File | Change |
|------|--------|
| `main/network/wifi.h` | Add `wifi_start_sta_test()` declaration |
| `main/network/wifi.c` | Implement `wifi_start_sta_test()` — reuses `wifi_init()`, event group, handlers; takes raw SSID/pwd + timeout |
| `main/serial/serial_protocol.c` | Replace `handle_test_wifi` stub with real implementation |

### Frontend

No changes needed. The dialog already sends `provision.test_wifi` and displays the result.

## Constraints

- `wifi_start_sta_test()` must not break the existing `wifi_start_sta()` — they share infrastructure (event group, event handler) but can't run concurrently (only one STA connection at a time)
- The `already_connected` guard in the ESP32-C3 handler (matching the transmitter) prevents testing while STA is active
- The ESP8266 handler skips the `already_connected` check since during provisioning mode WiFi STA is never running
- Password can be NULL (open WiFi) — both implementations handle this
- 15-second timeout matches the transmitter
- Cleanup via `wifi_stop()` (existing function on both platforms) — no new cleanup function needed
