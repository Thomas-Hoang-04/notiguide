# Receiver ESP8266 — Local Pairing Firmware Rewrite

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rewrite the ESP8266 (ESP-01) receiver firmware from a server-provisioned WiFi/MQTT device to a minimal ESP-NOW-paired 433 MHz RF pager. After pairing once with a transmitter hub, the receiver listens on 433 MHz ASK/OOK and vibrates on match. No WiFi network, no MQTT, no server contact.

**Architecture:** Same two-state machine as the ESP32-C3 receiver: **pairing** (ESP-NOW, WiFi radio on, blocking) → **listening** (433 MHz RF on, WiFi off). The ESP8266 is 433 MHz only — no nRF24 support.

**Tech Stack:** ESP8266_RTOS_SDK v3.x, ESP-NOW API, NVS, 433 MHz ASK receiver

**Spec:** `docs/spec/Local Pairing & Hub-Managed Dispatch Spec.md` — sections 3.1–3.7

**Self-contained note:** This plan is designed for implementation inside a Dev Container with the ESP8266_RTOS_SDK. It does not depend on the backend, frontend, or transmitter code.

---

## Key Differences from ESP32-C3 Receiver

| Area | ESP32-C3 | ESP8266 |
|------|----------|---------|
| SDK | ESP-IDF v6.0 | ESP8266_RTOS_SDK v3.x |
| Radio | 433 MHz OR nRF24 (Kconfig) | 433 MHz only |
| `rf_trigger_set()` | `(const uint8_t *code, size_t len, uint8_t bits, uint32_t ver)` | `(uint32_t code, uint8_t bits, uint32_t ver)` |
| `rf_trigger_init()` | `(gpio_num_t vibrator_gpio)` | `(VibratorHandler *vibrator)` |
| `rf_sup_start()` | `(const device_config_t *cfg)` | `(void)` — GPIO hardcoded to `GPIO_NUM_0` |
| `device_config_t.rf_code` | `uint8_t[16]` byte array | `uint32_t` scalar |
| ESP-NOW recv callback | `(const esp_now_recv_info_t *info, const uint8_t *data, int len)` | `(const uint8_t *mac_addr, const uint8_t *data, int len)` |
| ESP-NOW send callback | `(const esp_now_send_info_t *info, esp_now_send_status_t status)` | `(const uint8_t *mac_addr, esp_now_send_status_t status)` |
| Max encrypted peers | 17 (default 7) | 6 |
| `esp_mac.h` | Available | Not available — use `esp_wifi_get_mac()` directly |
| `esp_netif` | Yes | No — uses `tcpip_adapter` |
| HMAC-SHA256 | `mbedtls/md.h` — same API | `mbedtls/md.h` — same API |
| WiFi init | `esp_wifi_init(&WIFI_INIT_CONFIG_DEFAULT())` | Same |
| `esp_now_is_peer_exist()` | Available | Available |

---

## Retained Modules (DO NOT modify unless noted)

### `vibrator/vibrator.h` / `vibrator.c`

Same API as ESP32-C3 version. Contains its own `ESP_RETURN_ON_FALSE` macro shim for ESP8266 compatibility.

```c
esp_err_t vibrator_init(gpio_num_t vibrator_gpio, VibratorHandler *handler);
esp_err_t vibrator_deinit(VibratorHandler *handler);
esp_err_t vibrator_pulse(VibratorHandler *handler);
esp_err_t vibrator_set_pulsing(VibratorHandler *handler, bool enabled);
esp_err_t vibrator_toggle_pulsing(VibratorHandler *handler);
```

### `trigger/rf_trigger.h` / `rf_trigger.c`

Different API from ESP32-C3:

```c
esp_err_t rf_trigger_init(VibratorHandler *vibrator);   // takes VibratorHandler*, not gpio
void rf_trigger_set(uint32_t code, uint8_t bits, uint32_t version);  // uint32_t, not byte array
void rf_trigger_restore(uint32_t code, uint8_t bits, uint32_t version);
bool rf_trigger_has_code(void);
void rf_trigger_stop_output(void);
void rf_trigger_on_frame(uint32_t decoded, uint8_t decoded_bits, void *ctx);
```

### `trigger/rf_supervisor.h` / `rf_supervisor.c`

