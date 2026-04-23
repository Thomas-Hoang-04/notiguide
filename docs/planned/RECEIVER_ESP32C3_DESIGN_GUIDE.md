# Receiver Device Design Guide — ESP32-C3 Port

Authoritative end-to-end design guide for the **ESP32-C3 receiver**
firmware and its backend contract. The target firmware runs on
**ESP-IDF v6.0** and supports **dual radio** hardware
(433 MHz ASK receiver **or** 2.4 GHz `nRF24L01 / nRF24L01+`).

This guide is standalone for the C3 receiver line. The older
`RECEIVER_DESIGN_GUIDE.md` remains a historical ESP-01 reference only;
the normative C3 lifecycle, payloads, storage layout, backend behavior,
and ESP-IDF bindings are defined here.

Cross-refs:

- Historical reference: `RECEIVER_DESIGN_GUIDE.md`
- SDK reference: `ESP_IDF_SDK_REFERENCE.md`
- SDK root: `/opt/esp/idf/` (v6.0.0)
- ESP-MQTT headers for this repo: `managed_components/espressif__mqtt/include/mqtt_client.h`
- Datasheets in this repo (both referenced by §G.7):
  - `nRF24L01_Product_Specification_v2_0-9199.txt` — nRF24L01 PS v2.0 (legacy)
  - `nRF24L01P_PS_v1.0.txt` — nRF24L01+ PS v1.0

This document describes the **target end state** of the ESP32-C3 port.
The current repo snapshot is still transitional: `jgromes/radiolib`
remains in `main/idf_component.yml`, while `managed_components/` already
provides `espressif__mqtt` and `espressif__cjson`. Sections C–I describe
the intended post-port layout after §I.1 removes the temporary RadioLib
dependency.

---

## A. System Overview

Scope:

1. SoftAP provisioning and local recovery
2. Backend registration plus challenge-response activation
3. Post-activation operation: receive an RF trigger code from the backend,
   match decoded frames/packets against it, and toggle continuous vibrator
   pulsing on each qualifying match
4. Remote deactivation: suspend, resume, and decommission

Historical lineage note:

| Area           | ESP-01 (legacy)                                 | ESP32-C3 (this port)                                                                                             |
| -------------- | ----------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| SDK            | ESP8266_RTOS_SDK 3.x, `make`                    | ESP-IDF v6.0, `idf.py` / CMake                                                                                   |
| CPU            | Tensilica L106 80 MHz, 80 KB RAM                | RV32IMC 160 MHz, 400 KB SRAM                                                                                     |
| GPIO           | 2 usable (GPIO0, GPIO2)                         | 22 (GPIO0–21; avoid 18/19 = USB)                                                                                 |
| Wi-Fi security | WPA3 opt-in, PMF capable only                   | WPA3-SAE fully supported                                                                                         |
| Netif          | `tcpip_adapter`                                 | `esp_netif` (mandatory)                                                                                          |
| Event IDs      | `SYSTEM_EVENT_`*                                | `WIFI_EVENT` + `IP_EVENT` bases                                                                                  |
| MQTT config    | flat `.uri/.cert_pem/...`                       | nested `broker.*/credentials.*/session.`*                                                                        |
| Embedded files | `COMPONENT_EMBED_{TXT,}FILES` in `component.mk` | `EMBED_{TXT,}FILES` in `idf_component_register`                                                                  |
| Radios         | fixed 433 MHz ASK                               | **433 MHz ASK *or* nRF24L01 / nRF24L01+** (Kconfig-selected)                                                     |
| Language       | C                                               | Target end state: C-only `main/`, native ESP-IDF SPI `nRF24L01 / nRF24L01+` path, no RadioLib/Arduino in the final port |


The firmware stays a **single source tree**, but the radio choice is a
**build-time** Kconfig selection, not a runtime switch. In practice this
means one image per board population: the selected driver is compiled
in, and the unselected path stays out of the final image.

### A.1 Device lifecycle

```
boot ─► load NVS ─► wifi config missing?
                     ├─ yes ─► SoftAP + HTTP provisioning ─► POST config + token ─► reboot
                     └─ no  ─► STA Wi-Fi ─► MQTTS ─► ensure device keypair
                                      │
                                      ▼
                          public_id + op_state already stored?
                           ├─ yes ─► restore trigger state ─► subscribe cmd/# ─► dispatch op_state
                           └─ no  ─► enroll_token present?
                                     ├─ no  ─► SoftAP + HTTP recovery
                                     │        (fresh token or reprovision required)
                                     └─ yes ─► subscribe bootstrap/+ ─► publish register
                                               │
                                               ▼
                                     pending → narrow to bootstrap/{cid} → wait
                                               │
                              rejected ─► erase enroll_token ─► SoftAP recovery
                              challenge ─► sign response ─► wait result
                              result    ─► store public_id + device_name + rx_type,
                                           erase enroll_token, set op_state=PENDING_RF_CODE,
                                           restart MQTT with client_id=public_id
                                               │
                                               ▼
                                     subscribe receiver/device/{public_id}/cmd/# ─►
                                     first cmd/rf_code ─► store ─► ACTIVE
```

The same Wi-Fi + broker credentials are used for both bootstrap and
steady-state operation. What changes after activation is the topic scope
and MQTT `client_id`: the pre-activation session uses ESP-MQTT's
default `ESP32_%CHIPID%`, and the post-activation session is restarted
under the backend-minted `public_id`.

### A.2 Persistent state

One persisted `op_state` byte drives operational behavior:

| `op_state`        | RF task | Meaning                                        |
| ----------------- | ------- | ---------------------------------------------- |
| `PENDING_RF_CODE` | stopped | activated, waiting for first `cmd/rf_code`     |
| `ACTIVE`          | running | has RF code, matching and toggling vibrator pulsing |
| `SUSPENDED`       | paused  | backend requested temporary off                |
| `DECOMMISSIONED`  | deleted | backend requested permanent off                |

Derived, not stored:

- `UNPROVISIONED` = `wifi_ssid` key missing from NVS
- `PENDING_ACTIVATION` = `wifi_ssid` present, `op_state` missing, and
  `enroll_token` present
- `RECOVERY_REQUIRED` = `wifi_ssid` present, `op_state` missing, and
  `enroll_token` missing, or `rx_type` stored on flash disagrees with
  the compiled radio variant

Transient phases like "connecting Wi-Fi", "waiting for admin review",
and "waiting for challenge result" stay in RAM only.

### A.3 Reprovision

Reprovision is an action, not a top-level state: erase the `device_cfg`
namespace and reboot. The device comes back up `UNPROVISIONED`.

The `identity` namespace is **not** wiped, so the device EC keypair and
`public_key_b64` stay stable across reprovisioning. The backend can
therefore recognize the physical unit across tenants/sites if policy
allows it. The old `public_id` and its retained MQTT topics are dropped;
the next activation mints a fresh `public_id`, so prior retained
commands become orphaned automatically.

---

## B. Dual Radio Model

### B.1 Receiver types (backend-visible)

- `RECEIVER_433M` → 433 MHz ASK/OOK, GPIO edge-interrupt decoding.
- `RECEIVER_2_4G` → nRF24L01 / nRF24L01+ over SPI, payload-byte matching.

The device reports its active variant in the registration envelope
(§E.3). No extra envelope type is needed.

### B.2 Trigger-code shape

The trigger payload keeps a single logical code field per device, but
the 2.4 GHz path widens its legal size. `nRF24L01 / nRF24L01+` packets
are **byte strings up to 32 B** (256 bits), so the wire shape becomes:

- `rf_code_hex`: hex string, length in {2, 4, …, 64} (1–32 bytes).
- `rf_code_bits`: `1..32` for `RECEIVER_433M`, `{8,16,…,256}` (byte-aligned)
for `RECEIVER_2_4G`.

Backend validation must reject mismatched combinations before publish
(for example `bits=64` with `RECEIVER_433M`, or non-byte-aligned `bits`
with `RECEIVER_2_4G`). The canonical string for signing
(`rf-code-v1|…`, §E.3) is identical across both radio families: same
field names, same order, same encodings.

### B.3 Matching semantics

- `RECEIVER_433M`: decode a 1–32 bit value from timing; mask to `bits`;
  compare bitwise.
- `RECEIVER_2_4G`: on each received packet, `memcmp` the first
  `bits/8` bytes against the stored code. Packet length ≥ `bits/8` is
  required; extra bytes are ignored so fixed-length senders remain
  interoperable.

A match toggles continuous vibrator pulsing. The next qualifying match
toggles it back off.

---

## C. Target Module Layout

```
main/
├── main.c                  orchestrator
├── config/
│   └── device_config.[ch]  NVS I/O, schema, reprovision
├── network/
│   ├── wifi.[ch]           STA + SoftAP + WPA3 + modem PS
│   └── mqtt.[ch]           mqtts client + topic router (esp_mqtt v6 config)
├── provision/
│   ├── http_server.[ch]
│   └── index.html[.gz]     embedded UI (gzip-embedded asset)
├── security/
│   └── device_identity.[ch] device EC P-256 + pinned backend pubkey
├── trigger/
│   ├── rf_trigger.[ch]     stores code, matches frames, toggles vibrator
│   └── rf_supervisor.[ch]  dispatches start/suspend/resume/delete per variant
├── rf/                     existing 433 MHz decoder (kept, .c)
├── nrf24/
│   ├── nrf24_regs.h        register/command defines (datasheet §9)
│   ├── nrf24_receiver.h    public C API (matches rf_recv_* shape)
│   └── nrf24_receiver.c    SPI driver + ISR + RX task (pure C)
├── vibrator/               existing pulse generator (kept, .c)
└── network/certs/mqtt_ca.pem  broker CA (embedded via EMBED_TXTFILES)
```

This is the intended post-port layout. The current repo snapshot is
smaller (`main.c`, `rf/`, `vibrator/`) and still carries temporary
component scaffolding until the rest of this plan lands.

The target end state keeps `main/` C-only. The native
`nRF24L01 / nRF24L01+` path talks to the chip through the ESP-IDF SPI
master API directly (§G.7); the
temporary RadioLib dependency in the current tree is removed in §I.1.

Coupling rule: `rf/` and `nrf24/` and `vibrator/` do not
depend on each other. `rf/` or `nrf24/` call into `trigger/rf_trigger`
on each decoded frame; `trigger/` calls into `vibrator/` on match. The
supervisor is a four-line dispatcher (§G.2).

### C.1 Pin map (default Kconfig)

ESP32-C3 has enough GPIO to avoid the ESP-01 2-pin tax. Defaults:


| Signal           | Default GPIO | Notes                   |
| ---------------- | ------------ | ----------------------- |
| `VIBRATOR_GPIO`  | 10           | push-pull out           |
| `RF433_RX_GPIO`  | 4            | edge-IRQ input, pull-up |
| `NRF24_SPI_HOST` | `SPI2_HOST`  | fixed on ESP32-C3       |
| `NRF24_SCK`      | 6            | SPI clock               |
| `NRF24_MOSI`     | 7            | SPI MOSI                |
| `NRF24_MISO`     | 2            | SPI MISO                |
| `NRF24_CS`       | 5            | active-low CS           |
| `NRF24_CE`       | 3            | chip-enable             |
| `NRF24_IRQ`      | 1            | active-low packet IRQ   |


All GPIO rows above are `Kconfig` symbols (`CONFIG_RECEIVER_*_GPIO`);
the SPI host is fixed to `SPI2_HOST` because ESP32-C3 exposes only one
general-purpose SPI host. Avoid GPIO18/19 (USB-Serial-JTAG).
ESP32-C3 strapping pins are **GPIO2, GPIO8, GPIO9** (unlike ESP32
classic, GPIO0 is *not* a strap on the C3). The defaults intentionally
avoid GPIO8/GPIO9. GPIO2 is acceptable for SPI MISO because nRF24 MISO
is tri-stated while CSN stays high at reset. If you remap onto GPIO8 or
GPIO9, add the required external pull resistors and re-audit
boot/download behavior.

### C.2 Radio-variant gate

```kconfig
choice RECEIVER_RADIO_VARIANT
    prompt "Active radio"
    default RECEIVER_RADIO_433M
config RECEIVER_RADIO_433M
    bool "433 MHz ASK/OOK receiver"
config RECEIVER_RADIO_2_4G
    bool "nRF24L01 / nRF24L01+ 2.4 GHz"
endchoice
```

The chosen symbol decides which module the supervisor starts and
which string the device sends in `receiver_type` at registration.
The codebase supports both drivers, but each build includes only the
selected path in the final image.

---

## D. NVS Schema

Two namespaces. Keys stay within the NVS 15-character limit.

**Namespace `identity`** — device EC keypair; survives reprovision.

| Key       | Type   | Written by | Notes                    |
| --------- | ------ | ---------- | ------------------------ |
| `pubkey`  | `blob` | key gen    | DER EC P-256 public key  |
| `privkey` | `blob` | key gen    | DER EC P-256 private key |

The device does **not** self-assign an operational identifier. Before
activation it uses the last three bytes of its STA MAC only for local
labels such as the SoftAP SSID and status page.

**Namespace `device_cfg`** — runtime config plus activation state; wiped
on reprovision.

| Key             | Type           | Written by         | Notes                                                                                                       |
| --------------- | -------------- | ------------------ | ----------------------------------------------------------------------------------------------------------- |
| `schema_ver`    | `u8`           | provisioning       | `1`; bump only if one firmware must migrate older C3 flash layouts                                          |
| `wifi_ssid`     | `str`          | provisioning       | presence = provisioned                                                                                      |
| `wifi_pwd`      | `str`          | provisioning       | station password                                                                                            |
| `mqtt_uri`      | `str`          | provisioning       | must be `mqtts://...`                                                                                       |
| `mqtt_user`     | `str`          | provisioning       | shared broker credential                                                                                    |
| `mqtt_pwd`      | `str`          | provisioning       | shared broker credential                                                                                    |
| `enroll_token`  | `str`          | provisioning       | erased after successful activation or terminal bootstrap failure                                            |
| `public_id`     | `str`          | activation         | backend-minted operational identity                                                                         |
| `device_name`   | `str`          | activation         | backend-assigned human label                                                                                |
| `rx_type`       | `u8`           | activation         | `0=RECEIVER_433M`, `1=RECEIVER_2_4G`; copied from the compiled Kconfig variant                              |
| `rf_code`       | `blob` (≤32 B) | `cmd/rf_code` MQTT | active trigger code; for 433 MHz this stores the little-endian 32-bit value in the first 4 bytes           |
| `rf_code_bits`  | `u8`           | `cmd/rf_code` MQTT | `1..32` for 433 MHz; `{8,16,…,256}` for 2.4 GHz                                                            |
| `rf_code_ver`   | `u32`          | `cmd/rf_code` MQTT | monotonic per `public_id`                                                                                   |
| `op_state`      | `u8`           | activation + MQTT  | `PENDING_RF_CODE`, `ACTIVE`, `SUSPENDED`, `DECOMMISSIONED`                                                  |
| `last_deact_id` | `str`          | `cmd/deact` MQTT   | most recent applied deactivation `command_id` for replay suppression                                        |

Rules:

- `rx_type` is written on activation and must match the compiled radio
  variant on every subsequent boot. If stored `rx_type` and Kconfig
  disagree, firmware enters `RECOVERY_REQUIRED` and raises SoftAP.
- `rf_code` is a blob, not a `u32`. For `RECEIVER_433M`, the first
  four bytes hold the little-endian value and only the low
  `rf_code_bits` are significant. For `RECEIVER_2_4G`, the first
  `rf_code_bits/8` bytes hold the packet prefix.
