# Queue Device Dispatch Panel Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the hidden "Gửi qua thiết bị" dialog on the queue screen with an always-visible, paginated **Receivers panel** in the empty left-column space, with one-click per-receiver dispatch.

**Architecture:** A self-contained `DeviceDispatchPanel` (+ `DeviceDispatchCard`) renders in the queue page's left column below `ServingDisplay`. It reuses the existing `getAvailableDevices` / `issueDeviceTicket` API and the `refreshSignal` refresh pattern. A `useFitCount` hook (ResizeObserver) computes how many fixed-height cards fit so the screen does not scroll at ≥`2xl` (1024px); below that, a default page size with normal document flow. The page is made a bounded flex column at ≥`2xl` so flexbox owns the available-height math.

**Tech Stack:** Next.js 16 / React 19 (React Compiler on) / TypeScript 5, next-intl 4, Tailwind 4, shadcn/ui (Base UI), Biome, lucide-react.

**Source spec:** `docs/spec/Queue Device Dispatch Panel Spec.md`

---

## Conventions for this plan (read first)

- **No git commits in steps.** The executor decides when to commit (project rule).
- **No `yarn build` in steps.** Building is the separate audit flow (project rule). Each task verifies with lint + format + manual browser checks.
- **Verification command (run from `web/`):** `yarn format && yarn lint` — Biome formats then checks (lint + import organization). Expected: no errors, no diffs reported by `lint`.
- **No automated tests:** `web/` has no test harness (no Vitest/Jest/RTL) and the project workflow excludes post-implementation builds/tests. Verification is lint + format + the explicit manual checks in each task.
- **Manual checks** assume `yarn dev` running and login as a store ADMIN with at least one ACTIVE receiver paired to the store.
- **Impact analysis (project rule):** before editing `TicketLookup` (Task 7) and `QueuePage` (Task 8), run `gitnexus_impact({target: "TicketLookup"/"QueuePage", direction: "upstream"})` if the index is fresh; otherwise grep for importers (noted per task). Run `gitnexus_detect_changes()` before any eventual commit.
- **All UI follows** `docs/walkthrough/Web Styles.md` and `CLAUDE.md`: shadcn/ui only, bilingual EN/VI, canonical warning box, no bare `hover:underline`, complex CSS in `queue.css`.

---

## File structure

| Action | File | Responsibility |
|--------|------|----------------|
| Create | `web/src/hooks/use-media-query.ts` | SSR-safe `useMediaQuery(query)` boolean. |
| Create | `web/src/hooks/use-fit-count.ts` | `useFitCount(ref, slotPx, {enabled, fallback})` → cards that fit the measured container height. |
| Create | `web/src/features/queue/device-dispatch-card.tsx` | One receiver row: icon, name, kind label, Dispatch button (+ disabled/tooltip when no hub). |
| Create | `web/src/features/queue/device-dispatch-panel.tsx` | Orchestrator: fetch, search, pagination/fit, dispatch handler, warning, list, footer. |
| Modify | `web/src/features/queue/ticket-lookup.tsx` | Add reload button to the **right** of the ticket search input. |
| Modify | `web/src/app/[locale]/dashboard/queue/page.tsx` | Remove dispatch button + header refresh + dialog; add panel + `deviceRefreshSignal`; wire ticket reload; apply ≥`2xl` bounded-flex layout. |
| Modify | `web/src/styles/queue.css` | `.queue-device-card` fixed height + ≥`2xl` `.queue-page-shell` definite-height reset. |
| Modify | `web/src/messages/en.json`, `web/src/messages/vi.json` | i18n: add 7 keys (Task 1), retire 6 (Task 9). |
| Delete | `web/src/features/queue/device-dispatch-dialog.tsx` | Replaced by the inline panel. |
| Modify | `docs/CHANGELOGS.md` | Log all changes (Task 10). |

> `docs/walkthrough/Web Styles.md` § Status Alert Boxes was already reconciled during spec writing (added `shrink-0` to the icon rule); no further doc edit is needed here.

---

## Task 1: Add new i18n keys (EN + VI)

**Files:**
- Modify: `web/src/messages/en.json` (the `queue` object: `dispatch` sub-object + a new `queue.reloadTickets`)
- Modify: `web/src/messages/vi.json` (mirror)

Adds 7 keys to **both** files (parity preserved: 805 → 812 each). Retiring the 6 dialog keys happens in Task 9 (after their last references are removed) so no intermediate state references a missing key.

- [ ] **Step 1: Add the new keys to `queue.dispatch` in `en.json`**

Inside the `"queue"` → `"dispatch"` object, add these six keys (keep existing keys):

```json
      "panelTitle": "Receivers",
      "searchPlaceholder": "Search devices",
      "dispatchButton": "Dispatch",
      "pageIndicator": "Page {current} / {total}",
      "prevPage": "Previous page",
      "nextPage": "Next page"
```

- [ ] **Step 2: Add `reloadTickets` to the `queue` object in `en.json`**