```c
esp_err_t rf_sup_start(void);   // no args, GPIO hardcoded to GPIO_NUM_0
esp_err_t rf_sup_suspend(void);
esp_err_t rf_sup_resume(void);
esp_err_t rf_sup_delete(void);
```

### `rf/rf_receiver.c`, `rf/rf_data.c`, `rf/rf_common.h`

433 MHz ASK receiver. Unchanged. Uses `RFHandler` struct, same as ESP32-C3 variant.

---

## ESP-NOW API Reference (ESP8266_RTOS_SDK)

```c
#include "esp_now.h"

esp_err_t esp_now_init(void);
esp_err_t esp_now_deinit(void);
esp_err_t esp_now_send(const uint8_t *peer_addr, const uint8_t *data, size_t len);
esp_err_t esp_now_add_peer(const esp_now_peer_info_t *peer);
esp_err_t esp_now_set_pmk(const uint8_t *pmk);
esp_err_t esp_now_register_recv_cb(esp_now_recv_cb_t cb);
esp_err_t esp_now_register_send_cb(esp_now_send_cb_t cb);

// ESP8266 callback signatures (different from ESP-IDF v6.0):
typedef void (*esp_now_recv_cb_t)(const uint8_t *mac_addr,
                                  const uint8_t *data, int data_len);
typedef void (*esp_now_send_cb_t)(const uint8_t *mac_addr,
                                  esp_now_send_status_t status);

typedef struct {
    uint8_t peer_addr[6];
    uint8_t lmk[16];
    uint8_t channel;
    wifi_interface_t ifidx;
    bool encrypt;
    void *priv;
} esp_now_peer_info_t;

#define ESP_NOW_MAX_DATA_LEN        250
#define ESP_NOW_ETH_ALEN            6
#define ESP_NOW_KEY_LEN             16
#define ESP_NOW_MAX_TOTAL_PEER_NUM  20
#define ESP_NOW_MAX_ENCRYPT_PEER_NUM 6
```

---

## Pairing Protocol Messages

Same packed struct definitions as the ESP32-C3 plan. See that plan's protocol section for the full struct definitions. The only runtime difference is the receive callback shim.

---

## NVS Schema (New)

Namespace: `pair`

| Key | NVS type | Description |
|-----|----------|-------------|
| `slot_id` | `u8` | Hub-assigned slot (1-based) |
| `hub_mac` | `blob(6)` | Hub's WiFi MAC |
| `rf_code` | `u32` | 32-bit 433 MHz trigger code |
| `rf_bits` | `u8` | Bit width (always 32 for ESP8266) |

Note: `rf_band` is not stored — ESP8266 is always 433 MHz.

---

### Task 1: Strip removed modules from build

**Files:**
- Modify: `receiver-esp8266/main/CMakeLists.txt`

- [ ] **Step 1: Remove server-dependent sources and embedded files**

Replace the current `idf_component_register` with:

```cmake
idf_component_register(
    SRCS "main.c"
         "config/device_config.c"
         "pair/espnow_pair.c"
         "trigger/rf_trigger.c"
         "trigger/rf_supervisor.c"
         "rf/rf_receiver.c"
         "rf/rf_data.c"
         "vibrator/vibrator.c"
    INCLUDE_DIRS "." "config" "rf" "trigger" "vibrator"
    REQUIRES esp8266 esp_common esp_event log mbedtls nvs_flash
)
```

Removed: `network/wifi.c`, `network/mqtt.c`, `serial/serial_protocol.c`, `security/device_identity.c`, `EMBED_TXTFILES` (mqtt_ca.pem), `mqtt`, `json`, `tcpip_adapter`, `app_update`.

Added: `pair/espnow_pair.c`.

- [ ] **Step 2: Delete removed source files**

```bash
rm -f main/network/wifi.c main/network/wifi.h
rm -f main/network/mqtt.c main/network/mqtt.h
rm -rf main/network/certs/
rm -f main/serial/serial_protocol.c main/serial/serial_protocol.h
rm -f main/security/device_identity.c main/security/device_identity.h
rmdir main/network main/serial main/security 2>/dev/null || true
```

---

### Task 2: Add Kconfig for pairing

**Files:**
- Modify: `receiver-esp8266/main/Kconfig.projbuild`

- [ ] **Step 1: Replace Kconfig with minimal pairing config**

