# Receiver Device Design Guide

End-to-end plan for the ESP-01 receiver firmware and its Spring Boot + Kotlin
backend. The guide is the authoritative contract: payloads, topics, states,
and ESP SDK bindings are locked here.

Scope:

1. SoftAP provisioning
2. Backend registration + challenge-response activation
3. Post-activation operation: receive an RF trigger code from the backend,
  match decoded RF frames against it, toggle continuous vibrator pulsing on
  each qualifying match
4. Remote deactivation (suspend / resume / decommission)

Cross-references:

- `ESP8266_SDK_REFERENCE.md` — condensed SDK API sheet
- `/opt/esp/ESP8266_RTOS_SDK/` — installed SDK tree, including the
`examples/provisioning/legacy/softap_prov` reference

---

## 1. Device Lifecycle

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
                              result    ─► store public_id, erase enroll_token,
                                           set op_state=PENDING_RF_CODE
                                               │
                                               ▼
                                     restart MQTT with client_id=public_id ─►
                                     subscribe receiver/device/{public_id}/cmd/# ─►
                                     first cmd/rf_code ─► store ─► op_state=ACTIVE
```

### 1.1 Persistent State (single enum, stored in NVS)

One byte drives all operational behavior:


| `op_state`        | RF task | Meaning                                        |
| ----------------- | ------- | ---------------------------------------------- |
| `PENDING_RF_CODE` | stopped | activated, waiting for first `rf_code` message |
| `ACTIVE`          | running | has RF code, matching and toggling vibrator pulsing |
| `SUSPENDED`       | paused  | backend requested temporary off                |
| `DECOMMISSIONED`  | deleted | backend requested permanent off                |


Derived, not stored:

- `UNPROVISIONED` ≡ `wifi_ssid` key missing from NVS
- `PENDING_ACTIVATION` ≡ `wifi_ssid` present, `op_state` missing, and
`enroll_token` present
- `RECOVERY_REQUIRED` ≡ `wifi_ssid` present, `op_state` missing, and
`enroll_token` missing

Transient in-RAM phases (connecting Wi-Fi, connecting MQTT, waiting for
challenge, etc.) are internal to the boot flow and do not need to be
persisted or exposed as top-level states.

### 1.2 Reprovision

Action, not a state: erase the `device_cfg` NVS namespace, reboot. The
device comes up `UNPROVISIONED`. The `identity` namespace (EC key pair)
is **not** wiped, so `public_key_b64` stays stable across
reprovisioning — this lets the backend recognize the device across
tenants. The old `public_id` and its retained MQTT topics are dropped;
the next activation mints a fresh `public_id` (§3.6).

---

## 2. NVS Schema

Two namespaces. Keys ≤ 15 chars (SDK cap).

**Namespace `identity`** — EC key pair; survives reprovision.


| Key       | Type   | Written by | Notes                    |
| --------- | ------ | ---------- | ------------------------ |
| `pubkey`  | `blob` | key gen    | DER EC P-256 public key  |
| `privkey` | `blob` | key gen    | DER EC P-256 private key |


The device has **no** self-generated identifier. For any pre-activation
context that needs a label (SoftAP SSID, local logs) the firmware uses
the last three bytes of its STA MAC. The backend assigns a `public_id`
at activation (§3.6).

**Namespace `device_cfg`** — runtime config + activation state; wiped on
reprovision.


| Key             | Type  | Written by         | Notes                                    |
| --------------- | ----- | ------------------ | ---------------------------------------- |
| `schema_ver`    | `u8`  | provisioning       | `1`; bumped when layout breaks           |
| `wifi_ssid`     | `str` | provisioning       | presence = provisioned                   |
| `wifi_pwd`      | `str` | provisioning       | —                                        |
| `mqtt_uri`      | `str` | provisioning       | must be `mqtts://...` (TLS is mandatory) |
| `mqtt_user`     | `str` | provisioning       | shared broker credential; not rotated    |
| `mqtt_pwd`      | `str` | provisioning       | shared broker credential; not rotated    |
| `enroll_token`  | `str` | provisioning       | admin-issued single-use authorization; erased after activation |
| `public_id`     | `str` | activation         | backend-assigned; topic scope (§3.6)     |
| `device_name`   | `str` | activation         | backend-assigned label                   |
| `rf_code`       | `u32` | `cmd/rf_code` MQTT | trigger code; sensitive                  |
| `rf_code_bits`  | `u8`  | `cmd/rf_code` MQTT | 1..32; commit with `rf_code`             |
| `rf_code_ver`   | `u32` | `cmd/rf_code` MQTT | monotonic; commit with `rf_code`         |
| `op_state`      | `u8`  | activation + MQTT  | see §1.1                                 |
| `last_deact_id` | `str` | `cmd/deact` MQTT   | command_id of last applied command       |


Rules:

- `rf_code`, `rf_code_bits`, `rf_code_ver` are written under one
`nvs_commit()` to avoid torn updates.
- `enroll_token` is erased in the same commit that first writes
`public_id` and `op_state`.
- `enroll_token` is also erased when a bootstrap session is terminally
rejected or abandoned after the token has already been consumed, so a
reboot returns to local recovery instead of auto-retrying a spent
token.
- Never log sensitive values: RF code, private key, MQTT password,
enrollment token.
- Reprovision wipes this namespace. Because the retained broker state
was scoped to the old `public_id`, it becomes orphaned automatically
— no epoch bookkeeping needed (§3.6). Broker-side retained cleanup is
still recommended as hygiene.

NVS encryption is out of scope for v1. Physical flash extraction is treated
as out of scope.

---

## 3. MQTT Contract

### 3.1 Topics


| Topic                                     | Direction        | QoS | Retained | Notes                                                                         |
| ----------------------------------------- | ---------------- | --- | -------- | ----------------------------------------------------------------------------- |
| `receiver/bootstrap/register`             | device → backend | 1   | no       | pubkey + receiver_type + token + nonce (§3.3)                                 |
| `receiver/bootstrap/{challenge_id}`       | bidirectional    | 1   | no       | envelope `type: "pending" | "challenge" | "rejected" | "result" | "response"` |
| `receiver/device/{public_id}/cmd/rf_code` | backend → device | 1   | **yes**  | trigger code; reusable for rotation                                           |
| `receiver/device/{public_id}/cmd/deact`   | backend → device | 1   | **yes**  | suspend / resume / decommission                                               |
| `receiver/device/{public_id}/ack`         | device → backend | 1   | no       | single ack stream; envelope `ack_for` field                                   |


Design notes:

- Retained state exists only on the two operational command topics:
one retained payload for `rf_code`, one for `deact`.
- Acks collapse into one device-upstream topic and use `ack_for` to
distinguish `rf_code` from `deact`.
- Bootstrap uses one bidirectional topic per `challenge_id`. Because
MQTT 3.1.1 has no no-local flag, both sides must drop their own
echoed envelope types. The device starts on `receiver/bootstrap/+`,
filters on `registration_nonce`, then narrows to the exact
`receiver/bootstrap/{challenge_id}` topic. After activation it drops
the bootstrap session, reconnects with `client_id = public_id`, and
stays on `receiver/device/{public_id}/cmd/#`.

### 3.2 Security

- All MQTT traffic is `mqtts://` only. Plain MQTT is never used — the
provisioning UI rejects non-TLS URIs, and the firmware refuses to
connect (or subscribe) if `mqtt_uri` does not start with `mqtts://`.
The broker must present a CA-verified cert.
- v1 uses a single shared MQTT credential (broker does not support
per-device accounts). Broker-side ACL enforcement is therefore not
part of v1; isolation rests on TLS + topic-namespace unguessability.
See §15 for the constraint and the v2 path.
- The RF code is sensitive (32 bits, long-lived, physical-action
control). Never log it, never echo it in acks.
- Tokens, commands, and bootstrap payloads are validated for
`schema_version`, shape, and field presence before any side effect.

### 3.3 Payloads

All payloads carry `"schema_version": 1`.

**SoftAP provisioning (local HTTP POST `/api/provision`):**

```json
{
  "schema_version": 1,
  "wifi": { "ssid": "OfficeWiFi", "password": "super-secret" },
  "mqtt": {
    "broker_uri": "mqtts://broker.example.com:8883",
    "username": "shared-receiver-user",
    "password": "shared-receiver-password"
  },
  "enrollment": { "token": "<opaque string, format per §12.1>" }
}
```

**Registration (`receiver/bootstrap/register`):**

```json
{
  "schema_version": 1,
  "hardware_model": "ESP-01",
  "receiver_type": "RECEIVER_433M",
  "firmware_version": "dev",
  "public_key_b64": "BASE64_DER_PUBLIC_KEY",
  "enrollment_token": "<opaque string, format per §12.1>",
  "registration_nonce": "hG7aK2pQcR"
}
```

- `receiver_type` is one of `RECEIVER_433M` | `RECEIVER_2_4G`.
- `registration_nonce` is device-generated (≥ 8 random bytes, base64url,
session-scoped). The backend echoes it in the `pending` / `challenge`
envelopes so the device can correlate them to the registration it
just sent when envelopes arrive on the wildcard
`receiver/bootstrap/+` subscription.
- The device does **not** send a self-assigned identifier. The backend
mints the device's `public_id` on successful activation (§3.6).

All bootstrap envelopes flow over the single topic
`receiver/bootstrap/{challenge_id}`. Direction is inferred from
`type`: `pending`, `challenge`, `rejected`, `result` are
backend-originated; `response` is device-originated. Both sides
ignore any envelope whose `type` they own (self-echo).

**Pending (backend-originated, `type: "pending"`):**

```json
{
  "schema_version": 1,
  "type": "pending",
  "challenge_id": "6b99a4cb-2fc0-4559-8ae4-4ad65f8f8e0d",
  "registration_nonce": "hG7aK2pQcR",
  "issued_at": "2026-04-16T12:00:00Z"
}
```

Published immediately after registration + token validation. The device
checks `registration_nonce`, narrows its subscription to the exact
`receiver/bootstrap/{challenge_id}` topic, and waits for either
`challenge` or `rejected`.

**Rejected (backend-originated, `type: "rejected"`):**