Inside `"queue"` (a sibling of `dispatch`, e.g. next to `lookupTitle`), add:

```json
    "reloadTickets": "Reload waiting list",
```

- [ ] **Step 3: Add the mirrored keys to `queue.dispatch` in `vi.json`**

```json
      "panelTitle": "Thiết bị nhận",
      "searchPlaceholder": "Tìm thiết bị",
      "dispatchButton": "Phát vé",
      "pageIndicator": "Trang {current} / {total}",
      "prevPage": "Trang trước",
      "nextPage": "Trang sau"
```

- [ ] **Step 4: Add `reloadTickets` to the `queue` object in `vi.json`**

```json
    "reloadTickets": "Tải lại danh sách chờ",
```

> VI copy note: these are the spec's proposed strings (`Phát vé` is user-confirmed). If the user prefers `Bộ thu` over `Thiết bị nhận` for `panelTitle`, change Step 3's `panelTitle` value only.

- [ ] **Step 5: Verify parity and lint**

Run:
```bash
cd web && python3 -c "
import json
def count(o):
    n=0
    for v in o.values(): n += count(v) if isinstance(v,dict) else 1
    return n
print('en', count(json.load(open('src/messages/en.json'))))
print('vi', count(json.load(open('src/messages/vi.json'))))
" && yarn format && yarn lint
```
Expected: `en 812`, `vi 812`; Biome clean.

---

## Task 2: `useMediaQuery` hook

**Files:**
- Create: `web/src/hooks/use-media-query.ts`

- [ ] **Step 1: Create the hook**

```tsx
"use client";

import { useEffect, useState } from "react";

/**
 * SSR-safe media query hook. Initializes to `false` on the server / first paint,
 * then syncs to the real value on mount and subscribes to changes.
 */
export function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState(false);

  useEffect(() => {
    if (typeof window === "undefined") return;
    const mql = window.matchMedia(query);
    const onChange = (event: MediaQueryListEvent) => setMatches(event.matches);
    setMatches(mql.matches);
    mql.addEventListener("change", onChange);
    return () => mql.removeEventListener("change", onChange);
  }, [query]);

  return matches;
}
```

- [ ] **Step 2: Verify lint**

Run: `cd web && yarn format && yarn lint`
Expected: Biome clean.

---

## Task 3: `useFitCount` hook

**Files:**
- Create: `web/src/hooks/use-fit-count.ts`

- [ ] **Step 1: Create the hook**

```tsx
"use client";

import { type RefObject, useEffect, useState } from "react";

interface UseFitCountOptions {
  /** When false, measurement is skipped and `fallback` is returned. */
  enabled?: boolean;
  /** Value used when disabled or before the first measurement. */
  fallback?: number;
}

/**
 * Returns how many fixed-height items (`slotPx` tall, including inter-item gap)
 * fit the referenced container's measured height. ResizeObserver-driven.
 *
 * The container's height must be layout-determined (flex-fill), NOT content-driven,
 * so changing the rendered item count cannot feed back into the measurement.
 */
export function useFitCount(
  containerRef: RefObject<HTMLElement | null>,
  slotPx: number,
  { enabled = true, fallback = 5 }: UseFitCountOptions = {},
): number {
  const [count, setCount] = useState(fallback);

  useEffect(() => {
    const element = containerRef.current;
    if (!enabled || !element) {
      setCount(fallback);
      return;
    }
    const compute = () => {
      const usable = element.clientHeight;
      setCount(Math.max(1, Math.floor(usable / slotPx)));
    };
    compute();
    const observer = new ResizeObserver(compute);
    observer.observe(element);
    return () => observer.disconnect();
  }, [containerRef, slotPx, enabled, fallback]);

  return count;
}
```

- [ ] **Step 2: Verify lint**

