# Pairing & Dispatch Firmware Robustness Fixes — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix four firmware-level robustness issues in the ESP-NOW pairing protocol across the transmitter and both receiver codebases.

**Architecture:** All four fixes are independent, surgical patches to existing pairing code. No protocol changes, no new messages, no new files. Each fix modifies a single source file (except Fix 3 which mirrors across two receiver repos).

**Tech Stack:** C / ESP-IDF v6.0 (transmitter + receiver-esp32) / ESP8266 RTOS SDK (receiver-esp8266) / FreeRTOS

**Spec:** `docs/spec/Pairing & Dispatch Robustness Fixes Spec.md`

---

## Existing Codebase Context

### Key files


| File                                       | Role                                              | Fixes |
| ------------------------------------------ | ------------------------------------------------- | ----- |
| `transmitter/main/pair/espnow_pair_host.c` | Hub-side pairing state machine, ESP-NOW callbacks | 1, 2  |
| `receiver-esp32/main/pair/espnow_pair.c`   | ESP32-C3 receiver pairing state machine           | 3     |
| `receiver-esp8266/main/pair/espnow_pair.c` | ESP8266 receiver pairing state machine            | 3, 4  |


### Pairing context struct (transmitter)

```c
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
} pair_host_ctx_t;
```

### ESP-NOW API reference (verified via Context7)

- `esp_now_del_peer(const uint8_t *peer_addr)` → `ESP_OK`, `ESP_ERR_ESPNOW_NOT_FOUND`, `ESP_ERR_ESPNOW_NOT_INIT`, `ESP_ERR_ESPNOW_ARG`. Safe to call even if peer was already deleted (returns `ESP_ERR_ESPNOW_NOT_FOUND`).
- Same signature on both ESP-IDF v6.0 (ESP32-C3) and ESP8266 RTOS SDK.

---

### Task 1: Pair ACK slot validation (transmitter)

**Files:**

- Modify: `transmitter/main/pair/espnow_pair_host.c`
- **Step 1: Add `rx_ack` field to `pair_host_ctx_t`**

At line 87, after the existing `pair_response_t rx_response;` field, add:

```c
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
    pair_ack_t rx_ack;
    bool active;
} pair_host_ctx_t;
```

- **Step 2: Store ACK data in the receive callback**

Replace the `PAIR_MSG_ACK` case (lines 231–242) to copy the full ACK payload before setting the event bit:

```c
    case PAIR_MSG_ACK:
        if ((size_t)len >= sizeof(pair_ack_t)) {
            uint8_t rx_mac[6];
            portENTER_CRITICAL(&s_ctx_mux);
            memcpy(rx_mac, s_ctx.rx_mac, sizeof(rx_mac));
            portEXIT_CRITICAL(&s_ctx_mux);
            if (memcmp(rx_mac, info->src_addr, sizeof(rx_mac)) != 0) {
                return;
            }
            portENTER_CRITICAL(&s_ctx_mux);
            memcpy(&s_ctx.rx_ack, data, sizeof(s_ctx.rx_ack));
            portEXIT_CRITICAL(&s_ctx_mux);
            xEventGroupSetBits(s_ctx.events, PAIR_BIT_ACK);
        }
        break;
```

- **Step 3: Validate ACK slot in the main pairing loop**

After `PAIR_BIT_ACK` fires (line 456–460), add slot validation before the roster commit block. Replace:

```c
        if ((bits & PAIR_BIT_ACK) == 0) {
            ESP_LOGW(TAG, "No PAIR_ACK within 5s");
            (void)esp_now_del_peer(rx_mac);
            continue;
        }
```

With:

```c
        if ((bits & PAIR_BIT_ACK) == 0) {
            ESP_LOGW(TAG, "No PAIR_ACK within 5s");
            (void)esp_now_del_peer(rx_mac);
            continue;
        }

        uint8_t ack_slot;
        portENTER_CRITICAL(&s_ctx_mux);
        ack_slot = s_ctx.rx_ack.slot;
        portEXIT_CRITICAL(&s_ctx_mux);

        if (ack_slot != next_slot) {
            ESP_LOGW(TAG, "PAIR_ACK slot mismatch: expected=%u got=%u",
                     next_slot, ack_slot);
            (void)esp_now_del_peer(rx_mac);
            continue;
        }
```

