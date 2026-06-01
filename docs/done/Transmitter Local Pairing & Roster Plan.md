# Transmitter Hub — Local Pairing, Roster Sync & Strategic Logging

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend the ESP32-C3 transmitter hub with: (1) ESP-NOW pairing subsystem to locally pair receivers, (2) NVS-backed receiver roster with MQTT sync, (3) slot-based dispatch for hub-paired receivers, (4) OLED pairing UI (Screen 5 + overlays), (5) status bar spacing fix, and (6) strategic logging across all 11 modules (merged from `docs/planned/Transmitter Strategic Logging Plan.md`).

**Architecture:** The hub gains an ESP-NOW pairing host that coexists with the active WiFi STA/MQTT session. Paired receivers are stored in NVS with a roster sync protocol to the backend (reliable push with signed ACK). Dispatch for hub-paired receivers resolves RF config locally from NVS; passive receivers continue using the existing `cmd/transmit` full-payload path. Both payload shapes arrive on the same `cmd/transmit` topic — the hub distinguishes by JSON field presence (`slot` vs `rf_code_hex`).

**Tech Stack:** ESP-IDF v6.0, ESP-NOW, LVGL 8.3, NVS, MQTT (Eclipse Paho), ECDSA P-256 (mbedtls)

**Merged plan:** Strategic logging across all 11 modules is incorporated as Task 12 below.

**Self-contained note:** This plan is designed for implementation inside a Dev Container with the ESP-IDF v6.0 SDK. It does not depend on the backend, frontend, or receiver code.

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

typedef void (*esp_now_recv_cb_t)(const esp_now_recv_info_t *esp_now_info,
                                  const uint8_t *data, int data_len);

typedef void (*esp_now_send_cb_t)(const esp_now_send_info_t *tx_info,
                                  esp_now_send_status_t status);

typedef struct esp_now_recv_info {
    uint8_t *src_addr;
    uint8_t *des_addr;
    wifi_pkt_rx_ctrl_t *rx_ctrl;
} esp_now_recv_info_t;

typedef struct esp_now_peer_info {
    uint8_t peer_addr[ESP_NOW_ETH_ALEN];
    uint8_t lmk[ESP_NOW_KEY_LEN];
    uint8_t channel;
    wifi_interface_t ifidx;
    bool encrypt;
    void *priv;
} esp_now_peer_info_t;

#define ESP_NOW_MAX_DATA_LEN     250  /* v1.0 limit (alias for ESP_NOW_MAX_IE_DATA_LEN) */
#define ESP_NOW_MAX_DATA_LEN_V2  1470 /* v2.0 limit */
#define ESP_NOW_ETH_ALEN         6
#define ESP_NOW_KEY_LEN          16
```

---

## Pairing Protocol Messages

Hub is the **responder** (receives `PAIR_REQUEST`, sends `PAIR_CHALLENGE`, verifies `PAIR_RESPONSE`, sends `PAIR_OFFER`, receives `PAIR_ACK`, sends `PAIR_CONFIRM`).

```c
#define PAIR_MSG_REQUEST    0x01
#define PAIR_MSG_CHALLENGE  0x02
#define PAIR_MSG_RESPONSE   0x03
#define PAIR_MSG_OFFER      0x04
#define PAIR_MSG_ACK        0x05
#define PAIR_MSG_CONFIRM    0x06
```

---

## Existing Module Layout (for reference)

```
transmitter/main/
├── main.c
├── config/device_config.c|.h
├── controls/controls.c|.h
├── diagnostics/dispatch_counters.c|.h
├── dispatch/dispatch.c|.h, radio_supervisor.c|.h
├── display/display.c|.h, display_screens.c|.h, display_font.c
├── events/transmitter_events.h
├── network/mqtt.c|.h, wifi.c|.h, heartbeat.c|.h
├── nrf24/nrf24_transmitter.c|.h, nrf24_regs.h
├── provision/http_server.c|.h
├── rf/rf_common.h, rf_data.c, rf_timer.c, rf_transmitter.c
├── security/device_identity.c|.h
├── serial/serial_protocol.c|.h
└── utils/time_utils.c|.h
```

---

### Task 1: Add Kconfig for pairing

**Files:**
- Modify: `transmitter/main/Kconfig.projbuild`

- [ ] **Step 1: Add pairing config section**

Add the following at the end of the existing Kconfig (before the final `endmenu` if the file has one, otherwise as new entries):

```kconfig
config TRANSMITTER_PAIRING_ENABLED
    bool "Enable local pairing subsystem"
    default y
    help
        Enable the ESP-NOW pairing host, NVS receiver roster, roster
        MQTT sync, and slot-based dispatch. Can be used with or without
        the OLED display. When OLED is also enabled, adds Screen 5 and
        pair mode overlays.

config TRANSMITTER_PAIR_PSK
    string "Pairing pre-shared key (64 hex chars = 32 bytes)"
    default ""
    depends on TRANSMITTER_PAIRING_ENABLED
    help
        Shared secret for HMAC-SHA256 verification during ESP-NOW
        pairing with receivers. Must be exactly 64 hexadecimal
        characters (32 bytes). Must match the receivers'
        CONFIG_RECEIVER_PAIR_PSK.

config TRANSMITTER_PAIR_TIMEOUT_S
    int "Pair mode timeout (seconds)"
    default 60
    range 10 300
    depends on TRANSMITTER_PAIRING_ENABLED

config TRANSMITTER_PAIR_MAX_RECEIVERS
    int "Maximum paired receivers"
    default 32
    range 1 64
    depends on TRANSMITTER_PAIRING_ENABLED

config TRANSMITTER_ROSTER_RETRY_S
    int "Roster sync retry interval (seconds)"
    default 45
    range 10 120
    depends on TRANSMITTER_PAIRING_ENABLED
```

---

### Task 2: Roster NVS module

**Files:**
- Create: `transmitter/main/pair/roster.h`
- Create: `transmitter/main/pair/roster.c`

- [ ] **Step 1: Create `pair/` directory**

```bash
mkdir -p main/pair
```

- [ ] **Step 2: Write `roster.h`**

```c
/**
 * @file roster.h
 * @brief NVS-backed receiver roster for locally-paired devices.
 */

#ifndef TRANSMITTER_ROSTER_H
#define TRANSMITTER_ROSTER_H

#include <stdbool.h>
#include <stddef.h>
#include <stdint.h>
#include "esp_err.h"

#define ROSTER_NAMESPACE   "roster"
#define ROSTER_MAX_NAME_LEN 96
#define ROSTER_MAX_CODE_LEN 16

typedef struct {
    uint8_t  slot;
    uint8_t  mac[6];
    uint8_t  band;          // 0=433M, 1=2.4G
    uint8_t  rf_code[ROSTER_MAX_CODE_LEN];
    uint8_t  rf_code_len;
    uint8_t  rf_bits;
    char     name[ROSTER_MAX_NAME_LEN + 1];
    int64_t  paired_at_ms;
    bool     occupied;
} roster_entry_t;

typedef struct {
    roster_entry_t entries[CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS];
    uint8_t  count;
    uint32_t seq;
    bool     pending;
} roster_t;

esp_err_t roster_load(roster_t *r);
esp_err_t roster_add(roster_t *r, const roster_entry_t *entry);
esp_err_t roster_remove(roster_t *r, uint8_t slot);
esp_err_t roster_set_name(roster_t *r, uint8_t slot, const char *name);
const roster_entry_t *roster_find_slot(const roster_t *r, uint8_t slot);
uint8_t roster_next_free_slot(const roster_t *r);
esp_err_t roster_mark_synced(roster_t *r);

#endif /* TRANSMITTER_ROSTER_H */
```

- [ ] **Step 3: Write `roster.c`**

```c
/**
 * @file roster.c
 * @brief NVS-backed receiver roster.
 */

#include "pair/roster.h"

#include <stdio.h>
#include <string.h>
#include "esp_check.h"
#include "esp_log.h"
#include "nvs.h"
#include "utils/time_utils.h"

#define TAG "roster"