```kconfig
menu "Receiver Configuration"

config RECEIVER_VIBRATOR_GPIO
    int "Vibrator GPIO"
    default 2
    help
        GPIO for the vibrator output. Default GPIO2 on ESP-01.

config RECEIVER_RF433_RX_GPIO
    int "433 MHz RX GPIO"
    default 0
    help
        GPIO for 433 MHz ASK receiver data input. Default GPIO0 on ESP-01.

config RECEIVER_PAIR_PSK
    string "Pairing pre-shared key (64 hex chars = 32 bytes)"
    default ""
    help
        Shared secret for HMAC-SHA256 verification during ESP-NOW
        pairing. Must be exactly 64 hexadecimal characters (32 bytes).
        Must match the transmitter hub's CONFIG_TRANSMITTER_PAIR_PSK.

config RECEIVER_PAIR_CHANNEL_SCAN_MS
    int "ESP-NOW channel scan dwell time (ms)"
    default 300
    range 100 1000
    help
        How long to listen on each WiFi channel during pairing scan.
        Full 13-channel scan takes this value x 13.

endmenu
```

---

### Task 3: Rewrite `device_config.h` / `device_config.c`

**Files:**
- Rewrite: `receiver-esp8266/main/config/device_config.h`
- Rewrite: `receiver-esp8266/main/config/device_config.c`

- [ ] **Step 1: Write new `device_config.h`**

```c
/**
 * @file device_config.h
 * @brief Minimal NVS-backed pairing configuration for ESP8266 receivers.
 */

#ifndef DEVICE_CONFIG_H
#define DEVICE_CONFIG_H

#include <stdbool.h>
#include <stdint.h>
#include "esp_err.h"

#define DEVICE_CONFIG_TAG             "DEVICE_CFG"
#define DEVICE_CONFIG_PAIR_NAMESPACE  "pair"

typedef struct {
    uint8_t  slot_id;
    uint8_t  hub_mac[6];
    uint32_t rf_code;       // 32-bit 433 MHz trigger code
    uint8_t  rf_code_bits;  // always 32
    bool     paired;
} device_config_t;

esp_err_t nvs_init_or_recover(void);
esp_err_t device_config_load(device_config_t *cfg);
esp_err_t device_config_save_pairing(device_config_t *cfg,
                                     uint8_t slot_id,
                                     const uint8_t hub_mac[6],
                                     uint32_t rf_code,
                                     uint8_t rf_bits);
esp_err_t device_config_factory_reset(void);

#endif /* DEVICE_CONFIG_H */
```

- [ ] **Step 2: Write new `device_config.c`**

