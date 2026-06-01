# Transmitter Roster Delete Spec

Add the ability to delete a paired receiver from the transmitter hub via two channels: the OLED + button on the physical device, and the serial protocol over USB.

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Receiver notification | None | Hub forgets the receiver. Receiver keeps stale pairing data until manual factory reset. Avoids ESP-NOW coordination and unreachable-device edge cases. |
| OLED confirmation | Hold-to-confirm (2s) | Tactile safety gate without extra navigation steps. Mirrors the recovery-mode long-press pattern. |
| Serial confirmation | None | Web dashboard handles its own confirmation dialog. Serial command is trusted, matching existing commands (transmit, lifecycle, factory_reset). |
| Entry gesture | Triple-click on Screen 5 | Avoids hijacking existing gestures (single=cycle, double=pair mode, long=recovery). `BUTTON_MULTIPLE_CLICK` with `clicks=3` is supported by `iot_button` v4. |

## 1. OLED Delete Mode

### 1.1 Entry

Triple-click while on Screen 5 (Paired Receivers) enters Delete Mode.

- If the roster is empty (`roster.count == 0`), ignore the gesture (no-op).
- If a pair-mode overlay or other overlay is visible, ignore the gesture.
- On entry, the delete-mode overlay appears and the cursor starts on the first occupied slot.

### 1.2 Overlay Layout

The delete-mode overlay is a full-content-area overlay on `lv_layer_top()`, same pattern as `pair_mode_overlay_t` and `pair_confirm_overlay_t`.

```
  DELETE MODE
> 1: RX-Kitchen  433M
  3: RX-Counter  2.4G
  Hold 2s to delete
```

**Layout breakdown** (128x64 OLED, content area = 52px below 12px status bar):

| Row | Y offset | Content |
|-----|----------|---------|
| Header | +2 | `"DELETE MODE"` (left-aligned, same as Screen 5 header) |
| Slot rows (up to 3) | +14, +26, +38 | `"> "` cursor prefix on selected row. Format: `slot: name band` |
| Instruction | +44 | `"Hold 2s to delete"` |

The slot rows show up to 3 occupied entries at a time. The cursor prefix `"> "` appears only on the selected row. If there are more than 3 occupied slots, the visible window scrolls to keep the cursor in view.

Name is truncated to ~12 characters with LVGL `LV_LABEL_LONG_CLIP` to fit the 128px width.

### 1.3 Navigation

| Gesture | Action |
|---------|--------|
| Single-click | Move cursor to next occupied slot (wraps from last to first). If more than 3 occupied slots, scroll the visible window. |
| Double-click | Exit delete mode. Return to Screen 5. |
| Long-press (2s) | Confirm deletion of the currently highlighted device. See 1.4. |
| 30s inactivity | Auto-exit delete mode. Return to Screen 5. |

The long-press threshold for delete confirmation is 2 seconds, registered as a separate `BUTTON_LONG_PRESS_START` callback with `press_time = 2000`. This is distinct from the 5-second recovery-mode long-press. While in delete mode, the 5-second recovery gesture is suppressed (delete mode owns button interpretation).

### 1.4 Deletion Flow

When the admin holds the button for 2 seconds on a highlighted slot:

1. Call `roster_remove(roster, slot)` on the active roster.
2. Show brief feedback: replace the overlay content with `"Deleted: RX-name"` for 1.5 seconds.
3. After feedback:
   - If the roster still has occupied slots, return to delete mode with the cursor on the next occupied slot (or the first, if the deleted slot was last).
   - If the roster is now empty, exit delete mode and return to Screen 5.
4. Screen 5's receiver list refreshes automatically (it reads from the active roster on each render cycle).

### 1.5 Display API

New public functions in `display.h`:

```c
void display_enter_delete_mode(void);
```

Called from `controls.c` when triple-click is detected on Screen 5.

New overlay struct in `display_screens.h`:

```c
typedef struct {
    lv_obj_t *overlay;
    lv_obj_t *lbl_header;
    lv_obj_t *lbl_row[3];
    lv_obj_t *lbl_instruction;
} delete_mode_overlay_t;
```