static esp_err_t save_entry(nvs_handle_t h, const roster_entry_t *e)
{
    char key[16];

    snprintf(key, sizeof(key), "rx_%u_mac", e->slot);
    ESP_RETURN_ON_ERROR(nvs_set_blob(h, key, e->mac, 6), TAG, "set %s", key);

    snprintf(key, sizeof(key), "rx_%u_band", e->slot);
    ESP_RETURN_ON_ERROR(nvs_set_u8(h, key, e->band), TAG, "set %s", key);

    snprintf(key, sizeof(key), "rx_%u_code", e->slot);
    ESP_RETURN_ON_ERROR(nvs_set_blob(h, key, e->rf_code, e->rf_code_len), TAG, "set %s", key);

    snprintf(key, sizeof(key), "rx_%u_bits", e->slot);
    ESP_RETURN_ON_ERROR(nvs_set_u8(h, key, e->rf_bits), TAG, "set %s", key);

    snprintf(key, sizeof(key), "rx_%u_name", e->slot);
    ESP_RETURN_ON_ERROR(nvs_set_str(h, key, e->name), TAG, "set %s", key);

    snprintf(key, sizeof(key), "rx_%u_at", e->slot);
    ESP_RETURN_ON_ERROR(nvs_set_i64(h, key, e->paired_at_ms), TAG, "set %s", key);

    return ESP_OK;
}

static void erase_entry_keys(nvs_handle_t h, uint8_t slot)
{
    char key[16];
    const char *suffixes[] = {"mac", "band", "code", "bits", "name", "at"};
    for (size_t i = 0; i < sizeof(suffixes) / sizeof(suffixes[0]); i++) {
        snprintf(key, sizeof(key), "rx_%u_%s", slot, suffixes[i]);
        nvs_erase_key(h, key);
    }
}

esp_err_t roster_load(roster_t *r)
{
    memset(r, 0, sizeof(*r));

    nvs_handle_t h;
    esp_err_t err = nvs_open(ROSTER_NAMESPACE, NVS_READONLY, &h);
    if (err == ESP_ERR_NVS_NOT_FOUND) {
        return ESP_OK;
    }
    ESP_RETURN_ON_ERROR(err, TAG, "open");

    nvs_get_u8(h, "rx_count", &r->count);
    nvs_get_u32(h, "roster_seq", &r->seq);
    uint8_t pending = 0;
    nvs_get_u8(h, "roster_pend", &pending);
    r->pending = pending != 0;

    for (uint8_t s = 1; s <= CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS; s++) {
        roster_entry_t *e = &r->entries[s - 1];
        char key[16];

        snprintf(key, sizeof(key), "rx_%u_mac", s);
        size_t mac_len = 6;
        if (nvs_get_blob(h, key, e->mac, &mac_len) != ESP_OK) {
            continue;
        }

        e->slot = s;
        e->occupied = true;

        snprintf(key, sizeof(key), "rx_%u_band", s);
        nvs_get_u8(h, key, &e->band);

        snprintf(key, sizeof(key), "rx_%u_code", s);
        size_t code_len = sizeof(e->rf_code);
        nvs_get_blob(h, key, e->rf_code, &code_len);
        e->rf_code_len = (uint8_t)code_len;

        snprintf(key, sizeof(key), "rx_%u_bits", s);
        nvs_get_u8(h, key, &e->rf_bits);

        snprintf(key, sizeof(key), "rx_%u_name", s);
        size_t name_len = sizeof(e->name);
        if (nvs_get_str(h, key, e->name, &name_len) != ESP_OK) {
            snprintf(e->name, sizeof(e->name), "RX-%02u", s);
        }

        snprintf(key, sizeof(key), "rx_%u_at", s);
        nvs_get_i64(h, key, &e->paired_at_ms);
    }

    nvs_close(h);
    ESP_LOGI(TAG, "Loaded roster: %u receivers, seq=%lu, pending=%s",
             r->count, (unsigned long)r->seq, r->pending ? "yes" : "no");
    return ESP_OK;
}

esp_err_t roster_add(roster_t *r, const roster_entry_t *entry)
{
    if (r->count >= CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS) {
        return ESP_ERR_NO_MEM;
    }

    nvs_handle_t h;
    ESP_RETURN_ON_ERROR(nvs_open(ROSTER_NAMESPACE, NVS_READWRITE, &h), TAG, "open");

    esp_err_t err = save_entry(h, entry);
    if (err == ESP_OK) {
        r->count++;
        r->seq++;
        r->pending = true;
        nvs_set_u8(h, "rx_count", r->count);
        nvs_set_u32(h, "roster_seq", r->seq);
        nvs_set_u8(h, "roster_pend", 1);
        err = nvs_commit(h);

        r->entries[entry->slot - 1] = *entry;
        r->entries[entry->slot - 1].occupied = true;

        ESP_LOGI(TAG, "Added receiver: slot=%u band=%s name=%s",
                 entry->slot, entry->band == 0 ? "433M" : "2.4G", entry->name);
    }

    nvs_close(h);
    return err;
}

esp_err_t roster_remove(roster_t *r, uint8_t slot)
{
    if (slot < 1 || slot > CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS) {
        return ESP_ERR_INVALID_ARG;
    }

    roster_entry_t *e = &r->entries[slot - 1];
    if (!e->occupied) {
        return ESP_ERR_NOT_FOUND;
    }

    nvs_handle_t h;
    ESP_RETURN_ON_ERROR(nvs_open(ROSTER_NAMESPACE, NVS_READWRITE, &h), TAG, "open");

    erase_entry_keys(h, slot);
    r->count--;
    r->seq++;
    r->pending = true;
    nvs_set_u8(h, "rx_count", r->count);
    nvs_set_u32(h, "roster_seq", r->seq);
    nvs_set_u8(h, "roster_pend", 1);
    nvs_commit(h);
    nvs_close(h);

    memset(e, 0, sizeof(*e));
    ESP_LOGI(TAG, "Removed receiver: slot=%u", slot);
    return ESP_OK;
}

esp_err_t roster_set_name(roster_t *r, uint8_t slot, const char *name)
{
    if (slot < 1 || slot > CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS) {
        return ESP_ERR_INVALID_ARG;
    }

    roster_entry_t *e = &r->entries[slot - 1];
    if (!e->occupied) {
        return ESP_ERR_NOT_FOUND;
    }

    nvs_handle_t h;
    ESP_RETURN_ON_ERROR(nvs_open(ROSTER_NAMESPACE, NVS_READWRITE, &h), TAG, "open");

    char key[16];
    snprintf(key, sizeof(key), "rx_%u_name", slot);
    nvs_set_str(h, key, name);
    nvs_commit(h);
    nvs_close(h);

    strlcpy(e->name, name, sizeof(e->name));
    ESP_LOGI(TAG, "Renamed slot %u to '%s'", slot, name);
    return ESP_OK;
}

const roster_entry_t *roster_find_slot(const roster_t *r, uint8_t slot)
{
    if (slot < 1 || slot > CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS) {
        return NULL;
    }
    const roster_entry_t *e = &r->entries[slot - 1];
    return e->occupied ? e : NULL;
}

uint8_t roster_next_free_slot(const roster_t *r)
{
    for (uint8_t i = 0; i < CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS; i++) {
        if (!r->entries[i].occupied) {
            return i + 1;
        }
    }
    return 0;
}

esp_err_t roster_mark_synced(roster_t *r)
{
    nvs_handle_t h;
    ESP_RETURN_ON_ERROR(nvs_open(ROSTER_NAMESPACE, NVS_READWRITE, &h), TAG, "open");
    r->pending = false;
    nvs_set_u8(h, "roster_pend", 0);
    nvs_commit(h);
    nvs_close(h);
    return ESP_OK;
}
```

---

### Task 3: ESP-NOW pairing host module

**Files:**
- Create: `transmitter/main/pair/espnow_pair_host.h`
- Create: `transmitter/main/pair/espnow_pair_host.c`

**Protocol:** Hub is the responder. Receiver sends `PAIR_REQUEST`, hub sends `PAIR_CHALLENGE`, receiver sends `PAIR_RESPONSE` (HMAC), hub verifies → OLED confirm → hub sends `PAIR_OFFER`, receiver sends `PAIR_ACK`, hub sends `PAIR_CONFIRM` and persists to roster.

- [ ] **Step 1: Write `espnow_pair_host.h`**

```c
#ifndef TRANSMITTER_ESPNOW_PAIR_HOST_H
#define TRANSMITTER_ESPNOW_PAIR_HOST_H

#include <stdbool.h>
#include <stdint.h>
#include "esp_err.h"
#include "pair/roster.h"

