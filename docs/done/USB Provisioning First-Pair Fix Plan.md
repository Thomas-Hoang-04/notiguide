# USB Provisioning First-Pair Fix — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix the double-click-to-connect bug in the USB provisioning dialog when pairing an ESP32-C3 for the first time in Chrome.

**Architecture:** Two complementary changes — a retry loop in `useSerial().connect()` absorbs the transient USB enumeration failure, and a reactive connect step in the provisioning dialog prevents the jarring step reset. Combined, a single click reliably opens the port and shows the provision form.

**Tech Stack:** React 19, Web Serial API (WICG Serial spec), TypeScript

**Spec:** `docs/planned/USB Provisioning First-Pair Fix.md`

---

### Task 1: Add retry loop to `connect()` in `use-serial.ts`

**Files:**
- Modify: `web/src/lib/serial/use-serial.ts:143-156`

**Context:** When Chrome pairs a new ESP32-C3, the CDC-ACM USB interface resets during enumeration. `openPort()` fails because `port.open()` or the stream setup rejects. The port reference from `requestPort()` is still valid — we just need to wait and retry. Per the WICG Serial spec, `port.open()` can be called again after a failed open once `port.close()` completes (our `openPort()` catch block already releases reader/writer locks via `protocolRef.current.disconnect()` then calls `port.close()`).

- [ ] **Step 1: Replace the current `connect()` body with a retry-wrapped version**

In `web/src/lib/serial/use-serial.ts`, replace the current `connect` function (the `useCallback` starting around line 143):

```typescript
// BEFORE (lines 143-156):
const connect = useCallback(async () => {
    if (!canUseSerial) throw new Error("serial_not_supported");
    if (portStateRef.current !== "closed") return;

    manualConnectRef.current = true;
    try {
      const port = await navigator.serial.requestPort({
        filters: [ESP32_C3_USB_FILTER],
      });
      await openPort(port);
    } finally {
      manualConnectRef.current = false;
    }
  }, [canUseSerial, openPort]);
```

```typescript
// AFTER:
const connect = useCallback(async () => {
    if (!canUseSerial) throw new Error("serial_not_supported");
    if (portStateRef.current !== "closed") return;

    manualConnectRef.current = true;
    try {
      const port = await navigator.serial.requestPort({
        filters: [ESP32_C3_USB_FILTER],
      });

      let lastError: unknown;
      for (let attempt = 0; attempt < 3; attempt++) {
        try {
          await openPort(port);
          return;
        } catch (error) {
          lastError = error;
          if (attempt < 2) {
            await new Promise((resolve) => setTimeout(resolve, 1_000));
          }
        }
      }
      throw lastError;
    } finally {
      manualConnectRef.current = false;
    }
  }, [canUseSerial, openPort]);
```

**Why this is safe:**
- `requestPort()` is called once (before the loop). It requires a user gesture and is not retried — only `openPort()` retries.
- `openPort()`'s catch block releases stream locks, closes the port, nulls `portRef`, and sets `portState` back to `"closed"` before throwing. Each retry starts from a clean state.
- `manualConnectRef` stays `true` throughout all retries (the `finally` is outside the loop), so `handleSerialConnect` won't interfere during retry delays.
- If the user cancels Chrome's picker, `requestPort()` throws `DOMException` before the loop, bypassing retry entirely.

- [ ] **Step 2: Verify lint**

Run: `cd /home/thomas/Coding/notiguide/web && yarn lint`
Expected: no errors

- [ ] **Step 3: Verify types**

Run: `cd /home/thomas/Coding/notiguide/web && yarn tsc --noEmit`
Expected: no errors

---

### Task 2: Refactor provisioning dialog to reactive connect step

**Files:**
- Modify: `web/src/features/device/usb-provision-dialog.tsx`

