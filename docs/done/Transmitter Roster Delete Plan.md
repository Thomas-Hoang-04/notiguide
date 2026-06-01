# Transmitter Roster Delete Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the ability to delete paired receivers from the transmitter hub via OLED + button (triple-click delete mode with hold-to-confirm) and USB serial protocol, with a web dashboard roster panel.

**Architecture:** Three layers — firmware (OLED delete mode + serial commands + new event), frontend serial types, and a new hub roster panel component in the USB control panel. The backend requires no changes (existing `RosterSyncListener.deleteUnpairedReceivers()` handles DB cleanup automatically via MQTT roster sync).

**Tech Stack:** ESP-IDF v6.0 (C, iot_button, LVGL 8.x, cJSON, esp_timer), Next.js 16 / React 19 / TypeScript 5 (shadcn/ui, next-intl)

**Spec:** `docs/spec/Transmitter Roster Delete Spec.md`

---

### Task 1: Add `TX_EVENT_ROSTER_CHANGED` to the Transmitter Event System

**Files:**
- Modify: `transmitter/main/events/transmitter_events.h`

This event is consumed by the serial event handler (Task 6) and the display (Task 4). All roster mutation callers post it after the operation succeeds.

- [ ] **Step 1: Add the event ID and payload struct**

In `transmitter/main/events/transmitter_events.h`, add `TX_EVENT_ROSTER_CHANGED` to the enum and define the event payload struct. Insert the new event ID before the closing brace of the enum, and the struct after `dispatch_summary_t`:

```c
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
    TX_EVENT_ROSTER_CHANGED,
} transmitter_event_id_t;
```

Add the payload struct after `dispatch_summary_t`:

```c
/**
 * @brief Payload for roster change events (add, remove, rename).
 */
typedef struct {
    char action[16];
    uint8_t slot;
    char name[97];
    uint8_t count;
    char source[8];
} roster_change_event_t;
```

The `name` field is `ROSTER_MAX_NAME_LEN + 1` (96 + 1 = 97). Use the literal value here to avoid a cross-header include of `roster.h`.

---

### Task 2: Add Delete-Mode Overlay Widget Builder

**Files:**
- Modify: `transmitter/main/display/display_screens.h`
- Modify: `transmitter/main/display/display_screens.c`

- [ ] **Step 1: Add the overlay struct and builder declaration**

In `display_screens.h`, inside the `#if CONFIG_TRANSMITTER_PAIRING_ENABLED` block (after `pair_confirm_overlay_t`), add:

```c
/**
 * @brief Widget handles for the delete-mode overlay.
 */
typedef struct {
    lv_obj_t *overlay;
    lv_obj_t *lbl_header;
    lv_obj_t *lbl_row[3];
    lv_obj_t *lbl_instruction;
} delete_mode_overlay_t;
```

Add the builder declaration in the same `#if` block, after `screens_create_pair_confirm_overlay`:

```c
/**
 * @brief Create the delete-mode overlay on the LVGL top layer.
 *
 * @param ov Output widget handle bundle.
 */
void screens_create_delete_mode_overlay(delete_mode_overlay_t *ov);
```

- [ ] **Step 2: Implement the overlay builder**

In `display_screens.c`, inside the `#if CONFIG_TRANSMITTER_PAIRING_ENABLED` block at the end (after `screens_create_pair_confirm_overlay`), add:

```c
void screens_create_delete_mode_overlay(delete_mode_overlay_t *ov)
{
    ov->overlay = lv_obj_create(lv_layer_top());
    lv_obj_remove_style_all(ov->overlay);
    lv_obj_set_size(ov->overlay, DISPLAY_WIDTH, CONTENT_HEIGHT);
    lv_obj_set_pos(ov->overlay, 0, CONTENT_Y_OFFSET);
    lv_obj_set_style_bg_color(ov->overlay, DISPLAY_BLACK, 0);
    lv_obj_set_style_bg_opa(ov->overlay, LV_OPA_COVER, 0);
    lv_obj_set_style_text_font(ov->overlay, &proggy_clean_12, 0);

    ov->lbl_header = lv_label_create(ov->overlay);
    lv_obj_set_style_text_color(ov->lbl_header, DISPLAY_WHITE, 0);
    lv_obj_set_pos(ov->lbl_header, 2, 2);
    lv_label_set_text(ov->lbl_header, "DELETE MODE");

    for (size_t i = 0; i < 3; ++i) {
        ov->lbl_row[i] = lv_label_create(ov->overlay);
        lv_obj_set_style_text_color(ov->lbl_row[i], DISPLAY_WHITE, 0);
        lv_obj_set_pos(ov->lbl_row[i], 2, (lv_coord_t)(14 + i * 12));
        lv_label_set_text(ov->lbl_row[i], "");
    }

    ov->lbl_instruction = lv_label_create(ov->overlay);
    lv_obj_set_style_text_color(ov->lbl_instruction, DISPLAY_WHITE, 0);
    lv_obj_set_pos(ov->lbl_instruction, 2, 44);
    lv_label_set_text(ov->lbl_instruction, "Hold 2s to delete");

    lv_obj_add_flag(ov->overlay, LV_OBJ_FLAG_HIDDEN);
}
```

---

### Task 3: Add `display_enter_delete_mode()` Public API

**Files:**
- Modify: `transmitter/main/display/display.h`

- [ ] **Step 1: Declare the new function**

In `display.h`, after `display_enter_pair_mode()`, add:

```c
/**
 * @brief Enter receiver delete mode to remove a paired device.
 *
 * This is a no-op when pairing support is disabled or when the roster
 * is empty.
 */
void display_enter_delete_mode(void);
```

---

### Task 4: Implement Delete Mode in `display.c`

**Files:**
- Modify: `transmitter/main/display/display.c`

This is the largest task. It adds the delete-mode state machine, overlay rendering, and cursor navigation inside the display module.

