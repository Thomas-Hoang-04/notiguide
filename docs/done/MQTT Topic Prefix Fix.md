# MQTT Topic Prefix Fix

## Problem

All three firmware projects (transmitter, receiver-esp32, receiver-esp8266) publish and subscribe to bare MQTT topics (e.g., `transmitter/bootstrap/register`). The backend prepends a configurable prefix (`notiguide/` by default, via `MqttProperties.topicPrefix`). As a result, firmware and backend cannot communicate over MQTT — messages are published to topics no one subscribes to, and vice versa.

## Fix

Add a Kconfig string option to each firmware project (`CONFIG_*_MQTT_TOPIC_PREFIX`, default `"notiguide"`). Define a compile-time concatenation macro `TOPIC_PREFIX` that prepends this value plus `/` to every topic string.

## Kconfig Entries

### transmitter/main/Kconfig.projbuild

```
config TRANSMITTER_MQTT_TOPIC_PREFIX
    string "MQTT topic prefix"
    default "notiguide"
```

### receiver-esp32/main/Kconfig.projbuild

```
config RECEIVER_MQTT_TOPIC_PREFIX
    string "MQTT topic prefix"
    default "notiguide"
```

### receiver-esp8266/main/Kconfig.projbuild

```
config RECEIVER_MQTT_TOPIC_PREFIX
    string "MQTT topic prefix"
    default "notiguide"
```

## Macro Definition (per project)

Each mqtt.c (or a shared header) defines:

```c
#define TOPIC_PREFIX CONFIG_<PROJECT>_MQTT_TOPIC_PREFIX "/"
```

## Changes Per File

### transmitter/main/network/mqtt.c

| Line | Before | After |
|------|--------|-------|
| 92 | `"transmitter/bootstrap/register"` | `TOPIC_PREFIX "transmitter/bootstrap/register"` |
| 114 | `"transmitter/hub/%s/cmd/transmit"` | `TOPIC_PREFIX "transmitter/hub/%s/cmd/transmit"` |
| 116 | `"transmitter/hub/%s/cmd/deact"` | `TOPIC_PREFIX "transmitter/hub/%s/cmd/deact"` |
| 134 | `"transmitter/bootstrap/+"` | `TOPIC_PREFIX "transmitter/bootstrap/+"` |
| 172 | `"transmitter/bootstrap/%s"` | `TOPIC_PREFIX "transmitter/bootstrap/%s"` |
| 175 | `"transmitter/bootstrap/+"` | `TOPIC_PREFIX "transmitter/bootstrap/+"` |
| 211 | `"transmitter/bootstrap/%s"` | `TOPIC_PREFIX "transmitter/bootstrap/%s"` |
| 323 | `strncmp(topic, "transmitter/bootstrap/", 22)` | `strncmp(topic, TOPIC_PREFIX "transmitter/bootstrap/", sizeof(TOPIC_PREFIX "transmitter/bootstrap/") - 1)` |
| 333 | `"transmitter/hub/%s/cmd/transmit"` | `TOPIC_PREFIX "transmitter/hub/%s/cmd/transmit"` |
| 339 | `"transmitter/hub/%s/cmd/deact"` | `TOPIC_PREFIX "transmitter/hub/%s/cmd/deact"` |
| 572 | `"transmitter/bootstrap/+"` | `TOPIC_PREFIX "transmitter/bootstrap/+"` |

### transmitter/main/network/heartbeat.c

| Line | Before | After |
|------|--------|-------|
| 46 | `"transmitter/hub/%s/heartbeat"` | `TOPIC_PREFIX "transmitter/hub/%s/heartbeat"` |

### transmitter/main/dispatch/dispatch.c

| Line | Before | After |
|------|--------|-------|
| 208 | `"transmitter/hub/%s/ack"` | `TOPIC_PREFIX "transmitter/hub/%s/ack"` |
| 242 | `"transmitter/hub/%s/ack"` | `TOPIC_PREFIX "transmitter/hub/%s/ack"` |

### receiver-esp32/main/network/mqtt.c

All `"receiver/bootstrap/..."` and `"receiver/device/..."` topic strings get the `TOPIC_PREFIX` prepended. Same pattern as transmitter.

### receiver-esp8266/main/network/mqtt.c

All `"receiver/bootstrap/..."` and `"receiver/device/..."` topic strings get the `TOPIC_PREFIX` prepended. Same pattern as transmitter.

## Buffer Safety

Longest topic with prefix: `notiguide/transmitter/hub/HUB-XXXX/cmd/transmit` = ~52 chars. All topic buffers are 128 bytes. No overflow.

## Verification

After applying the fix:
1. Build each firmware project (`idf.py build`)
2. Flash transmitter, provision via USB, confirm bootstrap register message reaches backend
3. Confirm backend creates PENDING device
4. Confirm frontend `pollForPendingDevice` finds the new device
