# Transmitter Hub Domain

Analysis and recommendations for the device notification layer — the physical "buzzer pager" system powered by the ESP32 hub.

---

## What Exists Today

| Component | Status | Notes |
|---|---|---|
| **RF layer** (`transmitter/main/rf/`) | Working | 15 protocols, tri-state encoding, NVS persistence, GPIO TX/RX via timer ISR |
| **RadioLib** (idf_component.yml) | Declared | NRF24L01 support available but not integrated |
| **MQTT backend** (`core/mqtt/`) | Complete | MQTT v5, QoS 1, publishes 4 event types to `notiguide/store/{storeId}/queue` |
| **DB schema** (`notifier_device` table) | Exists | UUID id, device_token, name, store_id, battery, last_ping |
| **Backend domain** (entity/service/controller) | Not started | P1 in backlog |
| **Hub firmware** (`main.cpp`) | Empty stub | No WiFi, no MQTT client, no OLED, no NRF24 logic |

---

## The Intended Flow (Annotated)

```
Customer arrives
    |
    +-- Phone user: scans QR -> joins queue via client-web -> gets FCM notifications
    |
    +-- Pager user: admin provisions a physical device from the dashboard
         |
         +-- Backend creates a ticket linked to a device index
         +-- Hub is told: "device index N is now ticket T in store S"
         |
         ... time passes ...
         |
         Admin calls next ticket
         |
         +-- Backend publishes TICKET_CALLED via MQTT
         +-- Hub receives -> looks up device index N
         +-- Hub transmits on the appropriate band (433 MHz or 2.4 GHz) based on receiver type
         +-- Pager buzzes/vibrates
```

This flow works. Below are the gaps, risks, and recommendations.

---

## Problem 1: The Hub Is a Single Point of Failure

The hub is the only bridge between backend and pagers. If it loses WiFi, power, or crashes mid-shift, every pager-based customer is stranded.

**Recommendations:**

- **Offline queue buffer.** If MQTT connection drops, the hub should buffer incoming commands in a RAM queue (or SPIFFS/LittleFS if persistent recovery is needed). On reconnect, the hub replays missed `TICKET_CALLED` events. MQTT QoS 1 helps here — the broker will redeliver unacked messages — but the hub needs to handle duplicates (idempotent by ticket ID).

- **Visual status indicator.** The OLED (or a dedicated LED) should always show connection state: WiFi status, MQTT status, device count. The admin should never have to guess if the hub is alive.

- **Heartbeat to backend.** Publish `notiguide/hub/{hubId}/heartbeat` every 30-60s. If the backend stops seeing heartbeats, surface a warning on the admin dashboard prompting the admin to switch to Web Serial: "Hub MQTT offline — connect via USB for direct control." Since the admin always has the Chrome dashboard tab open, Web Serial is a natural fallback that's already available.

- **Web Serial as backup communication.** When MQTT is down (or unavailable), the admin dashboard can send device commands directly to the hub over Web Serial. The hub firmware should accept the same command format from both MQTT and serial input, so the dispatch logic is shared. The dashboard auto-detects which channel is available and falls through: MQTT first, Web Serial if MQTT heartbeat is stale.

---

## Problem 2: Dual-Band RF — 433 MHz and NRF24L01

The hub carries both a 433 MHz module and an NRF24L01 (2.4 GHz) because **different receivers use different bands**. Some pagers are 433 MHz (legacy or simple OOK/ASK receivers), others are 2.4 GHz (NRF24-equipped MCU-based devices). The hub must support both simultaneously.

| | 433 MHz (OOK/ASK) | NRF24L01 (2.4 GHz) |
|---|---|---|
| Range | 50-200m (open air), good wall penetration | 30-100m (open air), weaker through walls |
| Data rate | ~1-10 kbps | Up to 2 Mbps |
| Addressing | Code-based (existing RF layer) | Native 5-byte address, 6 pipes |
| Power | Very low TX, no RX needed on pager | Higher, but bidirectional |
| Interference | Less crowded band | WiFi/BT/microwave congestion |
| Acknowledgment | None (fire-and-forget) | Built-in auto-ack + retransmit |
| Receiver type | Legacy decoders (PT2272), simple pagers | MCU-based receivers with NRF24 |