- [ ] **Step 1: Add static state and overlay variables**

In the `/* ---------- UI widget structs ---------- */` section of `display.c`, after `s_pair_confirm_overlay`, add (inside the `#if CONFIG_TRANSMITTER_PAIRING_ENABLED` block):

```c
static delete_mode_overlay_t s_delete_mode_overlay;
```

After the `s_pair_confirm_visible` cached state variable, add:

```c
static bool               s_delete_mode_active;
static bool               s_delete_confirming;
static uint8_t            s_delete_cursor_slot;
static uint8_t            s_delete_visible_offset;
static esp_timer_handle_t s_delete_timeout_timer;
static esp_timer_handle_t s_delete_feedback_timer;
static uint8_t            s_delete_removed_slot;
```

Add a forward declaration in the forward declarations section:

```c
#if CONFIG_TRANSMITTER_PAIRING_ENABLED
static void delete_mode_exit_locked(void);
static void delete_mode_feedback_timeout_cb(void *arg);
static void delete_mode_refresh_locked(void);
#endif
```

- [ ] **Step 2: Create the overlay during init**

In `create_operational_screens()`, after `screens_create_pair_confirm_overlay(&s_pair_confirm_overlay);`, add:

```c
    screens_create_delete_mode_overlay(&s_delete_mode_overlay);
```

- [ ] **Step 3: Implement helper to build the occupied-slot list**

Add a helper that collects occupied slot numbers into an array (used by cursor navigation and rendering). Place it before the delete mode functions:

```c
#if CONFIG_TRANSMITTER_PAIRING_ENABLED
static uint8_t delete_mode_collect_slots(uint8_t *out, uint8_t max)
{
    roster_t *roster = roster_get_active();
    if (roster == NULL) return 0;
    uint8_t n = 0;
    for (uint8_t i = 0; i < CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS && n < max; ++i) {
        if (roster->entries[i].occupied) {
            out[n++] = roster->entries[i].slot;
        }
    }
    return n;
}
#endif
```

- [ ] **Step 4: Implement the overlay refresh function**

This redraws the 3 visible rows based on cursor position and scroll offset:

```c
#if CONFIG_TRANSMITTER_PAIRING_ENABLED
static void delete_mode_refresh_locked(void)
{
    roster_t *roster = roster_get_active();
    if (roster == NULL) return;

    uint8_t slots[CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS];
    uint8_t total = delete_mode_collect_slots(slots, CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS);
    if (total == 0) {
        delete_mode_exit_locked();
        return;
    }

    /* find cursor index in the sorted slot list */
    uint8_t cursor_idx = 0;
    for (uint8_t i = 0; i < total; ++i) {
        if (slots[i] == s_delete_cursor_slot) { cursor_idx = i; break; }
    }

    /* adjust visible window so cursor is in view */
    if (cursor_idx < s_delete_visible_offset) {
        s_delete_visible_offset = cursor_idx;
    } else if (cursor_idx >= s_delete_visible_offset + 3) {
        s_delete_visible_offset = cursor_idx - 2;
    }

    char buf[48];
    for (uint8_t row = 0; row < 3; ++row) {
        uint8_t idx = s_delete_visible_offset + row;
        if (idx >= total) {
            lv_label_set_text(s_delete_mode_overlay.lbl_row[row], "");
            continue;
        }
        const roster_entry_t *entry = roster_find_slot(roster, slots[idx]);
        if (entry == NULL) {
            lv_label_set_text(s_delete_mode_overlay.lbl_row[row], "");
            continue;
        }
        const char *prefix = (slots[idx] == s_delete_cursor_slot) ? "> " : "  ";
        const char *band = entry->band == 0 ? "433" : "2.4";
        (void)snprintf(buf, sizeof(buf), "%s%u:%.12s %s", prefix, entry->slot, entry->name, band);
        lv_label_set_text(s_delete_mode_overlay.lbl_row[row], buf);
    }
}
#endif
```

- [ ] **Step 5: Implement the timeout callback and exit function**

The timeout fires from `esp_timer` (outside LVGL context), so it must acquire the LVGL lock:

```c
#if CONFIG_TRANSMITTER_PAIRING_ENABLED
static void delete_mode_timeout_cb(void *arg)
{
    (void)arg;
    if (!lvgl_port_lock(50)) return;
    delete_mode_exit_locked();
    update_active_screen();
    lvgl_port_unlock();
}

static void delete_mode_exit_locked(void)
{
    s_delete_mode_active = false;
    s_delete_confirming = false;
    if (s_delete_timeout_timer) {
        esp_timer_stop(s_delete_timeout_timer);
    }
    if (s_delete_feedback_timer) {
        esp_timer_stop(s_delete_feedback_timer);
    }
    if (s_delete_mode_overlay.overlay != NULL) {
        lv_obj_add_flag(s_delete_mode_overlay.overlay, LV_OBJ_FLAG_HIDDEN);
    }
}

static void delete_mode_feedback_timeout_cb(void *arg)
{
    (void)arg;
    if (!lvgl_port_lock(50)) return;

    s_delete_confirming = false;
    roster_t *roster = roster_get_active();
    if (!s_delete_mode_active || roster == NULL || roster->count == 0) {
        delete_mode_exit_locked();
        update_active_screen();
        lvgl_port_unlock();
        return;
    }

    uint8_t slots[CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS];
    uint8_t total = delete_mode_collect_slots(slots, CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS);
    if (total == 0) {
        delete_mode_exit_locked();
        update_active_screen();
        lvgl_port_unlock();
        return;
    }

    uint8_t next_slot = slots[0];
    for (uint8_t i = 0; i < total; ++i) {
        if (slots[i] >= s_delete_removed_slot) { next_slot = slots[i]; break; }
    }
    s_delete_cursor_slot = next_slot;
    s_delete_visible_offset = 0;

    lv_label_set_text(s_delete_mode_overlay.lbl_header, "DELETE MODE");
    lv_label_set_text(s_delete_mode_overlay.lbl_instruction, "Hold 2s to delete");
    delete_mode_refresh_locked();
    lvgl_port_unlock();
}
#endif
```