**Context:** The current `handleConnect()` eagerly sets `step = "identify"` before awaiting `connect()`, then catches failures by resetting to `step = "connect"`. This causes a visible step-flash on first-time pairing. The fix adopts the same reactive pattern used by the device detail page (`devices/[id]/page.tsx:262-303`): fire-and-forget `connect()` on button click, then a `useEffect` detects `portState === "open"` and triggers the identify command.

- [ ] **Step 1: Remove `"identify"` from the step type and step index map**

In `web/src/features/device/usb-provision-dialog.tsx`, update these two declarations:

```typescript
// BEFORE (lines 37-48):
type ProvisionStep =
  | "connect"
  | "identify"
  | "form"
  | "testing_wifi"
  | "issuing_token"
  | "provisioning"
  | "restarting"
  | "pending"
  | "approving"
  | "done"
  | "error";
```

```typescript
// AFTER:
type ProvisionStep =
  | "connect"
  | "form"
  | "testing_wifi"
  | "issuing_token"
  | "provisioning"
  | "restarting"
  | "pending"
  | "approving"
  | "done"
  | "error";
```

And remove `identify` from the step index map:

```typescript
// BEFORE (lines 66-78):
const STEP_INDEX_MAP: Record<string, number> = {
  connect: -1,
  identify: -1,
  form: -1,
  testing_wifi: -1,
  issuing_token: 0,
  provisioning: 1,
  restarting: 2,
  pending: 5,
  approving: 5,
  done: 6,
  error: -1,
};
```

```typescript
// AFTER:
const STEP_INDEX_MAP: Record<string, number> = {
  connect: -1,
  form: -1,
  testing_wifi: -1,
  issuing_token: 0,
  provisioning: 1,
  restarting: 2,
  pending: 5,
  approving: 5,
  done: 6,
  error: -1,
};
```

- [ ] **Step 2: Delete `handleConnect()` and add reactive identify `useEffect`**

Delete the `handleConnect` function (lines 150-159):

```typescript
// DELETE this entire function:
async function handleConnect() {
    try {
      setStep("identify");
      await connect();
      const id = await sendCommand("identify");
      setIdentity(id);
      setStep("form");
    } catch {
      setStep("connect");
    }
  }
```

Add a new `useEffect` after the existing disconnect-on-close effect (after line 148). This effect watches for `portState` becoming `"open"` while still on the connect step, then sends the `identify` command:

```typescript
useEffect(() => {
    if (!open || step !== "connect" || portState !== "open") return;

    let cancelled = false;

    async function identifyDevice() {
      try {
        const id = await sendCommand("identify");
        if (!cancelled) {
          setIdentity(id);
          setStep("form");
        }
      } catch {
        if (!cancelled) {
          void disconnect();
        }
      }
    }

    void identifyDevice();

    return () => {
      cancelled = true;
    };
  }, [open, step, portState, sendCommand, disconnect]);
```

**Why the `cancelled` flag:** React StrictMode double-invokes effects in dev. The cleanup function sets `cancelled = true` so a stale invocation doesn't mutate state. In production, the cleanup runs if any dependency changes mid-flight (e.g., dialog closes while identify is in progress).

**Why `disconnect()` on identify failure:** If the port opened but the device doesn't respond to `identify`, we disconnect to return `portState` to `"closed"`, re-enabling the Connect button. The user can retry.

- [ ] **Step 3: Update the connect step's button rendering**

Replace the connect step's button section. The button now calls `connect()` fire-and-forget and shows a spinner while `portState !== "closed"`:

```tsx
// BEFORE (lines 327-346):
{step === "connect" && (
  <div className="space-y-4">
    <p className="text-sm text-muted-foreground">
      {tUsb("provision_desc")}
    </p>
    <DialogFooter>
      <Button variant="outline" onClick={() => onOpenChange(false)}>
        {tCommon("cancel")}
      </Button>
      <Button
        onClick={() => void handleConnect()}
        className="bg-primary text-primary-foreground hover:bg-primary-hover"
        disabled={!canUseSerial}
      >
        <Usb aria-hidden="true" className="mr-2 size-4" />
        {tUsb("connect")}
      </Button>
    </DialogFooter>
  </div>
)}
```

