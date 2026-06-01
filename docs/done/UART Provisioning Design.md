# UART Provisioning Protocol Specification

Replaces the SoftAP + HTTP provisioning flow with a command-response protocol over UART0, compatible with USB-serial adapters and the Web Serial API.

## Transport Layer

| Parameter | Value |
|-----------|-------|
| Interface | UART0 (GPIO1 TX, GPIO3 RX) |
| Baud rate | 115200 |
| Data bits | 8 |
| Parity | None |
| Stop bits | 1 |
| Flow control | None |
| Line ending (TX from ESP) | `\n` (LF) |
| Line ending (RX to ESP) | `\n` or `\r\n` (LF or CRLF) |

The console baud rate (`CONFIG_ESP_CONSOLE_UART_BAUDRATE`) changes from 74880 to 115200 so that ESP_LOG output and provisioning share the same UART at the same rate.

## Framing

Every message is a single line of JSON terminated by `\n`. No binary framing, no length prefixes.

- **Request** (host to ESP): one JSON object per line, max 2048 bytes
- **Response** (ESP to host): one JSON object per line, prefixed with `>` to distinguish from log output

The `>` prefix allows a Web Serial client to split the UART stream into log lines (no prefix) and protocol responses (`>` prefix) without ambiguity. Example raw UART output:

```
I (1234) MAIN: receiver-8266 f8a2469 starting
I (1250) MAIN: Platform init complete
I (1260) MAIN: Config state: UNPROVISIONED
I (1265) UART_PROV: Listening for provisioning commands
>{"status":"ok","action":"ready","device":{"mac_label":"AB12","firmware_version":"f8a2469"}}
```

## Command Format

Every request is a JSON object with an `"action"` field:

```json
{"action": "<command_name>", ...additional fields}
```

Every response is a JSON object prefixed with `>`, containing at minimum:

```json
>{"status": "ok"|"error", ...response fields}
```

On parse failure (malformed JSON, missing action, line too long), the ESP responds:

```json
>{"status":"error","error":"<reason>"}
```

Possible error reasons: `"parse_error"`, `"unknown_action"`, `"missing_field"`, `"invalid_field"`, `"line_too_long"`, `"persist_failed"`.

## Commands

### 1. `status`

Query the device's current provisioning and operational state.

**Request:**
```json
{"action":"status"}
```

**Response:**
```json
>{
  "status": "ok",
  "action": "status",
  "device": {
    "mac_label": "AB12",
    "firmware_version": "f8a2469",
    "boot_state": "UNPROVISIONED",
    "op_state": "UNPROVISIONED",
    "wifi_ssid": "",
    "public_id": "",
    "has_config": false,
    "can_retry": false,
    "recovery_reason": "UNPROVISIONED"
  }
}
```

The `device` object matches the existing HTTP `GET /api/status` response fields exactly.

### 2. `provision`

Write provisioning credentials to NVS and trigger a reboot.

**Request:**
```json
{
  "action": "provision",
  "schema_version": 1,
  "wifi": {
    "ssid": "MyNetwork",
    "password": "MyPassword"
  },
  "mqtt": {
    "broker_uri": "mqtts://broker.example.com:8883",
    "username": "device_user",
    "password": "device_pass"
  },
  "enrollment": {
    "token": "enroll-token-abc123"
  }
}
```

**Validation rules** (same as the existing HTTP `POST /api/provision`):
- `schema_version` must be integer `1`
- `wifi`, `mqtt`, `enrollment` must be JSON objects
- All string fields must be non-empty
- `mqtt.broker_uri` must start with `mqtts://`
- Total line length must not exceed 2048 bytes

**Success response (sent before reboot):**
```json
>{"status":"ok","action":"provision"}
```

The ESP flushes the UART TX buffer, waits 250ms for the host to read the response, then calls `esp_restart()`.

**Error response:**
```json
>{"status":"error","action":"provision","error":"missing_field"}
```

### 3. `retry`

Re-attempt the boot sequence using existing provisioning data from NVS. Only valid when `can_retry` is `true` in the status response.

**Request:**
```json
{"action":"retry"}
```

**Success response (sent before reboot):**
```json
>{"status":"ok","action":"retry"}
```

The ESP reboots after flushing UART.

**Error response** (if no provisioning data exists):
```json
>{"status":"error","action":"retry","error":"not_provisioned"}
```

### 4. `reset`

Erase all provisioning and runtime data from NVS and reboot to a clean state.

**Request:**
```json
{"action":"reset"}
```

**Success response (sent before reboot):**
```json
>{"status":"ok","action":"reset"}
```

The ESP erases the `device_cfg` NVS namespace, flushes UART, and reboots.

### 5. `identity`

Retrieve the device's public key for enrollment. Not present in the HTTP API but useful for serial-based onboarding flows.

**Request:**
```json
{"action":"identity"}
```

**Response:**
```json
>{
  "status": "ok",
  "action": "identity",
  "mac_label": "AB12",
  "public_key_b64": "MFkwEwYH...base64...=="
}
```

## Boot Flow Changes

The `app_main()` loop changes from:

```
if not provisioned -> start SoftAP + HTTP server -> wait for HTTP POST
```

to:

```
if not provisioned -> enter UART provisioning loop -> wait for UART command
```