- [ ] **Step 6: Implement `display_enter_delete_mode()`**

```c
void display_enter_delete_mode(void)
{
#if CONFIG_TRANSMITTER_PAIRING_ENABLED
    if (s_splash_state != SPLASH_DONE) return;
    if (!lvgl_port_lock(50)) return;

    if (s_delete_mode_active || espnow_pair_host_is_active() || s_pair_confirm_visible) {
        lvgl_port_unlock();
        return;
    }

    roster_t *roster = roster_get_active();
    if (roster == NULL || roster->count == 0) {
        lvgl_port_unlock();
        return;
    }

    /* find first occupied slot */
    uint8_t slots[CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS];
    uint8_t total = delete_mode_collect_slots(slots, CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS);
    if (total == 0) {
        lvgl_port_unlock();
        return;
    }

    s_delete_mode_active = true;
    s_delete_cursor_slot = slots[0];
    s_delete_visible_offset = 0;

    /* create timeout timer on first entry */
    if (s_delete_timeout_timer == NULL) {
        esp_timer_create_args_t args = {
            .callback = delete_mode_timeout_cb,
            .name = "del_mode_timeout",
        };
        ESP_ERROR_CHECK(esp_timer_create(&args, &s_delete_timeout_timer));
    }
    if (s_delete_feedback_timer == NULL) {
        esp_timer_create_args_t args = {
            .callback = delete_mode_feedback_timeout_cb,
            .name = "del_mode_feedback",
        };
        ESP_ERROR_CHECK(esp_timer_create(&args, &s_delete_feedback_timer));
    }
    esp_timer_stop(s_delete_timeout_timer);
    esp_timer_stop(s_delete_feedback_timer);
    esp_timer_start_once(s_delete_timeout_timer, 30000000); /* 30s */

    delete_mode_refresh_locked();
    lv_obj_clear_flag(s_delete_mode_overlay.overlay, LV_OBJ_FLAG_HIDDEN);

    lvgl_port_unlock();
#endif
}
```

- [ ] **Step 7: Add cursor navigation and delete-confirm public helpers**

These are called from `controls.c` (Task 5) when button events fire during delete mode:

```c
#if CONFIG_TRANSMITTER_PAIRING_ENABLED
bool display_delete_mode_is_active(void)
{
    return s_delete_mode_active;
}

void display_delete_mode_next(void)
{
    if (!s_delete_mode_active || s_delete_confirming) return;
    if (!lvgl_port_lock(50)) return;

    /* reset inactivity timer */
    esp_timer_stop(s_delete_timeout_timer);
    esp_timer_start_once(s_delete_timeout_timer, 30000000);

    uint8_t slots[CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS];
    uint8_t total = delete_mode_collect_slots(slots, CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS);
    if (total == 0) {
        delete_mode_exit_locked();
        lvgl_port_unlock();
        return;
    }

    /* find current index and advance */
    uint8_t cursor_idx = 0;
    for (uint8_t i = 0; i < total; ++i) {
        if (slots[i] == s_delete_cursor_slot) { cursor_idx = i; break; }
    }
    cursor_idx = (cursor_idx + 1) % total;
    s_delete_cursor_slot = slots[cursor_idx];

    delete_mode_refresh_locked();
    lvgl_port_unlock();
}

void display_delete_mode_exit(void)
{
    if (!s_delete_mode_active || s_delete_confirming) return;
    if (!lvgl_port_lock(50)) return;
    delete_mode_exit_locked();
    update_active_screen();
    lvgl_port_unlock();
}

void display_delete_mode_confirm(void)
{
    if (!s_delete_mode_active || s_delete_confirming) return;
    if (!lvgl_port_lock(50)) return;

    /* reset inactivity timer */
    esp_timer_stop(s_delete_timeout_timer);
    esp_timer_start_once(s_delete_timeout_timer, 30000000);

    roster_t *roster = roster_get_active();
    if (roster == NULL) {
        delete_mode_exit_locked();
        lvgl_port_unlock();
        return;
    }

    const roster_entry_t *entry = roster_find_slot(roster, s_delete_cursor_slot);
    if (entry == NULL) {
        delete_mode_exit_locked();
        lvgl_port_unlock();
        return;
    }

    /* capture name for feedback and event before removal */
    char removed_name[ROSTER_MAX_NAME_LEN + 1];
    strlcpy(removed_name, entry->name, sizeof(removed_name));
    uint8_t removed_slot = s_delete_cursor_slot;

    esp_err_t err = roster_remove(roster, s_delete_cursor_slot);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "roster_remove slot %u failed: %s", s_delete_cursor_slot, esp_err_to_name(err));
        lvgl_port_unlock();
        return;
    }

    /* post roster changed event */
    roster_change_event_t evt = { 0 };
    strlcpy(evt.action, "removed", sizeof(evt.action));
    evt.slot = removed_slot;
    strlcpy(evt.name, removed_name, sizeof(evt.name));
    evt.count = roster->count;
    strlcpy(evt.source, "oled", sizeof(evt.source));
    esp_event_post(TRANSMITTER_EVENTS, TX_EVENT_ROSTER_CHANGED, &evt, sizeof(evt), 0);

    /* trigger roster sync to backend */
    (void)roster_sync_publish();

    /* show feedback briefly — set confirming flag to block button callbacks */
    s_delete_confirming = true;
    s_delete_removed_slot = removed_slot;

    char feedback[48];
    (void)snprintf(feedback, sizeof(feedback), "Deleted: %.20s", removed_name);
    lv_label_set_text(s_delete_mode_overlay.lbl_header, feedback);
    for (int i = 0; i < 3; ++i) {
        lv_label_set_text(s_delete_mode_overlay.lbl_row[i], "");
    }
    lv_label_set_text(s_delete_mode_overlay.lbl_instruction, "");
    esp_timer_stop(s_delete_feedback_timer);
    esp_timer_start_once(s_delete_feedback_timer, 1500000); /* 1.5s */

    lvgl_port_unlock();
}
#endif
```

