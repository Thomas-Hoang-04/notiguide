# Receiver ESP32-C3 — Local Pairing Firmware Rewrite

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rewrite the ESP32-C3 receiver firmware from a server-provisioned WiFi/MQTT device to a minimal ESP-NOW-paired RF pager. After pairing once with a transmitter hub, the receiver listens on RF (433 MHz or nRF24) and vibrates on match. No WiFi network, no MQTT, no server contact.

**Architecture:** Two-state machine: **pairing** (ESP-NOW, WiFi radio on, blocking) → **listening** (RF radio on, WiFi off). Transition is one-time. Factory reset returns to pairing state. The pairing protocol uses a 6-message handshake with HMAC-SHA256 PSK verification over ESP-NOW's CCMP-encrypted channel.

**Tech Stack:** ESP-IDF v6.0, ESP-NOW API, NVS, 433 MHz ASK (Kconfig `RECEIVER_RADIO_433M`) or nRF24L01 PRX (Kconfig `RECEIVER_RADIO_2_4G`)

**Spec:** `docs/spec/Local Pairing & Hub-Managed Dispatch Spec.md` — sections 3.1–3.7

**Self-contained note:** This plan is designed for implementation inside a Dev Container with the ESP-IDF v6.0 SDK. It does not depend on the backend, frontend, or transmitter code. All required context (retained module APIs, NVS schema, pairing protocol) is included below.

---

## Retained Modules (DO NOT modify unless noted)

These existing source files are carried over unchanged. Their public APIs are listed here for reference when writing new code.

### `vibrator/vibrator.h` / `vibrator.c`

```c
esp_err_t vibrator_init(gpio_num_t vibrator_gpio, VibratorHandler *handler);
esp_err_t vibrator_deinit(VibratorHandler *handler);
esp_err_t vibrator_pulse(VibratorHandler *handler);        // one-shot pulse
esp_err_t vibrator_set_pulsing(VibratorHandler *handler, bool enabled);
esp_err_t vibrator_toggle_pulsing(VibratorHandler *handler);
```

### `trigger/rf_trigger.h` / `rf_trigger.c`

```c
esp_err_t rf_trigger_init(gpio_num_t vibrator_gpio);
esp_err_t rf_trigger_set(const uint8_t *code, size_t code_len, uint8_t bits, uint32_t version);
void rf_trigger_restore_from_cfg(const device_config_t *cfg);
bool rf_trigger_has_code(void);
esp_err_t rf_trigger_stop_output(void);
void rf_trigger_on_frame(uint32_t value, uint8_t value_bits);   // 433 MHz
void rf_trigger_on_packet(const uint8_t *pkt, size_t pkt_len);  // nRF24
```

**Important:** `rf_trigger_restore_from_cfg()` reads `cfg->rf_code`, `cfg->rf_code_len`, `cfg->rf_code_bits`, `cfg->rf_code_ver`. The new `device_config_t` must provide these fields.

### `trigger/rf_supervisor.h` / `rf_supervisor.c`

```c
esp_err_t rf_sup_start(const device_config_t *cfg);    // starts 433M or nRF24 based on Kconfig
esp_err_t rf_sup_suspend(void);
esp_err_t rf_sup_resume(void);
esp_err_t rf_sup_delete(void);
```

For `RECEIVER_RADIO_2_4G`, `rf_sup_start()` reads `cfg->rf_code` as a 5-byte nRF24 RX address and calls `nrf24_recv_start_task()`. For `RECEIVER_RADIO_433M`, `cfg` is unused (GPIO comes from Kconfig).

### `rf/rf_receiver.c`, `rf/rf_data.c`, `rf/rf_common.h`

433 MHz ASK receiver. Unchanged.

### `nrf24/nrf24_receiver.c`, `nrf24/nrf24_receiver.h`

nRF24L01 PRX driver. Unchanged.

---

## ESP-NOW API Reference (ESP-IDF v6.0)