Note: `next_slot` is the correct variable for both new entries and re-pairs. For new entries it is the freshly allocated slot. For existing entries (`existing_entry != NULL`), `next_slot` is set to `existing_entry->slot` earlier in the loop (line 341).

---

### Task 2: Pairing commit ordering (transmitter)

**Files:**

- Modify: `transmitter/main/pair/espnow_pair_host.c`
- **Step 1: Move roster persistence from after PAIR_ACK to after PAIR_CONFIRM**

Currently the code (lines 462–493) runs in this order:

1. Lines 462–480: `roster_add()` / `roster_update_existing()` (persists to NVS)
2. Lines 482–488: Send PAIR_CONFIRM
3. Lines 489–492: SUCCESS + cleanup

Replace lines 462–492 with the reordered version. The roster persist block moves to after `esp_now_send(PAIR_CONFIRM)` succeeds:

```c
        pair_confirm_t confirm = { .msg_type = PAIR_MSG_CONFIRM };
        send_err = esp_now_send(rx_mac, (const uint8_t *)&confirm, sizeof(confirm));
        if (send_err != ESP_OK) {
            ESP_LOGW(TAG, "PAIR_CONFIRM send failed: %s", esp_err_to_name(send_err));
            (void)esp_now_del_peer(rx_mac);
            continue;
        }

        if (existing_entry == NULL) {
            roster_entry_t entry = {
                .slot = next_slot,
                .band = rx_band,
                .rf_code_len = offer.rf_code_len,
                .rf_bits = offer.rf_bits,
                .paired_at_ms = time_utils_now_epoch_ms(),
                .occupied = true,
            };
            memcpy(entry.mac, rx_mac, sizeof(entry.mac));
            memcpy(entry.rf_code, offer.rf_code, offer.rf_code_len);
            snprintf(entry.name, sizeof(entry.name), "RX-%02u", next_slot);

            if (roster_add(s_ctx.roster, &entry) != ESP_OK) {
                result = PAIR_HOST_RESULT_ERROR;
                (void)esp_now_del_peer(rx_mac);
                break;
            }
        }

        result = PAIR_HOST_RESULT_SUCCESS;
        paired_slot = next_slot;
        (void)esp_now_del_peer(rx_mac);
        break;
```

The only change is the order: PAIR_CONFIRM is sent first, then `roster_add()` persists to NVS. If PAIR_CONFIRM send fails, the loop retries without leaving a ghost roster entry.

---

### Task 3: Encrypted peer cleanup on retry (receiver-esp32)

**Files:**

- Modify: `receiver-esp32/main/pair/espnow_pair.c`
- **Step 1: Add `hub_peer_added` tracking flag and cleanup before each `continue`**

In `espnow_pair_wait()`, after line 338 (`add_hub_peer()` call), the hub is registered as an encrypted ESP-NOW peer. There are 6 `continue` paths after this point (lines 346, 354, 363, 370, 380, 390) that resume the channel scan without deleting this peer. Add a tracking boolean and cleanup calls.

Add the flag declaration at the top of the outer `for (;;)` loop, just after line 311:

```c
    for (;;) {
        for (uint8_t ch = WIFI_CHANNEL_MIN; ch <= WIFI_CHANNEL_MAX; ch++) {
            bool hub_peer_added = false;
```

After line 338–339 (`add_hub_peer` succeeds — note: if it fails, `ESP_GOTO_ON_ERROR` jumps to `cleanup`, not `continue`, so no tracking needed on that path), add:

```c
            ESP_GOTO_ON_ERROR(add_hub_peer(s_hub_mac, &psk[ESPNOW_CCM_KEY_LEN]),
                              cleanup, TAG, "add hub peer");
            hub_peer_added = true;
```

Then wrap each of the 6 `continue` paths that follow. Replace each `continue;` with:

**Line 346 (HMAC fail):**

```c
            if (compute_hmac(s_nonce, NONCE_LEN, psk, PSK_LEN, hmac) != ESP_OK) {
                ESP_LOGE(TAG, "HMAC computation failed");
                if (hub_peer_added) {
                    (void)esp_now_del_peer(s_hub_mac);
                }
                continue;
            }
```

