# Transmitter Hub Design Guide — ESP32-C3

Authoritative end-to-end plan for the **transmitter hub** across both
ends of the link: the **ESP32-C3 firmware** running on **ESP-IDF v6.0**
and the **backend slice** that registers, elects, and dispatches to it.
The hub ships **both radios in a single image**: **433 MHz ASK/OOK**
transmit *and* **nRF24L01 / nRF24L01+** 2.4 GHz transmit. Band
selection is a per-dispatch decision made on each inbound
`cmd/transmit`, not a Kconfig gate.

This guide is the standalone transmitter-hub implementation plan.
Sections A–G define firmware behavior and code structure; sections H–I
capture the backend contract the firmware depends on and the
firmware-side build configuration.

Everything load-bearing that the transmitter firmware reuses from
sibling receiver/backend docs has been copied or restated here. Related
repo docs may still exist, but no implementation step below requires
them.

Reference baselines:

- ESP-IDF v6.0 SDK reference: `docs/walkthrough/ESP_IDF_SDK_REFERENCE.md`
  (mirrored into `transmitter/ESP_IDF_SDK_REFERENCE.md` for Dev Container use)
- ESP-MQTT 1.0.0 headers: `transmitter/managed_components/espressif__mqtt/include/mqtt_client.h`
- nRF24L01+ PS v1.0: `docs/nrf24/nRF24L01P_PS_v1.0.txt`
- nRF24L01 PS v2.0 (legacy): `docs/nrf24/nRF24L01_Product_Specification_v2_0-9199.txt`

This document describes the **target end state**. The transmitter hub
firmware is implemented: dual-radio dispatch (433 MHz + nRF24), MQTT
command handling, ECDSA signature verification, bootstrap/provisioning,
heartbeat, and lifecycle management. The `main/rf/` module provides
bit-oriented 433 MHz ASK/OOK encoding via the RC-Switch protocol table.

**Last updated:** 2026-05-02

---

## A. System Overview

Scope:

1. SoftAP provisioning and local recovery over a small local HTTP API.
2. Backend registration plus challenge-response activation using the
   bootstrap envelopes and `activate-v1|...` canonical defined in §E.3.
3. Post-activation operation: subscribe to dispatch commands, broadcast
   the requested RF code on the requested band, and publish a periodic
   heartbeat so the per-store hub election (§H.7.4) sees the unit live.
4. Remote deactivation with the signed
   `deact-v1|<public_id>|<command_id>|<action>|<issued_at>` lifecycle
   command defined in §E.3.

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
                                     result    ─► persist public_id,
                                                 assigned_device_name → device_name,
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

There is no stored radio-variant field on this device. The hub always
carries both radios, so firmware never enters a compiled-vs-stored
variant mismatch recovery branch.

### A.3 Reprovision

Reprovision erases the `device_cfg` namespace and keeps `identity`.
The EC keypair therefore persists across reactivations, so the backend
can still recognize the physical hub. The next activation mints a fresh
`hub-XXXXX` `public_id`, orphaning any retained `cmd/deact` on the old
topic automatically.

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
`band`. This transmitter contract pins the following width rules:

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
  `byte_len = ceil(bits/8)`, only the bottom `bits` are significant),
  encode as a pulse train with the
  active RC-Switch-family protocol, repeat `TX_REPEAT_COUNT` times,
  release GPIO (§G.6). The rf_code bytes are an application-layer
  match value. There is no separate toggle payload on this path: the
  transmitted 433 MHz frame itself is what the receiver's MCU-side
  matcher decodes and compares against its stored rf_code.
- **2.4 GHz**: the dispatch's `rf_code_hex` is the destination
  receiver's 5-byte nRF24 address (`bits == 40`,
  `byte_len == 5`). The hub writes those 5 bytes to **both** `TX_ADDR`
  and `RX_ADDR_P0` (auto-ACK requires `RX_ADDR_P0 == TX_ADDR`),
  loads the firmware-defined fixed `TOGGLE_MAGIC = {0xAA, 0x55}`
  payload into the TX FIFO with `W_TX_PAYLOAD`, pulses `CE` ≥ 10 µs
  (datasheet §6.1.7 Thce), waits for `TX_DS` or `MAX_RT` via IRQ, and
  returns the chip to Power-Down (§G.7). The address is per-dispatch;
  no nRF24 address ever lives in hub firmware or hub Kconfig. This
  fixed payload exists only on the 2.4 GHz path.
- Stop semantics: a `DEVICE_STOP_REQUESTED` event becomes another
  `cmd/transmit` carrying the same `rf_code_hex`. The receiver's
  firmware toggles its actuator on every qualifying match — for 433M
  that means the same decoded-value comparison; for 2.4G that means
  another address-matched + magic-validated frame. The hub never emits
  a distinct "stop" frame and does not vary the payload by request
  kind.

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
│   ├── wifi.[ch]            STA + SoftAP + WPA3-compatible handling
│   ├── mqtt.[ch]            mqtts client + topic router + phase restart
│   ├── heartbeat.[ch]       periodic 10 s publisher (§G.9)
│   └── certs/mqtt_ca.pem    broker CA (embedded via EMBED_TXTFILES)
├── provision/
│   ├── http_server.[ch]
│   └── index.html[.gz]      embedded UI (gzip-embedded asset)
├── security/
│   └── device_identity.[ch] device EC P-256 + pinned backend pubkey
├── dispatch/
│   ├── dispatch.[ch]        cmd/transmit handler: verify, route, ack
│   └── radio_supervisor.[ch] start/suspend/resume/deinit per radio (§G.2)
├── rf/                      433 MHz bit-oriented pulse engine
│   ├── rf_common.h          bits_to_pulses + rf433_tx_send API
│   ├── rf_data.c            RC-Switch protocol table; bits-based encoder
│   ├── rf_timer.c           gptimer ISR for pulse timing
│   └── rf_transmitter.c     rf433_tx_send_bits wrapper
└── nrf24/
    ├── nrf24_regs.h         local register + command definitions for the
    │                        PTX driver, including `NRF_DYNPD_P0 (1U << 0)`
    │                        for the ACK-receiving pipe (§G.7)
    ├── nrf24_transmitter.h  PTX-shaped public C API (§G.7)
    └── nrf24_transmitter.c  SPI driver in PTX mode (§G.7)