```c
#include "esp_now.h"

esp_err_t esp_now_init(void);
esp_err_t esp_now_deinit(void);
esp_err_t esp_now_send(const uint8_t *peer_addr, const uint8_t *data, size_t len);
esp_err_t esp_now_add_peer(const esp_now_peer_info_t *peer);
esp_err_t esp_now_set_pmk(const uint8_t *pmk);
esp_err_t esp_now_register_recv_cb(esp_now_recv_cb_t cb);
esp_err_t esp_now_register_send_cb(esp_now_send_cb_t cb);

// Receive callback (ESP-IDF v6.0 signature):
typedef void (*esp_now_recv_cb_t)(const esp_now_recv_info_t *info,
                                  const uint8_t *data, int data_len);

typedef struct {
    uint8_t *src_addr;    // sender MAC
    uint8_t *des_addr;    // destination MAC
    wifi_pkt_rx_ctrl_t *rx_ctrl;
} esp_now_recv_info_t;

typedef struct {
    uint8_t peer_addr[6];
    uint8_t lmk[16];
    uint8_t channel;
    wifi_interface_t ifidx;
    bool encrypt;
    void *priv;
} esp_now_peer_info_t;

#define ESP_NOW_MAX_DATA_LEN   250
#define ESP_NOW_ETH_ALEN       6
#define ESP_NOW_KEY_LEN        16
```

---

## Pairing Protocol Messages

All messages are packed structs sent via `esp_now_send()`. Max payload: 250 bytes.

```c
// Message type IDs
#define PAIR_MSG_REQUEST    0x01
#define PAIR_MSG_CHALLENGE  0x02
#define PAIR_MSG_RESPONSE   0x03
#define PAIR_MSG_OFFER      0x04
#define PAIR_MSG_ACK        0x05
#define PAIR_MSG_CONFIRM    0x06

// Receiver → Hub
typedef struct __attribute__((packed)) {
    uint8_t msg_type;           // PAIR_MSG_REQUEST
    uint8_t rx_mac[6];          // receiver's WiFi MAC
    uint8_t rx_band;            // 0=433M, 1=2.4G
} pair_request_t;               // 8 bytes

// Hub → Receiver
typedef struct __attribute__((packed)) {
    uint8_t msg_type;           // PAIR_MSG_CHALLENGE
    uint8_t nonce[16];          // random nonce
} pair_challenge_t;             // 17 bytes

// Receiver → Hub
typedef struct __attribute__((packed)) {
    uint8_t msg_type;           // PAIR_MSG_RESPONSE
    uint8_t hmac[32];           // HMAC-SHA256(nonce, PSK)
} pair_response_t;              // 33 bytes

// Hub → Receiver
typedef struct __attribute__((packed)) {
    uint8_t msg_type;           // PAIR_MSG_OFFER
    uint8_t slot;               // 1-based slot number
    uint8_t rf_band;            // 0=433M, 1=2.4G
    uint8_t rf_code[16];        // trigger code (4B for 433M, 5B for nRF24)
    uint8_t rf_code_len;        // valid bytes in rf_code
    uint8_t rf_bits;            // bit width (32 for 433M, 40 for nRF24)
} pair_offer_t;                 // 22 bytes

// Receiver → Hub
typedef struct __attribute__((packed)) {
    uint8_t msg_type;           // PAIR_MSG_ACK
    uint8_t slot;               // echo back slot
} pair_ack_t;                   // 2 bytes

// Hub → Receiver
typedef struct __attribute__((packed)) {
    uint8_t msg_type;           // PAIR_MSG_CONFIRM
} pair_confirm_t;               // 1 byte
```

---

## NVS Schema (New)

Namespace: `pair`

| Key | NVS type | Description |
|-----|----------|-------------|
| `slot_id` | `u8` | Hub-assigned slot (1-based) |
| `hub_mac` | `blob(6)` | Hub's WiFi MAC |
| `rf_code` | `blob(≤16)` | Trigger code |
| `rf_bits` | `u8` | Bit width (32 or 40) |
| `rf_band` | `u8` | 0=433M, 1=2.4G |
| `rf_code_ver` | `u32` | Monotonic trigger code version (set to 1 on pair) |

---

### Task 1: Strip removed modules from build

