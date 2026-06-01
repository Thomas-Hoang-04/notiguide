# Dispatch & Pairing Protocol Robustness — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix three protocol-level issues: (A) replace toggle-based vibrator control with explicit start/stop pulsing using a 31+1 action bit scheme, (B) add PAIR_SAVED persistence confirmation from receivers before hub roster commit, (C) add session nonce binding to PAIR_ACK and PAIR_SAVED.

**Architecture:** Fix A modifies the radio transmit API to carry an action parameter, then updates both the transmitter dispatch and all receiver trigger handlers. Fixes B and C modify the ESP-NOW pairing protocol structs and state machines across all three firmware codebases. All fixes are hub-paired-only — passive/server-managed receivers are unaffected.

**Tech Stack:** C / ESP-IDF v6.0 (transmitter + receiver-esp32) / ESP8266 RTOS SDK (receiver-esp8266) / FreeRTOS

**Spec:** `docs/spec/Dispatch & Pairing Protocol Robustness Spec.md`

---

## Existing Codebase Context

### Key files

| File | Role | Fixes |
|------|------|-------|
| `transmitter/main/dispatch/radio_supervisor.h` | Radio transmit API declaration | A |
| `transmitter/main/dispatch/radio_supervisor.c` | Dual-radio transmit arbiter (433M + nRF24) | A |
| `transmitter/main/nrf24/nrf24_transmitter.h` | nRF24 PTX driver API | A |
| `transmitter/main/nrf24/nrf24_transmitter.c` | nRF24 PTX driver — sends TOGGLE_MAGIC payload | A |
| `transmitter/main/dispatch/dispatch.c` | MQTT dispatch handler — slot + full-payload paths | A |
| `transmitter/main/serial/serial_protocol.c` | USB serial dispatch handler | A |
| `transmitter/main/pair/espnow_pair_host.c` | Hub-side pairing state machine | A, B, C |
| `receiver-esp32/main/trigger/rf_trigger.c` | Trigger matcher — 433M frames + nRF24 packets | A |
| `receiver-esp32/main/pair/espnow_pair.c` | ESP32 receiver pairing state machine | B, C |
| `receiver-esp8266/main/trigger/rf_trigger.c` | Trigger matcher — 433M frames only | A |
| `receiver-esp8266/main/pair/espnow_pair.c` | ESP8266 receiver pairing state machine | B, C |

### Current API signatures (verified)

- `radio_tx_send(radio_band_t band, const uint8_t *payload, size_t byte_len, uint8_t bits, bool proto_any)` — 3 callers: dispatch.c:433 (slot), dispatch.c:551 (full-payload), serial_protocol.c:496
- `nrf24_tx_send(nrf24_tx_t *h, const uint8_t addr[5])` — 1 caller: radio_supervisor.c:106. Sends static `TOGGLE_MAGIC = {0xAA, 0x55}`
- `vibrator_set_pulsing(VibratorHandler*, bool enabled)` — idempotent, exists on both receiver platforms
- `vibrator_toggle_pulsing(VibratorHandler*)` — being replaced; calls `set_pulsing(!current)` internally

### Pairing protocol constants (transmitter)

- `PAIR_MSG_REQUEST 0x01` through `PAIR_MSG_CONFIRM 0x06`
- `PAIR_BIT_REQUEST BIT0` through `PAIR_BIT_CONFIRM_BTN BIT4`
- `pair_ack_t` = `{msg_type, slot}` (2 bytes packed)
- `pair_host_ctx_t` has `rx_ack` field (added by prior robustness fix)

---

### Task 1: Add action parameter to radio transmit API (transmitter)

**Files:**

- Modify: `transmitter/main/dispatch/radio_supervisor.h`
- Modify: `transmitter/main/dispatch/radio_supervisor.c`
- Modify: `transmitter/main/nrf24/nrf24_transmitter.h`
- Modify: `transmitter/main/nrf24/nrf24_transmitter.c`

- **Step 1: Add action defines and update `radio_tx_send` signature in header**

In `radio_supervisor.h`, add defines after the `radio_band_t` enum (after line 24) and update the function signature (lines 53-57):

