# WiFi Test for Receivers — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the `provision.test_wifi` serial command on both receiver platforms so users can validate WiFi credentials before provisioning.

**Architecture:** Each receiver's WiFi module gets a `wifi_start_sta_test()` function that does a one-shot STA connection with a caller-specified timeout, reusing existing WiFi infrastructure. The serial protocol handler calls it and reports RSSI + IP on success. The ESP32-C3 version mirrors the transmitter's pattern closely.

**Tech Stack:** ESP-IDF v6.0 (C), ESP8266 RTOS SDK (C), cJSON

**Spec:** `docs/planned/WiFi Test for Receivers Spec.md`

---

### Task 1: ESP32-C3 Receiver — WiFi Test Function

**Files:**
- Modify: `receiver-esp32/main/network/wifi.h`
- Modify: `receiver-esp32/main/network/wifi.c`

- [ ] **Step 1: Add declaration to wifi.h**

Add after the `wifi_is_sta_connected()` declaration (line 60):

```c
/**
 * @brief One-shot STA connection test with custom timeout.
 *
 * Connects to the given SSID, waits up to @p timeout for an IP, then
 * returns.  Caller must call wifi_stop() afterwards to clean up.
 *
 * @param ssid     Access point SSID (required)
 * @param password Access point password (NULL for open networks)
 * @param timeout  FreeRTOS tick timeout for the connection attempt
 * @return ESP_OK on successful connection, ESP_FAIL otherwise
 */
esp_err_t wifi_start_sta_test(const char *ssid, const char *password, TickType_t timeout);
```

- [ ] **Step 2: Implement wifi_start_sta_test in wifi.c**

Add after `wifi_is_sta_connected()` (line 254), before the closing of the file. This function follows the same structure as `wifi_start_sta()` (lines 135-180) but takes raw strings and a custom timeout instead of `device_config_t*` with a hardcoded 30s wait:

```c
esp_err_t wifi_start_sta_test(const char *ssid, const char *password, TickType_t timeout)
{
    ESP_RETURN_ON_FALSE(ssid != NULL, ESP_ERR_INVALID_ARG, WIFI_TAG, "ssid is NULL");

    ESP_RETURN_ON_ERROR(wifi_init_common(), WIFI_TAG, "wifi init failed");
    if (s_sta_netif == NULL) {
        s_sta_netif = esp_netif_create_default_wifi_sta();
        ESP_RETURN_ON_FALSE(s_sta_netif != NULL, ESP_FAIL, WIFI_TAG, "failed to create STA netif");
    }
    ESP_RETURN_ON_ERROR(wifi_prepare_mode(RECEIVER_WIFI_MODE_STA), WIFI_TAG,
                        "failed to prepare STA mode");

    wifi_config_t sta_cfg = {
        .sta = {
            .scan_method = WIFI_ALL_CHANNEL_SCAN,
            .threshold.authmode = WIFI_AUTH_WPA2_WPA3_PSK,
            .pmf_cfg = { .required = true },
            .sae_pwe_h2e = WPA3_SAE_PWE_BOTH,
            .failure_retry_cnt = CONFIG_RECEIVER_WIFI_MAX_RETRY,
            .sae_h2e_identifier = "notiguide-recv",
        },
    };

    strlcpy((char *)sta_cfg.sta.ssid, ssid, sizeof(sta_cfg.sta.ssid));
    if (password != NULL) {
        strlcpy((char *)sta_cfg.sta.password, password, sizeof(sta_cfg.sta.password));
    }

    ESP_RETURN_ON_ERROR(esp_wifi_set_mode(WIFI_MODE_STA), WIFI_TAG, "esp_wifi_set_mode failed");
    ESP_RETURN_ON_ERROR(esp_wifi_set_config(WIFI_IF_STA, &sta_cfg), WIFI_TAG,
                        "esp_wifi_set_config failed");
    ESP_RETURN_ON_ERROR(esp_wifi_start(), WIFI_TAG, "esp_wifi_start failed");

    EventBits_t bits = xEventGroupWaitBits(s_events,
                                           WIFI_EVENT_CONNECTED_BIT | WIFI_EVENT_FAILED_BIT,
                                           pdFALSE,
                                           pdFALSE,
                                           timeout);
    if ((bits & WIFI_EVENT_CONNECTED_BIT) == 0) {
        return ESP_FAIL;
    }

    return ESP_OK;
}
```