Do not call `vTaskDelay()` or otherwise block from the button callback path. The iot_button callback invokes `display_delete_mode_confirm()` synchronously, so feedback/resume timing must be handled by `esp_timer`.

When `docs/planned/Cluster Roster Sync Plan.md` is implemented in the same branch, that plan's Task 12 adds `roster_cluster_publish()` immediately after this function's `roster_sync_publish()` call. Do not include the cluster call in a delete-only implementation because `roster_cluster.h` will not exist yet.

- [ ] **Step 8: Add unconditional declarations to `display.h`**

In `display.h`, declare these helpers unconditionally after `display_enter_delete_mode()`. `controls.c` calls them whenever OLED support is compiled in; pairing-disabled OLED builds must still compile and should receive no-op behavior from `display.c`.

```c
/**
 * @brief Check whether delete mode is currently active.
 *
 * @return true when delete mode overlay is visible.
 */
bool display_delete_mode_is_active(void);

/**
 * @brief Advance the delete-mode cursor to the next occupied slot.
 */
void display_delete_mode_next(void);

/**
 * @brief Exit delete mode and return to Screen 5.
 */
void display_delete_mode_exit(void);

/**
 * @brief Confirm deletion of the currently highlighted receiver.
 */
void display_delete_mode_confirm(void);
```

In `display.c`, provide no-op/false implementations when pairing is disabled, matching the existing `display_enter_pair_mode()` pattern:

```c
#if !CONFIG_TRANSMITTER_PAIRING_ENABLED
bool display_delete_mode_is_active(void) { return false; }
void display_delete_mode_next(void) {}
void display_delete_mode_exit(void) {}
void display_delete_mode_confirm(void) {}
#endif
```

- [ ] **Step 9: Handle `TX_EVENT_ROSTER_CHANGED` in display and LED event handlers**

In `display_event_handler()` in `display.c`, add a new case before the closing `}` of the switch (handles external roster changes refreshing the delete-mode overlay):

```c
#if CONFIG_TRANSMITTER_PAIRING_ENABLED
    case TX_EVENT_ROSTER_CHANGED:
        if (s_delete_mode_active && !s_delete_confirming) {
            const roster_change_event_t *rc = (const roster_change_event_t *)data;
            if (rc != NULL && strcmp(rc->action, "removed") == 0) {
                /* if the removed slot was our cursor, advance */
                if (rc->slot == s_delete_cursor_slot) {
                    uint8_t slots[CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS];
                    uint8_t total = delete_mode_collect_slots(slots, CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS);
                    if (total == 0) {
                        delete_mode_exit_locked();
                    } else {
                        uint8_t next = slots[0];
                        for (uint8_t i = 0; i < total; ++i) {
                            if (slots[i] >= rc->slot) { next = slots[i]; break; }
                        }
                        s_delete_cursor_slot = next;
                    }
                }
                delete_mode_refresh_locked();
            }
        }
        update_active_screen();
        break;
#else
    case TX_EVENT_ROSTER_CHANGED:
        break;
#endif
```

In `led_event_handler()` in `controls.c`, add before the `default:` case (prevents compiler warning for unhandled enum):

```c
    case TX_EVENT_ROSTER_CHANGED:
        break;
```

- [ ] **Step 10: Hide delete-mode overlay in `dismiss_overlay_locked()`**

In `display.c`, in the `dismiss_overlay_locked()` function, add after `hide_pair_overlays_locked();`:

```c
    if (s_delete_mode_active) {
        delete_mode_exit_locked();
    }
```

- [ ] **Step 11: Add stubs when OLED is disabled**

In `controls.c`, in the `#else` block for `CONFIG_TRANSMITTER_OLED_ENABLED` (near the top of the file, around line 13-24), add stubs alongside the existing ones:

```c
static inline bool display_delete_mode_is_active(void) { return false; }
static inline void display_enter_delete_mode(void) {}
static inline void display_delete_mode_next(void) {}
static inline void display_delete_mode_exit(void) {}
static inline void display_delete_mode_confirm(void) {}
```

---

### Task 5: Wire Button Callbacks in `controls.c`

**Files:**
- Modify: `transmitter/main/controls/controls.c`

- [ ] **Step 1: Add the triple-click callback**

After the `on_double_click` function:

```c
static void on_triple_click(void *arg, void *usr_data)
{
    (void)arg;
    (void)usr_data;

    bool was_inactive = display_is_asleep() || display_is_dimmed();
    display_wake();

    if (was_inactive) return;

#if CONFIG_TRANSMITTER_PAIRING_ENABLED
    if (display_current_screen() == SCREEN_RECEIVERS) {
        display_enter_delete_mode();
    }
#endif
}
```

- [ ] **Step 2: Add the delete-mode long-press callback**

After `on_triple_click`:

```c
static void on_delete_mode_long_press(void *arg, void *usr_data)
{
    (void)arg;
    (void)usr_data;

    if (display_delete_mode_is_active()) {
        display_delete_mode_confirm();
    }
}
```

`display_delete_mode_confirm()` must return immediately after scheduling the feedback timer. Do not add `vTaskDelay()` or any other blocking wait in this callback chain; Espressif `iot_button` callbacks run in the button service context and must stay non-blocking.

- [ ] **Step 3: Route existing callbacks through delete mode**

Modify `on_single_click` to check for delete mode before cycling screens. Replace the body after the `if (was_inactive) return;` line:

```c
    if (display_delete_mode_is_active()) {
        display_delete_mode_next();
        return;
    }

    display_dismiss_overlay_if_visible();
    display_cycle_screen();
```

Modify `on_double_click` similarly. Replace the body after `if (was_inactive) return;`:

```c
    if (display_delete_mode_is_active()) {
        display_delete_mode_exit();
        return;
    }

#if CONFIG_TRANSMITTER_PAIRING_ENABLED
    if (display_current_screen() == SCREEN_RECEIVERS) {
        display_enter_pair_mode();
        return;
    }
#endif

    display_dismiss_overlay_if_visible();
    display_jump_to_dashboard();
```

Modify `on_long_press_start` to skip recovery when delete mode is active. Add at the start, after `if (was_inactive) return;`:

```c
    if (display_delete_mode_is_active()) {
        return;
    }
```

- [ ] **Step 4: Register new button callbacks in `button_init()`**

In `button_init()`, after the existing `BUTTON_LONG_PRESS_UP` registration, add:

```c
    button_event_args_t triple_args = { .multiple_clicks.clicks = 3 };
    ESP_ERROR_CHECK(iot_button_register_cb(btn, BUTTON_MULTIPLE_CLICK, &triple_args, on_triple_click, NULL));

    button_event_args_t del_long_args = { .long_press.press_time = 2000 };
    ESP_ERROR_CHECK(iot_button_register_cb(btn, BUTTON_LONG_PRESS_START, &del_long_args, on_delete_mode_long_press, NULL));
```

---

### Task 6: Add Serial `roster.list` and `roster.unpair` Commands

**Files:**
- Modify: `transmitter/main/serial/serial_protocol.c`

- [ ] **Step 1: Add the `roster.list` handler**

Add at the top of `serial_protocol.c`, inside the existing includes, add the roster include (only when pairing is enabled):

```c
#if CONFIG_TRANSMITTER_PAIRING_ENABLED
#include "events/transmitter_events.h"
#include "pair/roster.h"
#include "pair/roster_sync.h"

#include "esp_event.h"
#endif
```

Add the handler function before `serial_handle_line()`:

```c
#if CONFIG_TRANSMITTER_PAIRING_ENABLED
static void handle_roster_list(const char *id)
{
    roster_t *roster = roster_get_active();
    cJSON *payload = cJSON_CreateObject();
    cJSON_AddNumberToObject(payload, "count", roster != NULL ? roster->count : 0);
    cJSON_AddNumberToObject(payload, "max", CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS);

    cJSON *receivers = cJSON_AddArrayToObject(payload, "receivers");

    if (roster != NULL) {
        for (uint8_t i = 0; i < CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS; ++i) {
            const roster_entry_t *entry = &roster->entries[i];
            if (!entry->occupied) continue;

            cJSON *rx = cJSON_CreateObject();
            cJSON_AddNumberToObject(rx, "slot", entry->slot);
            cJSON_AddStringToObject(rx, "name", entry->name);
            cJSON_AddStringToObject(rx, "band", entry->band == 0 ? "433M" : "2_4G");

            char mac_str[18];
            snprintf(mac_str, sizeof(mac_str), "%02X:%02X:%02X:%02X:%02X:%02X",
                     entry->mac[0], entry->mac[1], entry->mac[2],
                     entry->mac[3], entry->mac[4], entry->mac[5]);
            cJSON_AddStringToObject(rx, "mac", mac_str);
            cJSON_AddNumberToObject(rx, "paired_at_ms", (double)entry->paired_at_ms);
            cJSON_AddItemToArray(receivers, rx);
        }
    }

    serial_send_response(id, true, payload, NULL);
}
#endif
```

- [ ] **Step 2: Add the `roster.unpair` handler**

Add after `handle_roster_list`:

```c
#if CONFIG_TRANSMITTER_PAIRING_ENABLED
static void handle_roster_unpair(const char *id, const cJSON *payload)
{
    if (payload == NULL) {
        serial_send_error(id, "missing_payload");
        return;
    }

    const cJSON *slot_item = cJSON_GetObjectItem(payload, "slot");
    if (!cJSON_IsNumber(slot_item)) {
        serial_send_error(id, "missing_fields");
        return;
    }

    int slot_val = (int)slot_item->valuedouble;
    if (slot_val < 1 || slot_val > CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS) {
        serial_send_error(id, "invalid_slot");
        return;
    }

    roster_t *roster = roster_get_active();
    if (roster == NULL) {
        serial_send_error(id, "pairing_disabled");
        return;
    }

    uint8_t slot = (uint8_t)slot_val;
    const roster_entry_t *entry = roster_find_slot(roster, slot);
    if (entry == NULL) {
        serial_send_error(id, "slot_empty");
        return;
    }

    /* capture name before removal */
    char removed_name[ROSTER_MAX_NAME_LEN + 1];
    strlcpy(removed_name, entry->name, sizeof(removed_name));

    esp_err_t err = roster_remove(roster, slot);
    if (err != ESP_OK) {
        serial_send_error(id, "remove_failed");
        return;
    }

    /* post roster changed event */
    roster_change_event_t evt = { 0 };
    strlcpy(evt.action, "removed", sizeof(evt.action));
    evt.slot = slot;
    strlcpy(evt.name, removed_name, sizeof(evt.name));
    evt.count = roster->count;
    strlcpy(evt.source, "serial", sizeof(evt.source));
    esp_event_post(TRANSMITTER_EVENTS, TX_EVENT_ROSTER_CHANGED, &evt, sizeof(evt), 0);

    (void)roster_sync_publish();

    cJSON *resp = cJSON_CreateObject();
    cJSON_AddNumberToObject(resp, "slot", slot);
    cJSON_AddStringToObject(resp, "removed_name", removed_name);
    serial_send_response(id, true, resp, NULL);
}
#endif
```

