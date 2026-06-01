# Dispatch & Pairing Protocol Robustness — Spec

**Goal:** Fix three protocol-level issues across the transmitter and both receiver codebases: (A) replace toggle-based vibrator control with explicit on/off, (B) add receiver persistence confirmation before hub roster commit, (C) add session binding to PAIR_ACK.

**Scope:** Hub-paired receivers only. Passive/server-managed receivers are unaffected. These are breaking changes — all hub-paired receivers must be re-paired after firmware update.

**Tech Stack:** C / ESP-IDF v6.0 (transmitter + receiver-esp32) / ESP8266 RTOS SDK (receiver-esp8266) / FreeRTOS

---

## 1. Fix A — Explicit Start/Stop Pulsing (replaces toggle)

### Problem

Both receiver platforms call `vibrator_toggle_pulsing()` on every RF match. The hub sends the same RF code regardless of whether the backend requested "call" or "stop". A missed call + subsequent stop toggles the vibrator ON instead of OFF. A duplicated call toggles it immediately OFF. The pulsing behavior itself is correct — the problem is the toggle-based activation.

### Design

#### 433 MHz: 31-bit code + 1 action bit (MSB)

The 32-bit value transmitted over 433 MHz is partitioned:

```
Bit 31 (MSB)    Bits 30:0
┌───────────┬────────────────────┐
│ action    │ rf_code (31 bits)  │
│ 0 = start │                    │
│ 1 = stop  │                    │
└───────────┴────────────────────┘
```

**Pairing (code generation):** `generate_rf_code()` in `espnow_pair_host.c` currently generates 32-bit random codes for band 0. Clear the MSB **inside** the generation loop, before the forbidden check:

```c
do {
    esp_fill_random(code, 4);
    code[0] &= 0x7F;  // Reserve MSB for action bit — clear BEFORE forbidden check
} while (is_forbidden_433m(code));
```

The MSB must be cleared inside the loop because `is_forbidden_433m` checks nibble uniformity on the actual stored code. Without this, a random value like `{0x80, 0x00, 0x00, 0x00}` passes the forbidden check (nibbles are 8,0,0,0... — not uniform) but becomes `{0x00, 0x00, 0x00, 0x00}` after clearing — the all-zeros code.

The `rf_bits` field stays 32 in storage. The effective code entropy is 31 bits (~2 billion codes).

**Hub dispatch** (`dispatch.c`, slot dispatch path): After looking up the roster entry's RF code, set or clear the MSB based on `slot_cmd.action`:

```c
uint8_t tx_code[4];
memcpy(tx_code, entry->rf_code, 4);
if (strcmp(slot_cmd.action, "stop") == 0) {
    tx_code[0] |= 0x80;   // Set MSB = 1 (stop)
}
radio_tx_send(RADIO_BAND_433M, tx_code, 4, 32, true);
```

For "call", the MSB is already 0 (cleared during pairing).

**Receiver 433M** (`rf_trigger_on_frame` in both receivers): Extract the action bit, match on the lower 31 bits, call explicit start/stop:

```c
uint8_t action_bit = (value >> 31) & 1;
uint32_t code_only = value & 0x7FFFFFFFU;
uint32_t mask = state.bits >= 32 ? 0x7FFFFFFFU : ((1UL << state.bits) - 1UL);

if ((code_only & mask) != (expected & mask)) {
    return;
}
// ... debounce check ...
vibrator_set_pulsing(&s_vibrator, action_bit == 0);  // 0 = start pulsing, 1 = stop pulsing
```

`vibrator_set_pulsing(handler, bool enabled)` already exists on both platforms (verified). It is idempotent — calling `set_pulsing(true)` when already pulsing is a no-op. This eliminates the toggle problem entirely.

#### nRF24 (2.4 GHz): Extended packet payload

**Current:** Transmitter sends `TOGGLE_MAGIC = {0xAA, 0x55}` (2 bytes) as the nRF24 payload via `NRF_CMD_W_TX_PAYLOAD`. Receiver checks for these magic bytes in `rf_trigger_on_packet` and toggles.

**New:** Extend payload to 3 bytes: `{0xAA, 0x55, ACTION}` where:
- `ACTION = 0x01` → start pulsing
- `ACTION = 0x02` → stop pulsing

**Transmitter** (`nrf24_transmitter.c`): `nrf24_tx_send` currently writes a static `TOGGLE_MAGIC`. Add an `action` parameter:

```c
esp_err_t nrf24_tx_send(nrf24_tx_t *h, const uint8_t addr[5], uint8_t action);
```

Build the payload dynamically: `{NRF_TX_TOGGLE_MAGIC_HI, NRF_TX_TOGGLE_MAGIC_LO, action}`.