Note: no `esp_wifi_set_ps()` call — this is a test connection that will be stopped immediately.

---

### Task 2: ESP32-C3 Receiver — Serial Test WiFi Handler

**Files:**
- Modify: `receiver-esp32/main/serial/serial_protocol.c`

- [ ] **Step 1: Add required includes**

Check the existing includes at the top of the file. Add these if not already present:

```c
#include "network/wifi.h"
#include "esp_wifi.h"
#include "esp_netif.h"
```

- [ ] **Step 2: Replace the handle_test_wifi stub**

Replace the current stub (lines 186-190):

```c
static void handle_test_wifi(const char *id, const cJSON *cmd_payload)
{
    (void)cmd_payload;
    send_error(id, "not_supported");
}
```

With the full implementation matching the transmitter's pattern (transmitter/main/serial/serial_protocol.c:307-362):

```c
static void handle_test_wifi(const char *id, const cJSON *cmd_payload)
{
    if (cmd_payload == NULL) {
        send_error(id, "missing_payload");
        return;
    }

    if (wifi_is_sta_connected()) {
        send_error(id, "already_connected");
        return;
    }

    const char *ssid = json_get_string(cmd_payload, "wifi_ssid");
    if (ssid == NULL) {
        send_error(id, "missing_fields");
        return;
    }
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

---

### Task 3: ESP8266 Receiver — WiFi Test Function

**Files:**
- Modify: `receiver-esp8266/main/network/wifi.h`
- Modify: `receiver-esp8266/main/network/wifi.c`

- [ ] **Step 1: Add declaration to wifi.h**

Add after the `wifi_get_mac_label()` declaration (line 21):

```c
esp_err_t wifi_start_sta_test(const char *ssid, const char *password, TickType_t timeout);
```

Also add the required include for `TickType_t` if not already present:

```c
#include "freertos/FreeRTOS.h"
```

- [ ] **Step 2: Implement wifi_start_sta_test in wifi.c**

Add after `wifi_start_sta()` (line 232), before `wifi_stop()`. This follows the same structure as `wifi_start_sta()` (lines 150-232) but uses a caller-specified timeout instead of `portMAX_DELAY`, and handles NULL password for open networks:

```c
esp_err_t wifi_start_sta_test(const char *ssid, const char *password, TickType_t timeout)
{
    esp_err_t err;
    EventBits_t bits;
    wifi_config_t wifi_cfg = {
        .sta = {
            .scan_method = WIFI_FAST_SCAN,
            .sort_method = WIFI_CONNECT_AP_BY_SIGNAL,
            .threshold = {
                .rssi = -127,
                .authmode = WIFI_AUTH_WPA2_PSK,
            },
            .pmf_cfg = {
                .capable = true,
                .required = false,
            },
        },
    };

    if (!ssid) {
        return ESP_ERR_INVALID_ARG;
    }

    err = wifi_init();
    if (err != ESP_OK) {
        return err;
    }

    if (!s_wifi_events) {
        s_wifi_events = xEventGroupCreate();
        if (!s_wifi_events) {
            return ESP_ERR_NO_MEM;
        }
    }
    xEventGroupClearBits(s_wifi_events, WIFI_CONNECTED_BIT | WIFI_FAIL_BIT);
    s_retry_num = 0;

    err = wifi_register_handlers();
    if (err != ESP_OK) {
        return err;
    }

    snprintf((char *)wifi_cfg.sta.ssid, sizeof(wifi_cfg.sta.ssid), "%s", ssid);
    if (password != NULL) {
        snprintf((char *)wifi_cfg.sta.password, sizeof(wifi_cfg.sta.password), "%s", password);
    }
#if CONFIG_ESP8266_WIFI_ENABLE_WPA3_SAE
    if (password != NULL && strlen(password) > 0) {
        wifi_cfg.sta.threshold.authmode = WIFI_AUTH_WPA2_WPA3_PSK;
    }
#endif

    err = esp_wifi_set_mode(WIFI_MODE_STA);
    if (err != ESP_OK) {
        ESP_LOGE(WIFI_TAG, "set_mode STA failed: %s", esp_err_to_name(err));
        return err;
    }

    err = esp_wifi_set_config(ESP_IF_WIFI_STA, &wifi_cfg);
    if (err != ESP_OK) {
        ESP_LOGE(WIFI_TAG, "set_config STA failed: %s", esp_err_to_name(err));
        return err;
    }

    ESP_LOGI(WIFI_TAG, "STA test starting, connecting to %s", ssid);
    err = esp_wifi_start();
    if (err != ESP_OK && err != ESP_ERR_WIFI_CONN) {
        ESP_LOGE(WIFI_TAG, "esp_wifi_start failed: %s", esp_err_to_name(err));
        return err;
    }

    bits = xEventGroupWaitBits(s_wifi_events,
                               WIFI_CONNECTED_BIT | WIFI_FAIL_BIT,
                               pdTRUE,
                               pdFALSE,
                               timeout);
    if (bits & WIFI_CONNECTED_BIT) {
        return ESP_OK;
    }

    return ESP_FAIL;
}
```

Key difference from `wifi_start_sta()`: uses `timeout` parameter instead of `portMAX_DELAY`, accepts NULL password, no power save setup.

---

### Task 4: ESP8266 Receiver — Serial Test WiFi Handler

**Files:**
- Modify: `receiver-esp8266/main/serial/serial_protocol.c`

- [ ] **Step 1: Add required includes**

Check the existing includes. Add these if not already present:

```c
#include "network/wifi.h"
#include "esp_wifi.h"
#include "tcpip_adapter.h"
```

- [ ] **Step 2: Replace the handle_test_wifi stub**

The current ESP8266 stub (lines 171-174) has a different signature — it takes only `id` (no payload parameter). The dispatch at line 245 also doesn't pass payload. Both need updating.

Replace the stub:

```c
static void handle_test_wifi(const char *id)
{
    send_error(id, "not_supported");
}
```

With:

```c
static void handle_test_wifi(const char *id, const cJSON *payload)
{
    if (payload == NULL) {
        send_error(id, "missing_payload");
        return;
    }

    const char *ssid = json_get_string(payload, "wifi_ssid");
    if (ssid == NULL) {
        send_error(id, "missing_fields");
        return;
    }
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

- [ ] **Step 3: Update the dispatch call site**

Find the dispatch line (around line 245) that calls `handle_test_wifi(id)` and add the `payload` argument:

```c
// Was:
handle_test_wifi(id);

// Becomes:
handle_test_wifi(id, payload);
```

The `payload` variable is the `cJSON *payload = cJSON_GetObjectItem(root, "payload")` extracted from the parsed command — it's already available in the dispatch scope.

---

### Task 5: Changelog

- [ ] **Step 1: Add entry to docs/CHANGELOGS.md**

Add under the existing "USB Provisioning Receiver Extension" entry for 2026-05-22:

```markdown
### WiFi Test for Receivers
- **`receiver-esp32/main/network/wifi.c`**: Added `wifi_start_sta_test(ssid, pwd, timeout)` — one-shot STA test reusing existing WiFi infrastructure with caller-specified timeout
- **`receiver-esp32/main/serial/serial_protocol.c`**: Implemented `handle_test_wifi` — mirrors transmitter pattern (`already_connected` guard, `esp_wifi_sta_get_rssi`, `esp_netif_get_ip_info`, `wifi_stop`)
- **`receiver-esp8266/main/network/wifi.c`**: Added `wifi_start_sta_test(ssid, pwd, timeout)` — same pattern adapted for ESP8266 RTOS SDK (`portMAX_DELAY` → custom timeout, NULL password support)
- **`receiver-esp8266/main/serial/serial_protocol.c`**: Implemented `handle_test_wifi` — uses `esp_wifi_sta_get_ap_info` for RSSI, `tcpip_adapter_get_ip_info` for IP
```
