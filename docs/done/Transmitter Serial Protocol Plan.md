# Transmitter Hub — Serial Protocol Implementation Plan

> **Scope:** Implement the firmware-side USB serial protocol for the ESP32-C3 transmitter hub. This adds a JSON-lines protocol over the built-in USB Serial/JTAG controller, enabling the admin dashboard to provision, control, and diagnose the hub directly over USB.
>
> **Environment:** This plan is executed inside a Dev Container with ESP-IDF v6.0. It is self-contained — all context needed for implementation is included below.
>
> **Prerequisite:** The OLED & Button plan has been implemented. The `TRANSMITTER_EVENTS` event bus, `display/` module, and `controls/` module already exist.
>
> **Related web plan:** `docs/planned/Web Serial Plan.md` covers dashboard-side Web Serial UX. This firmware plan is the transmitter-domain source of truth for the ESP32-C3 serial protocol, current transmitter module references, and ESP-IDF integration details. It restates the external contracts needed by the firmware so it can be implemented inside a transmitter-only Dev Container.

---

## Existing Firmware Context

This section reflects the transmitter firmware tree that should exist before this plan is implemented. Paths are relative to the transmitter project root inside the Dev Container.

### Module Layout

```
transmitter/main/
├── main.c                          # app_main(), boot sequence, recovery loop
├── config/
│   ├── device_config.h             # device_config_t, op state enum, NVS load/save
│   └── device_config.c
├── dispatch/
│   ├── dispatch.h                  # dispatch_handle_transmit_json(), dispatch_handle_deact_json()
│   ├── dispatch.c                  # Signed command parsing, hex decode, ring buffer, ack publish
│   ├── radio_supervisor.h          # radio_band_t, radio_tx_send(), radio_supervisor_init()
│   └── radio_supervisor.c
├── network/
│   ├── mqtt.h                      # mqtt_start/stop(), mqtt_is_connected(), mqtt_publish()
│   ├── mqtt.c
│   ├── wifi.h                      # wifi_start_sta/softap(), wifi_is_sta_connected()
│   ├── wifi.c
│   ├── heartbeat.h / .c
│   └── certs/
├── nrf24/                          # nRF24L01+ SPI driver
├── rf/                             # 433 MHz + nRF24 transmit logic
├── provision/
│   ├── http_server.h / .c          # SoftAP provisioning HTTP server
│   └── index.html.gz
├── security/
│   ├── device_identity.h / .c      # EC P-256 key management + signature verification
├── events/
│   ├── transmitter_events.h        # TRANSMITTER_EVENTS base, event IDs, dispatch_summary_t
│   └── transmitter_events.c
├── display/                        # OLED (conditional on CONFIG_TRANSMITTER_OLED_ENABLED)
│   ├── display.h / .c
│   ├── display_screens.h / .c
│   └── display_font.c
├── controls/
│   ├── controls.h / .c             # Button + LED + event dispatch
└── utils/
    ├── time_utils.h / .c
```

### Boot Sequence (`app_main()`)

```c
void app_main(void)
{
    ESP_ERROR_CHECK(device_config_nvs_init_or_recover());
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    ESP_ERROR_CHECK(device_config_load());
    ESP_ERROR_CHECK(device_config_consume_recovery_mode_request(&force_recovery));

    ESP_ERROR_CHECK(display_init());    // conditional on CONFIG_TRANSMITTER_OLED_ENABLED
    ESP_ERROR_CHECK(controls_init());
    // ← serial_protocol_init() will be inserted HERE

    ESP_ERROR_CHECK(wait_for_station_or_recover(force_recovery));
    ESP_ERROR_CHECK(device_identity_ensure());
    ESP_ERROR_CHECK(radio_supervisor_init());
    ESP_ERROR_CHECK(mqtt_start());

    // Bootstrap activation loop...
    // TX_EVENT_ACTIVATED posted on success
    // Infinite idle loop
}
```

### Key Function Signatures

**Device config (`config/device_config.h`):**

```c
const device_config_t *device_config_get(void);
esp_err_t device_config_save_provisioning(const device_provisioning_request_t *request);
esp_err_t device_config_factory_reset(void);
esp_err_t device_config_commit_lifecycle(const char *command_id, bool update_state,
                                         device_operational_state_t new_state);
bool device_config_is_provisioned(const device_config_t *cfg);
bool device_config_is_activated(const device_config_t *cfg);
bool device_config_requires_recovery(const device_config_t *cfg);
const char *device_config_op_state_to_string(device_operational_state_t state);
```

**Radio (`dispatch/radio_supervisor.h`):**

```c
typedef enum { RADIO_BAND_433M = 0, RADIO_BAND_2_4G = 1 } radio_band_t;
esp_err_t radio_tx_send(radio_band_t band, const uint8_t *payload,
                        size_t byte_len, uint8_t bits, bool proto_any);
```

**Network (`network/wifi.h`, `network/mqtt.h`):**

```c
esp_err_t wifi_start_sta(const device_config_t *cfg);
esp_err_t wifi_wait_for_sta_ready(TickType_t timeout);
esp_err_t wifi_stop(void);
bool wifi_is_sta_connected(void);
wifi_runtime_mode_t wifi_get_runtime_mode(void);  // NONE / STA / SOFTAP
bool mqtt_is_connected(void);
```

**Heartbeat (`network/heartbeat.h`):**

```c
esp_err_t heartbeat_start(void);
esp_err_t heartbeat_stop(void);
```

**Events (`events/transmitter_events.h`):**

```c
ESP_EVENT_DECLARE_BASE(TRANSMITTER_EVENTS);

typedef enum {
    TX_EVENT_DISPATCH_OK = 0,
    TX_EVENT_DISPATCH_REJECTED,
    TX_EVENT_MQTT_CONNECTED,
    TX_EVENT_MQTT_DISCONNECTED,
    TX_EVENT_WIFI_CONNECTED,
    TX_EVENT_WIFI_DISCONNECTED,
    TX_EVENT_PROVISIONING_STARTED,
    TX_EVENT_LIFECYCLE_CHANGED,
    TX_EVENT_BOOTSTRAP_STARTED,
    TX_EVENT_ACTIVATED,
} transmitter_event_id_t;

typedef struct {
    char receiver_public_id[48];
    char band[8];
    int  rf_code_bits;
    bool proto_any;
    char status[16];      // "applied", "rejected", etc.
    char reason[32];
    int64_t applied_at_ms;
} dispatch_summary_t;
```

**Data structures (`config/device_config.h`):**

```c
typedef struct {
    char wifi_ssid[33];   char wifi_pwd[65];
    char mqtt_uri[192];   char mqtt_user[96];   char mqtt_pwd[128];
    char enroll_token[192];
} device_provisioning_request_t;

typedef struct {
    uint8_t schema_version;
    char wifi_ssid[33]; char wifi_pwd[65];
    char mqtt_uri[192]; char mqtt_user[96]; char mqtt_pwd[128];
    char enroll_token[192];
    char public_id[32]; char device_name[96]; char last_deact_id[64];
    device_operational_state_t op_state;
    bool has_schema_version, has_enroll_token, has_public_id;
    bool has_device_name, has_last_deact_id, has_op_state;
} device_config_t;
```

### Dispatch Path (from `dispatch.c`) — Reference for Serial `transmit`

The MQTT-signed transmit flow in `dispatch_handle_transmit_json()`:

1. Parse JSON (validate `schema_version == 1`) → `transmit_cmd_t { dispatch_id, receiver_public_id, band, rf_code_hex, rf_code_bits, proto_any, issued_at, signature_b64 }`
2. Validate RF code width: 433M → 1-32 bits; 2_4G → exactly 40 bits. Hex length must match `ceil(bits/8)*2`
3. Verify ECDSA signature ← **serial protocol skips this step**
4. Check dispatch ring (8-entry dedup buffer) for duplicate `dispatch_id` ← **serial protocol skips this step** (USB commands are local, not replayed by network; `dispatch_id` is optional for client-side correlation only)
5. Check `op_state == OP_STATE_ACTIVE`
6. `hex_decode()` → binary payload
7. `band = strcmp(cmd.band, "433M") == 0 ? RADIO_BAND_433M : RADIO_BAND_2_4G`
8. `radio_tx_send(band, plaintext, byte_len, bits, proto_any)` → ESP_OK / ESP_ERR_NOT_SUPPORTED / other
9. Publish MQTT ack ← **serial protocol skips this step**
10. Post `TX_EVENT_DISPATCH_OK` / `TX_EVENT_DISPATCH_REJECTED`

The serial `transmit` handler replicates steps 1-2, 5-8, 10 — skipping signature verification (step 3), dispatch ring dedup (step 4), and MQTT ack publication (step 9). Responses go back over USB serial instead.

### External Dashboard/Backend Contract Snapshot

The firmware does not create backend device records and has no `store_id` field in NVS. USB provisioning receives an enrollment token that was already issued by the dashboard/backend and stores it exactly like SoftAP provisioning. After restart, the hub still performs the normal MQTT bootstrap flow:

1. Firmware publishes its transmitter bootstrap registration over MQTT using the stored enrollment token.
2. Backend consumes that token, binds the pending transmitter hub to the token's store when the token is store-scoped, and publishes the bootstrap challenge.
3. Firmware signs the challenge and receives activation data.
4. Dashboard approval/review remains a backend/web concern, not a serial firmware concern.

For the current web/backend shape, the dashboard should issue the token through the existing enrollment-token API and inject only the returned token into the serial `provision` command. The serial `identify` command returns firmware-local identity only (`public_id`, `device_name`, `firmware_version`, `mac`, `op_state`); it does not return `store_id`.

---

## Serial Protocol Design

### Framing: JSON Lines

Each message is a single JSON object terminated by `\n`. The browser discriminates protocol messages from console log output by first character: `{` = JSON protocol, anything else = ESP-IDF log output.

### Message Envelope

**Request** (browser → hub):

```json
{"id":"<uuid>","type":"<type>","payload":{...}}
```

| Field     | Type   | Description                                |
| --------- | ------ | ------------------------------------------ |
| `id`      | string | UUID v4 for request/response correlation   |
| `type`    | string | Command type (see command table)           |
| `payload` | object | Command-specific data (optional for some)  |

**Response** (hub → browser):

```json
{"id":"<echoed-uuid>","type":"response","ok":true,"payload":{...}}
```

| Field     | Type    | Description                |
| --------- | ------- | -------------------------- |
| `id`      | string  | Echoed request ID          |
| `type`    | string  | Always `"response"`        |
| `ok`      | boolean | Success/failure            |
| `payload` | object  | Result data (on success)   |
| `error`   | string  | Error message (on failure) |

**Event** (hub → browser, unsolicited):

```json
{"id":null,"type":"event.wifi_connected","payload":{}}
```

### Command Table

#### Provisioning Commands

| Type                  | Payload                                                              | Response Payload                                  |
| --------------------- | -------------------------------------------------------------------- | ------------------------------------------------- |
| `status`              | (none)                                                               | Device status snapshot (see below)                |
| `provision`           | `{wifi_ssid, wifi_pwd, mqtt_uri, mqtt_user, mqtt_pwd, enroll_token}` | `{restarting: true}`                              |
| `provision.test_wifi` | `{wifi_ssid, wifi_pwd}`                                              | `{connected: bool, ip: string, rssi: int}`        |
| `update_mqtt`         | `{mqtt_uri, mqtt_user, mqtt_pwd}`                                    | `{restarting: true}`                              |
| `factory_reset`       | (none)                                                               | `{restarting: true}`                              |

#### Control Commands (post-activation)

| Type        | Payload                                                              | Response Payload                              |
| ----------- | -------------------------------------------------------------------- | --------------------------------------------- |
| `transmit`  | `{dispatch_id?, receiver_public_id?, band, rf_code_hex, rf_code_bits, proto_any}` | `{status, reason?, applied_at_ms?}` |
| `lifecycle` | `{action: "suspend" \| "resume" \| "decommission"}`                  | `{op_state: "<new state>"}`                   |

#### Diagnostic Commands

| Type       | Payload | Response Payload                                                      |
| ---------- | ------- | --------------------------------------------------------------------- |
| `ping`     | (none)  | `{uptime_ms: number}`                                                 |
| `identify` | (none)  | `{public_id, device_name, firmware_version, mac, op_state}` |

### Status Response Schema

```json
{
  "schema_version": 1,
  "provisioned": true,
  "activated": true,
  "recovery_required": false,
  "wifi_connected": true,
  "mqtt_connected": true,
  "op_state": "ACTIVE",
  "public_id": "HUB-7F3A",
  "device_name": "Store Front Hub",
  "wifi_ssid": "StoreLab-5G",
  "wifi_rssi": -52,
  "ip": "192.168.1.42",
  "uptime_ms": 3612000,
  "free_heap": 142560,
  "firmware_version": "1.0.0",
  "mac": "AA:BB:CC:DD:EE:FF"
}
```

### Trust Model: USB Commands Are Not Signed

USB is a direct physical connection. If someone has physical USB access, they already have full control (can reflash firmware). Serial `transmit` and `lifecycle` commands skip ECDSA signature verification. The `dispatch_id` field is optional — it is accepted for client-side correlation but not fed through the dispatch ring for dedup (USB commands are local, not replayed over an untrusted network).

---

## New Module: `serial/`

### File Structure

```
transmitter/main/
├── serial/
│   ├── serial_protocol.h    # Public API
│   └── serial_protocol.c    # USB driver, JSON parsing, command dispatch
```

### Public API (`serial_protocol.h`)

```c
#ifndef TRANSMITTER_SERIAL_PROTOCOL_H
#define TRANSMITTER_SERIAL_PROTOCOL_H

#include "esp_err.h"
#include <stdbool.h>

#ifdef __cplusplus
extern "C" {
#endif

/**
 * @brief Initialize the USB serial protocol driver and start the read-loop task.
 *
 * Installs the USB Serial/JTAG driver with TX/RX ring buffers, binds VFS
 * so log output routes through the driver, and spawns the protocol task.
 *
 * @return ESP_OK on success.
 */
esp_err_t serial_protocol_init(void);

/**
 * @brief Check whether an active serial protocol session exists.
 *
 * Combines the hardware-level USB host detection
 * (`usb_serial_jtag_is_connected()`) with an application-level session
 * flag that tracks valid JSON command activity. Returns true only when
 * the USB cable is connected AND a protocol session is active.
 *
 * @return true when a host is connected and a protocol session is active.
 */
bool serial_protocol_is_host_connected(void);

#ifdef __cplusplus
}
#endif

#endif /* TRANSMITTER_SERIAL_PROTOCOL_H */
```

---

## ESP-IDF API Reference

All APIs below are verified against the ESP-IDF v6.0 release branch headers (`esp_driver_usb_serial_jtag` component).

### USB Serial/JTAG Driver

**Header:** `driver/usb_serial_jtag.h`, `driver/usb_serial_jtag_vfs.h`