The UART provisioning loop:

1. Install UART driver on UART0 (115200, 8N1, RX buffer 2048 bytes)
2. Send a `ready` announcement so Web Serial knows the device is waiting:
   ```json
   >{"status":"ok","action":"ready","device":{"mac_label":"AB12","firmware_version":"f8a2469"}}
   ```
3. Read lines from UART0 (blocking, no timeout)
4. Parse JSON, dispatch to command handler
5. Send JSON response prefixed with `>`
6. If command was `provision`, `retry`, or `reset`: flush UART, delay 250ms, reboot

The provisioning loop also runs when any boot stage fails (WiFi, MQTT, activation), matching the current recovery mode behavior. The `recovery_reason` field in the `ready` announcement and `status` response tells the host why the device entered provisioning mode.

UART provisioning also listens during normal operational mode in the background, allowing `status` and `reset` commands at any time. Only `provision` and `retry` trigger a reboot.

## Architecture

### New files
- `main/provision/uart_prov.h` — Public API: `uart_prov_init()`, `uart_prov_run_blocking()`, `uart_prov_start_background()`
- `main/provision/uart_prov.c` — UART driver setup, line reader, JSON command dispatcher

### Modified files
- `main/main.c` — Replace `provision_run_recovery_mode()` calls with `uart_prov_run_blocking()`
- `sdkconfig` — Change `CONFIG_ESP_CONSOLE_UART_BAUDRATE` to 115200, `CONFIG_ESPTOOLPY_MONITOR_BAUD` to 115200

### Removed files
- `main/provision/http_server.c` / `http_server.h` — No longer needed
- `main/provision/recovery.c` / `recovery.h` — Replaced by uart_prov
- `main/network/wifi.c` — Remove `wifi_start_softap()` function
- `main/network/wifi.h` — Remove `wifi_start_softap()` declaration
- `main/Kconfig.projbuild` — Remove `RECEIVER_AP_PASSWORD` and `RECEIVER_AP_CHANNEL`
- Embedded provisioning HTML (`index.html.gz`) — No longer needed

### Reused without changes
- `main/config/device_config.c` — All NVS storage functions remain identical
- `main/security/device_identity.c` — Used by the `identity` command
- cJSON library — Already linked, used for parsing

## UART Driver Configuration

```c
uart_config_t uart_cfg = {
    .baud_rate = 115200,
    .data_bits = UART_DATA_8_BITS,
    .parity = UART_PARITY_DISABLE,
    .stop_bits = UART_STOP_BITS_1,
    .flow_ctrl = UART_HW_FLOWCTRL_DISABLE,
};
```

RX ring buffer: 2048 bytes (matches max line length).
TX ring buffer: 0 (blocking writes, no buffering needed).
No UART event queue needed — line reading is synchronous.

## Web Serial API Integration Notes

These notes are for the host-side implementation (browser or CLI tool), not the ESP firmware.

### Connecting
```javascript
const port = await navigator.serial.requestPort();
await port.open({ baudRate: 115200 });
```

### Sending a command
```javascript
const encoder = new TextEncoder();
const writer = port.writable.getWriter();
await writer.write(encoder.encode(JSON.stringify({action: "status"}) + "\n"));
writer.releaseLock();
```

### Reading responses
Read the stream line by line. Lines starting with `>` are protocol responses (strip the `>`, parse as JSON). All other lines are ESP log output — display or discard.

```javascript
const reader = port.readable.getReader();
let buffer = "";

while (true) {
  const { value, done } = await reader.read();
  if (done) break;
  buffer += new TextDecoder().decode(value);

  let newlineIdx;
  while ((newlineIdx = buffer.indexOf("\n")) !== -1) {
    const line = buffer.substring(0, newlineIdx).trimEnd();
    buffer = buffer.substring(newlineIdx + 1);

    if (line.startsWith(">")) {
      const response = JSON.parse(line.substring(1));
      handleResponse(response);
    } else {
      handleLogLine(line);
    }
  }
}
```

### Provisioning sequence
1. Open serial port at 115200
2. Wait for `>{"status":"ok","action":"ready",...}` line (device booted and waiting)
3. Optionally send `{"action":"status"}\n` to query current state
4. Send `{"action":"provision",...}\n` with full credentials
5. Wait for `>{"status":"ok","action":"provision"}` confirmation
6. Device reboots automatically — wait for next `ready` or log output to confirm boot

### Reconnection
After a `provision`, `retry`, or `reset` command, the device reboots. The serial port stays open on the host side. The host should watch for the next `ready` announcement or log lines to detect that the device has restarted.

## Constraints

- **Max line length**: 2048 bytes. Lines exceeding this are discarded with `>{"status":"error","error":"line_too_long"}`.
- **Concurrent commands**: None. Commands are processed sequentially. Do not send a new command before receiving the response to the previous one.
- **Log interleaving**: ESP_LOG output may appear between commands. The `>` prefix on responses prevents confusion. The host must filter by prefix.
- **UART0 sharing**: UART0 is shared between ESP_LOG output and provisioning. This is intentional — logs provide debugging context alongside provisioning. The `>` prefix disambiguates.
- **No authentication**: The UART interface has no auth. Physical serial access implies physical device access, which is sufficient for provisioning trust.
