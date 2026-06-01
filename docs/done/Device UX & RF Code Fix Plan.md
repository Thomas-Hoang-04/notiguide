# Device UX & RF Code Fix — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix a production bug where RF code rotation rejects 433MHz receivers, and improve the device management UX with a reload button, auto-polling, better pending-state messaging, and sizing fixes.

**Architecture:** Five independent changes — one backend one-liner and four frontend tweaks. No new components or APIs. Frontend changes touch the device list page, device detail page, pending review card, and i18n files.

**Tech Stack:** Kotlin/Spring Boot (backend), Next.js 16 / React 19 / TypeScript / next-intl (frontend)

**Spec:** `docs/planned/Device UX & RF Code Fix Spec.md`

---

### Task 1: Fix RF Code Rotate for 433MHz Devices

The `resolveRequestedBits` method in `RfCodeService` throws `width_out_of_range` when `requestedBits` is null for non-2.4GHz devices. The frontend `rotateRfCode()` call sends no `rfCodeBits`, so it always fails for 433MHz receivers. The `autoIssue` method already handles this correctly by falling back to `properties.rfCode.defaultBits433m`.

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/service/RfCodeService.kt:254`

- [ ] **Step 1: Apply the fix**

In `RfCodeService.kt`, change line 254 from:

```kotlin
else -> requestedBits ?: throw DeviceBadRequestEnvelopeException("width_out_of_range")
```

to:

```kotlin
else -> requestedBits ?: properties.rfCode.defaultBits433m
```

`properties` is the class-level `DeviceCommandSigningProperties` field (injected via constructor at line 43). `properties.rfCode.defaultBits433m` defaults to `32` (configured in `application.yaml:31`). This matches the fallback path in `autoIssue` at line 122.

---

### Task 2: Add i18n Keys

Add two new translation keys used by Tasks 3 and 4.

**Files:**
- Modify: `web/src/messages/en.json`
- Modify: `web/src/messages/vi.json`

- [ ] **Step 1: Add keys to en.json**

Inside the `devices.pending` object (currently at line 530), add the `waitingForDevice` key after the existing keys:

```json
"waitingForDevice": "Waiting for device..."
```

At the top level of the `devices` object (alongside `pendingTitle`, `emptyState`, etc.), add:

```json
"reloadAction": "Reload devices"
```

- [ ] **Step 2: Add keys to vi.json**

Inside the `devices.pending` object (line 530), add:

```json
"waitingForDevice": "Đang chờ thiết bị..."
```

At the top level of the `devices` object, add:

```json
"reloadAction": "Tải lại thiết bị"
```

- [ ] **Step 3: Verify structural mirror**

Run a quick check that both files have the same keys:

```bash
cd web && node -e "
const en = require('./src/messages/en.json');
const vi = require('./src/messages/vi.json');
const check = (a, b, path='') => {
  for (const k of Object.keys(a)) {
    if (!(k in b)) console.log('Missing in vi:', path+k);
  }
  for (const k of Object.keys(b)) {
    if (!(k in a)) console.log('Missing in en:', path+k);
  }
};
check(en.devices.pending, vi.devices.pending, 'devices.pending.');
console.log('reloadAction en:', !!en.devices.reloadAction, 'vi:', !!vi.devices.reloadAction);
"
```

Expected: no missing keys, both `reloadAction` values present.

---

### Task 3: Pending Card — Static Label + Bigger Buttons

Two changes to the same file: add a "Waiting for device..." label for `PENDING_RF_CODE` rows, and bump approve/reject buttons from `icon-sm` to `icon-lg`.

**Files:**
- Modify: `web/src/features/device/pending-review-card.tsx`

- [ ] **Step 1: Add static label for PENDING_RF_CODE**

Inside the `pendingDevices.map()` render (around line 91), the existing code has:

```tsx
{device.status === "PENDING" && (
  <>
    {/* approve button */}
    {/* reject button */}
  </>
)}
```

After this block (still inside the `<div className="ml-3 flex shrink-0 items-center gap-1">`), add:

```tsx
{device.status === "PENDING_RF_CODE" && (
  <span className="text-xs text-muted-foreground">
    {tDevices("pending.waitingForDevice")}
  </span>
)}
```

- [ ] **Step 2: Bump approve/reject button sizes**

In the same file, change both `Button` components (approve at ~line 107 and reject at ~line 125):

Change `size="icon-sm"` to `size="icon-lg"` on both buttons.

Change `className="size-4"` to `className="size-5"` on both `Check` and `X` icons.

The approve button becomes:

```tsx
<Button
  variant="ghost"
  size="icon-lg"
  onClick={() => setApproveTarget(device)}
  className="text-success hover:text-success"
  aria-label={tDevices("pending.approveButton")}
/>
```

```tsx
<Check aria-hidden="true" className="size-5" />
```

The reject button becomes:

```tsx
<Button
  variant="ghost"
  size="icon-lg"
  onClick={() => setRejectTarget(device)}
  className="text-destructive hover:text-destructive"
  aria-label={tDevices("pending.rejectButton")}