```c
// Driver config struct — default TX/RX buffer sizes are 256 bytes each
typedef struct {
    uint32_t tx_buffer_size;
    uint32_t rx_buffer_size;
} usb_serial_jtag_driver_config_t;

#define USB_SERIAL_JTAG_DRIVER_CONFIG_DEFAULT() (usb_serial_jtag_driver_config_t) { \
    .tx_buffer_size = 256, \
    .rx_buffer_size = 256, \
}

// Install driver with RX/TX ring buffers
esp_err_t usb_serial_jtag_driver_install(usb_serial_jtag_driver_config_t *usb_serial_jtag_config);

// Route stdout/stderr (ESP_LOGI, printf) through the driver's TX ring buffer
// instead of default non-blocking polling path. Only affects TX/write side.
// Does NOT spawn a background stdin reader — RX ring buffer stays exclusively
// available to our protocol task's read_bytes calls.
void usb_serial_jtag_vfs_use_driver(void);

// Blocking read from RX ring buffer. Returns bytes read, 0 on timeout.
int usb_serial_jtag_read_bytes(void *buf, uint32_t length, uint32_t ticks_to_wait);

// Blocking write to TX ring buffer. Returns bytes written.
int usb_serial_jtag_write_bytes(const void *src, size_t size, uint32_t ticks_to_wait);

// Block until the TX ring buffer is drained. Use before esp_restart().
esp_err_t usb_serial_jtag_wait_tx_done(uint32_t ticks_to_wait);

// Check whether a USB host is actively polling the serial port (SOF detection).
bool usb_serial_jtag_is_connected(void);

// Check whether the driver has been installed.
bool usb_serial_jtag_is_driver_installed(void);
```

### Host Connection Detection

`usb_serial_jtag_is_connected()` is a public v6.0 API that uses the USB Serial/JTAG controller's SOF detection. It returns `true` when a USB host is actively polling the port and `false` when the cable is unplugged or the host closes the port.

`serial_protocol_is_host_connected()` combines the hardware-level check with an application-level session flag for protocol-aware awareness:

- The hardware check (`usb_serial_jtag_is_connected()`) detects USB electrical connection.
- The session flag tracks active Web Serial protocol usage — a USB cable can be plugged in without a browser having the serial port open, and monitoring tools (e.g. `picocom`) do not send protocol commands.
- The read loop sets `s_session_active = true` and records `s_last_rx_ms` when it receives a valid JSON command.
- The session is treated as inactive after 30 seconds without RX.
- The browser sends `status` immediately after opening the port, so a valid Web Serial session becomes active before event streaming is needed.

```c
bool serial_protocol_is_host_connected(void)
{
    if (!usb_serial_jtag_is_connected()) {
        s_session_active = false;
        return false;
    }
    if (!s_session_active) {
        return false;
    }
    const int64_t now_ms = esp_timer_get_time() / 1000;
    if ((now_ms - s_last_rx_ms) > 30000) {
        s_session_active = false;
    }
    return s_session_active;
}
```

Do not use `usb_serial_jtag_ll_txfifo_writable()` as a host-connected substitute. It is a low-level FIFO writability check, not a session signal.

### cJSON (already used by `dispatch.c`)

**Header:** `cJSON.h` (from `espressif/cjson` managed component)

```c
cJSON *cJSON_Parse(const char *value);
cJSON *cJSON_GetObjectItem(const cJSON *object, const char *string);
cJSON_bool cJSON_IsString(const cJSON *item);
cJSON_bool cJSON_IsNumber(const cJSON *item);
cJSON_bool cJSON_IsBool(const cJSON *item);
char *cJSON_GetStringValue(const cJSON *item);
cJSON *cJSON_CreateObject(void);
cJSON *cJSON_AddStringToObject(cJSON *object, const char *name, const char *string);
cJSON *cJSON_AddBoolToObject(cJSON *object, const char *name, cJSON_bool boolean);
cJSON *cJSON_AddNumberToObject(cJSON *object, const char *name, double number);
void cJSON_AddItemToObject(cJSON *object, const char *name, cJSON *item);
char *cJSON_PrintUnformatted(const cJSON *item);  // returns malloc'd string
void cJSON_Delete(cJSON *item);
```

### sdkconfig Options

```
# Already set (console shares USB pipe with protocol):
CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y

# Enable when power management is enabled (prevents light-sleep from killing
# USB when host is connected):
CONFIG_USJ_NO_AUTO_LS_ON_CONNECTION=y
```

`CONFIG_USJ_NO_AUTO_LS_ON_CONNECTION` prevents the chip from automatically entering light sleep when the USB Serial/JTAG port is connected. It depends on `CONFIG_PM_ENABLE`; the current transmitter `sdkconfig` has power management disabled, so the option is not selectable and is not needed unless power management is enabled later.

---

## Implementation Phases

### Phase 1 — Serial Driver & Provisioning Commands

**Scope:** USB serial driver, JSON protocol, diagnostic + provisioning commands.

#### Step 1.1 — sdkconfig & Kconfig Changes

If `CONFIG_PM_ENABLE=y`, add to `transmitter/sdkconfig`:

```
CONFIG_USJ_NO_AUTO_LS_ON_CONNECTION=y
```

`CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y` is already set. In the current transmitter `sdkconfig`, `CONFIG_PM_ENABLE` is not set, so `CONFIG_USJ_NO_AUTO_LS_ON_CONNECTION` is intentionally absent.

Add to `transmitter/main/Kconfig.projbuild` (used by `identify` and `status` commands):

```
config TRANSMITTER_FIRMWARE_VERSION
    string "Firmware version string"
    default "1.0.0"
```

#### Step 1.2 — Create `serial/serial_protocol.h`

Public API as specified above: `serial_protocol_init()` and `serial_protocol_is_host_connected()`.

#### Step 1.3 — Create `serial/serial_protocol.c`

##### A. Driver Initialization

```c
#include "driver/usb_serial_jtag.h"
#include "driver/usb_serial_jtag_vfs.h"
#include "cJSON.h"
#include "config/device_config.h"
#include "dispatch/radio_supervisor.h"
#include "events/transmitter_events.h"
#include "esp_event.h"
#include "esp_log.h"
#include "esp_mac.h"
#include "esp_netif.h"
#include "esp_system.h"
#include "esp_timer.h"
#include "esp_wifi.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "network/heartbeat.h"
#include "network/mqtt.h"
#include "network/wifi.h"
#include "serial/serial_protocol.h"
#include "utils/time_utils.h"
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

static const char *TAG = "serial";
static bool s_session_active;
static int64_t s_last_rx_ms;

static void serial_mark_session_active(void)
{
    s_session_active = true;
    s_last_rx_ms = esp_timer_get_time() / 1000;
}

bool serial_protocol_is_host_connected(void)
{
    if (!usb_serial_jtag_is_connected()) {
        s_session_active = false;
        return false;
    }
    if (!s_session_active) {
        return false;
    }
    const int64_t now_ms = esp_timer_get_time() / 1000;
    if ((now_ms - s_last_rx_ms) > 30000) {
        s_session_active = false;
    }
    return s_session_active;
}

esp_err_t serial_protocol_init(void)
{
    usb_serial_jtag_driver_config_t cfg = USB_SERIAL_JTAG_DRIVER_CONFIG_DEFAULT();
    cfg.tx_buffer_size = 4096;
    cfg.rx_buffer_size = 4096;
    esp_err_t err = usb_serial_jtag_driver_install(&cfg);
    if (err != ESP_OK) {
        return err;
    }

    usb_serial_jtag_vfs_use_driver();

    err = esp_event_handler_register(TRANSMITTER_EVENTS, ESP_EVENT_ANY_ID,
                                     serial_event_handler, NULL);
    if (err != ESP_OK) {
        return err;
    }

    BaseType_t task_ok = xTaskCreate(serial_protocol_task, "serial_proto", 4096, NULL, 5, NULL);
    if (task_ok != pdPASS) {
        return ESP_ERR_NO_MEM;
    }

    ESP_LOGI(TAG, "Serial protocol initialized");
    return ESP_OK;
}
```

**Task stack:** 4096 bytes on ESP32-C3 (RISC-V port uses byte-sized stack units, not words). This accommodates the 1 KB line buffer, cJSON parse/print allocations, and the deepest call chain (provision.test_wifi → wifi_start_sta → event waits). If stack highwater mark is tight, increase to 6144. Verify with `uxTaskGetStackHighWaterMark()` after integration.