```json
{
  "schema_version": 1,
  "type": "rejected",
  "challenge_id": "6b99a4cb-2fc0-4559-8ae4-4ad65f8f8e0d",
  "reason": "admin_rejected"
}
```

Published if the admin rejects the registration. The device wipes its
in-RAM bootstrap session and stops the bootstrap flow. `reason` is
informational only.

**Challenge (backend-originated, `type: "challenge"`):**

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

Published after the admin approves (§12). The device already
narrowed its subscription via the earlier `pending` envelope, so it
simply signs and publishes the `response` envelope on the same topic.

**Canonical string signed by the device** (UTF-8, no trailing newline):

```
activate-v1|<challenge_id>|<nonce>|<issued_at>|<expires_at>
```

**Response (device-originated, `type: "response"`):**

```json
{
  "schema_version": 1,
  "type": "response",
  "challenge_id": "6b99a4cb-2fc0-4559-8ae4-4ad65f8f8e0d",
  "signature_b64": "BASE64_SIGNATURE"
}
```

The device echoes this back to its own subscription (3.1.1 no-local
is unavailable) and **must** drop any incoming envelope where
`type == "response"`.

**Activation result (backend-originated, `type: "result"`):**

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

`status` is always `"active"` here; earlier lifecycle outcomes use
their own envelope types on the same topic. The device persists
`public_id`, keeps its provisioned MQTT credential (§15), restarts the
MQTT client so `client_id` becomes `public_id`, and then waits for the
first retained `cmd/rf_code` on the new operational topic.

**RF code (`receiver/device/{public_id}/cmd/rf_code`, retained):**

```json
{
  "schema_version": 1,
  "rf_code_hex": "0A1B2C3D",
  "rf_code_bits": 24,
  "code_version": 4,
  "issued_at": "2026-04-19T10:15:00Z",
  "signature_b64": "BACKEND_ECDSA_SIGNATURE"
}
```

- `rf_code_hex` ≤ 8 hex chars (32 bits).
- `rf_code_bits` = significant LSB width used for comparison (common: 24).
- `code_version` is monotonic per `public_id`; device rejects older
versions. Because `public_id` is re-minted on every activation,
retained messages from a prior activation live on an orphan topic and
never reach the device (§3.6).
- `signature_b64` is the backend's ECDSA signature over the canonical
string below. The device verifies it with the firmware-pinned
backend public key (§3.7) and rejects any payload that fails
verification — this closes the "anyone-with-the-shared-broker-cred
can publish a fake RF code" gap (§15).

**Canonical string for RF-code signing** (UTF-8, no trailing newline):

```
rf-code-v1|<public_id>|<code_version>|<rf_code_hex>|<rf_code_bits>|<issued_at>
```

Firmware and backend must produce byte-identical canonical strings.
Fields are joined with `|`, in the order shown, with the exact spellings
used on the wire (`rf_code_hex` upper-cased hex, `issued_at` in the ISO
8601 Z form, `rf_code_bits` and `code_version` as unprefixed base-10
integers).

**Deactivation command (`receiver/device/{public_id}/cmd/deact`,
retained):**

```json
{
  "schema_version": 1,
  "action": "suspend",
  "command_id": "2f1c4b40-8b6b-4d17-9c43-2d4b98c7ce71",
  "issued_at": "2026-04-19T10:20:00Z",
  "signature_b64": "BACKEND_ECDSA_SIGNATURE"
}
```

`action` ∈ `suspend` | `resume` | `decommission`. `signature_b64` is
the backend's ECDSA signature over the canonical string below,
verified with the same pinned backend public key used for
`cmd/rf_code` (§3.7). Verification failure ⇒ ack
`status=rejected`, no state changes.

**Canonical string for deact signing** (UTF-8, no trailing newline):

```
deact-v1|<public_id>|<command_id>|<action>|<issued_at>
```

**Single ack stream (`receiver/device/{public_id}/ack`):**

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

- `ack_for` ∈ `rf_code` | `deact`.
- For `rf_code`: `status` ∈ `applied` | `rejected` | `unchanged`.
Never echo the raw code.
- For `deact`: `status` ∈ `ok` | `ignored` | `rejected`. `rejected`
covers RF lifecycle failures from the supervisor (§9.5); in that
case the device does **not** persist the new `op_state`.

### 3.4 Cryptography

Two EC P-256 key pairs are in play:

1. **Device key pair** — generated on the ESP-01 on first boot, stored
  in the `identity` NVS namespace, used to sign the activation
   challenge. Public half travels to the backend at registration time.
2. **Backend command-signing key pair** — generated **externally**
  (offline), private half wired into the backend via Application
   Properties, public half pinned in firmware via Kconfig (§3.7).
   Used to sign every operational `cmd/`* payload (`cmd/rf_code` and
   `cmd/deact`).

Common settings:

- Key type: EC P-256 (`MBEDTLS_ECP_DP_SECP256R1`)
- Signature algorithm: `SHA256withECDSA`
- Challenge nonce: ≥ 16 random bytes, base64url
- Registration nonce: ≥ 8 random bytes, base64url, session-scoped
- Device private key: DER in NVS (smaller than PEM)
- Device public key on wire: DER → base64
- Backend command-signing public key in firmware: base64-DER (one
Kconfig string, §3.7)

### 3.5 Identifiers

- `public_id`: backend-minted, opaque to the device, scopes all
operational MQTT topics. Rotated on every activation so retained
state from prior activations is automatically orphaned.
- `challenge_id`, `command_id`: backend-generated UUIDs.
- `registration_nonce`: device-generated, single-session, binds a
challenge back to the registration that triggered it.
- `enrollment_token`: single-use, short expiry, opaque string. Format
is backend-defined — the `ENROLL-7GQ3-4TRP-9K1M` style used in
examples is illustrative only. The device treats it as an opaque
UTF-8 string. See §12 for the backend's Redis-based storage and
lifecycle.

### 3.6 Topic Invalidation via `public_id`

Earlier drafts used an `activation_epoch` to invalidate retained
messages across reprovisioning. Because the backend now re-mints
`public_id` on every activation, the same property falls out of the
topic scheme for free:

- Operational topics are scoped by `public_id`. Old retained
`cmd/rf_code` or `cmd/deact` payloads live on orphaned paths and are
never delivered to a newly activated device.
- `code_version` is monotonic only within one `public_id`; a new
activation starts again from version 1 because it is a new topic
namespace.
- Broker-side retained cleanup (zero-length `retain=true` to the old
topics) is recommended hygiene, not a correctness requirement.

### 3.7 Backend Command-Signing Key Distribution

**Every `cmd/`* payload** (`cmd/rf_code` and `cmd/deact`) is signed by
the backend with an EC P-256 private key. The device verifies with the
matching public key compiled into firmware. This defends against the
v1 shared-broker credential weakness (§15): even if an attacker holds
the shared credential, they cannot forge a command that the device
will accept.

**Key generation.** The keypair is produced **externally** — offline,
on an operator machine, not in CI — using e.g.:

```
openssl ecparam -name prime256v1 -genkey -noout -out cmd_signing_priv.pem
openssl pkey -in cmd_signing_priv.pem -pubout -outform DER \
  | base64 -w0 > cmd_signing_pub.b64
```

Keep `cmd_signing_priv.pem` off source control. Rotating the key
requires a coordinated firmware + backend release; v1 treats it as a
long-lived key with no in-field rotation.

**Backend side.** The private key path (or PEM contents) is injected
at deploy time via Application Properties — Spring Boot pattern:

```properties
# application-production.properties (externalized, not committed)
receiver.cmdsigning.private-key-pem-path=/etc/receiver/cmd_signing_priv.pem
# or
receiver.cmdsigning.private-key-pem=-----BEGIN EC PRIVATE KEY-----\n...
```

The backend loads the key once at boot into a `java.security.PrivateKey`
via `KeyFactory.getInstance("EC")` + `PKCS8EncodedKeySpec`, and uses
`Signature.getInstance("SHA256withECDSA")` for every `cmd/*` publish.
Both `RfCodeService` and `DeactivationService` call the same signing
helper.

**Firmware side.** The matching public key is pinned at build time
via Kconfig:

```kconfig
config RECEIVER_BACKEND_PUBKEY_B64
    string "Backend command-signing public key (base64-DER EC P-256)"
    default ""
    help
        Base64-encoded DER-form EC P-256 public key. Paste the output of
          openssl pkey -pubin -inform DER -in cmd_signing_pub.der
          -pubout -outform DER | base64 -w0
        The receiver uses this key to verify every cmd/* payload
        signature (cmd/rf_code and cmd/deact). Leaving this empty
        boots the firmware but every cmd/* will be rejected with
        status=rejected at verification time.
```

At boot (device_identity module, alongside the device keypair), the
firmware base64-decodes the Kconfig string and calls
`mbedtls_pk_parse_public_key` once; the parsed `mbedtls_pk_context`
is kept in RAM for the life of the process. Signature verification on
every command receive is hashing + one ECDSA verify — budget for
≈ 300–500 ms on the 80 MHz ESP8266, acceptable for the low-frequency
command path (rotation / deactivation, not hot-path).

**What is signed.** The canonical strings in §3.3 — NOT the raw JSON.
JSON whitespace/key-ordering differences between Jackson on the
backend and cJSON on the device would otherwise break verification.
Each command type has its own versioned canonical string (`rf-code-v1|…`
and `deact-v1|…`) so the two payload shapes can evolve independently.

---

## Part 1 — ESP-01 Firmware

### 4. Module Layout

```
main/
├── main.c                       orchestrator
├── config/
│   ├── device_config.[ch]       NVS I/O, schema, reprovision
├── network/
│   ├── wifi.[ch]                STA + SoftAP + modem power save
│   └── mqtt.[ch]                MQTTS client + topic routing
├── provision/
│   └── http_server.[ch]         HTTP POST /api/provision (SoftAP only)
├── security/
│   └── device_identity.[ch]     EC P-256 gen / load / sign
├── trigger/
│   ├── rf_trigger.[ch]          stores active code, matches frames, toggles vibrator pulsing
│   └── rf_supervisor.[ch]       thin wrapper over rf/ lifecycle (no direct task ops)
├── rf/                          RF decode; owns RFHandler + task lifecycle (§4.1)
├── vibrator/                    pulse generator with continuous pulse mode
└── network/certs/mqtt_ca.pem    embedded broker CA (existing path)
```