- `rf_code`, `rf_code_bits`, and `rf_code_ver` are committed under one
  `nvs_commit()` transaction to avoid torn updates.
- The first activation commit writes `public_id`, `device_name`,
  `rx_type`, `op_state`, and erases `enroll_token` in one durable step.
- Never log `rf_code`, `mqtt_pwd`, `enroll_token`, or the private key.
- Reprovision wipes `device_cfg` only. Because operational topics are
  scoped by `public_id`, old retained broker state becomes orphaned
  automatically after the next activation.

Tiny implementation note: the current ESP32-C3 repo config enables NVS
encryption on the default `nvs` partition
(`CONFIG_NVS_ENCRYPTION=y`) with HMAC-backed key protection
(`CONFIG_NVS_SEC_KEY_PROTECT_USING_HMAC=y`). This does not change the
`identity` / `device_cfg` namespace layout above; it only changes the
storage-at-rest handling and requires the configured HMAC eFuse key
block during provisioning.

---

## E. MQTT Contract

### E.1 Topics

| Topic                                     | Direction        | QoS | Retained | Notes                                                                         |
| ----------------------------------------- | ---------------- | --- | -------- | ----------------------------------------------------------------------------- |
| `receiver/bootstrap/register`             | device → backend | 1   | no       | public key + receiver type + token + registration nonce                       |
| `receiver/bootstrap/{challenge_id}`       | bidirectional    | 1   | no       | envelope `type` values: `pending`, `challenge`, `rejected`, `result`, `response` |
| `receiver/device/{public_id}/cmd/rf_code` | backend → device | 1   | **yes**  | trigger code; reused for rotation                                             |
| `receiver/device/{public_id}/cmd/deact`   | backend → device | 1   | **yes**  | suspend / resume / decommission                                               |
| `receiver/device/{public_id}/ack`         | device → backend | 1   | no       | single ack stream; envelope `ack_for` discriminator                           |

Design notes:

- Retained state exists only on the two operational command topics:
  one retained `rf_code`, one retained `deact`.
- Acks collapse into one upstream topic and use `ack_for` to separate
  RF-code acks from deactivation acks.
- The C3 firmware uses MQTT v5 transport, but the topic contract stays
  intentionally compatible with MQTT 3.1.1 semantics. The device does
  **not** rely on MQTT 5-only subscription options such as `no-local`,
  so bootstrap still drops self-owned envelope types explicitly.
- During bootstrap the device starts on `receiver/bootstrap/+`,
  filters by `registration_nonce`, then narrows to the exact
  `receiver/bootstrap/{challenge_id}` topic.

### E.2 Security

- All MQTT traffic is `mqtts://` only. Provisioning rejects non-TLS
  URIs and firmware refuses to connect if `mqtt_uri` does not start
  with `mqtts://`.
- The broker must present a certificate that chains to the embedded
  CA bundle/PEM.
- v1 uses one shared MQTT credential for all receivers. Isolation
  therefore rests on TLS, unguessable `public_id`s, and signed
  operational commands rather than per-device broker ACLs.
- `cmd/rf_code` and `cmd/deact` are always verified against the
  firmware-pinned backend public key before any state change.
- RF codes, MQTT passwords, and enrollment tokens are sensitive and are
  never logged or echoed in acknowledgements.
- Every inbound payload is validated for `schema_version`, shape, and
  mandatory fields before side effects.

### E.3 Payloads

All wire payloads carry `"schema_version": 1`.

**SoftAP provisioning** (`POST /api/provision`):

```json
{
  "schema_version": 1,
  "wifi": { "ssid": "OfficeWiFi", "password": "super-secret" },
  "mqtt": {
    "broker_uri": "mqtts://broker.example.com:8883",
    "username": "shared-receiver-user",
    "password": "shared-receiver-password"
  },
  "enrollment": { "token": "<opaque string>" }
}
```

**Registration** (`receiver/bootstrap/register`):

```json
{
  "schema_version": 1,
  "hardware_model": "ESP32-C3",
  "receiver_type": "RECEIVER_433M",
  "firmware_version": "dev",
  "public_key_b64": "BASE64_DER_PUBLIC_KEY",
  "enrollment_token": "<opaque string>",
  "registration_nonce": "hG7aK2pQcR"
}
```

- `receiver_type` is `RECEIVER_433M` or `RECEIVER_2_4G`.
- `registration_nonce` is device-generated, base64url, session-scoped,
  and at least 8 random bytes before encoding.
- The device sends no self-minted operational identifier. The backend
  mints `public_id` only after activation succeeds.

All bootstrap envelopes share the topic
`receiver/bootstrap/{challenge_id}`. Direction is inferred from
`type`: `pending`, `challenge`, `rejected`, and `result` are
backend-originated; `response` is device-originated. Both sides ignore
any envelope whose `type` they own.

**Pending** (`type: "pending"`):

```json
{
  "schema_version": 1,
  "type": "pending",
  "challenge_id": "6b99a4cb-2fc0-4559-8ae4-4ad65f8f8e0d",
  "registration_nonce": "hG7aK2pQcR",
  "issued_at": "2026-04-16T12:00:00Z"
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
  "issued_at": "2026-04-16T12:00:00Z",
  "expires_at": "2026-04-16T12:05:00Z",
  "purpose": "activate-v1"
}
```

Canonical string signed by the device (UTF-8, no trailing newline):

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
  "public_id": "rcv-7Q4KZ",
  "assigned_device_name": "Front Gate Receiver"
}
```

The device persists `public_id`, `assigned_device_name`, `rx_type`, and
`op_state=PENDING_RF_CODE`, clears `enroll_token`, then restarts MQTT so
the operational session comes up immediately with
`client_id = public_id` and the operational topic subscription set.

**RF code** (`receiver/device/{public_id}/cmd/rf_code`, retained):

```json
{
  "schema_version": 1,
  "rf_code_hex": "C35A5A5AE7",
  "rf_code_bits": 40,
  "code_version": 4,
  "issued_at": "2026-04-19T10:15:00Z",
  "signature_b64": "BACKEND_ECDSA_SIGNATURE"
}
```

- `RECEIVER_433M`: `rf_code_bits ∈ [1, 32]` and
  `hex_len == 2 * ceil(bits / 8)`.
- `RECEIVER_2_4G`: `rf_code_bits ∈ [8, 256]`,
  `rf_code_bits % 8 == 0`, and `hex_len == bits / 4`.
- `code_version` is monotonic within one `public_id`.
- The backend signs the canonical string below; the device rebuilds it
  byte-for-byte before accepting the command.

Canonical string for RF-code signing:

```text
rf-code-v1|<public_id>|<code_version>|<rf_code_hex>|<rf_code_bits>|<issued_at>
```

**Deactivation command** (`receiver/device/{public_id}/cmd/deact`,
retained):

```json
{
  "schema_version": 1,
  "action": "suspend",
  "command_id": "2f1c4b40-8b6b-4d17-9c43-2d4b98c7ce71",
  "issued_at": "2026-04-19T10:20:00Z",
  "signature_b64": "BACKEND_ECDSA_SIGNATURE"
}
```

Canonical string for deactivation signing:

```text
deact-v1|<public_id>|<command_id>|<action>|<issued_at>
```

**Single ack stream** (`receiver/device/{public_id}/ack`):

```json
{
  "schema_version": 1,
  "ack_for": "rf_code",
  "code_version": 4,
  "status": "applied",
  "applied_at": "2026-04-19T10:15:02Z"
}
```

```json
{
  "schema_version": 1,
  "ack_for": "deact",
  "command_id": "2f1c4b40-8b6b-4d17-9c43-2d4b98c7ce71",
  "action": "suspend",
  "status": "ok",
  "applied_at": "2026-04-19T10:20:01Z"
}
```

- For `ack_for = "rf_code"`, `status` is `applied`, `unchanged`, or
  `rejected`.
- For `ack_for = "deact"`, `status` is `ok`, `ignored`, or `rejected`.
- Acks never echo the RF code itself.

### E.4 Cryptography

Two EC P-256 keypairs are in play:

1. **Device keypair** — generated on first boot, stored in the
   `identity` NVS namespace, used only to sign the activation
   challenge. Public half is sent in the registration envelope.
2. **Backend command-signing keypair** — generated offline. The private
   half stays on the backend and signs every operational command. The
   public half is pinned in firmware via
   `CONFIG_RECEIVER_BACKEND_PUBKEY_B64`.

Common settings:

- Curve: `secp256r1` / `MBEDTLS_ECP_DP_SECP256R1`
- Signature algorithm: `SHA256withECDSA`
- Device key storage: DER blobs in NVS
- Device public key on wire: DER → base64
- Backend public key in firmware: base64-encoded DER, parsed once at boot
- Activation nonce: at least 16 random bytes before base64url

### E.5 Identifiers and topic invalidation

- `public_id` is backend-minted, opaque to the device, and scopes all
  operational topics.
- `challenge_id` and `command_id` are backend-generated UUIDs.
- `registration_nonce` is device-generated and binds the challenge back
  to the registration that triggered it.
- `enrollment_token` is single-use, short-lived, and opaque to the
  device.
- Because `public_id` is re-minted on every activation, retained
  commands on old operational topics are orphaned automatically.
- `code_version` is monotonic only within one `public_id`.
- Broker-side retained clears on old topics are recommended hygiene,
  not required for correctness.

---

## F. Boot Orchestration and Runtime Cases

### F.1 `app_main` orchestration

```
1. nvs_init_or_recover() + esp_netif_init() + esp_event_loop_create_default()
2. device_config_load() -> in-RAM config
   - if stored rx_type mismatches compiled Kconfig variant:
       enter RECOVERY_REQUIRED and raise SoftAP
   - if stored rf_code_bits != 0:
       seed rf_trigger from NVS
3. if no wifi_ssid:
     wifi_start_softap() + http_server_start()
     wait PROV_DONE -> esp_restart()
4. wifi_start_sta()  [blocks on IP_EVENT_STA_GOT_IP]
5. device_identity_ensure()  [generate/load EC keypair]
6. mqtt_start(&g_cfg)
   - pre-activation client_id = ESP32_%CHIPID%
   - post-activation client_id = public_id
7. if public_id / op_state absent:
     if enroll_token missing:
       wifi_start_softap() + http_server_start()
       wait new provision, retry, or factory reset
     else:
       generate registration_nonce
       subscribe receiver/bootstrap/+
       publish receiver/bootstrap/register
       on matching pending: narrow to receiver/bootstrap/{challenge_id}
       on challenge: sign and publish response
       on result:
         persist public_id + device_name + rx_type + op_state=PENDING_RF_CODE
         erase enroll_token
         restart MQTT with client_id=public_id
8. after IP_EVENT_STA_GOT_IP:
     esp_wifi_set_ps(WIFI_PS_MIN_MODEM)
9. subscribe receiver/device/{public_id}/cmd/#
10. dispatch on op_state:
      PENDING_RF_CODE -> wait for first cmd/rf_code
      ACTIVE          -> rf_sup_start()
      SUSPENDED       -> leave RF idle until signed resume
      DECOMMISSIONED  -> stay on Wi-Fi/MQTT; RF remains deleted
11. idle loop
```

### F.2 Command handling

**`cmd/rf_code` received**

- Parse JSON and validate `schema_version`, `rf_code_hex`,
  `rf_code_bits`, `code_version`, and `issued_at`.
- Rebuild `rf-code-v1|...` and verify `signature_b64` with the pinned
  backend public key.
- Reject with `status = rejected` on shape error, signature failure,
  stale `code_version`, or radio-specific width mismatch.
- If `code_version` equals the stored version, ack `unchanged`.
- Otherwise:
  - if `op_state == PENDING_RF_CODE`, first call `rf_sup_start()`;
    on failure, ack `rejected` and leave both NVS and in-RAM trigger
    state untouched
  - commit `rf_code`, `rf_code_bits`, `rf_code_ver` in one NVS
    transaction
  - if this is the first code, persist `op_state = ACTIVE` in the same
    commit
  - update in-RAM trigger state under the trigger mutex
  - only then publish ack `applied`

**`cmd/deact` received**

- Parse JSON and validate `action`, `command_id`, and `issued_at`.
- Rebuild `deact-v1|...` and verify `signature_b64`.
- On verify failure, ack `rejected` and do not touch NVS or RF state.
- If `command_id == last_deact_id`, republish the prior ack semantics
  and skip side effects.
- `DECOMMISSIONED` is one-way at the firmware level: `resume` after
  decommission yields `ignored`.
- Otherwise:
  - `suspend` -> `rf_sup_suspend()`, persist `SUSPENDED` and
    `last_deact_id`, then ack `ok`
  - `resume` -> if a code is stored, `rf_sup_resume()` or
    `rf_sup_start()` as appropriate, persist `ACTIVE`; if no code is
    stored, persist `PENDING_RF_CODE`; write `last_deact_id`, then ack
  - `decommission` -> `rf_sup_delete()`, persist `DECOMMISSIONED` and
    `last_deact_id`, then ack `ok`

All NVS commits happen **before** the ack publish so a reboot after ack
comes back in the promised state.

### F.3 Failure and recovery cases

**`rf_code` while suspended or decommissioned**

- The device still validates and stores the new trigger code/version.
  The RF task simply does not act on it until a later `resume` or a new
  activation sequence.

**Wi-Fi repeatedly fails**

- After `CONFIG_RECEIVER_WIFI_MAX_RETRY` connection attempts, firmware
  brings SoftAP + HTTP up in place without wiping NVS.
- The recovery UI offers:
  - `POST /api/provision` — overwrite Wi-Fi/MQTT/token, then reboot
  - `POST /api/retry` — tear down recovery mode and reboot with the
    existing credentials unchanged
  - `POST /api/reset` — erase `device_cfg`, keep `identity`, reboot
    into `UNPROVISIONED`

**Pending envelope received**

- Confirm `registration_nonce` matches the in-RAM bootstrap session.
- Narrow the subscription from `receiver/bootstrap/+` to the exact
  `receiver/bootstrap/{challenge_id}` topic.
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

### F.4 SoftAP and local provisioning

- SSID: `RECEIVER-SETUP-<last_3_mac_bytes_hex>`
- `max_connection = 1`
- channel: `CONFIG_RECEIVER_AP_CHANNEL`
- password: `CONFIG_RECEIVER_AP_PASSWORD`
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

### F.5 Power management

- `esp_wifi_set_ps(WIFI_PS_MIN_MODEM)` only after `IP_EVENT_STA_GOT_IP`
- MQTT keepalive stays at 120 seconds
- No periodic heartbeat publish; MQTT keepalive pings are sufficient
- No deep sleep in v1; it breaks the low-latency control session model
- Do not call `esp_wifi_stop()` in the steady operational states
  `ACTIVE`, `SUSPENDED`, or `DECOMMISSIONED`
- The current 433 MHz polling loop (`vTaskDelay(10)`) is acceptable for
  v1; a future revision can switch to task notifications from the ISR
  side to improve sleep residency

---

## G. ESP-IDF v6 Code Guide

Snippets elide error handling. ESP-IDF core headers were cross-checked
against `/opt/esp/idf/components`; ESP-MQTT details were verified
against `managed_components/espressif__mqtt/include/mqtt_client.h`.
The target end state is C-only in `main/`; the temporary
`jgromes/radiolib` dependency is removed per §I.1.

### G.1 NVS — ESP-IDF pattern

The ESP32-C3 path uses normal ESP-IDF NVS helpers for atomic config
updates. Key calls:
`nvs_open("device_cfg", NVS_READWRITE, &h)`,
`nvs_set_blob(h, "rf_code", buf, len)`,
`nvs_set_u8(h, "rf_code_bits", bits)`,
`nvs_set_u32(h, "rf_code_ver", ver)`,
`nvs_commit(h)`.

In the current repo config, `nvs_flash_init()` still remains the right
entry point because the device uses the default `nvs` partition and the
SDK auto-enables encrypted NVS for that partition when
`CONFIG_NVS_ENCRYPTION=y`.

On first boot: `nvs_flash_init()` → on `ESP_ERR_NVS_NO_FREE_PAGES`
or `_NEW_VERSION_FOUND` → `nvs_flash_erase()` + retry.

### G.2 Radio supervisor — dispatch by variant

```c
#include "trigger/rf_supervisor.h"
#include "sdkconfig.h"