**Task priority:** 5 (same as MQTT control task). The serial protocol task blocks on `usb_serial_jtag_read_bytes()` when idle, so it consumes no CPU time unless data arrives.

##### B. Read Loop Task

```c
#define LINE_BUF_SIZE 1024

static void serial_protocol_task(void *arg)
{
    char line_buf[LINE_BUF_SIZE];
    int line_pos = 0;
    bool line_overflow = false;

    for (;;) {
        uint8_t byte;
        int len = usb_serial_jtag_read_bytes(&byte, 1, pdMS_TO_TICKS(100));
        if (len <= 0) {
            continue;
        }

        if (byte == '\n') {
            if (line_overflow) {
                ESP_LOGW(TAG, "Discarding oversized line (%d+ bytes)", LINE_BUF_SIZE);
                line_overflow = false;
            } else if (line_pos > 0) {
                line_buf[line_pos] = '\0';
                serial_handle_line(line_buf);
            }
            line_pos = 0;
        } else if (byte != '\r') {
            if (line_pos < LINE_BUF_SIZE - 1) {
                line_buf[line_pos++] = byte;
            } else {
                line_overflow = true;
            }
        }
    }
}
```

**`\r` stripping:** Browsers may send `\r\n` depending on platform. Strip `\r` to normalize.

**RX exclusivity:** No `esp_console` REPL runs alongside this task. `usb_serial_jtag_vfs_use_driver()` only affects the VFS write path for log output — it does not spawn a background stdin reader. The protocol task is the sole consumer of the RX ring buffer.

##### C. Command Dispatch

```c
static void serial_handle_line(const char *line)
{
    cJSON *root = cJSON_Parse(line);
    if (root == NULL) {
        return;  // Not valid JSON — ignore (could be echo or garbage)
    }

    const cJSON *id_item = cJSON_GetObjectItem(root, "id");
    const cJSON *type_item = cJSON_GetObjectItem(root, "type");

    if (!cJSON_IsString(id_item) || !cJSON_IsString(type_item)) {
        cJSON_Delete(root);
        return;
    }

    const char *id = cJSON_GetStringValue(id_item);
    const char *type = cJSON_GetStringValue(type_item);
    const cJSON *payload = cJSON_GetObjectItem(root, "payload");

    serial_mark_session_active();

    if (strcmp(type, "ping") == 0) {
        handle_ping(id);
    } else if (strcmp(type, "identify") == 0) {
        handle_identify(id);
    } else if (strcmp(type, "status") == 0) {
        handle_status(id);
    } else if (strcmp(type, "provision") == 0) {
        handle_provision(id, payload);
    } else if (strcmp(type, "provision.test_wifi") == 0) {
        handle_test_wifi(id, payload);
    } else if (strcmp(type, "update_mqtt") == 0) {
        handle_update_mqtt(id, payload);
    } else if (strcmp(type, "factory_reset") == 0) {
        handle_factory_reset(id);
    } else if (strcmp(type, "transmit") == 0) {
        handle_transmit(id, payload);
    } else if (strcmp(type, "lifecycle") == 0) {
        handle_lifecycle(id, payload);
    } else {
        serial_send_error(id, "unknown_command");
    }

    cJSON_Delete(root);
}
```

##### D. Response Writing

```c
static void serial_send_response(const char *id, bool ok, cJSON *payload, const char *error)
{
    cJSON *root = cJSON_CreateObject();
    cJSON_AddStringToObject(root, "id", id);
    cJSON_AddStringToObject(root, "type", "response");
    cJSON_AddBoolToObject(root, "ok", ok);
    if (ok && payload != NULL) {
        cJSON_AddItemToObject(root, "payload", payload);
    }
    if (!ok && error != NULL) {
        cJSON_AddStringToObject(root, "error", error);
    }

    char *json = cJSON_PrintUnformatted(root);
    if (json != NULL) {
        size_t json_len = strlen(json);
        char *line = malloc(json_len + 2);
        if (line != NULL) {
            memcpy(line, json, json_len);
            line[json_len] = '\n';
            usb_serial_jtag_write_bytes(line, json_len + 1, pdMS_TO_TICKS(200));
            free(line);
        } else {
            ESP_LOGE(TAG, "malloc failed for serial response (%zu bytes)", json_len + 2);
        }
        free(json);
    }

    // cJSON_Delete(root) frees both root and any attached payload
    // (transferred via cJSON_AddItemToObject).
    cJSON_Delete(root);
}

static void serial_send_error(const char *id, const char *error)
{
    serial_send_response(id, false, NULL, error);
}
```

**Pre-restart drain:** `usb_serial_jtag_write_bytes()` writes to the TX ring buffer, and the hardware drains it to USB autonomously via interrupts. For commands that call `esp_restart()` after responding, `usb_serial_jtag_wait_tx_done(pdMS_TO_TICKS(200))` blocks until the TX buffer is fully drained or the timeout elapses, ensuring the response reaches the host before the device resets.

##### E. Event Forwarding

```c
static void serial_send_event(const char *event_type, cJSON *payload)
{
    cJSON *root = cJSON_CreateObject();
    cJSON_AddNullToObject(root, "id");
    cJSON_AddStringToObject(root, "type", event_type);
    if (payload != NULL) {
        cJSON_AddItemToObject(root, "payload", payload);
    } else {
        cJSON_AddItemToObject(root, "payload", cJSON_CreateObject());
    }

    char *json = cJSON_PrintUnformatted(root);
    if (json != NULL) {
        size_t json_len = strlen(json);
        char *line = malloc(json_len + 2);
        if (line != NULL) {
            memcpy(line, json, json_len);
            line[json_len] = '\n';
            usb_serial_jtag_write_bytes(line, json_len + 1, pdMS_TO_TICKS(200));
            free(line);
        }
        free(json);
    }

    cJSON_Delete(root);
}

static void serial_event_handler(void *arg, esp_event_base_t base, int32_t id, void *data)
{
    if (!serial_protocol_is_host_connected()) {
        return;
    }

    switch ((transmitter_event_id_t)id) {
    case TX_EVENT_WIFI_CONNECTED:
        serial_send_event("event.wifi_connected", NULL);
        break;
    case TX_EVENT_WIFI_DISCONNECTED:
        serial_send_event("event.wifi_disconnected", NULL);
        break;
    case TX_EVENT_MQTT_CONNECTED:
        serial_send_event("event.mqtt_connected", NULL);
        break;
    case TX_EVENT_MQTT_DISCONNECTED:
        serial_send_event("event.mqtt_disconnected", NULL);
        break;
    case TX_EVENT_ACTIVATED:
        serial_send_event("event.activated", NULL);
        break;
    case TX_EVENT_LIFECYCLE_CHANGED: {
        const device_operational_state_t *st = (const device_operational_state_t *)data;
        cJSON *p = cJSON_CreateObject();
        cJSON_AddStringToObject(p, "op_state", device_config_op_state_to_string(*st));
        serial_send_event("event.lifecycle_changed", p);
        break;
    }
    case TX_EVENT_DISPATCH_OK:
    case TX_EVENT_DISPATCH_REJECTED: {
        const dispatch_summary_t *s = (const dispatch_summary_t *)data;
        cJSON *p = cJSON_CreateObject();
        cJSON_AddStringToObject(p, "receiver_public_id", s->receiver_public_id);
        cJSON_AddStringToObject(p, "band", s->band);
        cJSON_AddStringToObject(p, "status", s->status);
        if (s->reason[0] != '\0') {
            cJSON_AddStringToObject(p, "reason", s->reason);
        }
        serial_send_event(id == TX_EVENT_DISPATCH_OK ? "event.dispatch_ok"
                                                     : "event.dispatch_rejected", p);
        break;
    }
    default:
        break;
    }
}
```

**Thread safety:** The event handler runs on the default event loop task. `usb_serial_jtag_write_bytes()` is thread-safe — the TX ring buffer uses a FreeRTOS ring buffer internally, which serializes concurrent writes. Both the protocol task (responses) and the event loop task (events) can write to the TX buffer without additional locking.