When `docs/planned/Cluster Roster Sync Plan.md` is implemented in the same branch, that plan's Task 12 adds `#include "pair/roster_cluster.h"` and `roster_cluster_publish()` to this successful removal path. Do not include the cluster call in a delete-only implementation because `roster_cluster.h` will not exist yet.

- [ ] **Step 3: Wire commands into `serial_handle_line()`**

In the `serial_handle_line()` function's command dispatch chain, before the `} else {` unknown command fallback, add:

```c
#if CONFIG_TRANSMITTER_PAIRING_ENABLED
    } else if (strcmp(type, "roster.list") == 0) {
        serial_mark_session_active();
        handle_roster_list(id);
    } else if (strcmp(type, "roster.unpair") == 0) {
        serial_mark_session_active();
        handle_roster_unpair(id, payload);
#else
    } else if (strcmp(type, "roster.list") == 0 ||
               strcmp(type, "roster.unpair") == 0) {
        serial_mark_session_active();
        serial_send_error(id, "pairing_disabled");
#endif
```

- [ ] **Step 4: Forward `TX_EVENT_ROSTER_CHANGED` in the serial event handler**

In `serial_event_handler()`, add a new case before the `default:` case:

```c
    case TX_EVENT_ROSTER_CHANGED: {
        if (data == NULL) break;
        const roster_change_event_t *rc = (const roster_change_event_t *)data;
        cJSON *p = cJSON_CreateObject();
        cJSON_AddStringToObject(p, "action", rc->action);
        cJSON_AddNumberToObject(p, "slot", rc->slot);
        cJSON_AddStringToObject(p, "name", rc->name);
        cJSON_AddNumberToObject(p, "count", rc->count);
        cJSON_AddStringToObject(p, "source", rc->source);
        serial_send_event("event.roster_changed", p);
        break;
    }
```

---

### Task 6A: Emit Roster Change Events from Existing Mutation Sites

**Files:**
- Modify: `transmitter/main/display/display.c`
- Modify: `transmitter/main/pair/roster_sync.c`

The event added in Task 1 must be emitted by every successful roster mutation path required by the spec: OLED delete, serial unpair, MQTT unpair, new pairing, and MQTT label. Tasks 4 and 6 cover OLED delete and serial unpair. This task covers the existing pairing and MQTT mutation paths.

- [ ] **Step 1: Post `added` after successful local pairing**

In `pair_complete_cb()` in `display.c`, after the existing successful `roster_add()` flow and after `roster_sync_publish()`, post:

```c
        const roster_entry_t *added = roster_find_slot(roster_get_active(), slot);
        roster_change_event_t evt = { 0 };
        strlcpy(evt.action, "added", sizeof(evt.action));
        evt.slot = slot;
        if (added != NULL) {
            strlcpy(evt.name, added->name, sizeof(evt.name));
        }
        evt.count = roster_get_active() != NULL ? roster_get_active()->count : 0;
        strlcpy(evt.source, "oled", sizeof(evt.source));
        esp_event_post(TRANSMITTER_EVENTS, TX_EVENT_ROSTER_CHANGED, &evt, sizeof(evt), 0);
```

- [ ] **Step 2: Add a local event helper to `roster_sync.c`**

In `roster_sync.c`, add the event include and a small helper near the other static helpers:

```c
#include "events/transmitter_events.h"
#include "esp_event.h"

static void post_roster_changed(const char *action, uint8_t slot, const char *name, const char *source)
{
    roster_change_event_t evt = { 0 };
    strlcpy(evt.action, action, sizeof(evt.action));
    evt.slot = slot;
    if (name != NULL) {
        strlcpy(evt.name, name, sizeof(evt.name));
    }
    evt.count = s_roster != NULL ? s_roster->count : 0;
    strlcpy(evt.source, source, sizeof(evt.source));
    esp_event_post(TRANSMITTER_EVENTS, TX_EVENT_ROSTER_CHANGED, &evt, sizeof(evt), 0);
}
```

- [ ] **Step 3: Post `removed` from MQTT unpair**

In `roster_sync_handle_unpair()`, capture the receiver name before `roster_remove()`, then post the event after successful removal and before publishing the roster:

```c
    char removed_name[ROSTER_MAX_NAME_LEN + 1] = { 0 };
    const roster_entry_t *entry = roster_find_slot(s_roster, slot);
    if (entry != NULL) {
        strlcpy(removed_name, entry->name, sizeof(removed_name));
    }

    esp_err_t err = roster_remove(s_roster, slot);
    if (err == ESP_OK) {
        post_roster_changed("removed", slot, removed_name, "mqtt");
        (void)roster_sync_publish();
    }
```

- [ ] **Step 4: Post `renamed` from MQTT label updates**

In `roster_sync_handle_label()`, after `roster_set_name()` succeeds, post the event and publish the updated roster:

```c
    esp_err_t err = roster_set_name(s_roster, slot, label);
    if (err == ESP_OK) {
        post_roster_changed("renamed", slot, label, "mqtt");
        (void)roster_sync_publish();
    }
```

If Cluster Roster Sync is implemented in the same branch, the cluster plan's Task 12 adds `roster_cluster_publish()` to the same successful mutation blocks.

---

### Task 7: Add Frontend Serial Protocol Types

**Files:**
- Modify: `web/src/lib/serial/types.ts`

- [ ] **Step 1: Add payload and result interfaces**

After the `LifecycleResult` interface, add:

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

- [ ] **Step 2: Extend `SerialCommandMap`**

In the `SerialCommandMap` interface, before the closing brace, add:

```typescript
  "roster.list": { payload: undefined; response: RosterListResult };
  "roster.unpair": { payload: RosterUnpairPayload; response: RosterUnpairResult };
```

---

### Task 8: Add `event.roster_changed` to Serial Event Listeners

**Files:**
- Modify: `web/src/lib/serial/use-serial.ts`
- Modify: `web/src/features/device/usb-control-panel.tsx`

