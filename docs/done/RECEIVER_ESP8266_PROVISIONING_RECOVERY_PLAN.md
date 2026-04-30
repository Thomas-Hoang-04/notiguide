# ESP8266 Provisioning Recovery Implementation Plan

Standalone implementation plan for the `receiver-esp8266` module.

This document is intentionally self-contained. It assumes the implementer only has access to the `receiver-esp8266/` checkout inside its Dev Container and cannot read any other repo modules or design guides.

**Last updated:** 2026-04-30

---

## 1. Objective

Upgrade the ESP8266 receiver recovery flow so that the local provisioning page keeps its current lightweight UI and endpoint surface, but the recovery actions behave more like a real recovery loop:

- `POST /api/provision`
  - save new config
  - return success to the browser
  - reboot after a short delay

- `POST /api/retry`
  - leave SoftAP recovery mode
  - retry the stored STA + MQTT + activation path in the same boot
  - if retry fails, return to SoftAP recovery in the same boot

- `POST /api/reset`
  - erase runtime config only
  - keep device identity
  - reboot into unprovisioned state

The goal is behavioral improvement, not a UI redesign.

---

## 2. Non-Goals

Do not change these things unless container verification proves they are technically required:

- the existing local endpoint set
- the provisioning request schema
- the current single-card page structure
- the current glass/color/font direction of the page
- the current NVS split where `device_cfg` can be erased without erasing the identity keypair
- the command/activation MQTT protocol

---

## 3. Module Baseline

### 3.1 Relevant files

Core boot/runtime:

- `main/main.c`
- `main/config/device_config.c`
- `main/config/device_config.h`
- `main/network/wifi.c`
- `main/network/wifi.h`
- `main/network/mqtt.c`
- `main/network/mqtt.h`
- `main/security/device_identity.c`
- `main/security/device_identity.h`
- `main/trigger/rf_supervisor.c`
- `main/trigger/rf_trigger.c`

Provisioning / recovery:

- `main/provision/http_server.c`
- `main/provision/http_server.h`
- `main/provision/recovery.c`
- `main/provision/recovery.h`
- `main/provision/index.html`

Build / configuration / container:

- `main/CMakeLists.txt`
- `main/Kconfig.projbuild`
- `.devcontainer/devcontainer.json`
- `ESP8266_SDK_REFERENCE.md`

### 3.2 Current component registration

`main/CMakeLists.txt` already embeds:

- `provision/index.html.gz`
- `network/certs/mqtt_ca.pem`

and already links the required components for this work:

- `esp_http_server`
- `esp_event`
- `mqtt`
- `nvs_flash`
- `tcpip_adapter`
- `mbedtls`

No new framework dependency should be necessary for this plan.

### 3.3 Dev Container baseline

The checked-in Dev Container definition currently uses:

- file: `.devcontainer/devcontainer.json`
- image: `thomashoang04/thomas-esp:esp8266`
- post-create init: `/usr/local/bin/init-project.sh ${containerWorkspaceFolder}`

The local SDK snapshot says the container SDK lives at:

- `/opt/esp/ESP8266_RTOS_SDK/`

This plan must be validated against that actual container SDK before implementation is finalized.

### 3.4 Kconfig knobs already available

`main/Kconfig.projbuild` already provides the recovery-related configuration surface:

- `CONFIG_RECEIVER_WIFI_MAX_RETRY`
  - default `5`
  - current meaning: STA reconnect attempts before Wi-Fi reports failure

- `CONFIG_RECEIVER_AP_PASSWORD`
  - default `receiver-setup`
  - current meaning: WPA2-PSK password for provisioning SoftAP

- `CONFIG_RECEIVER_AP_CHANNEL`
  - default `1`
  - current meaning: provisioning SoftAP channel

- `CONFIG_RECEIVER_BACKEND_PUBKEY_B64`
  - backend command-signing public key
  - not directly part of recovery, but must remain untouched by this work

---

## 4. Current Behavior