```c
/**
 * @file device_config.c
 * @brief Minimal NVS-backed pairing configuration for ESP8266 receivers.
 */

#include "config/device_config.h"
#include <string.h>
#include "esp_log.h"
#include "nvs.h"
#include "nvs_flash.h"

#define TAG DEVICE_CONFIG_TAG

esp_err_t nvs_init_or_recover(void)
{
    esp_err_t err = nvs_flash_init();
    if (err == ESP_ERR_NVS_NO_FREE_PAGES) {
        ESP_LOGW(TAG, "NVS partition full, erasing");
        nvs_flash_erase();
        err = nvs_flash_init();
    }
    return err;
}

esp_err_t device_config_load(device_config_t *cfg)
{
    *cfg = (device_config_t){
        .slot_id = 0,
        .hub_mac = {0},
        .rf_code = 0,
        .rf_code_bits = 0,
        .paired = false,
    };

    nvs_handle_t handle;
    esp_err_t err = nvs_open(DEVICE_CONFIG_PAIR_NAMESPACE, NVS_READONLY, &handle);
    if (err == ESP_ERR_NVS_NOT_FOUND) {
        ESP_LOGI(TAG, "No pairing data, device is unpaired");
        return ESP_OK;
    }
    if (err != ESP_OK) {
        return err;
    }

    err = nvs_get_u8(handle, "slot_id", &cfg->slot_id);
    if (err == ESP_ERR_NVS_NOT_FOUND) {
        nvs_close(handle);
        return ESP_OK;
    }
    if (err != ESP_OK) {
        nvs_close(handle);
        return err;
    }

    size_t mac_len = 6;
    err = nvs_get_blob(handle, "hub_mac", cfg->hub_mac, &mac_len);
    if (err != ESP_OK) { nvs_close(handle); return err; }

    err = nvs_get_u32(handle, "rf_code", &cfg->rf_code);
    if (err != ESP_OK) { nvs_close(handle); return err; }

    err = nvs_get_u8(handle, "rf_bits", &cfg->rf_code_bits);
    if (err != ESP_OK) { nvs_close(handle); return err; }

    cfg->paired = true;
    ESP_LOGI(TAG, "Loaded pairing: slot=%u bits=%u", cfg->slot_id, cfg->rf_code_bits);

    nvs_close(handle);
    return ESP_OK;
}

esp_err_t device_config_save_pairing(device_config_t *cfg,
                                     uint8_t slot_id,
                                     const uint8_t hub_mac[6],
                                     uint32_t rf_code,
                                     uint8_t rf_bits)
{
    nvs_handle_t handle;
    esp_err_t err = nvs_open(DEVICE_CONFIG_PAIR_NAMESPACE, NVS_READWRITE, &handle);
    if (err != ESP_OK) { return err; }

    err = nvs_set_u8(handle, "slot_id", slot_id);
    if (err != ESP_OK) { nvs_close(handle); return err; }

    err = nvs_set_blob(handle, "hub_mac", hub_mac, 6);
    if (err != ESP_OK) { nvs_close(handle); return err; }

    err = nvs_set_u32(handle, "rf_code", rf_code);
    if (err != ESP_OK) { nvs_close(handle); return err; }

    err = nvs_set_u8(handle, "rf_bits", rf_bits);
    if (err != ESP_OK) { nvs_close(handle); return err; }

    err = nvs_commit(handle);
    nvs_close(handle);

    if (err == ESP_OK) {
        cfg->slot_id = slot_id;
        memcpy(cfg->hub_mac, hub_mac, 6);
        cfg->rf_code = rf_code;
        cfg->rf_code_bits = rf_bits;
        cfg->paired = true;
        ESP_LOGI(TAG, "Pairing saved: slot=%u", slot_id);
    }

    return err;
}

esp_err_t device_config_factory_reset(void)
{
    esp_err_t err = nvs_flash_erase();
    if (err == ESP_OK) {
        err = nvs_flash_init();
    }
    ESP_LOGW(TAG, "Factory reset %s", err == ESP_OK ? "complete" : "failed");
    return err;
}
```

---

### Task 4: Write ESP-NOW pairing module (ESP8266 variant)

**Files:**
- Create: `receiver-esp8266/main/pair/espnow_pair.h`
- Create: `receiver-esp8266/main/pair/espnow_pair.c`

- [ ] **Step 1: Create `pair/` directory**

```bash
mkdir -p main/pair
```

- [ ] **Step 2: Write `espnow_pair.h`**

```c
/**
 * @file espnow_pair.h
 * @brief ESP-NOW pairing state machine for ESP8266 receivers.
 */

#ifndef ESPNOW_PAIR_H
#define ESPNOW_PAIR_H

#include "config/device_config.h"
#include "esp_err.h"

esp_err_t espnow_pair_wait(device_config_t *cfg);

#endif /* ESPNOW_PAIR_H */
```

- [ ] **Step 3: Write `espnow_pair.c`**

Key differences from ESP32-C3 version:
- Uses `tcpip_adapter_init()` instead of `esp_netif_init()`
- ESP-NOW recv callback: `(const uint8_t *mac_addr, const uint8_t *data, int len)` — no `esp_now_recv_info_t`
- No `esp_mac.h` — use `esp_wifi_get_mac()` directly
- RF code stored as `uint32_t`, not byte array
- Band is always 433M
- `ESP_ERR_NVS_NEW_VERSION_FOUND` does not exist on ESP8266