typedef enum {
    PAIR_HOST_RESULT_SUCCESS,
    PAIR_HOST_RESULT_TIMEOUT,
    PAIR_HOST_RESULT_CANCELLED,
    PAIR_HOST_RESULT_FULL,
    PAIR_HOST_RESULT_ERROR,
} pair_host_result_t;

typedef void (*pair_host_receiver_found_cb_t)(const uint8_t mac[6], uint8_t band, uint8_t slot);
typedef void (*pair_host_complete_cb_t)(pair_host_result_t result, uint8_t slot);

esp_err_t espnow_pair_host_start(roster_t *roster,
                                 uint32_t timeout_s,
                                 pair_host_receiver_found_cb_t receiver_found_cb,
                                 pair_host_complete_cb_t complete_cb);

esp_err_t espnow_pair_host_confirm(void);

void espnow_pair_host_cancel(void);

bool espnow_pair_host_is_active(void);

#endif
```

- [ ] **Step 2: Write `espnow_pair_host.c`**

```c
#include "pair/espnow_pair_host.h"

#include <string.h>
#include "esp_check.h"
#include "esp_log.h"
#include "esp_now.h"
#include "esp_random.h"
#include "esp_wifi.h"
#include "freertos/FreeRTOS.h"
#include "freertos/event_groups.h"
#include "freertos/task.h"
#include "mbedtls/md.h"
#include "utils/time_utils.h"

#define TAG "pair_host"

#define PSK_LEN   32
#define NONCE_LEN 16
#define HMAC_LEN  32

#define PAIR_MSG_REQUEST   0x01
#define PAIR_MSG_CHALLENGE 0x02
#define PAIR_MSG_RESPONSE  0x03
#define PAIR_MSG_OFFER     0x04
#define PAIR_MSG_ACK       0x05
#define PAIR_MSG_CONFIRM   0x06

#define PAIR_BIT_REQUEST   BIT0
#define PAIR_BIT_RESPONSE  BIT1
#define PAIR_BIT_ACK       BIT2
#define PAIR_BIT_CANCEL    BIT3
#define PAIR_BIT_CONFIRM_BTN BIT4

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

typedef struct {
    roster_t *roster;
    uint32_t timeout_s;
    pair_host_receiver_found_cb_t receiver_found_cb;
    pair_host_complete_cb_t complete_cb;
    EventGroupHandle_t events;
    TaskHandle_t task;
    uint8_t psk[PSK_LEN];
    uint8_t nonce[NONCE_LEN];
    uint8_t rx_mac[6];
    uint8_t rx_band;
    pair_response_t rx_response;
    bool active;
    bool confirmed;
} pair_host_ctx_t;

static pair_host_ctx_t s_ctx;

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

static bool is_forbidden_433m(const uint8_t code[4])
{
    uint8_t nibble0 = code[0] >> 4;
    for (int i = 0; i < 4; i++) {
        if ((code[i] >> 4) != nibble0 || (code[i] & 0x0F) != nibble0) {
            return false;
        }
    }
    return true;
}

static bool is_forbidden_nrf24(const uint8_t addr[5])
{
    bool all_zero = true;
    bool all_one = true;
    uint8_t transitions = 0;
    for (int i = 0; i < 5; i++) {
        if (addr[i] != 0x00) all_zero = false;
        if (addr[i] != 0xFF) all_one = false;
        if (i > 0) {
            transitions += __builtin_popcount(addr[i] ^ addr[i - 1]);
        }
    }
    return all_zero || all_one || transitions < 4;
}

static void generate_rf_code(uint8_t band, uint8_t *code, uint8_t *code_len, uint8_t *bits)
{
    if (band == 0) {
        *code_len = 4;
        *bits = 32;
        do {
            esp_fill_random(code, 4);
        } while (is_forbidden_433m(code));
    } else {
        *code_len = 5;
        *bits = 40;
        do {
            esp_fill_random(code, 5);
        } while (is_forbidden_nrf24(code));
    }
}

static void recv_cb(const esp_now_recv_info_t *info, const uint8_t *data, int len)
{
    if (data == NULL || len < 1 || !s_ctx.active) {
        return;
    }

    switch (data[0]) {
    case PAIR_MSG_REQUEST:
        if ((size_t)len >= sizeof(pair_request_t)) {
            const pair_request_t *req = (const pair_request_t *)data;
            memcpy(s_ctx.rx_mac, req->rx_mac, 6);
            s_ctx.rx_band = req->rx_band;
            xEventGroupSetBits(s_ctx.events, PAIR_BIT_REQUEST);
        }
        break;

    case PAIR_MSG_RESPONSE:
        if ((size_t)len >= sizeof(pair_response_t)) {
            memcpy(&s_ctx.rx_response, data, sizeof(pair_response_t));
            xEventGroupSetBits(s_ctx.events, PAIR_BIT_RESPONSE);
        }
        break;

    case PAIR_MSG_ACK:
        if ((size_t)len >= sizeof(pair_ack_t)) {
            xEventGroupSetBits(s_ctx.events, PAIR_BIT_ACK);
        }
        break;

    default:
        break;
    }
}

static void send_cb(const esp_now_send_info_t *info, esp_now_send_status_t status)
{
    (void)info;
    if (status != ESP_NOW_SEND_SUCCESS) {
        ESP_LOGW(TAG, "ESP-NOW send failed");
    }
}

static esp_err_t add_rx_peer(const uint8_t mac[6])
{
    if (esp_now_is_peer_exist(mac)) {
        return ESP_OK;
    }
    esp_now_peer_info_t peer = { 0 };
    memcpy(peer.peer_addr, mac, 6);
    peer.channel = 0;
    peer.ifidx = WIFI_IF_STA;
    peer.encrypt = false;
    return esp_now_add_peer(&peer);
}