```

Modules intentionally absent on the hub: `trigger/`,
`rf/rf_receiver.c`, `vibrator/`, and any local RF matcher. The hub
never listens for RF and never actuates locally. `dispatch/` is the
hub-side control entrypoint instead.

Coupling rule:

- `dispatch/` calls `rf/` (433 path) and `nrf24/` (2.4 path) through
  `dispatch/radio_supervisor.[ch]`. Neither radio module knows MQTT.
- `network/heartbeat` calls `network/mqtt` only.
- `network/mqtt` calls `dispatch/` on `cmd/transmit` and a lifecycle
  service on `cmd/deact`. It owns no radio knowledge.

### C.1 Pin map (default Kconfig)

ESP32-C3 exposes 22 GPIO numbers (0–21), but board/module wiring and
boot strapping still matter. Both radios are on the board
simultaneously, so the default map keeps the 433 MHz and nRF24
interfaces on separate pins without introducing a runtime mux:

| Signal             | Default GPIO | Notes                                 |
| ------------------ | ------------ | ------------------------------------- |
| `RF433_TX_GPIO`    | 4            | push-pull 433 MHz data out            |
| `NRF24_SPI_HOST`   | `SPI2_HOST`  | only general-purpose SPI host on C3   |
| `NRF24_SCK`        | 6            | SPI clock                             |
| `NRF24_MOSI`       | 7            | SPI MOSI                              |
| `NRF24_MISO`       | 2            | SPI MISO                              |
| `NRF24_CS`         | 5            | active-low CS                         |
| `NRF24_CE`         | 3            | chip-enable                           |
| `NRF24_IRQ`        | 1            | active-low TX_DS / MAX_RT IRQ         |

Avoid GPIO18/19 when USB-Serial-JTAG is needed. GPIO2 is acceptable
for SPI MISO only if the board keeps the required boot level and the
nRF24 MISO line is high-impedance while CSN stays high at reset. No
vibrator GPIO exists on this board; the hub has no local actuator.

### C.2 Radio gate

There is no Kconfig `RADIO_VARIANT` choice. Both radios are in every
build. A future "radio-down" Kconfig may exclude one driver to save
flash on a single-band board population, but v1 ships dual.

---

## D. NVS Schema

Two namespaces. Keys stay within the NVS 15-character limit.

**Namespace `identity`** — device EC keypair; survives reprovision.

| Key       | Type   | Notes                    |
| --------- | ------ | ------------------------ |
| `pubkey`  | `blob` | DER EC P-256 public key  |
| `privkey` | `blob` | DER EC P-256 private key |

**Namespace `device_cfg`** — runtime config plus activation state;
wiped on reprovision. The hub schema omits any per-device RF-code
storage:

| Key             | Type  | Written by         | Notes                                                                |
| --------------- | ----- | ------------------ | -------------------------------------------------------------------- |
| `schema_ver`    | `u8`  | provisioning       | `1`                                                                  |
| `wifi_ssid`     | `str` | provisioning       | presence = provisioned                                               |
| `wifi_pwd`      | `str` | provisioning       | station password                                                     |
| `mqtt_uri`      | `str` | provisioning       | must be `mqtts://...`                                                |
| `mqtt_user`     | `str` | provisioning       | shared broker credential                                             |
| `mqtt_pwd`      | `str` | provisioning       | shared broker credential                                             |
| `enroll_token`  | `str` | provisioning       | admin-issued single-use authorization; erased after activation or terminal bootstrap failure |
| `public_id`     | `str` | activation         | backend-minted `hub-XXXXX`                                           |
| `device_name`   | `str` | activation         | backend-assigned human label                                         |
| `op_state`      | `u8`  | activation + MQTT  | `ACTIVE`, `SUSPENDED`, `DECOMMISSIONED`                              |
| `last_deact_id` | `str` | `cmd/deact` MQTT   | most recent applied lifecycle `command_id` for replay suppression    |

Not present on the hub: `rx_type`, `rf_code`, `rf_code_bits`,
`rf_code_ver`. The hub never stores a per-device RF code.

`last_deact_id` suppresses lifecycle-command replays: if an inbound
`cmd/deact` carries the same `command_id` as the persisted
`last_deact_id`, the hub republishes the prior lifecycle ack and skips
all state changes.

**Dispatch-replay rule.** The hub does **not** persist `dispatch_id`s
to NVS — they are bursty and ephemeral. Replay suppression for
`cmd/transmit` runs in RAM only, with a small ring buffer of the last
N (default 8) signed commands that reached a terminal ack outcome
(`applied` or `rejected`). The ring stores the prior ack status/reason
and never re-emits RF on a duplicate. A reboot wipes
the ring; this is acceptable because broker reconnection re-delivers
only un-acked QoS 1 messages and `cmd/transmit` is **not retained**
(§E.1), so a stale broadcast cannot replay across a reboot.

Implementation rules:

- `public_id`, `device_name`, `op_state`, and `enroll_token` changes
  committed during activation must land in one `nvs_commit()`.
- `last_deact_id` must be committed before the lifecycle ack is
  published.
- Never log `mqtt_pwd`, `enroll_token`, private-key material, or signed
  dispatch plaintext.
- The checked-in config enables NVS encryption and HMAC-backed key
  protection; the namespace layout above does not change when encryption
  is on.

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

**Provisioning** (`POST /api/provision`):

```json
{
  "schema_version": 1,
  "wifi": { "ssid": "OfficeWiFi", "password": "super-secret" },
  "mqtt": {
    "broker_uri": "mqtts://broker.example.com:8883",
    "username": "shared-transmitter-user",
    "password": "shared-transmitter-password"
  },
  "enrollment": { "token": "<opaque string>" }
}
```

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
`TRANSMITTER_HUB`.

All bootstrap envelopes share the topic
`transmitter/bootstrap/{challenge_id}`. Direction is inferred from
`type`: `pending`, `challenge`, `rejected`, and `result` are
backend-originated; `response` is hub-originated. Both sides ignore
any envelope whose `type` they own.

**Pending** (`type: "pending"`):

```json
{
  "schema_version": 1,
  "type": "pending",
  "challenge_id": "6b99a4cb-2fc0-4559-8ae4-4ad65f8f8e0d",
  "registration_nonce": "hG7aK2pQcR",
  "issued_at": "2026-04-29T12:00:00Z"
}
```

**Rejected** (`type: "rejected"`):

```json
{
  "schema_version": 1,
  "type": "rejected",
  "challenge_id": "6b99a4cb-2fc0-4559-8ae4-4ad65f8f8e0d",
  "reason": "admin_rejected"
}
```

**Challenge** (`type: "challenge"`):

```json
{
  "schema_version": 1,
  "type": "challenge",
  "challenge_id": "6b99a4cb-2fc0-4559-8ae4-4ad65f8f8e0d",
  "registration_nonce": "hG7aK2pQcR",
  "nonce": "QzZ3eUNhWmVQWGhJd0V1TQ",
  "issued_at": "2026-04-29T12:00:00Z",
  "expires_at": "2026-04-29T12:05:00Z",
  "purpose": "activate-v1"
}
```

Canonical string signed by the hub:

```text
activate-v1|<challenge_id>|<nonce>|<issued_at>|<expires_at>
```

**Response** (`type: "response"`):

```json
{
  "schema_version": 1,
  "type": "response",
  "challenge_id": "6b99a4cb-2fc0-4559-8ae4-4ad65f8f8e0d",
  "signature_b64": "BASE64_SIGNATURE"
}
```

**Activation result** (`type: "result"`):

```json
{
  "schema_version": 1,
  "type": "result",
  "challenge_id": "6b99a4cb-2fc0-4559-8ae4-4ad65f8f8e0d",
  "status": "active",
  "public_id": "hub-7Q4KZ",
  "assigned_device_name": "Front Desk Transmitter Hub"
}
```

