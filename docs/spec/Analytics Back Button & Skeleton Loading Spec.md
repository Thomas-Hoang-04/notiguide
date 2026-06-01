# Analytics Back Button & Skeleton Loading Plan

**Last updated**: 2026-05-19

---

## Overview

Two targeted additions to the per-store analytics dashboard (`StoreAnalytics`):

1. **Back button** — SUPER_ADMIN-only navigation back to the analytics overview
2. **Skeleton loading** — section-shaped skeleton placeholders while data is being fetched

---

## 1. Back Button (SUPER_ADMIN Only)

### Behavior

- Appears **only** when a SUPER_ADMIN drills into a specific store via the "View" link in Store Rankings
- Regular admins viewing their own store's analytics do **not** see a back button (they have no overview to return to)
- Links to `/dashboard/analytics` (the overview page)

### Implementation

**Prop threading:**

- `StoreAnalytics` gains an optional `showBackButton?: boolean` prop (default `false`)
- `[storeId]/page.tsx` passes `showBackButton={true}` (this page is already SUPER_ADMIN-gated)
- The main `analytics/page.tsx` renders `StoreAnalytics` for regular admins without the prop

**Header layout:**

```
[←] Analytics — Cà phê Bách Khoa     (SUPER_ADMIN drill-down)
Analytics — Cà phê Bách Khoa          (Regular ADMIN, no back button)
```

**Pattern to follow:** Reuse the exact back-button pattern from `devices/[id]/page.tsx`:

```tsx
<Link
  href="/dashboard/analytics"
  className="flex size-8 items-center justify-center rounded-lg text-muted-foreground transition-colors hover:bg-muted hover:text-foreground"
>
  <ArrowLeft aria-hidden="true" className="size-4" />
</Link>
```

Wrapped in a `flex items-center gap-3` container with the existing `<h1>`.

### i18n

New key: `analytics.backToOverview` — used as `aria-label` on the back link.

| Key | en.json | vi.json |
|-----|---------|---------|
| `analytics.backToOverview` | `Back to overview` | `Quay lại tổng quan` |

---

## 2. Skeleton Loading (Section-Shaped)

### Current problem

When the dashboard loads, each section either shows "Loading" text, returns `null` (disappears from DOM), or shows "No Data" — causing layout shift and a jarring experience.

### Approach

Replace the current loading fallbacks with Skeleton-based placeholders that match each section's shape. Uses the existing `Skeleton` component (`animate-pulse rounded-md bg-muted`).

### Per-section skeletons

#### Period stat cards (4 top cards)

Currently shows "Loading" text. Replace with:

- 4 glass-card skeletons in the same `analytics-stats-grid`
- Each card: one tall `Skeleton` line (value) + one short `Skeleton` line (label)

#### Summary panel (8-item grid)

Currently returns `null` while loading. Replace with:

- A glass-card with a `Skeleton` title line ("Summary" placeholder)
- 2×4 grid of skeleton pairs (short label line + medium value line), matching the real layout

#### Charts (4 in 2×2 grid)

Each chart currently shows "No Data" while loading. Replace with:

- A glass-card with a `Skeleton` title line
- A tall rectangular `Skeleton` block matching chart height (`h-56 l:h-64`)

#### Heatmap (bottom)

Currently shows "No Data" while loading. Replace with:

- A glass-card with a `Skeleton` title line
- A wide rectangular `Skeleton` block matching heatmap height

### Component changes

Each component handles its own skeleton internally — when `loading` is true, it renders the skeleton variant instead of the real content. No separate skeleton wrapper component needed.

| Component | Current loading state | New loading state |
|-----------|----------------------|-------------------|
| `StorePeriodStats` | "Loading" text | 4 glass-card skeletons with value+label lines |
| `SummaryCards` | `return null` | Glass-card with title skeleton + 2×4 grid of skeleton pairs |
| `OutcomeChart` | "No Data" text | Glass-card with title skeleton + tall skeleton block |
| `WaitDistributionChart` | "No Data" text | Glass-card with title skeleton + tall skeleton block |
| `PeakHoursChart` | "No Data" text | Glass-card with title skeleton + tall skeleton block |
| `ThroughputChart` | "No Data" text | Glass-card with title skeleton + tall skeleton block |
| `HourlyHeatmap` | "No Data" text | Glass-card with title skeleton + wide skeleton block |

### Skeleton sizing reference

- Stat card value: `h-7 w-16` (mimics large number)
- Stat card label: `h-3 w-24` (mimics uppercase label)
- Summary title: `h-4 w-20`
- Summary value: `h-6 w-14`
- Summary label: `h-3 w-20`
- Chart title: `h-4 w-32`
- Chart body: `h-56 l:h-64 w-full`
- Heatmap body: `h-48 w-full`

---

## Files to modify

| File | Change |
|------|--------|
| `web/src/features/analytics/store-analytics.tsx` | Add `showBackButton` prop, back button in header |
| `web/src/app/[locale]/dashboard/analytics/[storeId]/page.tsx` | Pass `showBackButton={true}` |
| `web/src/features/analytics/period-stats.tsx` | Skeleton loading for `StorePeriodStats` |
| `web/src/features/analytics/summary-cards.tsx` | Skeleton loading for `SummaryCards` |
| `web/src/features/analytics/outcome-chart.tsx` | Skeleton loading for `OutcomeChart` |
| `web/src/features/analytics/wait-distribution-chart.tsx` | Skeleton loading for `WaitDistributionChart` |
| `web/src/features/analytics/peak-hours-chart.tsx` | Skeleton loading for `PeakHoursChart` |
| `web/src/features/analytics/throughput-chart.tsx` | Skeleton loading for `ThroughputChart` |
| `web/src/features/analytics/hourly-heatmap.tsx` | Skeleton loading for `HourlyHeatmap` |
| `web/src/messages/en.json` | Add `analytics.backToOverview` |
| `web/src/messages/vi.json` | Add `analytics.backToOverview` |
| `docs/CHANGELOGS.md` | Log changes |