static void pair_host_task(void *arg)
{
    (void)arg;
    pair_host_result_t result = PAIR_HOST_RESULT_TIMEOUT;
    uint8_t paired_slot = 0;

    const TickType_t deadline = xTaskGetTickCount() + pdMS_TO_TICKS(s_ctx.timeout_s * 1000);

    for (;;) {
        TickType_t remaining = deadline - xTaskGetTickCount();
        if ((int32_t)remaining <= 0) {
            break;
        }

        EventBits_t bits = xEventGroupWaitBits(
            s_ctx.events, PAIR_BIT_REQUEST | PAIR_BIT_CANCEL,
            pdTRUE, pdFALSE, remaining);

        if (bits & PAIR_BIT_CANCEL) {
            result = PAIR_HOST_RESULT_CANCELLED;
            break;
        }
        if (!(bits & PAIR_BIT_REQUEST)) {
            continue;
        }

        uint8_t next_slot = roster_next_free_slot(s_ctx.roster);
        if (next_slot == 0) {
            result = PAIR_HOST_RESULT_FULL;
            break;
        }

        ESP_LOGI(TAG, "PAIR_REQUEST from %02x:%02x:%02x:%02x:%02x:%02x band=%u",
                 s_ctx.rx_mac[0], s_ctx.rx_mac[1], s_ctx.rx_mac[2],
                 s_ctx.rx_mac[3], s_ctx.rx_mac[4], s_ctx.rx_mac[5],
                 s_ctx.rx_band);

        add_rx_peer(s_ctx.rx_mac);

        esp_fill_random(s_ctx.nonce, NONCE_LEN);
        pair_challenge_t challenge = { .msg_type = PAIR_MSG_CHALLENGE };
        memcpy(challenge.nonce, s_ctx.nonce, NONCE_LEN);
        esp_now_send(s_ctx.rx_mac, (const uint8_t *)&challenge, sizeof(challenge));

        remaining = deadline - xTaskGetTickCount();
        if ((int32_t)remaining <= 0) break;
        bits = xEventGroupWaitBits(
            s_ctx.events, PAIR_BIT_RESPONSE | PAIR_BIT_CANCEL,
            pdTRUE, pdFALSE, pdMS_TO_TICKS(5000) < remaining ? pdMS_TO_TICKS(5000) : remaining);

        if (bits & PAIR_BIT_CANCEL) { result = PAIR_HOST_RESULT_CANCELLED; break; }
        if (!(bits & PAIR_BIT_RESPONSE)) {
            ESP_LOGW(TAG, "No PAIR_RESPONSE within 5s, resuming");
            esp_now_del_peer(s_ctx.rx_mac);
            continue;
        }

        uint8_t expected_hmac[HMAC_LEN];
        if (compute_hmac(s_ctx.nonce, NONCE_LEN, s_ctx.psk, PSK_LEN, expected_hmac) != ESP_OK) {
            ESP_LOGE(TAG, "HMAC compute failed");
            esp_now_del_peer(s_ctx.rx_mac);
            continue;
        }

        if (memcmp(expected_hmac, s_ctx.rx_response.hmac, HMAC_LEN) != 0) {
            ESP_LOGW(TAG, "HMAC verification failed, dropping");
            esp_now_del_peer(s_ctx.rx_mac);
            continue;
        }

        ESP_LOGI(TAG, "Receiver verified, awaiting admin confirm");

        if (s_ctx.receiver_found_cb != NULL) {
            s_ctx.receiver_found_cb(s_ctx.rx_mac, s_ctx.rx_band, next_slot);
        }

        remaining = deadline - xTaskGetTickCount();
        if ((int32_t)remaining <= 0) break;
        TickType_t confirm_wait = pdMS_TO_TICKS(10000) < remaining ? pdMS_TO_TICKS(10000) : remaining;
        bits = xEventGroupWaitBits(
            s_ctx.events, PAIR_BIT_CONFIRM_BTN | PAIR_BIT_CANCEL,
            pdTRUE, pdFALSE, confirm_wait);

        if (bits & PAIR_BIT_CANCEL) { result = PAIR_HOST_RESULT_CANCELLED; break; }
        if (!(bits & PAIR_BIT_CONFIRM_BTN)) {
            ESP_LOGI(TAG, "Admin did not confirm within 10s, skipping");
            esp_now_del_peer(s_ctx.rx_mac);
            continue;
        }

        pair_offer_t offer = { .msg_type = PAIR_MSG_OFFER, .slot = next_slot, .rf_band = s_ctx.rx_band };
        generate_rf_code(s_ctx.rx_band, offer.rf_code, &offer.rf_code_len, &offer.rf_bits);
        esp_now_send(s_ctx.rx_mac, (const uint8_t *)&offer, sizeof(offer));

        remaining = deadline - xTaskGetTickCount();
        if ((int32_t)remaining <= 0) break;
        bits = xEventGroupWaitBits(
            s_ctx.events, PAIR_BIT_ACK | PAIR_BIT_CANCEL,
            pdTRUE, pdFALSE, pdMS_TO_TICKS(5000) < remaining ? pdMS_TO_TICKS(5000) : remaining);

        if (bits & PAIR_BIT_CANCEL) { result = PAIR_HOST_RESULT_CANCELLED; break; }
        if (!(bits & PAIR_BIT_ACK)) {
            ESP_LOGW(TAG, "No PAIR_ACK within 5s");
            esp_now_del_peer(s_ctx.rx_mac);
            continue;
        }

        pair_confirm_t confirm = { .msg_type = PAIR_MSG_CONFIRM };
        esp_now_send(s_ctx.rx_mac, (const uint8_t *)&confirm, sizeof(confirm));

        roster_entry_t entry = {
            .slot = next_slot,
            .band = s_ctx.rx_band,
            .rf_code_len = offer.rf_code_len,
            .rf_bits = offer.rf_bits,
            .paired_at_ms = time_utils_now_epoch_ms(),
            .occupied = true,
        };
        memcpy(entry.mac, s_ctx.rx_mac, 6);
        memcpy(entry.rf_code, offer.rf_code, offer.rf_code_len);
        snprintf(entry.name, sizeof(entry.name), "RX-%02u", next_slot);

        if (roster_add(s_ctx.roster, &entry) == ESP_OK) {
            ESP_LOGI(TAG, "Paired: slot=%u band=%s", next_slot, s_ctx.rx_band == 0 ? "433M" : "2.4G");
            result = PAIR_HOST_RESULT_SUCCESS;
            paired_slot = next_slot;
        } else {
            ESP_LOGE(TAG, "Failed to persist roster entry");
            result = PAIR_HOST_RESULT_ERROR;
        }

        esp_now_del_peer(s_ctx.rx_mac);
        break;
    }

    esp_now_unregister_recv_cb();
    esp_now_unregister_send_cb();
    esp_now_deinit();
    s_ctx.active = false;

    ESP_LOGI(TAG, "Pair mode ended: result=%d", result);
    if (s_ctx.complete_cb != NULL) {
        s_ctx.complete_cb(result, paired_slot);
    }

    vEventGroupDelete(s_ctx.events);
    s_ctx.events = NULL;
    s_ctx.task = NULL;
    vTaskDelete(NULL);
}

esp_err_t espnow_pair_host_start(roster_t *roster,
                                 uint32_t timeout_s,
                                 pair_host_receiver_found_cb_t receiver_found_cb,
                                 pair_host_complete_cb_t complete_cb)
{
    if (s_ctx.active) {
        return ESP_ERR_INVALID_STATE;
    }

    if (!hex_to_bytes(CONFIG_TRANSMITTER_PAIR_PSK, s_ctx.psk, PSK_LEN)) {
        ESP_LOGE(TAG, "Invalid PSK in Kconfig (need 64 hex chars)");
        return ESP_ERR_INVALID_ARG;
    }

    s_ctx.roster = roster;
    s_ctx.timeout_s = timeout_s;
    s_ctx.receiver_found_cb = receiver_found_cb;
    s_ctx.complete_cb = complete_cb;
    s_ctx.confirmed = false;

    s_ctx.events = xEventGroupCreate();
    ESP_RETURN_ON_FALSE(s_ctx.events != NULL, ESP_ERR_NO_MEM, TAG, "events");

    ESP_RETURN_ON_ERROR(esp_now_init(), TAG, "esp_now_init");
    ESP_RETURN_ON_ERROR(esp_now_register_recv_cb(recv_cb), TAG, "register recv");
    ESP_RETURN_ON_ERROR(esp_now_register_send_cb(send_cb), TAG, "register send");

    s_ctx.active = true;
    BaseType_t ok = xTaskCreate(pair_host_task, "pair_host", 4096, NULL, 5, &s_ctx.task);
    if (ok != pdPASS) {
        s_ctx.active = false;
        esp_now_deinit();
        vEventGroupDelete(s_ctx.events);
        s_ctx.events = NULL;
        return ESP_ERR_NO_MEM;
    }

    ESP_LOGI(TAG, "Pair mode started, timeout=%lus", (unsigned long)timeout_s);
    return ESP_OK;
}

esp_err_t espnow_pair_host_confirm(void)
{
    if (!s_ctx.active || s_ctx.events == NULL) {
        return ESP_ERR_INVALID_STATE;
    }
    xEventGroupSetBits(s_ctx.events, PAIR_BIT_CONFIRM_BTN);
    return ESP_OK;
}

void espnow_pair_host_cancel(void)
{
    if (s_ctx.active && s_ctx.events != NULL) {
        xEventGroupSetBits(s_ctx.events, PAIR_BIT_CANCEL);
    }
}

bool espnow_pair_host_is_active(void)
{
    return s_ctx.active;
}
```

---

### Task 4: Roster MQTT sync module

**Files:**
- Create: `transmitter/main/pair/roster_sync.h`
- Create: `transmitter/main/pair/roster_sync.c`

- [ ] **Step 1: Write `roster_sync.h`**

```c
/**
 * @file roster_sync.h
 * @brief MQTT roster sync with backend (reliable push + signed ACK).
 */

#ifndef TRANSMITTER_ROSTER_SYNC_H
#define TRANSMITTER_ROSTER_SYNC_H

#include "esp_err.h"
#include "pair/roster.h"

esp_err_t roster_sync_init(roster_t *roster);
esp_err_t roster_sync_publish(void);
void roster_sync_handle_ack(const char *json, size_t len);
void roster_sync_handle_label(const char *json, size_t len);
void roster_sync_handle_unpair(const char *json, size_t len);
void roster_sync_deinit(void);

#endif /* TRANSMITTER_ROSTER_SYNC_H */
```

- [ ] **Step 2: Write `roster_sync.c`**

Publishes roster updates to `{prefix}/transmitter/hub/{publicId}/roster/update` as JSON. Handles signed ACK verification via `device_identity_verify_text()` with canonical `roster-ack-v1|{hubPublicId}|{seq}|{issuedAt}`. Handles `cmd/label` to update receiver names. Handles `cmd/unpair` to remove receivers. Uses an `esp_timer` for retry.

The JSON payload format:

```json
{
  "schema_version": 1,
  "seq": 3,
  "receivers": [
    {"slot": 1, "band": "433M", "label": "RX-01", "paired_at": "2026-05-30T10:00:00Z"}
  ]
}
```

The implementor should use `cJSON` (already a dependency in the transmitter project) for JSON construction and parsing, and `device_identity_verify_text()` for ACK signature verification.

---

### Task 5: Extend `dispatch.c` for slot-based payloads

**Files:**
- Modify: `transmitter/main/dispatch/dispatch.c`

- [ ] **Step 1: Add roster include and slot dispatch handler**

Guard with `#if CONFIG_TRANSMITTER_PAIRING_ENABLED`. Add `#include "pair/roster.h"` to the includes.