#### Step 1.4 — Implement `ping` Command

```c
static void handle_ping(const char *id)
{
    cJSON *payload = cJSON_CreateObject();
    cJSON_AddNumberToObject(payload, "uptime_ms",
                            (double)(esp_timer_get_time() / 1000));
    serial_send_response(id, true, payload, NULL);
}
```

#### Step 1.5 — Implement `identify` Command

```c
static void handle_identify(const char *id)
{
    const device_config_t *cfg = device_config_get();
    uint8_t mac[6];
    esp_read_mac(mac, ESP_MAC_WIFI_STA);

    char mac_str[18];
    snprintf(mac_str, sizeof(mac_str), "%02X:%02X:%02X:%02X:%02X:%02X",
             mac[0], mac[1], mac[2], mac[3], mac[4], mac[5]);

    cJSON *payload = cJSON_CreateObject();
    cJSON_AddStringToObject(payload, "public_id",
                            cfg->has_public_id ? cfg->public_id : "");
    cJSON_AddStringToObject(payload, "device_name",
                            cfg->has_device_name ? cfg->device_name : "");
    cJSON_AddStringToObject(payload, "op_state",
                            cfg->has_op_state
                                ? device_config_op_state_to_string(cfg->op_state)
                                : "INVALID");
    cJSON_AddStringToObject(payload, "firmware_version", CONFIG_TRANSMITTER_FIRMWARE_VERSION);
    cJSON_AddStringToObject(payload, "mac", mac_str);
    serial_send_response(id, true, payload, NULL);
}
```

**`CONFIG_TRANSMITTER_FIRMWARE_VERSION`:** Added to `Kconfig.projbuild` in Step 1.1.

#### Step 1.6 — Implement `status` Command

```c
static void handle_status(const char *id)
{
    const device_config_t *cfg = device_config_get();
    uint8_t mac[6];
    esp_read_mac(mac, ESP_MAC_WIFI_STA);

    char mac_str[18];
    snprintf(mac_str, sizeof(mac_str), "%02X:%02X:%02X:%02X:%02X:%02X",
             mac[0], mac[1], mac[2], mac[3], mac[4], mac[5]);

    cJSON *payload = cJSON_CreateObject();
    cJSON_AddNumberToObject(payload, "schema_version", DEVICE_CFG_SCHEMA_VERSION);
    cJSON_AddBoolToObject(payload, "provisioned", device_config_is_provisioned(cfg));
    cJSON_AddBoolToObject(payload, "activated", device_config_is_activated(cfg));
    cJSON_AddBoolToObject(payload, "recovery_required", device_config_requires_recovery(cfg));
    cJSON_AddBoolToObject(payload, "wifi_connected", wifi_is_sta_connected());
    cJSON_AddBoolToObject(payload, "mqtt_connected", mqtt_is_connected());
    cJSON_AddStringToObject(payload, "op_state",
                            cfg->has_op_state
                                ? device_config_op_state_to_string(cfg->op_state)
                                : "INVALID");
    cJSON_AddStringToObject(payload, "public_id",
                            cfg->has_public_id ? cfg->public_id : "");
    cJSON_AddStringToObject(payload, "device_name",
                            cfg->has_device_name ? cfg->device_name : "");

    if (wifi_is_sta_connected()) {
        cJSON_AddStringToObject(payload, "wifi_ssid", cfg->wifi_ssid);
        int rssi = 0;
        if (esp_wifi_sta_get_rssi(&rssi) == ESP_OK) {
            cJSON_AddNumberToObject(payload, "wifi_rssi", rssi);
        }
        esp_netif_t *netif = esp_netif_get_handle_from_ifkey("WIFI_STA_DEF");
        if (netif != NULL) {
            esp_netif_ip_info_t ip_info;
            if (esp_netif_get_ip_info(netif, &ip_info) == ESP_OK) {
                char ip_str[16];
                snprintf(ip_str, sizeof(ip_str), IPSTR, IP2STR(&ip_info.ip));
                cJSON_AddStringToObject(payload, "ip", ip_str);
            }
        }
    }

    cJSON_AddNumberToObject(payload, "uptime_ms",
                            (double)(esp_timer_get_time() / 1000));
    cJSON_AddNumberToObject(payload, "free_heap",
                            (double)esp_get_free_heap_size());
    cJSON_AddStringToObject(payload, "firmware_version", CONFIG_TRANSMITTER_FIRMWARE_VERSION);
    cJSON_AddStringToObject(payload, "mac", mac_str);

    serial_send_response(id, true, payload, NULL);
}
```

**`esp_wifi_sta_get_rssi(int *rssi)`:** Documented in ESP-IDF v6.0 for STA/APSTA mode after association. Returns `ESP_OK` when connected, `ESP_ERR_WIFI_NOT_CONNECT` otherwise.

**`esp_netif_get_handle_from_ifkey("WIFI_STA_DEF")`:** Returns the STA netif handle without reaching into `wifi.c` internals. This is the standard ESP-IDF pattern for accessing the default STA interface.

#### Step 1.7 — Implement `provision` Command

```c
static void handle_provision(const char *id, const cJSON *payload)
{
    if (payload == NULL) {
        serial_send_error(id, "missing_payload");
        return;
    }

    const char *wifi_ssid = json_get_string(payload, "wifi_ssid");
    const char *wifi_pwd  = json_get_string(payload, "wifi_pwd");
    const char *mqtt_uri  = json_get_string(payload, "mqtt_uri");
    const char *mqtt_user = json_get_string(payload, "mqtt_user");
    const char *mqtt_pwd  = json_get_string(payload, "mqtt_pwd");
    const char *enroll    = json_get_string(payload, "enroll_token");

    if (wifi_ssid == NULL || mqtt_uri == NULL || mqtt_user == NULL ||
        mqtt_pwd == NULL || enroll == NULL) {
        serial_send_error(id, "missing_fields");
        return;
    }

    // wifi_pwd is optional (open networks)
    device_provisioning_request_t req = { 0 };
    strlcpy(req.wifi_ssid, wifi_ssid, sizeof(req.wifi_ssid));
    if (wifi_pwd != NULL) {
        strlcpy(req.wifi_pwd, wifi_pwd, sizeof(req.wifi_pwd));
    }
    strlcpy(req.mqtt_uri, mqtt_uri, sizeof(req.mqtt_uri));
    strlcpy(req.mqtt_user, mqtt_user, sizeof(req.mqtt_user));
    strlcpy(req.mqtt_pwd, mqtt_pwd, sizeof(req.mqtt_pwd));
    strlcpy(req.enroll_token, enroll, sizeof(req.enroll_token));

    esp_err_t err = device_config_save_provisioning(&req);
    if (err != ESP_OK) {
        serial_send_error(id, "nvs_write_failed");
        return;
    }

    cJSON *resp = cJSON_CreateObject();
    cJSON_AddBoolToObject(resp, "restarting", true);
    serial_send_response(id, true, resp, NULL);

    usb_serial_jtag_wait_tx_done(pdMS_TO_TICKS(200));
    esp_restart();
}
```

**Helper for string extraction:**

```c
static const char *json_get_string(const cJSON *obj, const char *key)
{
    const cJSON *item = cJSON_GetObjectItem(obj, key);
    return cJSON_IsString(item) ? cJSON_GetStringValue(item) : NULL;
}
```

#### Step 1.8 — Implement `factory_reset` Command

```c
static void handle_factory_reset(const char *id)
{
    esp_err_t err = device_config_factory_reset();
    if (err != ESP_OK) {
        serial_send_error(id, "reset_failed");
        return;
    }

    cJSON *resp = cJSON_CreateObject();
    cJSON_AddBoolToObject(resp, "restarting", true);
    serial_send_response(id, true, resp, NULL);

    usb_serial_jtag_wait_tx_done(pdMS_TO_TICKS(200));
    esp_restart();
}
```