**Files:**
- Modify: `receiver-esp32/main/CMakeLists.txt`

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
         "nrf24/nrf24_receiver.c"
         "vibrator/vibrator.c"
    INCLUDE_DIRS "."
    PRIV_REQUIRES nvs_flash esp_wifi esp_netif esp_event
                  esp_driver_gpio esp_driver_spi esp_timer esp_hw_support
                  mbedtls
)
```

Removed: `network/wifi.c`, `network/mqtt.c`, `provision/http_server.c`, `security/device_identity.c`, `serial/serial_protocol.c`, `EMBED_TXTFILES` (mqtt_ca.pem), `EMBED_FILES` (index.html.gz), `espressif__mqtt`, `espressif__cjson`, `esp_http_server`, `esp_driver_usb_serial_jtag`.

Added: `pair/espnow_pair.c`, `mbedtls` (for HMAC-SHA256).

- [ ] **Step 2: Update `idf_component.yml`**

Replace contents with:

```yaml
dependencies:
  idf:
    version: '>=6.0.0'
```

Removed: `espressif/mqtt` and `espressif/cjson` managed components.

- [ ] **Step 3: Delete removed source files**

```bash
rm -f main/network/wifi.c main/network/wifi.h
rm -f main/network/mqtt.c main/network/mqtt.h
rm -rf main/network/certs/
rm -f main/provision/http_server.c main/provision/http_server.h
rm -f main/provision/index.html.gz
rm -f main/security/device_identity.c main/security/device_identity.h
rm -f main/serial/serial_protocol.c main/serial/serial_protocol.h
rmdir main/network main/provision main/security main/serial 2>/dev/null || true
```

---

### Task 2: Add Kconfig for pairing

**Files:**
- Modify: `receiver-esp32/main/Kconfig.projbuild`

- [ ] **Step 1: Replace Kconfig with minimal pairing config**

Keep the radio, nRF24, and vibrator GPIO sections. Remove WiFi STA retry, SoftAP, bootstrap timeout, MQTT topic prefix, backend pubkey, and firmware version. Add pairing PSK and ESP-NOW channel.

Replace the full file contents with:

```kconfig
menu "Receiver Firmware"

choice RECEIVER_RADIO_VARIANT
    prompt "Active radio"
    default RECEIVER_RADIO_433M

config RECEIVER_RADIO_433M
    bool "433 MHz ASK/OOK receiver"

config RECEIVER_RADIO_2_4G
    bool "nRF24L01 / nRF24L01+ 2.4 GHz"

endchoice

config RECEIVER_RF433_RX_GPIO
    int "433 MHz RX GPIO"
    default 4
    depends on RECEIVER_RADIO_433M

config RECEIVER_NRF24_SCK
    int "nRF24 SCK GPIO"
    default 6
    depends on RECEIVER_RADIO_2_4G

config RECEIVER_NRF24_MOSI
    int "nRF24 MOSI GPIO"
    default 7
    depends on RECEIVER_RADIO_2_4G

config RECEIVER_NRF24_MISO
    int "nRF24 MISO GPIO"
    default 2
    depends on RECEIVER_RADIO_2_4G

config RECEIVER_NRF24_CS
    int "nRF24 CS GPIO"
    default 5
    depends on RECEIVER_RADIO_2_4G

config RECEIVER_NRF24_CE
    int "nRF24 CE GPIO"
    default 3
    depends on RECEIVER_RADIO_2_4G

config RECEIVER_NRF24_IRQ
    int "nRF24 IRQ GPIO"
    default 10
    depends on RECEIVER_RADIO_2_4G

config RECEIVER_NRF24_CHANNEL
    int "nRF24 RF channel"
    default 100
    range 97 125
    depends on RECEIVER_RADIO_2_4G
    help
        nRF24 RF channel: f = 2400 MHz + N. Must match the transmitter.

choice RECEIVER_NRF24_DATA_RATE
    prompt "nRF24 air data rate"
    default RECEIVER_NRF24_DR_1M
    depends on RECEIVER_RADIO_2_4G