#if CONFIG_RECEIVER_RADIO_433M
#  include "rf/rf_common.h"
   static RFHandler g_rf;
   esp_err_t rf_sup_start  (void){ return rf_recv_start_task(CONFIG_RECEIVER_RF433_RX_GPIO, &g_rf); }
   esp_err_t rf_sup_suspend(void){ return rf_recv_suspend(&g_rf); }
   esp_err_t rf_sup_resume (void){ return rf_recv_resume (&g_rf); }
   esp_err_t rf_sup_delete (void){ return rf_recv_deinit (&g_rf); }
#elif CONFIG_RECEIVER_RADIO_2_4G
#  include "nrf24/nrf24_receiver.h"
   static nrf24_handle_t g_nrf;
   esp_err_t rf_sup_start  (void){ return nrf24_recv_start_task(&g_nrf); }
   esp_err_t rf_sup_suspend(void){ return nrf24_recv_suspend(&g_nrf); }
   esp_err_t rf_sup_resume (void){ return nrf24_recv_resume (&g_nrf); }
   esp_err_t rf_sup_delete (void){ return nrf24_recv_deinit (&g_nrf); }
#endif
```

The MQTT handler calls the `rf_sup_*` API; it never knows which radio
is active. On `ESP_OK`/`esp_err_t` failure, the deactivation handler
acks `rejected` and does **not** persist the new `op_state`.

### G.3 Wi-Fi — STA + SoftAP + WPA3-Compatible

ESP-IDF v6 requires `esp_netif` and separate event bases. The STA path
below uses v6's native **WPA3-Compatible Mode**: if the AP advertises
WPA3 (including WPA3-Personal RSN-override / transition-mode APs) the
station negotiates SAE, otherwise it falls back to WPA2 automatically.
In v6, leave `pmf_cfg.capable` alone and use the native compatibility
flags instead; `pmf_cfg.capable` itself is **deprecated** per
`esp_wifi_types_generic.h`:
*"Device will always connect in PMF mode if other device also
advertises PMF capability"*.

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
    wifi_init_config_t init = WIFI_INIT_CONFIG_DEFAULT();   // macro, keep as-is
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
            .threshold.authmode = WIFI_AUTH_WPA2_PSK,          // minimum; WPA3 accepted too
            .sae_pwe_h2e        = WPA3_SAE_PWE_BOTH,           // H2E + HnP (= v6 default)
            .pmf_cfg            = { .required = false },       // don't refuse non-PMF APs
            .disable_wpa3_compatible_mode = 0,                 // explicit: keep compat mode enabled
            .failure_retry_cnt  = CONFIG_RECEIVER_WIFI_MAX_RETRY,
        },
    };
    strlcpy((char*)sta_cfg.sta.ssid,     cfg->wifi_ssid, sizeof sta_cfg.sta.ssid);
    strlcpy((char*)sta_cfg.sta.password, cfg->wifi_pwd,  sizeof sta_cfg.sta.password);

    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &sta_cfg));
    ESP_ERROR_CHECK(esp_wifi_start());
    // ... wait for IP_EVENT_STA_GOT_IP ...
    ESP_ERROR_CHECK(esp_wifi_set_ps(WIFI_PS_MIN_MODEM));  // post-association only
}

void wifi_start_softap(void) {
    if (!s_ap_netif) s_ap_netif = esp_netif_create_default_wifi_ap();
    wifi_init_common();

    uint8_t mac[6];
    esp_read_mac(mac, ESP_MAC_WIFI_STA);        // label only — no radio state needed

    wifi_config_t ap_cfg = {
        .ap = {
            .channel              = CONFIG_RECEIVER_AP_CHANNEL,
            .password             = CONFIG_RECEIVER_AP_PASSWORD,
            .max_connection       = 1,
            .authmode             = WIFI_AUTH_WPA2_PSK,
            .sae_pwe_h2e          = WPA3_SAE_PWE_BOTH,
            .pmf_cfg              = { .required = false },
            .wpa3_compatible_mode = 1,                     // WPA3 to capable STAs, WPA2 otherwise
        },
    };
    int n = snprintf((char*)ap_cfg.ap.ssid, sizeof ap_cfg.ap.ssid,
                     "RECEIVER-SETUP-%02X%02X%02X", mac[3], mac[4], mac[5]);
    ap_cfg.ap.ssid_len = n;

    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_AP));
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_AP, &ap_cfg));
    ESP_ERROR_CHECK(esp_wifi_start());
}
```

Notes:

- `WIFI_IF_STA` / `WIFI_IF_AP` replace the legacy `ESP_IF_WIFI_*`.
- The two helpers share one Wi-Fi driver instance. `esp_wifi_init()`,
default netif creation, and event-handler registration are guarded so
a same-boot STA→SoftAP fallback reuses the existing stack instead of
trying to initialize it twice.
- If a pre-activation STA attempt already started Wi-Fi, stop or
disconnect that session before restarting in `WIFI_MODE_AP`. The
"do not call `esp_wifi_stop()`" rule below applies only to the
steady-state operational states after activation.
- **WPA3-Compatible mode is enabled on both sides.**
  - **STA:** `wifi_sta_config_t.disable_wpa3_compatible_mode` defaults
  to `0` (compat ON). Station negotiates SAE with any WPA3-advertising
  AP, including WPA3-Personal RSN-override transition APs, and falls
  back to WPA2 otherwise. The sample keeps that behavior explicit with
  `.disable_wpa3_compatible_mode = 0`.
  - **SoftAP:** `wifi_ap_config_t.wpa3_compatible_mode` is **opt-in**
  (default `0`, we set it to `1`). When on, per the v6 docstring
  *"the AP will operate as a WPA2 access point for all stations
  except for those that support WPA3 compatible mode. Only WPA3
  compatibility mode stations will be able to use WPA3-SAE"* — and
  this override **supersedes** `authmode` / `pairwise_cipher`. A
  WPA3-aware installer phone thus negotiates SAE; older phones
  still associate with the same PSK over WPA2.
- `pmf_cfg.capable` is **deprecated in v6** — omitted. If a deployment
is WPA3-only and must refuse WPA2 associations, set
`pmf_cfg.required = true` on the STA side instead.
- The config variables use nested designated initializers; only the
SSID/password arrays still need `strlcpy()` after initialization.
- `failure_retry_cnt` only takes effect with `WIFI_ALL_CHANNEL_SCAN`;
the STA sample above pairs those two fields accordingly.
- `threshold.authmode = WIFI_AUTH_WPA2_PSK` is a *minimum* — APs with
stronger auth (WPA3, WPA2/WPA3 mixed) are still accepted; open and
WEP are rejected.
- `WIFI_PS_MIN_MODEM` only after `IP_EVENT_STA_GOT_IP` — setting it
pre-association is silently ignored.
- Do **not** call `esp_wifi_stop()` during `ACTIVE` / `SUSPENDED` /
`DECOMMISSIONED` — the device must stay reachable.

### G.4 MQTT — new v6 config layout

The `esp_mqtt_client_config_t` fields are **nested** in ESP-MQTT 1.0.0
(`managed_components/espressif__mqtt/include/mqtt_client.h`). The CA PEM
is embedded via `EMBED_TXTFILES` (auto-NUL-terminated → valid C string).

```c
#include "mqtt_client.h"

extern const uint8_t ca_pem_start[] asm("_binary_mqtt_ca_pem_start");

static esp_mqtt_client_handle_t s_client;

static void on_mqtt(void *arg, esp_event_base_t base, int32_t id, void *data) {
    esp_mqtt_event_handle_t e = data;
    switch ((esp_mqtt_event_id_t)id) {
    case MQTT_EVENT_CONNECTED:   subscribe_current_phase(); break;
    case MQTT_EVENT_DATA:        /* check total_data_len vs data_len for fragments */
                                 topic_router_dispatch(e); break;
    default: break;
    }
}

void mqtt_start(const device_config_t *cfg) {
    esp_mqtt_client_config_t mqtt_cfg = {
        .broker = {
            .address = {
                .uri = cfg->mqtt_uri,                       // "mqtts://broker.example.com:8883"
            },
            .verification = {
                .certificate = (const char*)ca_pem_start,
                .certificate_len = 0,                       // PEM => len must stay 0
            },
        },
        .credentials = {
            .username = cfg->mqtt_user,
            .client_id = cfg->public_id,                    // NULL pre-activation => default ID
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

- Reject `mqtt_uri` that does not start with `mqtts://` at config-load
time.
- Keep the TCP port inside `mqtt_uri` (for example
`mqtts://broker.example.com:8883`). In the local ESP-MQTT header,
`broker.address.uri` takes precedence over `hostname` / `transport` /
`port`, so this plan does **not** set `.broker.address.port`
separately.
- With `set_null_client_id = false`, a pre-activation `client_id = NULL`
makes ESP-MQTT synthesize its built-in default client ID
(`ESP32_%CHIPID%`). After activation, reconnect with the minted
`public_id`.
- For PEM material, leave `certificate_len = 0`; ESP-MQTT treats
zero-length + NUL-terminated data as the PEM/string form. The
non-zero length form is for DER/raw buffers and must be the exact
byte count.
- Cherry-picked operational settings from the implementation template:
explicit reconnect timeout, network timeout, refresh interval,
retransmit timeout, and MQTT task sizing.
- Intentionally **not** adopted from the template: client-certificate
auth. The ESP32-C3 path can use `MQTT_PROTOCOL_V_5` because the
broker supports both MQTT 3.1.1 and MQTT 5; the legacy ESP-01 path
remains on MQTT 3.1.1. The higher-level bootstrap/topic plan stays
the same, so the C3 path does not rely on MQTT 5-only subscription
features such as `no-local`.
- `MQTT_EVENT_DATA` may deliver fragments: always check
`e->total_data_len > e->data_len` and buffer before JSON parsing.
- `esp_mqtt_client_publish` is thread-safe and safe from inside
`on_mqtt`.

#### G.4.1 Post-activation MQTT restart

Do **not** try to switch from the bootstrap session to the operational
session with `esp_mqtt_client_reconnect()` alone: the broker URI and
credentials stay the same, but the `client_id` changes from the
pre-activation default (`ESP32_%CHIPID%`) to the backend-minted
`public_id`. In ESP-MQTT 1.0 that means **stop + destroy + init/start**
with the updated config.

Also note the local SDK rule: `esp_mqtt_client_stop()` **cannot** be
called from the MQTT event handler. Defer the restart to a control task
or application event.

```c
typedef struct {
    char public_id[32];
    char assigned_device_name[64];
} activation_result_t;

static TaskHandle_t s_mqtt_ctl_task;

static void mqtt_restart_with_identity(const device_config_t *cfg) {
    if (s_client) {
        ESP_ERROR_CHECK(esp_mqtt_client_stop(s_client));   // not from on_mqtt()
        esp_mqtt_client_destroy(s_client);
        s_client = NULL;
    }
    mqtt_start(cfg);  // cfg->public_id now drives client_id + topic phase
}

static void handle_activation_result(const activation_result_t *r) {
    strlcpy(g_cfg.public_id,   r->public_id, sizeof g_cfg.public_id);
    strlcpy(g_cfg.device_name, r->assigned_device_name, sizeof g_cfg.device_name);
    g_cfg.op_state = OP_STATE_PENDING_RF_CODE;
    device_config_commit_activation(&g_cfg);   // persists IDs + clears enroll_token

    // MQTT stop/destroy is deferred out of the MQTT callback context.
    xTaskNotifyGive(s_mqtt_ctl_task);
}

static void mqtt_ctl_task(void *arg) {
    for (;;) {
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);
        mqtt_restart_with_identity(&g_cfg);
    }
}
```

Result: right after activation, the device drops the bootstrap session,
reconnects with `client_id = public_id`, and `subscribe_current_phase()`
switches from `receiver/bootstrap/+` to
`receiver/device/{public_id}/cmd/#`. `device_name` is persisted in the
same commit, so the local status/UI path can show the assigned name
immediately after the reconnect.

### G.5 HTTP Provisioning Server

The provisioning server in this layout is a normal ESP-IDF component.
Its HTTP contract is the one defined in §F.4: `GET /`,
`GET /api/status`, `POST /api/provision`, `POST /api/retry`, and
`POST /api/reset`. The main implementation difference vs the ESP-01
lineage is that ESP-IDF embeds the UI and CA PEM through the component
registration step:

```cmake
# main/CMakeLists.txt — pure-C component.
idf_component_register(
    SRCS "main.c"
         "config/device_config.c"
         "network/wifi.c"
         "network/mqtt.c"
         "provision/http_server.c"
         "security/device_identity.c"
         "trigger/rf_trigger.c"
         "trigger/rf_supervisor.c"
         "rf/rf_receiver.c"
         "rf/rf_data.c"
         "nrf24/nrf24_receiver.c"
         "vibrator/vibrator.c"
    INCLUDE_DIRS "."
    EMBED_TXTFILES "network/certs/mqtt_ca.pem"
    EMBED_FILES    "provision/index.html.gz"
    PRIV_REQUIRES  nvs_flash esp_wifi esp_netif esp_event esp_http_server
                   esp_driver_gpio esp_driver_spi esp_timer mbedtls
                   espressif__mqtt espressif__cjson
)
```

`**main.c` skeleton** (plain C — `app_main` is the standard ESP-IDF
entry point, no linkage wrapper needed):

```c
// main/main.c
#include "nvs_flash.h"
#include "esp_netif.h"
#include "esp_event.h"
#include "config/device_config.h"
#include "network/wifi.h"
#include "network/mqtt.h"
#include "trigger/rf_supervisor.h"

void app_main(void) {
    ESP_ERROR_CHECK(nvs_init_or_recover());     // see §G.1 for erase + retry
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    device_config_load(&g_cfg);
    // ... §F orchestration ...
}
```

Here `nvs_init_or_recover()` is the tiny §G.1 wrapper around
`nvs_flash_init()` plus the erase-and-retry path for
`ESP_ERR_NVS_NO_FREE_PAGES` / `_NEW_VERSION_FOUND`.

The UI remains a gzip-embedded static asset; only the embedding site
moves to `main/CMakeLists.txt`.

### G.6 433 MHz receiver — existing `rf/` module

The current `main/rf/rf_receiver.c` is already ESP-IDF v5/v6 idiomatic
(`gpio_config`, `gpio_install_isr_service`, `esp_timer_get_time`,
`IRAM_ATTR`). The one missing wrapper for the target module layout is
`rf_recv_start_task` helper — add its declaration to `rf_common.h` and
its implementation to `rf_receiver.c`:

```c
// rf/rf_receiver.c (addition)
static void rf_recv_task(void *arg) {
    RFHandler *h = (RFHandler*)arg;
    RFRecvData data = { 0 };
    for (;;) {
        if (recv_available(h) == ESP_OK) {
            output_recv(h, &data);           // fills original_value + bit_length
            rf_trigger_on_frame(data.original_value, h->recv_bit_length);
            reset_recv(h);
        }
        vTaskDelay(pdMS_TO_TICKS(10));        // v2: task notification from ISR
    }
}

esp_err_t rf_recv_start_task(gpio_num_t gpio, RFHandler *h) {
    if (h->rx_active) return ESP_OK;          // idempotent
    ESP_RETURN_ON_ERROR(rf_recv_init(gpio, h), RF_TAG, "init");
    BaseType_t ok = xTaskCreate(rf_recv_task, "rf_rx", 4096, h, 5, &h->rf_recv_handle);
    return ok == pdPASS ? ESP_OK : ESP_ERR_NO_MEM;
}
```