The hub persists `public_id`, stores `assigned_device_name` into the
local `device_name` field, sets `op_state = ACTIVE`, clears
`enroll_token`, then restarts MQTT so the operational session comes up
with `client_id = public_id`.

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
  "proto_any": true,
  "issued_at": "2026-04-25T10:00:00Z",
  "signature_b64": "BACKEND_ECDSA_SIGNATURE"
}
```

`proto_any` (boolean): when `true`, the hub randomizes the 433 MHz
transmission protocol across all 15 RC-Switch entries in the `proto[]`
table via `esp_random()`. When `false`, it uses Protocol 1
(PT2272-compatible, 350 µs base). Ignored for 2.4 GHz dispatches. The
backend populates this from `device.kind`: MCU receivers (`RECEIVER_433M`,
`RECEIVER_2_4G`) → `true`; passive receivers (`RECEIVER_433M_PASSIVE`) →
`false`. MCU receivers auto-detect any protocol with 60 % timing
tolerance in `recv_proto()`; PT2272 hardware decoders only respond to
their fixed oscillator timing.

Canonical string for signing:

```text
transmit-v1|<hub_public_id>|<dispatch_id>|<receiver_public_id>|<band>|<rf_code_hex>|<rf_code_bits>|<proto_any>|<issued_at>
```

`<proto_any>` is the literal string `true` or `false`.

Validation order on the hub:

1. JSON shape, `schema_version == 1`, all required fields present (including `proto_any`).
2. `band ∈ { "433M", "2_4G" }`.
3. Width per band:
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

```json
{
  "schema_version": 1,
  "action": "suspend",
  "command_id": "2f1c4b40-8b6b-4d17-9c43-2d4b98c7ce71",
  "issued_at": "2026-04-29T12:20:00Z",
  "signature_b64": "BACKEND_ECDSA_SIGNATURE"
}
```

Canonical string for lifecycle signing:

```text
deact-v1|<public_id>|<command_id>|<action>|<issued_at>
```

Legal actions are `suspend`, `resume`, and `decommission`. The firmware
applies the action to its own persistent state; there is no RF side
effect.

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

Two EC P-256 keypairs are in play:

1. **Hub keypair** — generated on first boot, stored in the `identity`
   NVS namespace, used only to sign the activation challenge. Public
   half is sent in the registration envelope.
2. **Backend command-signing keypair** — generated offline. The private
   half stays on the backend and signs every operational command. The
   public half is pinned in firmware via
   `CONFIG_TRANSMITTER_BACKEND_PUBKEY_B64`.

Common settings:

- Curve: `secp256r1` / `MBEDTLS_ECP_DP_SECP256R1`
- Signature algorithm: `SHA256withECDSA`
- Hub key storage: DER blobs in NVS
- Hub public key on wire: DER → base64
- Backend public key in firmware: base64-encoded DER, parsed once at
  boot
- Activation nonce: at least 16 random bytes before base64url

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
         persist public_id, assigned_device_name -> device_name,
         op_state=ACTIVE
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
power draw of any always-on receive mode and keeps the chips cool when
no dispatches are in flight.

### F.2 `cmd/transmit` handling

1. Parse, width-check, and signature-verify per §E.3.
2. If the `dispatch_id` is already in the replay ring, re-ack the
   prior terminal outcome and stop; do not emit RF.
3. If `op_state != ACTIVE`, remember the state-based reject in the
   replay ring, ack `rejected`, and stop.
4. Take `radio_mtx`. Two transmits cannot interleave.
5. Route by `band`:
   - `"433M"` → select protocol: if `proto_any` is `true`, pick a
     random index 0..14 via `esp_random() % PROTO_COUNT`; else use
     `PROTO_ID - 1` (Protocol 1, PT2272-compatible). Call
     `rf_trans_proto_select` then `rf433_tx_send_bits` (§G.6).
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

Lifecycle handling is simple on the hub because there is no RF-code
state to preserve. The topic namespace is `transmitter/hub/{public_id}`
and the only runtime side-effect beyond persistence is heartbeat
start/stop:

- `suspend` → `op_state = SUSPENDED`, `heartbeat_stop()`,
  persist `last_deact_id`, ack `ok`.
- `resume` → `op_state = ACTIVE`, `heartbeat_start()`, persist
  `last_deact_id`, ack `ok`.
- `decommission` → `op_state = DECOMMISSIONED`, `heartbeat_stop()`,
  persist `last_deact_id`, ack `ok`.
- `resume` after `DECOMMISSIONED` → ack `ignored`, no state change.
- `command_id == last_deact_id` → ack the prior result, no state
  change.

All NVS commits happen **before** the ack publish.

### F.4 Heartbeat task

A FreeRTOS task started by `heartbeat_start`, period 10 000 ms ± 500
ms jitter. It reads `g_cfg.public_id`, formats the JSON in §E.3, and
publishes via `esp_mqtt_client_publish(..., qos=0, retain=0)`. On
publish error (e.g. queue full because MQTT is reconnecting) the tick
is dropped silently — the next tick will retry. `heartbeat_stop`
deletes the task and closes the publish path.

### F.5 Failure and recovery cases

**Wi-Fi repeatedly fails**

- After the configured retry budget is exhausted, stop the STA attempt
  and return to SoftAP recovery with the existing credentials still
  intact. The recovery UI can then either retry or accept fresh Wi-Fi /
  MQTT / enrollment data.

**Pending envelope received**

- Confirm `registration_nonce` matches the in-RAM bootstrap session.
- Narrow the subscription from `transmitter/bootstrap/+` to the exact
  `transmitter/bootstrap/{challenge_id}` topic.
- Do not write NVS yet.

**Rejected envelope received**

- Wipe the in-RAM bootstrap session.
- Erase `enroll_token`.
- Unsubscribe from bootstrap and return to SoftAP recovery in place.

**Challenge or result timeout**

- Do not blindly republish registration because the token may already
  be consumed.
- After the bootstrap timeout expires, erase `enroll_token` and return
  to SoftAP recovery, requiring a fresh token or factory reset.

The hub's transmit path adds two more cases:

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
  once at `ERROR`. Surfacing this into the heartbeat is deferred
  (§H.10).

### F.6 SoftAP and local provisioning

- SSID: `TRANSMITTER-SETUP-<last_3_mac_bytes_hex>`
- `max_connection = 1`
- channel: `CONFIG_TRANSMITTER_AP_CHANNEL`
- password: `CONFIG_TRANSMITTER_AP_PASSWORD`
- auth mode: WPA3-compatible SoftAP (`authmode = WIFI_AUTH_WPA2_PSK`,
  `wpa3_compatible_mode = 1`)
- HTTP server runs only in SoftAP mode
- Endpoints:
  - `GET /` — provisioning UI
  - `GET /api/status` — local status summary
  - `POST /api/provision`
  - `POST /api/retry`
  - `POST /api/reset`

On successful `POST /api/provision`: validate fields, write NVS, return
HTTP 200, set the `PROV_DONE` event, and restart after a short delay.

### F.7 Power management

- `esp_wifi_set_ps(WIFI_PS_MIN_MODEM)` only after `IP_EVENT_STA_GOT_IP`
- MQTT keepalive stays at 120 seconds
- No deep sleep in v1; it breaks the low-latency control session model
- Do not call `esp_wifi_stop()` in the steady operational states
  `ACTIVE`, `SUSPENDED`, or `DECOMMISSIONED`

The hub adds one rule: nRF24 stays in `Power-Down` (`PWR_UP = 0`)
between dispatches. Each `band=2_4G` transmit walks the chip
`Power-Down → Standby-I → TX → Standby-I → Power-Down`, paying the
`Tpd2stby` (≤ 4.5 ms) and `Tstby2a` (130 µs) startup costs once per
dispatch. Dispatches are single-digit per minute in the worst-case
queue surge, so these costs are negligible against the dispatch
budget in §B.3.

---

## G. ESP-IDF v6 Code Guide

Snippets elide error handling. ESP-IDF API surfaces were validated
against `docs/walkthrough/ESP_IDF_SDK_REFERENCE.md`, the checked-in
ESP-MQTT header under `transmitter/managed_components/`, and Context7
for the current upstream v6 guidance.

### G.1 NVS

The transmitter uses the normal ESP-IDF NVS pattern:
`nvs_open("device_cfg", NVS_READWRITE, &h)`, `nvs_set_str`,
`nvs_set_u8`, `nvs_set_blob`, and `nvs_commit()`. First boot follows
the standard recovery path: `nvs_flash_init()` and, on
`ESP_ERR_NVS_NO_FREE_PAGES` or `_NEW_VERSION_FOUND`,
`nvs_flash_erase()` plus one retry. The checked-in `transmitter/sdkconfig`
already enables `CONFIG_NVS_ENCRYPTION=y` and
`CONFIG_NVS_SEC_KEY_PROTECT_USING_HMAC=y`; if this project later adds
`sdkconfig.defaults`, keep those settings pinned there too.

### G.2 Radio supervisor — dual-radio dispatch

The hub supervisor is a **switch on `band`**, not a Kconfig gate:

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
//                      nRF24 address. `bits` is always 40 and
//                      `byte_len` is always 5.
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
        // Width must be 40 / 5 bytes. Caller is expected to have
        // pre-validated this against the §E.3 width table.
        // nrf24_tx_send writes the address to TX_ADDR + RX_ADDR_P0
        // internally and sends the fixed TOGGLE_MAGIC.
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

### G.3 Wi-Fi — STA + SoftAP + WPA3-Compatible

ESP-IDF v6 requires `esp_netif` and separate event bases. The STA path
below uses the v6 WPA3-compatible flags explicitly: the current config
keeps `disable_wpa3_compatible_mode = 0`, so a WPA3-capable AP can
negotiate SAE while WPA2-only networks still work. PMF is still
configured through `pmf_cfg`; this guide treats it as the live control
surface and does not rely on any receiver-side shorthand.

```c
#include "esp_wifi.h"
#include "esp_netif.h"
#include "esp_event.h"