#### Step 1.9 — Implement `update_mqtt` Command

This command updates only MQTT credentials in NVS without touching WiFi config or enrollment state. It requires a new helper in `device_config`.

**New function in `config/device_config.h`:**

```c
/**
 * @brief Update only the MQTT credentials in NVS.
 *
 * Preserves WiFi config, enrollment token, and all activation-derived keys.
 *
 * @param mqtt_uri  New MQTT broker URI.
 * @param mqtt_user New MQTT username.
 * @param mqtt_pwd  New MQTT password.
 * @return ESP_OK on success, or an ESP-IDF error code.
 */
esp_err_t device_config_save_mqtt(const char *mqtt_uri,
                                  const char *mqtt_user,
                                  const char *mqtt_pwd);
```

**Implementation in `config/device_config.c`:**

Follow the same NVS write pattern as `device_config_save_provisioning()` but only write the three MQTT keys (`mqtt_uri`, `mqtt_user`, `mqtt_pwd`). Update the in-RAM snapshot (`s_cfg`) after a successful NVS commit. Do not erase activation-derived keys.

**Validation rules** (same as provisioning):
- `mqtt_uri` must start with `"mqtts://"` and not exceed `DEVICE_CFG_MQTT_URI_MAX_LEN - 1`
- `mqtt_user` must not exceed `DEVICE_CFG_MQTT_USER_MAX_LEN - 1`
- `mqtt_pwd` must not exceed `DEVICE_CFG_MQTT_PWD_MAX_LEN - 1`

**Serial handler:**

```c
static void handle_update_mqtt(const char *id, const cJSON *payload)
{
    if (payload == NULL) {
        serial_send_error(id, "missing_payload");
        return;
    }

    const char *uri  = json_get_string(payload, "mqtt_uri");
    const char *user = json_get_string(payload, "mqtt_user");
    const char *pwd  = json_get_string(payload, "mqtt_pwd");

    if (uri == NULL || user == NULL || pwd == NULL) {
        serial_send_error(id, "missing_fields");
        return;
    }

    esp_err_t err = device_config_save_mqtt(uri, user, pwd);
    if (err != ESP_OK) {
        serial_send_error(id, "nvs_write_failed");
        return;
    }

    cJSON *resp = cJSON_CreateObject();
    cJSON_AddBoolToObject(resp, "restarting", true);
    serial_send_response(id, true, resp, NULL);

    usb_serial_jtag_wait_tx_done(pdMS_TO_TICKS(200));
    esp_restart();
}
```

#### Step 1.10 — Implement `provision.test_wifi` Command

Temporarily connects to a WiFi network to test credentials, reports the result, then disconnects. Does not save anything to NVS.

```c
static void handle_test_wifi(const char *id, const cJSON *payload)
{
    if (payload == NULL) {
        serial_send_error(id, "missing_payload");
        return;
    }

    // Reject if already STA-connected — don't disrupt an active connection
    if (wifi_is_sta_connected()) {
        serial_send_error(id, "already_connected");
        return;
    }

    const char *ssid = json_get_string(payload, "wifi_ssid");
    const char *pwd  = json_get_string(payload, "wifi_pwd");
    if (ssid == NULL) {
        serial_send_error(id, "missing_fields");
        return;
    }

    // Build a temporary config for STA connection
    device_config_t test_cfg = { 0 };
    strlcpy(test_cfg.wifi_ssid, ssid, sizeof(test_cfg.wifi_ssid));
    if (pwd != NULL) {
        strlcpy(test_cfg.wifi_pwd, pwd, sizeof(test_cfg.wifi_pwd));
    }

    esp_err_t err = wifi_start_sta(&test_cfg);
    if (err != ESP_OK) {
        serial_send_error(id, "wifi_start_failed");
        return;
    }

    err = wifi_wait_for_sta_ready(pdMS_TO_TICKS(15000));

    cJSON *resp = cJSON_CreateObject();
    cJSON_AddBoolToObject(resp, "connected", err == ESP_OK);

    if (err == ESP_OK) {
        int rssi = 0;
        esp_wifi_sta_get_rssi(&rssi);
        cJSON_AddNumberToObject(resp, "rssi", rssi);

        esp_netif_t *netif = esp_netif_get_handle_from_ifkey("WIFI_STA_DEF");
        if (netif != NULL) {
            esp_netif_ip_info_t ip_info;
            if (esp_netif_get_ip_info(netif, &ip_info) == ESP_OK) {
                char ip_str[16];
                snprintf(ip_str, sizeof(ip_str), IPSTR, IP2STR(&ip_info.ip));
                cJSON_AddStringToObject(resp, "ip", ip_str);
            }
        }
    }

    serial_send_response(id, true, resp, NULL);

    // Disconnect and stop STA — release test connection
    (void)wifi_stop();
}
```

**Important considerations:**

- **SoftAP disruption:** If the hub is currently in SoftAP provisioning mode, `wifi_start_sta()` will switch the WiFi mode from AP to STA, dropping the SoftAP. This is acceptable because: (a) USB provisioning replaces the SoftAP flow — the admin is using USB, not SoftAP; (b) after the test, `wifi_stop()` releases WiFi. The SoftAP is NOT auto-restored — the next step in the USB flow is either `provision` (which restarts) or the admin cancels. If the admin abandons the USB session after test_wifi, the device must be restarted to recover SoftAP.
- **15-second timeout:** Shorter than the 45-second STA timeout used in boot. This is a quick connectivity test, not a production connection.
- **`wifi_stop()` at the end:** Releases the STA connection. Does not modify NVS.

#### Step 1.11 — Integrate into `app_main()`

In `main.c`, add the serial protocol init call after `controls_init()` and before `wait_for_station_or_recover()`:

```c
#include "serial/serial_protocol.h"

void app_main(void)
{
    // ... existing init ...
    ESP_ERROR_CHECK(controls_init());
    ESP_ERROR_CHECK(serial_protocol_init());     // ← NEW
    ESP_ERROR_CHECK(wait_for_station_or_recover(force_recovery));
    // ... rest unchanged ...
}
```

The serial protocol task starts early so the hub is responsive to USB commands even during provisioning wait states. It runs concurrently with the rest of the boot sequence.

**Initialization position:** Start serial after `controls_init()` and before `wait_for_station_or_recover()`. This keeps display/LED/control dependencies initialized before serial commands can trigger events, while still making USB available before Wi-Fi provisioning recovery begins.

#### Step 1.12 — Update `CMakeLists.txt`

```cmake
set(SRCS
    # ... existing sources ...
    "serial/serial_protocol.c"       # ← NEW
)

idf_component_register(
    # ... existing config ...
    PRIV_REQUIRES nvs_flash esp_wifi esp_netif esp_event esp_http_server
                  esp_driver_gpio esp_driver_spi esp_timer esp_hw_support
                  mbedtls espressif__mqtt espressif__cjson esp_driver_gptimer
                  esp_lcd esp_driver_i2c lvgl__lvgl espressif__esp_lvgl_port
                  espressif__button
                  esp_driver_usb_serial_jtag     # ← NEW
)
```

**`esp_driver_usb_serial_jtag`:** Required for `usb_serial_jtag_driver_install()`, `usb_serial_jtag_read_bytes()`, `usb_serial_jtag_write_bytes()`, `usb_serial_jtag_vfs_use_driver()`.

**Phase 1 deliverable:** Hub responds to USB serial commands for diagnostics, provisioning, WiFi testing, MQTT credential update, and factory reset. Testable with any serial terminal (e.g., `picocom /dev/ttyACM0 -b 115200`).

---

### Phase 2 — Control Commands & Event Forwarding

**Scope:** Transmit and lifecycle commands over USB, event streaming to browser.

#### Step 2.1 — Implement `transmit` Command