`rf_recv_init` must **not** be called again from inside the task body
— it's called exactly once in `rf_recv_start_task`. Suspend / resume /
deinit keep the current `rf_recv_*` semantics.

### G.7 nRF24L01 / nRF24L01+ receiver — native ESP-IDF SPI driver

The chip's programming surface is small (SPI + three GPIOs: CE, CS/CSN,
IRQ; ≈ 25 registers; ≤ 11 SPI commands). Writing it directly on top of
`driver/spi_master.h` is smaller and easier to audit than pulling in
RadioLib + Arduino shim, and lets the same driver run on both the
legacy nRF24L01 and the nRF24L01+ silicon (see §G.7.6 for the compat
analysis). Protocol references: nRF24L01 PS v2.0 §8 (SPI; command-set
Table 16) + §9 (register map); nRF24L01+ PS v1.0 §8 (SPI; command-set
Table 20) + §9 (register map; Table 28 for the bit-level diffs we
care about). Behavior we implement matches what the 433 MHz path
offers: start / suspend / resume / deinit, plus an IRQ-driven RX path
that hands each decoded packet to `rf_trigger_on_packet`.

#### G.7.1 Wire protocol in one glance

- **SPI mode 0**, MSB-first. Datasheet caps: 8 Mbps on nRF24L01
(PS v2.0 §8.2) and 10 Mbps on nRF24L01+ (PS v1.0 §8.2). The driver
hard-codes 4 MHz in `spi_device_interface_config_t.clock_speed_hz`
— well under both caps and robust on jumper-wire rigs. Not exposed
as a Kconfig in v1; edit the literal if a production board has
short controlled-impedance traces and wants to push higher.
- **One transaction = CS low → command byte → optional data bytes →
CS high.** The STATUS register is shifted out on MISO during the
command byte for *every* command; we capture it and use it instead
of issuing a separate NOP.
- Multi-byte registers (addresses) are sent **LSByte-first** on the
wire (datasheet §8.3.1). The driver uses `memcpy` — the byte order
in `CONFIG_RECEIVER_NRF24_RX_ADDR` is the byte order on the wire,
and must match what the transmitter writes to `TX_ADDR`.

#### G.7.2 Register / command defines

```c
// nrf24/nrf24_regs.h
#pragma once

// Register addresses. Identical address space on nRF24L01 (PS v2.0 §9
// Table 24) and nRF24L01+ (PS v1.0 §9 Table 28). See §G.7.6.
#define NRF_REG_CONFIG      0x00
#define NRF_REG_EN_AA       0x01
#define NRF_REG_EN_RXADDR   0x02
#define NRF_REG_SETUP_AW    0x03
#define NRF_REG_SETUP_RETR  0x04
#define NRF_REG_RF_CH       0x05
#define NRF_REG_RF_SETUP    0x06
#define NRF_REG_STATUS      0x07
#define NRF_REG_RX_ADDR_P0  0x0A
#define NRF_REG_RX_ADDR_P1  0x0B
#define NRF_REG_RX_PW_P0    0x11
#define NRF_REG_RX_PW_P1    0x12
#define NRF_REG_FIFO_STATUS 0x17
#define NRF_REG_DYNPD       0x1C
#define NRF_REG_FEATURE     0x1D

// SPI commands.
//   - Shared by both chips: all commands below except ACTIVATE.
//   - nRF24L01 only: ACTIVATE (0x50 + magic 0x73) — required on the legacy
//     chip before FEATURE / DYNPD can be written or R_RX_PL_WID used.
//     Omitted from the nRF24L01+ command set. Behavior on the + is not
//     documented, so we confine ACTIVATE to the FEATURE-probe failure path
//     described in §G.7.6 instead of sending it unconditionally at boot.
#define NRF_CMD_R_REGISTER   0x00   // OR with reg (low 5 bits)
#define NRF_CMD_W_REGISTER   0x20   // OR with reg (low 5 bits)
#define NRF_CMD_R_RX_PAYLOAD 0x61
#define NRF_CMD_R_RX_PL_WID  0x60   // width of top RX FIFO packet (DPL)
#define NRF_CMD_W_TX_PAYLOAD 0xA0
#define NRF_CMD_FLUSH_TX     0xE1
#define NRF_CMD_FLUSH_RX     0xE2
#define NRF_CMD_ACTIVATE     0x50   // followed by NRF_ACTIVATE_MAGIC, legacy only
#define NRF_CMD_NOP          0xFF
#define NRF_ACTIVATE_MAGIC   0x73

// CONFIG bits (address 0x00).
#define NRF_CFG_PRIM_RX      (1<<0)
#define NRF_CFG_PWR_UP       (1<<1)
#define NRF_CFG_CRCO         (1<<2)   // 0 = 8-bit, 1 = 16-bit CRC
#define NRF_CFG_EN_CRC       (1<<3)
#define NRF_CFG_MASK_MAX_RT  (1<<4)   // 1 = do not pull IRQ on MAX_RT
#define NRF_CFG_MASK_TX_DS   (1<<5)
#define NRF_CFG_MASK_RX_DR   (1<<6)

// RF_SETUP bits (address 0x06). Layout agrees on bits [4:1] across both
// chip revisions; bits [7,5,0] differ — see the compat table in §G.7.6.
// We write values that are legal and meaningful on both.
#define NRF_RF_DR_1M         0x00      // bit5=0, bit3=0: 1 Mbps on both
#define NRF_RF_DR_2M         (1<<3)    // bit5=0, bit3=1: 2 Mbps on both
#define NRF_RF_PWR_M18       (0<<1)
#define NRF_RF_PWR_M12       (1<<1)
#define NRF_RF_PWR_M6        (2<<1)
#define NRF_RF_PWR_0         (3<<1)    // 0 dBm — default
// Bit 0: LNA_HCURR on nRF24L01 (1 = high-sensitivity LNA, +1.5 dB RX;
// reset default = 1). Documented as "Obsolete / Don't care" on nRF24L01+
// (bit 0 writes are ignored; LNA is always engaged). Always OR this into
// RF_SETUP so both chips operate at peak sensitivity.
#define NRF_RF_LNA_HCURR     (1<<0)
// Bits the driver never writes as 1:
//   bit 7 CONT_WAVE : nRF24L01+-only test-mode carrier transmitter
//                     (Reserved on original — must be 0).
//   bit 5 RF_DR_LOW : nRF24L01+-only 250 kbps mode
//                     (Reserved on original — must be 0).
// Leaving both at 0 is the only setting that is safe on both chips.

// FEATURE register bits (address 0x1D). Reset = 0 on both chips. Writes
// to this register are gated by ACTIVATE on the legacy nRF24L01.
#define NRF_FEATURE_EN_DYN_ACK   (1<<0)  // unused — TX-side selective no-ack
#define NRF_FEATURE_EN_ACK_PAY   (1<<1)  // unused — RX-side ACK payload
#define NRF_FEATURE_EN_DPL       (1<<2)  // dynamic payload length

// DYNPD register bits (address 0x1C) — DPL enable per pipe.
#define NRF_DYNPD_P1             (1<<1)

// STATUS bits (write-1-to-clear for RX_DR/TX_DS/MAX_RT).
#define NRF_ST_RX_DR   (1<<6)
#define NRF_ST_TX_DS   (1<<5)
#define NRF_ST_MAX_RT  (1<<4)
#define NRF_ST_RX_P_NO_MASK  0x0E   // bits 3:1 — pipe that received data
#define NRF_ST_RX_FIFO_EMPTY 0x0E   // RX_P_NO = 0b111

// FIFO_STATUS bits.
#define NRF_FIFO_RX_EMPTY (1<<0)

// Payload upper bound on either chip. Also the static pipe-1 width we
// program when DPL is disabled.
#define NRF_PAYLOAD_WIDTH 32

// Timings. nRF24L01+ PS v1.0 §6.1.7 Table 16 lists Tpd2stby up to 4.5 ms
// (external crystal, Ls = 90 mH). Original nRF24L01 PS v2.0 §6.1.7 only
// specifies 1.5 ms (internal crystal) / 150 µs (external clock). 5 ms
// covers every published variant.
#define NRF_TPOR_MS        100   // power-on-reset settle
#define NRF_TPD2STBY_MS    10     // Tpd2stby max across both chips (+ Ls=90mH)
#define NRF_TSTBY2A_US     140   // Tstby2a 130 µs max; round up
```

#### G.7.3 Public API

Mirrors the `rf_recv_*` shape so `rf_supervisor` (§G.2) dispatches
with no special casing:

```c
// nrf24/nrf24_receiver.h
#pragma once
#include <stdbool.h>
#include "esp_err.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

typedef enum {
    NRF_CHIP_UNKNOWN = 0,  // probe failed (bus fault or no chip)
    NRF_CHIP_LEGACY,       // nRF24L01 — required ACTIVATE 0x73 to unlock FEATURE
    NRF_CHIP_PLUS,         // nRF24L01+ — FEATURE writable without ACTIVATE
} nrf24_variant_t;

typedef struct {
    bool             rx_active;
    bool             rx_suspended;
    TaskHandle_t     task;
    void            *spi;      // opaque spi_device_handle_t; concrete in .c
    nrf24_variant_t  chip;     // populated by nrf24_recv_start_task (§G.7.6)
} nrf24_handle_t;

esp_err_t nrf24_recv_start_task(nrf24_handle_t *h);
esp_err_t nrf24_recv_suspend  (nrf24_handle_t *h);
esp_err_t nrf24_recv_resume   (nrf24_handle_t *h);
esp_err_t nrf24_recv_deinit   (nrf24_handle_t *h);
```

#### G.7.4 Implementation