### 4.1 Current boot flow in `main/main.c`

Today `app_main()` is a straight-line boot path:

1. initialize NVS
2. initialize `esp_netif`
3. create the default event loop
4. initialize vibrator / RF trigger
5. load config from NVS
6. restore RF code into RAM if stored
7. if not provisioned, call `provision_run_recovery_mode()`
8. initialize Wi-Fi and try STA with `wifi_start_sta(cfg.wifi_ssid, cfg.wifi_pwd)`
9. initialize device identity
10. initialize receiver MQTT helpers
11. start MQTT with `mqtt_receiver_start(&cfg)`
12. if activation is incomplete, run bootstrap activation
13. if activation succeeds, reload config and restart MQTT using `public_id`
14. subscribe to command topics
15. start or stop RF supervisor based on `op_state`
16. idle forever

Every failure path enters `provision_run_recovery_mode()` and does not return.

### 4.2 Current recovery behavior in `main/provision/recovery.c`

Today recovery mode does this:

1. `mqtt_stop()`
2. `wifi_stop()`
3. `wifi_start_softap(ssid, sizeof(ssid))`
4. `http_server_start()`
5. wait forever on `http_server_wait_for_action(portMAX_DELAY)`
6. for any non-`NONE` action:
   - wait 500 ms
   - `esp_restart()`

This means the recovery page is real, but its actions are shallow:

- `retry` is not an in-place retry
- `reset` is implemented, but still exits via reboot
- `provision` saves config, but recovery itself does not decide what to do based on which action happened

### 4.3 Current Wi-Fi / SoftAP behavior in `main/network/wifi.c`

Current station behavior:

- `wifi_start_sta(...)` waits until either:
  - `IP_EVENT_STA_GOT_IP`, or
  - retry exhaustion via `CONFIG_RECEIVER_WIFI_MAX_RETRY`
- it uses an event group with:
  - `WIFI_CONNECTED_BIT`
  - `WIFI_FAIL_BIT`
- on success it enables `WIFI_PS_MIN_MODEM`

Current SoftAP behavior:

- SoftAP SSID format:
  - `RECEIVER-SETUP-<last_3_mac_bytes_hex>`
- channel:
  - `CONFIG_RECEIVER_AP_CHANNEL`
- auth mode:
  - `WIFI_AUTH_WPA2_PSK`
- password:
  - `CONFIG_RECEIVER_AP_PASSWORD`
- max clients:
  - `1`

This existing behavior should be preserved.

### 4.4 Current HTTP provisioning contract in `main/provision/http_server.c`

#### `GET /api/status`

Current response fields:

- `mac_label`
- `public_id`
- `firmware_version`
- `op_state`
- `wifi_ssid`
- `has_config`

Notes:

- `op_state` is derived by `device_config_state_name(&cfg)`, so it can be:
  - `UNPROVISIONED`
  - `PENDING_ACTIVATION`
  - `RECOVERY_REQUIRED`
  - `PENDING_RF_CODE`
  - `ACTIVE`
  - `SUSPENDED`
  - `DECOMMISSIONED`
  - `UNKNOWN`

#### `POST /api/provision`

Current request body:

```json
{
  "schema_version": 1,
  "wifi": {
    "ssid": "string",
    "password": "string"
  },
  "mqtt": {
    "broker_uri": "mqtts://...",
    "username": "string",
    "password": "string"
  },
  "enrollment": {
    "token": "string"
  }
}
```

Current validation rules:

- body length must be `1..2048`
- `schema_version` must be `1`
- all three sections must exist
- all string fields are required and must be non-empty
- `broker_uri` must start with `mqtts://`

Current success behavior:

- save provisioning with `device_config_store_provisioning(&prov)`
- set HTTP event bit `BIT_PROVISIONED`
- return `{"status":"ok"}`

#### `POST /api/retry`

Current success behavior:

- set HTTP event bit `BIT_RETRY`
- return `{"status":"retrying"}`

#### `POST /api/reset`