/>
```

```tsx
<X aria-hidden="true" className="size-5" />
```

---

### Task 4: Device List Page — Reload Button + Auto-Polling

Add a manual reload button next to the page title and an auto-poll effect that silently refreshes when `PENDING_RF_CODE` devices exist.

**Files:**
- Modify: `web/src/app/[locale]/dashboard/devices/page.tsx`

- [ ] **Step 1: Add RefreshCcw to imports**

Change the lucide-react import (line 3) from:

```tsx
import { Plus, Radio, Usb } from "lucide-react";
```

to:

```tsx
import { Plus, Radio, RefreshCcw, Usb } from "lucide-react";
```

The `Tooltip` imports (lines 9-13) and `Button` import (line 7) are already present.

- [ ] **Step 2: Make fetchDevices support silent mode**

Change the `fetchDevices` callback (line 65) to accept a `silent` parameter. When silent, skip `setLoading` and skip error toasts:

```tsx
const fetchDevices = useCallback(
  async (silent = false) => {
    if (!silent) setLoading(true);
    try {
      const filterKind = kindFilter === "all" ? null : kindFilter;
      const filterStore = isSuperAdmin
        ? storeFilter === "all"
          ? null
          : storeFilter
        : adminStoreId;
      const result = await listDevices(filterKind, filterStore);
      setDevices(result.devices);
    } catch (err) {
      if (!silent) {
        if (err instanceof ApiError) {
          toast.error(translateCommonApiError(err, tErrors));
        } else {
          toast.error(translateNetworkError(tErrors));
        }
      }
    } finally {
      if (!silent) setLoading(false);
    }
  },
  [kindFilter, storeFilter, isSuperAdmin, adminStoreId, tErrors],
);
```

- [ ] **Step 3: Add auto-poll effect**

After the existing `useEffect` that calls `fetchDevices` on mount (line 87), add a new effect:

```tsx
useEffect(() => {
  const hasPendingRf =
    devices?.some((d) => d.status === "PENDING_RF_CODE") ?? false;
  if (!hasPendingRf) return;

  const timer = setInterval(() => {
    void fetchDevices(true);
  }, 15_000);

  return () => clearInterval(timer);
}, [devices, fetchDevices]);
```

This silently re-fetches every 15 seconds while any `PENDING_RF_CODE` device exists. When the device transitions (e.g., to `ACTIVE`), the next poll updates `devices`, the effect re-evaluates, and the interval is not re-created.

- [ ] **Step 4: Add reload button in the header**

In the JSX, inside the first `<div className="flex items-center gap-3">` (line 134), add an outline button right after the `<h1>`. This matches the style of the existing action buttons ("Cấp phát qua USB", "Cấp mã") — `variant="outline"` with icon + text label:

```tsx
<div className="flex items-center gap-3">
  <h1 className="text-2xl font-bold">{tDevices("title")}</h1>
  <Button
    variant="outline"
    disabled={loading}
    onClick={() => void fetchDevices()}
  >
    <RefreshCcw
      aria-hidden="true"
      className={`mr-2 size-4 ${loading ? "animate-spin" : ""}`}
    />
    {tDevices("reloadAction")}
  </Button>
  {!isSuperAdmin && adminStoreHubCap && (
    <HubCapBadge
      registered={adminStoreHubCap.registered}
      max={adminStoreHubCap.max}
    />
  )}
</div>
```

The icon spins (`animate-spin`) only during manual reload (when `loading` is true). Auto-polls call `fetchDevices(true)` which doesn't set `loading`, so the icon stays static. No tooltip needed — the text label is self-explanatory.

---

### Task 5: Device Detail Page — Bigger Back Arrow

Bump the back arrow icon and its container in both the loaded and not-found states.

**Files:**
- Modify: `web/src/app/[locale]/dashboard/devices/[id]/page.tsx`

- [ ] **Step 1: Update the not-found state back arrow (line 142-145)**

Change:

```tsx
<Link
  href="/dashboard/devices"
  className="flex size-8 items-center justify-center rounded-lg text-muted-foreground transition-colors hover:bg-muted hover:text-foreground"
>
  <ArrowLeft aria-hidden="true" className="size-4" />
</Link>
```

to:

```tsx
<Link
  href="/dashboard/devices"
  className="flex size-9 items-center justify-center rounded-lg text-muted-foreground transition-colors hover:bg-muted hover:text-foreground"
>
  <ArrowLeft aria-hidden="true" className="size-5" />
</Link>
```

- [ ] **Step 2: Update the loaded state back arrow (line 161-166)**

Apply the same change — `size-8` → `size-9` on the Link container, `size-4` → `size-5` on the ArrowLeft icon.

---

### Task 6: Update Changelog

**Files:**
- Modify: `docs/CHANGELOGS.md`

- [ ] **Step 1: Add changelog entry**

Add an entry under the current date section:

```markdown
### Device UX & RF Code Fix

**Backend:**
- Fixed `RfCodeService.resolveRequestedBits` throwing `width_out_of_range` for 433MHz receivers when `rfCodeBits` is null — now falls back to `defaultBits433m` (matching `autoIssue` behavior)

**Frontend:**
- Added reload button (RefreshCcw) next to device list page title
- Added 15-second auto-polling when `PENDING_RF_CODE` devices are present in the list
- Added static "Waiting for device..." label for `PENDING_RF_CODE` rows in pending review card
- Bumped approve/reject buttons from `icon-sm` to `icon-lg` in pending review card
- Bumped back arrow from `size-4`/`size-8` to `size-5`/`size-9` on device detail page
```