`radio_tx_send` must propagate the action to the nRF24 path. Add a mandatory `action` parameter:

```c
#define RADIO_ACTION_START  0x01
#define RADIO_ACTION_STOP   0x02

esp_err_t radio_tx_send(radio_band_t band,
                        const uint8_t *payload,
                        size_t byte_len,
                        uint8_t bits,
                        bool proto_any,
                        uint8_t action);  // RADIO_ACTION_START or RADIO_ACTION_STOP
```

**All callers updated:** `radio_tx_send` has 3 call sites:
1. `dispatch.c` slot dispatch — maps `slot_cmd.action` to `RADIO_ACTION_START` / `RADIO_ACTION_STOP`
2. `dispatch.c` full-payload dispatch (line 551) — passes `RADIO_ACTION_START`. This path only serves passive receivers, which handle call/stop via different RF codes patched by the backend (PT2272 channel bytes). The action parameter is unused for 433M — MSB manipulation only applies to hub-paired codes, which are handled by the slot dispatch path.
3. `serial_protocol.c` (line 496) — passes `RADIO_ACTION_START`. Serial dispatch is for testing; the action could be extended to accept start/stop from the serial command in a future iteration.

**Per-band behavior:**
- **433M:** `action` is ignored by `radio_tx_send` — the caller (slot dispatch) has already set/cleared the MSB in the payload before calling. Passive receivers' codes are sent as-is.
- **nRF24:** `action` is passed to `nrf24_tx_send`, which builds the payload `{MAGIC_HI, MAGIC_LO, action}` (3 bytes). All nRF24 receivers must run the new firmware — no legacy 2-byte fallback needed since all non-passive receivers will be hub-paired and re-flashed.

**Receiver ESP32** (`rf_trigger_on_packet`): Require 3-byte payload with action byte:

```c
void rf_trigger_on_packet(const uint8_t *pkt, size_t pkt_len)
{
    if (pkt == NULL || pkt_len < 3U) {
        return;
    }
    if (pkt[0] != RF_TRIGGER_TOGGLE_MAGIC_HI || pkt[1] != RF_TRIGGER_TOGGLE_MAGIC_LO) {
        return;
    }
    vibrator_set_pulsing(&s_vibrator, pkt[2] == RADIO_ACTION_START);
}
```

**ESP8266:** No nRF24 hardware. No changes needed for this path.

#### Backend

No backend changes needed for Fix A. The backend already sends `action: "call"` or `action: "stop"` in the MQTT `cmd/transmit` payload for slot dispatch. The hub firmware currently ignores the action for RF transmission — after this fix, it uses it.

#### Collision check

The 31-bit code scheme does not affect collision detection. The backend's `findMaskedCollision` (planned in Local Pairing plan Task 14) compares codes by their stored values. The MSB is always 0 in storage, so two codes from different receivers cannot differ only in the MSB. No collision check changes needed.

---

## 2. Fix B — PAIR_SAVED Message

### Problem

The hub commits roster entries after `esp_now_send(PAIR_CONFIRM)` returns `ESP_OK`, which only means the packet was enqueued locally. The receiver may never receive CONFIRM, or its NVS save may fail. The hub ends up with a roster entry for a receiver that doesn't have the pairing.

### Design

New message type `PAIR_MSG_SAVED` (0x07). The receiver sends it back to the hub after successfully persisting the pairing to NVS. The hub only commits its roster after receiving PAIR_SAVED.

#### Updated protocol flow

```
Hub                              Receiver
 │                                │
 │── PAIR_OFFER ────────────────→ │
 │                                │
 │←──────────────── PAIR_ACK ────│
 │                                │
 │── PAIR_CONFIRM ──────────────→ │
 │                                │ saves to NVS (with retries)
 │                                │
 │←──────────────── PAIR_SAVED ──│  (new)
 │                                │
 │ commits roster to NVS          │
```

#### New struct (all 3 codebases)

```c
#define PAIR_MSG_SAVED  0x07

typedef struct __attribute__((packed)) {
    uint8_t msg_type;       // PAIR_MSG_SAVED (0x07)
    uint8_t slot;
    uint8_t nonce_tag[4];   // session binding (see Fix C)
} pair_saved_t;
```

#### Transmitter changes (`espnow_pair_host.c`)

1. Add `#define PAIR_BIT_SAVED BIT5` (or next available bit)
2. Add `pair_saved_t rx_saved;` field to `pair_host_ctx_t`
3. Add `PAIR_MSG_SAVED` case in `recv_cb`: spinlock MAC check + copy (same pattern as ACK handler)
4. In `pair_host_task`, after sending PAIR_CONFIRM successfully:
   - Wait for `PAIR_BIT_SAVED` with a 10-second timeout
   - On timeout → log, del_peer, continue (retry)
   - On receive → validate `slot` and `nonce_tag` (see Fix C), then proceed to `roster_add()`