### Implications for the Hub Firmware

- The hub needs **two independent TX paths** — one for the 433 MHz GPIO/timer driver (existing `rf_send()`), one for the NRF24L01 via SPI (RadioLib).
- Each registered receiver must have a `band` attribute (`RF_433` or `NRF24`) so the hub knows which radio to use.
- The `rf_trans_event_queue` should carry band information alongside the device index, so the transmitter task routes to the correct radio.

### Implications for NRF24L01 Receivers

NRF24L01 enables bidirectional communication, which opens possibilities not available on 433 MHz:
- **Delivery acknowledgment** — the hub knows the pager actually received the signal (auto-ack)
- **Battery reporting** — the pager can periodically send battery level back to the hub
- **Richer payloads** — ticket number, counter ID, custom vibration pattern can be transmitted directly rather than relying on a fixed code lookup

These are natural advantages of the 2.4 GHz path but not required for v1 — the NRF24L01 can initially be used in fire-and-forget mode just like 433 MHz, with bidirectional features added later.

---

## Problem 3: MQTTS and Web Serial — Dual Communication Layers

The hub has two communication paths to the admin dashboard, with clear priority ordering:

| | MQTTS (primary) | Web Serial (backup) |
|---|---|---|
| Requires WiFi | Yes | No |
| Requires USB connection | No | Yes |
| Requires Chrome tab open | No | Yes (but admin always has it open) |
| Works headless | Yes | No |
| Latency | ~50-200ms | ~10-30ms |
| Multi-admin support | Yes (broker fans out) | No (1:1 serial session) |

**MQTTS is the primary path.** It works over WiFi with no physical tether and supports multiple admin sessions via the broker.

**Web Serial is the backup runtime path.** Since the admin always has the Chrome dashboard tab open, Web Serial is available whenever the hub is plugged into the admin's computer via USB. When the backend detects a stale hub heartbeat, the dashboard prompts the admin to connect via USB. The dashboard then sends device commands directly over serial.

### How the Fallback Works

```
Dashboard sends command (assign/trigger/release)
    |
    +-- Is MQTT connected? (hub heartbeat recent)
    |   +-- Yes -> send via backend API -> MQTT -> hub
    |   +-- No  -> Is Web Serial connected?
    |       +-- Yes -> send directly over serial -> hub
    |       +-- No  -> Show "Connect hub via USB" prompt
```

### Firmware Requirement

The hub must accept the same JSON command format from both:
- MQTT message callback
- UART/CDC serial input

A shared `command_dispatch()` function parses the JSON payload and routes to the appropriate action regardless of source. This keeps the two paths from diverging.

### Additional Uses for Web Serial

Beyond backup runtime control, Web Serial is also useful for:
- Setting WiFi credentials on first boot
- Flashing/updating device code tables
- Reading debug logs
- Factory reset

---

## Problem 4: Device Provisioning Flow Needs More Detail

"Admin provisions a device from the dashboard" hides a lot of complexity. Here's what actually needs to happen:

### What the Hub Needs to Know

For each active pager, the hub stores:
- **Device index** (0-N): position in its local code table
- **Band** (`RF_433` or `NRF24`): which radio to use
- **Ticket ID** (UUID): which ticket this pager is serving
- **RF code** (tri-state string for 433 MHz, or NRF24 address for 2.4 GHz)
- **Protocol index**: which of the 15 protocols to use (433 MHz only)

### The Provisioning Sequence