static void on_wifi(void *arg, esp_event_base_t base, int32_t id, void *data) { /* ... */ }
static void on_ip  (void *arg, esp_event_base_t base, int32_t id, void *data) { /* ... */ }

static esp_netif_t *s_sta_netif;
static esp_netif_t *s_ap_netif;
static bool        s_wifi_stack_ready;

static void wifi_init_common(void) {
    if (s_wifi_stack_ready) return;
    wifi_init_config_t init = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&init));
    ESP_ERROR_CHECK(esp_event_handler_register(WIFI_EVENT, ESP_EVENT_ANY_ID, on_wifi, NULL));
    ESP_ERROR_CHECK(esp_event_handler_register(IP_EVENT,   IP_EVENT_STA_GOT_IP, on_ip, NULL));
    s_wifi_stack_ready = true;
}

void wifi_start_sta(const device_config_t *cfg) {
    if (!s_sta_netif) s_sta_netif = esp_netif_create_default_wifi_sta();
    wifi_init_common();

    wifi_config_t sta_cfg = {
        .sta = {
            .scan_method        = WIFI_ALL_CHANNEL_SCAN,
            .threshold.authmode = WIFI_AUTH_WPA2_PSK,
            .sae_pwe_h2e        = WPA3_SAE_PWE_BOTH,
            .pmf_cfg            = { .required = false },
            .disable_wpa3_compatible_mode = 0,
            .failure_retry_cnt  = CONFIG_TRANSMITTER_WIFI_MAX_RETRY,
        },
    };
    strlcpy((char*)sta_cfg.sta.ssid,     cfg->wifi_ssid, sizeof sta_cfg.sta.ssid);
    strlcpy((char*)sta_cfg.sta.password, cfg->wifi_pwd,  sizeof sta_cfg.sta.password);

    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &sta_cfg));
    ESP_ERROR_CHECK(esp_wifi_start());
    // Caller waits for IP_EVENT_STA_GOT_IP, then calls
    // esp_wifi_set_ps(WIFI_PS_MIN_MODEM) per §F.1 step 9.
}

void wifi_start_softap(void) {
    if (!s_ap_netif) s_ap_netif = esp_netif_create_default_wifi_ap();
    wifi_init_common();

    uint8_t mac[6];
    esp_read_mac(mac, ESP_MAC_WIFI_STA);

    wifi_config_t ap_cfg = {
        .ap = {
            .channel              = CONFIG_TRANSMITTER_AP_CHANNEL,
            .password             = CONFIG_TRANSMITTER_AP_PASSWORD,
            .max_connection       = 1,
            .authmode             = WIFI_AUTH_WPA2_PSK,
            .sae_pwe_h2e          = WPA3_SAE_PWE_BOTH,
            .pmf_cfg              = { .required = false },
            .wpa3_compatible_mode = 1,
        },
    };
    int n = snprintf((char*)ap_cfg.ap.ssid, sizeof ap_cfg.ap.ssid,
                     "TRANSMITTER-SETUP-%02X%02X%02X", mac[3], mac[4], mac[5]);
    ap_cfg.ap.ssid_len = n;

    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_AP));
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_AP, &ap_cfg));
    ESP_ERROR_CHECK(esp_wifi_start());
}
```

Notes:

- `WIFI_IF_STA` / `WIFI_IF_AP` replace the legacy `ESP_IF_WIFI_*`.
- The STA and SoftAP helpers share one Wi-Fi driver instance.
  `esp_wifi_init()`, default netif creation, and event-handler
  registration are guarded so a same-boot STA→SoftAP fallback reuses
  the existing stack instead of initializing it twice.
- `failure_retry_cnt` only takes effect with `WIFI_ALL_CHANNEL_SCAN`;
  keep those two fields paired.
- `threshold.authmode = WIFI_AUTH_WPA2_PSK` is a minimum: WPA3 and
  mixed WPA2/WPA3 networks still qualify, while open/WEP do not.
- `esp_wifi_set_ps(WIFI_PS_MIN_MODEM)` is applied after association and
  IP acquisition.
- Do not call `esp_wifi_stop()` during `ACTIVE`, `SUSPENDED`, or
  `DECOMMISSIONED`; the hub must stay reachable for lifecycle commands.

### G.4 MQTT

The `esp_mqtt_client_config_t` fields are nested in ESP-MQTT 1.0.0, and
ESP-IDF v6 ships ESP-MQTT as a managed component. The CA PEM is
embedded via `EMBED_TXTFILES` and exposed as a NUL-terminated C string.

```c
#include "mqtt_client.h"