- [ ] **Step 1: Add event to `use-serial.ts`**

In the `eventTypes` array inside the `useEffect` that registers `handleEvent` listeners (around line 269-276), add `"event.roster_changed"`:

```typescript
    const eventTypes = [
      "event.wifi_connected",
      "event.wifi_disconnected",
      "event.mqtt_connected",
      "event.mqtt_disconnected",
      "event.activated",
      "event.lifecycle_changed",
      "event.roster_changed",
    ];
```

- [ ] **Step 2: Add event to `usb-control-panel.tsx` event log**

In the `useEffect` that sets up `eventLog` listeners (around line 114-124), add `"event.roster_changed"` to the `eventTypes` array:

```typescript
    const eventTypes = [
      "event.wifi_connected",
      "event.wifi_disconnected",
      "event.mqtt_connected",
      "event.mqtt_disconnected",
      "event.activated",
      "event.lifecycle_changed",
      "event.dispatch_ok",
      "event.dispatch_rejected",
      "event.roster_changed",
    ];
```

---

### Task 9: Add i18n Keys

**Files:**
- Modify: `web/src/messages/en.json`
- Modify: `web/src/messages/vi.json`

- [ ] **Step 1: Add English keys**

In `en.json`, inside the `"devices"` → `"usb"` section, add a `"roster"` object:

```json
"roster": {
  "title": "Paired Receivers",
  "empty": "No paired receivers on this hub.",
  "slot": "Slot",
  "band": "Band",
  "paired": "Paired",
  "delete": "Remove",
  "confirm_title": "Remove paired receiver?",
  "confirm_desc": "This will unpair {name} (Slot {slot}) from the hub. The receiver will keep its pairing data until manually factory-reset.",
  "removed": "Removed {name} from slot {slot}"
}
```

- [ ] **Step 2: Add Vietnamese keys**

In `vi.json`, inside the `"devices"` → `"usb"` section, add:

```json
"roster": {
  "title": "Thiết bị đã ghép nối",
  "empty": "Hub chưa ghép nối thiết bị nào.",
  "slot": "Vị trí",
  "band": "Băng tần",
  "paired": "Đã ghép",
  "delete": "Gỡ",
  "confirm_title": "Gỡ thiết bị đã ghép nối?",
  "confirm_desc": "Thao tác này sẽ gỡ {name} (Vị trí {slot}) khỏi hub. Thiết bị vẫn giữ dữ liệu ghép nối cho đến khi khôi phục cài đặt gốc.",
  "removed": "Đã gỡ {name} khỏi vị trí {slot}"
}
```

---

### Task 10: Create `hub-roster-panel.tsx` Component

**Files:**
- Create: `web/src/features/device/hub-roster-panel.tsx`

- [ ] **Step 1: Create the component**

```tsx
"use client";

import { Loader2, Radio, Trash2 } from "lucide-react";
import { useTranslations } from "next-intl";
import { useCallback, useEffect, useState } from "react";
import { toast } from "sonner";
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
} from "@/components/ui/alert-dialog";
import { Button } from "@/components/ui/button";
import { Card } from "@/components/ui/card";
import type { UseSerialReturn } from "@/lib/serial/use-serial";
import type { RosterReceiver } from "@/lib/serial/types";

interface HubRosterPanelProps {
  serial: UseSerialReturn;
}

export function HubRosterPanel({ serial }: HubRosterPanelProps) {
  const tRoster = useTranslations("devices.usb.roster");
  const tCommon = useTranslations("common");

  const { sendCommand, events } = serial;

  const [receivers, setReceivers] = useState<RosterReceiver[]>([]);
  const [maxSlots, setMaxSlots] = useState(0);
  const [loading, setLoading] = useState(true);
  const [pairingDisabled, setPairingDisabled] = useState(false);
  const [deleteTarget, setDeleteTarget] = useState<RosterReceiver | null>(null);
  const [deleteLoading, setDeleteLoading] = useState(false);

  const fetchRoster = useCallback(async () => {
    try {
      const result = await sendCommand("roster.list");
      setReceivers(result.receivers);
      setMaxSlots(result.max);
      setPairingDisabled(false);
    } catch (err) {
      if (err instanceof Error && err.message.includes("pairing_disabled")) {
        setPairingDisabled(true);
      }
    } finally {
      setLoading(false);
    }
  }, [sendCommand]);

  useEffect(() => {
    void fetchRoster();
  }, [fetchRoster]);

  useEffect(() => {
    const handler = () => {
      void fetchRoster();
    };
    events.addEventListener("event.roster_changed", handler);
    return () => events.removeEventListener("event.roster_changed", handler);
  }, [events, fetchRoster]);

  async function handleDelete() {
    if (!deleteTarget) return;
    setDeleteLoading(true);
    try {
      const result = await sendCommand("roster.unpair", {
        slot: deleteTarget.slot,
      });
      toast.success(
        tRoster("removed", {
          name: result.removed_name,
          slot: String(result.slot),
        }),
      );
      setDeleteTarget(null);
      await fetchRoster();
    } catch (err) {
      toast.error(
        err instanceof Error ? err.message : "Failed to remove receiver",
      );
    } finally {
      setDeleteLoading(false);
    }
  }

  function formatPairedAt(value: number) {
    if (!Number.isFinite(value) || value <= 0) return "—";
    return new Intl.DateTimeFormat(undefined, {
      dateStyle: "medium",
      timeStyle: "short",
    }).format(new Date(value));
  }

  if (pairingDisabled || loading) return null;

  return (
    <>
      <Card className="rounded-lg p-3">
        <div className="mb-2 flex items-center gap-2">
          <Radio aria-hidden="true" className="size-4 text-primary" />
          <span className="text-sm font-medium">
            {tRoster("title")} ({receivers.length}/{maxSlots})
          </span>
        </div>

        {receivers.length === 0 ? (
          <p className="text-xs text-muted-foreground">{tRoster("empty")}</p>
        ) : (
          <div className="space-y-1.5">
            {receivers.map((rx) => (
              <div
                key={rx.slot}
                className="flex items-center justify-between rounded-md border border-border/60 px-2.5 py-1.5 text-xs"
              >
                <div className="flex items-center gap-3">
                  <span className="font-medium text-muted-foreground">
                    #{rx.slot}
                  </span>
                  <span className="font-medium">{rx.name || "—"}</span>
                  <span className="text-muted-foreground">{rx.band}</span>
                  <span className="text-muted-foreground">
                    {rx.mac.slice(0, 8)}...
                  </span>
                  <span className="text-muted-foreground">
                    {tRoster("paired")}: {formatPairedAt(rx.paired_at_ms)}
                  </span>
                </div>
                <Button
                  variant="ghost"
                  size="icon"
                  className="size-6 text-destructive hover:text-destructive"
                  onClick={() => setDeleteTarget(rx)}
                >
                  <Trash2 aria-hidden="true" className="size-3.5" />
                  <span className="sr-only">{tRoster("delete")}</span>
                </Button>
              </div>
            ))}
          </div>
        )}
      </Card>

      <AlertDialog
        open={deleteTarget !== null}
        onOpenChange={(open) => !open && setDeleteTarget(null)}
      >
        <AlertDialogContent>
          <AlertDialogHeader>
            <AlertDialogTitle>{tRoster("confirm_title")}</AlertDialogTitle>
            <AlertDialogDescription>
              {deleteTarget &&
                tRoster("confirm_desc", {
                  name: deleteTarget.name || `Slot ${deleteTarget.slot}`,
                  slot: String(deleteTarget.slot),
                })}
            </AlertDialogDescription>
          </AlertDialogHeader>
          <AlertDialogFooter>
            <AlertDialogCancel disabled={deleteLoading}>
              {tCommon("cancel")}
            </AlertDialogCancel>
            <AlertDialogAction
              disabled={deleteLoading}
              className="bg-destructive text-destructive-foreground hover:bg-destructive/90"
              onClick={() => void handleDelete()}
            >
              {deleteLoading && (
                <Loader2
                  aria-hidden="true"
                  className="mr-2 size-4 animate-spin"
                />
              )}
              {tRoster("delete")}
            </AlertDialogAction>
          </AlertDialogFooter>
        </AlertDialogContent>
      </AlertDialog>
    </>
  );
}
```