The current code flow (after Fix from the Robustness Fixes plan):
```
send PAIR_CONFIRM → roster_add → SUCCESS
```

Becomes:
```
send PAIR_CONFIRM → wait PAIR_SAVED → validate → roster_add → SUCCESS
```

#### Receiver ESP32 changes (`espnow_pair.c`)

After `device_config_save_pairing()` succeeds (with retries), send PAIR_SAVED:

```c
pair_saved_t saved = {
    .msg_type = PAIR_MSG_SAVED,
    .slot = s_offer.slot,
};
memcpy(saved.nonce_tag, s_nonce, sizeof(saved.nonce_tag));  // first 4 bytes
esp_now_send(s_hub_mac, (const uint8_t *)&saved, sizeof(saved));
```

If NVS save fails (all attempts exhausted), do NOT send PAIR_SAVED. The hub times out and the receiver resumes scanning.

#### Receiver ESP8266 changes (`espnow_pair.c`)

Same pattern as ESP32, adapted for ESP8266 code structure. After `device_config_save_pairing()` returns `ESP_OK`, send PAIR_SAVED to hub.

#### Timeout

10 seconds for PAIR_SAVED (receiver NVS write + retries + ESP-NOW send). The ESP32 receiver already retries NVS save 3× with 50ms delays. 10 seconds is generous.

---

## 3. Fix C — Session Nonce in PAIR_ACK

### Problem

`pair_ack_t` carries only `{msg_type, slot}`. A delayed ACK from a previous pairing attempt for the same slot and same receiver MAC could pass validation, since there is no session binding.

### Design

Echo the first 4 bytes of the challenge nonce in PAIR_ACK (and PAIR_SAVED). The nonce is already established — the hub generates it in PAIR_CHALLENGE, the receiver stores it as `s_nonce`. 4 bytes = 1/2^32 collision chance, sufficient for session binding.

#### Updated struct (all 3 codebases)

```c
typedef struct __attribute__((packed)) {
    uint8_t msg_type;       // PAIR_MSG_ACK (0x05)
    uint8_t slot;
    uint8_t nonce_tag[4];   // first 4 bytes of challenge nonce
} pair_ack_t;
```

This changes the struct size from 2 bytes to 6 bytes. Both `sizeof(pair_ack_t)` checks in recv_cb and all `esp_now_send` calls for ACK must account for the new size. The ESP-NOW maximum payload is 250 bytes — 6 bytes is well within limits.

#### Transmitter changes (`espnow_pair_host.c`)

In the slot validation block (after receiving ACK), add nonce_tag validation:

```c
uint8_t ack_nonce_tag[4];
portENTER_CRITICAL(&s_ctx_mux);
ack_slot = s_ctx.rx_ack.slot;
memcpy(ack_nonce_tag, s_ctx.rx_ack.nonce_tag, sizeof(ack_nonce_tag));
portEXIT_CRITICAL(&s_ctx_mux);

if (ack_slot != next_slot) { ... }

if (memcmp(ack_nonce_tag, s_ctx.nonce, sizeof(ack_nonce_tag)) != 0) {
    ESP_LOGW(TAG, "PAIR_ACK nonce_tag mismatch");
    (void)esp_now_del_peer(rx_mac);
    continue;
}
```

Same validation applies to PAIR_SAVED in Fix B.

#### Receiver changes (both platforms)

When building PAIR_ACK, copy the first 4 bytes of the stored nonce:

```c
pair_ack_t ack = {
    .msg_type = PAIR_MSG_ACK,
    .slot = s_offer.slot,
};
memcpy(ack.nonce_tag, s_nonce, sizeof(ack.nonce_tag));
```

The receiver already has the nonce from `PAIR_MSG_CHALLENGE` (stored in `s_nonce`). No additional state needed.

---

## 4. API Reference (Verified)

