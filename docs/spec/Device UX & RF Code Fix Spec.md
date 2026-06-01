# Device UX & RF Code Fix Spec

Four frontend UX improvements and one backend bug fix for the device management flow.

## 1. Pending Card — Static Label for PENDING_RF_CODE

**Problem:** `PENDING_RF_CODE` devices appear in the pending review card with no buttons and no explanation — a dead row. The admin approved the device already; it's now in a machine-to-machine handshake waiting for the firmware to ack the RF code. No admin action is possible.

**Solution:** When `device.status === "PENDING_RF_CODE"`, replace the approve/reject button area with a static text label (no spinner — a spinner would be distracting on a list page):

- i18n key `devices.pending.waitingForDevice`
  - EN: `"Waiting for device..."`
  - VI: `"Đang chờ thiết bị..."`
- Styled as `text-xs text-muted-foreground`

The row disappears from the pending card when a refresh (manual or auto-poll) confirms the device has transitioned out of `PENDING_RF_CODE`.

**Files:**
- `web/src/features/device/pending-review-card.tsx` — add `PENDING_RF_CODE` branch in the button area
- `web/src/messages/en.json` — add `devices.pending.waitingForDevice`
- `web/src/messages/vi.json` — add `devices.pending.waitingForDevice`

## 2. Reload Button + Auto-Polling on Device List Page

**Problem:** The device list only refreshes when the user navigates away or performs an action (approve/reject). State changes from MQTT-driven flows (activation, RF code ack) aren't reflected until the page is revisited.

**Solution — Manual reload:** Add a `RefreshCcw` outline button next to the page title (matching the style of the existing action buttons like "Cấp phát qua USB" and "Cấp mã"):

- Outline variant with icon + text label, placed right after the `<h1>` and before the hub cap badge
- On click: calls `fetchDevices()`
- While loading: icon gets `animate-spin`, button is `disabled`
- i18n label: `devices.reloadAction`
  - EN: `"Reload devices"`
  - VI: `"Tải lại thiết bị"`

**Solution — Auto-polling:** Complement the manual button with automatic background refresh:

- **Trigger:** poll only when the current device list contains at least one `PENDING_RF_CODE` device (not `PENDING` — those require admin action, polling adds no value)
- **Interval:** 15 seconds (the detail page polls at 5s for a single device; 15s is appropriate for a list endpoint)
- **Behavior:** calls `fetchDevices()` silently — no loading skeleton, no spinner on the table, no spin on the reload button icon
- **Stops automatically** when no `PENDING_RF_CODE` devices remain after a poll response
- **Cleanup:** clear the interval on unmount

**Files:**
- `web/src/app/[locale]/dashboard/devices/page.tsx` — add reload button + auto-poll effect
- `web/src/messages/en.json` — add `devices.reloadAction`
- `web/src/messages/vi.json` — add `devices.reloadAction`

## 3. Bigger Back Arrow on Device Detail Page

**Problem:** The back arrow (`ArrowLeft`) on the device detail page is `size-4` in a `size-8` container. Next to the `text-2xl font-bold` title, it looks undersized.

**Solution:** Bump the icon to `size-5` and the clickable container to `size-9`. Applies in both the loaded state and the not-found fallback state.

**Files:**
- `web/src/app/[locale]/dashboard/devices/[id]/page.tsx` — two `ArrowLeft` instances (loaded + not-found states)

## 4. Backend Bug — RF Code Rotate Rejects 433MHz Devices

**Problem:** `POST /api/devices/{id}/rf-code` throws `DeviceBadRequestEnvelopeException: width_out_of_range` for 433MHz receivers. The frontend calls `rotateRfCode(device.id)` without specifying `rfCodeBits`, so the backend receives `RotateRfCodeRequest(rfCodeBits = null)`. In `RfCodeService.resolveRequestedBits` (line 254), the `else` branch requires a non-null value for non-2.4GHz devices and throws when it's null.

Meanwhile, `autoIssue` (line 122) correctly falls back to `properties.rfCode.defaultBits433m` when no bits are specified.

**Root cause:** `resolveRequestedBits` has no default for 433MHz — it was written assuming the caller always sends `rfCodeBits`, but the frontend rotate flow doesn't.

**Fix:** In `resolveRequestedBits`, fall back to `properties.rfCode.defaultBits433m` when `requestedBits` is null for 433MHz devices, matching `autoIssue`'s behavior:

```kotlin
else -> requestedBits ?: properties.rfCode.defaultBits433m
```

**Files:**
- `backend/src/main/kotlin/com/thomas/notiguide/domain/device/service/RfCodeService.kt` — one-line change at line 254

## 5. Bigger Approve/Reject Buttons in Pending Card

**Problem:** The approve (check) and reject (X) buttons in the pending review card use `size="icon-sm"` (28px container, 16px icon). They're too small to comfortably tap/click, especially on touch screens.

**Solution:** Bump both buttons from `icon-sm` to `icon-lg` and their icons from `size-4` to `size-5`:

- Container: `size-7` (28px) → `size-9` (36px)
- Icon: `size-4` (16px) → `size-5` (20px)
- ~65% increase in tap target area

**Files:**
- `web/src/features/device/pending-review-card.tsx` — change `size="icon-sm"` to `size="icon-lg"` and `size-4` to `size-5` on both buttons