```c
typedef enum {
    RADIO_BAND_433M = 0,
    RADIO_BAND_2_4G = 1,
} radio_band_t;

#define RADIO_ACTION_START  0x01
#define RADIO_ACTION_STOP   0x02
```

Replace the `radio_tx_send` declaration:

```c
esp_err_t radio_tx_send(radio_band_t band,
                        const uint8_t *payload,
                        size_t byte_len,
                        uint8_t bits,
                        bool proto_any,
                        uint8_t action);
```

- **Step 2: Update `nrf24_tx_send` signature in header**

In `nrf24_transmitter.h`, replace the declaration at line 48:

```c
esp_err_t nrf24_tx_send(nrf24_tx_t *h, const uint8_t addr[5], uint8_t action);
```

- **Step 3: Update `nrf24_tx_send` implementation to build dynamic payload**

In `nrf24_transmitter.c`, remove the static TOGGLE_MAGIC at line 24:

```c
// DELETE: static const uint8_t TOGGLE_MAGIC[2] = { NRF_TX_TOGGLE_MAGIC_HI, NRF_TX_TOGGLE_MAGIC_LO };
```

Update the function signature at line 277:

```c
esp_err_t nrf24_tx_send(nrf24_tx_t *h, const uint8_t addr[5], uint8_t action)
```

Replace the `W_TX_PAYLOAD` write at line 304. Change:

```c
    ret = nrf_xfer(h, NRF_CMD_W_TX_PAYLOAD, TOGGLE_MAGIC, NULL, sizeof(TOGGLE_MAGIC), NULL);
```

To:

```c
    const uint8_t tx_payload[3] = { NRF_TX_TOGGLE_MAGIC_HI, NRF_TX_TOGGLE_MAGIC_LO, action };
    ret = nrf_xfer(h, NRF_CMD_W_TX_PAYLOAD, tx_payload, NULL, sizeof(tx_payload), NULL);
```

- **Step 4: Update `radio_tx_send` implementation to pass action**

In `radio_supervisor.c`, update the function signature at line 79:

```c
esp_err_t radio_tx_send(radio_band_t band,
                        const uint8_t *payload,
                        size_t byte_len,
                        uint8_t bits,
                        bool proto_any,
                        uint8_t action)
```

Update the nRF24 call at line 106. Change:

```c
            err = nrf24_tx_send(&s_nrf24, payload);
```

To:

```c
            err = nrf24_tx_send(&s_nrf24, payload, action);
```

---

### Task 2: Update transmitter dispatch callers (transmitter)

**Files:**

- Modify: `transmitter/main/dispatch/dispatch.c`
- Modify: `transmitter/main/serial/serial_protocol.c`

- **Step 1: Update slot dispatch to use action bit for 433M and pass action**

In `dispatch.c`, replace the slot dispatch radio call at lines 432-437. Change:

```c
    slot_to_transmit_cmd(&slot_cmd, entry, &tx_cmd);
    const esp_err_t tx_result = radio_tx_send((radio_band_t)entry->band,
                                              entry->rf_code,
                                              entry->rf_code_len,
                                              entry->rf_bits,
                                              true);
```

To:

```c
    slot_to_transmit_cmd(&slot_cmd, entry, &tx_cmd);

    uint8_t tx_code[ROSTER_MAX_CODE_LEN];
    memcpy(tx_code, entry->rf_code, entry->rf_code_len);
    const uint8_t action = strcmp(slot_cmd.action, "stop") == 0
                         ? RADIO_ACTION_STOP
                         : RADIO_ACTION_START;
    if (action == RADIO_ACTION_STOP && entry->band == RADIO_BAND_433M) {
        tx_code[0] |= 0x80;
    }

    const esp_err_t tx_result = radio_tx_send((radio_band_t)entry->band,
                                              tx_code,
                                              entry->rf_code_len,
                                              entry->rf_bits,
                                              true,
                                              action);
```

Note: `ROSTER_MAX_CODE_LEN` is 16, defined in `pair/roster.h` which is already included. The `radio_supervisor.h` include at line 15 provides `RADIO_ACTION_START`/`RADIO_ACTION_STOP`.

- **Step 2: Update full-payload dispatch to pass RADIO_ACTION_START**

In `dispatch.c`, update the full-payload dispatch call at line 551. Change:

```c
    const esp_err_t tx_result = radio_tx_send(band, plaintext, byte_len, (uint8_t)cmd.rf_code_bits, cmd.proto_any);
```

To:

```c
    const esp_err_t tx_result = radio_tx_send(band, plaintext, byte_len, (uint8_t)cmd.rf_code_bits, cmd.proto_any, RADIO_ACTION_START);
```

- **Step 3: Update serial dispatch to pass RADIO_ACTION_START**

In `serial_protocol.c`, update the radio call at lines 496-497. Change:

```c
    esp_err_t tx_result = radio_tx_send(band, plaintext, byte_len,
                                        (uint8_t)rf_code_bits, proto_any);
```

To:

```c
    esp_err_t tx_result = radio_tx_send(band, plaintext, byte_len,
                                        (uint8_t)rf_code_bits, proto_any,
                                        RADIO_ACTION_START);
```

---

### Task 3: Update 433M code generation to reserve MSB (transmitter)

**Files:**

- Modify: `transmitter/main/pair/espnow_pair_host.c`

- **Step 1: Clear MSB inside the generation loop**

In `espnow_pair_host.c`, replace `generate_rf_code` band-0 block at lines 178-184. Change:

```c
    if (band == 0) {
        *code_len = 4;
        *bits = 32;
        do {
            esp_fill_random(code, 4);
        } while (is_forbidden_433m(code));
        return;
    }
```

To:

```c
    if (band == 0) {
        *code_len = 4;
        *bits = 32;
        do {
            esp_fill_random(code, 4);
            code[0] &= 0x7F;
        } while (is_forbidden_433m(code));
        return;
    }
```

The MSB must be cleared inside the loop before `is_forbidden_433m` to prevent a value like `{0x80, 0x00, 0x00, 0x00}` (passes forbidden check as non-uniform nibbles) from becoming `{0x00, 0x00, 0x00, 0x00}` (all-zeros) after clearing.

---

### Task 4: Update ESP32 receiver rf_trigger for explicit start/stop

**Files:**

- Modify: `receiver-esp32/main/trigger/rf_trigger.c`

- **Step 1: Replace toggle with explicit start/stop in `rf_trigger_on_frame`**

In `rf_trigger.c`, replace the matching and toggle block at lines 149-168. Change:

```c
    uint32_t expected = 0;
    for (size_t i = 0; i < required_bytes; ++i) {
        expected = (expected << 8) | state.code[i];
    }
    uint32_t mask = state.bits >= 32 ? UINT32_MAX : ((1UL << state.bits) - 1UL);

    if ((value & mask) != (expected & mask)) {
        return;
    }

    if (esp_timer_get_time() - s_last_toggle_us < RF_TRIGGER_TOGGLE_DEBOUNCE_US) {
        return;
    }

    ESP_LOGI(RF_TRIGGER_TAG, "RF match on %u bits", value_bits);
    if (vibrator_toggle_pulsing(&s_vibrator) != ESP_OK) {
        ESP_LOGE(RF_TRIGGER_TAG, "failed to toggle vibrator pulsing");
        return;
    }
    s_last_toggle_us = esp_timer_get_time();
```

To:

```c
    uint32_t expected = 0;
    for (size_t i = 0; i < required_bytes; ++i) {
        expected = (expected << 8) | state.code[i];
    }

    uint8_t action_bit = (value >> 31) & 1U;
    uint32_t code_only = value & 0x7FFFFFFFU;
    uint32_t exp_masked = expected & 0x7FFFFFFFU;
    uint32_t mask = state.bits >= 32 ? 0x7FFFFFFFU : ((1UL << state.bits) - 1UL);

    if ((code_only & mask) != (exp_masked & mask)) {
        return;
    }

    if (esp_timer_get_time() - s_last_toggle_us < RF_TRIGGER_TOGGLE_DEBOUNCE_US) {
        return;
    }

    bool start = (action_bit == 0);
    ESP_LOGI(RF_TRIGGER_TAG, "RF match on %u bits, action=%s", value_bits, start ? "start" : "stop");
    if (vibrator_set_pulsing(&s_vibrator, start) != ESP_OK) {
        ESP_LOGE(RF_TRIGGER_TAG, "failed to set vibrator pulsing");
        return;
    }
    s_last_toggle_us = esp_timer_get_time();
```