```c
/**
 * @file espnow_pair.c
 * @brief ESP-NOW pairing state machine for ESP8266 receivers.
 */

#include "pair/espnow_pair.h"

#include <string.h>
#include "config/device_config.h"
#include "esp_event.h"
#include "esp_log.h"
#include "esp_now.h"
#include "esp_wifi.h"
#include "freertos/FreeRTOS.h"
#include "freertos/event_groups.h"
#include "mbedtls/md.h"
#include "sdkconfig.h"
#include "tcpip_adapter.h"

#ifndef ESP_RETURN_ON_ERROR
#define ESP_RETURN_ON_ERROR(x, tag, msg) do {                  \
        esp_err_t err_rc_ = (x);                               \
        if (err_rc_ != ESP_OK) {                                \
            ESP_LOGE(tag, "%s: %s", msg, esp_err_to_name(err_rc_)); \
            return err_rc_;                                     \
        }                                                       \
    } while(0)
#endif

#define TAG "pair"

#define PAIR_MSG_REQUEST    0x01
#define PAIR_MSG_CHALLENGE  0x02
#define PAIR_MSG_RESPONSE   0x03
#define PAIR_MSG_OFFER      0x04
#define PAIR_MSG_ACK        0x05
#define PAIR_MSG_CONFIRM    0x06

#define PAIR_BIT_CHALLENGE  BIT0
#define PAIR_BIT_OFFER      BIT1
#define PAIR_BIT_CONFIRM    BIT2

#define PSK_LEN  32
#define NONCE_LEN 16
#define HMAC_LEN 32

typedef struct __attribute__((packed)) {
    uint8_t msg_type;
    uint8_t rx_mac[6];
    uint8_t rx_band;
} pair_request_t;

typedef struct __attribute__((packed)) {
    uint8_t msg_type;
    uint8_t nonce[NONCE_LEN];
} pair_challenge_t;

typedef struct __attribute__((packed)) {
    uint8_t msg_type;
    uint8_t hmac[HMAC_LEN];
} pair_response_t;

typedef struct __attribute__((packed)) {
    uint8_t msg_type;
    uint8_t slot;
    uint8_t rf_band;
    uint8_t rf_code[16];
    uint8_t rf_code_len;
    uint8_t rf_bits;
} pair_offer_t;

typedef struct __attribute__((packed)) {
    uint8_t msg_type;
    uint8_t slot;
} pair_ack_t;

typedef struct __attribute__((packed)) {
    uint8_t msg_type;
} pair_confirm_t;

static EventGroupHandle_t s_pair_events;
static uint8_t s_hub_mac[6];
static uint8_t s_nonce[NONCE_LEN];
static pair_offer_t s_offer;

static bool hex_to_bytes(const char *hex, uint8_t *out, size_t out_len)
{
    size_t hex_len = strlen(hex);
    if (hex_len != out_len * 2) {
        return false;
    }
    for (size_t i = 0; i < out_len; i++) {
        unsigned int byte;
        if (sscanf(&hex[i * 2], "%2x", &byte) != 1) {
            return false;
        }
        out[i] = (uint8_t)byte;
    }
    return true;
}

static esp_err_t compute_hmac(const uint8_t *nonce, size_t nonce_len,
                              const uint8_t *psk, size_t psk_len,
                              uint8_t *hmac_out)
{
    const mbedtls_md_info_t *md = mbedtls_md_info_from_type(MBEDTLS_MD_SHA256);
    if (md == NULL) {
        return ESP_FAIL;
    }
    int ret = mbedtls_md_hmac(md, psk, psk_len, nonce, nonce_len, hmac_out);
    return ret == 0 ? ESP_OK : ESP_FAIL;
}

/* ESP8266 callback signature — no esp_now_recv_info_t */
static void recv_cb(const uint8_t *mac_addr, const uint8_t *data, int len)
{
    if (data == NULL || len < 1) {
        return;
    }

    switch (data[0]) {
    case PAIR_MSG_CHALLENGE:
        if ((size_t)len >= sizeof(pair_challenge_t)) {
            const pair_challenge_t *ch = (const pair_challenge_t *)data;
            memcpy(s_nonce, ch->nonce, NONCE_LEN);
            memcpy(s_hub_mac, mac_addr, 6);
            xEventGroupSetBits(s_pair_events, PAIR_BIT_CHALLENGE);
        }
        break;

    case PAIR_MSG_OFFER:
        if ((size_t)len >= sizeof(pair_offer_t)) {
            memcpy(&s_offer, data, sizeof(pair_offer_t));
            xEventGroupSetBits(s_pair_events, PAIR_BIT_OFFER);
        }
        break;

    case PAIR_MSG_CONFIRM:
        xEventGroupSetBits(s_pair_events, PAIR_BIT_CONFIRM);
        break;

    default:
        break;
    }
}

static esp_err_t wifi_init_sta_no_connect(void)
{
    tcpip_adapter_init();
    ESP_ERROR_CHECK(esp_event_loop_create_default());

    wifi_init_config_t wifi_cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_RETURN_ON_ERROR(esp_wifi_init(&wifi_cfg), TAG, "wifi init");
    ESP_RETURN_ON_ERROR(esp_wifi_set_mode(WIFI_MODE_STA), TAG, "wifi mode");
    ESP_RETURN_ON_ERROR(esp_wifi_start(), TAG, "wifi start");

    return ESP_OK;
}

static void wifi_deinit_all(void)
{
    esp_now_deinit();
    esp_wifi_stop();
    esp_wifi_deinit();
}

static esp_err_t add_broadcast_peer(void)
{
    esp_now_peer_info_t peer = {
        .peer_addr = {0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF},
        .channel = 0,
        .ifidx = ESP_IF_WIFI_STA,
        .encrypt = false,
    };
    return esp_now_add_peer(&peer);
}

static esp_err_t add_hub_peer(const uint8_t hub_mac[6])
{
    if (esp_now_is_peer_exist(hub_mac)) {
        return ESP_OK;
    }
    esp_now_peer_info_t peer = {
        .channel = 0,
        .ifidx = ESP_IF_WIFI_STA,
        .encrypt = false,
    };
    memcpy(peer.peer_addr, hub_mac, 6);
    return esp_now_add_peer(&peer);
}

static uint32_t bytes_to_u32(const uint8_t *buf, size_t len)
{
    uint32_t val = 0;
    for (size_t i = 0; i < len && i < 4; i++) {
        val |= ((uint32_t)buf[i]) << (i * 8);
    }
    return val;
}

esp_err_t espnow_pair_wait(device_config_t *cfg)
{
    uint8_t psk[PSK_LEN];
    const char *psk_hex = CONFIG_RECEIVER_PAIR_PSK;
    if (!hex_to_bytes(psk_hex, psk, PSK_LEN)) {
        ESP_LOGE(TAG, "Invalid PSK in Kconfig (need 64 hex chars)");
        return ESP_ERR_INVALID_ARG;
    }

    s_pair_events = xEventGroupCreate();
    if (s_pair_events == NULL) {
        return ESP_ERR_NO_MEM;
    }

    ESP_RETURN_ON_ERROR(wifi_init_sta_no_connect(), TAG, "wifi init");
    ESP_RETURN_ON_ERROR(esp_now_init(), TAG, "espnow init");
    ESP_RETURN_ON_ERROR(esp_now_register_recv_cb(recv_cb), TAG, "register recv cb");
    ESP_RETURN_ON_ERROR(add_broadcast_peer(), TAG, "add broadcast peer");

    ESP_LOGI(TAG, "Entering pair mode, scanning channels...");

    uint8_t my_mac[6];
    esp_wifi_get_mac(ESP_IF_WIFI_STA, my_mac);

    pair_request_t req = {
        .msg_type = PAIR_MSG_REQUEST,
        .rx_band = 0,  // always 433M on ESP8266
    };
    memcpy(req.rx_mac, my_mac, 6);

    const uint32_t scan_ms = CONFIG_RECEIVER_PAIR_CHANNEL_SCAN_MS;

    for (;;) {
        for (uint8_t ch = 1; ch <= 13; ch++) {
            esp_wifi_set_channel(ch, WIFI_SECOND_CHAN_NONE);
            esp_now_send(NULL, (const uint8_t *)&req, sizeof(req));

            EventBits_t bits = xEventGroupWaitBits(
                s_pair_events, PAIR_BIT_CHALLENGE,
                pdTRUE, pdFALSE, pdMS_TO_TICKS(scan_ms));

            if ((bits & PAIR_BIT_CHALLENGE) == 0) {
                continue;
            }

            ESP_LOGI(TAG, "Challenge on ch=%u from %02x:%02x:%02x:%02x:%02x:%02x",
                     ch, s_hub_mac[0], s_hub_mac[1], s_hub_mac[2],
                     s_hub_mac[3], s_hub_mac[4], s_hub_mac[5]);

            add_hub_peer(s_hub_mac);

            uint8_t hmac[HMAC_LEN];
            if (compute_hmac(s_nonce, NONCE_LEN, psk, PSK_LEN, hmac) != ESP_OK) {
                ESP_LOGE(TAG, "HMAC computation failed");
                continue;
            }

            pair_response_t resp = { .msg_type = PAIR_MSG_RESPONSE };
            memcpy(resp.hmac, hmac, HMAC_LEN);
            esp_now_send(s_hub_mac, (const uint8_t *)&resp, sizeof(resp));

            bits = xEventGroupWaitBits(
                s_pair_events, PAIR_BIT_OFFER,
                pdTRUE, pdFALSE, pdMS_TO_TICKS(15000));

            if ((bits & PAIR_BIT_OFFER) == 0) {
                ESP_LOGW(TAG, "No PAIR_OFFER within 15s, resuming scan");
                continue;
            }

            ESP_LOGI(TAG, "Offer: slot=%u bits=%u", s_offer.slot, s_offer.rf_bits);

            uint32_t rf_code = bytes_to_u32(s_offer.rf_code, s_offer.rf_code_len);

            esp_err_t save_err = device_config_save_pairing(
                cfg, s_offer.slot, s_hub_mac, rf_code, s_offer.rf_bits);

            if (save_err != ESP_OK) {
                ESP_LOGE(TAG, "Failed to save pairing");
                continue;
            }

            pair_ack_t ack = { .msg_type = PAIR_MSG_ACK, .slot = s_offer.slot };
            esp_now_send(s_hub_mac, (const uint8_t *)&ack, sizeof(ack));

            bits = xEventGroupWaitBits(
                s_pair_events, PAIR_BIT_CONFIRM,
                pdTRUE, pdFALSE, pdMS_TO_TICKS(10000));

            if ((bits & PAIR_BIT_CONFIRM) == 0) {
                ESP_LOGW(TAG, "No PAIR_CONFIRM within 10s, pairing saved anyway");
            }

            ESP_LOGI(TAG, "Pairing complete, slot=%u", s_offer.slot);

            vEventGroupDelete(s_pair_events);
            s_pair_events = NULL;
            wifi_deinit_all();
            return ESP_OK;
        }
    }
}
```

