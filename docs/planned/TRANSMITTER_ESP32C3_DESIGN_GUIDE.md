# Transmitter Hub Design Guide — ESP32-C3

Authoritative end-to-end plan for the **transmitter hub** across both
ends of the link: the **ESP32-C3 firmware** running on **ESP-IDF v6.0**
and the **backend slice** that registers, elects, and dispatches to it.
The hub ships **both radios in a single image**: **433 MHz ASK/OOK**
transmit *and* **nRF24L01 / nRF24L01+** 2.4 GHz transmit. Band
selection is a per-dispatch decision made on each inbound
`cmd/transmit`, not a Kconfig gate.

This guide is the **only** transmitter contract going forward — the
older backend-only prep was merged into §H and removed. Section H is
the backend implementation plan; sections A–G describe the firmware;
sections I–K cover firmware build config, audit, and self-
verification.

This guide stands on top of:

- **Receiver design guide** — `docs/planned/RECEIVER_ESP32C3_DESIGN_GUIDE.md`.
  The radio peer at the other end of every dispatch. The transmitter
  reuses the receiver's Wi-Fi (§G.3), MQTT framing (§G.4), HTTP
  provisioning (§G.5), nRF24 register knowledge (§G.7), and mbedTLS
  identity (§G.10) verbatim; receiver-side material is referenced, not
  duplicated.
- **Device Integration Implementation Plan** —
  `docs/planned/Device Integration Implementation Plan.md`. Owns
  the unified `device` domain end-to-end (receivers **and** transmitter
  hubs): schema, enrollment tokens, signing-key plumbing, RF-code
  shape, lifecycle, queue-side dispatch broadcaster, transmitter
  bootstrap intake, heartbeat ingestion, active-hub election, and the
  dispatch consumer. §H below originated the backend slice for the
  transmitter hub and has since been folded into that plan; treat the
  implementation plan as the source of truth and §H here as a
  duplicate kept for firmware-side context only.

Cross-refs:

- SDK reference: `docs/walkthrough/ESP_IDF_SDK_REFERENCE.md` (the only
  checked-in indexed ESP-IDF reference in this workspace as of this
  audit; no `transmitter/` or `receiver-esp32/` copy is present)
- SDK root: `/home/thomas/esp/v6.0/esp-idf` (v6.0.0)
- ESP-MQTT headers (managed component used by both firmwares):
  `transmitter/managed_components/espressif__mqtt/include/mqtt_client.h`
- Datasheets:
  - `docs/walkthrough/nRF24L01P_PS_v1.0.txt` — nRF24L01+ PS v1.0
  - `docs/walkthrough/nRF24L01_Product_Specification_v2_0-9199.txt` — nRF24L01 PS v2.0 (legacy)

This document describes the **target end state**. The current
`transmitter/` repo snapshot is a fresh ESP-IDF skeleton: `main.c` is
an empty `app_main`, and only an RC-Switch-style 433 MHz pulse engine
exists under `main/rf/` (`rf_common.h`, `rf_data.c`, `rf_timer.c`,
`rf_transmitter.c`).

**Last updated:** 2026-04-27 (audit fixes: SDK refs, replay order, 433 completion, nRF24 channel bounds)

---

## A. System Overview

Scope:

1. SoftAP provisioning and local recovery — same wire format as the
   receiver (Receiver §F.4).
2. Backend registration plus challenge-response activation — same
   `activate-v1|...` canonical, different bootstrap topic namespace
   (`transmitter/...`, §E.1).
3. Post-activation operation: subscribe to dispatch commands, broadcast
   the requested RF code on the requested band, and publish a periodic
   heartbeat so the per-store hub election (§H.7.4) sees the unit live.
4. Remote deactivation: `suspend`, `resume`, `decommission` — same
   `deact-v1|...` canonical as the receiver, applied to the hub's own
   topic family.

Key behavioural difference from the receiver:

- A hub does **nothing autonomously**. It owns no `rf_code`, runs no
  matcher, drives no vibrator. Every RF emission is the consequence of
  exactly one signed `cmd/transmit` (§H.1).
- A hub keeps the backend's view of liveness explicit. It emits a
  10 s heartbeat (§E.5 / §H.7.3); silence is fail-safe — missing
  heartbeats drop the hub from the candidate pool, and the next
  dispatch elects a peer.

### A.1 Device lifecycle

```
boot ─► load NVS ─► wifi config missing?
                     ├─ yes ─► SoftAP + HTTP provisioning ─► POST config + token ─► reboot
                     └─ no  ─► STA Wi-Fi ─► MQTTS ─► ensure device keypair
                                      │
                                      ▼
                          public_id + op_state already stored?
                           ├─ yes ─► subscribe transmitter/hub/{public_id}/cmd/#
                           │        ─► start heartbeat task ─► dispatch op_state
                           └─ no  ─► enroll_token present?
                                     ├─ no  ─► SoftAP recovery
                                     └─ yes ─► subscribe transmitter/bootstrap/+
                                               ─► publish transmitter/bootstrap/register
                                                  with kind=TRANSMITTER_HUB
                                               │
                                               ▼
                                     pending → narrow to bootstrap/{cid}
                                     rejected ─► erase enroll_token ─► SoftAP recovery
                                     challenge ─► sign response ─► wait result
                                     result    ─► persist public_id, device_name,
                                                 op_state=ACTIVE,
                                                 erase enroll_token,
                                                 restart MQTT with client_id=public_id
                                               │
                                               ▼
                                     subscribe transmitter/hub/{public_id}/cmd/# ─►
                                     start heartbeat ─► idle awaiting cmd/transmit
```

The post-activation status goes **directly to `ACTIVE`** (§H.7.2).
Hubs have no `PENDING_RF_CODE` waiting state and no per-device RF
code; the backend never publishes `cmd/rf_code` to a hub topic.

### A.2 Persistent state

One persisted `op_state` byte drives operational behaviour. The hub
schema is intentionally a strict subset of the receiver schema — no
`PENDING_RF_CODE`:

| `op_state`       | Heartbeat | Dispatch | Meaning                                 |
| ---------------- | --------- | -------- | --------------------------------------- |
| `ACTIVE`         | published | accepted | Normal operating state after activation |
| `SUSPENDED`      | paused    | dropped  | Backend requested temporary off         |
| `DECOMMISSIONED` | paused    | dropped  | Backend requested permanent off         |

`SUSPENDED` and `DECOMMISSIONED` both stop heartbeat publication so
the election service (§H.7.4) drops the hub immediately, without
waiting for the 30 s liveness TTL. The MQTT session itself stays
connected so the hub can still receive a signed `resume` (from
`SUSPENDED`) and so the Redis lifecycle ack is durable.

Derived, not stored:

- `UNPROVISIONED` = `wifi_ssid` key missing from NVS.
- `PENDING_ACTIVATION` = `wifi_ssid` present, `op_state` missing,
  `enroll_token` present.
- `RECOVERY_REQUIRED` = `wifi_ssid` present, `op_state` missing,
  `enroll_token` missing.

There is no `rx_type` on this device — the hub always carries both
radios — so the receiver's compiled-vs-stored mismatch path
(Receiver §A.2) does not exist here.

### A.3 Reprovision

Same model as the receiver (Receiver §A.3): erase `device_cfg`, keep
`identity`. The EC keypair persists, so the backend can recognise the
physical hub across reactivations; the next activation mints a fresh
`hub-XXXXX` `public_id`, orphaning any retained `cmd/deact` on the
old topic.

---

## B. Dual Radio Model

### B.1 Hub roles vs receiver families

The transmitter hub serves **all** receiver kinds in its store:

| Backend `device.kind` (receiver) | Reached by hub via | Source of plaintext     |
| -------------------------------- | ------------------ | ----------------------- |
| `RECEIVER_433M`                  | 433 MHz ASK/OOK    | Backend `SecureRandom`  |
| `RECEIVER_433M_PASSIVE` (PT2272) | 433 MHz ASK/OOK    | Admin-set preset        |
| `RECEIVER_2_4G`                  | nRF24L01 / +       | Backend `SecureRandom`  |

A single store holds at most one elected `ACTIVE` hub at a time
(§H.1). That hub must reach any receiver kind in its store, hence
dual-radio in firmware.

### B.2 Band selection from the dispatch envelope

`cmd/transmit` carries `rf_code_hex`, `rf_code_bits`, and an explicit
`band`. Per the Device Integration Plan §2.4 validation table:

- `RECEIVER_433M` and `RECEIVER_433M_PASSIVE`: `rf_code_bits ∈ [1, 32]`,
  `hex_len == 2 * ceil(bits / 8)`. The bytes are the application-layer
  match value the receiver MCU compares against its stored rf_code.
- `RECEIVER_2_4G`: `rf_code_bits == 40`, `hex_len == 10`. The bytes
  are the destination receiver's nRF24 `RX_ADDR_P1` — silicon-level
  identity, not an application-layer match value.

The bit-width rules are now disjoint (40 only appears in 2.4G), so
`band` is no longer load-bearing for disambiguation. It stays on the
wire because (1) it lets the hub firmware route to the right radio
without back-deriving from `bits`, (2) it makes the canonical string
self-describing for audit, and (3) it leaves room for future widths
without re-cutting the wire format. Firmware-side validation of the
field lives in §E.3; the §H.4 contract delta defines who populates
it on the backend.

### B.3 Transmit semantics

- **433 MHz**: rebuild the integer plaintext from the hex (big-endian,
  `byte_len = ceil(bits/8)`, only the bottom `bits` LSBs significant —
  Device Integration Plan §3.1 schema comment), encode as a pulse train with the
  active RC-Switch-family protocol, repeat `TX_REPEAT_COUNT` times,
  release GPIO (§G.6). The rf_code bytes are an application-layer
  match value — the receiver's MCU-side matcher compares the decoded
  pulse-train value against its stored rf_code.
- **2.4 GHz**: the dispatch's `rf_code_hex` is the destination
  receiver's 5-byte nRF24 address (Device Integration Plan §2.4 — `bits == 40`,
  `byte_len == 5`). The hub writes those 5 bytes to **both** `TX_ADDR`
  and `RX_ADDR_P0` (auto-ACK requires `RX_ADDR_P0 == TX_ADDR`),
  loads the firmware-defined fixed `TOGGLE_MAGIC = {0xAA, 0x55}`
  payload into the TX FIFO with `W_TX_PAYLOAD`, pulses `CE` ≥ 10 µs
  (datasheet §6.1.7 Thce), waits for `TX_DS` or `MAX_RT` via IRQ, and
  returns the chip to Power-Down (§G.7). The address is per-dispatch;
  no nRF24 address ever lives in hub firmware or hub Kconfig.
- Stop semantics: a `DEVICE_STOP_REQUESTED` event becomes another
  `cmd/transmit` carrying the same `rf_code_hex`. The receiver's
  matcher (Receiver §G.8) toggles vibrator state on every qualifying
  match — for 433M that means the same bitwise comparison; for 2.4G
  that means another address-matched + magic-validated frame. The
  hub never emits a distinct "stop" frame and does not vary the
  payload by request kind.

The latency budget is band-specific. For nRF24, the firmware target is
**< 250 ms from broker delivery to RF off-air**: signature verify
(§G.10) is 70–100 ms on C3 @ 160 MHz, and the radio transaction is
single-digit milliseconds plus the Power-Down wake cost. For 433 MHz,
off-air time is dominated by `TX_REPEAT_COUNT` and the selected
RC-Switch-family protocol; with protocol 1, 32 bits, and 10 repeats,
the pulse train is roughly 560 ms before signature-verify cost. The
backend SLA must therefore treat 433 MHz completion as repeat-count
dependent rather than a hard 250 ms property.

---

## C. Target Module Layout

```
main/
├── main.c                   orchestrator
├── config/
│   └── device_config.[ch]   NVS I/O, schema, reprovision (TRANSMITTER variant)
├── network/
│   ├── wifi.[ch]            STA + SoftAP + WPA3-Compat (reuses receiver §G.3)
│   ├── mqtt.[ch]            mqtts client + topic router (reuses receiver §G.4)
│   ├── heartbeat.[ch]       periodic 10 s publisher (§G.9)
│   └── certs/mqtt_ca.pem    broker CA (embedded via EMBED_TXTFILES)
├── provision/
│   ├── http_server.[ch]
│   └── index.html[.gz]      embedded UI (gzip-embedded asset)
├── security/
│   └── device_identity.[ch] device EC P-256 + pinned backend pubkey (reuses receiver §G.10)
├── dispatch/
│   ├── dispatch.[ch]        cmd/transmit handler: verify, route, ack
│   └── radio_supervisor.[ch] start/suspend/resume/deinit per radio (§G.2)
├── rf/                      existing 433 MHz pulse engine
│   ├── rf_common.h          add bits_to_pulses + rf433_tx_send
│   ├── rf_data.c            keep RC-Switch protocols; add bits-based encoder
│   ├── rf_timer.c           unchanged
│   └── rf_transmitter.c     unchanged
└── nrf24/
    ├── nrf24_regs.h         reused from receiver §G.7.2 with one PTX-only
    │                        addition: `NRF_DYNPD_P0 (1U << 0)` for the
    │                        ACK-receiving pipe (§G.7).
    ├── nrf24_transmitter.h  PTX-shaped public C API (§G.7)
    └── nrf24_transmitter.c  SPI driver in PTX mode (§G.7)
```