- **Step 2: Replace toggle with explicit start/stop in `rf_trigger_on_packet`**

In `rf_trigger.c`, replace `rf_trigger_on_packet` at lines 171-180:

```c
void rf_trigger_on_packet(const uint8_t *pkt, size_t pkt_len)
{
    if (pkt == NULL || pkt_len < 2U) {
        return;
    }
    if (pkt[0] != RF_TRIGGER_TOGGLE_MAGIC_HI || pkt[1] != RF_TRIGGER_TOGGLE_MAGIC_LO) {
        return;
    }
    vibrator_toggle_pulsing(&s_vibrator);
}
```

To:

```c
void rf_trigger_on_packet(const uint8_t *pkt, size_t pkt_len)
{
    if (pkt == NULL || pkt_len < 3U) {
        return;
    }
    if (pkt[0] != RF_TRIGGER_TOGGLE_MAGIC_HI || pkt[1] != RF_TRIGGER_TOGGLE_MAGIC_LO) {
        return;
    }
    vibrator_set_pulsing(&s_vibrator, pkt[2] == 0x01);
}
```

---

### Task 5: Update ESP8266 receiver rf_trigger for explicit start/stop

**Files:**

- Modify: `receiver-esp8266/main/trigger/rf_trigger.c`

- **Step 1: Replace toggle with explicit start/stop in `rf_trigger_on_frame`**

In `rf_trigger.c`, replace the matching and toggle block at lines 124-138. Change:

```c
    mask = (snapshot.bits >= 32) ? UINT32_MAX : ((1u << snapshot.bits) - 1u);
    if ((decoded & mask) != (snapshot.code & mask)) {
        return;
    }

    if (esp_timer_get_time() - s_last_toggle_us < RF_TRIGGER_TOGGLE_DEBOUNCE_US) {
        return;
    }

    ESP_LOGI(RF_TRIGGER_TAG, "RF match on %u bits", decoded_bits);
    if (vibrator_toggle_pulsing(s_vibrator) != ESP_OK) {
        ESP_LOGE(RF_TRIGGER_TAG, "Failed to toggle vibrator pulsing");
        return;
    }
    s_last_toggle_us = esp_timer_get_time();
```

To:

```c
    uint8_t action_bit = (decoded >> 31) & 1u;
    uint32_t code_only = decoded & 0x7FFFFFFFu;
    uint32_t exp_masked = snapshot.code & 0x7FFFFFFFu;
    mask = (snapshot.bits >= 32) ? 0x7FFFFFFFu : ((1u << snapshot.bits) - 1u);

    if ((code_only & mask) != (exp_masked & mask)) {
        return;
    }

    if (esp_timer_get_time() - s_last_toggle_us < RF_TRIGGER_TOGGLE_DEBOUNCE_US) {
        return;
    }

    bool start = (action_bit == 0);
    ESP_LOGI(RF_TRIGGER_TAG, "RF match on %u bits, action=%s", decoded_bits, start ? "start" : "stop");
    if (vibrator_set_pulsing(s_vibrator, start) != ESP_OK) {
        ESP_LOGE(RF_TRIGGER_TAG, "Failed to set vibrator pulsing");
        return;
    }
    s_last_toggle_us = esp_timer_get_time();
```

Note: ESP8266 uses `s_vibrator` (pointer) not `&s_vibrator` (address-of). The ESP8266 has no nRF24, so no `rf_trigger_on_packet` changes needed.

---

### Task 6: Update pairing protocol structs and hub state machine (transmitter)

**Files:**

- Modify: `transmitter/main/pair/espnow_pair_host.c`

- **Step 1: Add PAIR_MSG_SAVED define and PAIR_BIT_SAVED**

After `PAIR_MSG_CONFIRM` (line 34) and `PAIR_BIT_CONFIRM_BTN` (line 40), add:

```c
#define PAIR_MSG_SAVED        0x07

#define PAIR_BIT_SAVED        BIT5
```

- **Step 2: Update `pair_ack_t` to include nonce_tag**

Replace `pair_ack_t` at lines 67-70:

```c
typedef struct __attribute__((packed)) {
    uint8_t msg_type;
    uint8_t slot;
} pair_ack_t;
```

With:

```c
typedef struct __attribute__((packed)) {
    uint8_t msg_type;
    uint8_t slot;
    uint8_t nonce_tag[4];
} pair_ack_t;
```

- **Step 3: Add `pair_saved_t` struct**

After `pair_confirm_t` (lines 72-74), add:

```c
typedef struct __attribute__((packed)) {
    uint8_t msg_type;
    uint8_t slot;
    uint8_t nonce_tag[4];
} pair_saved_t;
```

- **Step 4: Add `rx_saved` field to `pair_host_ctx_t`**

In `pair_host_ctx_t` (lines 76-90), add `pair_saved_t rx_saved;` after `pair_ack_t rx_ack;`:

```c
    pair_response_t rx_response;
    pair_ack_t rx_ack;
    pair_saved_t rx_saved;
    bool active;
```

- **Step 5: Add `PAIR_MSG_SAVED` case in `recv_cb`**

After the `PAIR_MSG_ACK` case (lines 232-246), add a new case before the `default:`:

```c
    case PAIR_MSG_SAVED:
        if ((size_t)len >= sizeof(pair_saved_t)) {
            uint8_t rx_mac[6];
            portENTER_CRITICAL(&s_ctx_mux);
            memcpy(rx_mac, s_ctx.rx_mac, sizeof(rx_mac));
            portEXIT_CRITICAL(&s_ctx_mux);
            if (memcmp(rx_mac, info->src_addr, sizeof(rx_mac)) != 0) {
                return;
            }
            portENTER_CRITICAL(&s_ctx_mux);
            memcpy(&s_ctx.rx_saved, data, sizeof(s_ctx.rx_saved));
            portEXIT_CRITICAL(&s_ctx_mux);
            xEventGroupSetBits(s_ctx.events, PAIR_BIT_SAVED);
        }
        break;
```

- **Step 6: Add nonce_tag validation after ACK slot check**

In `pair_host_task`, after the existing slot validation block (lines 466-476), add nonce_tag validation. After the slot mismatch check:

```c
        if (ack_slot != next_slot) {
            ESP_LOGW(TAG, "PAIR_ACK slot mismatch: expected=%u got=%u",
                     next_slot, ack_slot);
            (void)esp_now_del_peer(rx_mac);
            continue;
        }
```

Add:

```c
        uint8_t ack_nonce_tag[4];
        portENTER_CRITICAL(&s_ctx_mux);
        memcpy(ack_nonce_tag, s_ctx.rx_ack.nonce_tag, sizeof(ack_nonce_tag));
        portEXIT_CRITICAL(&s_ctx_mux);

        if (memcmp(ack_nonce_tag, s_ctx.nonce, sizeof(ack_nonce_tag)) != 0) {
            ESP_LOGW(TAG, "PAIR_ACK nonce_tag mismatch");
            (void)esp_now_del_peer(rx_mac);
            continue;
        }
```

Note: `s_ctx.rx_ack.slot` was already read under the spinlock in the existing slot validation block. The nonce_tag read is a separate critical section to keep each section minimal.

- **Step 7: Replace roster commit with PAIR_SAVED wait + validation**

Replace the block from `pair_confirm_t confirm = ...` (line 478) through `break;` (line 509) with:

```c
        pair_confirm_t confirm = { .msg_type = PAIR_MSG_CONFIRM };
        send_err = esp_now_send(rx_mac, (const uint8_t *)&confirm, sizeof(confirm));
        if (send_err != ESP_OK) {
            ESP_LOGW(TAG, "PAIR_CONFIRM send failed: %s", esp_err_to_name(send_err));
            (void)esp_now_del_peer(rx_mac);
            continue;
        }

        remaining = ticks_until(deadline);
        if (remaining == 0) {
            break;
        }
        TickType_t saved_wait = pdMS_TO_TICKS(10000);
        if (saved_wait > remaining) {
            saved_wait = remaining;
        }
        bits = xEventGroupWaitBits(
            s_ctx.events, PAIR_BIT_SAVED | PAIR_BIT_CANCEL, pdTRUE, pdFALSE, saved_wait);
        if ((bits & PAIR_BIT_CANCEL) != 0) {
            result = PAIR_HOST_RESULT_CANCELLED;
            break;
        }
        if ((bits & PAIR_BIT_SAVED) == 0) {
            ESP_LOGW(TAG, "No PAIR_SAVED within 10s");
            (void)esp_now_del_peer(rx_mac);
            continue;
        }

        uint8_t saved_slot;
        uint8_t saved_nonce_tag[4];
        portENTER_CRITICAL(&s_ctx_mux);
        saved_slot = s_ctx.rx_saved.slot;
        memcpy(saved_nonce_tag, s_ctx.rx_saved.nonce_tag, sizeof(saved_nonce_tag));
        portEXIT_CRITICAL(&s_ctx_mux);

        if (saved_slot != next_slot ||
            memcmp(saved_nonce_tag, s_ctx.nonce, sizeof(saved_nonce_tag)) != 0) {
            ESP_LOGW(TAG, "PAIR_SAVED validation failed: slot=%u nonce_match=%d",
                     saved_slot,
                     memcmp(saved_nonce_tag, s_ctx.nonce, sizeof(saved_nonce_tag)) == 0);
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

---

### Task 7: Update ESP32 receiver pairing for nonce_tag + PAIR_SAVED

**Files:**

- Modify: `receiver-esp32/main/pair/espnow_pair.c`

- **Step 1: Add PAIR_MSG_SAVED define**

After `PAIR_MSG_CONFIRM` (line 33), add:

```c
#define PAIR_MSG_SAVED    0x07
```

- **Step 2: Update `pair_ack_t` to include nonce_tag**

Replace `pair_ack_t` at lines 73-76:

```c
typedef struct __attribute__((packed)) {
    uint8_t msg_type;
    uint8_t slot;
} pair_ack_t;
```

With:

```c
typedef struct __attribute__((packed)) {
    uint8_t msg_type;
    uint8_t slot;
    uint8_t nonce_tag[4];
} pair_ack_t;
```

- **Step 3: Add `pair_saved_t` struct**

After `pair_confirm_t` (lines 78-80), add:

```c
typedef struct __attribute__((packed)) {
    uint8_t msg_type;
    uint8_t slot;
    uint8_t nonce_tag[4];
} pair_saved_t;
```

- **Step 4: Include nonce_tag when building PAIR_ACK**

In `espnow_pair_wait`, replace the ACK construction at line 389:

```c
            pair_ack_t ack = { .msg_type = PAIR_MSG_ACK, .slot = s_offer.slot };
```

With:

```c
            pair_ack_t ack = { .msg_type = PAIR_MSG_ACK, .slot = s_offer.slot };
            memcpy(ack.nonce_tag, s_nonce, sizeof(ack.nonce_tag));
```

- **Step 5: Send PAIR_SAVED after successful NVS save**

After the NVS save success and the "Pairing complete" log (lines 433-434), before `goto cleanup`, add the PAIR_SAVED send:

```c
            ESP_LOGI(TAG, "Pairing complete, slot=%u", s_offer.slot);

            pair_saved_t saved = { .msg_type = PAIR_MSG_SAVED, .slot = s_offer.slot };
            memcpy(saved.nonce_tag, s_nonce, sizeof(saved.nonce_tag));
            esp_err_t saved_err = esp_now_send(s_hub_mac, (const uint8_t *)&saved, sizeof(saved));
            if (saved_err != ESP_OK) {
                ESP_LOGW(TAG, "PAIR_SAVED send failed: %s", esp_err_to_name(saved_err));
            }

            ret = ESP_OK;
            goto cleanup;