#### 4.1 `rf/` module — existing API, one addition

The current receiver already owns its full lifecycle; the supervisor
will call straight into it. The API (per `main/rf/rf_common.h` and
`main/rf/rf_receiver.c`) is:

- `rf_recv_init(gpio, &handler)` — configure GPIO, install ISR.
- `rf_recv_suspend(&handler)` — remove ISR, `vTaskSuspend` the task,
set `rx_suspended`.
- `rf_recv_resume(&handler)` — re-add ISR, `vTaskResume` the task,
clear `rx_suspended`.
- `rf_recv_deinit(&handler)` — remove ISR, reset GPIO, uninstall ISR
service, and `vTaskDelete` the task. Do not call `vTaskDelete`
externally; `rf_recv_deinit` already does it.

Only one gap: the `rf_recv_task` function is currently defined in
`main/main.c` (static, not yet invoked — the existing `app_main` never
starts it). Move the task function into `rf/rf_receiver.c` and add a
new `rf_recv_start_task(gpio, &handler)` helper there that calls
`rf_recv_init(gpio, &handler)`, then `xTaskCreate`s the task and stores
the handle in `handler.rf_recv_handle`. When the task is moved, remove
the in-task `rf_recv_init(...)` call from the current `main.c` version —
initialization must happen exactly once in `rf_recv_start_task()`, not
again inside the task body. Make the helper a no-op when
`handler.rx_active` is already set. After this change, `main.c` and the
supervisor never touch `xTaskCreate` / `vTaskDelete` / `vTaskSuspend` /
`vTaskResume` directly for the RF task.

Coupling rule: `rf/` and `vibrator/` do **not** depend on each other.
`rf/` calls into `trigger/rf_trigger` on each decoded frame; `trigger/`
calls into `vibrator/` on match to toggle continuous pulsing. `main.c`
wires them at startup.

### 5. app_main Orchestration

```
1. nvs_flash_init + esp_netif_init + esp_event_loop_create_default
2. device_config_load() → struct in RAM;
   if stored `rf_code_bits != 0`, seed `rf_trigger` from NVS
3. if no wifi_ssid:
     wifi_start_softap() + http_server_start()
     wait PROV_DONE event → esp_restart()
4. wifi_start_sta()  [blocks on WIFI_CONNECTED_BIT]
5. device_identity_ensure()  [generates and persists EC pair on first boot]
6. mqtt_start()  [MQTT credentials from NVS — same shared creds used
   for bootstrap and operational phases]
7. if public_id / op_state absent:
     if enroll_token missing:
       wifi_start_softap() + http_server_start()
       wait new provision or factory reset
     else:
       gen registration_nonce → subscribe receiver/bootstrap/+ →
       publish receiver/bootstrap/register →
       wait for first backend envelope matching registration_nonce, then
       narrow subscription to receiver/bootstrap/{challenge_id}:
         - drop any envelope with type=="response" (self-echo)
         - "pending"   → keep waiting (admin review)
         - "rejected"  → wipe in-RAM session, erase enroll_token,
                         bring SoftAP + HTTP up
         - "challenge" → sign, publish "response" on same topic,
                         wait "result"
       on "result": store public_id + device_name,
         erase enroll_token, set op_state=PENDING_RF_CODE
         (MQTT creds are unchanged — provisioning creds stay in use)
8. restart MQTT  [stop + destroy + init/start so client_id changes
   from the broker-assigned bootstrap session to `public_id`; do this
   outside the MQTT event handler]
9. esp_wifi_set_ps(WIFI_PS_MIN_MODEM)  [after STA association]
10. subscribe receiver/device/{public_id}/cmd/# on the restarted
    session
11. dispatch on op_state:
      PENDING_RF_CODE → wait for first cmd/rf_code; that handler
                        validates, starts RF, persists ACTIVE, then acks
      ACTIVE          → rf_sup_start()  // trigger state already restored
      SUSPENDED       → keep RF task stopped; wait for signed resume
      DECOMMISSIONED  → (do nothing; stay on MQTT for reprovision)
12. idle loop
```

### 6. Flow Cases

**cmd/rf_code received.** Parse JSON. Reject (`status=rejected`) if hex

> 8 chars, `rf_code_bits` not in `1..32`, or schema mismatched. Rebuild
> the canonical string (§3.3), verify `signature_b64` against the pinned
> backend public key (§3.7). On verify failure, ack `rejected` and do
> **not** touch NVS or the RF task state. If `code_version < stored rf_code_ver`, ack `rejected` (stale). If equal, ack `unchanged`. Else:

- if `op_state == PENDING_RF_CODE`, first attempt `rf_sup_start()`. On
failure, ack `rejected`, do **not** advance `op_state`, and leave the
existing in-RAM / on-flash trigger state untouched.
- after any required start succeeds, write `{rf_code, rf_code_bits, rf_code_ver}` under one commit and update in-RAM trigger state under
a mutex.
- if `op_state` was `PENDING_RF_CODE`, persist `ACTIVE` in the same
commit as the new RF code.
- only then ack `applied` on `receiver/device/{public_id}/ack` with
`ack_for: "rf_code"`.

**cmd/deact received.** Parse JSON. Rebuild the canonical string
(§3.3), verify `signature_b64` against the pinned backend public key
(§3.7). On verify failure, ack `rejected` and do **not** touch NVS or
the RF task state. If `command_id == last_deact_id`, republish ack but
skip side effects (retained-message replay). If `op_state == DECOMMISSIONED` and `action != decommission`, ack `status=ignored`
and skip — decommission is one-way at the firmware level; only
reprovision can re-enable. Else:

- `suspend`: `rf_sup_suspend()`; persist `op_state=SUSPENDED` and
`last_deact_id`; ack.
- `resume`: restore the pre-suspend state —
if `rf_code` is stored, persist `op_state=ACTIVE` and call
`rf_sup_resume()` when the RF task already exists in suspended state,
otherwise `rf_sup_start()` (covers cold boot into `SUSPENDED` and any
prior delete);
else persist `op_state=PENDING_RF_CODE` and leave the RF task
stopped until the first `rf_code` arrives. Persist `last_deact_id`
under the same commit; ack.
- `decommission`: `rf_sup_delete()`; persist
`op_state=DECOMMISSIONED` and `last_deact_id`; ack.

NVS commit happens **before** the ack is published so a reboot after ack
comes up in the promised state.

**rf_code while suspended / decommissioned.** Store the code and version
as normal. The live trigger state in RAM is also updated, but since the RF
task is not running it has no effect until a later `resume` or
re-activation.

**Wi-Fi repeatedly fails.** After N retries (Kconfig
`CONFIG_RECEIVER_WIFI_MAX_RETRY`), bring SoftAP + HTTP up *in place*
(no reboot, no NVS wipe). `op_state` and all stored data are left
untouched. The provisioning UI (§9.8) offers three choices:

1. **Provision** — submit new Wi-Fi / MQTT / token; new creds overwrite
  the old in `device_cfg`, then `esp_restart()`.
2. **Retry connection** — `POST /api/retry`: firmware tears down SoftAP
  and reboots with existing creds unchanged. For the "the router was
   briefly down" case.
3. **Factory reset** — `POST /api/reset`: firmware erases the
  `device_cfg` namespace (but keeps the `identity` namespace with the
   EC key pair, per §1.2), then reboots into `UNPROVISIONED`. Requires
   explicit UI confirmation.

**Pending envelope received.** The backend has accepted registration +
token and is waiting for an admin decision. Confirm
`registration_nonce` matches the in-RAM value (otherwise drop — replay
or cross-device delivery). Narrow the MQTT subscription from
`receiver/bootstrap/+` to the exact
`receiver/bootstrap/{challenge_id}`. Keep Wi-Fi + MQTT up and
block. No NVS writes. While waiting, drop any envelope with
`type == "response"` (self-echo from the device's own publish on the
shared topic).

**Rejected envelope received.** Admin rejected the device. Wipe the
in-RAM bootstrap session (registration_nonce, challenge_id, any pending
challenge state), erase `enroll_token`, unsubscribe from the bootstrap
topic, and bring SoftAP + HTTP back up in place. Do not auto-retry — a
fresh token is required. The persisted state becomes the derived
`RECOVERY_REQUIRED` case.

**Challenge expired (no admin decision in time).** The backend may
drop a long-pending session; the device gets no explicit signal. After
a configurable bootstrap-timeout (default 15 min), erase
`enroll_token`, then bring SoftAP + HTTP up *in place* so the
installer can retry with a fresh token or factory-reset. This keeps the
recovery path reboot-safe.

**Challenge expired after response / result never arrives.** Do **not**
blindly re-publish registration: the token may already have been
consumed. Erase `enroll_token`, keep Wi-Fi / MQTT config, and return to
SoftAP recovery so the installer can enter a fresh token or factory
reset.

**Activation result never received.** Stay in the pre-activation loop
until the bootstrap-timeout above fires. Then take the same recovery
path: erase `enroll_token`, return to SoftAP, require a fresh token.

### 7. SoftAP

- SSID: `RECEIVER-SETUP-<last_3_mac_bytes_hex>`
- `max_connection = 1`, `beacon_interval = 100`, `channel = 1`.
- `authmode = WIFI_AUTH_WPA2_PSK`. ESP8266 SoftAP cannot do WPA3:
`wifi_ap_config_t` has no PMF field and the chip lacks SAE support on
the AP side. WPA2-PSK is the strongest the hardware supports in AP
mode. WPA3 applies only to the STA side (§9.2). If WPA3 SoftAP is a
hard requirement, the target hardware needs to change (ESP32-C3 or
later).
- AP password: Kconfig default; rotate to per-device printed password if
the deployment workflow supports it. The enrollment token remains the
authoritative backend trust anchor.
- HTTP server runs only in SoftAP mode. Endpoints: `GET /` (UI, see §9.8),
`POST /api/provision`, `GET /api/status`.
- On successful POST: validate fields, write to NVS, respond 200, set the
`PROV_DONE` event bit, `esp_restart()` after a short delay.

### 8. Power Management