| API | Source | Signature | Notes |
|-----|--------|-----------|-------|
| `vibrator_set_pulsing` | Both receiver vibrator.h | `esp_err_t vibrator_set_pulsing(VibratorHandler*, bool enabled)` | Idempotent. Replaces `vibrator_toggle_pulsing`. |
| `vibrator_toggle_pulsing` | Both receiver vibrator.h | `esp_err_t vibrator_toggle_pulsing(VibratorHandler*)` | Being replaced. Calls `set_pulsing(!current)` internally. |
| `radio_tx_send` | transmitter radio_supervisor.h | `esp_err_t radio_tx_send(radio_band_t, const uint8_t*, size_t, uint8_t, bool)` | 433M: `rf433_tx_send_bits`. nRF24: `nrf24_tx_send`. |
| `nrf24_tx_send` | transmitter nrf24_transmitter.h | `esp_err_t nrf24_tx_send(nrf24_tx_t*, const uint8_t addr[5])` | Sends hardcoded `TOGGLE_MAGIC = {0xAA, 0x55}`. Needs action param. |
| `rf_trigger_on_frame` | Both receiver rf_trigger.h | ESP32: `void rf_trigger_on_frame(uint32_t value, uint8_t value_bits)` / ESP8266: `void rf_trigger_on_frame(uint32_t decoded, uint8_t decoded_bits, void *ctx)` | 433M callback. uint32_t limits to 32 bits — hence 31+1 scheme. |
| `rf_trigger_on_packet` | receiver-esp32 rf_trigger.h | `void rf_trigger_on_packet(const uint8_t *pkt, size_t pkt_len)` | nRF24 callback. Checks `{0xAA, 0x55}` magic. ESP8266 has no nRF24. |
| `esp_now_send` | ESP-IDF / ESP8266 RTOS SDK | `esp_err_t esp_now_send(const uint8_t *peer_addr, const uint8_t *data, size_t len)` | Verified via Context7 (ESP-IDF v6.0). Max payload 250 bytes. |
| `esp_now_del_peer` | ESP-IDF / ESP8266 RTOS SDK | `esp_err_t esp_now_del_peer(const uint8_t *peer_addr)` | Returns `ESP_OK` or `ESP_ERR_ESPNOW_NOT_FOUND`. Safe for best-effort cleanup. |

---

## 5. Files Changed

| File | Fix | Summary |
|------|-----|---------|
| `transmitter/main/pair/espnow_pair_host.c` | B, C | Add PAIR_SAVED handler + wait; add nonce_tag validation for ACK and SAVED |
| `transmitter/main/dispatch/dispatch.c` | A | Slot dispatch: set/clear MSB based on action for 433M; map action string to `RADIO_ACTION_*` for nRF24. Full-payload dispatch: pass `RADIO_ACTION_START` |
| `transmitter/main/dispatch/radio_supervisor.c` | A | Add `action` parameter, pass to nRF24 path; ignored for 433M |
| `transmitter/main/dispatch/radio_supervisor.h` | A | Add `action` parameter to `radio_tx_send`; define `RADIO_ACTION_START`/`RADIO_ACTION_STOP` |
| `transmitter/main/serial/serial_protocol.c` | A | Pass `RADIO_ACTION_START` to `radio_tx_send` |
| `transmitter/main/nrf24/nrf24_transmitter.c` | A | Accept action param, build `{MAGIC_HI, MAGIC_LO, action}` payload |
| `transmitter/main/nrf24/nrf24_transmitter.h` | A | Add action parameter to `nrf24_tx_send` |
| `transmitter/main/pair/espnow_pair_host.c` | A | `generate_rf_code`: clear MSB for band 0 |
| `receiver-esp32/main/trigger/rf_trigger.c` | A | `rf_trigger_on_frame`: extract MSB, match 31 bits, call `set_pulsing`. `rf_trigger_on_packet`: read action byte, call `set_pulsing`. |
| `receiver-esp32/main/pair/espnow_pair.c` | B, C | Send PAIR_SAVED after NVS save; include nonce_tag in ACK and SAVED |
| `receiver-esp8266/main/trigger/rf_trigger.c` | A | `rf_trigger_on_frame`: extract MSB, match 31 bits, call `set_pulsing` |
| `receiver-esp8266/main/pair/espnow_pair.c` | B, C | Send PAIR_SAVED after NVS save; include nonce_tag in ACK and SAVED |

---

## 6. Backward Compatibility

These are **breaking changes** for hub-paired receivers. After firmware update:
- All existing pairings become invalid (31-bit codes vs old 32-bit codes, new ACK/SAVED struct sizes)
- Receivers must be re-paired with the hub
- The hub's NVS roster should be cleared on firmware update (or handle gracefully via schema version)

Passive/server-managed receivers are completely unaffected — they use the PT2272 channel-patching path which is not modified.

---

## 7. Out of Scope

- **Transmit rejection ACK propagation** (finding 1) — already designed in Local Pairing plan Task 10
- **Safety auto-off timer** — receiver trusts explicit stop command
- **Full-payload dispatch path changes** — only hub-paired (slot) dispatch is modified
- **Backend changes** — backend already sends `action` field in MQTT payload; no changes needed