Run: `cd web && yarn format && yarn lint`
Expected: Biome clean. (Biome's `useExhaustiveDependencies` is satisfied — `containerRef` identity is stable.)

---

## Task 4: CSS — device card height + shell reset

**Files:**
- Modify: `web/src/styles/queue.css` (append at end of file)

- [ ] **Step 1: Append the new rules**

```css

/* ── Device dispatch panel ───────────────────────────────── */

/* Fixed height so the visible-card count is deterministic.
   slot = card height (3.5rem / 56px) + list gap (0.5rem / 8px) = 64px. */
.queue-device-card {
  height: 3.5rem;
}

@media (min-width: 64rem) {
  /* 2xl breakpoint — 1024px. Replace the page-shell's min-height bleed bump with a
     DEFINITE height so the inner h-full / flex-1 chain resolves against <main>'s
     definite height (main is flex-1 inside an h-screen column). Without a definite
     height here, percentage/flex heights collapse and the device list will not bound. */
  .queue-page-shell {
    height: 100%;
    min-height: 0;
  }
}
```

- [ ] **Step 2: Verify lint**

Run: `cd web && yarn format && yarn lint`
Expected: Biome clean.

---

## Task 5: `DeviceDispatchCard` component

**Files:**
- Create: `web/src/features/queue/device-dispatch-card.tsx`

Mirrors the waiting-list call-button family (`size="sm"`, `bg-action`). When no hub is ready, the button is disabled and wrapped in a `Tooltip` using the established render-prop pattern from `queue/page.tsx`.

- [ ] **Step 1: Create the component**

```tsx
"use client";

import { Loader2, Radio, Send } from "lucide-react";
import { useTranslations } from "next-intl";
import { Button } from "@/components/ui/button";
import {
  Tooltip,
  TooltipContent,
  TooltipTrigger,
} from "@/components/ui/tooltip";
import type { DeviceDto } from "@/types/device";

interface DeviceDispatchCardProps {
  device: DeviceDto;
  dispatchReady: boolean;
  dispatching: boolean;
  onDispatch: (deviceId: string) => void;
}

export function DeviceDispatchCard({
  device,
  dispatchReady,
  dispatching,
  onDispatch,
}: DeviceDispatchCardProps) {
  const tQueue = useTranslations("queue");
  const tDevices = useTranslations("devices");

  const name = device.assignedName || device.publicId || "";

  return (
    <div className="queue-device-card glass-panel glass-panel-primary flex items-center justify-between gap-3 rounded-lg px-4">
      <div className="flex min-w-0 items-center gap-2.5">
        <Radio aria-hidden="true" className="size-4 shrink-0 text-primary" />
        <span className="truncate text-sm font-medium">{name}</span>
        <span className="shrink-0 text-xs text-muted-foreground">
          {tDevices(`kind.${device.kind}`)}
        </span>
      </div>

      {dispatchReady ? (
        <Button
          size="sm"
          disabled={dispatching}
          onClick={() => onDispatch(device.id)}
          className="shrink-0 gap-1.5 bg-action text-action-foreground hover:bg-action-hover"
        >
          {dispatching ? (
            <Loader2 aria-hidden="true" className="size-3.5 animate-spin" />
          ) : (
            <Send aria-hidden="true" className="size-3.5" />
          )}
          {tQueue("dispatch.dispatchButton")}
        </Button>
      ) : (
        <Tooltip>
          <TooltipTrigger
            render={
              <Button
                size="sm"
                disabled
                aria-label={tQueue("dispatch.dispatchButton")}
                className="shrink-0 gap-1.5 bg-action text-action-foreground hover:bg-action-hover"
              >
                <Send aria-hidden="true" className="size-3.5" />
                {tQueue("dispatch.dispatchButton")}
              </Button>
            }
          />
          <TooltipContent>{tQueue("dispatch.disabledNoHub")}</TooltipContent>
        </Tooltip>
      )}
    </div>
  );
}
```

- [ ] **Step 2: Verify lint**

Run: `cd web && yarn format && yarn lint`
Expected: Biome clean.

---

## Task 6: `DeviceDispatchPanel` component

**Files:**
- Create: `web/src/features/queue/device-dispatch-panel.tsx`

- [ ] **Step 1: Create the component**

```tsx
"use client";

import {
  AlertTriangle,
  ChevronLeft,
  ChevronRight,
  RefreshCcw,
  Search,
} from "lucide-react";
import { useTranslations } from "next-intl";
import { useCallback, useEffect, useMemo, useRef, useState } from "react";
import { toast } from "sonner";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Skeleton } from "@/components/ui/skeleton";
import { getAvailableDevices, issueDeviceTicket } from "@/features/queue/api";
import { useFitCount } from "@/hooks/use-fit-count";
import { useMediaQuery } from "@/hooks/use-media-query";
import {
  translateCommonApiError,
  translateNetworkError,
} from "@/lib/api-error";
import { ApiError } from "@/types/api";
import type { DeviceDto } from "@/types/device";
import { DeviceDispatchCard } from "./device-dispatch-card";

// .queue-device-card height (56px) + list gap (8px)
const DEVICE_CARD_SLOT_PX = 64;
const DEFAULT_PAGE_SIZE = 5;

interface DeviceDispatchPanelProps {
  storeId: string;
  refreshSignal: number;
  onDispatched: () => void;
}

export function DeviceDispatchPanel({
  storeId,
  refreshSignal,
  onDispatched,
}: DeviceDispatchPanelProps) {
  const tQueue = useTranslations("queue");
  const tErrors = useTranslations("errors");

  const [devices, setDevices] = useState<DeviceDto[]>([]);
  const [dispatchReady, setDispatchReady] = useState(false);
  const [loading, setLoading] = useState(true);
  const [reloading, setReloading] = useState(false);
  const [query, setQuery] = useState("");
  const [page, setPage] = useState(0);
  const [dispatchingIds, setDispatchingIds] = useState<Set<string>>(new Set());

  const listRef = useRef<HTMLDivElement>(null);
  const is2xlUp = useMediaQuery("(min-width: 64rem)");
  const pageSize = useFitCount(listRef, DEVICE_CARD_SLOT_PX, {
    enabled: is2xlUp,
    fallback: DEFAULT_PAGE_SIZE,
  });

  const fetchDevices = useCallback(async () => {
    try {
      const res = await getAvailableDevices(storeId);
      setDevices(res.devices);
      setDispatchReady(res.dispatchReady);
    } catch {
      setDevices([]);
      setDispatchReady(false);
    } finally {
      setLoading(false);
    }
  }, [storeId]);

  useEffect(() => {
    void fetchDevices();
  }, [fetchDevices]);

  useEffect(() => {
    if (refreshSignal) void fetchDevices();
  }, [refreshSignal, fetchDevices]);

  const handleReload = useCallback(() => {
    setReloading(true);
    void fetchDevices().finally(() => setReloading(false));
  }, [fetchDevices]);

  const handleDispatch = useCallback(
    async (deviceId: string) => {
      setDispatchingIds((prev) => new Set(prev).add(deviceId));
      try {
        const ticket = await issueDeviceTicket(storeId, { deviceId });
        toast.success(
          tQueue("dispatch.successToast", { number: ticket.number }),
        );
        onDispatched();
      } catch (err) {
        if (err instanceof ApiError) {
          if (err.error === "no_active_transmitter") {
            toast.error(tQueue("dispatch.errorNoActiveTransmitter"));
          } else if (err.error === "device_busy") {
            toast.error(tQueue("dispatch.errorDeviceBusy"));
          } else {
            toast.error(translateCommonApiError(err, tErrors));
          }
        } else {
          toast.error(translateNetworkError(tErrors));
        }
      } finally {
        setDispatchingIds((prev) => {
          const next = new Set(prev);
          next.delete(deviceId);
          return next;
        });
      }
    },
    [storeId, onDispatched, tQueue, tErrors],
  );

  const filtered = useMemo(() => {
    const normalized = query.trim().toLowerCase();
    if (!normalized) return devices;
    return devices.filter((d) =>
      (d.assignedName || d.publicId || "").toLowerCase().includes(normalized),
    );
  }, [devices, query]);

  const totalPages = Math.max(1, Math.ceil(filtered.length / pageSize));
  // Derive the in-range page during render — no setState-in-effect. When the list
  // shrinks or pageSize changes, `safePage` clamps automatically without an extra render.
  const safePage = Math.min(Math.max(0, page), totalPages - 1);

  const visible = filtered.slice(
    safePage * pageSize,
    (safePage + 1) * pageSize,
  );

  return (
    <div className="flex flex-col gap-3 2xl:min-h-0 2xl:flex-1">
      {/* Header: title + reload (left of search) + search */}
      <div className="flex items-center justify-between gap-3">
        <h3 className="text-base font-semibold">
          {tQueue("dispatch.panelTitle")}
        </h3>
        <div className="flex items-center gap-2">
          <Button
            variant="outline"
            size="icon"
            disabled={reloading}
            onClick={handleReload}
            aria-label={tQueue("dispatch.refresh")}
          >
            <RefreshCcw
              aria-hidden="true"
              className={`size-4 ${reloading ? "animate-spin" : ""}`}
            />
          </Button>
          <div className="relative w-40 l:w-48">
            <Search
              aria-hidden="true"
              className="pointer-events-none absolute left-3 top-1/2 size-4 -translate-y-1/2 text-muted-foreground"
            />
            <Input
              value={query}
              onChange={(e) => {
                setQuery(e.target.value);
                setPage(0);
              }}
              placeholder={tQueue("dispatch.searchPlaceholder")}
              className="h-9 pl-9"
            />
          </div>
        </div>
      </div>

      {/* No-hub warning (between title row and first card; also above the
          empty-state message when there are no available receivers) */}
      {!loading && !dispatchReady && (
        <div className="flex items-center gap-2.5 rounded-xl border border-warning/40 bg-warning/15 px-3 py-2.5 text-xs text-warning dark:border-warning/50 dark:bg-warning/20">
          <AlertTriangle
            aria-hidden="true"
            className="size-4 shrink-0 text-warning"
          />
          {tQueue("dispatch.disabledNoHub")}
        </div>
      )}

      {/* Card list — flex-filled; height is layout-determined, not content-driven */}
      <div
        ref={listRef}
        className="flex min-h-0 flex-1 flex-col gap-2 overflow-hidden"
      >
        {loading &&
          ["a", "b", "c"].map((id) => (
            <Skeleton key={id} className="queue-device-card rounded-lg" />
          ))}

        {!loading && filtered.length === 0 && (
          <p className="py-6 text-center text-sm text-muted-foreground">
            {tQueue("dispatch.disabledNoDevice")}
          </p>
        )}

        {!loading &&
          visible.map((device) => (
            <DeviceDispatchCard
              key={device.id}
              device={device}
              dispatchReady={dispatchReady}
              dispatching={dispatchingIds.has(device.id)}
              onDispatch={handleDispatch}
            />
          ))}
      </div>

      {/* Pagination footer */}
      {!loading && totalPages > 1 && (
        <div className="flex items-center justify-center gap-3">
          <Button
            variant="ghost"
            size="icon"
            disabled={safePage === 0}
            onClick={() => setPage(Math.max(0, safePage - 1))}
            aria-label={tQueue("dispatch.prevPage")}
          >
            <ChevronLeft aria-hidden="true" className="size-4" />
          </Button>
          <span className="text-xs tabular-nums text-muted-foreground">
            {tQueue("dispatch.pageIndicator", {
              current: safePage + 1,
              total: totalPages,
            })}
          </span>
          <Button
            variant="ghost"
            size="icon"
            disabled={safePage >= totalPages - 1}
            onClick={() => setPage(Math.min(totalPages - 1, safePage + 1))}
            aria-label={tQueue("dispatch.nextPage")}
          >
            <ChevronRight aria-hidden="true" className="size-4" />
          </Button>
        </div>
      )}
    </div>
  );
}
```

- [ ] **Step 2: Verify lint**

Run: `cd web && yarn format && yarn lint`
Expected: Biome clean. (`ApiError.error` is the discriminator field already used by `device-dispatch-dialog.tsx`.)

> **Known consideration — footer/pageSize coupling:** the footer renders only when
> `totalPages > 1`, and it consumes ~40px, which lowers the measured list height and thus
> `pageSize`. This converges (reducing `pageSize` only *increases* `totalPages`, so the
> "footer shown" state stays self-consistent; the "footer hidden" state needs
> `pageSize ≥ count`, also self-consistent), and `overflow-hidden` masks any one-frame
> settle. If manual testing at the exact boundary reveals visible flicker, make the footer
> height constant (always render the footer row, disabling the arrows when `totalPages === 1`)
> so the list height no longer depends on pagination.

---

## Task 7: `TicketLookup` — add waiting-list reload button

**Files:**
- Modify: `web/src/features/queue/ticket-lookup.tsx` (full rewrite — small file)

- [ ] **Step 1 (impact):** Confirm callers. `TicketLookup` is rendered only by `queue/page.tsx`. Run `gitnexus_impact({target: "TicketLookup", direction: "upstream"})` if the index is fresh; otherwise: `grep -rn "TicketLookup" web/src`. Expected: only `queue/page.tsx`.

- [ ] **Step 2: Replace the file contents**

```tsx
"use client";

import { RefreshCcw, Search } from "lucide-react";
import { useTranslations } from "next-intl";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";

interface TicketLookupProps {
  searchQuery: string;
  onSearchQueryChange: (query: string) => void;
  onReload: () => void;
  reloading: boolean;
}

export function TicketLookup({
  searchQuery,
  onSearchQueryChange,
  onReload,
  reloading,
}: TicketLookupProps) {
  const tQueue = useTranslations("queue");

  return (
    <div className="space-y-2">
      <h3 className="text-base font-semibold">{tQueue("lookupTitle")}</h3>
      <div className="flex items-center gap-2">
        <div className="relative flex-1">
          <Search
            aria-hidden="true"
            className="pointer-events-none absolute left-3 top-1/2 size-4 -translate-y-1/2 text-muted-foreground"
          />
          <Input
            value={searchQuery}
            onChange={(e) => onSearchQueryChange(e.target.value)}
            placeholder={tQueue("lookupPlaceholder")}
            className="pl-9"
          />
        </div>
        <Button
          variant="outline"
          size="icon"
          disabled={reloading}
          onClick={onReload}
          aria-label={tQueue("reloadTickets")}
        >
          <RefreshCcw
            aria-hidden="true"
            className={`size-4 ${reloading ? "animate-spin" : ""}`}
          />
        </Button>
      </div>
    </div>
  );
}
```

- [ ] **Step 3: Verify lint**

Run: `cd web && yarn format && yarn lint`
Expected: Biome clean. (`queue/page.tsx` will not yet pass the new required props — that is fixed in Task 8; do not run a type-check in isolation here.)

---

## Task 8: `QueuePage` integration

**Files:**
- Modify: `web/src/app/[locale]/dashboard/queue/page.tsx`

Apply the following hunks in order. Each shows the exact existing text and its replacement.

- [ ] **Step 1 (impact):** Run `gitnexus_impact({target: "QueuePage", direction: "upstream"})` if fresh; this is a route page (leaf), so the blast radius is the route itself. Report it.

- [ ] **Step 2: Replace the lucide + tooltip + api + dialog imports**

Find:
```tsx
import { Loader2, PauseCircle, Radio, RefreshCcw } from "lucide-react";
```
Replace with:
```tsx
import { Loader2, PauseCircle } from "lucide-react";
```

Find and **delete** this import block entirely:
```tsx
import {
  Tooltip,
  TooltipContent,
  TooltipTrigger,
} from "@/components/ui/tooltip";
```

Find:
```tsx
import {
  callNext,
  getAvailableDevices,
  getPublicStoreInfo,
} from "@/features/queue/api";
```
Replace with:
```tsx
import { callNext, getPublicStoreInfo } from "@/features/queue/api";
```

Find:
```tsx
import { DeviceDispatchDialog } from "@/features/queue/device-dispatch-dialog";
```
Replace with:
```tsx
import { DeviceDispatchPanel } from "@/features/queue/device-dispatch-panel";
```

- [ ] **Step 3: Replace the dispatch-related state**

Find:
```tsx
  const [dispatchDialogOpen, setDispatchDialogOpen] = useState(false);
  const [dispatchReady, setDispatchReady] = useState(false);
  const [hasAvailableDevices, setHasAvailableDevices] = useState(false);
  const [dispatchRefreshing, setDispatchRefreshing] = useState(false);
```
Replace with:
```tsx
  const [deviceRefreshSignal, setDeviceRefreshSignal] = useState(0);
  const [ticketReloading, setTicketReloading] = useState(false);
```

- [ ] **Step 4: Delete the `fetchDispatchAvailability` callback**

Find and delete the entire block:
```tsx
  // Fetch dispatch availability
  const fetchDispatchAvailability = useCallback(async () => {
    if (!storeId) return;
    try {
      const res = await getAvailableDevices(storeId);
      setDispatchReady(res.dispatchReady);
      setHasAvailableDevices(res.devices.length > 0);
    } catch {
      setDispatchReady(false);
      setHasAvailableDevices(false);
    }
  }, [storeId]);

```

- [ ] **Step 5: Remove the availability call from the settings effect**

Find:
```tsx
    })();
    void fetchDispatchAvailability();
  }, [storeId, fetchDispatchAvailability]);
```
Replace with:
```tsx
    })();
  }, [storeId]);
```

- [ ] **Step 6: Update the SSE handler to bump `deviceRefreshSignal`**

Find:
```tsx
      void fetchDispatchAvailability();
      return;
    }
```
Replace with:
```tsx
      setDeviceRefreshSignal((s) => s + 1);
      return;
    }
```

Find:
```tsx
      if (servingTicketsRef.current.some((t) => t.id === event.ticketId)) {
        removeServingTicket(event.ticketId);
      }
      void fetchDispatchAvailability();
    }
```
Replace with:
```tsx
      if (servingTicketsRef.current.some((t) => t.id === event.ticketId)) {
        removeServingTicket(event.ticketId);
      }
      setDeviceRefreshSignal((s) => s + 1);
    }
```

- [ ] **Step 7: Delete the `dispatchEnabled` / `dispatchTooltip` computed values**

Find and delete:
```tsx
  const dispatchEnabled = dispatchReady && hasAvailableDevices;
  const dispatchTooltip = !hasAvailableDevices
    ? tQueue("dispatch.disabledNoDevice")
    : !dispatchReady
      ? tQueue("dispatch.disabledNoHub")
      : null;

```

- [ ] **Step 8: Add the dispatch + ticket-reload handlers**

Immediately after `const storeName = useStoreName(storeId);`, add:
```tsx

  const handleDeviceDispatched = useCallback(() => {
    setStatsRefreshSignal((s) => s + 1);
    setDeviceRefreshSignal((s) => s + 1);
  }, []);

  const handleTicketReload = useCallback(() => {
    setTicketReloading(true);
    setStatsRefreshSignal((s) => s + 1);
    window.setTimeout(() => setTicketReloading(false), 500);
  }, []);
```

- [ ] **Step 9: Remove the dispatch + refresh buttons from the header**

Find and delete this entire block (the dispatch button and the refresh tooltip button), leaving `QueueStateToggle` and `CleanupButton`:
```tsx
            {dispatchEnabled ? (
              <Button
                variant="outline"
                onClick={() => setDispatchDialogOpen(true)}
              >
                <Radio aria-hidden="true" className="mr-2 size-4" />
                {tQueue("dispatch.action")}
              </Button>
            ) : (
              <Tooltip>
                <TooltipTrigger
                  render={
                    <Button
                      variant="outline"
                      disabled
                      aria-label={tQueue("dispatch.action")}
                    >
                      <Radio aria-hidden="true" className="mr-2 size-4" />
                      {tQueue("dispatch.action")}
                    </Button>
                  }
                />
                <TooltipContent>
                  {dispatchTooltip ?? tQueue("dispatch.disabledNoDevice")}
                </TooltipContent>
              </Tooltip>
            )}
            <Tooltip>
              <TooltipTrigger
                render={
                  <Button
                    variant="outline"
                    size="icon"
                    disabled={dispatchRefreshing}
                    onClick={() => {
                      setDispatchRefreshing(true);
                      void fetchDispatchAvailability().finally(() =>
                        setDispatchRefreshing(false),
                      );
                    }}
                    aria-label={tQueue("dispatch.refresh")}
                  >
                    <RefreshCcw
                      aria-hidden="true"
                      className={`size-4 ${dispatchRefreshing ? "animate-spin" : ""}`}
                    />
                  </Button>
                }
              />
              <TooltipContent>{tQueue("dispatch.refresh")}</TooltipContent>
            </Tooltip>
```

- [ ] **Step 10: Make the content wrapper a bounded flex column at ≥`2xl`**

Find:
```tsx
      <div className="space-y-4 l:space-y-6">
        {/* Header */}
```
Replace with:
```tsx
      <div className="space-y-4 l:space-y-6 2xl:flex 2xl:h-full 2xl:min-h-0 2xl:flex-col 2xl:space-y-0 2xl:gap-6 2xl:overflow-hidden">
        {/* Header */}
```

- [ ] **Step 11: Make the grid fill the remaining height; bound the columns**

Find:
```tsx
        <div className="grid gap-4 l:gap-6 2xl:grid-cols-3">
          {/* Left column — main controls */}
          <div className="space-y-4 l:space-y-6 2xl:col-span-2">
```
Replace with:
```tsx
        <div className="grid gap-4 l:gap-6 2xl:min-h-0 2xl:flex-1 2xl:grid-cols-3">
          {/* Left column — main controls */}
          <div className="space-y-4 l:space-y-6 2xl:col-span-2 2xl:flex 2xl:min-h-0 2xl:flex-col 2xl:space-y-0 2xl:gap-6">
```

- [ ] **Step 12: Add the panel below `ServingDisplay`**

Find:
```tsx
            {/* Currently Serving */}
            <ServingDisplay storeId={storeId} allowNoShow={allowNoShow} />
          </div>
```
Replace with:
```tsx
            {/* Currently Serving */}
            <ServingDisplay storeId={storeId} allowNoShow={allowNoShow} />

            {/* Receivers — inline device dispatch */}
            <DeviceDispatchPanel
              storeId={storeId}
              refreshSignal={deviceRefreshSignal}
              onDispatched={handleDeviceDispatched}
            />
          </div>
```

- [ ] **Step 13: Bound the right column (internal scroll) and wire the ticket reload**

Find:
```tsx
          {/* Right column — ticket lookup + waiting list */}
          <div className="space-y-4 l:space-y-6">
            <TicketLookup
              searchQuery={ticketSearchQuery}
              onSearchQueryChange={setTicketSearchQuery}
            />
```
Replace with:
```tsx
          {/* Right column — ticket lookup + waiting list */}
          <div className="space-y-4 l:space-y-6 2xl:min-h-0 2xl:overflow-y-auto">
            <TicketLookup
              searchQuery={ticketSearchQuery}
              onSearchQueryChange={setTicketSearchQuery}
              onReload={handleTicketReload}
              reloading={ticketReloading}
            />
```

- [ ] **Step 14: Remove the dialog at the bottom of the component**

Find and delete:
```tsx

      <DeviceDispatchDialog
        open={dispatchDialogOpen}
        onOpenChange={setDispatchDialogOpen}
        storeId={storeId}
        onSuccess={() => {
          setStatsRefreshSignal((s) => s + 1);
          void fetchDispatchAvailability();
        }}
      />
```

- [ ] **Step 15: Verify lint**

Run: `cd web && yarn format && yarn lint`
Expected: Biome clean. No unused imports/vars remain (`Radio`, `RefreshCcw`, `Tooltip*`, `getAvailableDevices`, `DeviceDispatchDialog` all removed).

- [ ] **Step 16: Manual verification (browser, ≥1024px)**

With `yarn dev`, store ADMIN, ≥1 ACTIVE receiver:
1. Queue screen shows the **Receivers** panel in the left column below "Currently serving"; no "Gửi qua thiết bị" button or standalone `⟳` in the header.
2. Click a receiver's **Phát vé** → success toast, the device drops off the list, a new device-bound ticket appears in "Vé đang chờ", and the count increments.
3. The ticket search row has a `⟳` to the **right**; clicking it spins briefly and refreshes the waiting list.
4. Resize the window narrower/wider → the number of receiver cards adjusts; the page does **not** scroll at ≥1024px. Call a ticket (serving card grows) → fewer cards per page.
5. Below 1024px → layout stacks, page scrolls normally, panel shows up to 5 cards with pagination.

---

## Task 9: Delete the dialog + retire its i18n keys

**Files:**
- Delete: `web/src/features/queue/device-dispatch-dialog.tsx`
- Modify: `web/src/messages/en.json`, `web/src/messages/vi.json`

- [ ] **Step 1: Confirm no remaining importers**

Run: `grep -rn "device-dispatch-dialog\|DeviceDispatchDialog" web/src`
Expected: no matches (Task 8 removed the only import).

- [ ] **Step 2: Delete the dialog file**

Run: `rm "web/src/features/queue/device-dispatch-dialog.tsx"`

- [ ] **Step 3: Retire the 6 dialog-only keys from BOTH `en.json` and `vi.json`**

Delete these keys from the `"queue"` → `"dispatch"` object in **both** files:
`action`, `dialogTitle`, `dialogDescription`, `deviceLabel`, `cancel`, `confirm`.

> Keep `disabledNoDevice` (now the panel empty-state, **also used by** `features/device/usb-dispatch-dialog.tsx`), `disabledNoHub`, `refresh`, `successToast`, `errorNoActiveTransmitter`, `errorDeviceBusy`, `errorDeviceNotFound`, `errorInfrastructure`, `badgeLabel`.

- [ ] **Step 4: Verify no references to retired keys remain**

Run:
```bash
grep -rn "dispatch.action\|dispatch.dialogTitle\|dispatch.dialogDescription\|dispatch.deviceLabel\|dispatch.cancel\|dispatch.confirm" web/src
```
Expected: no matches.

- [ ] **Step 5: Verify parity and lint**

Run:
```bash
cd web && python3 -c "
import json
def count(o):
    n=0
    for v in o.values(): n += count(v) if isinstance(v,dict) else 1
    return n
print('en', count(json.load(open('src/messages/en.json'))))
print('vi', count(json.load(open('src/messages/vi.json'))))
" && yarn format && yarn lint
```
Expected: `en 806`, `vi 806`; Biome clean.

---

## Task 10: Changelog

**Files:**
- Modify: `docs/CHANGELOGS.md` (prepend a dated entry under the top `# Changelogs` heading)

- [ ] **Step 1: Prepend the entry**

```markdown
## 2026-06-08 — Queue: inline Receivers dispatch panel (replaces dialog)

Replaced the header "Gửi qua thiết bị" dialog with an always-visible, paginated Receivers
panel in the queue screen's left column (below "Currently serving").

**New:**
- `features/queue/device-dispatch-panel.tsx` — fetches `getAvailableDevices`, client-side
  name search, per-receiver one-click dispatch (`issueDeviceTicket`), no-hub warning +
  disabled buttons, fixed-height list with dynamic fit + pagination.
- `features/queue/device-dispatch-card.tsx` — one receiver row (icon, name, kind, Phát vé/Dispatch).
- `hooks/use-fit-count.ts` — ResizeObserver visible-card count.
- `hooks/use-media-query.ts` — SSR-safe media query (gates fit-measurement at ≥2xl).

**Modified:**
- `app/[locale]/dashboard/queue/page.tsx` — removed dispatch button + standalone header
  refresh + dialog; added `deviceRefreshSignal` (bumped by SSE + dispatch); ≥2xl bounded
  flex layout (no page scroll); right column scrolls internally at ≥2xl.
- `features/queue/ticket-lookup.tsx` — added waiting-list reload button (right of search).
- `styles/queue.css` — `.queue-device-card` fixed height; ≥2xl `.queue-page-shell` definite-height reset.
- `messages/en.json` / `vi.json` — added 7 keys, retired 6 dialog keys (806 = 806).
- `docs/walkthrough/Web Styles.md` — Status Alert Boxes icon rule now mandates `shrink-0`.

**Deleted:** `features/queue/device-dispatch-dialog.tsx`.

**Device-card button** labelled **Phát vé / Dispatch** (distinct from the serving card's
`Phục vụ` / Serve, resolving the label collision).
```

- [ ] **Step 2: Verify**

Run: `cd web && yarn format && yarn lint`
Expected: Biome clean (markdown is not linted by Biome here; this confirms the web app still passes).

---

## Self-review

**Spec coverage:**
- §3 goal 1 (remove button + dialog) → Tasks 8 (button), 9 (dialog). ✅
- §3 goal 2 (inline panel) → Tasks 6, 8. ✅
- §3 goal 3 (one-click dispatch, Phát vé) → Tasks 5, 6; label in Task 1. ✅
- §3 goal 4 (title + reload + search) → Task 6 header. ✅
- §3 goal 5 (no-hub: list + warning + disabled + tooltip) → Tasks 5, 6. ✅
- §3 goal 6 (fixed-height, dynamic-fit pagination) → Tasks 3, 4, 6. ✅
- §3 goal 7 (ticket reload right of search) → Tasks 7, 8. ✅
- §5 file structure → matches the File structure table. ✅
- §6.1 single-refetch path → Task 8 Step 8 (`handleDeviceDispatched` bumps both signals); panel has no self-refetch. ✅
- §7 layout (definite-height shell reset, flex chain) → Tasks 4, 8. ✅
- §8 i18n (add 7 / retire 6, parity) → Tasks 1, 9. ✅
- §9 UI rules (warning box, action button, shadcn-only, bilingual) → Tasks 5, 6. ✅

**Placeholder scan:** No TBD/TODO; every code step has complete code; every command has expected output.

**Type consistency:** `DeviceDispatchPanelProps {storeId, refreshSignal, onDispatched}` matches the render in Task 8 Step 12. `DeviceDispatchCardProps {device, dispatchReady, dispatching, onDispatch}` matches Task 6's usage. `TicketLookupProps {searchQuery, onSearchQueryChange, onReload, reloading}` matches Task 8 Step 13. `useFitCount(ref, slotPx, {enabled, fallback})` and `useMediaQuery(query)` signatures match call sites. `DEVICE_CARD_SLOT_PX = 64` matches the `.queue-device-card` 56px + 8px gap in Task 4.