The serial `transmit` handler replicates the MQTT dispatch path but **skips ECDSA signature verification and MQTT ack publication**. It calls `radio_tx_send()` directly.

```c
static bool hex_decode(const char *hex, uint8_t *out, size_t out_size, size_t *out_len)
{
    size_t hex_len = strlen(hex);
    if ((hex_len % 2) != 0 || (hex_len / 2) > out_size) {
        return false;
    }
    for (size_t i = 0; i < hex_len / 2; ++i) {
        unsigned int byte = 0;
        if (sscanf(&hex[i * 2], "%2x", &byte) != 1) {
            return false;
        }
        out[i] = (uint8_t)byte;
    }
    *out_len = hex_len / 2;
    return true;
}

static void handle_transmit(const char *id, const cJSON *payload)
{
    if (payload == NULL) {
        serial_send_error(id, "missing_payload");
        return;
    }

    // Parse fields — dispatch_id is accepted but unused (client-side correlation only)
    const char *receiver_id = json_get_string(payload, "receiver_public_id");
    const char *band_str    = json_get_string(payload, "band");
    const char *rf_hex      = json_get_string(payload, "rf_code_hex");
    const cJSON *bits_item  = cJSON_GetObjectItem(payload, "rf_code_bits");
    const cJSON *proto_item = cJSON_GetObjectItem(payload, "proto_any");

    if (band_str == NULL || rf_hex == NULL || !cJSON_IsNumber(bits_item)) {
        serial_send_error(id, "missing_fields");
        return;
    }

    int rf_code_bits = (int)bits_item->valuedouble;
    bool proto_any = cJSON_IsBool(proto_item) ? cJSON_IsTrue(proto_item) : false;

    // Validate band
    radio_band_t band;
    if (strcmp(band_str, "433M") == 0) {
        band = RADIO_BAND_433M;
    } else if (strcmp(band_str, "2_4G") == 0) {
        band = RADIO_BAND_2_4G;
    } else {
        serial_send_error(id, "invalid_band");
        return;
    }

    // Validate RF code width (same rules as dispatch.c)
    size_t hex_len = strlen(rf_hex);
    if (band == RADIO_BAND_433M) {
        if (rf_code_bits < 1 || rf_code_bits > 32 ||
            hex_len != (size_t)(((rf_code_bits + 7) / 8) * 2)) {
            serial_send_error(id, "width_mismatch");
            return;
        }
    } else {
        if (rf_code_bits != 40 || hex_len != 10) {
            serial_send_error(id, "width_mismatch");
            return;
        }
    }

    // Check device state
    const device_config_t *cfg = device_config_get();
    if (cfg->op_state != OP_STATE_ACTIVE) {
        serial_send_error(id, cfg->op_state == OP_STATE_SUSPENDED
                                  ? "device_suspended"
                                  : "device_decommissioned");
        return;
    }

    // Decode hex payload
    uint8_t plaintext[16] = { 0 };
    size_t byte_len = 0;
    if (!hex_decode(rf_hex, plaintext, sizeof(plaintext), &byte_len)) {
        serial_send_error(id, "bad_hex");
        return;
    }

    // Dispatch RF — no signature verification, no MQTT ack
    esp_err_t tx_result = radio_tx_send(band, plaintext, byte_len,
                                        (uint8_t)rf_code_bits, proto_any);

    cJSON *resp = cJSON_CreateObject();
    if (tx_result == ESP_OK) {
        int64_t applied_at_ms = time_utils_now_epoch_ms();
        cJSON_AddStringToObject(resp, "status", "applied");
        cJSON_AddNumberToObject(resp, "applied_at_ms", (double)applied_at_ms);

        // Post event for display/LED
        dispatch_summary_t summary = { 0 };
        if (receiver_id != NULL) {
            strlcpy(summary.receiver_public_id, receiver_id,
                    sizeof(summary.receiver_public_id));
        }
        strlcpy(summary.band, band_str, sizeof(summary.band));
        summary.rf_code_bits = rf_code_bits;
        summary.proto_any = proto_any;
        strlcpy(summary.status, "applied", sizeof(summary.status));
        summary.applied_at_ms = applied_at_ms;
        esp_event_post(TRANSMITTER_EVENTS, TX_EVENT_DISPATCH_OK,
                       &summary, sizeof(summary), 0);
    } else {
        const char *reason = (tx_result == ESP_ERR_NOT_SUPPORTED)
                                 ? "no_2_4g_radio" : "tx_failed";
        cJSON_AddStringToObject(resp, "status", "rejected");
        cJSON_AddStringToObject(resp, "reason", reason);

        dispatch_summary_t summary = { 0 };
        if (receiver_id != NULL) {
            strlcpy(summary.receiver_public_id, receiver_id,
                    sizeof(summary.receiver_public_id));
        }
        strlcpy(summary.band, band_str, sizeof(summary.band));
        summary.rf_code_bits = rf_code_bits;
        summary.proto_any = proto_any;
        strlcpy(summary.status, "rejected", sizeof(summary.status));
        strlcpy(summary.reason, reason, sizeof(summary.reason));
        esp_event_post(TRANSMITTER_EVENTS, TX_EVENT_DISPATCH_REJECTED,
                       &summary, sizeof(summary), 0);
    }

    serial_send_response(id, true, resp, NULL);
}
```

**Key differences from MQTT path:**
- No `signature_b64` field required or validated
- No dispatch ring dedup (USB commands are local, not replayed by network)
- No MQTT ack publication
- Posts `TX_EVENT_DISPATCH_OK`/`TX_EVENT_DISPATCH_REJECTED` so the OLED display and LED react
- `dispatch_id` is optional (used for client-side correlation only)

**Concurrent dispatch safety:** If a serial `transmit` and an MQTT `cmd/transmit` arrive simultaneously, `radio_supervisor`'s internal mutex serializes radio access. No additional locking needed.

#### Step 2.2 — Implement `lifecycle` Command

```c
static void handle_lifecycle(const char *id, const cJSON *payload)
{
    if (payload == NULL) {
        serial_send_error(id, "missing_payload");
        return;
    }

    const char *action = json_get_string(payload, "action");
    if (action == NULL) {
        serial_send_error(id, "missing_fields");
        return;
    }

    device_operational_state_t new_state;
    if (strcmp(action, "suspend") == 0) {
        new_state = OP_STATE_SUSPENDED;
    } else if (strcmp(action, "resume") == 0) {
        new_state = OP_STATE_ACTIVE;
    } else if (strcmp(action, "decommission") == 0) {
        new_state = OP_STATE_DECOMMISSIONED;
    } else {
        serial_send_error(id, "invalid_action");
        return;
    }

    const device_config_t *cfg = device_config_get();

    // Guard: cannot resume from decommissioned
    if (strcmp(action, "resume") == 0 && cfg->op_state == OP_STATE_DECOMMISSIONED) {
        serial_send_error(id, "cannot_resume_decommissioned");
        return;
    }

    // No-op guard: skip NVS write if already in the requested state
    if (cfg->has_op_state && cfg->op_state == new_state) {
        cJSON *resp = cJSON_CreateObject();
        cJSON_AddStringToObject(resp, "op_state",
                                device_config_op_state_to_string(new_state));
        serial_send_response(id, true, resp, NULL);
        return;
    }

    // Generate a synthetic command_id for NVS dedup tracking
    char cmd_id[32];
    snprintf(cmd_id, sizeof(cmd_id), "usb-%lld", (long long)(esp_timer_get_time() / 1000));

    esp_err_t err = device_config_commit_lifecycle(cmd_id, true, new_state);
    if (err != ESP_OK) {
        serial_send_error(id, "lifecycle_failed");
        return;
    }

    // Post event for display/LED
    esp_event_post(TRANSMITTER_EVENTS, TX_EVENT_LIFECYCLE_CHANGED,
                   &new_state, sizeof(new_state), 0);

    // Manage heartbeat: stop on suspend/decommission, start on resume
    if (new_state == OP_STATE_SUSPENDED || new_state == OP_STATE_DECOMMISSIONED) {
        (void)heartbeat_stop();
    } else if (new_state == OP_STATE_ACTIVE) {
        (void)heartbeat_start();
    }

    cJSON *resp = cJSON_CreateObject();
    cJSON_AddStringToObject(resp, "op_state",
                            device_config_op_state_to_string(new_state));
    serial_send_response(id, true, resp, NULL);
}
```