Current success behavior:

- call `device_config_erase_runtime()`
- set HTTP event bit `BIT_RESET`
- return `{"status":"reset"}`

### 4.5 Current UI behavior in `main/provision/index.html`

The page is already lightweight and should stay that way.

Current visible sections:

- title and language toggle
- top metadata strip:
  - MAC
  - Public ID
  - Firmware
  - State
- provisioning form:
  - Wi-Fi SSID
  - Wi-Fi password
  - MQTT broker URI
  - MQTT username
  - MQTT password
  - enrollment token
- recovery section:
  - shown when `has_config == true`
  - shows current SSID if available
  - shows `Retry connection` only when `wifi_ssid` exists
  - always shows `Factory reset` inside the recovery panel

Current frontend behavior after action success:

- submit: shows `Device restarting...`
- retry: shows `Device restarting...`
- reset: shows `Device restarting...`

The UI already matches the intended simplicity. The gap is runtime behavior, not layout.

---

## 5. Current Persisted State Model

### 5.1 NVS namespace layout

`main/config/device_config.c` uses two namespaces:

- `identity`
- `device_cfg`

This plan must preserve that split.

`main/security/device_identity.h` makes the recovery implication explicit:

- the identity module owns the device EC keypair
- it also owns backend signature verification state

So factory reset in this plan must continue to mean:

- erase `device_cfg`
- keep `identity`

### 5.2 Runtime keys in `device_cfg`

Current runtime keys:

- `schema_ver`
- `wifi_ssid`
- `wifi_pwd`
- `mqtt_uri`
- `mqtt_user`
- `mqtt_pwd`
- `enroll_token`
- `public_id`
- `device_name`
- `rf_code`
- `rf_code_bits`
- `rf_code_ver`
- `op_state`
- `last_deact_id`

### 5.3 Current boot-state interpretation

The current state helpers in `main/config/device_config.c` behave like this:

- provisioned:
  - `has_wifi_ssid && has_mqtt_uri && has_mqtt_user && has_mqtt_pwd`
- pending activation:
  - provisioned and no `op_state` and `has_enroll_token`
- recovery required:
  - provisioned and no `op_state` and no `enroll_token`

`device_config_erase_runtime()` currently erases only `device_cfg`.

That is the correct reset boundary for this plan.

---

## 6. Root Problem To Solve

The page already offers recovery options, but the runtime does not implement a true recovery loop.

The specific defect is:

- `http_server_wait_for_action()` distinguishes `PROVISIONED`, `RETRY`, and `RESET`
- `provision_run_recovery_mode()` ignores that distinction and restarts on all of them

So the intended local semantics are lost. The implementation needs to preserve the current local API while making those action outcomes meaningful.

---

## 7. Target End State

### 7.1 Functional behavior

After implementation:

- unprovisioned boot still enters SoftAP provisioning
- Wi-Fi failure still enters SoftAP recovery
- MQTT startup failure still enters SoftAP recovery
- missing enrollment token during incomplete activation still enters SoftAP recovery
- bootstrap activation failure still enters SoftAP recovery

But once in recovery:

- `provision` saves new data and restarts
- `reset` erases runtime config and restarts
- `retry` leaves recovery mode and retries normal runtime in the same boot

If same-boot retry fails:

- the device must re-enter SoftAP recovery mode without wiping stored config
- the page must become available again
- the user must be able to retry again or overwrite credentials

### 7.2 UX constraints

Keep these unchanged unless a tiny additive status field is truly useful:

- single-card layout
- current glass styling
- current bilingual behavior
- current recovery-panel placement
- current provision payload shape

No dashboard expansion. No extra panels. No React migration.

---

## 8. Implementation Design

### 8.1 Refactor recovery into a result-bearing API

Current API:

```c
void provision_run_recovery_mode(void);
```

Target direction:

```c
typedef enum {
    PROVISION_RECOVERY_RESULT_RESTART = 0,
    PROVISION_RECOVERY_RESULT_RETRY_EXISTING,
} provision_recovery_result_t;

typedef enum {
    PROVISION_RECOVERY_REASON_UNPROVISIONED = 0,
    PROVISION_RECOVERY_REASON_WIFI_FAILED,
    PROVISION_RECOVERY_REASON_MQTT_FAILED,
    PROVISION_RECOVERY_REASON_MISSING_ENROLL_TOKEN,
    PROVISION_RECOVERY_REASON_BOOTSTRAP_FAILED,
} provision_recovery_reason_t;

provision_recovery_result_t provision_run_recovery_mode(provision_recovery_reason_t reason);
```

Intent:

- `RESTART` covers provision and reset
- `RETRY_EXISTING` covers retry

The exact enum names can differ, but the contract should look like this.

### 8.2 Keep HTTP server action ownership local

`main/provision/http_server.h` already defines:

```c
typedef enum {
    HTTP_SERVER_ACTION_NONE = 0,
    HTTP_SERVER_ACTION_PROVISIONED,
    HTTP_SERVER_ACTION_RETRY,
    HTTP_SERVER_ACTION_RESET,
} http_server_action_t;
```

That is already enough for the HTTP layer. There is no strong reason to redesign that enum unless the implementation becomes cleaner by sharing a slightly richer recovery context.

### 8.3 Rework `app_main()` into a retryable boot loop

The most important structural change belongs in `main/main.c`.

The boot path should be split into:

One-time setup:

- `nvs_flash_init()` with erase-and-retry fallback
- `esp_netif_init()`
- `esp_event_loop_create_default()`
- vibrator init
- RF trigger init
- `rf_recv_set_frame_callback(...)`
- `wifi_init()`
- `device_identity_init(&s_identity)`
- `mqtt_receiver_init(&s_identity, PROJECT_VER)`

Retryable runtime loop:

1. load `device_config_t`
2. restore RF code into RAM if present
3. decide whether recovery is needed before STA attempt
4. if recovery is entered:
   - if result is restart, restart after response flush
   - if result is retry, continue the loop
5. try `wifi_start_sta(cfg.wifi_ssid, cfg.wifi_pwd)`
6. try `mqtt_receiver_start(&cfg)`
7. if activation is incomplete:
   - require `enroll_token`
   - run `mqtt_receiver_bootstrap_activate(&cfg)`
   - reload config
   - run `mqtt_receiver_restart_with_public_id(&cfg)`
8. subscribe to command topics
9. restore final RF operational state
10. enter steady-state idle loop

This should be implemented so that recovery mode returns control to `main.c`, rather than duplicating the full runtime logic inside `recovery.c`.

### 8.4 Recommended helper split

To keep `main.c` maintainable, split straight-line logic into helpers similar to:

- `static esp_err_t init_platform_once(void);`
- `static esp_err_t load_runtime_config(device_config_t *cfg);`
- `static void restore_rf_snapshot(const device_config_t *cfg);`
- `static esp_err_t attempt_wifi_stage(const device_config_t *cfg);`
- `static esp_err_t attempt_mqtt_stage(device_config_t *cfg);`
- `static esp_err_t attempt_activation_stage(device_config_t *cfg);`
- `static void enter_operational_state(const device_config_t *cfg);`
- `static void restart_after_response_flush(void);`

Exact helper names do not matter. The split does.

### 8.5 Recovery loop behavior

Inside `provision_run_recovery_mode(...)`:

1. `mqtt_stop()`
2. `wifi_stop()`
3. `wifi_start_softap(...)`
4. `http_server_start()`
5. wait for `http_server_wait_for_action(...)`
6. branch by action

Required behavior by action:

- `HTTP_SERVER_ACTION_PROVISIONED`
  - stop HTTP server
  - optionally stop Wi-Fi if needed
  - return `PROVISION_RECOVERY_RESULT_RESTART`