```
Admin clicks "Assign Pager" on ticket T
    |
    +-- Frontend sends POST /api/queue/admin/{storeId}/tickets/{ticketId}/assign-device
    |   Body: { deviceId: "uuid-of-device" }
    |
    +-- Backend:
    |   1. Validates ticket exists and is WAITING
    |   2. Validates device exists, is active, and isn't already assigned
    |   3. Stores assignment: ticket hash gets device reference, device gets current_ticket
    |   4. Publishes MQTT (or relayed via Web Serial):
    |      { type: DEVICE_ASSIGNED, ticketId, deviceIndex, band, ... }
    |
    +-- Hub receives DEVICE_ASSIGNED:
        1. Marks device index as "in use" for ticket T
        2. Optionally: brief blink/beep on the pager to confirm pairing (2.4 GHz only — requires bidirectional)
```

### The Notification Sequence (on TICKET_CALLED)

```
Admin calls next -> Backend publishes TICKET_CALLED with ticketId
    |
    +-- Hub:
        1. Looks up ticketId -> finds device index and band
        2. If RF_433: loads tri-state code from NVS, calls rf_send()
           If NRF24: loads address, sends payload via RadioLib
        3. Shows on OLED: "Calling #A-042 -> Pager 3"
        4. Optionally publishes MQTT ack: { type: DEVICE_TRIGGERED, ticketId, deviceIndex }
```

### The Return Sequence (on TICKET_SERVED or TICKET_CANCELLED)

```
Admin serves/cancels ticket -> Backend publishes event
    |
    +-- Hub:
        1. Looks up ticketId -> finds device index
        2. Clears assignment: device is now available
        3. Updates OLED: removes pager from active list
        4. For NRF24 receivers: can send a "stop" command
           For 433 MHz legacy: pager times out on its own
```