config RECEIVER_NRF24_DR_1M
    bool "1 Mbps"

config RECEIVER_NRF24_DR_2M
    bool "2 Mbps"

endchoice

config RECEIVER_NRF24_ENABLE_DPL
    bool "Enable Dynamic Payload Length"
    default y
    depends on RECEIVER_RADIO_2_4G

config RECEIVER_VIBRATOR_GPIO
    int "Vibrator GPIO"
    default 1

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
        Full 13-channel scan takes this value × 13.

endmenu
```

---

### Task 3: Rewrite `device_config.h` / `device_config.c`

**Files:**
- Rewrite: `receiver-esp32/main/config/device_config.h`
- Rewrite: `receiver-esp32/main/config/device_config.c`

- [ ] **Step 1: Write new `device_config.h`**

```c
/**
 * @file device_config.h
 * @brief Minimal NVS-backed pairing configuration for locally-paired receivers.
 */

#ifndef DEVICE_CONFIG_H
#define DEVICE_CONFIG_H

#include <stdbool.h>
#include <stddef.h>
#include <stdint.h>
#include "esp_err.h"

#define DEVICE_CONFIG_MAX_RF_CODE_LEN 16U
#define DEVICE_CONFIG_PAIR_NAMESPACE  "pair"

typedef struct {
    uint8_t slot_id;
    uint8_t hub_mac[6];
    uint8_t rf_code[DEVICE_CONFIG_MAX_RF_CODE_LEN];
    size_t  rf_code_len;
    uint8_t rf_code_bits;
    uint32_t rf_code_ver;
    uint8_t rf_band;        // 0=433M, 1=2.4G
    bool    paired;
} device_config_t;

/**
 * @brief Initialize NVS or recover from corruption.
 */
esp_err_t nvs_init_or_recover(void);

/**
 * @brief Load pairing config from NVS into RAM.
 */
esp_err_t device_config_load(device_config_t *cfg);

/**
 * @brief Persist a completed pairing to NVS.
 */
esp_err_t device_config_save_pairing(device_config_t *cfg,
                                     uint8_t slot_id,
                                     const uint8_t hub_mac[6],
                                     const uint8_t *rf_code,
                                     size_t rf_code_len,
                                     uint8_t rf_bits,
                                     uint8_t rf_band,
                                     uint32_t rf_code_ver);

/**
 * @brief Erase the pairing namespace (factory reset).
 */
esp_err_t device_config_factory_reset(void);

/**
 * @brief Check if rf_code fields are populated and valid.
 */
bool device_config_has_rf_code(const device_config_t *cfg);

#endif /* DEVICE_CONFIG_H */
```

- [ ] **Step 2: Write new `device_config.c`**

```c
/**
 * @file device_config.c
 * @brief Minimal NVS-backed pairing configuration.
 */

#include "config/device_config.h"
#include <string.h>
#include "esp_check.h"
#include "esp_log.h"
#include "nvs.h"
#include "nvs_flash.h"

#define TAG "device_config"

esp_err_t nvs_init_or_recover(void)
{
    esp_err_t err = nvs_flash_init();
    if (err == ESP_ERR_NVS_NO_FREE_PAGES || err == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_LOGW(TAG, "NVS partition truncated, erasing");
        ESP_RETURN_ON_ERROR(nvs_flash_erase(), TAG, "erase");
        err = nvs_flash_init();
    }
    return err;
}