New builder in `display_screens.c`:

```c
void screens_create_delete_mode_overlay(delete_mode_overlay_t *ov);
```

All gated behind `#if CONFIG_TRANSMITTER_PAIRING_ENABLED`.

### 1.6 Delete Mode State

Internal to `display.c`, a small state struct tracks delete mode:

```c
typedef struct {
    bool active;
    uint8_t cursor_slot;      // 1-based slot of highlighted entry
    uint8_t visible_offset;   // index into occupied-slot list for scroll window
    esp_timer_handle_t timeout_timer;  // 30s inactivity auto-exit
} delete_mode_state_t;
```

The inactivity timer resets on every button press. On expiry, delete mode exits cleanly.

## 2. Serial Protocol

### 2.1 `roster.list` Command

Lists all occupied slots in the roster. The web dashboard needs this to show which devices can be deleted.

**Request:**
```json
{"id": "req-1", "type": "roster.list"}
```

**Response:**
```json
{
  "id": "req-1",
  "type": "response",
  "ok": true,
  "payload": {
    "count": 2,
    "max": 32,
    "receivers": [
      {
        "slot": 1,
        "name": "RX-Kitchen",
        "band": "433M",
        "mac": "AA:BB:CC:DD:EE:FF",
        "paired_at_ms": 1717000000000
      },
      {
        "slot": 3,
        "name": "RX-Counter",
        "band": "2_4G",
        "mac": "11:22:33:44:55:66",
        "paired_at_ms": 1717100000000
      }
    ]
  }
}
```

`max` is `CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS`. Only occupied slots appear in the `receivers` array. MAC is formatted as uppercase colon-separated hex.

**Error (pairing disabled):**
```json
{"id": "req-1", "type": "response", "ok": false, "error": "pairing_disabled"}
```

### 2.2 `roster.unpair` Command

Removes a paired receiver by slot number.

**Request:**
```json
{"id": "req-2", "type": "roster.unpair", "payload": {"slot": 3}}
```

**Response (success):**
```json
{
  "id": "req-2",
  "type": "response",
  "ok": true,
  "payload": {
    "slot": 3,
    "removed_name": "RX-Counter"
  }
}
```

**Errors:**

| Error string | Condition |
|-------------|-----------|
| `"missing_payload"` | No payload object |
| `"missing_fields"` | `slot` field missing or not a number |
| `"invalid_slot"` | Slot < 1 or > `CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS` |
| `"slot_empty"` | Slot exists but is not occupied |
| `"pairing_disabled"` | `CONFIG_TRANSMITTER_PAIRING_ENABLED` is off |

**Behavior:**

1. Validate slot range and occupancy.
2. Capture the entry name before removal (for the response).
3. Call `roster_remove(roster, slot)` on the active roster.
4. If the OLED is currently in delete mode, refresh the overlay (cursor may need to move). If delete mode's highlighted slot was the one removed, advance cursor.
5. If Screen 5 is visible, it refreshes on the next render cycle.
6. Return success with the removed slot and name.

No device activation check is required. Unpairing should work regardless of `op_state` (an admin may need to clean up the roster on a suspended device).

### 2.3 `event.roster_changed` Event

A new serial event forwarded to the host when the roster changes from any source (OLED delete, serial unpair, MQTT `cmd/unpair`, or a new pairing).

```json
{
  "id": null,
  "type": "event.roster_changed",
  "payload": {
    "action": "removed",
    "slot": 3,
    "name": "RX-Counter",
    "count": 1,
    "source": "oled"
  }
}
```

| Field | Values |
|-------|--------|
| `action` | `"added"`, `"removed"`, `"renamed"` |
| `slot` | 1-based slot number affected |
| `name` | Current name (for added/renamed) or removed name (for removed) |
| `count` | Total occupied slots after the change |
| `source` | `"oled"`, `"serial"`, `"mqtt"` |

This event is emitted via a new transmitter event `TX_EVENT_ROSTER_CHANGED`, which the serial event handler picks up and forwards.