```c
// nrf24/nrf24_receiver.c
#include "nrf24_receiver.h"
#include "nrf24_regs.h"
#include "trigger/rf_trigger.h"
#include "driver/gpio.h"
#include "driver/spi_master.h"
#include "esp_log.h"
#include "esp_rom_sys.h"
#include "sdkconfig.h"
#include <string.h>

#define NRF_TAG "NRF24"
#define CE_GPIO  ((gpio_num_t)CONFIG_RECEIVER_NRF24_CE)
#define IRQ_GPIO ((gpio_num_t)CONFIG_RECEIVER_NRF24_IRQ)

static spi_device_handle_t s_spi;
static TaskHandle_t        s_notify_to;  // set before IRQ is enabled, never reassigned

/* ---- SPI helpers (all run under CS auto-toggled by the driver) ---- */

static uint8_t nrf_xfer(uint8_t cmd, const uint8_t *tx, uint8_t *rx, size_t n) {
    uint8_t buf_tx[1 + 32] = { cmd };
    uint8_t buf_rx[1 + 32] = { 0 };
    if (tx && n) memcpy(&buf_tx[1], tx, n);
    spi_transaction_t t = {
        .length    = 8 * (1 + n),           // bits
        .tx_buffer = buf_tx,
        .rx_buffer = buf_rx,
    };
    ESP_ERROR_CHECK(spi_device_polling_transmit(s_spi, &t));
    if (rx && n) memcpy(rx, &buf_rx[1], n);
    return buf_rx[0];                       // STATUS shifted out during cmd byte
}

static inline uint8_t nrf_cmd (uint8_t cmd)                        { return nrf_xfer(cmd, NULL, NULL, 0); }
static inline uint8_t nrf_rreg(uint8_t reg, uint8_t *rx, size_t n) { return nrf_xfer(NRF_CMD_R_REGISTER | (reg & 0x1F), NULL, rx, n); }
static inline uint8_t nrf_wreg(uint8_t reg, const uint8_t *tx, size_t n) { return nrf_xfer(NRF_CMD_W_REGISTER | (reg & 0x1F), tx, NULL, n); }

static inline uint8_t nrf_rreg8(uint8_t reg)              { uint8_t v; nrf_rreg(reg, &v, 1); return v; }
static inline void    nrf_wreg8(uint8_t reg, uint8_t v)   { nrf_wreg(reg, &v, 1); }

/* ---- ISR: must live in IRAM; only unblocks the task ---- */

static void IRAM_ATTR nrf_isr(void *arg) {
    (void)arg;
    BaseType_t hpw = pdFALSE;
    vTaskNotifyGiveFromISR(s_notify_to, &hpw);
    portYIELD_FROM_ISR(hpw);
}

/* ---- RX task: follows the STATUS-register footnote in both datasheets
   verbatim — per-packet *read → clear RX_DR → re-check FIFO_STATUS*. The
   per-packet clear is load-bearing: RX_DR is edge-triggered on packet
   arrival, so if we batched the clear to once per drain, a packet that
   arrived after our final FIFO_STATUS check but before the W1C would
   sit in the FIFO with RX_DR already = 1 (no new edge, no new notify)
   until the *next* packet's arrival event produced a fresh edge — at
   which point one real event would fire two rf_trigger_on_packet() calls
   (silent double-trigger). Clearing per-packet makes every arrival a
   clean 0→1 edge.
   Body forks on CONFIG_RECEIVER_NRF24_ENABLE_DPL (default y). With DPL
   we ask the chip for the actual payload length on every read; without
   it we read the fixed 32-byte slot programmed into RX_PW_P1.        ---- */

static void nrf_rx_task(void *arg) {
    nrf24_handle_t *h = arg;
    uint8_t buf[NRF_PAYLOAD_WIDTH];
    for (;;) {
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);
        // Drain every queued packet before re-arming. FIFO holds up to 3.
        for (;;) {
            if (nrf_rreg8(NRF_REG_FIFO_STATUS) & NRF_FIFO_RX_EMPTY) break;
#if CONFIG_RECEIVER_NRF24_ENABLE_DPL
            // nRF24L01+ PS v1.0 §7.3.4 / note under R_RX_PL_WID: if the
            // returned width is > 32, the packet is corrupt and must be
            // flushed before proceeding.
            uint8_t len = 0;
            nrf_xfer(NRF_CMD_R_RX_PL_WID, NULL, &len, 1);
            if (len == 0 || len > NRF_PAYLOAD_WIDTH) {
                nrf_cmd(NRF_CMD_FLUSH_RX);
                nrf_wreg8(NRF_REG_STATUS, NRF_ST_RX_DR);
                break;
            }
            nrf_xfer(NRF_CMD_R_RX_PAYLOAD, NULL, buf, len);
            rf_trigger_on_packet(buf, len);
#else
            nrf_xfer(NRF_CMD_R_RX_PAYLOAD, NULL, buf, NRF_PAYLOAD_WIDTH);
            rf_trigger_on_packet(buf, NRF_PAYLOAD_WIDTH);
#endif
            // Clear RX_DR (W1C) per packet — see comment block above.
            nrf_wreg8(NRF_REG_STATUS, NRF_ST_RX_DR);
        }
        (void)h;
    }
}

/* ---- Chip probe: detects nRF24L01 vs nRF24L01+ and, as a side effect,
   unlocks FEATURE-register writes on the legacy chip so DPL and friends
   are programmable regardless of which silicon was populated. See §G.7.6
   for the full algorithm and risk analysis.                            ---- */

static nrf24_variant_t nrf_probe_chip(void) {
    // Both chips reset FEATURE = 0x00. Probe with EN_DPL; if the write
    // sticks, FEATURE is unlocked (nRF24L01+ always, or an already-ACTIVATE'd
    // legacy part — rare after a clean POR).
    nrf_wreg8(NRF_REG_FEATURE, NRF_FEATURE_EN_DPL);
    if (nrf_rreg8(NRF_REG_FEATURE) == NRF_FEATURE_EN_DPL) {
        // Clean up: restore reset state so the caller can program from 0.
        nrf_wreg8(NRF_REG_FEATURE, 0x00);
        return NRF_CHIP_PLUS;
    }
    // FEATURE locked. Send ACTIVATE + magic byte and retry the probe.
    // Only reachable on a chip that rejected FEATURE writes, which under
    // normal operation only ever includes the legacy nRF24L01.
    const uint8_t magic = NRF_ACTIVATE_MAGIC;
    nrf_xfer(NRF_CMD_ACTIVATE, &magic, NULL, 1);
    nrf_wreg8(NRF_REG_FEATURE, NRF_FEATURE_EN_DPL);
    if (nrf_rreg8(NRF_REG_FEATURE) == NRF_FEATURE_EN_DPL) {
        nrf_wreg8(NRF_REG_FEATURE, 0x00);
        return NRF_CHIP_LEGACY;
    }
    return NRF_CHIP_UNKNOWN;
}

/* ---- Lifecycle ---- */

static esp_err_t nrf_bringup(nrf24_handle_t *h) {
    // CE low = RX off; hold for power-on settle.
    ESP_ERROR_CHECK(gpio_config(&(gpio_config_t){
        .pin_bit_mask = 1ULL << CE_GPIO,
        .mode         = GPIO_MODE_OUTPUT,
        .pull_up_en   = GPIO_PULLUP_DISABLE,
        .pull_down_en = GPIO_PULLDOWN_DISABLE,
        .intr_type    = GPIO_INTR_DISABLE,
    }));
    gpio_set_level(CE_GPIO, 0);
    vTaskDelay(pdMS_TO_TICKS(NRF_TPOR_MS));

    // Liveness probe: SETUP_AW reset value is 0b11 (5-byte address), with
    // bits 7:2 reserved 0. A raw 0x00 or garbage in the reserved bits
    // means no chip on the bus, or MISO tied high / low.
    uint8_t aw = nrf_rreg8(NRF_REG_SETUP_AW);
    if ((aw & 0x03) == 0 || (aw & 0xFC) != 0) {
        ESP_LOGE(NRF_TAG, "SETUP_AW=0x%02x (no chip?)", aw);
        return ESP_ERR_NOT_FOUND;
    }

    // Discriminate nRF24L01 vs nRF24L01+ and unlock FEATURE on the legacy
    // silicon. Logged at INFO so production boards can report which chip
    // they enumerated via the device shadow / MQTT later.
    h->chip = nrf_probe_chip();
    if (h->chip == NRF_CHIP_UNKNOWN) {
        ESP_LOGE(NRF_TAG, "FEATURE write locked even after ACTIVATE — bus fault");
        return ESP_ERR_INVALID_RESPONSE;
    }
    ESP_LOGI(NRF_TAG, "chip detected: %s",
             h->chip == NRF_CHIP_PLUS ? "nRF24L01+" : "nRF24L01 (legacy)");

    // Core config (still PWR_UP=0; RX configuration before we power up).
    nrf_wreg8(NRF_REG_CONFIG,
              NRF_CFG_MASK_TX_DS | NRF_CFG_MASK_MAX_RT
              | NRF_CFG_EN_CRC | NRF_CFG_CRCO
              | NRF_CFG_PRIM_RX);                       // MASK_RX_DR=0 -> IRQ on RX
    nrf_wreg8(NRF_REG_EN_AA,      0x02);                // auto-ACK pipe 1 only
    nrf_wreg8(NRF_REG_EN_RXADDR,  0x02);                // pipe 1 only
    nrf_wreg8(NRF_REG_SETUP_AW,   0x03);                // 5-byte addresses
    nrf_wreg8(NRF_REG_SETUP_RETR, 0x00);                // pure RX — no retransmits
    nrf_wreg8(NRF_REG_RF_CH,      CONFIG_RECEIVER_NRF24_CHANNEL);  // 97..125
    // Data rate + power + LNA (bit 0 set for legacy; ignored by nRF24L01+).
#if   CONFIG_RECEIVER_NRF24_DR_2M
    nrf_wreg8(NRF_REG_RF_SETUP, NRF_RF_DR_2M | NRF_RF_PWR_0 | NRF_RF_LNA_HCURR);
#else   /* CONFIG_RECEIVER_NRF24_DR_1M, the default */
    nrf_wreg8(NRF_REG_RF_SETUP, NRF_RF_DR_1M | NRF_RF_PWR_0 | NRF_RF_LNA_HCURR);
#endif

    // Dynamic vs static payload. FEATURE is now writable on both chips
    // because nrf_probe_chip ran ACTIVATE 0x73 on the legacy part.
#if CONFIG_RECEIVER_NRF24_ENABLE_DPL
    nrf_wreg8(NRF_REG_FEATURE, NRF_FEATURE_EN_DPL);
    nrf_wreg8(NRF_REG_DYNPD,   NRF_DYNPD_P1);
    // RX_PW_Px is ignored on pipes with DPL enabled; leave at reset 0.
#else
    nrf_wreg8(NRF_REG_FEATURE,  0x00);
    nrf_wreg8(NRF_REG_DYNPD,    0x00);
    nrf_wreg8(NRF_REG_RX_PW_P1, NRF_PAYLOAD_WIDTH);     // fixed 32-byte payload
#endif

    const uint8_t addr[5] = {
        CONFIG_RECEIVER_NRF24_RX_ADDR_B0, CONFIG_RECEIVER_NRF24_RX_ADDR_B1,
        CONFIG_RECEIVER_NRF24_RX_ADDR_B2, CONFIG_RECEIVER_NRF24_RX_ADDR_B3,
        CONFIG_RECEIVER_NRF24_RX_ADDR_B4,
    };
    nrf_wreg(NRF_REG_RX_ADDR_P1, addr, 5);              // LSByte first on the wire

    nrf_cmd(NRF_CMD_FLUSH_RX);
    nrf_cmd(NRF_CMD_FLUSH_TX);
    nrf_wreg8(NRF_REG_STATUS, NRF_ST_RX_DR | NRF_ST_TX_DS | NRF_ST_MAX_RT);  // W1C

    // Power up into Standby-I (PWR_UP=1; PRIM_RX already set, CE still low).
    // Caller flips CE high *after* the ISR is installed, so a packet arriving
    // during bringup cannot deadlock on a missed edge.
    uint8_t cfg = nrf_rreg8(NRF_REG_CONFIG) | NRF_CFG_PWR_UP;
    nrf_wreg8(NRF_REG_CONFIG, cfg);
    vTaskDelay(pdMS_TO_TICKS(NRF_TPD2STBY_MS));         // Tpd2stby worst-case
    return ESP_OK;
}

esp_err_t nrf24_recv_start_task(nrf24_handle_t *h) {
    if (h->rx_active) return ESP_OK;

    // SPI bus: SPI_DMA_DISABLED because our transfers are ≤ 33 B (1 cmd +
    // 32 payload); CPU mode is faster at that size and drops the DMA-
    // alignment requirement on the stack-allocated scratch buffers inside
    // nrf_xfer().
    spi_bus_config_t bus_cfg = {
        .mosi_io_num     = CONFIG_RECEIVER_NRF24_MOSI,
        .miso_io_num     = CONFIG_RECEIVER_NRF24_MISO,
        .sclk_io_num     = CONFIG_RECEIVER_NRF24_SCK,
        .quadwp_io_num   = -1,
        .quadhd_io_num   = -1,
        .max_transfer_sz = 64,
    };
    ESP_ERROR_CHECK(spi_bus_initialize(SPI2_HOST, &bus_cfg, SPI_DMA_DISABLED));

    // SPI device: mode 0, 4 MHz (datasheet caps: 8 Mbps legacy / 10 Mbps +).
    // .spics_io_num lets the driver toggle CS automatically per transaction.
    spi_device_interface_config_t dev_cfg = {
        .mode           = 0,
        .clock_speed_hz = 4 * 1000 * 1000,
        .spics_io_num   = CONFIG_RECEIVER_NRF24_CS,
        .queue_size     = 1,
        .flags          = 0,                            // full duplex
    };
    ESP_ERROR_CHECK(spi_bus_add_device(SPI2_HOST, &dev_cfg, &s_spi));
    h->spi = s_spi;

    // IRQ pin: active-low, pulled up, negedge.
    ESP_ERROR_CHECK(gpio_config(&(gpio_config_t){
        .pin_bit_mask = 1ULL << IRQ_GPIO,
        .mode         = GPIO_MODE_INPUT,
        .pull_up_en   = GPIO_PULLUP_ENABLE,
        .pull_down_en = GPIO_PULLDOWN_DISABLE,
        .intr_type    = GPIO_INTR_NEGEDGE,
    }));

    esp_err_t e = nrf_bringup(h);
    if (e != ESP_OK) return e;

    // Spawn task *before* arming the ISR.
    if (xTaskCreate(nrf_rx_task, "nrf_rx", 4096, h, 5, &h->task) != pdPASS)
        return ESP_ERR_NO_MEM;
    s_notify_to = h->task;

    esp_err_t is = gpio_install_isr_service(ESP_INTR_FLAG_IRAM);
    if (is != ESP_OK && is != ESP_ERR_INVALID_STATE) ESP_ERROR_CHECK(is);
    ESP_ERROR_CHECK(gpio_isr_handler_add(IRQ_GPIO, nrf_isr, NULL));

    // ISR is live — now enable RX. Any packet arriving during the preceding
    // Tpd2stby wait is already cleared by the FLUSH_RX in nrf_bringup().
    gpio_set_level(CE_GPIO, 1);
    esp_rom_delay_us(NRF_TSTBY2A_US);                    // Tstby2a 130 µs
    // Kickstart: if an edge landed in the tiny window between CE-high and the
    // first ISR registration, the FIFO may already have a packet that no
    // further edge will announce. Poke the task to drain unconditionally.
    xTaskNotifyGive(h->task);

    h->rx_active = true; h->rx_suspended = false;
    return ESP_OK;
}

esp_err_t nrf24_recv_suspend(nrf24_handle_t *h) {
    if (!h->rx_active || h->rx_suspended) return ESP_ERR_INVALID_STATE;
    gpio_set_level(CE_GPIO, 0);                          // RX -> standby-I
    ESP_ERROR_CHECK(gpio_intr_disable(IRQ_GPIO));
    vTaskSuspend(h->task);
    h->rx_suspended = true;
    return ESP_OK;
}

esp_err_t nrf24_recv_resume(nrf24_handle_t *h) {
    if (!h->rx_active || !h->rx_suspended) return ESP_ERR_INVALID_STATE;
    // Clear any stale flags the chip accumulated while paused.
    nrf_wreg8(NRF_REG_STATUS, NRF_ST_RX_DR | NRF_ST_TX_DS | NRF_ST_MAX_RT);
    nrf_cmd(NRF_CMD_FLUSH_RX);
    ESP_ERROR_CHECK(gpio_intr_enable(IRQ_GPIO));
    vTaskResume(h->task);
    gpio_set_level(CE_GPIO, 1);
    esp_rom_delay_us(NRF_TSTBY2A_US);
    h->rx_suspended = false;
    return ESP_OK;
}

esp_err_t nrf24_recv_deinit(nrf24_handle_t *h) {
    if (!h->rx_active) return ESP_ERR_INVALID_STATE;
    ESP_ERROR_CHECK(gpio_isr_handler_remove(IRQ_GPIO));
    gpio_set_level(CE_GPIO, 0);
    nrf_wreg8(NRF_REG_CONFIG,                             // PWR_UP = 0 -> power-down
              NRF_CFG_MASK_TX_DS | NRF_CFG_MASK_MAX_RT
              | NRF_CFG_EN_CRC   | NRF_CFG_CRCO
              | NRF_CFG_PRIM_RX);
    if (h->task) { vTaskDelete(h->task); h->task = NULL; }
    spi_bus_remove_device(s_spi); s_spi = NULL;
    spi_bus_free(SPI2_HOST);
    h->spi = NULL;
    h->rx_active = false; h->rx_suspended = false;
    return ESP_OK;
}
```

#### G.7.5 Design notes

- **ISR body is tiny.** Only `vTaskNotifyGiveFromISR` + `portYIELD_FROM_ISR`.
No SPI, no logging, no blocking. The RX task does all the real work
(FIFO drain, trigger dispatch, flag clear) — cache-disabled windows
during flash writes are safe because the ISR is in IRAM and doesn't
touch flash.
- **FIFO drain order.** The RX task follows the STATUS-register
footnote verbatim: per packet, *read payload → W1C RX_DR →
re-check FIFO_STATUS*. The per-packet clear matters because
`RX_DR` is edge-triggered on packet arrival — if we batched the
clear to once per drain, a packet landing between the final
FIFO check and the clear would sit in the FIFO with `RX_DR`
already asserted, and the chip would never produce another
edge for it; the stuck packet would ride into the next genuine
arrival and fire two `rf_trigger_on_packet` calls for one real
event. The RX FIFO holds up to 3 packets, so a single IRQ
typically represents one but can batch up to three.
- **Dynamic Payload Length** (DPL) is enabled by default
(`CONFIG_RECEIVER_NRF24_ENABLE_DPL=y`). The bringup unconditionally
runs the chip-probe sequence (§G.7.6) so FEATURE becomes writable
on the legacy nRF24L01 exactly the same way it already is on the
nRF24L01+; DPL is then programmed identically on both. When DPL is
off, the chip returns to the reset-default static-payload path and
`RX_PW_P1` is programmed to 32 bytes. The matcher (§G.8) only
inspects `rf_code_bits/8` bytes in either mode.
- **DPL is a two-sided config.** A receiver in DPL mode expects the
Enhanced ShockBurst packet control field (length) to be present; a
sender without DPL produces frames that fail CRC on this side. The
transmitter must be configured to match — if you have mixed-mode
senders, set `CONFIG_RECEIVER_NRF24_ENABLE_DPL=n`.
- **Address byte order.** The datasheet specifies multi-byte
registers ship **LSByte first** on the wire. The five
`CONFIG_RECEIVER_NRF24_RX_ADDR_Bn` bytes are written in index
order — so `B0` is the first byte clocked out and matches the
byte a transmitter puts in its `TX_ADDR[0]` slot. Document this
convention with the Kconfig prompt (§I.2).
- **CE handling.** CE is plain GPIO toggled by us — not by
`.spics_io_num`. Keep CE high for the full RX window; set low to
enter Standby-I on suspend. Only the low→high edge (Standby-I →
RX) waits `NRF_TSTBY2A_US` (datasheet Tstby2a, 130 µs max); the
high→low edge is immediate.
- **SPI CS handling.** `.spics_io_num = CONFIG_RECEIVER_NRF24_CS`
lets the ESP-IDF driver toggle CS automatically — exactly once per
`spi_device_polling_transmit`, matching the "one command + data
bytes per CS-low window" contract in datasheet §8.3.1.
- The SPI examples now use named local config structs (`bus_cfg`,
`dev_cfg`) with designated initializers, matching the same nested
config style used in the Wi-Fi and MQTT examples.
- **Status capture.** Every SPI transaction clocks STATUS out on
MISO during the command byte. `nrf_xfer` returns it, so any caller
that cares (e.g. a future transmit path) can inspect TX_FULL /
MAX_RT without a separate NOP.
- **Single instance.** The file-static `s_spi` / `s_notify_to` and
the single `nrf24_handle_t` in `rf_supervisor` (§G.2) assume one
nRF24 per board — true for v1.