esp_err_t device_config_load(device_config_t *cfg)
{
    memset(cfg, 0, sizeof(*cfg));

    nvs_handle_t handle;
    esp_err_t err = nvs_open(DEVICE_CONFIG_PAIR_NAMESPACE, NVS_READONLY, &handle);
    if (err == ESP_ERR_NVS_NOT_FOUND) {
        ESP_LOGI(TAG, "No pairing data, device is unpaired");
        return ESP_OK;
    }
    ESP_RETURN_ON_ERROR(err, TAG, "open pair namespace");

    err = nvs_get_u8(handle, "slot_id", &cfg->slot_id);
    if (err == ESP_ERR_NVS_NOT_FOUND) {
        nvs_close(handle);
        return ESP_OK;
    }
    ESP_GOTO_ON_ERROR(err, cleanup, TAG, "read slot_id");

    size_t mac_len = 6;
    ESP_GOTO_ON_ERROR(nvs_get_blob(handle, "hub_mac", cfg->hub_mac, &mac_len),
                      cleanup, TAG, "read hub_mac");

    size_t code_len = sizeof(cfg->rf_code);
    ESP_GOTO_ON_ERROR(nvs_get_blob(handle, "rf_code", cfg->rf_code, &code_len),
                      cleanup, TAG, "read rf_code");
    cfg->rf_code_len = code_len;

    ESP_GOTO_ON_ERROR(nvs_get_u8(handle, "rf_bits", &cfg->rf_code_bits),
                      cleanup, TAG, "read rf_bits");
    ESP_GOTO_ON_ERROR(nvs_get_u8(handle, "rf_band", &cfg->rf_band),
                      cleanup, TAG, "read rf_band");

    uint32_t code_ver = 0;
    if (nvs_get_u32(handle, "rf_code_ver", &code_ver) == ESP_OK) {
        cfg->rf_code_ver = code_ver;
    }

    cfg->paired = true;
    ESP_LOGI(TAG, "Loaded pairing: slot=%u band=%s bits=%u",
             cfg->slot_id, cfg->rf_band == 0 ? "433M" : "2.4G", cfg->rf_code_bits);

cleanup:
    nvs_close(handle);
    return err;
}