The receiver's `trigger/`, `rf/rf_receiver.c`, `vibrator/`, and the
matcher (`trigger/rf_trigger.c`) are absent — none of those modules
have a transmitter analogue. `dispatch/` is new and replaces the
receiver's `trigger/`.

Coupling rule:

- `dispatch/` calls `rf/` (433 path) and `nrf24/` (2.4 path) through
  `dispatch/radio_supervisor.[ch]`. Neither radio module knows MQTT.
- `network/heartbeat` calls `network/mqtt` only.
- `network/mqtt` calls `dispatch/` on `cmd/transmit` and a lifecycle
  service on `cmd/deact`. It owns no radio knowledge.

### C.1 Pin map (default Kconfig)

ESP32-C3 exposes 22 GPIO numbers (0–21), but board/module wiring and
boot strapping still matter. Avoid GPIO18/19 when USB-Serial-JTAG is
needed. Both radios are on the board simultaneously, so the pin map is
the union of the receiver's two radio configurations, with one swap to
free a GPIO:

| Signal             | Default GPIO | Notes                                 |
| ------------------ | ------------ | ------------------------------------- |
| `RF433_TX_GPIO`    | 4            | push-pull out (was the receiver's RX) |
| `NRF24_SPI_HOST`   | `SPI2_HOST`  | only general-purpose SPI host on C3   |
| `NRF24_SCK`        | 6            | SPI clock                             |
| `NRF24_MOSI`       | 7            | SPI MOSI                              |
| `NRF24_MISO`       | 2            | SPI MISO                              |
| `NRF24_CS`         | 5            | active-low CS                         |
| `NRF24_CE`         | 3            | chip-enable                           |
| `NRF24_IRQ`        | 1            | active-low TX_DS / MAX_RT IRQ         |

Strapping-pin and USB-pin advisories from Receiver §C.1 still apply.
GPIO2 is acceptable for SPI MISO only if the board keeps the boot
strap at its required level and the nRF24 MISO line is high-impedance
while CSN stays high at reset. No vibrator GPIO; the hub has no local
actuator.

### C.2 Radio gate

There is no Kconfig `RADIO_VARIANT` choice. Both radios are in every
build. A future "radio-down" Kconfig may exclude one driver to save
flash on a single-band board population, but v1 ships dual.

---

## D. NVS Schema

Two namespaces, same NVS rules and 15-char key limit as the receiver.

**Namespace `identity`** — device EC keypair; survives reprovision.
Same shape as Receiver §D:

| Key       | Type   | Notes                    |
| --------- | ------ | ------------------------ |
| `pubkey`  | `blob` | DER EC P-256 public key  |
| `privkey` | `blob` | DER EC P-256 private key |

**Namespace `device_cfg`** — runtime config plus activation state;
wiped on reprovision. The hub schema is **smaller than the receiver's**
because there is no per-device RF code stored on the hub:

| Key             | Type  | Written by         | Notes                                                                |
| --------------- | ----- | ------------------ | -------------------------------------------------------------------- |
| `schema_ver`    | `u8`  | provisioning       | `1`                                                                  |
| `wifi_ssid`     | `str` | provisioning       | presence = provisioned                                               |
| `wifi_pwd`      | `str` | provisioning       | station password                                                     |
| `mqtt_uri`      | `str` | provisioning       | must be `mqtts://...`                                                |
| `mqtt_user`     | `str` | provisioning       | shared broker credential                                             |
| `mqtt_pwd`      | `str` | provisioning       | shared broker credential                                             |
| `enroll_token`  | `str` | provisioning       | erased after successful activation or terminal bootstrap failure     |
| `public_id`     | `str` | activation         | backend-minted `hub-XXXXX`                                           |
| `device_name`   | `str` | activation         | backend-assigned human label                                         |
| `op_state`      | `u8`  | activation + MQTT  | `ACTIVE`, `SUSPENDED`, `DECOMMISSIONED`                              |
| `last_deact_id` | `str` | `cmd/deact` MQTT   | most recent applied lifecycle `command_id` for replay suppression    |

Removed compared to Receiver §D: `rx_type`, `rf_code`, `rf_code_bits`,
`rf_code_ver`. The hub never stores a code.

`last_deact_id` keeps the same semantics as on the receiver: rules
from Receiver §F.2 ("If `command_id == last_deact_id`, republish the
prior ack and skip side effects") apply unchanged.

**Dispatch-replay rule.** The hub does **not** persist `dispatch_id`s
to NVS — they are bursty and ephemeral. Replay suppression for
`cmd/transmit` runs in RAM only, with a small ring buffer of the last
N (default 8) signed commands that reached a terminal ack outcome
(`applied` or `rejected`). The ring stores the prior ack status/reason
and never re-emits RF on a duplicate. A reboot wipes
the ring; this is acceptable because broker reconnection re-delivers
only un-acked QoS 1 messages and `cmd/transmit` is **not retained**
(§E.1), so a stale broadcast cannot replay across a reboot.

Encryption-at-rest, atomic-commit, and "never log secrets" rules from
Receiver §D apply verbatim.

---

## E. MQTT Contract (firmware view)

This section pins what the firmware sees on the wire. The
backend-side enforcement, subscription set, and publishers live in
§H.2 / §H.3.

### E.1 Topics

| Topic                                       | Direction        | QoS | Retained | Notes                                    |
| ------------------------------------------- | ---------------- | --- | -------- | ---------------------------------------- |
| `transmitter/bootstrap/register`            | hub → backend    | 1   | no       | registration payload                     |
| `transmitter/bootstrap/{challenge_id}`      | bidirectional    | 1   | no       | `pending`, `challenge`, `rejected`, `result`, `response` envelopes |
| `transmitter/hub/{public_id}/cmd/transmit`  | backend → hub    | 1   | **no**   | signed dispatch                          |
| `transmitter/hub/{public_id}/cmd/deact`     | backend → hub    | 1   | **yes**  | signed lifecycle                         |
| `transmitter/hub/{public_id}/heartbeat`     | hub → backend    | 0   | no       | 10 s cadence                             |
| `transmitter/hub/{public_id}/ack`           | hub → backend    | 1   | no       | single ack stream, `ack_for` discriminator |

`cmd/transmit` is intentionally **not retained**: the broker must
never replay a stale broadcast on hub reconnect. `cmd/deact` is
retained so a hub coming back online learns its current lifecycle
state before processing fresh transmits.

### E.2 Security

Same model as Receiver §E.2:

- All MQTT traffic is `mqtts://` only; provisioning rejects non-TLS
  URIs and firmware refuses to connect if `mqtt_uri` does not start
  with `mqtts://`.
- The broker certificate must chain to the embedded CA PEM.
- v1 uses one shared MQTT credential for all hubs; isolation rests on
  TLS, unguessable `public_id`s, and signed operational commands.
- Every inbound payload is validated for `schema_version`, shape, and
  mandatory fields before side effects.

### E.3 Payloads

All wire payloads carry `"schema_version": 1`.

**Provisioning** (`POST /api/provision`): identical to Receiver §E.3.

**Registration** (`transmitter/bootstrap/register`):

```json
{
  "schema_version": 1,
  "hardware_model": "ESP32-C3",
  "kind": "TRANSMITTER_HUB",
  "firmware_version": "dev",
  "public_key_b64": "BASE64_DER_PUBLIC_KEY",
  "enrollment_token": "<opaque string>",
  "registration_nonce": "hG7aK2pQcR"
}
```

The discriminator field is `"kind"`, set to the literal
`TRANSMITTER_HUB`. The receiver-plan generalisation in §H.7.1 routes
this through the same `DeviceRegistrationService` with a
`DeviceFamily = TRANSMITTER` discriminator inferred from the topic;
the firmware does not need to know about that.

**Bootstrap envelopes** (`pending`, `challenge`, `rejected`,
`response`, `result`): byte-identical to Receiver §E.3, including the
canonical `activate-v1|...` (Receiver §2.3, reused unchanged here —
see §H.3). Only the topic namespace changes.

**Dispatch** (`transmitter/hub/{public_id}/cmd/transmit`, **not
retained**, signed by the backend with the pinned EC key):

```json
{
  "schema_version": 1,
  "dispatch_id": "f6e60f38-2f1f-4f22-9ca0-71a55a2d65bb",
  "receiver_public_id": "rcv-7Q4KZ",
  "band": "433M",
  "rf_code_hex": "C35A5A5A",
  "rf_code_bits": 32,
  "issued_at": "2026-04-25T10:00:00Z",
  "signature_b64": "BACKEND_ECDSA_SIGNATURE"
}
```

Canonical string for signing (the §H.4 amendment):

```text
transmit-v1|<hub_public_id>|<dispatch_id>|<receiver_public_id>|<band>|<rf_code_hex>|<rf_code_bits>|<issued_at>
```

Validation order on the hub:

1. JSON shape, `schema_version == 1`, all required fields present.
2. `band ∈ { "433M", "2_4G" }`.
3. Width per band (mirrors Device Integration Plan §2.4 exactly):
   - `433M`: `bits ∈ [1, 32]`, `hex_len == 2 * ceil(bits/8)`.
   - `2_4G`: `bits == 40`, `hex_len == 10` — the bytes are the
     destination receiver's `RX_ADDR_P1`. The hub does not own a
     fallback width.
4. Rebuild the canonical string byte-for-byte and verify
   `signature_b64` against the pinned backend public key
   (`CONFIG_TRANSMITTER_BACKEND_PUBKEY_B64`).
5. `dispatch_id` not in the in-RAM replay ring (§D); if it is, re-ack
   the prior terminal outcome and skip RF emission. The ring lookup is
   read-only; only a signed command that reaches a terminal ack outcome
   records a new `dispatch_id`.
6. Reject with `status = rejected` on shape error, signature failure,
   or width mismatch — `ack_for = "transmit"`, no RF emission.

**Deactivation** (`transmitter/hub/{public_id}/cmd/deact`, retained):
byte-identical to Receiver §E.3 — same `deact-v1|public_id|command_id|action|issued_at`
canonical, same `action ∈ {suspend, resume, decommission}`. The
firmware applies the action to its own persistent state; there is no
RF impact.

**Heartbeat** (`transmitter/hub/{public_id}/heartbeat`, QoS 0, not
retained):

```json
{ "schema_version": 1, "issued_at": "2026-04-25T10:00:10Z" }
```

QoS 0 because the heartbeat is a liveness signal, not a fact: the
backend's 30 s TTL absorbs single missed packets and any longer
silence already means the hub is gone. QoS 1 here would amplify
broker write load without making liveness more accurate.

**Ack stream** (`transmitter/hub/{public_id}/ack`):

```json
{ "schema_version": 1, "ack_for": "transmit", "dispatch_id": "...", "status": "applied",   "applied_at": "2026-04-25T10:00:00.230Z" }
{ "schema_version": 1, "ack_for": "transmit", "dispatch_id": "...", "status": "rejected",  "reason": "signature_failed" }
{ "schema_version": 1, "ack_for": "transmit", "dispatch_id": "...", "status": "unchanged", "applied_at": "2026-04-25T10:00:00.230Z" }
{ "schema_version": 1, "ack_for": "deact",    "command_id":  "...", "action": "suspend", "status": "ok" }
```

Status values:

- `ack_for = "transmit"`: `applied`, `unchanged` (replay of a prior
  applied command), or `rejected`. Replays of prior rejects repeat the
  original reject reason. Acks never echo `rf_code_hex`.
- `ack_for = "deact"`: `ok`, `ignored`, or `rejected` — same shape as
  the receiver.

### E.4 Cryptography

Same as Receiver §E.4: device EC P-256 keypair on first boot, signed
with `SHA256withECDSA`; backend pins its EC P-256 public half via
`CONFIG_TRANSMITTER_BACKEND_PUBKEY_B64`. §H.7.0 confirms one signing
key covers both fleets, so the dev-time PEM workflow from Receiver
§I.2 is reused unchanged.

### E.5 Heartbeat cadence and backoff

- Cadence: 10 s while `op_state == ACTIVE` (matches §H.6
  `heartbeat-interval-seconds`).
- Stop conditions: `SUSPENDED`, `DECOMMISSIONED`, MQTT disconnected.
- On reconnect: resume publishing; do not "catch up". The backend
  computes liveness from the *latest* heartbeat, not a count.
- Jitter: ±0.5 s uniform random on each tick to avoid all hubs in a
  store coordinating their broker writes.

---

## F. Boot Orchestration and Runtime Cases

### F.1 `app_main` orchestration

```
1. nvs_init_or_recover() + esp_netif_init() + esp_event_loop_create_default()
2. device_config_load() -> in-RAM config
3. if no wifi_ssid:
     wifi_start_softap() + http_server_start()
     wait PROV_DONE -> esp_restart()
4. wifi_start_sta()  [blocks on IP_EVENT_STA_GOT_IP]
5. device_identity_ensure()  [generate/load EC keypair]
6. radio_supervisor_init()    [§G.2: bring up both radios into idle]
7. mqtt_start(&g_cfg)
   - pre-activation client_id = ESP32_%CHIPID%
   - post-activation client_id = public_id
8. if public_id absent:
     if enroll_token missing:
       wifi_start_softap() + http_server_start()
       wait new provision, retry, or factory reset
     else:
       generate registration_nonce
       subscribe transmitter/bootstrap/+
       publish transmitter/bootstrap/register with kind=TRANSMITTER_HUB
       on matching pending: narrow to transmitter/bootstrap/{challenge_id}
       on challenge: sign and publish response
       on result:
         persist public_id, device_name, op_state=ACTIVE
         erase enroll_token
         restart MQTT with client_id=public_id
9. after IP_EVENT_STA_GOT_IP:
     esp_wifi_set_ps(WIFI_PS_MIN_MODEM)
10. subscribe transmitter/hub/{public_id}/cmd/#
11. dispatch on op_state:
      ACTIVE          -> heartbeat_start()
      SUSPENDED       -> heartbeat_stop(); RF idle
      DECOMMISSIONED  -> heartbeat_stop(); RF idle
12. idle loop
```

Step 6 brings both radios up to **idle** (433 GPIO low, nRF24 powered
down with `PWR_UP=0`). Each `cmd/transmit` then briefly raises the
chosen radio out of idle, emits, returns it to idle. This avoids the
receiver's continuous-RX power profile and keeps the chips cool when
no dispatches are in flight.

### F.2 `cmd/transmit` handling

1. Parse, width-check, and signature-verify per §E.3.
2. If the `dispatch_id` is already in the replay ring, re-ack the
   prior terminal outcome and stop; do not emit RF.
3. If `op_state != ACTIVE`, remember the state-based reject in the
   replay ring, ack `rejected`, and stop.
4. Take `radio_mtx`. Two transmits cannot interleave.
5. Route by `band`:
   - `"433M"` → `rf433_tx_send_bits(plaintext, byte_len, bits)` (§G.6).
   - `"2_4G"` → `nrf24_tx_send(addr)` (§G.7) — `addr` is the 5
     plaintext bytes parsed from `rf_code_hex`; the send path writes
     them to `TX_ADDR` + `RX_ADDR_P0` and emits the fixed
     `TOGGLE_MAGIC` payload.
6. Release `radio_mtx`.
7. Insert `dispatch_id` plus the terminal ack outcome into the replay
   ring; on overflow, evict the oldest entry. This includes
   `tx_failed`, so QoS duplicates cannot double-toggle a receiver after
   an ambiguous ACK-loss case.
8. Publish the terminal ack. For success, use `status = "applied"` and
   the time-of-day right after the radio reports completion (`gptimer`
   fired all repeats / `TX_DS` IRQ). For failure, use `rejected` with
   the reason selected above.

Failures inside step 5 (`ESP_FAIL` from the radio) ack `rejected`
with `reason = "tx_failed"` and surface a single warning log. The
backend treats repeated `tx_failed` like silence for election
purposes — operations triage it via the device-detail page.

### F.3 `cmd/deact` handling

Mirrors Receiver §F.2 deactivation block exactly; the only differences
are the topic namespace and the side-effects on the heartbeat task:

- `suspend` → `op_state = SUSPENDED`, `heartbeat_stop()`,
  persist `last_deact_id`, ack `ok`.
- `resume` → `op_state = ACTIVE`, `heartbeat_start()`, persist
  `last_deact_id`, ack `ok`.
- `decommission` → `op_state = DECOMMISSIONED`, `heartbeat_stop()`,
  persist `last_deact_id`, ack `ok`.
- `resume` after `DECOMMISSIONED` → ack `ignored`, no state change.
- `command_id == last_deact_id` → ack the prior result, no state
  change.

All NVS commits happen **before** the ack publish (Receiver §F.2
invariant kept).

### F.4 Heartbeat task

A FreeRTOS task started by `heartbeat_start`, period 10 000 ms ± 500
ms jitter. It reads `g_cfg.public_id`, formats the JSON in §E.3, and
publishes via `esp_mqtt_client_publish(..., qos=0, retain=0)`. On
publish error (e.g. queue full because MQTT is reconnecting) the tick
is dropped silently — the next tick will retry. `heartbeat_stop`
deletes the task and closes the publish path.

### F.5 Failure and recovery cases

Cases shared with the receiver — Wi-Fi repeatedly fails, pending
envelope received, rejected envelope received, challenge or result
timeout — apply unchanged from Receiver §F.3. The hub's transmit
path adds two:

**`cmd/transmit` while suspended or decommissioned**

- The hub MUST NOT emit RF. It still validates + signature-verifies
  the payload (cheap, 70–100 ms ECDSA verify) so it can ack with
  `status = "rejected"`, `reason = "device_suspended"` or
  `"device_decommissioned"`. This makes admin-side UX honest: a
  command issued to a suspended hub is not silently swallowed.

**Radio bringup fails at boot (§F.1 step 6)**

- 433 path failure (gptimer create / GPIO config) is fatal — log and
  reboot; the loop in §F.1 will retry on the next boot. ESP-IDF
  gptimer creation failures are typically misconfiguration, so
  rebooting is the right loop.
- nRF24 path failure (`SETUP_AW = 0` on probe → no chip on bus, or
  `nrf_probe_chip` returns `NRF_CHIP_UNKNOWN`) is **not** fatal: the
  hub continues in 433-only mode and acks `rejected` with
  `reason = "no_2_4g_radio"` for any `band=2_4G` dispatch. Logged
  once at `ERROR`. (Surfacing this into the heartbeat is deferred —
  see §J.4.)

### F.6 SoftAP and local provisioning

Reused from Receiver §F.4 verbatim. Only renames:

- SSID prefix: `TRANSMITTER-SETUP-<last_3_mac_bytes_hex>`
- Kconfig prefix: `CONFIG_TRANSMITTER_*` instead of `CONFIG_RECEIVER_*`

Endpoints, payload shapes (`POST /api/provision`, `/api/retry`,
`/api/reset`), and SoftAP security (WPA2-PSK with WPA3-compat) are
unchanged.

### F.7 Power management

Same rules as Receiver §F.5: `WIFI_PS_MIN_MODEM` only after
`IP_EVENT_STA_GOT_IP`, MQTT keepalive 120 s, no deep sleep, never
`esp_wifi_stop()` in `ACTIVE`/`SUSPENDED`/`DECOMMISSIONED`.

The hub adds one rule: nRF24 stays in `Power-Down` (`PWR_UP = 0`)
between dispatches. Each `band=2_4G` transmit walks the chip
`Power-Down → Standby-I → TX → Standby-I → Power-Down`, paying the
`Tpd2stby` (≤ 4.5 ms) and `Tstby2a` (130 µs) startup costs once per
dispatch. Dispatches are single-digit per minute in the worst-case
queue surge, so these costs are negligible against the dispatch
budget in §B.3.

---

## G. ESP-IDF v6 Code Guide

Snippets elide error handling. ESP-IDF API surfaces match those used
by the receiver firmware in `receiver-esp32/main/`, validated against
`docs/walkthrough/ESP_IDF_SDK_REFERENCE.md` (and Context7 for upstream
docs that postdate the in-repo snapshot).

### G.1 NVS

Same as Receiver §G.1: `nvs_open("device_cfg", NVS_READWRITE, &h)`,
`nvs_set_str / set_u8 / commit`. First boot: `nvs_flash_init()` →
on `ESP_ERR_NVS_NO_FREE_PAGES` / `_NEW_VERSION_FOUND` →
`nvs_flash_erase()` + retry. The final firmware config must enable
`CONFIG_NVS_ENCRYPTION=y` and
`CONFIG_NVS_SEC_KEY_PROTECT_USING_HMAC=y` through `sdkconfig.defaults`
or `menuconfig`; this checkout does not currently include a
`transmitter/sdkconfig` file.

### G.2 Radio supervisor — dual-radio dispatch

Unlike the receiver's Kconfig-gated supervisor (Receiver §G.2), the
hub supervisor is a **switch on `band`**, not a `#if`:

```c
// dispatch/radio_supervisor.h
#pragma once
#include "esp_err.h"
#include <stddef.h>
#include <stdint.h>

esp_err_t radio_supervisor_init(void);          // bring up both radios into idle
esp_err_t radio_supervisor_deinit(void);

// Single transmit point. The bytes parsed from rf_code_hex play
// different roles by band:
//   RADIO_BAND_433M : `payload` = encoded RC-Switch value (1..32 bits)
//                      that the receiver MCU matches at app layer.
//                      `bits` is the significant width.
//   RADIO_BAND_2_4G : `payload` = the destination receiver's 5-byte
//                      nRF24 address (Device Integration Plan §2.4 fixes
//                      bits == 40, byte_len == 5). `bits` is 40.
typedef enum { RADIO_BAND_433M = 0, RADIO_BAND_2_4G = 1 } radio_band_t;

esp_err_t radio_tx_send(radio_band_t band,
                        const uint8_t *payload,
                        size_t byte_len,
                        uint8_t bits);
```

```c
// dispatch/radio_supervisor.c (sketch)
static SemaphoreHandle_t s_radio_mtx;
static RFHandler         s_rf;     // 433 path, see rf/rf_common.h
static nrf24_tx_t        s_nrf;    // 2.4 path, see nrf24/nrf24_transmitter.h
static bool              s_have_nrf;

esp_err_t radio_supervisor_init(void) {
    s_radio_mtx = xSemaphoreCreateMutex();
    ESP_ERROR_CHECK(rf433_tx_init(CONFIG_TRANSMITTER_RF433_TX_GPIO,
                                  TX_REPEAT_COUNT, &proto[PROTO_ID - 1], &s_rf));
    s_have_nrf = (nrf24_tx_init(&s_nrf) == ESP_OK);
    return ESP_OK;
}

esp_err_t radio_tx_send(radio_band_t band, const uint8_t *p, size_t n, uint8_t bits) {
    xSemaphoreTake(s_radio_mtx, portMAX_DELAY);
    esp_err_t r;
    if (band == RADIO_BAND_433M) {
        r = rf433_tx_send_bits(&s_rf, p, n, bits);
    } else {
        // Width must be 40 / 5 bytes (Device Integration Plan §2.4) — caller is
        // expected to have pre-validated this against the §E.3 width
        // table. nrf24_tx_send writes the address to TX_ADDR +
        // RX_ADDR_P0 internally and sends the fixed TOGGLE_MAGIC.
        if (n != 5 || bits != 40) {
            r = ESP_ERR_INVALID_ARG;
        } else if (!s_have_nrf) {
            r = ESP_ERR_NOT_SUPPORTED;               // → "no_2_4g_radio"
        } else {
            r = nrf24_tx_send(&s_nrf, p);
        }
    }
    xSemaphoreGive(s_radio_mtx);
    return r;
}
```

The mutex serialises the two external RF transmit paths so 433 MHz and
nRF24 emissions never overlap and peak current stays bounded. It does
not coordinate with the ESP32-C3 Wi-Fi radio; nRF24/Wi-Fi coexistence
is handled by channel choice, short airtime, and normal retry behavior.

### G.3 Wi-Fi

Reuse Receiver §G.3 verbatim (STA + SoftAP + WPA3-Compatible Mode).
Only renames: `CONFIG_TRANSMITTER_AP_*`, SSID prefix
`TRANSMITTER-SETUP-`. The functions `wifi_start_sta`,
`wifi_start_softap`, the event handler split (`WIFI_EVENT` vs
`IP_EVENT`), the `failure_retry_cnt` + `WIFI_ALL_CHANNEL_SCAN`
pairing, the `pmf_cfg.capable` deprecation, and the post-STA-GOT_IP
power-save rule all carry over. No changes.

### G.4 MQTT

Reuse Receiver §G.4 verbatim, including the post-activation MQTT
restart pattern (Receiver §G.4.1) for switching
`client_id = ESP32_%CHIPID%` → `client_id = public_id`. Two
firmware-side changes only:

1. The topic-router dispatch list points at three handlers instead of
   two:
   - `transmitter/bootstrap/+` → `bootstrap_listener` (drops backend-
     owned envelope types — same self-echo rule as the receiver).
   - `transmitter/hub/{public_id}/cmd/transmit` → `dispatch_handler`
     (§G.8).
   - `transmitter/hub/{public_id}/cmd/deact` → `lifecycle_handler`
     (mirrors Receiver §F.2 deact path; namespace renamed).
2. The post-activation `subscribe_current_phase()` subscribes to
   `transmitter/hub/{public_id}/cmd/#` and starts the heartbeat task
   (§G.9). On disconnect, the heartbeat task drops ticks but does not
   exit; on reconnect, ticks resume.

`MQTT_EVENT_DATA` fragment handling (`e->total_data_len > e->data_len`
buffer-and-reassemble) is mandatory and applies unchanged.

### G.5 HTTP provisioning

Reuse Receiver §G.5. The component registration step in
`main/CMakeLists.txt` (§I.2) replaces `EMBED_TXTFILES` for the CA and
`EMBED_FILES` for the gzipped UI; the existing transmitter
`CMakeLists.txt` already declares `esp_http_server` in
`PRIV_REQUIRES`, so no extra component is needed.

### G.6 433 MHz transmitter — extend the existing `rf/` module

The current `transmitter/main/rf/` already implements RC-Switch-style
encoding via `rf_common.h` / `rf_data.c` / `rf_timer.c` /
`rf_transmitter.c`. The pulse engine and `gptimer` ISR are
production-shaped; what's missing for backend dispatch is a **bits-
oriented** encoder that complements the existing `tristate_to_pulses`
(PT2272 remote emulation).

`rf_data.c`'s `tristate_to_pulses` represents each tri-state symbol
('0', '1', 'F') as **two** pulse-pairs — the PT2272 wire format.
Backend `cmd/transmit` for `band=433M` carries a flat `uint32_t` of
`bits ∈ [1, 32]` (Device Integration Plan §3.1 schema comment). Encoding it as
PT2272 tri-state would only work for codes whose 2-bit groups are in
`{00, 01, 11}` and would silently misencode `10` groups. Don't go
through tri-state; emit one pulse-pair per logic bit:

```c
// rf/rf_common.h (additions)
#include "freertos/FreeRTOS.h"

#define RF433_TX_TIMEOUT_MARGIN_MS 50

esp_err_t bits_to_pulses(const uint8_t *plaintext, size_t byte_len, uint8_t bits,
                         RFPulse **pulses, size_t *pulse_count, const Protocol *proto);

esp_err_t rf433_tx_init(gpio_num_t tx_gpio, int8_t repeat_count,
                        Protocol *proto, RFHandler *h);
esp_err_t rf433_tx_send_bits(RFHandler *h, const uint8_t *plaintext,
                             size_t byte_len, uint8_t bits);
esp_err_t rf433_tx_wait_done(RFHandler *h, TickType_t timeout);
TickType_t rf433_tx_timeout_ticks(const RFHandler *h);
```

```c
// rf/rf_data.c (addition)
esp_err_t bits_to_pulses(const uint8_t *plaintext, size_t byte_len, uint8_t bits,
                         RFPulse **pulses, size_t *pulse_count, const Protocol *proto) {
    ESP_RETURN_ON_FALSE(plaintext && proto && bits >= 1 && bits <= 32 &&
                        byte_len == (bits + 7) / 8,
                        ESP_ERR_INVALID_ARG, RF_TAG, "bad args");

    if (*pulses) { free(*pulses); *pulses = NULL; }
    *pulse_count = bits * 2 + 2;            // one pair per bit + sync pair
    *pulses = malloc(*pulse_count * sizeof(RFPulse));
    if (!*pulses) return ESP_ERR_NO_MEM;

    const uint8_t hi = proto->inverted ? 0 : 1;
    const uint8_t lo = proto->inverted ? 1 : 0;

    // Walk MSB-first across the bottom `bits` of the big-endian plaintext.
    int idx = 0;
    for (int b = bits - 1; b >= 0; b--) {
        const int byte_off = byte_len - 1 - (b / 8);
        const uint8_t bit  = (plaintext[byte_off] >> (b % 8)) & 1;
        const RFTicks *t   = bit ? &proto->one : &proto->zero;
        (*pulses)[idx].level        = hi;
        (*pulses)[idx].pulse_length = t->high * proto->pulse_length;
        (*pulses)[idx + 1].level        = lo;
        (*pulses)[idx + 1].pulse_length = t->low  * proto->pulse_length;
        idx += 2;
    }
    (*pulses)[idx].level        = hi;
    (*pulses)[idx].pulse_length = proto->sync_factor.high * proto->pulse_length;
    (*pulses)[idx + 1].level        = lo;
    (*pulses)[idx + 1].pulse_length = proto->sync_factor.low  * proto->pulse_length;
    return ESP_OK;
}
```

```c
// rf/rf_transmitter.c (additions)
esp_err_t rf433_tx_init(gpio_num_t tx_gpio, int8_t repeat_count, Protocol *proto, RFHandler *h) {
    return rf_trans_init(tx_gpio, repeat_count, proto, h);  // existing fn, kept
}

esp_err_t rf433_tx_send_bits(RFHandler *h, const uint8_t *plaintext,
                             size_t byte_len, uint8_t bits) {
    ESP_RETURN_ON_ERROR(bits_to_pulses(plaintext, byte_len, bits,
                                       &h->pulses[0], &h->pulse_count_per_chn,
                                       h->proto), RF_TAG, "encode");
    h->chn_count = 1;
    ESP_RETURN_ON_ERROR(rf_send(h, 0), RF_TAG, "start tx");
    return rf433_tx_wait_done(h, rf433_tx_timeout_ticks(h));
}
```

`rf_send` already programs the `gptimer` alarm action for the first
pulse, and the existing IRAM `rf_timer_callback` walks the buffer,
executes `repeat_count` repetitions, and drops the GPIO low when
done. Add a completion signal (`SemaphoreHandle_t` or `EventGroup`)
to the 433 wrapper and give it from the final-repeat branch of
`rf_timer_callback`; `rf433_tx_send_bits` must not return before the
off-air condition is true, because §F.2 publishes `applied` only after
the radio reports completion. `rf433_tx_timeout_ticks` sums the
generated pulse lengths, multiplies by `repeat_count`, and adds
`RF433_TX_TIMEOUT_MARGIN_MS`; do not use the nRF24 250 ms budget as a
433 timeout. The MIN/MAX bounds and protocol selection from
`rf_common.h` (`PROTO_ID`, `TX_REPEAT_COUNT`, the `Protocol proto[]`
table) carry over unchanged. Existing
`tristate_to_pulses` and `tristate_to_uint32` stay in the source —
they are useful for bench-side remote emulation tooling and have no
runtime cost when unused.

### G.7 nRF24L01 / nRF24L01+ transmitter — adapt receiver §G.7

The register/command defines in Receiver §G.7.2 (`nrf24_regs.h`) are
band-agnostic and reused as-is, with **one PTX-only addition** the
receiver doesn't need: `NRF_DYNPD_P0 (1U << 0)` for enabling DPL on
the ACK-receiving pipe (the receiver header already defines
`NRF_DYNPD_P1` for its data pipe; pipes 2–5 stay unused). The
chip-probe sequence in Receiver §G.7.6 is reused unchanged. What
changes is the role: the hub is a **PTX**, not a PRX, so
`CONFIG.PRIM_RX = 0` and the address set moves to `TX_ADDR` +
`RX_ADDR_P0` (Receiver §G.7.7 explicitly calls out the address
ownership table for the transmitter side).

Public API:

```c
// nrf24/nrf24_transmitter.h
#pragma once
#include "esp_err.h"
#include "freertos/FreeRTOS.h"
#include "freertos/semphr.h"
#include <stdbool.h>

typedef struct {
    bool             ready;
    nrf24_variant_t  chip;       // populated by nrf_probe_chip()
    SemaphoreHandle_t tx_done;   // released by IRQ on TX_DS or MAX_RT
    void            *spi;        // opaque spi_device_handle_t
} nrf24_tx_t;

esp_err_t nrf24_tx_init(nrf24_tx_t *h);
// `addr` is the destination receiver's rf_code (5 bytes, LSByte first).
// The send path writes it to TX_ADDR + RX_ADDR_P0 immediately before
// loading the fixed TOGGLE_MAGIC payload.
esp_err_t nrf24_tx_send(nrf24_tx_t *h, const uint8_t addr[5]);
esp_err_t nrf24_tx_deinit(nrf24_tx_t *h);

// Two-byte payload the hub sends to every 2.4G dispatch. Mirrors the
// receiver's RF_TRIGGER_TOGGLE_MAGIC_HI / _LO. Stored as a static
// const so the byte order on the wire is byte 0 = HI = 0xAA, then
// byte 1 = LO = 0x55.
#define NRF_TX_TOGGLE_MAGIC_HI 0xAA
#define NRF_TX_TOGGLE_MAGIC_LO 0x55
```

Bringup deltas vs Receiver §G.7.4:

```c
// nrf24/nrf24_transmitter.c — bringup (deltas only; everything else mirrors §G.7.4)
static esp_err_t nrf_tx_bringup(nrf24_tx_t *h) {
    // CE low; power-on settle; SETUP_AW liveness probe + chip probe.
    // (Same as receiver §G.7.4 nrf_bringup — copy verbatim.)

    // Configure for PTX with auto-ACK enabled on pipe 0 so the chip can
    // accept the ACK frame the PRX returns (the receiver firmware has
    // EN_AA = 0x02 = pipe 1 enabled). The only IRQ source we mask off is
    // RX_DR — a PTX never sources data packets, so an asserted RX_DR
    // would only ever be a stray ACK echo. TX_DS and MAX_RT both stay
    // unmasked: the ISR releases tx_done on either, and the send path
    // reads STATUS afterwards to disambiguate (datasheet PS v1.0 §6.5).
    nrf_wreg8(NRF_REG_CONFIG,
              NRF_CFG_MASK_RX_DR                          // PTX has no RX path
              | NRF_CFG_EN_CRC | NRF_CFG_CRCO);           // PRIM_RX = 0, PWR_UP = 0

    nrf_wreg8(NRF_REG_EN_AA,      0x01);                   // pipe 0 auto-ACK only
    nrf_wreg8(NRF_REG_EN_RXADDR,  0x01);                   // pipe 0 enabled (for ACK)
    nrf_wreg8(NRF_REG_SETUP_AW,   0x03);                   // 5-byte addresses
    // ARD = 750 µs (datasheet ARD field 0010), ARC = 3 retransmits.
    // The 750 µs floor satisfies the ACK-without-payload requirement on
    // 1 Mbps and 2 Mbps (datasheet PS v1.0 §7.4.2 ACK-payload note); we
    // intentionally don't expose 250 kbps, which would force ARD ≥ 500 µs.
    nrf_wreg8(NRF_REG_SETUP_RETR, (2 << 4) | 3);
    nrf_wreg8(NRF_REG_RF_CH,      CONFIG_TRANSMITTER_NRF24_CHANNEL);

#if CONFIG_TRANSMITTER_NRF24_DR_2M
    // NRF_RF_LNA_HCURR is documented as obsolete on the nRF24L01+ (writes
    // ignored) and helps RX sensitivity by ~+1.5 dB on the legacy
    // nRF24L01. We OR it into RF_SETUP so the ACK reception path benefits
    // on whichever silicon is populated. See receiver §G.7.6 compat table.
    nrf_wreg8(NRF_REG_RF_SETUP, NRF_RF_DR_2M | NRF_RF_PWR_0 | NRF_RF_LNA_HCURR);
#else
    nrf_wreg8(NRF_REG_RF_SETUP, NRF_RF_DR_1M | NRF_RF_PWR_0 | NRF_RF_LNA_HCURR);
#endif

#if CONFIG_TRANSMITTER_NRF24_ENABLE_DPL
    nrf_wreg8(NRF_REG_FEATURE, NRF_FEATURE_EN_DPL);
    // PTX-only: DPL must be enabled on the ACK-receiving pipe (pipe 0).
    // `NRF_DYNPD_P0` is the one extension this firmware adds to the
    // shared `nrf24_regs.h` from receiver §G.7.2 — see §G.7 intro.
    nrf_wreg8(NRF_REG_DYNPD,   NRF_DYNPD_P0);
#else
    nrf_wreg8(NRF_REG_FEATURE, 0x00);
    nrf_wreg8(NRF_REG_DYNPD,   0x00);
#endif

    // PTX address ownership: TX_ADDR == RX_ADDR_P0 == destination
    // receiver's RX_ADDR_P1 (= the receiver's rf_code, Device
    // Integration Plan §2.4). Per-dispatch addressing — the hub does not bake any
    // address at boot. `nrf24_tx_send` writes both registers from the
    // 5 bytes carried in `cmd/transmit.rf_code_hex` immediately
    // before each W_TX_PAYLOAD.

    nrf_cmd(NRF_CMD_FLUSH_TX);
    nrf_cmd(NRF_CMD_FLUSH_RX);
    nrf_wreg8(NRF_REG_STATUS, NRF_ST_RX_DR | NRF_ST_TX_DS | NRF_ST_MAX_RT);

    // Bringup leaves the chip in Power-Down (PWR_UP = 0) per the §F.7
    // power policy. `nrf24_tx_send` walks Power-Down → Standby-I → TX →
    // Standby-I → Power-Down on each dispatch.
    return ESP_OK;
}
```

Send path:

```c
esp_err_t nrf24_tx_send(nrf24_tx_t *h, const uint8_t addr[5]) {
    if (!h->ready) return ESP_ERR_INVALID_STATE;
    if (!addr) return ESP_ERR_INVALID_ARG;

    // Power-Down → Standby-I. The chip idles in Power-Down between
    // dispatches (§F.7); `Tpd2stby` is bounded by NRF_TPD2STBY_MS, which
    // is set to a value generous enough to cover both PS v2.0 (1.5 ms)
    // and PS v1.0 (4.5 ms with Ls = 90 mH) — see receiver §G.7.2.
    uint8_t cfg = nrf_rreg8(NRF_REG_CONFIG) | NRF_CFG_PWR_UP;
    nrf_wreg8(NRF_REG_CONFIG, cfg);
    vTaskDelay(pdMS_TO_TICKS(NRF_TPD2STBY_MS));

    // Per-dispatch addressing: TX_ADDR == RX_ADDR_P0 == destination
    // receiver's RX_ADDR_P1 (= rf_code). Auto-ACK on pipe 0 requires
    // the equality. Both writes are 5 SPI bytes each (~24 µs total
    // at 4 MHz SPI), negligible against Tpd2stby.
    nrf_wreg(NRF_REG_TX_ADDR,    addr, 5);
    nrf_wreg(NRF_REG_RX_ADDR_P0, addr, 5);

    // Clear stale flags then load the fixed 2-byte TOGGLE_MAGIC.
    nrf_wreg8(NRF_REG_STATUS, NRF_ST_RX_DR | NRF_ST_TX_DS | NRF_ST_MAX_RT);
    static const uint8_t k_toggle_magic[2] = {
        NRF_TX_TOGGLE_MAGIC_HI, NRF_TX_TOGGLE_MAGIC_LO,
    };
    nrf_xfer(NRF_CMD_W_TX_PAYLOAD, k_toggle_magic, NULL, sizeof k_toggle_magic);

    // CE high ≥ 10 µs → Standby-I → TX. Datasheet PS v1.0 §6.1.7 Thce.
    gpio_set_level(CE_GPIO, 1);
    esp_rom_delay_us(15);                                  // 15 > 10 µs Thce
    gpio_set_level(CE_GPIO, 0);

    // Wait for TX_DS or MAX_RT. 50 ms cap dominates worst-case TX time:
    // initial frame + ARC=3 retransmits at ARD=750 µs each plus on-air
    // frame time (≤ ~1 ms per frame at 1 Mbps with a 32-byte payload —
    // the magic is 2 B so even shorter) is single-digit ms; 50 ms is
    // comfortable margin without coupling RF latency to MQTT
    // scheduling jitter.
    esp_err_t ret = ESP_OK;
    if (xSemaphoreTake(h->tx_done, pdMS_TO_TICKS(50)) != pdTRUE) {
        nrf_cmd(NRF_CMD_FLUSH_TX);
        ret = ESP_ERR_TIMEOUT;
    } else {
        uint8_t status = nrf_rreg8(NRF_REG_STATUS);
        nrf_wreg8(NRF_REG_STATUS, NRF_ST_TX_DS | NRF_ST_MAX_RT);  // W1C
        if (status & NRF_ST_MAX_RT) {
            nrf_cmd(NRF_CMD_FLUSH_TX);
            ret = ESP_FAIL;                                // ack-side timed out
        }
        // else: TX_DS asserted → ret = ESP_OK
    }

    // Standby-I → Power-Down. Always taken so a TX failure can't leave
    // the chip drawing Standby-I current indefinitely.
    cfg = nrf_rreg8(NRF_REG_CONFIG) & ~NRF_CFG_PWR_UP;
    nrf_wreg8(NRF_REG_CONFIG, cfg);
    return ret;
}
```

ISR is the same shape as the receiver's (`nrf_isr` in Receiver §G.7.4)
with one change: it gives the `tx_done` semaphore instead of
unblocking an RX task. The `MASK_RX_DR` bit in `CONFIG` ensures the
IRQ pin only asserts on `TX_DS` or `MAX_RT`, so a single ISR body
covers both events; the send path inspects `STATUS` to disambiguate.

DPL on the **TX side** is symmetric: with
`CONFIG_TRANSMITTER_NRF24_ENABLE_DPL=y`, the chip sends a
variable-length frame whose length matches the `W_TX_PAYLOAD` write.
Receiver §G.7.5 already documents that DPL is bilateral — the receiver
must also have DPL on, or it CRC-rejects the frame silently.
Deployments should keep both `CONFIG_RECEIVER_NRF24_ENABLE_DPL` and
`CONFIG_TRANSMITTER_NRF24_ENABLE_DPL` defaulted to `y`.

Address byte order is the same convention as Receiver §G.7.7: byte
index 0 is the LSByte on the wire, byte 4 is the MSByte. The hub
copies the 5 bytes from `cmd/transmit.rf_code_hex` (decoded
big-endian-textually but laid out as wire-order bytes) directly into
the SPI payload of `W_REGISTER(TX_ADDR, …)` and
`W_REGISTER(RX_ADDR_P0, …)` per dispatch — no in-firmware swap, no
Kconfig involvement. The receiver firmware applies the same bytes
to its `RX_ADDR_P1`, so a frame transmitted to address `A` is
silicon-matched by exactly one receiver in the global 2.4G uniqueness
namespace (Device Integration Plan §D5.0).

### G.8 Dispatch executor

The `dispatch_handler` is the firmware's only consumer of
`cmd/transmit`. It owns no radio state directly — it goes through
`radio_supervisor.radio_tx_send` so the same mutex covers both bands.

```c
// dispatch/dispatch.c (sketch)
typedef struct {
    char     dispatch_id[40];
    char     status[16];
    char     reason[32];
    uint64_t accepted_at_us;
} dispatch_ring_entry_t;
static dispatch_ring_entry_t s_ring[8];
static uint8_t s_ring_head;

static const dispatch_ring_entry_t *ring_find(const char *dispatch_id) {
    for (int i = 0; i < (int)(sizeof s_ring / sizeof s_ring[0]); i++)
        if (s_ring[i].dispatch_id[0] && strcmp(s_ring[i].dispatch_id, dispatch_id) == 0)
            return &s_ring[i];
    return NULL;
}

static void ring_remember(const char *dispatch_id, const char *status, const char *reason) {
    strlcpy(s_ring[s_ring_head].dispatch_id, dispatch_id,
            sizeof s_ring[s_ring_head].dispatch_id);
    strlcpy(s_ring[s_ring_head].status, status, sizeof s_ring[s_ring_head].status);
    strlcpy(s_ring[s_ring_head].reason, reason ? reason : "", sizeof s_ring[s_ring_head].reason);
    s_ring[s_ring_head].accepted_at_us = esp_timer_get_time();
    s_ring_head = (s_ring_head + 1) % (sizeof s_ring / sizeof s_ring[0]);
}

void dispatch_handle(const char *json, size_t len) {
    parsed_t p;
    if (!parse_transmit_json(json, len, &p)) { ack_reject("bad_shape", &p); return; }
    if (!verify_band_and_width(&p))     { ack_reject("width_mismatch", &p); return; }
    if (!verify_signature(&p))          { ack_reject("signature_failed", &p); return; }
    const dispatch_ring_entry_t *prev = ring_find(p.dispatch_id);
    if (prev)                           { ack_replay(&p, prev->status, prev->reason); return; }
    if (g_cfg.op_state != OP_STATE_ACTIVE) {
        const char *reason = g_cfg.op_state == OP_STATE_SUSPENDED
                             ? "device_suspended" : "device_decommissioned";
        ring_remember(p.dispatch_id, "rejected", reason);
        ack_reject(reason, &p);
        return;
    }

    uint8_t plaintext[32];
    size_t  byte_len = 0;
    if (!hex_decode(p.rf_code_hex, plaintext, sizeof plaintext, &byte_len)) {
        ring_remember(p.dispatch_id, "rejected", "bad_hex");
        ack_reject("bad_hex", &p); return;
    }
    radio_band_t band = (strcmp(p.band, "433M") == 0) ? RADIO_BAND_433M : RADIO_BAND_2_4G;
    esp_err_t r = radio_tx_send(band, plaintext, byte_len, p.rf_code_bits);
    if (r == ESP_OK) {
        ring_remember(p.dispatch_id, "applied", NULL);
        ack_applied(&p);
    } else if (r == ESP_ERR_NOT_SUPPORTED) {
        ring_remember(p.dispatch_id, "rejected", "no_2_4g_radio");
        ack_reject("no_2_4g_radio", &p);
    } else {
        ring_remember(p.dispatch_id, "rejected", "tx_failed");
        ack_reject("tx_failed", &p);
    }
}
```

Notes:

- Signature verification happens before op-state rejection and before
  replay-ring lookup. That keeps §F.5 honest for suspended hubs and
  prevents an invalidly signed frame from poisoning the replay ring
  with a real `dispatch_id`.
- `ack_replay` republishes the prior terminal outcome stored in the
  ring. For an earlier `applied`, it can emit `status="unchanged"`;
  for earlier rejects, it repeats the same reject reason. In all cases
  the duplicate causes no RF emission.
- The mutex order is `dispatch_handle → radio_supervisor.radio_tx_send`.
  Never publish the ack while holding `radio_mtx`: ESP-MQTT
  `_publish` is thread-safe, but holding a radio lock across a network
  call would couple TX latency to broker latency.
- `parse_transmit_json` uses `cJSON` from `espressif__cjson` (already
  in `transmitter/main/idf_component.yml`).
- Signature verify reuses the same `device_identity_verify(...)`
  helper as the receiver's `cmd/rf_code` and `cmd/deact` paths.

### G.9 Heartbeat publisher

```c
// network/heartbeat.h
esp_err_t heartbeat_start(void);
esp_err_t heartbeat_stop(void);

// network/heartbeat.c (sketch)
#define HB_INTERVAL_MS  10000
#define HB_JITTER_MS     500

static TaskHandle_t s_hb;
static volatile bool s_hb_run;

static void hb_task(void *arg) {
    char topic[96];
    snprintf(topic, sizeof topic, "transmitter/hub/%s/heartbeat", g_cfg.public_id);
    while (s_hb_run) {
        char payload[64];
        char ts[32];
        iso8601_now(ts, sizeof ts);                // helper in security/utils
        int n = snprintf(payload, sizeof payload,
                         "{\"schema_version\":1,\"issued_at\":\"%s\"}", ts);
        (void)mqtt_publish(topic, payload, n, /*qos=*/0, /*retain=*/0);
        // ±500 ms jitter
        int jitter = (esp_random() % (2 * HB_JITTER_MS + 1)) - HB_JITTER_MS;
        vTaskDelay(pdMS_TO_TICKS(HB_INTERVAL_MS + jitter));
    }
    vTaskDelete(NULL);
}

esp_err_t heartbeat_start(void) {
    if (s_hb) return ESP_OK;
    s_hb_run = true;
    return xTaskCreate(hb_task, "hb", 3072, NULL, 4, &s_hb) == pdPASS
           ? ESP_OK : ESP_ERR_NO_MEM;
}

esp_err_t heartbeat_stop(void) {
    s_hb_run = false;
    s_hb = NULL;
    return ESP_OK;
}
```

`mqtt_publish` is a thin wrapper around `esp_mqtt_client_publish`
that returns silently when `s_client == NULL` or the queue is full —
the same pattern the receiver uses for ack publishes. A dropped tick
is a non-event; only sustained silence (≥ 30 s, §H.6) has operational
meaning.

### G.10 Device identity

Reuse Receiver §G.10 verbatim. The pinned backend public key Kconfig
key is renamed `CONFIG_TRANSMITTER_BACKEND_PUBKEY_B64`. The signing
key is shared with the receiver fleet (§H.7.0 / Device Integration Plan §6: one
signing key covers both families), so the dev `.pem` generated for
receivers can be reused for hubs without rotation.

Performance budget (C3 @ 160 MHz):

- ECDSA verify on a `transmit-v1|...` canonical: 70–100 ms (Receiver
  §G.10 figure, validated for SHA-256 + ECDSA on EC P-256).
- ECDSA sign on the activation challenge: 100–150 ms (one-shot per
  activation).

Both are bounded by mbedTLS hardware acceleration and well within the
§B.3 dispatch budget.

---

## H. Backend Implementation Plan (DUPLICATE — see Device Integration Implementation Plan)

> **Status (2026-04-27):** This section originated the backend slice
> for the transmitter hub and has been folded into
> `docs/planned/Device Integration Implementation Plan.md` (Phases F0,
> T1–T7). That plan is now the source of truth for backend work and
> covers receivers and transmitters together. The text below is kept
> in place for firmware-side cross-references (§A–§G refer to §H.x
> anchors); when the two diverge, fix the implementation plan first
> and refresh §H here.

This section is the backend slice of the transmitter hub plan: the
package additions, configuration, and phased rollout that turn the
firmware contract above into a working server. It builds on top of
the unified `device` domain established by the
`Device Integration Implementation Plan.md` and reuses receiver-
domain code wherever the contracts overlap.

### H.1 Topology (the contract pinned for firmware)

These rules are the load-bearing pins behind the firmware sections;
they exist here so the backend implementation has a single source of
truth and the firmware sections can reference them with §H.1
anchors instead of restating.

- **MQTT-driven, not autonomous.** A hub does nothing on its own
  (§A.1). It subscribes to
  `transmitter/hub/{public_id}/cmd/transmit` and broadcasts the
  contained RF code on receipt. No RF activity happens without a
  backend dispatch event.
- **Heartbeat-required.** Each hub publishes a periodic heartbeat
  (10 s interval, 30 s liveness window) on
  `transmitter/hub/{public_id}/heartbeat` (§E.1). The backend updates
  `device.last_seen_at` and bumps a Redis liveness key (§H.7.3).
- **Per-store registered cap: 3 (ceiling, not floor).** A store may
  operate with **1**, **2**, or **3** registered hubs. The cap matters
  at registration time only: don't accept a 4th. A store can run with
  one hub for a stretch — for example, while waiting for a redundancy
  unit to ship — and that single hub serves as both active and the
  entire pool. There is no minimum.
- **Per-store active cap: 1.** With 0 registered: dispatch fails with
  `no_active_transmitter`. With 1 registered and live: that hub is
  the active. With 2–3 registered: election picks the hub with the
  most recent live heartbeat (ties broken by lowest `device.id`).
  When *no* registered hub is heartbeating, dispatch fails — admin
  sees the failure and triages, no silent queueing.
- **Failover is automatic.** When the cached primary stops
  heartbeating, the next dispatch event re-elects from live hubs. No
  promote/demote endpoint needed; admin `suspend` / `decommission`
  removes a hub from the candidate pool.
- **No simultaneous broadcast.** Because only one hub transmits per
  dispatch event, RF collisions between peer hubs are not a concern
  and the hubs need no inter-hub coordination protocol.

What the **receiver plan already covers** (no schema retrofit when
this slice lands):

- `device_kind` reserves `TRANSMITTER_HUB`.
- `device.last_seen_at` carries the heartbeat timestamp.
- `device.status` (`ACTIVE` / `SUSPENDED` / `DECOMMISSIONED`) covers
  candidate-pool admission. `REJECTED` is a backend-only state.
- `DevicePublicIdMinter` reserves `hub-XXXXX`.
- `device_rf_code` is scoped to receivers — hubs never get a row.

### H.2 Backend MQTT subscriptions and publishers

Backend startup subscriptions (via `MqttClientManager`):

- `transmitter/bootstrap/register`
- `transmitter/bootstrap/+`
- `transmitter/hub/+/ack`
- `transmitter/hub/+/heartbeat`

The backend already uses the Paho MQTT v5 client, but the current
`MqttClientManager` wrapper only exposes `subscribe(topicFilter, qos)`
and caches subscriptions as `topic -> qos` for reconnect. Phase T0
therefore extends that wrapper with a v5 subscription overload (or an
equivalent stored subscription descriptor) so
`transmitter/bootstrap/+` can subscribe with
`MqttSubscription.setNoLocal(true)` and preserve that option across
reconnects. The bootstrap listener still drops backend-owned envelope
types (`pending`, `challenge`, `rejected`, `result`) defensively if
they arrive on the `+` wildcard: those envelope types are
backend-originated and must never advance the hub registration state
machine (§H.7.1).

Publishers (extend `DeviceMqttPublisher`):

| Method                                         | Topic                                          | QoS | Retained |
| ---------------------------------------------- | ---------------------------------------------- | --- | -------- |
| `publishTransmit(hubPublicId, payload)`        | `transmitter/hub/{publicId}/cmd/transmit`     | 1   | **no**   |
| `publishDeact(...)` (hub variant)              | `transmitter/hub/{publicId}/cmd/deact`        | 1   | yes      |
| `clearRetained(publicId, kind=TRANSMITTER_HUB)`| `transmitter/hub/{publicId}/cmd/deact`        | 1   | yes      |

`publishDeact` already exists in the receiver path and gains a kind-
aware route to the hub topic family when `device.kind = TRANSMITTER_HUB`.
`clearRetained` issues a zero-length retained publish on
decommission and on reactivation that mints a new `public_id`, same
hygiene as the receiver path. The `publishTransmit` overload is
**not retained** — the broker must never replay a stale broadcast.

### H.3 Canonical strings (backend builders)

Three canonical strings the backend builds and signs. Builders live
in `core/device/DeviceCanonical.kt` next to the receiver-side ones.
UTF-8, no trailing newline, exact field order:

- `activate-v1|<challenge_id>|<nonce>|<issued_at>|<expires_at>` —
  reused verbatim from the Device Integration Plan §2.3. The transmitter
  activation path differs only in the topic namespace.
- `deact-v1|<public_id>|<command_id>|<action>|<issued_at>` — reused
  verbatim from the Device Integration Plan §2.3. Applied to the hub topic
  namespace via the `DeviceMqttPublisher` route.
- `transmit-v1|<hub_public_id>|<dispatch_id>|<receiver_public_id>|<band>|<rf_code_hex>|<rf_code_bits>|<issued_at>`
  — new, with the `band` field per §H.4.

Rules:

- `dispatch_id` is a UUID minted by the backend per dispatch event so
  the hub can dedupe replays.
- `rf_code_hex` is uppercase hex.
- `issued_at` is ISO-8601 UTC ending in `Z`.
- Integer fields are base-10 without prefixes.

The builder for `transmitV1(...)` ships with a golden-string unit
test pinned against the example at the end of §E.3.

### H.4 Required `transmit-v1` band field

The §2.4 width rules now make `bits` disjoint per band (40 only on
2.4G; 1..32 only on 433M), so `band` is no longer required for
disambiguation. It stays on the wire because:

- It lets the hub firmware route to the right radio without
  back-deriving from `bits` (a one-line `if (band == "433M")` is
  cheaper to read and audit than width-table reasoning).
- It makes the signed canonical string self-describing for audit
  and replay analysis without forcing the reader to remember the
  width-to-band mapping.
- It leaves room for future widths without re-cutting the wire
  format — for example, if 24-bit PT2272 codes ever flow through a
  hub, `band = "433M"` already disambiguates the 24-bit case.

Population:

- JSON field `"band": "433M" | "2_4G"` is part of `cmd/transmit`
  (§E.3), filled from `device.kind` on the receiver row:
  - `RECEIVER_433M` and `RECEIVER_433M_PASSIVE` → `"433M"`.
  - `RECEIVER_2_4G` → `"2_4G"`.
- Canonical string is the §H.3 form, which includes the field.
- The dispatch consumer (§H.7.5 step 3) reads `device.kind` from the
  receiver row it just loaded the rf_code from, and uses it to pick
  `band` before signing. No new DB roundtrip — the kind is on the
  same row.

This is not a contract break against an existing firmware: no
transmitter firmware is in production yet, so `band` lands as part
of `transmit-v1` from day one rather than a future `transmit-v2`.

### H.5 Package additions to the unified `device` domain

Add the following to the unified `device` domain established by the
Device Integration Plan §5:

```text
backend/src/main/kotlin/com/thomas/notiguide/domain/device/
├── listener/
│   ├── TransmitterBootstrapListener.kt    # transmitter/bootstrap/{register,+}     (§H.7.1)
│   └── TransmitterOperationalListener.kt  # transmitter/hub/+/{heartbeat,ack}      (§H.7.3)
└── service/
    ├── TransmitterDispatchService.kt      # broadcaster consumer → cmd/transmit    (§H.7.5)
    └── TransmitterElectionService.kt      # active-hub election, Redis-cached      (§H.7.4)
```

Existing receiver-domain code is reused verbatim:

| Concern                                                              | Reused from Device Integration Plan                                                       |
| -------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| Enrollment tokens                                                    | `EnrollmentTokenService` (D1)                                                   |
| Bootstrap intake (validation, token consume, upsert, challenge mint) | `DeviceRegistrationService` (D2)                                                |
| Approval / activation                                                | `DeviceApprovalService` (D3), `DeviceActivationService` (D4)                    |
| Lifecycle commands                                                   | `DeviceLifecycleService` (D6)                                                   |
| MQTT publish                                                         | `DeviceMqttPublisher` (gains `publishTransmit` and a hub variant of `publishDeact` / `clearRetained`) |
| Public ID minting                                                    | `DevicePublicIdMinter` (mints `hub-XXXXX` based on `kind`)                      |
| Command signing                                                      | `DeviceCommandSigner` (one signing key covers both families)                    |

### H.6 Backend configuration

Add to `application.yaml`, next to the `device:` block from the
Device Integration Plan §6:

```yaml
device:
  transmitter:
    enabled: ${DEVICE_TRANSMITTER_ENABLED:false}     # feature gate; flip to true when firmware ships
    heartbeat-interval-seconds: 10                   # informational; firmware default
    heartbeat-liveness-seconds: 30                   # Redis TTL for device:hub:alive:*
    active-cache-seconds: 60                         # Redis TTL for store:{id}:transmitter:active
    max-registered-per-store: 3                      # ceiling, not floor (§H.1)
```

`enabled = false` keeps the transmitter listeners and dispatch
consumer **unregistered**, so the backend silently ignores any stray
transmitter MQTT traffic and the queue-side dispatch service surfaces
`no_active_transmitter` immediately. Flipping to `true` in deployment
config and restarting the backend once firmware exists turns the
feature on without further code changes.

`heartbeat-*` and `active-cache-*` are deliberate hardcoded ceilings,
config-readable for ops visibility but not per-store overridable.
Bending them requires firmware coordination, so they belong in YAML,
not in `store_settings`.

Bind via `DeviceTransmitterProperties` `@ConfigurationProperties` bean.
All new listeners and `TransmitterDispatchService` are
`@ConditionalOnProperty(prefix = "device.transmitter", name = "enabled", havingValue = "true")`.
Spring Boot 3.5 evaluates **multiple** `name` entries on a single
`@ConditionalOnProperty` with AND semantics ("all properties must
pass the test for the condition to match"); since this guide uses
exactly one `name`, the AND/OR distinction does not bite here.

### H.7 Implementation phases

Each phase is a standalone slice — audit between phases per the
project convention. Every phase updates `docs/CHANGELOGS.md` with
every file touched and any skipped item.

#### H.7.0 Phase T0 — Foundation

1. Add `DeviceCanonical.transmitV1(...)` per §H.3. Unit-test against
   the golden string from §E.3.
2. Extend `MqttClientManager` so it can register and reconnect an MQTT
   v5 `MqttSubscription` with `setNoLocal(true)` for
   `transmitter/bootstrap/+`; the current wrapper only supports
   `subscribe(topicFilter, qos)`.
3. Extend `DeviceMqttPublisher`:
   - `publishTransmit(hubPublicId, payload)` — QoS 1, **not retained**.
   - `publishDeact(...)` re-targeted: route to
     `transmitter/hub/{publicId}/cmd/deact` when
     `device.kind = TRANSMITTER_HUB`, else the existing receiver path.
   - `clearRetained(publicId, kind)` gains a hub variant.
4. Add `device.transmitter.*` properties + `DeviceTransmitterProperties`
   `@ConfigurationProperties` bean.
5. All new listeners and `TransmitterDispatchService` are
   `@ConditionalOnProperty(prefix = "device.transmitter", name = "enabled", havingValue = "true")`
   so the feature gate is real.

The signing key, `DeviceCommandSigner`, and the
`DeviceCommandSigningProperties` config from the Device Integration Plan §6 are
reused unchanged. One signing key covers both fleets.

#### H.7.1 Phase T1 — Bootstrap intake

- `TransmitterBootstrapListener` subscribes to
  `transmitter/bootstrap/register` and `transmitter/bootstrap/+`,
  with the same self-echo drop rule as the receiver listener (§H.2).
- Generalise `DeviceRegistrationService.onRegister` to accept a
  `DeviceFamily` discriminator (`RECEIVER` / `TRANSMITTER`) inferred
  from the topic. Receiver registration rejects `TRANSMITTER_HUB`;
  transmitter registration rejects everything except `TRANSMITTER_HUB`.
- Per-store cap check **before** insert: count rows where
  `kind = TRANSMITTER_HUB AND store_id = X AND status NOT IN ('DECOMMISSIONED', 'REJECTED')`.
  Reject with `rejected` envelope `reason: "hub_cap_reached"` if
  `count >= max-registered-per-store`. Decommissioning a hub frees a
  slot.

#### H.7.2 Phase T2 — Activation

- Reuse `DeviceActivationService.onResponse` verbatim — the canonical
  `activate-v1` string and the ECDSA verify path are family-agnostic.
- `DevicePublicIdMinter` mints `hub-XXXXX` instead of `rcv-XXXXX`
  based on `device.kind`.
- After a hub activates, `status` goes straight to `ACTIVE` (not
  `PENDING_RF_CODE` — hubs have no per-device code; they execute
  whatever the dispatch event tells them to).
- No `device_rf_code` row is ever created for `TRANSMITTER_HUB`. The
  dispatch consumer (§H.7.5) skips the `device_rf_code` lookup for
  hubs.

#### H.7.3 Phase T3 — Operational topic ingestion

- `TransmitterOperationalListener` subscribes to
  `transmitter/hub/+/heartbeat` and `transmitter/hub/+/ack`.
- For each heartbeat frame:
  1. Update `device.last_seen_at` for the matching `public_id`
     (one-row UPDATE; non-blocking).
  2. Bump a Redis liveness key:
     ```
     device:hub:alive:{deviceId} → "1"   EX 30s
     ```
  TTL = 30 s = 3× the suggested 10 s heartbeat interval. Anything not
  pinging within that window falls out of the candidate pool
  automatically.
- For each `ack_for = "transmit"` frame: update `device.last_seen_at`,
  match on `dispatch_id`, and log/drop stale or out-of-order acks.
- For each `ack_for = "deact"` frame: reuse the receiver-plan
  lifecycle-ack handling keyed by `command_id` / `public_id`; the
  transmitter path differs only by topic namespace and
  `TRANSMITTER_HUB` routing.

#### H.7.4 Phase T4 — Active-hub election

- `TransmitterElectionService.electActive(storeId)` returns the
  elected hub:
  1. Read cached `store:{storeId}:transmitter:active`. If present
     **and** the cached hub still has a live `device:hub:alive:*`
     key, return it.
  2. Otherwise scan candidates:
     `kind = TRANSMITTER_HUB AND store_id = X AND status = 'ACTIVE'`,
     filter by live alive-key, pick the one with the highest
     `last_seen_at` (ties broken by lowest `device.id` for
     determinism), cache the result with `active-cache-seconds` TTL,
     and return.
  3. If no candidate, return `null` — the dispatch service translates
     that into `no_active_transmitter`.
- Election runs **lazily, only on dispatch**, so heartbeat traffic
  doesn't pay election cost.

#### H.7.5 Phase T5 — Dispatch consumer

- `TransmitterDispatchService` subscribes to
  `DeviceDispatchEventBroadcaster`. The broadcaster shape is
  documented by the Device Integration Plan §D7a alongside the queue-side
  publisher; the Reactor underpinning is `Sinks.many().multicast().directBestEffort()`
  — fan out to coroutine-side consumers, drop on slow subscriber so
  one stuck listener never stalls dispatch.
- On `DEVICE_CALL_REQUESTED`:
  1. `electActive(event.storeId)`. If null, mark the queue-side
     dispatch as failed through the same admin-visible queue-dispatch
     failure path owned by Device Integration Plan §D7a; this guide does not pin
     a separate transmitter-specific SSE event name.
  2. Decrypt the receiver's RF code (only point in the system that
     does so on the dispatch hot path).
  3. Build `transmit-v1|hub_public_id|dispatch_id|receiver_public_id|band|rf_code_hex|rf_code_bits|issued_at`,
     where `band` is derived from the receiver row's `kind` (§H.4),
     sign with the existing command-signing key.
  4. Publish on `transmitter/hub/{hubPublicId}/cmd/transmit`
     (QoS 1, not retained).
- On `DEVICE_STOP_REQUESTED`:
  - For the receiver vibrator-toggle model, the firmware design has
    the next matching RF frame toggle the vibrator off. So "stop" is
    just another `transmit-v1` with the same RF code — no new
    canonical string needed. Re-elect, sign, publish.
- Transmit-ack ingestion lives in §H.7.3's operational-listener path.

#### H.7.6 Phase T6 — Lifecycle (reuse from Device Integration Plan §D6)

- `DeviceLifecycleService` already publishes `cmd/deact` with
  `deact-v1`. Hub case routes via the extended `DeviceMqttPublisher`
  from §H.7.0.
- Suspending the elected hub immediately invalidates the Redis
  election cache (`store:{storeId}:transmitter:active`) so the next
  dispatch re-elects.
- Decommissioning a hub frees a registered-cap slot for that store
  (§H.7.1 counts non-terminal status only).

#### H.7.7 Phase T7 — Cap enforcement (formalised)

The cap check from §H.7.1 is the only enforcement point. Surface it
through the existing admin UI cleanly:

- `GET /api/devices?kind=TRANSMITTER_HUB&storeId=X` returns the
  current registered count alongside the list so the admin web can
  show "2 / 3 hubs registered" on the device list page.
- A 4th-hub registration attempt over MQTT fails with the
  `hub_cap_reached` envelope; firmware shows a clear local error and
  goes back into provisioning.

#### H.7.8 Phase T8 — Out of scope (backend-side)

- Hub firmware (this guide §A–§G; not a backend concern).
- Web Serial bridge.
- Dedicated transmitter admin-web screens beyond the unified Device
  tab from the Device Integration Plan.
- Per-device MQTT credentials and broker ACLs.

### H.8 Suggested execution order

Land this slice **after** the receiver plan stabilises end-to-end:

1. **H.7.0** — canonical helper, publisher extension, properties,
   feature gate.
2. **H.7.1 + H.7.2** — bootstrap intake + activation. Verifiable by a
   `mosquitto_pub` mock of the bootstrap flow against
   `transmitter/bootstrap/register`.
3. **H.7.3 + H.7.4** — heartbeat listener + election service.
   Verifiable by simulated heartbeats and `electActive` unit tests.
4. **H.7.5** — dispatch consumer. End-to-end test: issue a
   device-ticket via the receiver plan's queue dispatch UI, confirm
   `cmd/transmit` lands on the broker.
5. **H.7.6 + H.7.7** — lifecycle wiring, cap enforcement, admin-list
   count surface.

The feature flag stays `false` in dev/prod profiles until firmware
exists.

### H.9 Cross-domain dependencies

- The Device Integration Implementation Plan owns the unified
  `device` domain, the `DeviceDispatchEventBroadcaster` producer, the
  RF-code shape, and the queue-side D7a plumbing. This guide consumes
  all of it; nothing here re-implements receiver-domain code.
- Until the firmware in §A–§G ships and a hub is `ACTIVE` for a store,
  the receiver plan's D7a preflight returns `no_active_transmitter`
  and the queue UI dispatch surface (D7b) stays unrendered.

### H.10 Deferred (backend-specific)

- Per-device MQTT credentials and broker ACLs.
- A dedicated transmitter admin screen with hub map / heartbeat
  trace.
- Transmit ack analytics emission (how many dispatches actually fired
  vs. failed silently).
- In-field rotation of the backend command-signing key — shared with
  the Device Integration Plan's deferred list.
- Surfacing `nrf24` probe failures into the heartbeat envelope as a
  capability bitmap so admin can flag a 433-only hub before it tries
  to dispatch a 2.4 G code (related: §J.4).

---

## I. Build Configuration (firmware-side)

### I.1 `main/idf_component.yml`

The current file already declares `idf >= 4.1.0`, `espressif/mqtt`,
and `espressif/cjson`. Tighten the IDF floor to the v6 baseline used
by the receiver and pin the MQTT range:

```yaml
dependencies:
  idf:             '>=6.0.0'
  espressif/mqtt:  '~1.0.0'
  espressif/cjson: '*'
```

Regenerate `dependencies.lock` after the change. The lock currently
records `idf 6.0.0` already, so no managed-component churn is
expected.

### I.2 `main/CMakeLists.txt`

Replace the current `SRCS` list with the post-port set:

```cmake
idf_component_register(
    SRCS "main.c"
         "config/device_config.c"
         "network/wifi.c"
         "network/mqtt.c"
         "network/heartbeat.c"
         "provision/http_server.c"
         "security/device_identity.c"
         "dispatch/dispatch.c"
         "dispatch/radio_supervisor.c"
         "rf/rf_data.c"
         "rf/rf_timer.c"
         "rf/rf_transmitter.c"
         "nrf24/nrf24_transmitter.c"
    INCLUDE_DIRS "."
    EMBED_TXTFILES "network/certs/mqtt_ca.pem"
    EMBED_FILES    "provision/index.html.gz"
    PRIV_REQUIRES  nvs_flash esp_wifi esp_netif esp_event esp_http_server
                   esp_driver_gpio esp_driver_spi esp_timer esp_hw_support
                   mbedtls espressif__mqtt espressif__cjson esp_driver_gptimer
)
```

`esp_hw_support` and `esp_driver_gptimer` are already in
`PRIV_REQUIRES` in the current `transmitter/main/CMakeLists.txt`;
both are needed by `rf/` and `nrf24/`. Drop the bare `main.c`-only
registration the skeleton ships.

### I.3 Kconfig (Transmitter submenu)

Create `main/Kconfig.projbuild` (the transmitter does not yet have
one):

```kconfig
menu "Transmitter Hub Firmware"

# 433 MHz transmit pin (same default as the receiver's RX, repurposed
# for output; the hub has no 433 MHz RX).
config TRANSMITTER_RF433_TX_GPIO
    int "433 MHz TX GPIO"
    default 4

# nRF24 SPI / control pins (mirror receiver defaults).
config TRANSMITTER_NRF24_SCK
    int "nRF24 SCK GPIO"
    default 6
config TRANSMITTER_NRF24_MOSI
    int "nRF24 MOSI GPIO"
    default 7
config TRANSMITTER_NRF24_MISO
    int "nRF24 MISO GPIO"
    default 2
config TRANSMITTER_NRF24_CS
    int "nRF24 CS GPIO"
    default 5
config TRANSMITTER_NRF24_CE
    int "nRF24 CE GPIO"
    default 3
config TRANSMITTER_NRF24_IRQ
    int "nRF24 IRQ GPIO"
    default 1

# nRF24 RF channel: f = 2400 MHz + N. Must match the receiver.
# The current receiver policy uses 97..125 to place nRF traffic above
# Wi-Fi channel 14's upper edge. The nRF24L01+ datasheet allows tuning
# through 2525 MHz; validate local rules before field deployment.
config TRANSMITTER_NRF24_CHANNEL
    int "nRF24 RF channel (97..125 -> 2497..2525 MHz; above Wi-Fi band)"
    default 100
    range 97 125

# Air data rate. Must match the receiver. 250 kbps is intentionally
# omitted (nRF24L01+ only).
choice TRANSMITTER_NRF24_DATA_RATE
    prompt "nRF24 air data rate"
    default TRANSMITTER_NRF24_DR_1M
config TRANSMITTER_NRF24_DR_1M
    bool "1 Mbps"
config TRANSMITTER_NRF24_DR_2M
    bool "2 Mbps"
endchoice

# Dynamic Payload Length — bilateral, must match the receiver.
config TRANSMITTER_NRF24_ENABLE_DPL
    bool "Enable Dynamic Payload Length"
    default y

# TX address is NOT a Kconfig — every dispatch carries its own
# destination address in `cmd/transmit.rf_code_hex` (= the receiver's
# rf_code, see Device Integration Plan §2.4). The hub writes TX_ADDR +
# RX_ADDR_P0 from those 5 bytes immediately before W_TX_PAYLOAD;
# nothing about the address is built into the firmware image.

# SoftAP & provisioning.
config TRANSMITTER_AP_CHANNEL
    int "SoftAP channel"
    default 1
config TRANSMITTER_AP_PASSWORD
    string "SoftAP password"
    default "setup1234"
config TRANSMITTER_WIFI_MAX_RETRY
    int "STA max retries before SoftAP fallback"
    default 6
config TRANSMITTER_BOOTSTRAP_TIMEOUT_MS
    int "Bootstrap timeout in milliseconds"
    default 120000

# Pinned backend command-signing key — same key as the receiver fleet.
config TRANSMITTER_BACKEND_PUBKEY_B64
    string "Backend command-signing public key (base64-DER EC P-256)"
    default ""
    help
        Base64-encoded DER-form EC P-256 public key used to verify
        backend-signed cmd/* payloads. Generate with the same OpenSSL
        recipe documented in the receiver's
        CONFIG_RECEIVER_BACKEND_PUBKEY_B64 help text; one signing key
        covers both fleets per §H.7.0.

endmenu
```

There is **no** `TRANSMITTER_RADIO_VARIANT` choice — both radios are
always built (§C.2).

### I.4 Target

```
idf.py set-target esp32c3
idf.py menuconfig       # set GPIOs, channel, address bytes, paste backend pubkey
idf.py build flash monitor
```

The current checkout has no `transmitter/sdkconfig`; `dependencies.lock`
records `target: esp32c3`, and `idf.py set-target esp32c3` is still
required for a fresh local config before menuconfig/build.

---

## J. Audit & Open Questions

A self-review pass. Each item is either **resolved** in the text
above or is an **accepted** trade-off noted for visibility.

1. **Band field on `transmit-v1` (resolved, §B.2 / §H.4).** Without
   an explicit band, `bits ∈ {8,16,24,32}` is ambiguous between
   433 MHz and 2.4 GHz. This guide pins the canonical and JSON
   extensions and makes them a hard prerequisite for §H.7.5.
2. **Signing-key reuse (resolved, §G.10 / §H.7.0).** One EC P-256
   signing key covers both fleets, so the hub pins the same key the
   receiver firmware does.
3. **No `PENDING_RF_CODE` for hubs (resolved, §A.1 / §A.2 / §H.7.2).**
   Backend sends a hub straight to `ACTIVE` after activation. The
   firmware's NVS `op_state` enum drops the receiver's intermediate
   state and the bootstrap flow never waits for a first
   `cmd/rf_code`.
4. **Radio probe failure surfacing (accepted, §F.5 / §H.10).** A
   boot-time nRF24 probe failure leaves the hub in 433-only mode and
   acks `band=2_4G` dispatches with `reason=no_2_4g_radio`. Surfacing
   the condition into the heartbeat (e.g. a `flags` bitfield) would
   let the admin UI flag a hardware fault without waiting for the
   first 2.4G dispatch attempt — deferred; the wire change is small
   but the backend has nowhere to put it today.
5. **Dispatch replay across reboots (resolved, §D / §G.8).** The
   replay ring is RAM-only because `cmd/transmit` is **not retained**
   (§E.1) and the broker re-delivers only un-acked QoS 1 messages on
   reconnect — so a reboot cannot resurrect a stale `dispatch_id`
   from broker state alone. Persisting the ring would buy nothing
   and add a wear-amplifying NVS write per dispatch. The ring records
   only signature-verified commands that reached a terminal ack
   outcome; invalid frames cannot reserve an ID, and ambiguous nRF24
   `tx_failed` duplicates cannot double-toggle a receiver.
6. **`cmd/transmit` while `SUSPENDED` / `DECOMMISSIONED` (resolved,
   §F.5).** The hub still parses + signature-verifies and acks
   `rejected` with a state-specific reason, instead of silently
   dropping. Keeps admin UX honest.
7. **No vibrator on the hub (resolved, §C / §C.1).** The hub never
   actuates anything locally; the vibrator-toggle behaviour belongs
   to the *receiver* on the other end. The hub's only output is RF.
8. **Mutex ordering (resolved, §G.2 / §G.8).** `dispatch_mtx` does
   not exist as a separate lock — `radio_mtx` in the supervisor is
   the single TX serialiser. Acks are always published *outside* the
   lock so broker latency never stretches the radio-busy window.
9. **DPL bilateral wire-format setting (resolved, §G.7 / §I.3).**
   Same constraint as Receiver §G.7.5. Defaulting
   `CONFIG_TRANSMITTER_NRF24_ENABLE_DPL=y` to match the receiver's
   `CONFIG_RECEIVER_NRF24_ENABLE_DPL=y` keeps pairs operating out of
   the box; flip both together to disable.
10. **`SETUP_RETR` ARD = 750 µs / ARC = 3 (resolved, §G.7).** The
    register write is `(2 << 4) | 3`, which encodes ARD field
    `0010` per the datasheet PS v1.0 SETUP_RETR table — i.e.
    750 µs, not 1000 µs. Three retransmits at 750 µs each add at
    most ~3 ms over a single shot, well under the 50 ms send-path
    timeout in `nrf24_tx_send`. The 750 µs floor satisfies the
    ACK-without-payload requirement at 1 Mbps and 2 Mbps; 250 kbps
    is intentionally not exposed (would require ARD ≥ 500 µs and is
    legacy-incompatible).
11. **Per-dispatch TX_ADDR + RX_ADDR_P0 (resolved, §B.3 / §G.7).**
    The rf_code-as-address model means the destination address is
    different per dispatch. `nrf24_tx_send` takes the 5-byte address
    from the caller and writes it to **both** `TX_ADDR` and
    `RX_ADDR_P0` immediately before `W_TX_PAYLOAD`. PTX auto-ACK
    requires the equality (Receiver §G.7.7), and the per-dispatch
    rewrite costs ~24 µs of SPI bus time — negligible against the
    `Tpd2stby` walk already in the budget. No address ever lives in
    hub firmware or hub Kconfig.
12. **Heartbeat QoS 0 (resolved, §E.3 / §E.5).** Liveness is a
    sliding window, not a fact. QoS 1 here would amplify broker
    write load without making the 30 s TTL more accurate.
13. **Per-store cap enforcement is backend-only (resolved, §H.1 /
    §H.7.1 / §H.7.7).** The backend pre-counts non-terminal hubs at
    registration and rejects with `hub_cap_reached` if `count >= 3`.
    The firmware surfaces that envelope as a clear local error and
    goes back into provisioning — no firmware change needed for the
    cap itself.
14. **Existing `tristate_to_*` kept (resolved, §G.6).** The PT2272
    tri-state encoder lives next to the bits encoder. Both are pure
    functions; carrying tri-state has no runtime cost when unused
    and avoids deleting working bench tooling.
15. **No HMAC eFuse path documented (accepted).** The final
    transmitter config must enable `CONFIG_NVS_ENCRYPTION=y` /
    `CONFIG_NVS_SEC_KEY_PROTECT_USING_HMAC=y`, but this checkout has
    no `transmitter/sdkconfig` yet. The eFuse HMAC key block must be
    programmed before the first write — same operator step as the
    receiver fleet.
16. **Power management for the nRF24 (resolved, §F.7).** The chip
    stays in Power-Down between dispatches; each `band=2_4G` send
    pays the `Tpd2stby` (≤ 4.5 ms) + `Tstby2a` (130 µs) walk once.
    Worst-case dispatch budget §B.3 absorbs this.
17. **`@ConditionalOnProperty` semantics (resolved, §H.6).** The
    feature gate uses a single `name` so the only load-bearing fact
    here is Spring Boot's documented rule that multiple entries in one
    `name` array are ANDed.
18. **MQTT v5 `noLocal` plus envelope guard (resolved, §H.2).** Paho
    v5 Java exposes `MqttSubscription.setNoLocal(true)`, so the
    transmitter bootstrap subscription can ask the broker not to loop
    back backend self-publishes. The explicit envelope-type drop stays
    in place as defensive validation if those backend-owned types ever
    arrive on the wildcard.
19. **Sinks.Many strategy for the dispatch broadcaster (resolved,
    §H.7.5).** `multicast().directBestEffort()` drops `onNext` only
    for slow subscribers, so one stuck listener never stalls
    dispatch. The receiver plan §D7a defines the broadcaster shape;
    this guide consumes it.
20. **No SSE / streaming acks (deferred).** v1 acks are single
    publishes per outcome. A future "transmit progress" stream is
    out of scope.
21. **No flash size / OTA pinning (deferred).** OTA / `app_update`
    is out of scope for v1; same trade-off the receiver guide
    accepts.
22. **Receiver `assigned_device_name` parity (resolved, §A.1).** The
    firmware persists `device_name` from the activation `result`
    payload (same field name as the receiver, same persistence
    pattern). Backend-side the field is `assigned_name`
    (Device Integration Plan §3.1); the wire field stays `assigned_device_name`
    for cross-fleet symmetry.
23. **Single-instance globals (accepted, §G.7).** The nRF24 driver
    keeps file-static `s_spi` / single-handle state, matching the
    receiver. Multi-radio HATs are not a v1 target.
24. **nRF24 channel bounds (resolved, §I.3).** The transmitter keeps
    the receiver's `97..125` range so paired devices share the same
    RF channel and sit above Wi-Fi channel 14's upper edge. The
    nRF24L01+ datasheet supports `RF_CH` through 125
    (`2400 + RF_CH` MHz, up to 2525 MHz), while the common 2.4 GHz ISM
    band statement is 2.400–2.4835 GHz; deployment therefore needs a
    local regulatory decision rather than a silent downgrade into the
    Wi-Fi band.
25. **433 MHz completion budget (resolved, §B.3 / §G.6).** The
    previous blanket `<250 ms to RF off-air` target only fits the nRF24
    path. 433 MHz off-air time is a function of protocol timing,
    bit-width, and `TX_REPEAT_COUNT`; the wrapper now waits on a
    calculated timeout derived from the generated pulse train.

---

## K. Self-Verification

Cross-checks performed during drafting and subsequent audit passes.

- **Backend topics, cadence, and cap.** `transmitter/...` topic list,
  QoS, retain flags, heartbeat interval (10 s) and liveness TTL
  (30 s), per-store cap (3) all match what the merged §H.1–§H.7
  pins. The cap rationale "ceiling, not floor" and the election
  tiebreak "lowest `device.id`" stay consistent across §H.1 and
  §H.7.4.
- **Bootstrap envelope shapes.** `pending`, `challenge`, `rejected`,
  `response`, `result` payload fields match the Receiver guide §E.3
  shapes.
- **`activate-v1` and `deact-v1` canonicals.** Reused unchanged from
  the receiver walkthrough §2.3 and receiver guide §E.3.
- **Receiver-side address ownership for nRF24 PTX.** `TX_ADDR ==
  RX_ADDR_P0 == receiver's RX_ADDR_P1` matches the table in Receiver
  §G.7.7. With the rf_code-as-address model, the destination
  receiver's `RX_ADDR_P1` is the bytes the backend mints as the
  receiver's `device_rf_code` (Device Integration Plan §2.4 — `bits == 40`),
  and the hub writes those exact bytes to its `TX_ADDR` and
  `RX_ADDR_P0` per dispatch.
- **DPL bilateral, retransmit retry, channel range, byte order.**
  DPL, retry, and byte-order rules pinned in §I.3 match the receiver's
  §G.7.5 design notes; deviations (TX-side EN_AA pipe, retransmit
  retry, MASK_RX_DR, per-dispatch address writes) are explicitly
  justified inline. Channel bounds intentionally match the receiver's
  `97..125` Wi-Fi-avoidance policy; the datasheet supports the silicon
  tuning range, but local regulatory compliance must be checked before
  field deployment.
- **433 MHz plaintext encoding.** "Big-endian, left-padded to
  ceil(bits/8), only bottom `bits` significant" matches Receiver
  Plan §3.1 schema comment. The bits-to-pulses encoder in §G.6
  walks MSB-first across exactly those `bits`, one pulse-pair per
  logic bit, terminated by the protocol's sync pair — matching what
  the receiver's RC-Switch decoder expects.
- **ESP-IDF API surfaces.** `esp_mqtt_client_publish` thread-safety,
  `gptimer` 1 MHz resolution, SPI `flags = 0` full-duplex, ISR IRAM
  rule, NVS `nvs_flash_init` + erase-retry, `esp_wifi_set_ps` post-
  GOT_IP — every surface is one already used by the receiver
  firmware in this repo (`receiver-esp32/main/`), and by the
  documented v6 conventions in
  `docs/walkthrough/ESP_IDF_SDK_REFERENCE.md`. No `transmitter/` or
  `receiver-esp32/` SDK-reference copy is checked in at this audit
  point.
- **Backend techstack — Context7-verified during merge.**
  - `@ConditionalOnProperty` (Spring Boot 3.5): when multiple
    `name` entries are listed on a single annotation, "all properties
    must pass the test for the condition to match" (AND semantics).
    The §H.6 / §H.7.0 feature gate uses a single `name`, so the
    distinction is recorded for future-proofing rather than load-
    bearing today. (Source: spring.io/spring-boot/3.5 API docs.)
  - Paho MQTT v5 Java client: `MqttSubscription.setNoLocal(true)` is
    available on the v5 client. The current repo wrapper still only
    exposes `subscribe(topicFilter, qos)`, so §H.2 / §H.7.0 explicitly
    add a v5 subscription overload before relying on `noLocal`.
    (Source: eclipse/paho.mqtt.java MQTTv5 reference plus current
    `backend/core/mqtt/MqttClientManager.kt`.)
  - Reactor Core `Sinks.many().multicast().directBestEffort()` is
    documented as the multicast strategy that drops `onNext` only
    for the specific slow subscriber, leaving healthy subscribers
    unaffected — exactly what `DeviceDispatchEventBroadcaster` (§H.7.5)
    needs. (Source: reactor/reactor-core docs/processors.adoc.)
- **Existing transmitter source.** `rf/rf_common.h`, `rf_data.c`,
  `rf_timer.c`, `rf_transmitter.c` are kept; the only additions are
  `bits_to_pulses` and the `rf433_tx_*` thin wrappers in §G.6 (no
  changes to the existing `gptimer` ISR or the protocol table). The
  current `main.c` is empty and must be replaced per §F.1.
- **Hub `op_state` set.** `{ACTIVE, SUSPENDED, DECOMMISSIONED}`
  matches §H.1 / §H.7.4 (`status NOT IN ('DECOMMISSIONED', 'REJECTED')`
  excluded from the candidate pool; `REJECTED` is a backend-only
  state that never reaches the firmware).
- **No Kconfig radio gate.** §C.2 / §I.3 — both radios are always
  built; `idf_component_register` lists both `rf/*.c` and
  `nrf24/nrf24_transmitter.c` unconditionally.
- **Backend-only content is fully absorbed.** Scope/topology, topic
  contract, canonicals, package additions, configuration, phases,
  execution order, cross-domain dependencies, and deferred backend
  items all live inside §H.1–§H.10. No separate backend transmitter
  plan remains.
- **Cross-references after merge.** Every internal `§H.x` anchor
  used in §A–§G and §J resolves to a heading present in this file.
  The receiver walkthrough now points to this guide for transmitter-
  side work, so no live doc link depends on a separate backend-prep
  plan.