- `HTTP_SERVER_ACTION_RESET`
  - stop HTTP server
  - optionally stop Wi-Fi if needed
  - return `PROVISION_RECOVERY_RESULT_RESTART`

- `HTTP_SERVER_ACTION_RETRY`
  - stop HTTP server
  - stop SoftAP
  - return `PROVISION_RECOVERY_RESULT_RETRY_EXISTING`

Important:

- Do not restart from inside `recovery.c` anymore.
- Let `main.c` decide when to restart.

### 8.6 Restart policy

Recommended helper:

```c
static void restart_after_response_flush(void)
{
    vTaskDelay(pdMS_TO_TICKS(250));
    esp_restart();
}
```

Use it only after `provision` or `reset`.

For `retry`, do not restart. Fall back to the retryable boot loop.

### 8.7 Minimal status additions

The current page can work without any API expansion, but adding a few fields will make implementation and diagnostics cleaner.

Recommended additive fields for `GET /api/status`:

- `boot_state`
  - derived from current config snapshot
- `recovery_reason`
  - populated only while in recovery
- `can_retry`
  - explicit boolean instead of deriving solely from `wifi_ssid`

These fields are optional but recommended.

Do not remove existing fields. Do not require the page to become multi-panel.

### 8.8 RF state handling after retry

Current code restores RF data into RAM before network startup:

- `rf_trigger_restore(cfg.rf_code, cfg.rf_code_bits, cfg.rf_code_ver);`

Final operational entry is based on:

- `cfg.op_state == RECEIVER_OP_STATE_ACTIVE` -> `rf_sup_start()`
- `cfg.op_state == RECEIVER_OP_STATE_DECOMMISSIONED` -> `rf_sup_delete()`

The retryable boot loop must preserve this behavior.

Do not start the RF supervisor during recovery mode.

### 8.9 MQTT lifecycle constraints

Existing useful functions already available:

- `mqtt_stop()`
- `mqtt_receiver_start(const device_config_t *cfg)`
- `mqtt_receiver_restart_with_public_id(const device_config_t *cfg)`
- `mqtt_receiver_subscribe_commands(const char *public_id)`
- `mqtt_receiver_bootstrap_activate(const device_config_t *cfg)`

Implementation guidance:

- keep `mqtt_receiver_init(&s_identity, PROJECT_VER)` as one-time init
- let retryable runtime stages call start/stop helpers as needed
- do not invent a second activation protocol

---

## 9. Exact File-by-File Work Plan

### 9.1 `main/provision/recovery.h`

Change from a void, non-returning helper to a result-bearing recovery contract.

Add:

- recovery result enum
- recovery reason enum
- new function signature

### 9.2 `main/provision/recovery.c`

Replace the unconditional reboot behavior with action-aware return behavior.

Tasks:

- accept a recovery reason
- start SoftAP + HTTP as today
- wait for HTTP action as today
- stop HTTP server before leaving recovery
- branch action outcome into either:
  - `RESTART`
  - `RETRY_EXISTING`

Optional:

- store the active recovery reason in a static variable if `status_get()` should expose it

### 9.3 `main/provision/http_server.c`

Keep the endpoint surface stable.

Required tasks:

- preserve current request/response compatibility
- optionally add `boot_state`, `recovery_reason`, `can_retry` to `GET /api/status`
- avoid changing `POST /api/provision` payload schema
- keep `reset` wiping only runtime config

Recommended small cleanup:

- add a response-flush-friendly path if testing shows `httpd_stop()` immediately after an action can cut off responses

### 9.4 `main/main.c`

This is the main implementation file.

Tasks:

- split one-time init from retryable runtime work
- make recovery re-entrant from multiple failure points
- replace direct calls to `provision_run_recovery_mode()` with:
  - reason-specific recovery entry
  - handling of returned recovery result
- ensure retry can bring the firmware back through:
  - Wi-Fi
  - MQTT
  - activation
  - final command subscription

### 9.5 `main/provision/index.html`

Keep changes minimal.

Possible tasks:

- if `GET /api/status` adds `can_retry`, use it for button gating
- if `GET /api/status` adds `recovery_reason`, surface it only if it helps and only in a compact way
- keep the page single-card
- keep bilingual structure mirrored

Do not add heavy status chrome.

### 9.6 `main/network/wifi.c` / `main/network/wifi.h`

Most likely the current APIs are already enough:

- `wifi_init()`
- `wifi_start_sta(...)`
- `wifi_start_softap(...)`
- `wifi_stop()`

Only change this layer if Dev Container verification shows mode switching needs one extra helper, for example:

- a dedicated `wifi_restart_sta(...)`
- a dedicated runtime-mode query

Prefer not to expand this layer unless the runtime loop genuinely benefits from it.

---

## 10. Implementation Sequence

Recommended order:

1. validate container SDK assumptions
2. refactor `recovery.h` / `recovery.c`
3. refactor `main.c` into one-time init + retryable runtime loop
4. add optional status fields in `http_server.c`
5. make minimal UI adjustments in `index.html`
6. rebuild `index.html.gz` if the page changes
7. build and verify in the container

This order keeps the hard behavioral work first and the optional UI refinements last.

---

## 11. Dev Container Validation Checklist

Run this inside the `receiver-esp8266` Dev Container.

### 11.1 SDK/API validation

Confirm the following against the actual SDK in the container:

- `esp_wifi_stop()` is sufficient before switching from SoftAP back to STA
- `esp_wifi_set_mode(WIFI_MODE_STA)` after prior AP mode works reliably in the same boot
- `httpd_stop()` can be called safely after action responses are sent
- repeated AP -> STA -> AP transitions do not require a full reboot

### 11.2 Functional validation

Validate these scenarios:

1. Fresh boot with empty `device_cfg`
   - device enters SoftAP
   - page loads
   - recovery panel hidden or minimal as expected

2. Valid provision submit
   - config saves
   - page shows restarting message
   - device reboots

3. Stored config, temporary Wi-Fi outage
   - device enters SoftAP recovery
   - user presses retry
   - device leaves recovery without reboot
   - if Wi-Fi comes back, runtime proceeds normally

4. Stored config, retry still fails
   - device returns to SoftAP recovery in the same boot
   - stored config remains available

5. Reset path
   - runtime config erased
   - identity preserved
   - device reboots into unprovisioned state

6. Pending activation without token
   - recovery reason is correct if surfaced
   - recovery page remains usable

### 11.3 UI validation

Confirm:

- `Retry connection` still only appears when retry is meaningful
- `Factory reset` remains available in recovery
- EN/VI strings still mirror structurally
- page remains lightweight and visually consistent

### 11.4 Build validation

Recommended container commands after implementation:

```bash
idf.py reconfigure
idf.py build
```

If the HTML file changes, make sure the compressed asset is refreshed as part of the normal firmware asset workflow before building.

---

## 12. Acceptance Criteria

The implementation is done when all items below are true:

- The local provisioning page still uses the same simple single-card UI.
- The endpoint set is still exactly:
  - `GET /`
  - `GET /api/status`
  - `POST /api/provision`
  - `POST /api/retry`
  - `POST /api/reset`
- `POST /api/retry` no longer requires a reboot before attempting the stored network path.
- A failed retry returns to SoftAP recovery in the same boot.
- `POST /api/reset` still preserves device identity and only clears runtime config.
- Activation/bootstrap behavior remains compatible with the existing MQTT flow.
- No new external dependency or protocol surface was introduced unnecessarily.
- The implementation is verified inside the real ESP8266 Dev Container and not only by static source review.

---

## 13. Short Reference Summary For The Implementer

If carrying only one mental model into the implementation, use this:

- The page is already fine.
- The endpoints are already fine.
- The stored state model is already fine.
- The problem is that recovery currently throws away action intent and reboots for everything.
- Fix that by making recovery return a meaningful result to `main.c`.
- Then make `main.c` capable of retrying the normal boot path without process restart.