extern const uint8_t ca_pem_start[] asm("_binary_mqtt_ca_pem_start");

static esp_mqtt_client_handle_t s_client;

static void on_mqtt(void *arg, esp_event_base_t base, int32_t id, void *data) {
    esp_mqtt_event_handle_t e = data;
    switch ((esp_mqtt_event_id_t)id) {
    case MQTT_EVENT_CONNECTED:
        subscribe_current_phase();
        break;
    case MQTT_EVENT_DATA:
        // Check total_data_len/current_data_offset/data_len and reassemble
        // long payloads before JSON parsing.
        topic_router_dispatch(e);
        break;
    default:
        break;
    }
}

void mqtt_start(const device_config_t *cfg) {
    esp_mqtt_client_config_t mqtt_cfg = {
        .broker = {
            .address = {
                .uri = cfg->mqtt_uri,
            },
            .verification = {
                .certificate = (const char*)ca_pem_start,
                .certificate_len = 0,
            },
        },
        .credentials = {
            .username = cfg->mqtt_user,
            .client_id = cfg->has_public_id ? cfg->public_id : NULL,
            .set_null_client_id = false,
            .authentication = {
                .password = cfg->mqtt_pwd,
            },
        },
        .session = {
            .keepalive = 120,
            .protocol_ver = MQTT_PROTOCOL_V_5,
            .message_retransmit_timeout = 1000,
        },
        .network = {
            .reconnect_timeout_ms = 5000,
            .timeout_ms = 15000,
            .refresh_connection_after_ms = 5 * 60 * 1000,
        },
        .task = {
            .priority = 7,
            .stack_size = 8192,
        },
        .buffer = {
            .size = 2048,
        },
    };
    s_client = esp_mqtt_client_init(&mqtt_cfg);
    esp_mqtt_client_register_event(s_client, ESP_EVENT_ANY_ID, on_mqtt, NULL);
    esp_mqtt_client_start(s_client);
}
```

Notes:

- Reject `mqtt_uri` that does not start with `mqtts://` at config-load
  time.
- Keep the TCP port inside `mqtt_uri`; `broker.address.uri` takes
  precedence over host/port split fields.
- With `set_null_client_id = false`, a pre-activation `client_id = NULL`
  lets ESP-MQTT synthesize its default `ESP32_%CHIPID%`. After
  activation, reconnect with the minted `public_id`.
- For PEM data, keep `certificate_len = 0`; the managed component
  treats zero length plus NUL termination as the PEM/string form.
- `MQTT_EVENT_DATA` may fragment one message across multiple events;
  always check `total_data_len`, `current_data_offset`, and `data_len`
  before parsing.
- `esp_mqtt_client_publish` is thread-safe, and the managed-component
  header documents the MQTT APIs as callable from either a user task or
  the MQTT event callback where noted.
- The topic router has three transmitter-specific consumers:
  `transmitter/bootstrap/+` → `bootstrap_listener`,
  `transmitter/hub/{public_id}/cmd/transmit` → `dispatch_handler`,
  and `transmitter/hub/{public_id}/cmd/deact` → `lifecycle_handler`.

#### G.4.1 Post-activation MQTT restart

Do not switch from the bootstrap session to the operational session
with `esp_mqtt_client_reconnect()` alone: the broker URI and shared
credentials stay the same, but the `client_id` changes from the
pre-activation default to the backend-minted `public_id`. In ESP-MQTT
1.0 that means stop + destroy + init/start with the updated config.

The managed-component header also documents that
`esp_mqtt_client_stop()` cannot be called from the MQTT event handler,
so defer the restart to a control task or application event.

```c
typedef struct {
    char public_id[32];
    char assigned_device_name[64];
} activation_result_t;

static TaskHandle_t s_mqtt_ctl_task;

static void mqtt_restart_with_identity(const device_config_t *cfg) {
    if (s_client) {
        ESP_ERROR_CHECK(esp_mqtt_client_stop(s_client));
        esp_mqtt_client_destroy(s_client);
        s_client = NULL;
    }
    mqtt_start(cfg);
}

static void handle_activation_result(const activation_result_t *r) {
    strlcpy(g_cfg.public_id,   r->public_id, sizeof g_cfg.public_id);
    strlcpy(g_cfg.device_name, r->assigned_device_name, sizeof g_cfg.device_name);
    g_cfg.op_state = OP_STATE_ACTIVE;
    device_config_commit_activation(&g_cfg);

    xTaskNotifyGive(s_mqtt_ctl_task);
}

static void mqtt_ctl_task(void *arg) {
    for (;;) {
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);
        mqtt_restart_with_identity(&g_cfg);
    }
}
```

Right after activation, the hub drops the bootstrap session,
reconnects with `client_id = public_id`, and `subscribe_current_phase()`
switches from `transmitter/bootstrap/+` to
`transmitter/hub/{public_id}/cmd/#`. In the operational phase,
`subscribe_current_phase()` also starts or stops the heartbeat task
according to `op_state`.

### G.5 HTTP provisioning

The provisioning server is a normal ESP-IDF component. Its HTTP
contract is the one defined in §F.6: `GET /`, `GET /api/status`,
`POST /api/provision`, `POST /api/retry`, and `POST /api/reset`. The
full `idf_component_register(...)` call — including `EMBED_TXTFILES`,
`EMBED_FILES`, and `PRIV_REQUIRES` — is in §I.2.

`main.c` stays a plain ESP-IDF `app_main` entry point:

```c
#include "nvs_flash.h"
#include "esp_netif.h"
#include "esp_event.h"
#include "config/device_config.h"
#include "network/wifi.h"
#include "network/mqtt.h"
#include "dispatch/radio_supervisor.h"

void app_main(void) {
    ESP_ERROR_CHECK(nvs_init_or_recover());
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    device_config_load(&g_cfg);
    // ... §F orchestration ...
}
```

Here `nvs_init_or_recover()` is the thin §G.1 wrapper around
`nvs_flash_init()` plus the erase-and-retry path for
`ESP_ERR_NVS_NO_FREE_PAGES` / `_NEW_VERSION_FOUND`.

### G.6 433 MHz transmitter — extend the existing `rf/` module

`transmitter/main/rf/` implements bit-oriented 433 MHz ASK/OOK
encoding via `rf_common.h` / `rf_data.c` / `rf_timer.c` /
`rf_transmitter.c`. The pulse engine and `gptimer` ISR are
production-complete; `bits_to_pulses` encodes backend-dispatched
payloads as one pulse-pair per logic bit.

Within the `proto[]` table, protocol IDs **1..4** are the
PT2272 / pin-compatible subset: all four are non-inverted and keep the
same waveform shape (`sync 1:31`, `zero 1:3`, `one 3:1`), differing
only by base pulse length (`350`, `320`, `240`, `150` µs). `PROTO_ID`
defaults to `1` today. Protocol 5+ remain available in the RF engine
for other 433 MHz families but are outside the PT2272-compatible
contract this guide assumes for passive receivers.