- `esp_wifi_set_ps(WIFI_PS_MIN_MODEM)` **after** STA association. Not before
`esp_wifi_start()` — the SDK ignores it.
- MQTT `keepalive = 120` seconds. Longer = more modem sleep but slower dead
TCP detection.
- No heartbeat publish on a timer. MQTT keepalive pings suffice for
liveness; LWT is v2.
- Never `esp_wifi_stop()` in ACTIVE / SUSPENDED / DECOMMISSIONED — the
device must stay reachable for code rotation and deactivation commands.
- Do not use `esp_deep_sleep`. It drops the TCP session, so every wakeup
re-runs TLS/MQTT — incompatible with low-latency rotation.
- The current `rf_recv_task`'s `vTaskDelay(10)` poll is acceptable for v1.
v2: switch to a task notification from the RMT callback so the task
blocks, giving the idle task and modem sleep more runtime.

---

## 9. ESP SDK Code Guide

Everything below is verified against `/opt/esp/ESP8266_RTOS_SDK` headers.
Snippets are illustrative; elide error handling for brevity.

### 9.1 NVS I/O

Header: `nvs_flash.h`, `nvs.h`. See `ESP8266_SDK_REFERENCE.md`.

Two-step size query is mandatory for strings and blobs:

```c
esp_err_t read_str(nvs_handle_t h, const char *key, char **out) {
    size_t len = 0;
    esp_err_t err = nvs_get_str(h, key, NULL, &len);   // length includes NUL
    if (err != ESP_OK) return err;
    char *buf = malloc(len);
    if (!buf) return ESP_ERR_NO_MEM;
    err = nvs_get_str(h, key, buf, &len);
    if (err != ESP_OK) { free(buf); return err; }
    *out = buf;
    return ESP_OK;
}
```

Transactional `rf_code` write:

```c
nvs_handle_t h;
nvs_open("device_cfg", NVS_READWRITE, &h);
nvs_set_u32(h, "rf_code",      code);
nvs_set_u8 (h, "rf_code_bits", bits);
nvs_set_u32(h, "rf_code_ver",  version);
nvs_commit(h);                  // all three land atomically
nvs_close(h);
```

Caps on ESP8266: key name ≤ 15 chars; blob ≤ ~3968 bytes per entry.
Repeated writes to the same key wear flash — acceptable for rotation
frequency (hours to days), not for high-rate telemetry.

### 9.2 Wi-Fi STA and SoftAP

Headers: `esp_wifi.h`, `esp_wifi_types.h`. See
`ESP8266_SDK_REFERENCE.md`.

The firmware has no self-assigned identifier (§3.5). The SoftAP SSID is
labelled with the last three bytes of the STA MAC purely as a
human-readable hint for the installer.

SoftAP bring-up (after `esp_wifi_init` — the SDK requires init before
any `esp_wifi_*` call other than event registration):

```c
wifi_init_config_t init = WIFI_INIT_CONFIG_DEFAULT();
ESP_ERROR_CHECK(esp_wifi_init(&init));

uint8_t mac[6];
esp_wifi_get_mac(ESP_IF_WIFI_STA, mac);   // SSID label only

wifi_config_t cfg = { 0 };
int n = snprintf((char *)cfg.ap.ssid, sizeof cfg.ap.ssid,
                 "RECEIVER-SETUP-%02X%02X%02X", mac[3], mac[4], mac[5]);
cfg.ap.ssid_len        = n;
cfg.ap.channel         = 1;
cfg.ap.authmode        = WIFI_AUTH_WPA2_PSK;
cfg.ap.max_connection  = 1;           // hard-capped at 4 on ESP8266
cfg.ap.beacon_interval = 100;
strncpy((char *)cfg.ap.password, CONFIG_RECEIVER_AP_PASSWORD,
        sizeof cfg.ap.password);

ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_AP));
ESP_ERROR_CHECK(esp_wifi_set_config(ESP_IF_WIFI_AP, &cfg));
ESP_ERROR_CHECK(esp_wifi_start());
```

STA + WPA3 (SAE) + power save, after association event:

```c
wifi_config_t sta = { 0 };
strncpy((char *)sta.sta.ssid,     cfg->wifi_ssid, sizeof sta.sta.ssid);
strncpy((char *)sta.sta.password, cfg->wifi_pwd,  sizeof sta.sta.password);
sta.sta.pmf_cfg.capable = true;    // required to negotiate WPA3-SAE
// leave sta.sta.pmf_cfg.required = false so WPA2-only APs still associate
sta.sta.threshold.authmode = WIFI_AUTH_WPA2_PSK;   // reject WEP / open

ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
ESP_ERROR_CHECK(esp_wifi_set_config(ESP_IF_WIFI_STA, &sta));
ESP_ERROR_CHECK(esp_wifi_start());
// ... wait for SYSTEM_EVENT_STA_GOT_IP ...
ESP_ERROR_CHECK(esp_wifi_set_ps(WIFI_PS_MIN_MODEM));   // post-association only
```

WPA3 notes:

- The Wi-Fi driver gate is `CONFIG_ESP8266_WIFI_ENABLE_WPA3_SAE` — enable
it in `menuconfig` ("Component config → ESP8266-specific → WiFi WPA3
support"). `WIFI_INIT_CONFIG_DEFAULT()` (esp_wifi.h) propagates this
to `wifi_init_config_t.wpa3_sae_enable`, so just call
`esp_wifi_init(&WIFI_INIT_CONFIG_DEFAULT())` — no per-call flag to
set at runtime.
- `pmf_cfg.capable = true` on the STA config lets the supplicant
negotiate SAE with WPA3 and WPA2/WPA3 transition APs.
- Leave `pmf_cfg.required = false` unless the deployment is WPA3-only —
requiring PMF will fail to associate with common WPA2-PSK APs.
- `threshold.authmode = WIFI_AUTH_WPA2_PSK` blocks open / WEP networks
even if the installer typed a matching SSID.
- Avoid `WIFI_PS_MAX_MODEM`: it relies on `listen_interval` which many
APs ignore, causing dropped pings and spurious disconnects.

### 9.3 HTTP Provisioning Server

Header: `esp_http_server.h`. Reference example:
`/opt/esp/ESP8266_RTOS_SDK/examples/provisioning/legacy/softap_prov`.

```c
#define PROV_BODY_MAX 2048

static esp_err_t send_json_err(httpd_req_t *req, const char *status,
                               const char *msg) {
    httpd_resp_set_status(req, status);
    httpd_resp_set_type(req, "application/json");
    char body[96];
    int n = snprintf(body, sizeof body, "{\"error\":\"%s\"}", msg);
    return httpd_resp_send(req, body, n);
}

static esp_err_t provision_post(httpd_req_t *req) {
    if (req->content_len <= 0 || req->content_len > PROV_BODY_MAX)
        return send_json_err(req, "400 Bad Request", "bad length");

    char *buf = malloc(req->content_len + 1);
    if (!buf) return send_json_err(req, "500 Internal Error", "oom");

    size_t got = 0;
    while (got < req->content_len) {
        int n = httpd_req_recv(req, buf + got, req->content_len - got);
        if (n <= 0) { free(buf); return ESP_FAIL; }   // 0 = closed, <0 = err
        got += n;
    }
    buf[got] = '\0';

    esp_err_t err = device_config_apply_json(buf);
    free(buf);
    if (err != ESP_OK)
        return send_json_err(req, "400 Bad Request", "invalid payload");

    httpd_resp_set_type(req, "application/json");
    httpd_resp_sendstr(req, "{\"status\":\"ok\"}");
    xEventGroupSetBits(s_prov_events, PROV_DONE_BIT);
    return ESP_OK;
}

httpd_handle_t http_server_start(void) {
    httpd_config_t cfg = HTTPD_DEFAULT_CONFIG();
    httpd_handle_t h;
    ESP_ERROR_CHECK(httpd_start(&h, &cfg));
    httpd_uri_t routes[] = {
        { .uri="/",               .method=HTTP_GET,  .handler=index_get    },
        { .uri="/api/status",     .method=HTTP_GET,  .handler=status_get   },
        { .uri="/api/provision",  .method=HTTP_POST, .handler=provision_post },
        { .uri="/api/retry",      .method=HTTP_POST, .handler=retry_post   },
        { .uri="/api/reset",      .method=HTTP_POST, .handler=reset_post   },
    };
    for (size_t i = 0; i < sizeof routes / sizeof routes[0]; i++)
        httpd_register_uri_handler(h, &routes[i]);
    return h;
}
```

`index_get` is defined in §9.8; `status_get` returns JSON with
`mac_label`, `public_id` (empty pre-activation), `firmware_version`,
`op_state`, `wifi_ssid` (empty if not stored), and `has_config`.
`op_state` may be derived as `UNPROVISIONED`, `PENDING_ACTIVATION`, or
`RECOVERY_REQUIRED` when no persisted enum exists yet. Secrets never
leave the device.
`firmware_version` comes from the build system (`PROJECT_VER` /
`CONFIG_APP_PROJECT_VER`), not NVS.

`retry_post` sets `Content-Type: application/json`, returns
`{"status":"retrying"}`, signals main, which triggers `esp_restart()`
after ~500 ms. No NVS writes. `reset_post` does the same with
`{"status":"reset"}` and, before the restart, erases only the
`device_cfg` namespace (`nvs_erase_all` on its handle + commit). The
`identity` namespace (EC keys) is left intact. Main waits ~500 ms
after the relevant event bit before `esp_restart()` so the response
is flushed.

Response contract: every reachable error path in `provision_post`,
`retry_post`, and `reset_post` returns JSON (`send_json_err` or the
inline success body). The one exception is the `httpd_req_recv()` loop
bailing with `return ESP_FAIL` — that path is unreachable by a healthy
client, because `<= 0` from `httpd_req_recv` means the client has
already closed or errored the socket, so no response can be sent
anyway. The UI's `catch()` handles this as "Network error", which is
the correct label.

### 9.4 MQTT Client (TLS)

Header: `mqtt_client.h`. See `ESP8266_SDK_REFERENCE.md`. The CA PEM is already embedded by the
repo's build: `main/component.mk` has
`COMPONENT_EMBED_TXTFILES := network/certs/mqtt_ca.pem` and
`main/CMakeLists.txt` has `EMBED_TXTFILES "network/certs/mqtt_ca.pem"`.
Both paths produce the symbols `_binary_mqtt_ca_pem_{start,end}` —
`EMBED_TXTFILES` appends a NUL so the buffer is a valid C string. Refer
to the symbols directly:

```c
extern const uint8_t ca_pem_start[] asm("_binary_mqtt_ca_pem_start");
extern const uint8_t ca_pem_end[]   asm("_binary_mqtt_ca_pem_end");

static void on_mqtt_event(void *arg, esp_event_base_t base,
                          int32_t event_id, void *event_data) {
    esp_mqtt_event_handle_t e = event_data;
    switch (e->event_id) {
    case MQTT_EVENT_CONNECTED:
        // on first connect: subscribe to bootstrap OR operational set
        break;
    case MQTT_EVENT_DATA:
        // e->topic / topic_len only populated on the first fragment;
        // check e->total_data_len > e->data_len to detect fragmentation
        topic_router_dispatch(e);
        break;
    default: break;
    }
}

void mqtt_start(const device_config_t *cfg) {
    esp_mqtt_client_config_t mqtt_cfg = {
        .uri        = cfg->mqtt_uri,       // "mqtts://..."
        .username   = cfg->mqtt_user,
        .password   = cfg->mqtt_pwd,
        // client_id is fixed at client init: bootstrap starts with NULL
        // (broker auto-assigns), then activation persists public_id and
        // forces a stop/destroy/init/start so cfg->public_id becomes the
        // operational client_id. Credential stays shared in both phases.
        .client_id  = cfg->public_id,
        .cert_pem   = (const char *)ca_pem_start,
        .cert_len   = ca_pem_end - ca_pem_start,
        .keepalive  = 120,
        .buffer_size = 2048,
    };
    s_client = esp_mqtt_client_init(&mqtt_cfg);
    esp_mqtt_client_register_event(s_client, ESP_EVENT_ANY_ID,
                                   on_mqtt_event, NULL);
    esp_mqtt_client_start(s_client);
}
```

`esp_mqtt_client_publish` is thread-safe and safe to call from inside
`on_mqtt_event`. `MQTT_EVENT_DATA` can fire multiple times for a single
message — always check `total_data_len` and buffer the fragments before
JSON-parsing.

### 9.5 RF Task Supervisor

Headers: `rf/rf_common.h`. The supervisor is a four-line wrapper over
the existing `rf/` API (§4.1). No raw FreeRTOS task calls, no handle
ownership, no ISR bookkeeping — `rf/` owns all of that.

```c
#include "rf/rf_common.h"
extern RFHandler g_rf;             // defined and initialized in main.c

// ESP-01 exposes exactly two usable GPIOs. Pin assignment is fixed:
//   GPIO0 → RF receiver interrupt
//   GPIO2 → vibrator output
// No Kconfig knob is needed; the constants live in the firmware.
#define RF_RX_GPIO      GPIO_NUM_0
#define VIBRATOR_GPIO   GPIO_NUM_2

esp_err_t rf_sup_start  (void) { return rf_recv_start_task(RF_RX_GPIO, &g_rf); }
esp_err_t rf_sup_suspend(void) { return rf_recv_suspend(&g_rf); }
esp_err_t rf_sup_resume (void) { return rf_recv_resume (&g_rf); }
esp_err_t rf_sup_delete (void) { return rf_recv_deinit (&g_rf); }
```

Notes:

- ESP-01 hardware note: GPIO1/GPIO3 are UART0 TX/RX and stay with the
console. GPIO0 and GPIO2 are the only two pins broken out on the
module header; the design reserves them for RF RX and the vibrator
respectively. A port to a module with more GPIOs would reintroduce
a Kconfig knob.
- `rf_recv_suspend` removes the ISR before suspending the task, so
interrupts do not queue timings against a non-running consumer.
- `rf_recv_deinit` fully tears down task + ISR + GPIO. After a delete,
`rf_sup_start` re-inits everything from scratch.
- Every call returns `esp_err_t`; the MQTT deactivation handler acks
`rejected` (not `ok`) on failure and does not persist the new
`op_state`.

### 9.6 Device Identity (mbedTLS EC P-256)

Headers: `mbedtls/pk.h`, `mbedtls/ecp.h`, `mbedtls/ctr_drbg.h`,
`mbedtls/entropy.h`, `mbedtls/sha256.h`. See
`ESP8266_SDK_REFERENCE.md`.

Confirmed enabled in SDK default config:
`MBEDTLS_ECDSA_C`, `MBEDTLS_ECP_DP_SECP256R1_ENABLED`,
`MBEDTLS_ECDSA_DETERMINISTIC`.

Keygen + DER export:

```c
mbedtls_pk_context pk;
mbedtls_entropy_context ent;
mbedtls_ctr_drbg_context drbg;
mbedtls_pk_init(&pk);
mbedtls_entropy_init(&ent);
mbedtls_ctr_drbg_init(&drbg);
mbedtls_ctr_drbg_seed(&drbg, mbedtls_entropy_func, &ent, NULL, 0);

mbedtls_pk_setup(&pk, mbedtls_pk_info_from_type(MBEDTLS_PK_ECKEY));
mbedtls_ecp_gen_key(MBEDTLS_ECP_DP_SECP256R1, mbedtls_pk_ec(pk),
                    mbedtls_ctr_drbg_random, &drbg);

uint8_t der[160];
int n = mbedtls_pk_write_pubkey_der(&pk, der, sizeof der);
// NB: mbedTLS writes DER at the END of the buffer.
//     The public key is at (der + sizeof der - n), length n.
```

Signing the canonical string:

```c
uint8_t hash[32];
mbedtls_sha256(canonical, canonical_len, hash, 0);

uint8_t sig[MBEDTLS_PK_SIGNATURE_MAX_SIZE];
size_t siglen;
mbedtls_pk_sign(&pk, MBEDTLS_MD_SHA256, hash, 32, sig, &siglen,
                mbedtls_ctr_drbg_random, &drbg);
```

Perf note: one P-256 sign ≈ 500–800 ms on the 80 MHz ESP8266. That's fine
for activation (once) but not a hot path. Cache the `mbedtls_pk_context`
between boots via DER in NVS; re-seed the DRBG only on boot.

**Pinned backend public key load (once, at boot):**

```c
static mbedtls_pk_context g_backend_pubkey;

esp_err_t backend_pubkey_load(void) {
    const char *b64 = CONFIG_RECEIVER_BACKEND_PUBKEY_B64;
    if (!b64 || !*b64) return ESP_ERR_INVALID_STATE;  // Kconfig empty
    uint8_t der[160];
    size_t der_len = 0;
    if (mbedtls_base64_decode(der, sizeof der, &der_len,
                              (const uint8_t *)b64, strlen(b64)) != 0)
        return ESP_ERR_INVALID_ARG;
    mbedtls_pk_init(&g_backend_pubkey);
    return mbedtls_pk_parse_public_key(&g_backend_pubkey, der, der_len) == 0
             ? ESP_OK : ESP_ERR_INVALID_ARG;
}
```

**Verify any `cmd/`* signature** — same helper is called by the
`cmd/rf_code` and `cmd/deact` handlers with their respective canonical
strings:

```c
bool backend_verify(const char *canonical, size_t canonical_len,
                    const uint8_t *sig, size_t sig_len) {
    uint8_t hash[32];
    mbedtls_sha256((const uint8_t *)canonical, canonical_len, hash, 0);
    return mbedtls_pk_verify(&g_backend_pubkey, MBEDTLS_MD_SHA256,
                             hash, 32, sig, sig_len) == 0;
}
```

ECDSA verify is ≈ 300–500 ms on ESP8266 — acceptable for the low-rate
command path, not a per-RF-frame operation. Canonical strings are the
exact `rf-code-v1|…` and `deact-v1|…` forms defined in §3.3.

### 9.7 RF Trigger — Concurrency

The MQTT task writes the trigger state, the RF task reads it. Keep a mutex
guarding a small struct copy; don't hold it across the vibrator toggle.

```c
static SemaphoreHandle_t s_mtx;
static struct { uint32_t code; uint8_t bits; uint32_t ver; } s_trg;

void rf_trigger_set(uint32_t c, uint8_t b, uint32_t v) {
    xSemaphoreTake(s_mtx, portMAX_DELAY);
    s_trg.code = c; s_trg.bits = b; s_trg.ver = v;
    xSemaphoreGive(s_mtx);
}

void rf_trigger_restore_from_cfg(const device_config_t *cfg) {
    if (!cfg->rf_code_bits) return;
    rf_trigger_set(cfg->rf_code, cfg->rf_code_bits, cfg->rf_code_ver);
}

void rf_trigger_on_frame(uint32_t decoded, uint8_t decoded_bits) {
    uint32_t code; uint8_t bits;
    xSemaphoreTake(s_mtx, portMAX_DELAY);
    code = s_trg.code; bits = s_trg.bits;
    xSemaphoreGive(s_mtx);
    if (!bits) return;
    uint32_t mask = (bits >= 32) ? 0xFFFFFFFFu : ((1u << bits) - 1);
    if ((decoded & mask) == (code & mask))
        vibrator_toggle_pulsing(&s_vibrator);   // outside the lock
}
```

### 9.8 Provisioning Page UI

Served by the HTTP handler (§9.3) when the device is in SoftAP mode. One
self-contained HTML file, gzipped at build time and embedded into the
firmware binary — no external assets, no CDN, no web fonts. SoftAP mode
has no route to the internet; any missing asset is a broken install.

Budget & design constraints:

- ≤ 8 KB uncompressed, ≤ 3 KB gzipped.
- Translucent "liquid glass" card over a soft gradient backdrop
(`backdrop-filter: blur()` with `-webkit-` prefix).
- Cross-browser fallback via `@supports not (backdrop-filter: ...)` to a
semi-opaque solid card. Verified-working: Safari 9+, Chrome 76+, Edge
79+, Firefox 103+, Samsung Internet 12+, Android WebView. Older engines
get the solid fallback — still readable and functional.
- System font stack — `-apple-system` first so Apple hardware renders in
San Francisco for the native look; other OSes fall through.
- No JS framework. Vanilla `fetch`, a single IIFE, no build step for JS.

Form fields (match the `POST /api/provision` schema in §3.3):


| Field            | Input type  | Rules                                                                                 |
| ---------------- | ----------- | ------------------------------------------------------------------------------------- |
| Wi-Fi SSID       | text        | required, max 32                                                                      |
| Wi-Fi password   | password    | required, max 63, show/hide toggle                                                    |
| MQTT URI         | text (mono) | required, pattern `mqtts://.+` (TLS-only)                                             |
| MQTT username    | text        | required                                                                              |
| MQTT password    | password    | required, show/hide toggle                                                            |
| Enrollment token | text (mono) | required, 8..128 chars; format is backend-defined (§12.1) so no client-side transform |


Read-only metadata pulled from `GET /api/status` at page load:
`mac_label` (the last three MAC bytes shown as hex — matches the SoftAP
SSID suffix so installers can confirm they're on the right device),
`public_id` (only populated once activated), `firmware_version`,
`op_state`, and `wifi_ssid`. Never surface RF code, private key, MQTT
password, or any activation secret — the provisioning flow doesn't
need them.

Primary submit flow: inline validation → `fetch('/api/provision', ...)` →
status updates → on 200 the button disables and the page says "Device
restarting…".

Recovery panel (shown only when `has_config` is true — i.e. the
`device_cfg` namespace is non-empty):

- **Retry connection** → `POST /api/retry`. Shown **only if `wifi_ssid`
is present**; a retry with no stored creds would just loop back to
SoftAP. Device reboots and retries the existing Wi-Fi creds.
- **Factory reset** → `POST /api/reset`, guarded by a `confirm()` prompt.
Always shown inside the recovery panel so partial/corrupt state can
still be wiped. Device erases `device_cfg` and reboots to
`UNPROVISIONED`. Identity and EC keys survive (§1.2).

`**main/provision/index.html`:**

```html
<!doctype html>
<html lang="en"><head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>Receiver Setup</title>
<style>
  :root{
    --fg:#111;--fg-dim:#555;--err:#c1272d;--ok:#1f6f3a;
    --glass:rgba(255,255,255,.55);--border:rgba(255,255,255,.7);
    --shadow:0 10px 40px rgba(0,0,0,.25);
  }
  *{box-sizing:border-box}
  html,body{margin:0;height:100%;font:15px/1.4 -apple-system,BlinkMacSystemFont,
    "SF Pro","Segoe UI",Roboto,system-ui,sans-serif;color:var(--fg)}
  body{
    min-height:100%;
    background:
      radial-gradient(at 20% 10%,#a7c7ff 0%,transparent 55%),
      radial-gradient(at 85% 20%,#ffb4d1 0%,transparent 50%),
      radial-gradient(at 60% 90%,#b8f1cf 0%,transparent 60%),
      linear-gradient(135deg,#e9edf3,#f5ecf7);
    background-attachment:fixed;
    display:flex;align-items:center;justify-content:center;padding:24px;
  }
  .card{
    width:100%;max-width:420px;padding:24px 22px;border-radius:22px;
    background:var(--glass);
    -webkit-backdrop-filter:blur(22px) saturate(160%);
            backdrop-filter:blur(22px) saturate(160%);
    border:1px solid var(--border);
    box-shadow:var(--shadow),inset 0 1px 0 rgba(255,255,255,.75);
  }
  @supports not ((backdrop-filter:blur(1px)) or (-webkit-backdrop-filter:blur(1px))){
    .card{background:rgba(255,255,255,.92)}
  }
  h1{margin:0 0 4px;font-size:20px;font-weight:600}
  .meta{font-size:12px;color:var(--fg-dim);margin-bottom:18px}
  .meta code{background:rgba(0,0,0,.06);padding:1px 6px;border-radius:6px}
  label{display:block;font-size:12px;color:var(--fg-dim);margin:12px 0 4px}
  .row{position:relative}
  input{
    width:100%;padding:10px 12px;border-radius:12px;
    border:1px solid rgba(0,0,0,.12);background:rgba(255,255,255,.7);
    font:inherit;color:inherit;outline:none;
    transition:border-color .15s,background .15s;
  }
  input:focus{border-color:#3a86ff;background:rgba(255,255,255,.92)}
  .toggle{
    position:absolute;right:8px;top:26px;border:0;background:transparent;
    font-size:12px;color:var(--fg-dim);cursor:pointer;
    padding:6px 8px;border-radius:8px;
  }
  .toggle:hover{background:rgba(0,0,0,.05)}
  button.submit{
    margin-top:18px;width:100%;padding:12px;border-radius:14px;border:0;
    background:linear-gradient(180deg,#3a86ff,#2563eb);color:#fff;
    font-weight:600;font-size:15px;cursor:pointer;
    box-shadow:0 4px 16px rgba(37,99,235,.35);
  }
  button.submit:disabled{opacity:.55;cursor:not-allowed}
  .status{margin-top:14px;font-size:13px;min-height:1.4em}
  .status.ok{color:var(--ok)} .status.err{color:var(--err)}
  .mono{font-family:ui-monospace,SFMono-Regular,Menlo,Consolas,monospace;
    letter-spacing:.02em}
  .recovery{display:none;margin-top:16px;padding-top:14px;
    border-top:1px solid rgba(0,0,0,.08)}
  .recovery.show{display:block}
  .recovery h2{margin:0 0 8px;font-size:11px;font-weight:600;
    color:var(--fg-dim);text-transform:uppercase;letter-spacing:.08em}
  .actions{display:flex;gap:10px}
  button.secondary{
    flex:1;padding:10px;border-radius:12px;border:1px solid rgba(0,0,0,.12);
    background:rgba(255,255,255,.6);font:inherit;color:var(--fg);
    cursor:pointer;
  }
  button.secondary:hover{background:rgba(255,255,255,.85)}
  button.danger{color:var(--err);border-color:rgba(193,39,45,.35)}
  button.secondary:disabled{opacity:.55;cursor:not-allowed}
</style>
</head><body>
  <form class="card" id="f" autocomplete="off">
    <h1>Receiver Setup</h1>
    <div class="meta">
      <div>MAC <code id="mac">loading…</code> · Public ID <code id="pid">—</code></div>
      <div>Firmware <code id="fw">—</code> · State <code id="st">—</code></div>
    </div>

    <label for="ssid">Wi-Fi SSID</label>
    <div class="row"><input id="ssid" name="ssid" required maxlength="32"></div>

    <label for="wpw">Wi-Fi password</label>
    <div class="row">
      <input id="wpw" name="wpw" type="password" required maxlength="63">
      <button type="button" class="toggle" data-for="wpw">show</button>
    </div>

    <label for="uri">MQTT broker URI</label>
    <div class="row">
      <input id="uri" name="uri" required class="mono"
             placeholder="mqtts://broker.example.com:8883"
             pattern="mqtts://.+"
             title="Must start with mqtts:// — plain MQTT is not supported">
    </div>

    <label for="mu">MQTT username</label>
    <div class="row"><input id="mu" name="mu" required></div>

    <label for="mp">MQTT password</label>
    <div class="row">
      <input id="mp" name="mp" type="password" required>
      <button type="button" class="toggle" data-for="mp">show</button>
    </div>

    <label for="tok">Enrollment token</label>
    <div class="row">
      <input id="tok" name="tok" required class="mono"
             placeholder="paste from admin dashboard"
             minlength="8" maxlength="128"
             autocapitalize="off" autocomplete="off" spellcheck="false">
    </div>

    <button type="submit" class="submit" id="go">Provision</button>
    <div class="status" id="s"></div>

    <div class="recovery" id="recov">
      <h2>Existing configuration</h2>
      <div class="meta" id="ssidLine">Current SSID: <code id="curSsid">—</code></div>
      <div class="actions">
        <button type="button" class="secondary" id="retry" hidden>Retry connection</button>
        <button type="button" class="secondary danger" id="reset">Factory reset</button>
      </div>
    </div>
  </form>
<script>
(function(){
  var $=function(id){return document.getElementById(id)};
  var set=function(msg,cls){var s=$('s');s.textContent=msg;s.className='status '+(cls||'')};
  fetch('/api/status').then(function(r){return r.json()}).then(function(j){
    $('mac').textContent=j.mac_label||'?';
    $('pid').textContent=j.public_id||'—';
    $('fw').textContent=j.firmware_version||'?';
    $('st').textContent=j.op_state||'UNPROVISIONED';
    if(j.has_config){ $('recov').classList.add('show'); }
    if(j.wifi_ssid){
      $('curSsid').textContent=j.wifi_ssid;
      $('retry').hidden=false;
    } else {
      $('ssidLine').style.display='none';
    }
  }).catch(function(){});
  document.querySelectorAll('.toggle').forEach(function(b){
    b.addEventListener('click',function(){
      var i=$(b.dataset.for), p=i.type==='password';
      i.type=p?'text':'password'; b.textContent=p?'hide':'show';
    });
  });
  // Enrollment-token format is backend-owned; no client-side transform.
  $('f').addEventListener('submit',function(ev){
    ev.preventDefault();
    var body={schema_version:1,
      wifi:{ssid:$('ssid').value,password:$('wpw').value},
      mqtt:{broker_uri:$('uri').value,username:$('mu').value,password:$('mp').value},
      enrollment:{token:$('tok').value.trim()}};
    $('go').disabled=true; set('Submitting…');
    fetch('/api/provision',{method:'POST',
      headers:{'Content-Type':'application/json'},
      body:JSON.stringify(body)})
      .then(function(r){return r.json().then(function(j){return {ok:r.ok,j:j}})})
      .then(function(x){
        if(x.ok){set('Saved. Device restarting…','ok')}
        else{$('go').disabled=false; set((x.j&&x.j.error)||'Rejected','err')}
      })
      .catch(function(){$('go').disabled=false; set('Network error','err')});
  });
  $('retry').addEventListener('click',function(){
    $('retry').disabled=$('reset').disabled=true; set('Reconnecting…','ok');
    fetch('/api/retry',{method:'POST'}).catch(function(){});
  });
  $('reset').addEventListener('click',function(){
    if(!confirm('Wipe all stored configuration? Identity keys are kept.'))return;
    $('retry').disabled=$('reset').disabled=true; set('Factory reset…','ok');
    fetch('/api/reset',{method:'POST'}).catch(function(){});
  });
})();
</script>
</body></html>
```

**Build-time gzip + embed.** Run `gzip -9 -k -n main/provision/index.html`
before build (Git-ignore the `.gz`; `-n` strips the mtime for reproducible
builds).

The repo's default build path is GNU Make via `project.mk`
(see `Makefile` and `main/component.mk`), so `main/component.mk` is the
authoritative place to register embedded assets. Alongside the existing
`COMPONENT_EMBED_TXTFILES := network/certs/mqtt_ca.pem` line, add:

```makefile
# main/component.mk
COMPONENT_EMBED_FILES += provision/index.html.gz
```

`COMPONENT_EMBED_TXTFILES` is for NUL-terminated PEM; `COMPONENT_EMBED_FILES`
is for raw binary blobs (no NUL appended) — correct for the gzipped HTML.
If the project later migrates to CMake, the equivalent is:

```cmake
# main/CMakeLists.txt
idf_component_register(
    SRCS "main.c" "provision/http_server.c" # ...
    EMBED_FILES "provision/index.html.gz"
)
```

**Serve it** from the `GET /` handler registered in §9.3:

```c
extern const uint8_t index_gz_start[] asm("_binary_index_html_gz_start");
extern const uint8_t index_gz_end  [] asm("_binary_index_html_gz_end");

static esp_err_t index_get(httpd_req_t *req) {
    httpd_resp_set_type(req, "text/html");
    httpd_resp_set_hdr(req, "Content-Encoding", "gzip");
    return httpd_resp_send(req, (const char *)index_gz_start,
                           index_gz_end - index_gz_start);
}
```

`GET /api/status` returns a JSON object with six fields: `mac_label`,
`public_id` (empty pre-activation), `firmware_version`, `op_state`,
`wifi_ssid`, `has_config`. No secrets. `POST /api/provision` accepts
the JSON body defined in §3.3.

---

## Part 2 — Spring Boot + Kotlin Backend

### 10. Services


| Service                      | Responsibility                                                                                                                                    |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| `EnrollmentTokenService`     | Redis-backed issue / hash / expire / consume (§12.1)                                                                                              |
| `DeviceRegistrationService`  | listen on `receiver/bootstrap/register`, validate token, create `PENDING` device, publish `pending` envelope                                      |
| `DeviceApprovalService`      | admin approve/reject actions; triggers challenge on approve, wipes session + publishes `rejected` on reject                                       |
| `ActivationChallengeService` | issue nonce challenges on approval, enforce expiry, mark one-time                                                                                 |
| `DeviceActivationService`    | verify signature, mint `public_id`, mark device active (no MQTT cred issuance — shared creds)                                                     |
| `RfCodeService`              | assign/rotate RF code, sign payload (§3.7), publish retained, consume acks                                                                        |
| `DeactivationService`        | sign (§3.7) and publish `suspend`/`resume`/`decommission`, consume acks                                                                           |
| `MqttBootstrapListener`      | MQTT subscriber for `receiver/bootstrap/register` and `receiver/bootstrap/+`; ignores any envelope whose `type` is backend-originated (self-echo) |
| `MqttOperationalListener`    | MQTT subscriber for `receiver/device/+/ack`                                                                                                       |


### 11. Data Model

```
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

- `id` is an internal UUID primary key (the "UUID per device at the
backend" in the project brief). `public_id` is the short,
MQTT-topic-safe identifier the device knows; it is minted on every
successful activation and rotates on reprovision.
- `receiver_type` ∈ `RECEIVER_433M` | `RECEIVER_2_4G`, taken from the
registration payload.
- `device_pk` on child tables is the internal UUID FK.
- Enrollment tokens live in Redis, not Postgres (§12.1).

Status enums:

- `devices.status` ∈ `PENDING` | `ACTIVE` | `SUSPENDED` |
`DECOMMISSIONED` | `REJECTED`. `PENDING` means "registered, awaiting
admin decision" — the review queue. `REJECTED` is a tombstone kept
for audit (optional; see §12 step 3).
- `activation_challenges.status` ∈ `PENDING_APPROVAL` | `ISSUED` |
`RESPONDED` | `EXPIRED` | `REJECTED`
- `device_rf_codes.last_ack_status` ∈ `applied` | `unchanged` |
`rejected` | `pending`
- `device_deactivation_commands.ack_status` ∈ `ok` | `ignored` |
`rejected` | `pending`

Rules:

- `public_id` is the single scoping identifier for operational topics.
A new `public_id` on re-activation guarantees orphaning of prior
retained state (§3.6) — no epoch column is needed.
- Every `device_rf_codes` and `device_deactivation_commands` row
records the `public_id` it was published under, for audit.
- RF code is encrypted at rest (KMS-backed column or secret-storage
ref).
- `code_version` is monotonic per `public_id` and is the basis for
dedup / stale-ack detection within that activation.

### 12. Backend Flow

1. Admin issues an enrollment token (§12.1); the plaintext is returned
  to the admin once and is never stored.
2. On `receiver/bootstrap/register`: validate `schema_version`, shape,
  and `receiver_type`; hash the token and consume it atomically from
   Redis (§12.1); reject invalid / expired / already-consumed. Look up
   any existing `devices` row by `public_key_der` (the stable identity
   from the `identity` NVS namespace) — if present, re-use the row and
   re-scope it to the new tenant/site; otherwise insert a `PENDING`
   row. Persist `receiver_type` and `registration_nonce`. Allocate
   `challenge_id` (UUID) and create an `activation_challenges` row
   with `status=PENDING_APPROVAL`. Publish the `pending` envelope on
   `receiver/bootstrap/{challenge_id}`. The device is now reachable on
   that topic; no nonce is issued yet. The backend subscribes to
   `receiver/bootstrap/+` to receive `response` envelopes and must
   drop any envelope whose `type` is backend-originated (self-echo).
3. **Admin review.** The device shows up in
  `GET /api/admin/devices?status=PENDING`. Admin inspects
   `public_key_der`, `receiver_type`, `tenant_id`, etc., then invokes
   one of:
  - `POST /api/admin/devices/{id}/approve` — `ActivationChallengeService`
  generates a fresh `nonce`, flips the challenge row to
  `status=ISSUED` with `expires_at`, and publishes the `challenge`
  envelope on the same `receiver/bootstrap/{challenge_id}` topic.
  - `POST /api/admin/devices/{id}/reject` — `DeviceApprovalService`
  publishes the `rejected` envelope on
  `receiver/bootstrap/{challenge_id}` (reason supplied by the admin
  or defaulted to `admin_rejected`), then **wipes the session**:
  deletes the `activation_challenges` row, deletes the `devices`
  row (or marks it `REJECTED` for audit, per tenant policy),
  deletes any Redis scratch keys scoped to this registration. The
  token was already consumed in step 2 and is not refunded — the
  installer needs a fresh one.
4. On the next `response` envelope matching this `challenge_id`: load
  the challenge, reject if expired / already responded / not in
   `ISSUED`; rebuild the canonical string byte-for-byte; verify the
   ECDSA signature against `devices.public_key_der`. On success: mint
   a fresh `public_id` (short, URL/topic-safe, collision-checked
   against the `devices` table), mark device `ACTIVE`, and publish the
   activation result on the same `receiver/bootstrap/{challenge_id}`
   topic with envelope `type: "result"` and the new `public_id`. No
   MQTT credential is issued — the device keeps its provisioning
   creds (§15).
5. Immediately after step 4, `RfCodeService` assigns a code, writes a
  new `device_rf_codes` row with `code_version = 1` and the new
   `public_id`, builds the canonical string (§3.3), signs it with the
   backend command-signing private key (§3.7), attaches `signature_b64`,
   and publishes the payload to
   `receiver/device/{public_id}/cmd/rf_code` retained + QoS 1. On
   `ack_for=rf_code`, update `last_ack_status` / `last_ack_at`.
6. **Rotation** reuses step 5: new row, incremented `code_version`,
  same topic, retained overwrites previous.
7. **Reprovision / decommission takeover.** Because the new activation
  mints a fresh `public_id`, old retained topics are orphaned
   automatically. As hygiene, the backend should publish zero-length
   `retain=true` payloads to
   `receiver/device/{old_public_id}/cmd/rf_code` and
   `receiver/device/{old_public_id}/cmd/deact` so the broker's
   retained store stays bounded. Not required for correctness.
8. **Deactivation.** Admin triggers `suspend` / `resume` /
  `decommission`. Service creates a `device_deactivation_commands`
   row with a fresh `command_id`, builds the canonical string (§3.3),
   signs it with the backend command-signing private key (§3.7),
   attaches `signature_b64`, and publishes retained + QoS 1. On
   `ack_for=deact`, update `acknowledged_at` / `ack_status`.
   `decommission` additionally flags the device row; re-activation is
   a separate (future) workflow.

### 12.1 Enrollment Tokens — Redis Handling

The backend stores enrollment tokens in Redis, not Postgres. This gives
TTL expiry and atomic single-use consumption via `GETDEL`.

**Format.** The token format is owned by the backend. The
`ENROLL-7GQ3-4TRP-9K1M` style in examples is illustrative; pick any
format that is:

- random (≥ 96 bits of entropy from a CSPRNG);
- single-use;
- URL-safe and human-copyable (the installer re-types it into the
provisioning page);
- short enough to fit on one line of the admin dashboard.

Recommended: 4×4 Crockford-base32 groups, or a 22-char base64url of 128
random bits. The device treats the token as opaque UTF-8.

**Storage.** Only the hash of the plaintext lands in Redis. SHA-256
over the canonical (trimmed, upper-cased) token is sufficient; HMAC
with a backend-side pepper is preferable if Redis is shared.

```
# On issue, with TTL of e.g. 1 hour:
SET enroll:<sha256_hex> '{"tenant_id":"...","site_id":"...",
                          "issued_by":"...","issued_at":"..."}'
    EX 3600 NX

# On consumption (atomic, single-use):
GETDEL enroll:<sha256_hex>
```

- `NX` prevents accidental overwrite of an in-flight token.
- `GETDEL` (Redis 6.2+) makes read+delete atomic; older Redis needs a
Lua equivalent.
- Missing key on consumption means "invalid, expired, or already used".
- Never store the plaintext token on `devices` or in Postgres.

**Revocation.** `DEL enroll:<sha256_hex>` on the Redis key — no
database transaction needed.

**Admin dashboard.** Return the plaintext exactly once and make that
explicit in the UI.

### 13. Signature Verification (Kotlin)

```kotlin
val pubKey: PublicKey = KeyFactory.getInstance("EC")
    .generatePublic(X509EncodedKeySpec(Base64.getDecoder().decode(publicKeyB64)))

val canonical = "activate-v1|$challengeId|$nonce|$issuedAt|$expiresAt"
    .toByteArray(Charsets.UTF_8)

val sig = Signature.getInstance("SHA256withECDSA").apply {
    initVerify(pubKey)
    update(canonical)
}
val ok = sig.verify(Base64.getDecoder().decode(signatureB64))
```

Firmware and backend **must** produce byte-identical canonical strings.

### 14. Admin Endpoints

Every new registration lands in `status=PENDING` and requires explicit
approve/reject. Path parameter `{id}` is the backend internal UUID
(from `devices.id`), not `public_id`, so admin URLs stay stable across
re-activations.

- `POST /api/admin/enrollment-tokens` (returns plaintext once, §12.1)
- `GET  /api/admin/devices` (list + filter by status / receiver_type;
`status=PENDING` is the review queue)
- `POST /api/admin/devices/{id}/approve` (issues challenge, §12 step 3)
- `POST /api/admin/devices/{id}/reject` (body: `{"reason": "..."}`;
publishes `rejected` envelope, wipes the bootstrap session, §12 step 3)
- `POST /api/admin/devices/{id}/reprovision` (publish retained-clear
on the current `public_id`'s cmd topics, mark device ready for a
fresh activation)
- `POST /api/admin/devices/{id}/rf-code` (rotation under current
`public_id`)
- `POST /api/admin/devices/{id}/deactivation` (action in body)

### 15. Broker Credentials & ACLs

**v1 constraint:** the managed broker does **not** support per-device
accounts. Every receiver uses the same MQTT credential provisioned in
SoftAP mode.

This gap is **acknowledged but inactionable in v1**: broker account /
ACL capability is outside the project's control, so the plan does not
attempt to solve it beyond TLS, unguessable `public_id`s, and signed
`cmd/`* payloads. It is an accepted deployment limitation, not an open
implementation task for this firmware/backend workstream.

**What protects the system in v1:**

1. `mqtts://` is mandatory (§3.2).
2. Operational topics are scoped by backend-minted `public_id`.
3. Every `cmd/`* payload is signed and verified (§3.7).

**What the shared credential still allows:**

- observe other devices' `cmd/`* traffic if the attacker can enumerate
topic paths;
- overwrite retained `cmd/*` slots with invalid payloads, causing
availability noise until the backend republishes a valid retained
command;
- publish forged acks or interfere with bootstrap topics.

So v1 closes command forgery, but not confidentiality or availability.
Store the shared credential only in NVS, never log it, and accept that
rotating it forces reprovisioning of every deployed device.

**v2 path:** switch the broker to per-device accounts / ACLs.
Wire format does not need to change because the lifecycle already scopes
all operational traffic by `public_id`:

- bootstrap cred (shared): pub+sub on `receiver/bootstrap/+`, pub on
`receiver/bootstrap/register`;
- per-device cred: pub `receiver/device/{public_id}/ack`, sub
`receiver/device/{public_id}/cmd/#`.

Retained-message backing storage should be encrypted in both v1 and v2.

---

## 16. End-to-End Example

1. Admin issues an enrollment token (format per §12.1, e.g.
  `ENROLL-7GQ3-4TRP-9K1M`); plaintext shown to admin once, hash
   stored in Redis with TTL.
2. Installer powers ESP-01; SoftAP `RECEIVER-SETUP-12AB34` appears
  (label derived from the STA MAC).
3. Installer opens the provisioning page, POSTs Wi-Fi + bootstrap
  MQTT creds + token; device reboots.
4. Device joins Wi-Fi, connects `mqtts://broker.example.com:8883`.
5. Device generates `pubkey/privkey` on first boot (persisted to
  `identity` namespace), picks a random `registration_nonce`,
   subscribes `receiver/bootstrap/+`, publishes registration
   `{receiver_type: "RECEIVER_433M", public_key_b64, enrollment_token,  registration_nonce}`.
6. Backend `GETDEL`s the token from Redis, verifies, creates a
  `PENDING` device row, allocates `challenge_id`, publishes
   `{type:"pending", registration_nonce, challenge_id}` on
   `receiver/bootstrap/{challenge_id}`.
7. Device filters on `registration_nonce`, narrows its subscription to
  exactly `receiver/bootstrap/{challenge_id}`, and blocks waiting for
   an admin decision.
8. Admin opens the dashboard, sees the device in the `PENDING` queue,
  inspects its public key and receiver_type, and clicks **Approve**.
   `ActivationChallengeService` generates a fresh `nonce` and publishes
   `{type:"challenge", nonce, expires_at, ...}` on the same
   `receiver/bootstrap/{challenge_id}` topic. (Had the admin clicked
   **Reject**, the backend would instead publish `{type:"rejected"}`
   and the device would return to SoftAP recovery in place, requiring a
   fresh token.)
9. Device signs the canonical string, publishes
  `{type:"response", signature_b64}` on
   `receiver/bootstrap/{challenge_id}` (and drops its own self-echo
   when the broker delivers it back).
10. Backend verifies the signature, mints `public_id="rcv-7Q4KZ"`,
  publishes `{type:"result", public_id, assigned_device_name}` on
    the same `receiver/bootstrap/{challenge_id}` topic. No MQTT
    credential is issued.
11. Device stores `public_id` + `device_name`, erases `enroll_token`,
  sets `op_state=PENDING_RF_CODE`, stops + destroys the bootstrap MQTT
  client, re-inits + starts MQTT with `client_id="rcv-7Q4KZ"`, enables
    `WIFI_PS_MIN_MODEM`, and subscribes
    `receiver/device/rcv-7Q4KZ/cmd/#` on the restarted session.
12. Backend builds canonical `rf-code-v1|rcv-7Q4KZ|1|0A1B2C3D|24|...`,
  signs it with the backend command-signing private key, and publishes
    `{rf_code_hex:"0A1B2C3D", bits:24, version:1, signature_b64}`
    retained on `receiver/device/rcv-7Q4KZ/cmd/rf_code`.
13. Device verifies `signature_b64` against the Kconfig-pinned public
  key, calls `rf_sup_start()`, stores code + `op_state=ACTIVE` in
    one commit, then publishes `{ack_for:"rf_code", version:1,   status:"applied"}` on `receiver/device/rcv-7Q4KZ/ack`.
14. Steady state: RF frames → match → toggle continuous vibrator pulsing;
    a later qualifying match toggles pulsing back off.
15. Admin rotates to `version=2` on the same topic; device overwrites
  NVS, re-acks.
16. Admin sends a signed `{action:"suspend", command_id, signature_b64}`
  on `receiver/device/rcv-7Q4KZ/cmd/deact`; device verifies the
    `deact-v1|...` signature, calls `rf_sup_suspend()`, persists
    `SUSPENDED`, acks `{ack_for:"deact", status:"ok"}`.
17. Admin sends a signed `{action:"resume", ...}`; device verifies,
  calls `rf_sup_resume()`, persists `ACTIVE`, acks.
18. Admin sends a signed `{action:"decommission", ...}`; device
  verifies, calls `rf_sup_delete()`, persists `DECOMMISSIONED`,
    acks. Wi-Fi / MQTT remain up for future recovery.

---

## 17. Implementation Order

**Firmware:**

1. `device_config` NVS module (new schema, §2)
2. Refactor `wifi.c` for STA + SoftAP runtime config; add PS call
3. HTTP provisioning server (`provision/http_server`)
4. Author and embed the provisioning UI (§9.8): gzip `index.html` at
  build time, add to `COMPONENT_EMBED_FILES` in `main/component.mk`
   (the repo's default Make build), serve from `GET /`
5. Refactor `mqtt.c` for runtime config + topic router + TLS via PEM blob
6. `security/device_identity` with mbedTLS EC P-256 (device keypair
  **and** `backend_pubkey_load()` for the pinned backend key from
   `CONFIG_RECEIVER_BACKEND_PUBKEY_B64`, §3.7)
7. Wire bootstrap registration → pending / rejected / challenge →
  response → result envelope handling
8. `trigger/rf_trigger` — connect `rf/` ↔ `vibrator/` via this module
9. `trigger/rf_supervisor` — start/suspend/resume/delete
10. Subscribe `receiver/device/{public_id}/cmd/#`; implement handlers
  for `cmd/rf_code` and `cmd/deact`; verify signature before
    applying either command (§3.7 / §6); publish acks on
    `receiver/device/{public_id}/ack` with `ack_for` discriminator
11. Factory-reset action (erase `device_cfg`)

**Backend:**

1. Redis-backed enrollment-token service + admin issue endpoint (§12.1)
2. `MqttBootstrapListener` + registration handling (validates
  `receiver_type`, echoes `registration_nonce`)
3. Challenge persistence (Postgres) + publisher with `type:"challenge"`
  envelope
4. Signature verification + activation + `public_id` minter +
  `type:"result"` envelope (no MQTT credential minting in v1, §15)
5. Backend command-signing key loader (Application Properties →
  `PrivateKey`, §3.7)
6. `RfCodeService` and `DeactivationService` sign every `cmd/`*
  payload with the backend command-signing key and publish retained;
   `MqttOperationalListener` (single
   `receiver/device/+/ack` subscription; dispatch on `ack_for`)
7. Invalid-signature telemetry / operator visibility for rejected
  `cmd/*` payloads
8. Admin dashboard views (receiver_type filter, RF code version,
  op state, re-provision/retained-clear action)

Not in scope for v1: broker-side per-device accounts / ACLs. The plan
assumes the current managed-broker limitation in §15 and treats it as
an accepted external constraint.