Add a new function `dispatch_handle_slot(cJSON *root)` that:

1. Parses `slot`, `action`, `dispatch_id`, `issued_at`, `signature_b64` from the cJSON root
2. Builds canonical: `dispatch-v1|{hubPublicId}|{dispatchId}|{slot}|{action}|{issuedAt}`
3. Verifies signature via `device_identity_verify_text()`
4. Looks up `slot` in the roster via `roster_find_slot()` → gets `rf_code`, `rf_code_len`, `rf_bits`, `band`
5. Converts band: `(radio_band_t)entry->band` (enum values match: 0=433M, 1=2.4G)
6. Calls `radio_tx_send((radio_band_t)entry->band, entry->rf_code, entry->rf_code_len, entry->rf_bits, true)`
   Note: `proto_any` is hardcoded `true` for slot dispatches (hub-paired receivers always use randomized protocols for 433M)
7. Records in dispatch ring and publishes ACK

- [ ] **Step 2: Modify `dispatch_handle_transmit_json` to detect payload format**

The existing function calls `parse_transmit_json()` which internally calls `cJSON_ParseWithLength`. To add slot detection without double-parsing, pre-parse at the top and pass the `cJSON *root` to both paths:

```c
void dispatch_handle_transmit_json(const char *json, size_t len)
{
    cJSON *root = cJSON_ParseWithLength(json, len);
    if (root == NULL) {
        transmit_cmd_t best_effort = { 0 };
        reject_transmit(&best_effort, "bad_shape");
        post_dispatch_event(&best_effort, "rejected", "bad_shape", 0);
        return;
    }

    if (cJSON_HasObjectItem(root, "slot")) {
        dispatch_handle_slot(root);
        cJSON_Delete(root);
        return;
    }

    // full-payload path — extract fields from already-parsed root
    transmit_cmd_t cmd;
    if (!parse_transmit_from_root(root, &cmd)) {
        transmit_cmd_t best_effort = { 0 };
        (void)parse_transmit_from_root(root, &best_effort);
        cJSON_Delete(root);
        reject_transmit(&best_effort, "bad_shape");
        post_dispatch_event(&best_effort, "rejected", "bad_shape", 0);
        return;
    }
    cJSON_Delete(root);

    // ...rest of existing validation/dispatch logic (signature, width, ring, tx)
}
```

Note: This requires refactoring `parse_transmit_json` into `parse_transmit_from_root(cJSON *root, transmit_cmd_t *cmd)` that accepts an already-parsed root, eliminating the double-parse. The existing `parse_transmit_json` wrapper can remain for backward compat but should call `parse_transmit_from_root` internally.

---

### Task 6: MQTT topic subscriptions for roster/label/unpair

**Files:**
- Modify: `transmitter/main/network/mqtt.c`

- [ ] **Step 1: Subscribe to new topics in `mqtt_subscribe_current_phase` (activated branch)**

After the existing `cmd/transmit` and `cmd/deact` subscriptions, add:

```c
(void)snprintf(topic, sizeof(topic), TOPIC_PREFIX "transmitter/hub/%s/roster/ack", cfg->public_id);
(void)esp_mqtt_client_subscribe(s_client, topic, 1);

(void)snprintf(topic, sizeof(topic), TOPIC_PREFIX "transmitter/hub/%s/cmd/label", cfg->public_id);
(void)esp_mqtt_client_subscribe(s_client, topic, 1);

(void)snprintf(topic, sizeof(topic), TOPIC_PREFIX "transmitter/hub/%s/cmd/unpair", cfg->public_id);
(void)esp_mqtt_client_subscribe(s_client, topic, 1);
```

- [ ] **Step 2: Route incoming messages to roster_sync handlers**

In `route_message()` (the centralized topic router in mqtt.c), add the new topic matches using the existing exact-match pattern with `snprintf`+`strcmp`. Place them **before** the existing `cmd/transmit` check:

```c
// In route_message(), after the bootstrap prefix check and the !has_public_id guard:

(void)snprintf(expected_topic, sizeof(expected_topic), TOPIC_PREFIX "transmitter/hub/%s/roster/ack", cfg->public_id);
if (strcmp(topic, expected_topic) == 0) {
    roster_sync_handle_ack(payload, payload_len);
    return;
}

(void)snprintf(expected_topic, sizeof(expected_topic), TOPIC_PREFIX "transmitter/hub/%s/cmd/label", cfg->public_id);
if (strcmp(topic, expected_topic) == 0) {
    roster_sync_handle_label(payload, payload_len);
    return;
}

(void)snprintf(expected_topic, sizeof(expected_topic), TOPIC_PREFIX "transmitter/hub/%s/cmd/unpair", cfg->public_id);
if (strcmp(topic, expected_topic) == 0) {
    roster_sync_handle_unpair(payload, payload_len);
    return;
}

// ...existing cmd/transmit and cmd/deact checks follow
```

Note: The existing codebase uses exact `strcmp` matching against formatted topic strings in `route_message()` — NOT `strstr`. The new routes must follow this same pattern for security (prevents partial topic injection).

- [ ] **Step 3: On MQTT connect, trigger pending roster sync**

After the activated-branch subscriptions complete in `mqtt_subscribe_current_phase()`, check if a roster update is pending and publish it:

```c
roster_sync_publish();  // no-op internally if !roster.pending
```

Note: `roster_sync_publish()` (declared in `roster_sync.h`) already contains a `pending` guard via the roster struct — no separate `roster_sync_pending()` function is needed.

---

### Task 7: OLED Screen 5 — Paired Receivers

**Files:**
- Modify: `transmitter/main/display/display_screens.h`
- Modify: `transmitter/main/display/display_screens.c`
- Modify: `transmitter/main/display/display.c`

- [ ] **Step 1: Add Screen 5 struct to `display_screens.h`**

```c
typedef struct {
    lv_obj_t *lbl_header;
    lv_obj_t *lbl_row[3];
} screen_receivers_t;
```

- [ ] **Step 2: Add `screens_create_receivers()` to `display_screens.c`**

Follow the same pattern as `screens_create_dashboard()` — flex column container at `CONTENT_Y_OFFSET`, `proggy_clean_12` font, 3 label rows.

- [ ] **Step 3: Add Screen 5 to the screen cycle**

Update `#define SCREEN_COUNT 4` to `#define SCREEN_COUNT 5` in `display_screens.h` (the define lives there, not in `display.c`). Then in `display_init()`, add the screen creation call and insert it at index 4 in the `s_screens[]` array.

- [ ] **Step 4: Add update function for Screen 5**

`update_screen_receivers()` reads from the roster and formats each row as `{slot}:{name} {band} {dot}`.

---

### Task 8: OLED pair mode and confirm overlays

**Files:**
- Modify: `transmitter/main/display/display_screens.h`
- Modify: `transmitter/main/display/display_screens.c`
- Modify: `transmitter/main/display/display.c`
- Modify: `transmitter/main/controls/controls.c`

- [ ] **Step 1: Add pair overlay structs**

```c
typedef struct {
    lv_obj_t *overlay;
    lv_obj_t *lbl_status;
    lv_obj_t *lbl_countdown;
    lv_obj_t *lbl_instruction;
} pair_mode_overlay_t;

typedef struct {
    lv_obj_t *overlay;
    lv_obj_t *lbl_found;
    lv_obj_t *lbl_mac;
    lv_obj_t *lbl_band_slot;
    lv_obj_t *lbl_instruction;
} pair_confirm_overlay_t;
```

- [ ] **Step 2: Implement overlay create/show/hide/update functions**

Follow the same pattern as `dispatch_overlay_t`.

- [ ] **Step 3: Add double-click handler on Screen 5 for pair mode entry**