#### G.7.6 nRF24L01 vs nRF24L01+ compatibility & chip probe

The driver targets both chip revisions from a single source tree, with
no Kconfig switch for chip type. This section is the reference for the
per-register / per-command differences that justify that stance, and
the probe sequence that makes it safe.

**Address space.** Register addresses 0x00..0x17, 0x1C, 0x1D are
identical on both chips. Most reset values match as well; the
named/reset differences that matter to this driver are:


| Site                                              | nRF24L01 (PS v2.0 §9)                 | nRF24L01+ (PS v1.0 §9)                                 | Driver handling                                                                                                          |
| ------------------------------------------------- | ------------------------------------- | ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| `RF_SETUP[7]`                                     | Reserved (must be 0)                  | `CONT_WAVE` — test-mode carrier TX                     | We always write 0.                                                                                                       |
| `RF_SETUP[5]`                                     | Reserved (must be 0)                  | `RF_DR_LOW` — 250 kbps                                 | We always write 0 (250 kbps is +-only and intentionally unsupported).                                                    |
| `RF_SETUP[3]`                                     | `RF_DR` (0 = 1 M / 1 = 2 M)           | `RF_DR_HIGH` (same encoding when `RF_DR_LOW = 0`)      | Written per Kconfig.                                                                                                     |
| `RF_SETUP[0]`                                     | `LNA_HCURR` (1 = +1.5 dB RX; reset 1) | Obsolete / don't-care (writes ignored)                 | Always write 1 (`NRF_RF_LNA_HCURR`).                                                                                     |
| `RF_SETUP` reset                                  | `0x0F`                                | `0x0E`                                                 | Driver overwrites; reset delta is informational.                                                                         |
| Reg 0x09 name                                     | `CD` (Carrier Detect)                 | `RPD` (Received Power Detector, −64 dBm, 40 µs filter) | Driver doesn't touch.                                                                                                    |
| `FEATURE` (0x1D) / `DYNPD` (0x1C) / `R_RX_PL_WID` | Gated by `ACTIVATE` + `0x73`          | Accessible unconditionally                             | Unlocked at boot via `nrf_probe_chip` (see below).                                                                       |
| SPI opcode `0x50`                                 | `ACTIVATE`                            | Not in command set                                     | Issued only after a FEATURE-write has been shown to fail → only sent to what we've already classified as legacy silicon. |
| `Tpd2stby` worst case                             | 1.5 ms (internal crystal)             | 4.5 ms (Ls = 90 mH crystal)                            | `NRF_TPD2STBY_MS = 10` covers both.                                                                                       |
| 250 kbps sensitivity row                          | N/A                                   | −94 dBm                                                | Rate not offered; irrelevant.                                                                                            |


Bits not listed (`RF_SETUP[4]` PLL_LOCK, `[2:1]` RF_PWR, CONFIG, STATUS,
FIFO_STATUS, EN_AA, EN_RXADDR, SETUP_AW, SETUP_RETR, RF_CH, RX_ADDR_*,
TX_ADDR, RX_PW_P*, pinout, state-machine timings Tstby2a / Thce /
Tpece2csn) are bit-for-bit identical between the two chips.

**Probe sequence (`nrf_probe_chip`).** Run once at boot, before any
FEATURE / DYNPD / DPL write.

```
1. Write FEATURE := EN_DPL (0x04).
2. Read FEATURE back.
3a. If readback == 0x04 → writes are unlocked. Classify as nRF24L01+.
3b. If readback == 0x00 → writes are gated.
    i.  Issue ACTIVATE (SPI cmd 0x50) + magic 0x73.
    ii. Write FEATURE := EN_DPL again, read back.
    iii. If now 0x04  → classify as nRF24L01 (legacy).
         If still 0x00 → return NRF_CHIP_UNKNOWN (bus fault, not a chip).
4. In every success path, restore FEATURE := 0x00 so the main
   bringup can re-program it from a known state.
```

Only one step has a residual risk: step 3b-i. If a `+` somehow returned
0 in step 2, we would send `0x50 + 0x73` to a chip whose datasheet does
not document that opcode. The plan therefore confines ACTIVATE to this
failure path, instead of issuing it on every boot. If lab testing ever
shows contrary behavior on `+` silicon, this probe must be revisited.

**ACK-payload / W_TX_PAYLOAD_NOACK stay unused.** The probe dance
unlocks those features on the legacy chip too, but this firmware is
a receive-only trigger — it never generates ACK payloads nor issues
no-ACK transmits. `NRF_FEATURE_EN_ACK_PAY` and `NRF_FEATURE_EN_DYN_ACK`
are declared for completeness and left at 0.

#### G.7.7 Address registers — which chip writes what, and byte order

Both chips carry **both** TX and RX address registers (same register
map on both sides); the chip's role (`CONFIG.PRIM_RX` bit) decides
which ones matter at runtime.


| Register                      | Width    | Role                                                                                                       | Meaningful in                        | Used by us?                                             |
| ----------------------------- | -------- | ---------------------------------------------------------------------------------------------------------- | ------------------------------------ | ------------------------------------------------------- |
| `TX_ADDR` (0x10)              | 5 B      | Destination address for outgoing packets                                                                   | PTX (`PRIM_RX = 0`)                  | **No** — we are receive-only (`PRIM_RX = 1`).           |
| `RX_ADDR_P0` (0x0A)           | 5 B      | Pipe 0 receive address. On a PTX with auto-ACK, this must equal `TX_ADDR` so the chip can receive the ACK. | PTX (auto-ACK) + PRX (pipe 0 in use) | **No** — we listen on pipe 1 only (`EN_RXADDR = 0x02`). |
| `RX_ADDR_P1` (0x0B)           | 5 B      | Pipe 1 receive address                                                                                     | PRX                                  | **Yes** — our pipe.                                     |
| `RX_ADDR_P2..P5` (0x0C..0x0F) | 1 B each | Pipes 2–5 receive address (LSByte only; upper 4 B shared with P1)                                          | PRX                                  | No.                                                     |


For **this firmware** (PRX only): we write 5 bytes to `RX_ADDR_P1`.
`TX_ADDR` is left at its reset default and never touched — a transmit
path would be a separate module in a future extension.

For **the transmitter** (out-of-scope for this doc, but the same
driver shape would apply): set `TX_ADDR` and `RX_ADDR_P0` to the
same 5 bytes (auto-ACK path), then pulse CE.

**Byte order on the wire (nRF24L01 PS v2.0 §8.3.1 / nRF24L01+ PS v1.0
§8.3.1):** multi-byte registers are clocked **LSByte first, MSBit
first within each byte**. The 5-byte SPI payload for a
`W_REGISTER(RX_ADDR_P1, …)` therefore starts with the byte that the
radio will match *first in time* against the preamble.

**Concrete example.** Agreed-on address (as you'd write it in a logic
trace from earliest-transmitted to latest): `0xC3 0x5A 0x5A 0x5A 0xE7`.


|                                               | Value  | Where it lives                                                   |
| --------------------------------------------- | ------ | ---------------------------------------------------------------- |
| Wire-order byte 0 (LSByte, first out of MOSI) | `0xC3` | `RX_ADDR_P1[0]` on the receiver; `TX_ADDR[0]` on the transmitter |
| Wire-order byte 1                             | `0x5A` | `RX_ADDR_P1[1]` / `TX_ADDR[1]`                                   |
| Wire-order byte 2                             | `0x5A` | `RX_ADDR_P1[2]` / `TX_ADDR[2]`                                   |
| Wire-order byte 3                             | `0x5A` | `RX_ADDR_P1[3]` / `TX_ADDR[3]`                                   |
| Wire-order byte 4 (MSByte, last out of MOSI)  | `0xE7` | `RX_ADDR_P1[4]` / `TX_ADDR[4]`                                   |


Our Kconfig exposes these as five separate hex symbols, in *wire
order*:

```
CONFIG_RECEIVER_NRF24_RX_ADDR_B0 = 0xC3   # LSByte on the wire
CONFIG_RECEIVER_NRF24_RX_ADDR_B1 = 0x5A
CONFIG_RECEIVER_NRF24_RX_ADDR_B2 = 0x5A
CONFIG_RECEIVER_NRF24_RX_ADDR_B3 = 0x5A
CONFIG_RECEIVER_NRF24_RX_ADDR_B4 = 0xE7   # MSByte on the wire
```

In firmware we simply `memcpy` those five bytes in index order into
the SPI payload (see `nrf_wreg(NRF_REG_RX_ADDR_P1, addr, 5)` in
§G.7.4) — no byte-swapping, no endian conversion. The transmitter
writes the **same** five bytes in the **same** index order to its
`TX_ADDR` register, and the on-air preamble match succeeds.

Common pitfall: if someone describes the address to you as a single
hex literal like `0xE75A5A5AC3`, that is the **natural** (big-endian,
MSByte-first) spelling. Read it right-to-left when converting to our
wire-order `B0..B4` byte array — `B0 = 0xC3`, `B4 = 0xE7`. The
Kconfig prompts for `B0` and `B4` explicitly label which end of the
wire they live on to avoid this flip.

### G.8 RF trigger — unified matcher

One module, two match functions. MQTT owns the write side (mutex);
the radio owns the read side.

```c
// trigger/rf_trigger.h
esp_err_t rf_trigger_set(const uint8_t *code, size_t code_len, uint8_t bits, uint32_t ver);
void rf_trigger_restore_from_cfg(const device_config_t *cfg);
void rf_trigger_on_frame (uint32_t value, uint8_t value_bits);  // 433 MHz path
void rf_trigger_on_packet(const uint8_t *pkt, size_t pkt_len);  // nRF24 path
```

```c
// trigger/rf_trigger.c
static SemaphoreHandle_t s_mtx;
static struct {
    uint8_t  code[32];
    size_t   code_len;     // bytes used (bits/8 rounded up)
    uint8_t  bits;
    uint32_t ver;
} s_trg;

static VibratorHandler s_vibrator;

void rf_trigger_on_frame(uint32_t value, uint8_t value_bits) {
    uint8_t buf[4] = { 0 }; uint8_t bits;
    xSemaphoreTake(s_mtx, portMAX_DELAY);
    size_t n = (s_trg.code_len > 4) ? 4 : s_trg.code_len;
    memcpy(buf, s_trg.code, n);          // byte-safe copy; no strict-aliasing trap on RV32
    bits = s_trg.bits;
    xSemaphoreGive(s_mtx);
    if (!bits || bits > 32) return;
    uint32_t code32 = (uint32_t)buf[0]      | ((uint32_t)buf[1] << 8)
                    | ((uint32_t)buf[2] << 16) | ((uint32_t)buf[3] << 24);
    uint32_t mask = (bits >= 32) ? 0xFFFFFFFFu : ((1u << bits) - 1);
    if ((value & mask) == (code32 & mask))
        vibrator_toggle_pulsing(&s_vibrator);
}

void rf_trigger_on_packet(const uint8_t *pkt, size_t pkt_len) {
    uint8_t buf[32]; size_t need; uint8_t bits;
    xSemaphoreTake(s_mtx, portMAX_DELAY);
    memcpy(buf, s_trg.code, s_trg.code_len);
    need = s_trg.code_len; bits = s_trg.bits;
    xSemaphoreGive(s_mtx);
    if (!bits || (bits % 8) != 0 || pkt_len < need) return;
    if (memcmp(pkt, buf, need) == 0)
        vibrator_toggle_pulsing(&s_vibrator);
}
```

Never hold the mutex across `vibrator_toggle_pulsing` — the vibrator
task may contend on its own state.

### G.9 Vibrator — existing module

`main/vibrator/vibrator.[ch]` is already ESP-IDF idiomatic and is
reused as-is from the current tree. One small hardening: the
`vibrator_active` and
`vibrator_pulsing` fields are touched from both the pulse task and
`vibrator_set_pulsing` without a lock. Acceptable for v1 (single
writer per flag, torn reads are harmless for `bool`), but if BLE or
other preemptors join later, wrap with the same trigger mutex.

### G.10 Device identity — mbedTLS on C3

The device-identity flow from §E.4 maps directly onto ESP-IDF's
mbedTLS APIs. Hardware SHA-256 and big-int acceleration are enabled by
default on C3. Default Kconfig gates:

```kconfig
CONFIG_MBEDTLS_HARDWARE_SHA=y
CONFIG_MBEDTLS_HARDWARE_MPI=y
CONFIG_MBEDTLS_ECDSA_C=y
CONFIG_MBEDTLS_ECP_DP_SECP256R1_ENABLED=y
```

Perf budget on C3 @ 160 MHz: ECDSA sign ≈ 100–150 ms, verify ≈ 70–100
ms. Command-verification is not a hot path (rotation / deactivation
only); loading the pinned backend key is a one-shot at boot.

The pinned backend command-signing public key is still supplied through
`CONFIG_RECEIVER_BACKEND_PUBKEY_B64`.

---

## H. Backend Contract

The backend must manage ESP-01 and ESP32-C3 receivers through **one**
logical contract. Do not fork topic names, payload field names,
envelope types, or management flags by hardware family. The only
device-family-specific inputs are `hardware_model`, the legal
`receiver_type` set for that model, and the RF-code width rules.

### H.1 Cross-device invariants

These items must stay identical for ESP-01 and ESP32-C3 devices so the
same broker, database, and admin workflows can manage both:

| Area                     | Shared requirement                                                                                                      |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------- |
| Topics                   | `receiver/bootstrap/register`, `receiver/bootstrap/{challenge_id}`, `receiver/device/{public_id}/cmd/rf_code`, `receiver/device/{public_id}/cmd/deact`, `receiver/device/{public_id}/ack` |
| Payload versioning       | every wire payload carries `schema_version = 1`                                                                         |
| Bootstrap envelope types | `pending`, `challenge`, `rejected`, `result`, `response`                                                               |
| Ack discriminator        | `ack_for = "rf_code"` or `ack_for = "deact"`                                                                            |
| Deactivation actions     | `suspend`, `resume`, `decommission`                                                                                     |
| RF-code ack statuses     | `applied`, `unchanged`, `rejected`                                                                                      |
| Deact ack statuses       | `ok`, `ignored`, `rejected`                                                                                             |
| Canonical strings        | `activate-v1|...`, `rf-code-v1|...`, `deact-v1|...`                                                                     |
| Activation result fields | `public_id`, `assigned_device_name`                                                                                     |
| Topic scoping            | operational traffic is always scoped by backend-minted `public_id`                                                      |
| Re-activation semantics  | every successful activation mints a fresh `public_id`; old retained operational topics become orphaned automatically   |

Transport differences do **not** change the backend contract: the
ESP-01 stays on MQTT 3.1.1 while the ESP32-C3 may connect with MQTT v5,
but both publish the same JSON payloads on the same topics.

### H.2 Services

| Service                      | Responsibility                                                                                                                                    |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| `EnrollmentTokenService`     | issue / hash / expire / consume enrollment tokens                                                                                                 |
| `DeviceRegistrationService`  | consume `receiver/bootstrap/register`, validate token + model/type pair, create or reopen a pending device row, publish `pending`               |
| `DeviceApprovalService`      | admin approve/reject actions; on reject, publish `rejected` and wipe the bootstrap session                                                        |
| `ActivationChallengeService` | mint activation nonce, set expiry, publish `challenge`                                                                                            |
| `DeviceActivationService`    | verify the device signature over `activate-v1|...`, mint `public_id`, mark device active, publish `result`                                      |
| `RfCodeService`              | validate RF-code width by `receiver_type`, sign `rf-code-v1|...`, publish retained `cmd/rf_code`, consume RF-code acks                          |
| `DeactivationService`        | sign `deact-v1|...`, publish retained `cmd/deact`, consume deactivation acks                                                                      |
| `MqttBootstrapListener`      | subscribe to `receiver/bootstrap/register` and `receiver/bootstrap/+`; drop backend-owned envelope types on self-echo                            |
| `MqttOperationalListener`    | subscribe to `receiver/device/+/ack` and dispatch by `ack_for`                                                                                    |