```tsx
// AFTER:
{step === "connect" && (
  <div className="space-y-4">
    <p className="text-sm text-muted-foreground">
      {tUsb("provision_desc")}
    </p>
    <DialogFooter>
      <Button variant="outline" onClick={() => onOpenChange(false)}>
        {tCommon("cancel")}
      </Button>
      <Button
        onClick={() => void connect()}
        className="bg-primary text-primary-foreground hover:bg-primary-hover"
        disabled={!canUseSerial || portState !== "closed"}
      >
        {portState !== "closed" ? (
          <Loader2 aria-hidden="true" className="mr-2 size-4 animate-spin" />
        ) : (
          <Usb aria-hidden="true" className="mr-2 size-4" />
        )}
        {portState !== "closed" ? tUsb("connecting") : tUsb("connect")}
      </Button>
    </DialogFooter>
  </div>
)}
```

**Changes:**
- `onClick`: `void handleConnect()` → `void connect()` (fire-and-forget, no step management)
- `disabled`: `!canUseSerial` → `!canUseSerial || portState !== "closed"` (prevents double-click while connecting)
- Icon: swaps `Usb` icon for `Loader2` spinner while connecting
- Label: swaps `tUsb("connect")` for `tUsb("connecting")` while connecting

Note: `Loader2` is already imported at line 3. `portState` and `connect` are already destructured from `useSerial()` at lines 93-95.

- [ ] **Step 4: Delete the `step === "identify"` rendering block**

Remove the identify spinner block entirely (lines 348-353):

```tsx
// DELETE this entire block:
{step === "identify" && (
  <div className="flex items-center gap-2 py-4 text-sm text-muted-foreground">
    <Loader2 aria-hidden="true" className="size-4 animate-spin" />
    {tUsb("connecting")}
  </div>
)}
```

This block is no longer reachable since `"identify"` was removed from `ProvisionStep`.

- [ ] **Step 5: Verify lint**

Run: `cd /home/thomas/Coding/notiguide/web && yarn lint`
Expected: no errors

- [ ] **Step 6: Verify types**

Run: `cd /home/thomas/Coding/notiguide/web && yarn tsc --noEmit`
Expected: no errors

---

### Task 3: Update changelogs

**Files:**
- Modify: `docs/CHANGELOGS.md`

- [ ] **Step 1: Add changelog entry**

Prepend a new section at the top of `docs/CHANGELOGS.md`:

```markdown
## 2026-05-20 — USB Provisioning First-Pair Fix

### Summary
Fixed a bug where first-time USB pairing in the provisioning dialog required two connect attempts. The ESP32-C3's CDC-ACM enumeration reset caused `port.open()` to fail on the first try, and the dialog's step machine reset to the "Connect USB" prompt.

### Changes Applied

#### Retry loop in `useSerial().connect()` (`use-serial.ts`)
- After `requestPort()` succeeds, `openPort()` is now retried up to 3 times with a 1-second delay between attempts
- Each failed attempt properly cleans up (releases stream locks, closes port, resets state) before retrying
- `manualConnectRef` stays true throughout all retries, preventing interference from `navigator.serial` connect events
- `requestPort()` is not retried — only `openPort()` on the already-paired port

#### Reactive connect step in provisioning dialog (`usb-provision-dialog.tsx`)
- Removed the `"identify"` provision step and its dedicated spinner rendering
- Replaced `handleConnect()` try/catch step machine with a reactive `useEffect` that watches `portState`
- When `portState` becomes `"open"` during the connect step, the effect sends the `identify` command and transitions to the form only on success
- Connect button is fire-and-forget (`void connect()`), disabled with spinner while `portState !== "closed"`
- Follows the same reactive pattern used by the device detail page

### Files updated
- `web/src/lib/serial/use-serial.ts` — retry loop in `connect()`
- `web/src/features/device/usb-provision-dialog.tsx` — reactive connect step
- `docs/CHANGELOGS.md`
```