esp_err_t device_config_save_pairing(device_config_t *cfg,
                                     uint8_t slot_id,
                                     const uint8_t hub_mac[6],
                                     const uint8_t *rf_code,
                                     size_t rf_code_len,
                                     uint8_t rf_bits,
                                     uint8_t rf_band,
                                     uint32_t rf_code_ver)
{
    nvs_handle_t handle;
    ESP_RETURN_ON_ERROR(nvs_open(DEVICE_CONFIG_PAIR_NAMESPACE, NVS_READWRITE, &handle),
                        TAG, "open pair namespace");

    esp_err_t err = ESP_OK;
    ESP_GOTO_ON_ERROR(nvs_set_u8(handle, "slot_id", slot_id), cleanup, TAG, "write slot_id");
    ESP_GOTO_ON_ERROR(nvs_set_blob(handle, "hub_mac", hub_mac, 6), cleanup, TAG, "write hub_mac");
    ESP_GOTO_ON_ERROR(nvs_set_blob(handle, "rf_code", rf_code, rf_code_len), cleanup, TAG, "write rf_code");
    ESP_GOTO_ON_ERROR(nvs_set_u8(handle, "rf_bits", rf_bits), cleanup, TAG, "write rf_bits");
    ESP_GOTO_ON_ERROR(nvs_set_u8(handle, "rf_band", rf_band), cleanup, TAG, "write rf_band");
    ESP_GOTO_ON_ERROR(nvs_set_u32(handle, "rf_code_ver", rf_code_ver), cleanup, TAG, "write rf_code_ver");
    ESP_GOTO_ON_ERROR(nvs_commit(handle), cleanup, TAG, "commit");

    cfg->slot_id = slot_id;
    memcpy(cfg->hub_mac, hub_mac, 6);
    memcpy(cfg->rf_code, rf_code, rf_code_len);
    cfg->rf_code_len = rf_code_len;
    cfg->rf_code_bits = rf_bits;
    cfg->rf_code_ver = rf_code_ver;
    cfg->rf_band = rf_band;
    cfg->paired = true;

    ESP_LOGI(TAG, "Pairing saved: slot=%u band=%s", slot_id, rf_band == 0 ? "433M" : "2.4G");

cleanup:
    nvs_close(handle);
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

bool device_config_has_rf_code(const device_config_t *cfg)
{
    return cfg != NULL && cfg->paired && cfg->rf_code_len > 0 && cfg->rf_code_bits > 0;
}
```

---

### Task 4: Write ESP-NOW pairing module

**Files:**
- Create: `receiver-esp32/main/pair/espnow_pair.h`
- Create: `receiver-esp32/main/pair/espnow_pair.c`

- [ ] **Step 1: Create `pair/` directory**

```bash
mkdir -p main/pair
```

- [ ] **Step 2: Write `espnow_pair.h`**

```c
/**
 * @file espnow_pair.h
 * @brief ESP-NOW pairing state machine for locally-paired receivers.
 */

#ifndef ESPNOW_PAIR_H
#define ESPNOW_PAIR_H

#include "config/device_config.h"
#include "esp_err.h"

/**
 * @brief Block until paired with a transmitter hub.
 *
 * Initializes WiFi (STA, no connection) and ESP-NOW. Scans channels
 * 1-13, broadcasts PAIR_REQUEST, completes the 6-message handshake,
 * and persists the binding to NVS. Deinitializes WiFi before returning.
 *
 * @param cfg Device config to populate on successful pairing
 * @return ESP_OK on success
 */
esp_err_t espnow_pair_wait(device_config_t *cfg);

#endif /* ESPNOW_PAIR_H */
```

- [ ] **Step 3: Write `espnow_pair.c`**

```c
/**
 * @file espnow_pair.c
 * @brief ESP-NOW pairing state machine for locally-paired receivers.
 */

#include "pair/espnow_pair.h"

#include <string.h>
#include "config/device_config.h"
#include "esp_check.h"
#include "esp_event.h"
#include "esp_log.h"
#include "esp_mac.h"
#include "esp_now.h"
#include "esp_wifi.h"
#include "freertos/FreeRTOS.h"
#include "freertos/event_groups.h"
#include "mbedtls/md.h"
#include "sdkconfig.h"

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

#define PSK_LEN             32
#define NONCE_LEN           16
#define HMAC_LEN            32

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

static void recv_cb(const esp_now_recv_info_t *info, const uint8_t *data, int len)
{
    if (data == NULL || len < 1) {
        return;
    }

    switch (data[0]) {
    case PAIR_MSG_CHALLENGE:
        if ((size_t)len >= sizeof(pair_challenge_t)) {
            const pair_challenge_t *ch = (const pair_challenge_t *)data;
            memcpy(s_nonce, ch->nonce, NONCE_LEN);
            memcpy(s_hub_mac, info->src_addr, 6);
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
    ESP_RETURN_ON_ERROR(esp_netif_init(), TAG, "netif init");
    ESP_RETURN_ON_ERROR(esp_event_loop_create_default(), TAG, "event loop");

    esp_netif_create_default_wifi_sta();
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
    esp_netif_deinit();
    esp_event_loop_delete_default();
}

static esp_err_t add_broadcast_peer(void)
{
    esp_now_peer_info_t peer = { 0 };
    memset(peer.peer_addr, 0xFF, 6);
    peer.channel = 0;
    peer.ifidx = WIFI_IF_STA;
    peer.encrypt = false;
    return esp_now_add_peer(&peer);
}

static esp_err_t add_hub_peer(const uint8_t hub_mac[6])
{
    if (esp_now_is_peer_exist(hub_mac)) {
        return ESP_OK;
    }
    esp_now_peer_info_t peer = { 0 };
    memcpy(peer.peer_addr, hub_mac, 6);
    peer.channel = 0;
    peer.ifidx = WIFI_IF_STA;
    peer.encrypt = false;
    return esp_now_add_peer(&peer);
}

static uint8_t compiled_band(void)
{
#if CONFIG_RECEIVER_RADIO_2_4G
    return 1;
#else
    return 0;
#endif
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
    esp_wifi_get_mac(WIFI_IF_STA, my_mac);

    pair_request_t req = {
        .msg_type = PAIR_MSG_REQUEST,
        .rx_band = compiled_band(),
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

            ESP_LOGI(TAG, "Challenge received on ch=%u from %02x:%02x:%02x:%02x:%02x:%02x",
                     ch, s_hub_mac[0], s_hub_mac[1], s_hub_mac[2],
                     s_hub_mac[3], s_hub_mac[4], s_hub_mac[5]);

            ESP_ERROR_CHECK(add_hub_peer(s_hub_mac));

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

            ESP_LOGI(TAG, "Offer received: slot=%u band=%u bits=%u",
                     s_offer.slot, s_offer.rf_band, s_offer.rf_bits);

            esp_err_t save_err = device_config_save_pairing(
                cfg, s_offer.slot, s_hub_mac,
                s_offer.rf_code, s_offer.rf_code_len,
                s_offer.rf_bits, s_offer.rf_band, 1);

            if (save_err != ESP_OK) {
                ESP_LOGE(TAG, "Failed to save pairing: %s", esp_err_to_name(save_err));
                continue;
            }

            pair_ack_t ack = { .msg_type = PAIR_MSG_ACK, .slot = s_offer.slot };
            esp_now_send(s_hub_mac, (const uint8_t *)&ack, sizeof(ack));

            bits = xEventGroupWaitBits(
                s_pair_events, PAIR_BIT_CONFIRM,
                pdTRUE, pdFALSE, pdMS_TO_TICKS(10000));

            if ((bits & PAIR_BIT_CONFIRM) == 0) {
                ESP_LOGW(TAG, "No PAIR_CONFIRM within 10s, but pairing saved");
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
- Rewrite: `receiver-esp32/main/main.c`

- [ ] **Step 1: Write new `main.c`**

```c
/**
 * @file main.c
 * @brief Receiver firmware boot: pair if needed, then RF listen + vibrate.
 */

#include "config/device_config.h"
#include "esp_check.h"
#include "esp_log.h"
#include "pair/espnow_pair.h"
#include "trigger/rf_supervisor.h"
#include "trigger/rf_trigger.h"
#include "vibrator/vibrator.h"
#include "sdkconfig.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#define TAG "APP"

static device_config_t g_cfg;

void app_main(void)
{
    ESP_ERROR_CHECK(nvs_init_or_recover());
    ESP_ERROR_CHECK(device_config_load(&g_cfg));

    if (!g_cfg.paired) {
        ESP_LOGI(TAG, "Not paired, entering ESP-NOW pair mode");

        VibratorHandler pair_vib = { 0 };
        if (vibrator_init((gpio_num_t)CONFIG_RECEIVER_VIBRATOR_GPIO, &pair_vib) == ESP_OK) {
            vibrator_pulse(&pair_vib);
            vTaskDelay(pdMS_TO_TICKS(500));
            vibrator_deinit(&pair_vib);
        }

        ESP_ERROR_CHECK(espnow_pair_wait(&g_cfg));
        ESP_LOGI(TAG, "Paired successfully, continuing to RF listen");
    }

    ESP_ERROR_CHECK(rf_trigger_init((gpio_num_t)CONFIG_RECEIVER_VIBRATOR_GPIO));
    rf_trigger_stop_output();

    if (device_config_has_rf_code(&g_cfg)) {
        rf_trigger_set(g_cfg.rf_code, g_cfg.rf_code_len,
                       g_cfg.rf_code_bits, g_cfg.rf_code_ver);
    }

    ESP_ERROR_CHECK(rf_sup_start(&g_cfg));
    ESP_LOGI(TAG, "RF listener started, waiting for triggers");

    for (;;) {
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

---

### Task 6: Build verification

- [ ] **Step 1: Run `idf.py build` for 433 MHz variant**

```bash
idf.py set-target esp32c3
idf.py menuconfig  # set RECEIVER_RADIO_433M, set RECEIVER_PAIR_PSK
idf.py build
```

Expected: Build succeeds with no errors. Warnings about unused variables in retained modules are acceptable.

- [ ] **Step 2: Run `idf.py build` for nRF24 variant**

```bash
idf.py menuconfig  # switch to RECEIVER_RADIO_2_4G
idf.py build
```

Expected: Build succeeds.

- [ ] **Step 3: Verify removed files don't interfere**

```bash
find main/ -name "*.c" -o -name "*.h" | sort
```

Expected: Only `main.c`, `config/`, `pair/`, `trigger/`, `rf/`, `nrf24/`, `vibrator/` directories remain.