**Key differences from MQTT path:**
- No ECDSA signature verification
- No MQTT deact ack publication
- Synthetic `command_id` for NVS dedup (prevents repeated USB commands from conflicting with MQTT command IDs)
- Posts `TX_EVENT_LIFECYCLE_CHANGED` for display/LED
- Manages heartbeat start/stop directly

**`heartbeat_start/stop`:** Declared in `network/heartbeat.h`, included in the serial module's header block (Step 1.3A). The INCLUDE_DIRS "." in CMakeLists makes all module headers accessible.

#### Step 2.3 — Event Forwarding (Already Implemented in Step 1.3E)

The event handler registered in `serial_protocol_init()` forwards `TRANSMITTER_EVENTS` to USB serial as JSON event messages. Only forwards when `serial_protocol_is_host_connected()` returns true, avoiding unnecessary TX writes when no host is listening.

Events forwarded:

| Event                       | Serial event type              | Payload                           |
| --------------------------- | ------------------------------ | --------------------------------- |
| `TX_EVENT_WIFI_CONNECTED`   | `event.wifi_connected`         | (none)                            |
| `TX_EVENT_WIFI_DISCONNECTED`| `event.wifi_disconnected`      | (none)                            |
| `TX_EVENT_MQTT_CONNECTED`   | `event.mqtt_connected`         | (none)                            |
| `TX_EVENT_MQTT_DISCONNECTED`| `event.mqtt_disconnected`      | (none)                            |
| `TX_EVENT_ACTIVATED`        | `event.activated`              | (none)                            |
| `TX_EVENT_LIFECYCLE_CHANGED`| `event.lifecycle_changed`      | `{op_state}`                      |
| `TX_EVENT_DISPATCH_OK`      | `event.dispatch_ok`            | `{receiver_public_id, band, status}` |
| `TX_EVENT_DISPATCH_REJECTED`| `event.dispatch_rejected`      | `{receiver_public_id, band, status, reason}` |

**Phase 2 deliverable:** Hub can receive transmit/lifecycle commands over USB. Hub streams events to USB when a host is connected.

---

## Changes to Existing Modules (Summary)

| File                    | Change                                                | Type     |
| ----------------------- | ----------------------------------------------------- | -------- |
| `main.c`                | Add `#include "serial/serial_protocol.h"`, call `serial_protocol_init()` after `controls_init()` | Additive |
| `config/device_config.h`| Add `device_config_save_mqtt()` declaration           | Additive |
| `config/device_config.c`| Implement `device_config_save_mqtt()` (NVS write of 3 MQTT fields + snapshot update) | Additive |
| `main/CMakeLists.txt`   | Add `serial/serial_protocol.c` to SRCS, add `esp_driver_usb_serial_jtag` to PRIV_REQUIRES | Additive |
| `sdkconfig`             | Add `CONFIG_USJ_NO_AUTO_LS_ON_CONNECTION=y` only if `CONFIG_PM_ENABLE=y`; current config leaves PM disabled | Conditional |
| `Kconfig.projbuild`     | Add `CONFIG_TRANSMITTER_FIRMWARE_VERSION` string option (if not present) | Additive |

All changes are additive. No existing function signatures or behaviors are modified.

---

## Memory Budget

| Component                       | Estimate     |
| ------------------------------- | ------------ |
| Protocol task stack             | ~4 KB        |
| Line buffer (on stack)          | ~1 KB        |
| TX ring buffer                  | 4 KB         |
| RX ring buffer                  | 4 KB         |
| cJSON transient allocations     | ~2 KB peak   |
| Event handler overhead          | Negligible   |
| **Total**                       | **~15 KB**   |

Combined with the OLED plan's ~22 KB estimate, the total display + serial budget is ~37 KB out of 400 KB SRAM. Leaves ample headroom for WiFi (~40 KB), MQTT (~10 KB), and radio operations.

---

## Risk Assessment

| Risk                                        | Severity | Mitigation                                                                                                   |
| ------------------------------------------- | -------- | ------------------------------------------------------------------------------------------------------------ |
| USB cable plugged in without active protocol session | Low | `serial_protocol_is_host_connected()` combines `usb_serial_jtag_is_connected()` (hardware SOF) with an application-level session flag; events only forward after a valid JSON command is received |
| TX buffer not fully drained before restart  | Low      | `usb_serial_jtag_wait_tx_done(pdMS_TO_TICKS(200))` blocks until TX drained before `esp_restart()` in provision/reset/update_mqtt handlers |
| Console log interleaves with protocol JSON  | Low      | By design: browser discriminates by `{` prefix. Each write is atomic at ring buffer level                    |
| `provision.test_wifi` disrupts SoftAP       | Low      | Acceptable: admin using USB replaces SoftAP flow. Test is rejected if already STA-connected                  |
| cJSON allocation failure on status response | Low      | Status response is ~500 bytes. cJSON ceiling trivial. Use `cJSON_PrintUnformatted` (no whitespace)           |
| Concurrent serial + MQTT transmit           | Low      | `radio_supervisor` mutex serializes radio access. No additional locking needed                                |
| VFS line ending translation on log output   | Low      | Only affects log lines via VFS (may add `\r`). Protocol writes bypass VFS. Browser parser trims `\r`        |
| Serial task stack overflow                  | Low      | 4096 bytes. Verify with `uxTaskGetStackHighWaterMark()` after integration; bump to 6144 if tight            |

---

## Testing Checklist

All commands can be tested with a serial terminal connected to `/dev/ttyACM0` at 115200 baud:

```bash
picocom /dev/ttyACM0 -b 115200 --omap crcrlf
```

| Test | Command | Expected |
|------|---------|----------|
| Ping | `{"id":"1","type":"ping"}` | `{"id":"1","type":"response","ok":true,"payload":{"uptime_ms":...}}` |
| Identify | `{"id":"2","type":"identify"}` | Response with MAC, op_state, firmware_version |
| Status | `{"id":"3","type":"status"}` | Full device snapshot |
| Provision | `{"id":"4","type":"provision","payload":{"wifi_ssid":"test","mqtt_uri":"mqtts://...","mqtt_user":"u","mqtt_pwd":"p","enroll_token":"tok"}}` | Response with `restarting: true`, then device reboots |
| Test WiFi | `{"id":"5","type":"provision.test_wifi","payload":{"wifi_ssid":"MyNetwork","wifi_pwd":"pass"}}` | Response with connected/ip/rssi |
| Update MQTT | `{"id":"6","type":"update_mqtt","payload":{"mqtt_uri":"mqtts://new","mqtt_user":"u","mqtt_pwd":"p"}}` | Response with `restarting: true` |
| Factory Reset | `{"id":"7","type":"factory_reset"}` | Response with `restarting: true`, NVS wiped |
| Transmit | `{"id":"8","type":"transmit","payload":{"band":"433M","rf_code_hex":"ABCD","rf_code_bits":16,"proto_any":false}}` | Response with status: applied/rejected |
| Lifecycle | `{"id":"9","type":"lifecycle","payload":{"action":"suspend"}}` | Response with new op_state |
| Console logs | (just observe) | ESP-IDF log lines (starting with `I`, `W`, `E`, etc.) interspersed with JSON responses |
| Events | (trigger WiFi disconnect) | Unsolicited `{"id":null,"type":"event.wifi_disconnected","payload":{}}` |