Backend `cmd/transmit` for `band=433M` carries a flat binary payload
of `bits ∈ [1, 32]`. The hub transmits one pulse-pair per logic bit
(no tri-state encoding):

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
table) carry over unchanged. Legacy tri-state encoding functions
(`tristate_to_pulses`, `tristate_to_uint32`, `load_rf_chn_pulses`)
and their associated globals (`tristate_code`, `chn_count`,
`chn_2`/`chn_3`/`chn_4`) have been removed — the hub only uses
bit-oriented encoding via `bits_to_pulses`.

### G.7 nRF24L01 / nRF24L01+ transmitter — native ESP-IDF SPI driver

The hub uses the same low-level substrate on both nRF24L01 and
nRF24L01+: SPI mode 0, MSB-first within each byte, LSByte-first for
multi-byte address registers on the wire, one STATUS byte shifted out
during every command, and a small FEATURE-register probe that
distinguishes legacy vs plus silicon while unlocking FEATURE/DPL on the
legacy part.

Minimal shared definitions that must exist in `nrf24_regs.h` and the
driver's private helpers:

```c
typedef enum {
    NRF_CHIP_UNKNOWN = 0,
    NRF_CHIP_LEGACY,
    NRF_CHIP_PLUS,
} nrf24_variant_t;

#define NRF_REG_CONFIG      0x00
#define NRF_REG_EN_AA       0x01
#define NRF_REG_EN_RXADDR   0x02
#define NRF_REG_SETUP_AW    0x03
#define NRF_REG_SETUP_RETR  0x04
#define NRF_REG_RF_CH       0x05
#define NRF_REG_RF_SETUP    0x06
#define NRF_REG_STATUS      0x07
#define NRF_REG_RX_ADDR_P0  0x0A
#define NRF_REG_TX_ADDR     0x10
#define NRF_REG_DYNPD       0x1C
#define NRF_REG_FEATURE     0x1D

#define NRF_CMD_W_TX_PAYLOAD 0xA0
#define NRF_CMD_FLUSH_TX     0xE1
#define NRF_CMD_FLUSH_RX     0xE2
#define NRF_CMD_ACTIVATE     0x50
#define NRF_ACTIVATE_MAGIC   0x73

#define NRF_CFG_PWR_UP      (1U << 1)
#define NRF_CFG_CRCO        (1U << 2)
#define NRF_CFG_EN_CRC      (1U << 3)
#define NRF_CFG_MASK_MAX_RT (1U << 4)
#define NRF_CFG_MASK_TX_DS  (1U << 5)
#define NRF_CFG_MASK_RX_DR  (1U << 6)

#define NRF_RF_DR_1M        0x00
#define NRF_RF_DR_2M        (1U << 3)
#define NRF_RF_PWR_0        (3U << 1)
#define NRF_RF_LNA_HCURR    (1U << 0)

#define NRF_FEATURE_EN_DPL  (1U << 2)
#define NRF_DYNPD_P0        (1U << 0)

#define NRF_ST_RX_DR        (1U << 6)
#define NRF_ST_TX_DS        (1U << 5)
#define NRF_ST_MAX_RT       (1U << 4)

#define NRF_TPOR_MS         100
#define NRF_TPD2STBY_MS     10
```

`nrf_probe_chip()` must follow this sequence:

1. Write `FEATURE := EN_DPL`.
2. Read `FEATURE` back.
3. If it sticks, classify the chip as `NRF_CHIP_PLUS` and restore
   `FEATURE := 0`.
4. If it does not stick, issue `ACTIVATE` with magic byte `0x73`,
   retry the FEATURE write/read, and classify as `NRF_CHIP_LEGACY` if
   the second attempt succeeds.
5. If the second attempt still fails, return `NRF_CHIP_UNKNOWN`.

Private SPI helpers stay small and polling-based: `nrf_xfer()`,
`nrf_cmd()`, `nrf_rreg()`, `nrf_wreg()`, `nrf_rreg8()`, and
`nrf_wreg8()`. Every bringup/send snippet below assumes those helpers
already exist, and the IRQ ISR remains minimal: it only gives the
`tx_done` semaphore and leaves all SPI work to task context.

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

// Two-byte payload the hub sends to every 2.4G dispatch. Stored as a static
// const so the byte order on the wire is byte 0 = HI = 0xAA, then
// byte 1 = LO = 0x55.
#define NRF_TX_TOGGLE_MAGIC_HI 0xAA
#define NRF_TX_TOGGLE_MAGIC_LO 0x55
```

Bringup core:

```c
// nrf24/nrf24_transmitter.c — shared SPI/probe helpers omitted for brevity
static esp_err_t nrf_tx_bringup(nrf24_tx_t *h) {
    // CE low; power-on settle; SETUP_AW liveness probe + chip probe.
    // Shared with the receiver-side substrate; keep the sequence identical.

    // Configure for PTX with auto-ACK enabled on pipe 0 so the chip can
    // accept the ACK frame the PRX returns (the receiver firmware has
    // EN_AA = 0x02 = pipe 1 enabled). The only IRQ source we mask off is
    // RX_DR — a PTX never sources data packets, so an asserted RX_DR
    // would only ever be a stray ACK echo. TX_DS and MAX_RT both stay
    // unmasked: the ISR releases tx_done on either, and the send path
    // reads STATUS afterwards to disambiguate (datasheet PS v1.0 §6.5).
    nrf_wreg8(NRF_REG_CONFIG,
              NRF_CFG_MASK_RX_DR                          // mask RX_DR: PTX only receives
              | NRF_CFG_EN_CRC | NRF_CFG_CRCO);           // empty ACKs, no RX_DR needed
                                                           // PRIM_RX = 0, PWR_UP = 0

    nrf_wreg8(NRF_REG_EN_AA,      0x01);                   // pipe 0 auto-ACK only
    nrf_wreg8(NRF_REG_EN_RXADDR,  0x01);                   // pipe 0 enabled (for ACK)
    nrf_wreg8(NRF_REG_SETUP_AW,   0x03);                   // 5-byte addresses
    // ARD = 750 µs (datasheet ARD field 0010), ARC = 3 retransmits.
    // 250 µs already satisfies empty-ACK at 1/2 Mbps; 750 µs provides
    // margin and headroom if ACK payloads are ever enabled (datasheet
    // PS v1.0 §7.4.2). We intentionally omit 250 kbps.
    nrf_wreg8(NRF_REG_SETUP_RETR, (2 << 4) | 3);
    nrf_wreg8(NRF_REG_RF_CH,      CONFIG_TRANSMITTER_NRF24_CHANNEL);

#if CONFIG_TRANSMITTER_NRF24_DR_2M
    // NRF_RF_LNA_HCURR is documented as obsolete on the nRF24L01+ (writes
    // ignored) and helps RX sensitivity by ~+1.5 dB on the legacy
    // nRF24L01. We OR it into RF_SETUP so the ACK reception path benefits
    // on whichever silicon is populated.
    nrf_wreg8(NRF_REG_RF_SETUP, NRF_RF_DR_2M | NRF_RF_PWR_0 | NRF_RF_LNA_HCURR);
#else
    nrf_wreg8(NRF_REG_RF_SETUP, NRF_RF_DR_1M | NRF_RF_PWR_0 | NRF_RF_LNA_HCURR);
#endif

#if CONFIG_TRANSMITTER_NRF24_ENABLE_DPL
    nrf_wreg8(NRF_REG_FEATURE, NRF_FEATURE_EN_DPL);
    // PTX-only: DPL must be enabled on the ACK-receiving pipe (pipe 0).
    nrf_wreg8(NRF_REG_DYNPD,   NRF_DYNPD_P0);
#else
    nrf_wreg8(NRF_REG_FEATURE, 0x00);
    nrf_wreg8(NRF_REG_DYNPD,   0x00);
#endif

    // PTX address ownership: TX_ADDR == RX_ADDR_P0 == destination
    // receiver's RX_ADDR_P1 (= the destination receiver's 5-byte
    // rf_code). Per-dispatch addressing — the hub does not bake any
    // address at boot. `nrf24_tx_send` writes both registers from the
    // 5 bytes carried in `cmd/transmit.rf_code_hex` immediately before
    // each W_TX_PAYLOAD.

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
    // and PS v1.0 (4.5 ms with Ls = 90 mH).
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

The IRQ ISR gives the `tx_done` semaphore and does nothing else.
`MASK_RX_DR` in `CONFIG` ensures the IRQ pin only asserts on `TX_DS` or
`MAX_RT`, so one ISR body covers both outcomes; the send path inspects
`STATUS` afterwards to disambiguate.

DPL on the **TX side** is bilateral: with
`CONFIG_TRANSMITTER_NRF24_ENABLE_DPL=y`, the chip sends a
variable-length frame whose length matches the `W_TX_PAYLOAD` write.
The destination receiver must also have DPL enabled or it silently
CRC-rejects the frame. Deployments should therefore keep transmitter
and receiver DPL settings aligned.

Address byte order is fixed: byte index 0 is the LSByte on the wire,
byte 4 is the MSByte. The hub copies the 5 bytes from
`cmd/transmit.rf_code_hex` directly into the SPI payload of
`W_REGISTER(TX_ADDR, …)` and `W_REGISTER(RX_ADDR_P0, …)` per dispatch
with no in-firmware swap and no Kconfig involvement. The destination
receiver writes the same bytes into `RX_ADDR_P1`, so those registers
describe one shared on-air identity.

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
  helper as the activation and lifecycle paths defined in this guide.

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
        iso8601_now(ts, sizeof ts);
        int n = snprintf(payload, sizeof payload,
                         "{\"schema_version\":1,\"issued_at\":\"%s\"}", ts);
        (void)mqtt_publish(topic, payload, n, /*qos=*/0, /*retain=*/0);
        int jitter = (esp_random() % (2 * HB_JITTER_MS + 1)) - HB_JITTER_MS;
        vTaskDelay(pdMS_TO_TICKS(HB_INTERVAL_MS + jitter));
    }
    s_hb = NULL;
    vTaskDelete(NULL);
}