---

### Task 11: Integrate Roster Panel into USB Control Panel

**Files:**
- Modify: `web/src/features/device/usb-control-panel.tsx`

- [ ] **Step 1: Import the component**

Add at the top of the file, after the existing feature imports:

```typescript
import { HubRosterPanel } from "./hub-roster-panel";
```

- [ ] **Step 2: Render the panel**

In the JSX, inside the `<CollapsibleContent>` → `<div className="mt-4 space-y-4">`, after the status cards grid block (`{isConnected && deviceState && (<div className="grid grid-cols-2 gap-3 ...">...`), and before the action buttons row (`{isConnected && (<div className="flex flex-wrap ...">...`), add:

```tsx
            {isConnected && !isMismatched && (
              <HubRosterPanel serial={serial} />
            )}
```

---

### Task 12: Update `docs/CHANGELOGS.md`

**Files:**
- Modify: `docs/CHANGELOGS.md`

- [ ] **Step 1: Add changelog entry**

Add a new entry at the top of the changelog:

```markdown
## Transmitter Roster Delete

**Spec:** `docs/spec/Transmitter Roster Delete Spec.md`

### Transmitter Firmware
- Added `TX_EVENT_ROSTER_CHANGED` event with `roster_change_event_t` payload
- Added delete-mode overlay (OLED): triple-click on Screen 5 to enter, single-click to navigate, hold 2s to confirm delete, double-click to exit, 30s auto-timeout
- Used `esp_timer` for post-delete feedback/resume; no blocking calls run from the `iot_button` callback path
- Added `display_enter_delete_mode()`, `display_delete_mode_next()`, `display_delete_mode_exit()`, `display_delete_mode_confirm()`, `display_delete_mode_is_active()` public APIs
- Declared delete-mode public helpers unconditionally and added pairing-disabled no-op implementations for OLED-enabled / pairing-disabled builds
- Added `delete_mode_overlay_t` widget struct and `screens_create_delete_mode_overlay()` builder
- Wired triple-click (`BUTTON_MULTIPLE_CLICK` clicks=3) and 2s long-press callbacks in `controls.c`
- Existing single/double/long-press callbacks now check `display_delete_mode_is_active()` and route accordingly
- Added serial `roster.list` command — returns all occupied slots with name, band, MAC, paired_at_ms
- Added serial `roster.unpair` command — removes a receiver by slot number, returns removed name
- Added `event.roster_changed` serial event forwarding for add/remove/rename from any source
- Posted `TX_EVENT_ROSTER_CHANGED` from OLED delete, serial unpair, MQTT unpair, MQTT label, and new pairing
- If Cluster Roster Sync is implemented in the same branch, OLED delete and serial unpair also publish the encrypted retained cluster roster after local mutation

### Web Frontend
- Added `RosterReceiver`, `RosterListResult`, `RosterUnpairPayload`, `RosterUnpairResult` types to serial types
- Added `roster.list` and `roster.unpair` to `SerialCommandMap`
- Added `event.roster_changed` to serial event listeners in `use-serial.ts` and `usb-control-panel.tsx`
- Created `hub-roster-panel.tsx` — displays paired receivers in USB control panel with delete buttons and confirmation dialog
- Added `devices.usb.roster.*` i18n keys (English + Vietnamese)

### Backend
- No changes required — existing `RosterSyncListener.deleteUnpairedReceivers()` handles DB cleanup via MQTT roster sync
```