**Line 354 (RESPONSE send fail):**

```c
            if (send_err != ESP_OK) {
                ESP_LOGW(TAG, "PAIR_RESPONSE send failed: %s", esp_err_to_name(send_err));
                if (hub_peer_added) {
                    (void)esp_now_del_peer(s_hub_mac);
                }
                continue;
            }
```

**Line 363 (OFFER timeout):**

```c
            if ((bits & PAIR_BIT_OFFER) == 0) {
                ESP_LOGW(TAG, "No PAIR_OFFER within %u ms, resuming scan", OFFER_TIMEOUT_MS);
                if (hub_peer_added) {
                    (void)esp_now_del_peer(s_hub_mac);
                }
                continue;
            }
```

**Line 370 (invalid offer):**

```c
            if (!offer_is_valid(&s_offer)) {
                ESP_LOGW(TAG, "Invalid offer: slot=%u band=%u len=%u bits=%u",
                         s_offer.slot, s_offer.rf_band, s_offer.rf_code_len, s_offer.rf_bits);
                if (hub_peer_added) {
                    (void)esp_now_del_peer(s_hub_mac);
                }
                continue;
            }
```

**Line 380 (ACK send fail):**

```c
            if (send_err != ESP_OK) {
                ESP_LOGW(TAG, "PAIR_ACK send failed: %s", esp_err_to_name(send_err));
                if (hub_peer_added) {
                    (void)esp_now_del_peer(s_hub_mac);
                }
                continue;
            }
```

**Line 390 (CONFIRM timeout):**

```c
            if ((bits & PAIR_BIT_CONFIRM) == 0) {
                ESP_LOGW(TAG, "No PAIR_CONFIRM within %u ms, resuming scan without saving",
                         CONFIRM_TIMEOUT_MS);
                if (hub_peer_added) {
                    (void)esp_now_del_peer(s_hub_mac);
                }
                continue;
            }
```

On the success path (line 413–415, `goto cleanup`), the peer remains — `wifi_deinit_all()` in `cleanup` at line 425 tears down ESP-NOW entirely, discarding the peer list.

---

### Task 4: Encrypted peer cleanup on retry (receiver-esp8266)

**Files:**

- Modify: `receiver-esp8266/main/pair/espnow_pair.c`
- **Step 1: Add `hub_peer_added` tracking flag and cleanup before each `continue`**

Same pattern as Task 3, but adapted for the ESP8266 code structure. In `espnow_pair_wait()`, after line 380 (`add_hub_peer()` call), there are 6 `continue` paths (lines 396, 404, 411, 417, 432, 439).

Note: On the ESP8266, line 381 checks `err != ESP_OK && err != ESP_ERR_ESPNOW_EXIST` — if the peer already exists, it's tolerated and the peer IS still present. So `hub_peer_added` should be set to `true` whenever `add_hub_peer()` does NOT return a failure (i.e., returns `ESP_OK` or `ESP_ERR_ESPNOW_EXIST`).

Add the flag declaration at the top of the inner `for` loop, just after line 351:

```c
        for (uint8_t channel = 1; channel <= 13; channel++) {
            EventBits_t bits;
            bool hub_peer_added = false;
```

After line 380–385 (`add_hub_peer` + error check), set the flag:

```c
            err = add_hub_peer(psk + PMK_LEN);
            if (err != ESP_OK && err != ESP_ERR_ESPNOW_EXIST) {
                ESP_LOGW(TAG, "hub peer add failed: %s, resuming scan",
                         esp_err_to_name(err));
                continue;
            }
            hub_peer_added = true;
```

Then wrap each of the 6 `continue` paths:

**Line 396 (HMAC fail):**

```c
            if (err != ESP_OK) {
                ESP_LOGW(TAG, "HMAC computation failed, resuming scan");
                if (hub_peer_added) {
                    (void)esp_now_del_peer(s_hub_mac);
                }
                continue;
            }
```

**Line 404 (RESPONSE send fail):**