esp_err_t heartbeat_start(void) {
    if (s_hb) return ESP_OK;
    s_hb_run = true;
    return xTaskCreate(hb_task, "hb", 3072, NULL, 4, &s_hb) == pdPASS
           ? ESP_OK : ESP_ERR_NO_MEM;
}

esp_err_t heartbeat_stop(void) {
    if (!s_hb) return ESP_OK;
    s_hb_run = false;
    // Task self-deletes after the current delay tick; s_hb is NULLed
    // by the task itself before vTaskDelete so heartbeat_start sees a
    // clean state.
    return ESP_OK;
}
```

`mqtt_publish` is a thin wrapper around `esp_mqtt_client_publish`
that returns silently when `s_client == NULL` or the queue is full —
the same pattern used for transmitter ack publishes. A dropped tick
is a non-event; only sustained silence (≥ 30 s, §H.6) has operational
meaning.

### G.10 Device identity

The device-identity flow from §E.4 maps directly onto ESP-IDF's
mbedTLS APIs. Hardware SHA-256 and big-int acceleration are enabled by
default on ESP32-C3. Default Kconfig gates:

```kconfig
CONFIG_MBEDTLS_HARDWARE_SHA=y
CONFIG_MBEDTLS_HARDWARE_MPI=y
CONFIG_MBEDTLS_ECDSA_C=y
CONFIG_MBEDTLS_ECP_DP_SECP256R1_ENABLED=y
```

The pinned backend command-signing public key is supplied through
`CONFIG_TRANSMITTER_BACKEND_PUBKEY_B64`.

Performance budget (C3 @ 160 MHz):

- ECDSA verify on a `transmit-v1|...` canonical: 70–100 ms.
- ECDSA sign on the activation challenge: 100–150 ms (one-shot per
  activation).

Both are bounded by mbedTLS hardware acceleration and well within the
§B.3 dispatch budget.

---

## H. Backend Implementation Context

This section records the backend behavior the transmitter firmware
expects to exist around it: package additions, configuration, and the
phased rollout that turn the firmware contract above into a working
server.

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

Shared backend assumptions already in place:

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
in `core/device/DeviceCanonical.kt`.
UTF-8, no trailing newline, exact field order:

- `activate-v1|<challenge_id>|<nonce>|<issued_at>|<expires_at>` —
  reused unchanged for the transmitter bootstrap flow.
- `deact-v1|<public_id>|<command_id>|<action>|<issued_at>` — reused
  unchanged for the transmitter lifecycle topic family.
- `transmit-v1|<hub_public_id>|<dispatch_id>|<receiver_public_id>|<band>|<rf_code_hex>|<rf_code_bits>|<proto_any>|<issued_at>`
  — new, with the `band` and `proto_any` fields per §H.4 / §E.3.

Rules:

- `dispatch_id` is a UUID minted by the backend per dispatch event so
  the hub can dedupe replays.
- `rf_code_hex` is uppercase hex.
- `issued_at` is ISO-8601 UTC ending in `Z`.
- Integer fields are base-10 without prefixes.

The builder for `transmitV1(...)` ships with a golden-string unit
test pinned against the example at the end of §E.3.

### H.4 Required `transmit-v1` band field

The §B.2 width rules now make `bits` disjoint per band (40 only on
2.4G; 1..32 only on 433M), so `band` is no longer required for
disambiguation. It stays on the wire because:

- It lets the hub firmware route to the right radio without
  back-deriving from `bits` (a one-line `if (band == "433M")` is
  cheaper to read and audit than width-table reasoning).
- It makes the signed canonical string self-describing for audit
  and replay analysis without forcing the reader to remember the
  width-to-band mapping.
- It leaves room for future widths without re-cutting the wire
  format — for example, PT2272 dispatch codes are 24-bit
  (16-bit address + 8-bit channel, patched by the backend), and
  `band = "433M"` already disambiguates the 24-bit case.

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

The transmitter firmware already implements `transmit-v1` with `band`
and `proto_any` from its initial release, so these fields are part of
the contract from day one rather than a future `transmit-v2`.

### H.5 Backend package additions

Add the following transmitter-specific pieces to the backend's shared
`device` domain:

```text
backend/src/main/kotlin/com/thomas/notiguide/domain/device/
├── listener/
│   ├── TransmitterBootstrapListener.kt    # transmitter/bootstrap/{register,+}     (§H.7.1)
│   └── TransmitterOperationalListener.kt  # transmitter/hub/+/{heartbeat,ack}      (§H.7.3)
└── service/
    ├── TransmitterDispatchService.kt      # broadcaster consumer → cmd/transmit    (§H.7.5)
    └── TransmitterElectionService.kt      # active-hub election, Redis-cached      (§H.7.4)