## 3. Button Gesture Summary (Updated)

### Global (all screens)

| Gesture | Action |
|---------|--------|
| Any press while asleep/dimmed | Wake display only |
| Single-click | Dismiss overlay (if visible), cycle to next screen |
| Double-click | Jump to dashboard |
| Long-press (5s) | Arm recovery mode |

### Screen 5 (Paired Receivers) overrides

| Gesture | Action |
|---------|--------|
| Double-click | Enter pair mode |
| Triple-click | Enter delete mode (if roster non-empty) |

### Delete Mode overrides (all gestures are consumed by the overlay)

| Gesture | Action |
|---------|--------|
| Single-click | Move cursor to next occupied slot |
| Double-click | Exit delete mode |
| Long-press (2s) | Delete highlighted device |

While delete mode is active, the 5-second recovery long-press is not reachable because the 2-second long-press fires first and is handled by delete mode.

## 4. New Transmitter Event

```c
TX_EVENT_ROSTER_CHANGED
```

Added to the `transmitter_event_id_t` enum in `events/transmitter_events.h`.

Event data struct:

```c
typedef struct {
    char action[16];    // "added", "removed", "renamed"
    uint8_t slot;
    char name[ROSTER_MAX_NAME_LEN + 1];
    uint8_t count;
    char source[8];     // "oled", "serial", "mqtt"
} roster_change_event_t;
```

This event is posted by each caller after the roster operation succeeds (`roster.c` is a pure storage module and does not post events itself):
- `roster_remove()` callers: OLED delete mode, serial `roster.unpair`, MQTT `cmd/unpair` handler
- `roster_add()` callers: pairing host after successful pair
- `roster_set_name()` callers: MQTT `cmd/label` handler

The serial event handler in `serial_protocol.c` listens for this event and forwards it as `event.roster_changed` to the USB host.

## 5. Backend (No Changes Required)

The existing backend already handles roster deletions automatically:

1. When the hub removes a device (via OLED or serial), `roster_remove()` bumps the sequence number and marks the roster as pending sync.
2. The hub's `roster_sync` module publishes the updated roster to `{prefix}/transmitter/hub/{publicId}/roster/update` via MQTT.
3. The backend's `RosterSyncListener.deleteUnpairedReceivers()` detects that a previously paired device is absent from the new roster and hard-deletes it from the database (`DELETE FROM device WHERE store_id = :storeId AND hub_slot IS NOT NULL AND hub_slot NOT IN (:activeSlots)`).
4. The backend publishes a signed ACK back to the hub, clearing the pending flag.

No new backend endpoints, entities, or MQTT handlers are needed. The bidirectional flow is already complete:
- **Hub → Backend:** Roster update exclusion (device absent from list → DB delete)
- **Backend → Hub:** `cmd/unpair` MQTT command (already exists, triggered by device decommission)

## 6. Frontend: Serial Protocol Types

Add `roster.list` and `roster.unpair` to the serial protocol infrastructure in `web/src/lib/serial/types.ts`.

### 6.1 New Type Definitions

```typescript
export interface RosterUnpairPayload {
  slot: number;
}

export interface RosterReceiver {
  slot: number;
  name: string;
  band: "433M" | "2_4G";
  mac: string;
  paired_at_ms: number;
}

export interface RosterListResult {
  count: number;
  max: number;
  receivers: RosterReceiver[];
}

export interface RosterUnpairResult {
  slot: number;
  removed_name: string;
}
```

### 6.2 SerialCommandMap Additions

```typescript
"roster.list": { payload: undefined; response: RosterListResult };
"roster.unpair": { payload: RosterUnpairPayload; response: RosterUnpairResult };
```

### 6.3 New Serial Event

Add `"event.roster_changed"` to the event listener list in `use-serial.ts`. The event payload matches the `roster_change_event_t` structure from Section 2.3.

## 7. Frontend: Hub Roster Panel