---

### Task 5: Rewrite `main.c`

**Files:**
- Rewrite: `receiver-esp8266/main/main.c`

- [ ] **Step 1: Write new `main.c`**

```c
/**
 * @file main.c
 * @brief ESP8266 receiver: pair if needed, then 433 MHz RF listen + vibrate.
 */

#include "config/device_config.h"
#include "esp_log.h"
#include "pair/espnow_pair.h"
#include "rf/rf_common.h"
#include "trigger/rf_supervisor.h"
#include "trigger/rf_trigger.h"
#include "vibrator/vibrator.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#define TAG "APP"

RFHandler g_rf = { .rx_active = false, .rx_suspended = false, .rx_gpio = -1 };
static device_config_t g_cfg = { .paired = false };
static VibratorHandler g_vibrator = { .vibrator_task_handle = NULL, .state_lock = NULL };

void app_main(void)
{
    ESP_ERROR_CHECK(nvs_init_or_recover());
    ESP_ERROR_CHECK(device_config_load(&g_cfg));

    ESP_ERROR_CHECK(vibrator_init((gpio_num_t)CONFIG_RECEIVER_VIBRATOR_GPIO, &g_vibrator));

    if (!g_cfg.paired) {
        ESP_LOGI(TAG, "Not paired, entering ESP-NOW pair mode");
        vibrator_pulse(&g_vibrator);
        vTaskDelay(pdMS_TO_TICKS(500));

        ESP_ERROR_CHECK(espnow_pair_wait(&g_cfg));
        ESP_LOGI(TAG, "Paired successfully");
    }

    ESP_ERROR_CHECK(rf_trigger_init(&g_vibrator));
    rf_recv_set_frame_callback(rf_trigger_on_frame, NULL);
    rf_trigger_stop_output();

    if (g_cfg.paired && g_cfg.rf_code_bits > 0) {
        rf_trigger_restore(g_cfg.rf_code, g_cfg.rf_code_bits, 1);
    }

    ESP_ERROR_CHECK(rf_sup_start());
    ESP_LOGI(TAG, "433 MHz listener started, waiting for triggers");

    for (;;) {
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

---

### Task 6: Build verification

- [ ] **Step 1: Run build**

```bash
idf.py build
```

Expected: Build succeeds. The ESP8266_RTOS_SDK will use `make` or `idf.py` depending on your setup. If using `make`:

```bash
make defconfig
make menuconfig  # set RECEIVER_PAIR_PSK
make -j$(nproc)
```

- [ ] **Step 2: Verify removed files don't interfere**

```bash
find main/ -name "*.c" -o -name "*.h" | sort
```

Expected: Only `main.c`, `config/`, `pair/`, `trigger/`, `rf/`, `vibrator/` directories remain.