### H.3 Registration validation

`DeviceRegistrationService` must enforce:

1. `hardware_model` ∈ {`ESP-01`, `ESP32-C3`}.
2. `(hardware_model, receiver_type)` is one of:

   | hardware_model | legal receiver_type              |
   | -------------- | -------------------------------- |
   | `ESP-01`       | `RECEIVER_433M`                  |
   | `ESP32-C3`     | `RECEIVER_433M`, `RECEIVER_2_4G` |

3. `schema_version`, `public_key_b64`, `enrollment_token`,
   `registration_nonce`, and `firmware_version` keep the same field
   names for both families.

Violations publish `type: "rejected"` with
`reason: "model_radio_mismatch"` or the relevant validation error.

The backend should still key device identity off the device public key,
not `public_id`, so the same physical unit can be re-activated and
re-scoped without creating duplicate hardware identities.

### H.4 Data model

```text
devices(id, public_id, public_key_der, hardware_model, receiver_type,
        firmware_version, status, assigned_name, tenant_id, site_id,
        registration_nonce, pending_since,
        created_at, activated_at, updated_at)

activation_challenges(id, challenge_id, device_pk, registration_nonce,
                      nonce, purpose, issued_at, expires_at,
                      responded_at, status)

device_rf_codes(id, device_pk, public_id, rf_code_encrypted,
                rf_code_bits, code_version, issued_at,
                last_ack_status, last_ack_at)

device_deactivation_commands(id, device_pk, public_id, command_id,
                             action, issued_by_user_id, issued_at,
                             acknowledged_at, ack_status)
```

Notes:

- `id` is the stable backend UUID; `public_id` is the rotating
  device-visible operational identifier.
- `device_pk` on child tables is the internal UUID foreign key to
  `devices.id`, not `public_id`.
- `assigned_name` is the backend-side label. The activation payload
  still publishes it as `assigned_device_name`, and the firmware stores
  that value locally as `device_name`.
- `hardware_model` records the family (`ESP-01` or `ESP32-C3`);
  `receiver_type` records the radio mode (`RECEIVER_433M` or
  `RECEIVER_2_4G`).
- `rf_code_encrypted` must be a binary/blob-capable column because the
  plaintext command can now span 1..32 bytes.
- `rf_code_bits` should be `smallint`, not `u8`/`int` constrained to
  32-bit legacy widths.
- Every `device_rf_codes` and `device_deactivation_commands` row keeps
  the `public_id` it was published under for audit and replay analysis.

Status enums stay shared across both device families:

- `devices.status` ∈ `PENDING` | `ACTIVE` | `SUSPENDED` |
  `DECOMMISSIONED` | `REJECTED`
- `activation_challenges.status` ∈ `PENDING_APPROVAL` | `ISSUED` |
  `RESPONDED` | `EXPIRED` | `REJECTED`
- `device_rf_codes.last_ack_status` ∈ `applied` | `unchanged` |
  `rejected` | `pending`
- `device_deactivation_commands.ack_status` ∈ `ok` | `ignored` |
  `rejected` | `pending`

### H.5 Backend flow

1. Admin issues an enrollment token. Only the plaintext shown once to
   the admin; the backend stores only a hash plus TTL metadata.
2. On `receiver/bootstrap/register`, validate `schema_version`,
   `hardware_model`, `receiver_type`, field presence, and token
   usability. Consume the token atomically, then look up or create the
   device row by `public_key_der`. Persist `hardware_model`,
   `receiver_type`, and `registration_nonce`; create
   `activation_challenges(status=PENDING_APPROVAL)`; publish the
   `pending` envelope on `receiver/bootstrap/{challenge_id}`.
3. Admin approves or rejects:
   - approve: create a fresh nonce, set `status=ISSUED`, publish
     `type: "challenge"`
   - reject: publish `type: "rejected"`, wipe the bootstrap session,
     and optionally keep a tombstone `devices.status = REJECTED`
4. On a valid `response`, rebuild `activate-v1|...`, verify the ECDSA
   signature against `devices.public_key_der`, mint a fresh
   `public_id`, mark the device active, and publish `type: "result"`
   with `public_id` and `assigned_device_name`. No new MQTT credential
   is issued in v1; both device families keep the provisioned broker
   credential and only rotate their operational identity.
5. Immediately after activation, `RfCodeService` assigns or restores
   the initial trigger code for that activation, validates it against
   the stored `receiver_type`, signs `rf-code-v1|...`, writes the
   `device_rf_codes` row, and publishes retained `cmd/rf_code`.
6. `cmd/rf_code` rotation reuses step 5 with a higher `code_version`
   under the same `public_id`.
7. Deactivation writes a `device_deactivation_commands` row, signs
   `deact-v1|...`, publishes retained `cmd/deact`, and updates the row
   on `ack_for = "deact"`.
8. Reprovision or re-activation always mints a new `public_id`. As
   hygiene, the backend should publish zero-length retained clears to
   the old `cmd/rf_code` and `cmd/deact` topics, but correctness does
   not depend on that cleanup.

The backend does **not** need a branch for "MQTT 3.1.1 device" vs
"MQTT v5 device" in any of the steps above. The transport-level choice
is local to the firmware; the backend lifecycle stays topic- and
payload-driven.

### H.6 RF-code validation and signing

`RfCodeService` must dispatch validation by the device row's
`receiver_type`:

- `RECEIVER_433M`: `rf_code_bits ∈ [1, 32]`,
  `hex_len == 2 * ceil(bits / 8)`
- `RECEIVER_2_4G`: `rf_code_bits ∈ [8, 256]`,
  `rf_code_bits % 8 == 0`, `hex_len == bits / 4`

Do this validation **before** signing or publishing. If the backend
signs an invalid retained payload, the device will correctly reject it,
but the retained slot will stay poisoned until the backend republishes
a valid command.

The canonical string stays identical across device families:

```text
rf-code-v1|<public_id>|<code_version>|<rf_code_hex>|<rf_code_bits>|<issued_at>
```

### H.7 Enrollment tokens

Enrollment tokens stay radio-agnostic and hardware-agnostic. Recommended
handling:

- format: opaque random string, at least 96 bits of entropy, short
  enough to re-type from the admin UI
- storage: Redis keyed by a SHA-256 (or HMAC-SHA-256) of the canonical
  token form
- issuance: `SET ... EX ... NX`
- single-use consumption: `GETDEL`
- revocation: `DEL`

Missing key on consume means invalid, expired, or already used. Never
store the plaintext token in Postgres or on the device beyond the local
pre-activation NVS slot.

### H.8 Admin API and dashboard

Admin endpoints can stay shared across both families. Path parameter
`{id}` should remain the backend UUID from `devices.id`, not the
rotating `public_id`, so admin URLs stay stable across re-activations:

- `POST /api/admin/enrollment-tokens`
- `GET /api/admin/devices`
- `POST /api/admin/devices/{id}/approve`
- `POST /api/admin/devices/{id}/reject`
- `POST /api/admin/devices/{id}/reprovision`
- `POST /api/admin/devices/{id}/rf-code`
- `POST /api/admin/devices/{id}/deactivation`

`GET /api/admin/devices` should support filtering by `status`,
`receiver_type`, and `hardware_model`. List and detail views should
show both `hardware_model` and `receiver_type` so operators can see
whether they are managing an ESP-01 433 MHz unit or an ESP32-C3 433 MHz
or 2.4 GHz unit.

The RF-code editor should adapt to `receiver_type`:

- `RECEIVER_433M` → up to 8 hex chars, common preset 24 bits
- `RECEIVER_2_4G` → up to 64 hex chars, default example 40 bits

### H.9 Broker credentials and ACLs

v1 still uses one shared MQTT credential for all receivers, regardless
of hardware family. The shared v1 protection model is:

1. `mqtts://` only
2. operational topic scoping by backend-minted `public_id`
3. backend-signed `cmd/rf_code` and `cmd/deact`

That means ESP-01 and ESP32-C3 units can coexist on the same broker
without any topic-shape change. A future per-device-ACL broker rollout
would tighten isolation, but the payload contract in this guide does
not need to change for that upgrade.

---

## I. Build Configuration

### I.1 `main/idf_component.yml`

```yaml
dependencies:
  idf:             '>=6.0.0'
  espressif/mqtt:  '~1.0.0'
  espressif/cjson: '*'
```

The current manifest in-tree still lists `jgromes/radiolib: '*'` — drop
it when this plan lands. `dependencies.lock` should then retain only the
managed `espressif__mqtt` / `espressif__cjson` components, and the
vendored `components/jgromes__radiolib/` tree can be dropped.

### I.2 Kconfig (Receiver submenu)

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
    int "433 MHz RX GPIO" default 4
    depends on RECEIVER_RADIO_433M

config RECEIVER_NRF24_SCK  int "nRF24 SCK GPIO"  default 6   depends on RECEIVER_RADIO_2_4G
config RECEIVER_NRF24_MOSI int "nRF24 MOSI GPIO" default 7   depends on RECEIVER_RADIO_2_4G
config RECEIVER_NRF24_MISO int "nRF24 MISO GPIO" default 2   depends on RECEIVER_RADIO_2_4G
config RECEIVER_NRF24_CS   int "nRF24 CS GPIO"   default 5   depends on RECEIVER_RADIO_2_4G
config RECEIVER_NRF24_CE   int "nRF24 CE GPIO"   default 3   depends on RECEIVER_RADIO_2_4G
config RECEIVER_NRF24_IRQ  int "nRF24 IRQ GPIO"  default 1   depends on RECEIVER_RADIO_2_4G

# nRF24 RF channel: f = 2400 MHz + N. Must match the transmitter.
# Lower bound 97 keeps the center ≥ 2497 MHz, above JP Wi-Fi ch 14's
# upper edge (2495 MHz) with ≥1 MHz guard at 2 Mbps. Upper bound 125
# is the silicon limit (verify local ISM rules before deployment).
config RECEIVER_NRF24_CHANNEL
    int "nRF24 RF channel (97..125 -> 2497..2525 MHz; keep above Wi-Fi band)"
    default 100
    range 97 125
    depends on RECEIVER_RADIO_2_4G

# nRF24 air data rate. Must match the transmitter. 1 Mbps gives ~3 dB
# better sensitivity than 2 Mbps and works on every nRF24L01/+ revision.
# 250 kbps is omitted — nRF24L01+ only, and we support both revisions.
choice RECEIVER_NRF24_DATA_RATE
    prompt "nRF24 air data rate"
    default RECEIVER_NRF24_DR_1M
    depends on RECEIVER_RADIO_2_4G
config RECEIVER_NRF24_DR_1M
    bool "1 Mbps"
config RECEIVER_NRF24_DR_2M
    bool "2 Mbps"
endchoice

# Enhanced ShockBurst Dynamic Payload Length. On: variable 1..32 B
# frames via R_RX_PL_WID. Off: fixed 32-byte width on pipe 1.
# Bilateral — must match the transmitter, else frames fail CRC.
# Chip compatibility (ACTIVATE 0x73 on legacy nRF24L01) is handled
# automatically by the driver (§G.7.6).
config RECEIVER_NRF24_ENABLE_DPL
    bool "Enable Dynamic Payload Length"
    default y
    depends on RECEIVER_RADIO_2_4G

# Five-byte RX address on pipe 1. Byte 0 goes out first (datasheet
# §8.3.1 LSByte-first) and must match the transmitter's TX_ADDR[0].
config RECEIVER_NRF24_RX_ADDR_B0 hex "nRF24 RX addr byte 0 (LSByte on wire)" default 0xE7 depends on RECEIVER_RADIO_2_4G
config RECEIVER_NRF24_RX_ADDR_B1 hex "nRF24 RX addr byte 1" default 0xE7 depends on RECEIVER_RADIO_2_4G
config RECEIVER_NRF24_RX_ADDR_B2 hex "nRF24 RX addr byte 2" default 0xE7 depends on RECEIVER_RADIO_2_4G
config RECEIVER_NRF24_RX_ADDR_B3 hex "nRF24 RX addr byte 3" default 0xE7 depends on RECEIVER_RADIO_2_4G
config RECEIVER_NRF24_RX_ADDR_B4 hex "nRF24 RX addr byte 4 (MSByte on wire)" default 0xE7 depends on RECEIVER_RADIO_2_4G

config RECEIVER_VIBRATOR_GPIO int "Vibrator GPIO" default 10

config RECEIVER_AP_CHANNEL int "SoftAP channel" default 1
config RECEIVER_AP_PASSWORD  string "SoftAP password" default "setup1234"
config RECEIVER_WIFI_MAX_RETRY int "STA max retries before SoftAP fallback" default 6

config RECEIVER_BACKEND_PUBKEY_B64
    string "Backend command-signing public key (base64-DER EC P-256)"
    default ""