**Critical gap in your current MQTT message format:** `QueueEvent` doesn't carry device info. You'll need to either:
1. Add device fields to `QueueEvent` (simple, but pollutes events that don't involve devices), or
2. Publish device events on a separate topic: `notiguide/store/{storeId}/device` (cleaner separation)

Option 2 is better. The hub subscribes to the device topic only; it doesn't need to process every queue event.

---

## Problem 5: Code Space — Legacy Fixed Codes and Dynamic Expansion

The current RF layer uses a fixed code table with 4 channels (`CHN_COUNT_MAX = 4`), designed for legacy decoders like PT2272. This is intentionally limited — PT2272 has a fixed address space and the 4-channel model maps directly to how those chips work.

### Two-Tier Code Strategy

| Tier | Receiver type | Code model | Capacity |
|---|---|---|---|
| **Legacy** | PT2272-based, fixed-code 433 MHz | Current tri-state code table, 4 channels, 15 protocols | ~20-30 usable unique codes |
| **Dynamic** | MCU-based receivers (433 MHz with software decoder, or NRF24L01) | Hub assigns codes at provisioning time from a larger address space | Effectively unlimited |

The legacy tier keeps the current `rf_transmitter.c` / `rf_data.c` code as-is. The dynamic tier is a modified copy with:
- Larger address space (not constrained to PT2272's 2^12 tri-state codes)
- Assignable at runtime rather than baked into NVS at factory
- For NRF24: uses the native 5-byte address — no tri-state encoding needed

### Firmware Implications

- The hub needs two code paths: `rf_send_legacy()` (existing) and `rf_send_dynamic()` (new)
- Each registered device carries a `device_type` flag: `LEGACY_433`, `DYNAMIC_433`, or `NRF24`
- Legacy devices use the fixed NVS code table. Dynamic devices get their code assigned from the backend at provisioning time and stored in a separate runtime table (RAM or NVS partition).

### Backend Implications

The device registration endpoint needs a `deviceType` field. For dynamic devices, the backend (or hub) generates the RF code/address. For legacy devices, the code is fixed at manufacturing time and the admin enters it manually (or scans it from the device).

---

## Problem 6: What the OLED Should Display

The 0.96" OLED (128x64, likely SSD1306) is small. Don't try to show everything. Prioritize:

**Idle screen:**
```
+--------------------+
| NotiGuide Hub      |
| WiFi: OK  MQTT: OK |
| Pagers: 8/20       |
| Store: MegaMart    |
+--------------------+
```

**On transmission:**
```
+--------------------+
| >> CALLING <<      |
| Ticket: A-042      |
| Pager:  #3 [433]   |
| [========  ] TX... |
+--------------------+
```

**On error (MQTT down, Web Serial available):**
```
+--------------------+
| !! MQTT OFFLINE    |
| Serial: READY      |
| Buffered: 3 msgs   |
| Last OK: 14:32     |
+--------------------+
```

**On error (both down):**
```
+--------------------+
| !! NO CONNECTION   |
| MQTT: DOWN         |
| Serial: N/A        |
| Connect USB        |
+--------------------+
```

Keep it status-oriented. The admin's detailed view is the web dashboard, not a 1-inch screen.

---

## Problem 7: LED Strategy

> "Some LEDs (single-color or WS2812)"

LEDs on the hub itself serve as status indicators. Don't overdo it.

| LED | Purpose | Behavior |
|---|---|---|
| Green (or WS2812 green) | System OK | Solid when WiFi + MQTT connected |
| Yellow (or WS2812 yellow) | Degraded | Solid when MQTT down but Web Serial active |
| Red | Error | Solid = no WiFi and no serial; Blinking = WiFi OK but MQTT down, no serial |
| Blue (brief flash) | TX active | Flash on each RF transmission |

If using WS2812 (addressable), a single LED can do all four states via color. Save addressable LED strips for the pager itself (customer-facing feedback).

---

## Problem 8: Buttons on the Hub

Buttons need a clear purpose. On a headless hub, consider:

| Button | Short press | Long press (3s) |
|---|---|---|
| **Button 1** | Cycle OLED display pages | Enter WiFi provisioning mode (AP or Web Serial) |
| **Button 2** | Manual "call next pager" (emergency fallback if dashboard is unreachable) | Factory reset (with confirmation LED sequence) |

Button 2 is the interesting one — it gives the hub standalone capability if both MQTT and Web Serial are down. The hub would call the next device in its local queue (ordered by assignment time). This is a resilience feature, not a primary flow.

---

## Recommended Additional Hardware

| Item | Why | Priority |
|---|---|---|
| **External antenna for 433 MHz** | The PCB trace antenna on cheap 433 MHz modules has terrible range. A coiled or whip antenna gets you 3-5x range | High |
| **Buzzer (passive, on hub)** | Audio confirmation of TX events. Admin hears the "beep" = pager was buzzed | Nice-to-have |
| **USB-C power (not micro-USB)** | Durability in a commercial environment | Medium |
| **Enclosure with wall-mount option** | Hub needs to be mounted near the counter, not loose on a desk | Medium |

For the **pager/receiver** side (not hub, but worth noting):
- Vibration motor (coin type, 3V)
- 433 MHz receiver module (superheterodyne, not superregenerative — better sensitivity) — for legacy receivers
- NRF24L01 module — for MCU-based receivers
- Single LED (red or RGB)
- CR2032 or small LiPo + charging circuit
- Power switch

---

## Backend Changes Needed

### 1. Device Table — Distinguish Hub from Receiver

Each store has one dedicated hub. Receivers are the individual pager devices. The `notifier_device` table must distinguish between these roles.

```sql
CREATE TYPE device_role AS ENUM ('HUB', 'RECEIVER');
CREATE TYPE device_type AS ENUM ('LEGACY_433', 'DYNAMIC_433', 'NRF24');

CREATE TABLE notifier_device (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    store_id        UUID NOT NULL REFERENCES store(id) ON DELETE CASCADE,
    role            device_role NOT NULL,
    device_type     device_type,                 -- NULL for HUB, required for RECEIVER
    device_index    INT,                          -- NULL for HUB, unique per store for RECEIVER
    name            VARCHAR(100),
    rf_code         TEXT,                          -- Tri-state code (433) or NRF24 address hex
    rf_protocol     INT,                           -- Protocol index (433 only)
    is_active       BOOLEAN DEFAULT TRUE,
    is_assigned     BOOLEAN DEFAULT FALSE,         -- RECEIVER only: currently bound to a ticket
    current_ticket  UUID,                          -- RECEIVER only: active ticket UUID
    battery_level   INT CHECK (battery_level >= 0 AND battery_level <= 100),
    last_ping       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,

    UNIQUE (store_id, device_index)
);

CREATE INDEX idx_notifier_device_store ON notifier_device(store_id);
CREATE INDEX idx_notifier_device_store_role ON notifier_device(store_id, role);
```

Key design points:
- **HUB row**: `role = HUB`, `device_type = NULL`, `device_index = NULL`. One per store. Used for heartbeat tracking (`last_ping`) and online status.
- **RECEIVER rows**: `role = RECEIVER`, `device_type` required, `device_index` unique per store.
- The unique constraint on `(store_id, device_index)` only applies to receivers (NULLs are not considered duplicates in Postgres).

### 2. New MQTT Topic and Event Types

```
Topic: notiguide/store/{storeId}/device

Events:
  DEVICE_ASSIGNED    -> hub should bind pager to ticket
  DEVICE_RELEASED    -> hub should unbind pager (ticket served/cancelled)
  DEVICE_TRIGGER     -> hub should transmit RF to pager (ticket called)
```

Payload:
```json
{
  "type": "DEVICE_TRIGGER",
  "storeId": "uuid",
  "ticketId": "uuid",
  "ticketNumber": "A-042",
  "deviceIndex": 3,
  "deviceType": "LEGACY_433",
  "counterId": "Counter-1",
  "timestamp": 1712500000000
}
```

The same JSON format is used for Web Serial commands — the hub parses identically regardless of transport.

### 3. New Endpoints

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/devices/admin/{storeId}` | List all devices for store (hub + receivers) |
| `POST` | `/api/devices/admin/{storeId}` | Register a new device (hub or receiver) |
| `PUT` | `/api/devices/admin/{storeId}/{deviceId}` | Update device (name, protocol, active) |
| `DELETE` | `/api/devices/admin/{storeId}/{deviceId}` | Remove device |
| `POST` | `/api/queue/admin/{storeId}/tickets/{ticketId}/assign-device` | Assign receiver to ticket |
| `POST` | `/api/queue/admin/{storeId}/tickets/{ticketId}/release-device` | Release receiver from ticket |
| `GET` | `/api/devices/admin/{storeId}/hub/status` | Hub online status (last heartbeat, connection method) |

### 4. QueueService Integration

When `callNext()` or `callSpecificTicket()` is invoked and the ticket has an assigned device:
1. Publish `DEVICE_TRIGGER` on the device MQTT topic (in addition to existing queue events)
2. Record `DEVICE_TRIGGERED` analytics event

When `serveTicket()` or `cancelTicket()` is invoked and the ticket has an assigned device:
1. Publish `DEVICE_RELEASED` on the device MQTT topic
2. Mark receiver's `is_assigned = false`, `current_ticket = null`

---

## Hub Firmware Architecture (Recommended)

```
app_main()
+-- wifi_init()          -> Connect to stored WiFi creds (NVS)
+-- mqtt_init()          -> Connect to broker, subscribe to device topic
+-- serial_init()        -> UART/CDC listener for Web Serial commands
+-- oled_init()          -> SSD1306 via I2C
+-- rf_433_init()        -> Existing rf_trans_init() for 433 MHz
+-- nrf24_init()         -> RadioLib NRF24L01 via SPI
+-- led_init()           -> Status LED(s)
|
+-- command_dispatch()   -> Shared JSON parser for both MQTT and serial input
|     +-- on DEVICE_TRIGGER  -> route to rf_433 or nrf24 based on deviceType
|     +-- on DEVICE_ASSIGNED -> update local assignment table
|     +-- on DEVICE_RELEASED -> clear local assignment
|
+-- Task: mqtt_handler   -> Receives MQTT messages, calls command_dispatch()
+-- Task: serial_handler -> Receives serial input, calls command_dispatch()
|
+-- Task: rf_transmitter -> Pulls from rf_trans_event_queue, calls rf_send() or nrf24_send()
|
+-- Task: oled_updater   -> Refreshes display every 500ms from shared state
|
+-- Task: heartbeat      -> Publishes hub status every 30s (MQTT when available)
|
+-- Task: button_handler -> Debounce + short/long press detection
```

This maps cleanly onto FreeRTOS tasks. The existing `rf_trans_event_queue` is already set up for inter-task communication. The `command_dispatch()` function is the key abstraction — it ensures MQTT and Web Serial produce identical behavior.

---

## What to Skip for v1

| Feature | Why skip |
|---|---|
| NRF24 bidirectional features (ack, battery report) | Fire-and-forget is sufficient initially; add feedback loop later |
| Dynamic code assignment for 433 MHz | Legacy fixed code table covers v1; dynamic codes for MCU receivers can follow |
| Multi-hub per store | One hub per store is the model; no need to overcomplicate |
| OTA firmware updates | Nice but not launch-blocking; can flash via USB for now |
| Pager-side firmware | Can be developed independently; only needs to match the RF codes the hub transmits |

---

## Implementation Order

| Phase | Scope | Depends on |
|---|---|---|
| **Phase 1** | Backend: notifier device domain (entity, repo, service, controller) with hub/receiver distinction | Schema revision |
| **Phase 2** | Backend: device MQTT topic + assign/release endpoints + QueueService integration | Phase 1 + existing MQTT |
| **Phase 3** | Hub firmware: WiFi + MQTT client + serial listener + shared `command_dispatch()` | Phase 2 (needs messages to consume) |
| **Phase 4** | Hub firmware: 433 MHz transmission on DEVICE_TRIGGER (wire up existing RF layer) | Phase 3 |
| **Phase 5** | Hub firmware: NRF24L01 transmission via RadioLib | Phase 3 (parallel with Phase 4) |
| **Phase 6** | Hub firmware: OLED display + LED status + button handling | Phase 3 (parallel with Phases 4-5) |
| **Phase 7** | Web admin: device management UI + "Assign Pager" button on ticket cards | Phase 2 |
| **Phase 8** | Receiver firmware: legacy 433 MHz (PT2272-based) + vibration/LED | Independent |
| **Phase 9** | Receiver firmware: MCU-based (dynamic 433 or NRF24) | Independent, after Phase 5 defines the NRF24 payload format |

Phases 4, 5, and 6 can run in parallel. Phases 8-9 are independent of everything except agreeing on the RF code/payload format.

---

## Open Questions

1. **ESP32-C3 vs ESP32-S3.** The sdkconfig currently targets S3. The C3 is cheaper and sufficient (WiFi + enough GPIO for SPI + I2C + 433 MHz TX). The S3 has more RAM, USB-OTG (better Web Serial support), and dual-core. Given Web Serial is a key backup path, S3's native USB-OTG may justify the cost.

2. **How many pagers per store?** This determines whether the legacy fixed code table is sufficient on its own or if dynamic codes are needed from day one. 20 is comfortable with legacy only. Beyond that, dynamic or NRF24 receivers are needed.

3. **Hub placement.** Is it on the counter (USB power, near admin for Web Serial) or elsewhere? Since Web Serial requires physical USB, counter placement makes the backup path more practical.

4. **Multiple frequency support.** Are you also considering 315 MHz (common in some regions) or is 433 MHz the only sub-GHz target?

5. **Receiver MCU choice.** For MCU-based receivers (non-PT2272), what chip? An ATtiny/STM32 with NRF24L01 is minimal. An ESP32-C3 with NRF24 gives you WiFi too (but overkill for a pager).