```

Existing shared device-domain code is reused:

| Concern                                                              | Existing shared service                                                       |
| -------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| Enrollment tokens                                                    | `EnrollmentTokenService`                                                        |
| Bootstrap intake (validation, token consume, upsert, challenge mint) | `DeviceRegistrationService`                                                     |
| Approval / activation                                                | `DeviceApprovalService`, `DeviceActivationService`                              |
| Lifecycle commands                                                   | `DeviceLifecycleService`                                                        |
| MQTT publish                                                         | `DeviceMqttPublisher` (gains `publishTransmit` and a hub variant of `publishDeact` / `clearRetained`) |
| Public ID minting                                                    | `DevicePublicIdMinter` (mints `hub-XXXXX` based on `kind`)                      |
| Command signing                                                      | `DeviceCommandSigner` (one signing key covers both families)                    |

### H.6 Backend configuration

Add to `application.yaml` under the `device:` block:

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

Each phase is independently shippable and lands in order.

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

The signing key and `DeviceCommandSigningProperties` are reused
unchanged. One signing key covers both fleets.

#### H.7.1 Phase T1 — Bootstrap intake

- `TransmitterBootstrapListener` subscribes to
  `transmitter/bootstrap/register` and `transmitter/bootstrap/+`,
  with the self-echo drop rule described in §H.2.
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

- `DeviceActivationService.onResponse` can stay family-agnostic — the
  `activate-v1` string and the ECDSA verify path do not change for
  transmitter hubs.
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
- For each `ack_for = "deact"` frame: reuse the existing lifecycle-ack
  handling keyed by `command_id` / `public_id`; the transmitter path
  differs only by topic namespace and `TRANSMITTER_HUB` routing.

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
  `DeviceDispatchEventBroadcaster`. The broadcaster uses
  `Sinks.many().multicast().directBestEffort()`
  — fan out to coroutine-side consumers, drop on slow subscriber so
  one stuck listener never stalls dispatch.
- On `DEVICE_CALL_REQUESTED`:
  1. `electActive(event.storeId)`. If null, emit
     `DEVICE_DISPATCH_FAILED` through the queue-event path used by the
     queue page: widen backend `MqttPublisher.QueueEventType`, publish
     the queue MQTT event, and also broadcast the local SSE event so
     multi-instance and single-instance admin sessions behave the
     same.
  2. Decrypt the receiver's RF code (only point in the system that
     does so on the dispatch hot path).
  3. Build `transmit-v1|hub_public_id|dispatch_id|receiver_public_id|band|rf_code_hex|rf_code_bits|proto_any|issued_at`,
     where `band` and `proto_any` are derived from the receiver row's `kind` (§H.4 / §E.3),
     sign with the existing command-signing key.
  4. Publish on `transmitter/hub/{hubPublicId}/cmd/transmit`
     (QoS 1, not retained).
- On `DEVICE_STOP_REQUESTED`:
  - For the receiver vibrator-toggle model, the firmware design has
    the next matching RF frame toggle the vibrator off. So "stop" is
    just another `transmit-v1` with the same RF code — no new
    canonical string needed. Re-elect, sign, publish. If no active
    hub is available at dispatch time, emit the same
    `DEVICE_DISPATCH_FAILED` signal and leave queue-side reservation
    handling fail-closed until the operator retries.
- Transmit-ack ingestion lives in §H.7.3's operational-listener path.

#### H.7.6 Phase T6 — Lifecycle

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
- Dedicated transmitter admin-web screens beyond the shared device
  admin surface.
- Per-device MQTT credentials and broker ACLs.

### H.8 Suggested execution order

Land this slice after the shared device-domain groundwork is stable:

1. **H.7.0** — canonical helper, publisher extension, properties,
   feature gate.
2. **H.7.1 + H.7.2** — bootstrap intake + activation. Verifiable by a
   `mosquitto_pub` mock of the bootstrap flow against
   `transmitter/bootstrap/register`.
3. **H.7.3 + H.7.4** — heartbeat listener + election service.
   Verifiable by simulated heartbeats and `electActive` unit tests.
4. **H.7.5** — dispatch consumer. End-to-end test: issue a
   device-ticket via the queue dispatch UI, confirm
   `cmd/transmit` lands on the broker.
5. **H.7.6 + H.7.7** — lifecycle wiring, cap enforcement, admin-list
   count surface.

The feature flag stays `false` in dev/prod profiles until firmware
exists.

### H.9 Queue/admin integration surface

- The backend side exposes a shared `device` domain, a
  `DeviceDispatchEventBroadcaster` producer, the receiver RF-code
  shape consumed on the dispatch hot path, and the queue/admin routes
  that issue device-bound tickets.
- `GET /api/queue/admin/{storeId}/available-devices` returns a
  preflight envelope rather than a bare list:
  `devices`, `dispatchReady`, and optional `error`.
- When no live `ACTIVE` hub is available for the store, the preflight
  returns `dispatchReady = false` and `error = "no_active_transmitter"`.
  The queue page renders the dispatch action in a disabled state with
  that reason instead of posting blindly or hiding the surface.
- When dispatch cannot be completed after the ticket is already in a
  device-bound state, the queue page receives
  `DEVICE_DISPATCH_FAILED` through the same queue-event path used by
  the rest of the admin flow and leaves ticket state unchanged until
  an operator triages the failure.

### H.10 Deferred (backend-specific)

- Per-device MQTT credentials and broker ACLs.
- A dedicated transmitter admin screen with hub map / heartbeat
  trace.
- Transmit ack analytics emission (how many dispatches actually fired
  vs. failed silently).
- In-field rotation of the backend command-signing key.
- Surfacing `nrf24` probe failures into the heartbeat envelope as a
  capability bitmap so admin can flag a 433-only hub before it tries
  to dispatch a 2.4 G code (the runtime fallback is in §F.5).

---

## I. Build Configuration (firmware-side)

### I.1 `main/idf_component.yml`

The checked-in file already declares `idf >= 4.1.0`,
`espressif/mqtt`, and `espressif/cjson`. Tighten the IDF floor to the
v6 baseline used by the receiver and pin the MQTT range:

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
# destination address in `cmd/transmit.rf_code_hex` (= the destination
# receiver's 5-byte rf_code). The hub writes TX_ADDR +
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
        backend-signed cmd/* payloads. Generate from the backend
        signing key with:
        openssl pkey -in signing-key.pem -pubout -outform DER | base64 -w0
        One signing key covers both fleets per §H.7.0.

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

The checked-in `transmitter/sdkconfig` already targets `esp32c3`, and
`dependencies.lock` records `idf 6.0.0`. Run `idf.py set-target
esp32c3` when regenerating a fresh local config or when the local
`sdkconfig` has been removed or reset.