endmenu
```

### I.3 Target

```
idf.py set-target esp32c3
idf.py menuconfig       # pick radio variant, paste backend pubkey, set GPIOs
idf.py build flash monitor
```

---

## J. Audit & Open Questions

A self-review pass. Each item is either **resolved** in the text
above or is an **accepted** trade-off noted for visibility.

1. **Variant mismatch at boot (resolved, §D).** Device stores `rx_type`
  at activation; if boot-time Kconfig differs, firmware refuses to
   dispatch the radio and falls back to SoftAP recovery. Prevents a
   reflash from mismatching against a stale `public_id`/`rf_code`
   that belongs to the other radio.
2. **`rf_code` NVS widening is backward-incompatible with an ESP-01
  schema dump (resolved).** The original `rf_code` key was `u32`; the
   new firmware writes a `blob`. This only affects ESP32-C3 devices
   (ESP-01 firmware does not run on this flash image), so NVS blob-vs-u32
   is never seen on the same device. Kept the key name for code
   symmetry; bump `schema_ver` to `2` if a **single** codebase must
   read both legacy and new layouts.
3. **Backend bits/hex validation must precede signing (resolved, §H.2).**
  If the backend signs an invalid payload (e.g. hex length mismatch)
   the firmware will still reject at `rf_code_bits` bounds, but the
   ack will be `rejected` and the retained slot will be poisoned until
   admin re-rotates. Validating on the backend avoids the poison.
4. **ISR service ownership (resolved, §G.6 / §G.7).**
  `gpio_install_isr_service` is safe to attempt from either radio path:
   duplicate installs return `ESP_ERR_INVALID_STATE`, which both
   `rf_recv_init` (§G.6) and `nrf24_recv_start_task` (§G.7) tolerate.
   The nRF24 path installs with `ESP_INTR_FLAG_IRAM` so the
   IRAM-resident `nrf_isr` is dispatchable with the cache disabled.
   The current 433 MHz implementation may uninstall the service during
   deinit; that remains safe because the Kconfig gate ensures only one
   radio variant exists in a given build.
5. **No heap churn in the RX hot path (resolved, §G.7).** The native
  driver does zero allocation after `nrf24_recv_start_task`: register
   helpers use stack buffers sized for one 32-byte payload + command
   byte. No `malloc`, no `new`, no RadioLib.
6. **nRF24 ISR is a pure-C IRAM handler (resolved, §G.7).** `nrf_isr`
  is a plain `static void IRAM_ATTR` function that only calls
   `vTaskNotifyGiveFromISR` and `portYIELD_FROM_ISR`. No SPI, no
   logging, no virtual dispatch, no flash access — safe during
   cache-disabled windows (e.g. concurrent `spi_flash` writes).
7. **nRF24 pipe address distribution (accepted gap).** §G.7 takes a
  5-byte address from Kconfig. For multi-site deployments the address
   must be unique per site; fold into `cmd/rf_code` in v2 (extra field
  - re-version canonical string to `rf-code-v2|…`). v1 assumes a
   single site-wide address configured at build time.
8. **(superseded — see item 20).** Initial draft tracked nRF24 / Wi-Fi
  coexistence twice; item 20 is the live entry.
9. **MQTT credentials config shape (resolved, §G.4).** Older
  ESP-01-oriented flat `.uri/.username/.password` examples are **not**
   valid on
   `esp_mqtt_client_config_t` in the pulled-in `espressif/mqtt ^1.0`
   component; the nested form is the only accepted one. The same local
   header also requires PEM material to leave `certificate_len = 0`.
10. **Wi-Fi event base split (resolved, §G.3).** `GOT_IP` is on
  `IP_EVENT`, not `WIFI_EVENT`. A handler registered only on
    `WIFI_EVENT_ANY_ID` will never see `GOT_IP` and boot will hang
    forever at "wait for IP".
11. **GPIO strapping pins (resolved, §C.1).** ESP32-C3 straps are
  GPIO2 / GPIO8 / GPIO9 (GPIO0 is *not* a strap on C3, unlike the
    classic ESP32). The defaults avoid GPIO8 / GPIO9 entirely and use
    GPIO2 only for SPI MISO, where the nRF24 line is tri-stated while
    CSN stays high at reset. Integrators who change pins must re-audit
    boot-mode straps.
12. **Reprovision still wipes `rx_type` (resolved).** §D places
  `rx_type` in `device_cfg`, which is wiped on reprovision per the
    base §1.2 rule. On next boot, the Kconfig build tells firmware
    which radio to use — `rx_type` is re-written at next activation.
    No orphan state.
13. **Backend (hardware_model, receiver_type) allowlist (resolved, §H.1).**
  Without the allowlist, an attacker holding a valid token could
    register as `ESP-01 / RECEIVER_2_4G` and get a `public_id`. The
    allowlist closes this.
14. **Signature-verify canonical string is radio-independent (resolved).**
  `rf-code-v1|<public_id>|<code_version>|<rf_code_hex>|<rf_code_bits>|<issued_at>`
    — all fields flow through both backends unchanged, so the same
    pinned public key verifies for both radio variants.
15. **433 MHz `rf_recv_task` previously lived in `main.c` (resolved,
  §G.6).** Moved into `rf/rf_receiver.c` and bound to `rf_recv_start_task`;
    `main.c` no longer touches FreeRTOS task APIs for the RF task,
    matching the coupling rule in §C.
16. **Vibrator flag concurrency (accepted).** §G.9: single-writer
  `bool` fields, torn-read-safe on 32-bit RISC-V. If future features
    add a second writer, migrate to the trigger mutex.
17. **SPI device flags (resolved, §G.7).** `flags = 0` (full duplex):
  STATUS is latched on MISO during the command byte while the
    command byte is shifted out on MOSI, so full-duplex is mandatory.
    `.spics_io_num = CONFIG_RECEIVER_NRF24_CS` lets the driver toggle
    CS once per transaction, matching the datasheet's "one command +
    data bytes per CS window" contract (§8.3.1).
18. **Strict-aliasing / unaligned load in matcher (resolved, §G.8).**
  An earlier `*(uint32_t*)s_trg.code` on a `uint8_t[32]` would UB
    on strict-aliasing and unaligned-load on some RISC-V configs.
    Replaced with `memcpy` + explicit byte shifts — same codegen on
    `-O2`, no traps.
19. **Managed-component name mapping (resolved, §G.5 / §I.1).** The
  IDF Component Manager places `espressif/mqtt` and `espressif/cjson`
    at `managed_components/espressif__mqtt/` and
    `managed_components/espressif__cjson/` (double-underscore).
    `PRIV_REQUIRES` uses those exact names. The current vendored
    `jgromes__radiolib` tree is transitional and is removed by §I.1.
20. **nRF24 / Wi-Fi 2.4 GHz coexistence (resolved, §I.2).** The driver
  defaults to channel **100 (2500 MHz)** and Kconfig *enforces* a
    floor of 97 (2497 MHz). That keeps the nRF24 center frequency above
    even Japan-only Wi-Fi channel 14's upper edge (2495 MHz), with at
    least 1 MHz of guard for the 2 Mbps mode. The lower bound is
    compile-time because every Wi-Fi-local channel we could pick is a
    correctness hazard (silent packet loss), not a tuning knob. CRC-16
    catches any residual cross-band bit flips: failed packets are
    silently dropped by the chip and never reach `rf_trigger_on_packet`.
21. **`public_id` as client_id pre-activation (resolved, §G.4).**
  `cfg->public_id` is `NULL` before activation (NVS key absent).
    With `set_null_client_id = false`,
    `esp_mqtt_client_config_t.credentials.client_id = NULL` makes
    ESP-MQTT use its built-in
    default client ID (`ESP32_%CHIPID%`), not a broker-assigned one.
    Post-activation reconnect with the minted `public_id`; creds
    stay the same because the v1 shared MQTT account stays the same
    (§H.9).
22. **Byte-count for `rf_code_bits == 0` (resolved, §G.8).** Both
  matchers return early when `bits == 0` — that represents "no code
    stored yet" (pre-first-`cmd/rf_code`). This keeps the dormant
    pre-code state explicit on both receiver families.
23. **nRF24 address sourced from Kconfig (resolved, §I.2).** Five hex
  bytes, `B0..B4`, written in index order (= LSByte-first on the
    wire, per datasheet §8.3.1). Kconfig prompts call this out so
    integrators don't silently diverge from the transmitter's
    `TX_ADDR[0]` byte.
24. **Schema-version bump (accepted, §D).** `schema_ver = 1` is kept
  because no device ever sees a legacy layout on the same flash —
    ESP-01 firmware never runs on a C3 image and vice versa. If, in
    the future, a **single** firmware must read both legacy (`u32`
    `rf_code`) and new (`blob` `rf_code`), bump to `2` and add a
    one-shot migration in `device_config_load`.
25. **Pure-C end state (resolved in plan).** After §I.1 removes
  RadioLib, every source file in `main/` is `.c`. `main.c` defines `app_main`
    directly — no `extern "C"` wrapper, no `.cpp`, no `.hpp`.
    `espressif/cjson` and `espressif/mqtt` both ship C headers; the
    project pulls no C++-only dependency. Dependent audit items
    (earlier drafts' C/C++ contract and component-name rules for
    RadioLib) are obsolete and have been removed from this list.
26. **nRF24 driver provenance (resolved, §G.7).** Wire protocol
  cross-checked against **both** datasheets: nRF24L01 PS v2.0
    §8.3.1 Table 16 / nRF24L01+ PS v1.0 §8.3.1 Table 20 (command
    set), §6.1.7 timing tables (Tpd2stby 1.5 ms typical up to 4.5 ms
    on +-with-90-mH crystal, Tstby2a 130 µs, Thce 10 µs, same on
    both), §9 register map (STATUS W1C semantics). The RX-task
    drain / clear loop follows the STATUS-register interrupt-handling
    footnote *verbatim* — read payload → clear RX_DR → re-check
    FIFO_STATUS, per packet. An earlier draft batched the clear to
    once per drain; see §G.7.5 for the silent double-trigger race
    that forced the per-packet form.
27. **Payload-length mode is a Kconfig (resolved, §G.7 / §I.2).**
  `CONFIG_RECEIVER_NRF24_ENABLE_DPL` (default `y`) selects Dynamic
    Payload Length vs fixed 32-byte. With DPL on, the RX task reads
    per-packet width via `R_RX_PL_WID` and flushes if it exceeds 32
    (nRF24L01+ PS v1.0 §7.3.4). With DPL off, `RX_PW_P1 = 32` and
    every frame is read in full. Earlier drafts hard-pinned the
    static path to sidestep ACTIVATE 0x73 on the legacy chip; the
    probe dance (item 33) removes that constraint.
28. **CE vs CS ownership (resolved, §G.7).** CE is a plain GPIO that
  firmware drives directly; CS is owned by the ESP-IDF SPI driver
    via `.spics_io_num` and toggled once per transaction. Mixing them
    up in an earlier draft would have kept CE low during reception
    (silent failure, no packets). The Kconfig labels distinguish the
    two.
29. **SPI clock 4 MHz default (accepted).** Datasheet caps differ
  across chip revisions: **8 Mbps on nRF24L01** (PS v2.0 §8.2),
    **10 Mbps on nRF24L01+** (PS v1.0 §8.2). 4 MHz stays under both
    with healthy margin and is robust on jumper-wire rigs. Not
    exposed as a Kconfig in v1 — a production board with short
    controlled-impedance traces can bump the `clock_speed_hz`
    literal, but it must not exceed 8 MHz if there is any chance
    the board might end up populated with legacy silicon.
30. **Startup race on IRQ (resolved, §G.7).** Edge-triggered ISRs
  miss edges that fire before the handler is registered. Fix:
    `nrf_bringup` leaves CE low (chip in Standby-I); the caller
    creates the RX task, installs the IRAM ISR, *then* raises CE.
    A single `xTaskNotifyGive` kickstarts the task in case a packet
    landed in the tiny window between CE-high and the first ISR
    registration — the task drains and clears, returning the chip to
    edge-driven operation.
31. **DMA alignment vs stack buffers (resolved, §G.7).** ESP-IDF's
  SPI driver requires DMA-capable (word-aligned, internal-SRAM)
    buffers when a DMA channel is allocated. The per-transaction
    scratch buffers in `nrf_xfer` live on the stack and aren't
    guaranteed word-aligned. Passing `SPI_DMA_DISABLED` to
    `spi_bus_initialize` forces CPU-mode transfers, which is actually
    faster for ≤ 64-byte transactions and sidesteps the alignment
    contract entirely.
32. **Data rate as first-class Kconfig (resolved, §G.7 / §I.2).**
  RF_SETUP is composed from the `CONFIG_RECEIVER_NRF24_DR_{1M,2M}`
    choice and the pinned `NRF_RF_PWR_0` (0 dBm). The DR bit lives at
    RF_SETUP[3] (RF_DR_HIGH) per datasheet §9; the constants
    `NRF_RF_DR_1M` / `NRF_RF_DR_2M` encode it directly. 250 kbps
    (RF_SETUP[5], nRF24L01+-only) is deliberately omitted from both
    the header and the Kconfig choice — the driver targets nRF24L01 and
    nRF24L01+ interchangeably, so offering a mode that half of that
    silicon family cannot negotiate would be a foot-gun.
33. **Transparent chip discrimination via FEATURE probe (resolved,
  §G.7.6).** Bringup writes `FEATURE := EN_DPL` and reads back. If
    the bit sticks → `nRF24L01+`. If not → send `ACTIVATE 0x50 + 0x73`
    and retry. If the bit sticks this time → `nRF24L01`
    (legacy). If it still does not → `NRF_CHIP_UNKNOWN` (bus fault,
    caller returns `ESP_ERR_INVALID_RESPONSE`). The detected variant
    is stored in `nrf24_handle_t.chip` and logged at `INFO`. FEATURE
    is restored to 0x00 before the main config block programs it
    from a known state. Residual risk: a `+` that returned 0 in step 1
    would then receive `0x50 + 0x73`, whose behavior is not documented
    by Nordic on the `+`. Accepted vs. the alternative of sending
    ACTIVATE on every boot.
34. **DPL is a bilateral wire-format setting (accepted, §I.2 help
  text).** A DPL-enabled receiver expects the Enhanced ShockBurst
    packet control field (9-bit length) in every frame; a non-DPL
    transmitter produces frames that fail CRC here — silent packet
    loss. `CONFIG_RECEIVER_NRF24_ENABLE_DPL` Kconfig prompt says so
    explicitly. Default `y` assumes the transmitter is under our
    control and also DPL-enabled (user-confirmed).
35. **LNA_HCURR parity (resolved, §G.7.2 / §G.7.6).** RF_SETUP bit 0
  = `LNA_HCURR` on the legacy chip (1 = +1.5 dB RX sensitivity;
    reset default 1) and "obsolete / don't care" on the `+` (writes
    ignored; LNA always on). Every RF_SETUP write OR's in
    `NRF_RF_LNA_HCURR` so the legacy chip stays at peak sensitivity
    and the `+` is unaffected. A previous draft wrote RF_SETUP with
    bit 0 = 0, which would have silently knocked 1.5 dB off
    sensitivity on legacy boards.
36. **ACK-payload and NOACK stay unused (accepted, §G.7.6).** The
  FEATURE bits `EN_ACK_PAY` and `EN_DYN_ACK` become writable after
    the probe dance but we leave them at 0 — this firmware is
    receive-only and does not generate ACK payloads or issue no-ACK
    transmits. Constants are declared so a future TX path can
    enable them without revisiting the bringup.
37. **WPA3-Compatible Mode on both sides (resolved, §G.3).** ESP-IDF
  v6 exposes native WPA3-compat flags —
    `wifi_sta_config_t.disable_wpa3_compatible_mode`
    (inverse polarity, default 0 = on)
    and `wifi_ap_config_t.wpa3_compatible_mode` (default 0, we set
    to 1). STA negotiates SAE against any WPA3-advertising AP with
    automatic WPA2 fallback; `failure_retry_cnt` is paired with
    `WIFI_ALL_CHANNEL_SCAN` because that is the local SDK contract.
    SoftAP advertises WPA2 but upgrades
    capable clients to SAE. The SoftAP flag **overrides**
    `authmode` and `pairwise_cipher` per the SDK docstring —
    `authmode = WIFI_AUTH_WPA2_PSK` is left for clarity but is
    effectively redundant when compat mode is on. `pmf_cfg.capable`
    is deprecated in v6 and omitted from both configs;
    `pmf_cfg.required` is retained as the opt-in to force PMF.
38. **Config-struct style: nested designated initializers (resolved,
  §G.3 / §G.4 / §G.7.4).** Every ESP-IDF v6 config struct
    (`wifi_config_t`, `esp_mqtt_client_config_t`, `spi_bus_config_t`,
    `spi_device_interface_config_t`, `gpio_config_t`) is now populated
    as one nested compound literal at the call site — no
    declare-then-assign prelude, no post-hoc field writes for stack
    locals (the two `strlcpy` calls for `.sta.ssid` / `.sta.password`
    remain because `wifi_sta_config_t` fields are fixed-size byte
    arrays, not pointers). This matches the v6 SDK convention, keeps
    each bus config auditable in one literal, and avoids "forgot to
    zero an unset field" regressions when the struct grows in a
    future IDF revision.
39. **TX_ADDR vs RX_ADDR ownership (documented, §G.7.7).** Both chips
  carry both TX and RX address registers; role is decided by
    `CONFIG.PRIM_RX`. This firmware writes `RX_ADDR_P1` only;
    `TX_ADDR` stays at reset. Byte order on the wire is LSByte-first
    per datasheet §8.3.1 and our Kconfig symbols `B0..B4` are already
    in wire order so no in-firmware swap is needed. §G.7.7 includes
    a worked example mapping a natural big-endian hex literal
    (`0xE75A5A5AC3`) to the `B0..B4` wire-order bytes.
40. **Cross-device backend invariants (resolved, §H.1).** ESP-01 and
  ESP32-C3 receivers keep the same topic topology, envelope types,
    `ack_for` values, deactivation actions, canonical strings, and
    management-status enums. The backend branches only on
    `hardware_model`, legal `receiver_type`, and RF-code width rules;
    it does not fork the operational MQTT or JSON contract by device
    family.
