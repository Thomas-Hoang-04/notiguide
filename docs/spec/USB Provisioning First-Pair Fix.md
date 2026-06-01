# USB Provisioning First-Pair Fix

## Problem

When an ESP32-C3 transmitter hub has never been paired with Chrome, the provisioning dialog requires two connect attempts:

1. User clicks "Connect USB" → Chrome picker opens → user selects device → spinner appears briefly → dialog resets to "Connect USB"
2. User clicks "Connect USB" again → selects already-paired device → provision form appears

The root cause is a race condition during first-time USB pairing. The ESP32-C3's CDC-ACM interface does a USB enumeration reset after initial pairing, which causes `port.open()` or the stream setup to reject. The provisioning dialog's `handleConnect()` catches this error and resets `step` back to `"connect"`, producing a jarring UI reset.

## Design

Two complementary changes: a retry mechanism in the `useSerial` hook and a reactive connect step in the provisioning dialog.

### 1. Retry `openPort()` in `connect()` (hook-level)

**File:** `web/src/lib/serial/use-serial.ts`

After `requestPort()` succeeds but `openPort()` fails, retry `openPort()` on the same port up to 2 times with a 1-second delay between attempts. The port reference is already valid (the user paired it via Chrome's picker), so the retry just waits for USB enumeration to settle.

```
connect() flow:
  manualConnectRef = true
  port = await requestPort()
  attempt 1: openPort(port) → fail? → wait 1s
  attempt 2: openPort(port) → fail? → wait 1s
  attempt 3: openPort(port) → fail? → throw
  manualConnectRef = false
```

This benefits all consumers of `useSerial().connect()`, not just the provisioning dialog.

### 2. Reactive connect step in provisioning dialog

**File:** `web/src/features/device/usb-provision-dialog.tsx`

Replace the current `handleConnect()` try/catch step machine with a reactive pattern matching the device detail page (`devices/[id]/page.tsx`):

**Current (broken):**
```
handleConnect() {
  setStep("identify")      ← eagerly shows spinner
  await connect()           ← fails on first pair
  await sendCommand("identify")
  setStep("form")
}
catch { setStep("connect") } ← jarring reset
```

**New (reactive):**
- `step === "connect"` always renders the "Connect USB" button
- Button click calls `serial.connect()` fire-and-forget (no await, no step change)
- Button is disabled with a spinner while `portState !== "closed"` (connection in progress)
- A `useEffect` watches `portState`: when it becomes `"open"` and `step === "connect"`:
  - Send `identify` command
  - If identify succeeds → set `step = "form"`, populate `identity` state
  - If identify fails → disconnect, button re-enables (portState returns to "closed")

**State flow:**
```
[Button enabled]              portState="closed", step="connect"
      ↓ click
[Button disabled + spinner]   portState="opening", step="connect"
      ↓ port opens
[Button disabled + spinner]   portState="open", identify in flight
      ↓ identify succeeds
[Provision form]              step="form", identity populated
```

If anything fails, the button re-enables. No step reset, no dialog flash.

**Removed:** The `"identify"` step's dedicated spinner rendering (`step === "identify"` block) is no longer needed — the spinner is on the button itself.

### What doesn't change

- `openPort()` internals
- `handleSerialConnect` / `handleSerialDisconnect` event handlers
- `reconnectKnownPort()` and auto-reconnect behavior
- All provisioning steps after `"form"` (testing_wifi, issuing_token, provisioning, restarting, pending, approving, done, error)
- Device detail page's connect button (already works with the reactive pattern)
- i18n keys

## Files to modify

| File | Change |
|------|--------|
| `web/src/lib/serial/use-serial.ts` | Add retry loop in `connect()` around `openPort()` |
| `web/src/features/device/usb-provision-dialog.tsx` | Replace `handleConnect()` with reactive `useEffect` + fire-and-forget button |