```

The PAIR_SAVED send is best-effort — if it fails, the hub will time out and the receiver can be re-paired. The receiver's NVS state is already committed at this point.

---

### Task 8: Update ESP8266 receiver pairing for nonce_tag + PAIR_SAVED

**Files:**

- Modify: `receiver-esp8266/main/pair/espnow_pair.c`

- **Step 1: Add PAIR_MSG_SAVED define**

After `PAIR_MSG_CONFIRM` (line 39), add:

```c
#define PAIR_MSG_SAVED    0x07
```

- **Step 2: Update `pair_ack_t` to include nonce_tag**

Replace `pair_ack_t` at lines 77-80:

```c
typedef struct __attribute__((packed)) {
    uint8_t msg_type;
    uint8_t slot;
} pair_ack_t;
```

With:

```c
typedef struct __attribute__((packed)) {
    uint8_t msg_type;
    uint8_t slot;
    uint8_t nonce_tag[4];
} pair_ack_t;
```

- **Step 3: Add `pair_saved_t` struct**

After `pair_confirm_t` (lines 83-85), add:

```c
typedef struct __attribute__((packed)) {
    uint8_t msg_type;
    uint8_t slot;
    uint8_t nonce_tag[4];
} pair_saved_t;
```

- **Step 4: Include nonce_tag when building PAIR_ACK**

In `espnow_pair_wait`, replace the ACK construction at lines 441-444:

```c
            pair_ack_t ack = {
                .msg_type = PAIR_MSG_ACK,
                .slot = s_offer.slot,
            };
```

With:

```c
            pair_ack_t ack = {
                .msg_type = PAIR_MSG_ACK,
                .slot = s_offer.slot,
            };
            memcpy(ack.nonce_tag, s_nonce, sizeof(ack.nonce_tag));
```

- **Step 5: Send PAIR_SAVED after successful NVS save**

After the "Pairing complete" log (line 480), before `goto cleanup`, add the PAIR_SAVED send:

```c
            ESP_LOGI(TAG, "Pairing complete, slot=%u", s_offer.slot);

            pair_saved_t saved = { .msg_type = PAIR_MSG_SAVED, .slot = s_offer.slot };
            memcpy(saved.nonce_tag, s_nonce, sizeof(saved.nonce_tag));
            esp_err_t saved_err = esp_now_send(s_hub_mac, (const uint8_t *)&saved, sizeof(saved));
            if (saved_err != ESP_OK) {
                ESP_LOGW(TAG, "PAIR_SAVED send failed: %s", esp_err_to_name(saved_err));
            }

            err = ESP_OK;
            goto cleanup;
```

Note: ESP8266 uses `err` (not `ret`) for the return value, and `s_nonce` is `uint8_t[NONCE_LEN]` (same as ESP32).

---

### Task 9: Update CHANGELOGS.md

**Files:**

- Modify: `docs/CHANGELOGS.md`

- **Step 1: Add changelog entry**

Add entry to `docs/CHANGELOGS.md` documenting:

- Transmitter: `radio_supervisor.h/.c` — `radio_tx_send` gains `action` parameter (`RADIO_ACTION_START`/`RADIO_ACTION_STOP`); 433M ignores it (caller handles MSB), nRF24 encodes it in 3-byte payload
- Transmitter: `nrf24_transmitter.c` — `nrf24_tx_send` builds dynamic `{MAGIC_HI, MAGIC_LO, action}` payload, removes static `TOGGLE_MAGIC`
- Transmitter: `dispatch.c` — slot dispatch sets/clears MSB for 433M action bit, maps action string to `RADIO_ACTION_*`; full-payload dispatch passes `RADIO_ACTION_START`
- Transmitter: `serial_protocol.c` — passes `RADIO_ACTION_START` to `radio_tx_send`
- Transmitter: `espnow_pair_host.c` — `generate_rf_code` clears MSB inside loop for 31+1 scheme; `pair_ack_t` gains `nonce_tag[4]`; new `pair_saved_t` struct and `PAIR_MSG_SAVED` handler; roster commit gated on receiver PAIR_SAVED confirmation with slot + nonce_tag validation
- Receiver ESP32: `rf_trigger.c` — `rf_trigger_on_frame` extracts MSB action bit, matches on 31 bits, calls `vibrator_set_pulsing`; `rf_trigger_on_packet` reads 3rd byte for action
- Receiver ESP32: `espnow_pair.c` — includes `nonce_tag` in PAIR_ACK; sends PAIR_SAVED after NVS save
- Receiver ESP8266: `rf_trigger.c` — same 31+1 action bit scheme as ESP32 (433M only)
- Receiver ESP8266: `espnow_pair.c` — includes `nonce_tag` in PAIR_ACK; sends PAIR_SAVED after NVS save