**Prerequisite:** Add the following declarations to `display.h` (they don't exist yet):

```c
uint8_t display_current_screen(void);
void display_enter_pair_mode(void);

#define SCREEN_RECEIVERS 4  /* 0-indexed: dashboard=0, dispatch=1, network=2, device=3, receivers=4 */
```

Also add no-op stubs to the `#else` block in `controls.c` for the OLED-disabled build:

```c
static inline uint8_t display_current_screen(void) { return 0; }
static inline void display_enter_pair_mode(void) {}
#define SCREEN_RECEIVERS 4
```

Then modify `on_double_click()`:

```c
static void on_double_click(void *arg, void *usr_data)
{
    (void)arg;
    (void)usr_data;

    bool was_inactive = display_is_asleep() || display_is_dimmed();
    display_wake();

    if (was_inactive) return;

    display_dismiss_overlay_if_visible();

    if (display_current_screen() == SCREEN_RECEIVERS) {
        display_enter_pair_mode();
        return;
    }

    display_jump_to_dashboard();
}
```

---

### Task 9: Status bar spacing fix

**Files:**
- Modify: `transmitter/main/display/display_screens.c`

- [ ] **Step 1: Fix state label / uptime overlap**

In `screens_create_status_bar()`, the `lbl_state` is at x=76 and `lbl_uptime` at x=104. With "ACTIVE" (6 chars × ~6px = ~36px), the label extends to ~112px, overlapping the uptime at 104.

Fix by shortening the state labels in the update function:

| Full state | Short label |
|---|---|
| `ACTIVE` | `ACT` |
| `SUSPENDED` | `SUS` |
| `DECOMMISSIONED` | `DEC` |
| `BOOT` | `BOOT` |
| `PROV` | `PROV` |
| `WAIT` | `WAIT` |

This keeps the state label within 4 chars × ~6px = ~24px, ending at ~100px — safely before the uptime at 104px.

Update the state-setting code in `display.c` wherever `lv_label_set_text(bar->lbl_state, ...)` is called.

---

### Task 10: Update CMakeLists.txt

**Files:**
- Modify: `transmitter/main/CMakeLists.txt`

- [ ] **Step 1: Add new source files conditionally**

Add after the existing `if(CONFIG_TRANSMITTER_OLED_ENABLED)` block:

```cmake
if(CONFIG_TRANSMITTER_PAIRING_ENABLED)
    list(APPEND SRCS
        "pair/roster.c"
        "pair/espnow_pair_host.c"
        "pair/roster_sync.c"
    )
endif()
```

Also add `esp_wifi` to the `PRIV_REQUIRES` list if not already present (needed for `esp_now.h` — it's provided by the `esp_wifi` component, which is already listed).

---

### Task 11: Wire up in `main.c`

**Files:**
- Modify: `transmitter/main/main.c`

- [ ] **Step 1: Load roster after config**

Add includes at the top of `main.c` and a static roster instance:

```c
#include "pair/roster.h"
#include "pair/roster_sync.h"

static roster_t g_roster;
```

In `app_main()`, after `device_config_load()` and `device_config_consume_recovery_mode_request()`, but before `dispatch_counters_init()`:

```c
ESP_ERROR_CHECK(roster_load(&g_roster));
```

This placement ensures the roster is available in memory before any display or control init that may reference it (Screens 5 reads from roster).

- [ ] **Step 2: Init roster sync after activation loop exits**

After the activation `for(;;)` loop exits (i.e., after line `(void)esp_event_post(TRANSMITTER_EVENTS, TX_EVENT_ACTIVATED, ...)`) and before the idle loop:

```c
ESP_ERROR_CHECK(roster_sync_init(&g_roster));
```

This must happen **after** activation is confirmed because `roster_sync_publish()` requires `cfg->has_public_id` to build the MQTT topic. Placing it after `mqtt_start()` but before activation (as originally stated) would fail because `public_id` isn't available until bootstrap completes.

---

### Task 12: Strategic logging across all modules

Adds field-debugging log calls to all 11 modules. No logging from ISR context. No sensitive data (passwords, tokens, private keys). Don't duplicate what `ESP_RETURN_ON_ERROR`/`ESP_RETURN_ON_FALSE` already log.

---

- [ ] **Step 1: `network/wifi.c` — add 6 log calls**

The file already has `#include "esp_log.h"` and `TAG = "wifi"`.

In `wifi_event_handler`, after `++s_sta_retry_count;` (line 55):
```c
                ++s_sta_retry_count;
                ESP_LOGW(TAG, "STA disconnected, retry %u/%u", s_sta_retry_count, (uint8_t)CONFIG_TRANSMITTER_WIFI_MAX_RETRY);
                (void)esp_wifi_connect();
```

After the `} else if (s_mode == WIFI_RUNTIME_MODE_STA)` block sets `WIFI_BIT_STA_FAILED` (line 58):
```c
                ESP_LOGW(TAG, "STA retries exhausted");
                xEventGroupSetBits(s_wifi_events, WIFI_BIT_STA_FAILED);
```

In the `IP_EVENT_STA_GOT_IP` handler, after `xEventGroupSetBits` (line 64):
```c
        ESP_LOGI(TAG, "STA connected");
        xEventGroupSetBits(s_wifi_events, WIFI_BIT_STA_CONNECTED);
```

In `wifi_start_sta`, after `s_wifi_started = true;` (line 147):
```c
    s_wifi_started = true;
    ESP_LOGI(TAG, "STA starting, SSID=%.32s", (const char *)sta_cfg.sta.ssid);
    return ESP_OK;
```

In `wifi_start_softap`, after `s_wifi_started = true;` (line 204):
```c
    s_wifi_started = true;
    ESP_LOGI(TAG, "SoftAP started, SSID=%s", s_softap_ssid);
    return ESP_OK;
```

In `stop_if_started`, after `s_wifi_started = false;` (line 106), only when it was previously true:
```c
    s_wifi_started = false;
    s_sta_connected = false;
    ESP_LOGI(TAG, "WiFi stopped");
    return ESP_OK;
```

---

- [ ] **Step 2: `network/mqtt.c` — add 11 log calls**

Already has `#include "esp_log.h"` and `TAG = "mqtt"`.

In `mqtt_event_handler` — `MQTT_EVENT_CONNECTED` case (line 411):
```c
    case MQTT_EVENT_CONNECTED:
        ESP_LOGI(TAG, "MQTT connected");
        xEventGroupSetBits(s_mqtt_events, MQTT_BIT_CONNECTED);
```

In `mqtt_event_handler` — `MQTT_EVENT_DISCONNECTED` case (line 416):
```c
    case MQTT_EVENT_DISCONNECTED:
        ESP_LOGW(TAG, "MQTT disconnected");
        xEventGroupClearBits(s_mqtt_events, MQTT_BIT_CONNECTED);
```

In `mqtt_subscribe_current_phase` — activated branch, after the two subscribe calls (after line 119):
```c
        (void)esp_mqtt_client_subscribe(s_client, topic, 1);
        ESP_LOGI(TAG, "Subscribed to operational topics");
```

In `mqtt_subscribe_current_phase` — bootstrap branch, after the subscribe (line 136–138 area):
```c
        ESP_LOGI(TAG, "Subscribed to bootstrap topics");
```

In `mqtt_publish_register` — after `s_bootstrap.register_published = true;` (line 100):
```c
    s_bootstrap.register_published = true;
    ESP_LOGI(TAG, "Bootstrap register published");
    return ESP_OK;
```

In `handle_bootstrap_pending` — after storing challenge_id (line 172 area):
```c
    (void)esp_mqtt_client_subscribe(s_client, s_bootstrap.challenge_topic, 1);
    ESP_LOGI(TAG, "Bootstrap pending, challenge_id assigned");
```

In `handle_bootstrap_rejected` — at the start (line 182):
```c
static void handle_bootstrap_rejected(void)
{
    ESP_LOGW(TAG, "Bootstrap rejected by backend");
    mqtt_reset_bootstrap_session();
```

In `handle_bootstrap_challenge` — after publishing the response (line 245):
```c
    (void)mqtt_publish(s_bootstrap.challenge_topic, payload, payload_len, 1, 0);
    ESP_LOGI(TAG, "Bootstrap challenge response sent");
```

In `handle_bootstrap_result` — after `mqtt_reset_bootstrap_session` (line 275):
```c
    mqtt_reset_bootstrap_session();
    ESP_LOGI(TAG, "Bootstrap activation confirmed");
    xEventGroupSetBits(s_mqtt_events, MQTT_BIT_BOOTSTRAP_DONE);
```

In `mqtt_start` — after `mqtt_start_client()` returns OK (end of function, line 539):
```c
    ESP_LOGI(TAG, "MQTT client started");
    return mqtt_start_client();
```
(Note: log before `return` if wrapping, or log on success only)

In `mqtt_stop` — after the `s_client != NULL` guard (line 544):
```c
    ESP_LOGI(TAG, "MQTT client stopped");
    (void)heartbeat_stop();
```

---

- [ ] **Step 3: `dispatch/dispatch.c` — add TAG + include + 11 log calls**

Add `#include "esp_log.h"` to the includes and `static const char *TAG = "dispatch";` after the includes.

After `reject_transmit(&best_effort, "bad_shape");` (line 331):
```c
        ESP_LOGW(TAG, "Transmit rejected: bad_shape");
```

After `reject_transmit(&cmd, "width_mismatch");` (line 338):
```c
        ESP_LOGW(TAG, "Transmit rejected: width_mismatch, band=%s bits=%d", cmd.band, cmd.rf_code_bits);
```

After `reject_transmit(&cmd, "signature_failed");` (line 343):
```c
        ESP_LOGW(TAG, "Transmit rejected: signature_failed");
```

In the duplicate path, after `publish_transmit_ack` (line 353):
```c
        ESP_LOGI(TAG, "Transmit duplicate: dispatch_id=%.8s status=%s", cmd.dispatch_id, status);
```

In the device-not-active path, after `reject_transmit` (line 364):
```c
        ESP_LOGW(TAG, "Transmit rejected: %s", reason);
```

In the bad_hex path, after `reject_transmit` (line 372):
```c
        ESP_LOGW(TAG, "Transmit rejected: bad_hex");
```

In the `tx_result == ESP_OK` path, after `publish_transmit_ack` (line 382):
```c
        ESP_LOGI(TAG, "Transmit applied: band=%s bits=%d", cmd.band, cmd.rf_code_bits);
```

In the `tx_result != ESP_OK` paths (lines 385, 389):
```c
        ESP_LOGW(TAG, "Transmit rejected: %s, band=%s",
                 tx_result == ESP_ERR_NOT_SUPPORTED ? "no_2_4g_radio" : "tx_failed", cmd.band);
```
(single log for both failure paths, place after the final `post_dispatch_event` in the else block)

In `dispatch_handle_deact_json` — parse failure (line 399):
```c
    if (!parse_deact_json(json, len, &cmd)) {
        ESP_LOGW(TAG, "Deact rejected: bad_shape");
        return;
    }
```

After `publish_deact_ack(&cmd, "rejected");` for signature fail (line 405):
```c
        ESP_LOGW(TAG, "Deact rejected: signature_failed");
```

At the end of each successful deact action (suspend/resume/decommission), after `publish_deact_ack(&cmd, "ok");`:
```c
        ESP_LOGI(TAG, "Deact processed: action=%s status=ok", cmd.action);
```

---

- [ ] **Step 4: `network/heartbeat.c` — add TAG + include + 2 log calls**

Add `#include "esp_log.h"` and `static const char *TAG = "heartbeat";` after the existing includes.

In `heartbeat_start`, after the `xTaskCreate` succeeds (line 144):
```c
    s_heartbeat_running = true;
    ESP_LOGI(TAG, "Heartbeat started");
    return xTaskCreate(...) == pdPASS ? ESP_OK : ESP_ERR_NO_MEM;
```
(log before the return expression; restructure if needed)

In `heartbeat_stop`, after `xTaskNotifyGive` (line 156):
```c
    xTaskNotifyGive(s_heartbeat_task);
    ESP_LOGI(TAG, "Heartbeat stopped");
    return ESP_OK;
```

---

- [ ] **Step 5: `provision/http_server.c` — add include + 6 log calls**

Add `#include "esp_log.h"` to the includes (TAG `"http_server"` already exists).

In `provision_http_server_start`, after `httpd_start` succeeds (line 235):
```c
    ESP_RETURN_ON_ERROR(httpd_start(&s_server, &config), TAG, "httpd_start");
    ESP_LOGI(TAG, "HTTP provisioning server started");
```

In `provision_http_server_stop`, after `httpd_stop` succeeds (line 283):
```c
    ESP_RETURN_ON_ERROR(httpd_stop(s_server), TAG, "httpd_stop");
    ESP_LOGI(TAG, "HTTP provisioning server stopped");
    s_server = NULL;
```

In `provision_post_handler`, after `device_config_save_provisioning` succeeds (line 182):
```c
    ESP_LOGI(TAG, "Provisioning saved via HTTP");
```

In `provision_post_handler`, at the `!ok || strncmp(...)` rejection (line 171):
```c
    if (!ok || strncmp(request.mqtt_uri, "mqtts://", 8) != 0) {
        ESP_LOGW(TAG, "Provisioning rejected: invalid payload");
        cJSON_Delete(root);
        return httpd_resp_send_err(req, HTTPD_400_BAD_REQUEST, "invalid provisioning payload");
    }
```

In `retry_post_handler`, before `xEventGroupSetBits` (line 199):
```c
    ESP_LOGI(TAG, "WiFi retry requested via HTTP");
    xEventGroupSetBits(s_events, HTTP_SERVER_EVENT_RETRY);
```

In `reset_post_handler`, after `device_config_factory_reset` succeeds (line 206):
```c
    ESP_LOGW(TAG, "Factory reset requested via HTTP");
```

---

- [ ] **Step 6: `config/device_config.c` — add 6 log calls**

Already has `#include "esp_log.h"` and `TAG = "device_config"`.

At the end of `device_config_load`, before `return ESP_OK;` (line 178):
```c
    ESP_LOGI(TAG, "Config loaded: provisioned=%s activated=%s op_state=%s",
             device_config_is_provisioned(&s_cfg) ? "yes" : "no",
             device_config_is_activated(&s_cfg) ? "yes" : "no",
             device_config_op_state_to_string(s_cfg.op_state));
    return ESP_OK;
```

At the end of `device_config_save_provisioning`, before `return ESP_OK;` (line 242):
```c
    ESP_LOGI(TAG, "Provisioning config saved");
    return ESP_OK;
```

At the end of `device_config_commit_activation`, before `return ESP_OK;` (line 320):
```c
    ESP_LOGI(TAG, "Activation committed: public_id=%s", s_cfg.public_id);
    return ESP_OK;
```

At the end of `device_config_commit_lifecycle`, before `return ESP_OK;` (line 359):
```c
    ESP_LOGI(TAG, "Lifecycle committed: op_state=%s", device_config_op_state_to_string(s_cfg.op_state));
    return ESP_OK;
```

In `device_config_factory_reset`, after `device_config_reset_snapshot(&s_cfg);` in the success path (line 442):
```c
    device_config_reset_snapshot(&s_cfg);
    ESP_LOGW(TAG, "Factory reset complete");
```

At the end of `device_config_clear_enroll_token`, before `return ESP_OK;` (line 385):
```c
    ESP_LOGI(TAG, "Enrollment token cleared");
    return ESP_OK;
```

---

- [ ] **Step 7: `security/device_identity.c` — add 4 log calls**

Already has `#include "esp_log.h"` and `TAG = "device_identity"`.

In `generate_device_keypair`, after `s_device_key_ready = true;` (line 307):
```c
    s_device_key_ready = true;
    ESP_LOGI(TAG, "Device keypair generated and saved");
    return ESP_OK;
```

In `load_device_keypair_from_nvs`, after `s_device_key_ready = true;` in the success path (line 390):
```c
    s_device_key_ready = true;
    ESP_LOGI(TAG, "Device keypair loaded from NVS");
    ret = ESP_OK;
```

In `load_backend_public_key`, after `s_backend_key_ready = true;` (line 437):
```c
    s_backend_key_ready = true;
    ESP_LOGI(TAG, "Backend public key loaded");
    return ESP_OK;
```

In `device_identity_ensure`, in the `ESP_ERR_NOT_FOUND` branch (line 448–449):
```c
        if (err == ESP_ERR_NOT_FOUND) {
            ESP_LOGI(TAG, "No existing keypair, generating new identity");
            ESP_RETURN_ON_ERROR(generate_device_keypair(), TAG, "generate keypair");
```

---

- [ ] **Step 8: `nrf24/nrf24_transmitter.c` — fix 4 severity errors + add 9 log calls**

Already has `#include "esp_log.h"` and `TAG = "nrf24"`.

**Severity fixes:** At the `fail_irq`, `fail_device`, `fail_bus`, and `fail` labels (lines 245–265), there are no explicit log calls — the `ESP_GOTO_ON_ERROR` macros handle the error logging. However, if any label has an explicit `ESP_LOGI`, change it to `ESP_LOGW`. (Verify by reading — these labels currently have no standalone logs, the macros do the logging.)

In `nrf24_tx_init`, after the SPI bus config is set up but before `spi_bus_initialize` (line 229 area):
```c
    ESP_LOGI(TAG, "SPI: MOSI=%d MISO=%d SCK=%d CS=%d CE=%d IRQ=%d",
             CONFIG_TRANSMITTER_NRF24_MOSI, CONFIG_TRANSMITTER_NRF24_MISO,
             CONFIG_TRANSMITTER_NRF24_SCK, CONFIG_TRANSMITTER_NRF24_CS,
             CONFIG_TRANSMITTER_NRF24_CE, CONFIG_TRANSMITTER_NRF24_IRQ);
```

In `nrf_probe_chip`, after determining the variant — `NRF_CHIP_PLUS` branch (line 110):
```c
        *variant = NRF_CHIP_PLUS;
        ESP_LOGI(TAG, "Chip detected: nRF24L01+");
        return ESP_OK;
```

And the `NRF_CHIP_LEGACY` branch (line 120):
```c
        *variant = NRF_CHIP_LEGACY;
        ESP_LOGI(TAG, "Chip detected: nRF24L01 (legacy)");
        return ESP_OK;
```

In `nrf24_tx_init`, after `h->ready = true;` (line 242):
```c
    h->ready = true;
    ESP_LOGI(TAG, "nRF24 initialized: %s, ch=%d",
             h->chip == NRF_CHIP_PLUS ? "nRF24L01+" : "nRF24L01",
             CONFIG_TRANSMITTER_NRF24_CHANNEL);
    return ESP_OK;
```

In `nrf24_tx_send`, before CE pulse (line 300):
```c
    ESP_LOGI(TAG, "TX start: addr=%02x:%02x:%02x:%02x:%02x",
             addr[0], addr[1], addr[2], addr[3], addr[4]);
    gpio_set_level(CONFIG_TRANSMITTER_NRF24_CE, 1);
```

After the semaphore timeout (`xSemaphoreTake` returns `pdFALSE`, line 305):
```c
        (void)nrf_cmd(h, NRF_CMD_FLUSH_TX);
        ESP_LOGW(TAG, "TX timeout: no IRQ within 50ms");
        result = ESP_ERR_TIMEOUT;
```

After MAX_RT detection (line 318):
```c
            (void)nrf_cmd(h, NRF_CMD_FLUSH_TX);
            ESP_LOGW(TAG, "TX MAX_RT: auto-retransmit exhausted");
            result = ESP_FAIL;
```

After successful TX (TX_DS set, when `result` remains `ESP_OK`). Add after the `nrf_wreg8` clears status (line 314):
```c
        // TX_DS success case: result stays ESP_OK
```
Actually, add after the else block that handles `(status & NRF_ST_MAX_RT)` — if we reach past that without setting `result = ESP_FAIL`, it means success. Add before `power_down:`:
```c
    if (result == ESP_OK && ret == ESP_OK) {
        ESP_LOGI(TAG, "TX success");
    }

power_down:
```

In `power_down` label, in the else branch where `nrf_rreg8` fails (line 324):
```c
    if (nrf_rreg8(h, NRF_REG_CONFIG, &config) == ESP_OK) {
        config &= (uint8_t)~NRF_CFG_PWR_UP;
        (void)nrf_wreg8(h, NRF_REG_CONFIG, config);
    } else {
        ESP_LOGW(TAG, "Power-down read-back failed");
    }
```

In `nrf24_tx_deinit`, before `h->ready = false;` (line 355):
```c
    ESP_LOGI(TAG, "nRF24 deinitialized");
    h->ready = false;
```

---

- [ ] **Step 9: `dispatch/radio_supervisor.c` — add 4 log calls**

Already has `#include "esp_log.h"` and `TAG = "radio_supervisor"`.

In `radio_supervisor_init`, after `s_initialized = true;` (line 52):
```c
    s_initialized = true;
    ESP_LOGI(TAG, "Radio supervisor initialized: 433M=ready 2.4G=%s",
             s_have_nrf24 ? "ready" : "unavailable");
    return ESP_OK;
```

In `radio_tx_send`, after `xSemaphoreGive(s_radio_mtx);` (line 109) — log the outcome:
```c
    xSemaphoreGive(s_radio_mtx);

    if (err == ESP_OK) {
        ESP_LOGI(TAG, "TX %s: %u bits", band == RADIO_BAND_433M ? "433M" : "2.4G", bits);
    } else {
        ESP_LOGW(TAG, "TX failed: %s err=%s",
                 band == RADIO_BAND_433M ? "433M" : "2.4G", esp_err_to_name(err));
    }

    return err;
```

In `radio_supervisor_deinit`, after `s_initialized = false;` (line 72):
```c
    s_initialized = false;
    ESP_LOGI(TAG, "Radio supervisor deinitialized");
    return ESP_OK;
```

---

- [ ] **Step 10: `serial/serial_protocol.c` — add 5 log calls**

Already has `#include "esp_log.h"` and `TAG = "serial"`.

In `serial_handle_line`, after `const char *type = cJSON_GetStringValue(type_item);` (line 627):
```c
    const char *type = cJSON_GetStringValue(type_item);
    ESP_LOGI(TAG, "Serial cmd: %s", type);
```

In the `else` block (unknown command, line 658):
```c
    } else {
        ESP_LOGW(TAG, "Serial unknown cmd: %s", type);
        serial_send_error(id, "unknown_command");
    }
```

In `handle_provision`, before `esp_restart()` (line 304):
```c
    ESP_LOGI(TAG, "Serial provision received, restarting");
    usb_serial_jtag_wait_tx_done(pdMS_TO_TICKS(200));
    esp_restart();
```

In `handle_factory_reset`, before `esp_restart()` (line 412):
```c
    ESP_LOGW(TAG, "Serial factory reset, restarting");
    usb_serial_jtag_wait_tx_done(pdMS_TO_TICKS(200));
    esp_restart();
```

In `handle_update_mqtt`, before `esp_restart()` (line 396):
```c
    ESP_LOGI(TAG, "Serial MQTT config updated, restarting");
    usb_serial_jtag_wait_tx_done(pdMS_TO_TICKS(200));
    esp_restart();
```

---

- [ ] **Step 11: `controls/controls.c` — add 3 log calls**

Already has `#include "esp_log.h"` and `TAG = "controls"`.

In `on_long_press_start`, after `s_recovery_armed = true;` (line 382):
```c
    s_recovery_armed = true;
    ESP_LOGW(TAG, "Recovery mode armed (hold 5s more)");
    display_show_recovery_prompt();
```

In `recovery_stage2_timer_cb`, after `device_config_request_recovery_mode` succeeds (line 369):
```c
    ESP_LOGW(TAG, "Recovery mode triggered, restarting");
    esp_restart();
```

In `on_long_press_release`, when cancelling (line 395):
```c
    if (s_recovery_armed) {
        s_recovery_armed = false;
        ESP_LOGI(TAG, "Recovery mode cancelled");
        esp_timer_stop(s_recovery_timer);
```


---

### Task 13: Build verification

- [ ] **Step 1: Build with OLED + Pairing enabled**

```bash
idf.py set-target esp32c3
idf.py menuconfig  # enable OLED, enable PAIRING, set PAIR_PSK
idf.py build
```

- [ ] **Step 2: Build with Pairing enabled, OLED disabled**

```bash
idf.py menuconfig  # disable OLED, keep PAIRING enabled
idf.py build
```

- [ ] **Step 3: Build with both OLED and Pairing disabled**

```bash
idf.py menuconfig  # disable OLED, disable PAIRING
idf.py build
```

Expected: All three builds succeed. Core pairing features (roster, sync, slot dispatch) are gated behind `CONFIG_TRANSMITTER_PAIRING_ENABLED`. OLED-specific features (Screen 5, pair overlays, double-click entry) are gated behind both `CONFIG_TRANSMITTER_PAIRING_ENABLED && CONFIG_TRANSMITTER_OLED_ENABLED`.