A new component `hub-roster-panel.tsx` in `web/src/features/device/` renders inside the existing `UsbControlPanel` when the hub is connected and not mismatched. It displays the hub's paired receivers and offers deletion.

### 7.1 Behavior

1. **Auto-fetch on mount:** When the panel renders (hub connected, not mismatched), send `roster.list` via serial to populate the receiver list.
2. **Display:** Show a card with a table/list of paired receivers. Each row shows: slot number, name, band, MAC (truncated), paired date (relative). Empty state: "No paired receivers" message.
3. **Delete action:** Each row has a delete button (trash icon). Clicking it opens a confirmation `AlertDialog` (same pattern as existing lifecycle/factory-reset confirmations in `UsbControlPanel`). On confirm, send `roster.unpair` with the slot number.
4. **Live updates:** Listen for `event.roster_changed` on the serial event bus. On any roster change event, re-fetch `roster.list` to refresh the list. This handles the case where a device is deleted via the OLED while the USB dashboard is open.
5. **Error handling:** If `roster.list` returns `pairing_disabled`, hide the panel entirely. If `roster.unpair` fails, show a toast with the error string.

### 7.2 Placement

The panel renders inside `UsbControlPanel`'s `CollapsibleContent`, after the status cards grid and before the action buttons row. It only renders when:
- `isConnected` is true
- `isMismatched` is false
- The initial `roster.list` call did not return `pairing_disabled`

### 7.3 i18n Keys

New keys under `devices.usb.roster`:

**English (`en.json`):**

| Key | Value |
|-----|-------|
| `devices.usb.roster.title` | `"Paired Receivers"` |
| `devices.usb.roster.empty` | `"No paired receivers on this hub."` |
| `devices.usb.roster.slot` | `"Slot"` |
| `devices.usb.roster.band` | `"Band"` |
| `devices.usb.roster.paired` | `"Paired"` |
| `devices.usb.roster.delete` | `"Remove"` |
| `devices.usb.roster.confirm_title` | `"Remove paired receiver?"` |
| `devices.usb.roster.confirm_desc` | `"This will unpair {name} (Slot {slot}) from the hub. The receiver will keep its pairing data until manually factory-reset."` |
| `devices.usb.roster.removed` | `"Removed {name} from slot {slot}"` |

**Vietnamese (`vi.json`):**

| Key | Value |
|-----|-------|
| `devices.usb.roster.title` | `"Thiết bị đã ghép nối"` |
| `devices.usb.roster.empty` | `"Hub chưa ghép nối thiết bị nào."` |
| `devices.usb.roster.slot` | `"Vị trí"` |
| `devices.usb.roster.band` | `"Băng tần"` |
| `devices.usb.roster.paired` | `"Đã ghép"` |
| `devices.usb.roster.delete` | `"Gỡ"` |
| `devices.usb.roster.confirm_title` | `"Gỡ thiết bị đã ghép nối?"` |
| `devices.usb.roster.confirm_desc` | `"Thao tác này sẽ gỡ {name} (Vị trí {slot}) khỏi hub. Thiết bị vẫn giữ dữ liệu ghép nối cho đến khi khôi phục cài đặt gốc."` |
| `devices.usb.roster.removed` | `"Đã gỡ {name} khỏi vị trí {slot}"` |

## 8. Scope Boundaries

**In scope:**
- Transmitter OLED delete mode (overlay, navigation, hold-to-confirm, auto-exit)
- Triple-click gesture registration on Screen 5
- Transmitter serial `roster.list` command handler
- Transmitter serial `roster.unpair` command handler
- Transmitter `event.roster_changed` serial event + `TX_EVENT_ROSTER_CHANGED` event
- Frontend serial protocol type additions (`roster.list`, `roster.unpair`)
- Frontend `hub-roster-panel.tsx` component in USB control panel
- Frontend i18n keys (English + Vietnamese)

**Out of scope:**
- ESP-NOW unpair notification to receiver
- Serial `roster.rename` command (exists via MQTT `cmd/label` already)
- Backend code changes (roster sync already handles deletions)
- Remote dashboard unpair action (decommission flow already handles this)