```c
            if (err != ESP_OK) {
                ESP_LOGW(TAG, "PAIR_RESPONSE send failed: %s, resuming scan",
                         esp_err_to_name(err));
                if (hub_peer_added) {
                    (void)esp_now_del_peer(s_hub_mac);
                }
                continue;
            }
```

**Line 411 (OFFER timeout):**

```c
            if (!(bits & PAIR_BIT_OFFER)) {
                ESP_LOGW(TAG, "No PAIR_OFFER within 15s, resuming scan");
                if (hub_peer_added) {
                    (void)esp_now_del_peer(s_hub_mac);
                }
                continue;
            }
```

**Line 417 (invalid offer):**

```c
            if (err != ESP_OK) {
                ESP_LOGW(TAG, "Invalid offer, resuming scan");
                if (hub_peer_added) {
                    (void)esp_now_del_peer(s_hub_mac);
                }
                continue;
            }
```

**Line 432 (ACK send fail):**

```c
            if (err != ESP_OK) {
                ESP_LOGW(TAG, "PAIR_ACK send failed: %s, resuming scan",
                         esp_err_to_name(err));
                if (hub_peer_added) {
                    (void)esp_now_del_peer(s_hub_mac);
                }
                continue;
            }
```

**Line 439 (CONFIRM timeout):**

```c
            if (!(bits & PAIR_BIT_CONFIRM)) {
                ESP_LOGW(TAG, "No PAIR_CONFIRM within 10s, resuming scan without saving");
                if (hub_peer_added) {
                    (void)esp_now_del_peer(s_hub_mac);
                }
                continue;
            }
```

On the success path (line 453–454, `goto cleanup`), the peer remains — `wifi_deinit_all()` in `cleanup` at line 460 tears down ESP-NOW entirely, discarding the peer list.

---

### Task 5: ESP8266 offer band validation (receiver-esp8266)

**Files:**

- Modify: `receiver-esp8266/main/pair/espnow_pair.c`
- **Step 1: Add `rf_band` check in `validate_offer()`**

In `validate_offer()` (line 260), insert a band check after the existing `slot == 0` check (line 262–265) and before the `rf_bits` check (line 266):

```c
static esp_err_t validate_offer(const pair_offer_t *offer)
{
    if (offer->slot == 0) {
        ESP_LOGE(TAG, "Pairing offer has invalid slot 0");
        return ESP_ERR_INVALID_ARG;
    }
    if (offer->rf_band != 0) {
        ESP_LOGE(TAG, "Pairing offer has unsupported band=%u "
                 "(ESP8266 is 433M only)", offer->rf_band);
        return ESP_ERR_NOT_SUPPORTED;
    }
    if (offer->rf_bits != 32) {
        ESP_LOGE(TAG, "Pairing offer has unsupported rf_bits=%u", offer->rf_bits);
        return ESP_ERR_INVALID_ARG;
    }
    if (offer->rf_code_len == 0 || offer->rf_code_len > RF_CODE_MAX_LEN) {
        ESP_LOGE(TAG, "Pairing offer has invalid rf_code_len=%u",
                 offer->rf_code_len);
        return ESP_ERR_INVALID_ARG;
    }
    if (offer->rf_code_len != RF_CODE_MAX_LEN) {
        ESP_LOGW(TAG, "Pairing offer rf_code_len=%u for 32-bit code",
                 offer->rf_code_len);
    }

    return ESP_OK;
}
```

---

### Task 6: Update CHANGELOGS.md

- **Step 1: Add changelog entry**

Add entry to `docs/CHANGELOGS.md` documenting:

- Transmitter: `espnow_pair_host.c` — PAIR_ACK handler validates slot field matches offered slot, rejects stale/mismatched ACKs
- Transmitter: `espnow_pair_host.c` — roster commit moved from after PAIR_ACK to after PAIR_CONFIRM send success, prevents ghost NVS entries on CONFIRM failure
- Receiver ESP32: `espnow_pair.c` — encrypted hub ESP-NOW peer deleted before retry `continue`, prevents plaintext challenge rejection on pairing retry
- Receiver ESP8266: `espnow_pair.c` — same encrypted peer cleanup as ESP32
- Receiver ESP8266: `espnow_pair.c` — `validate_offer()` rejects offers with `rf_band != 0` (ESP8266 is 433 MHz only)

