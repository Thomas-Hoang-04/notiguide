# Analytics Back Button & Skeleton Loading — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a SUPER_ADMIN-only back button to the per-store analytics dashboard and replace all loading fallbacks with section-shaped skeleton placeholders — both on the per-store view and the SUPER_ADMIN overview.

**Architecture:** The back button is prop-driven — `StoreAnalytics` receives `showBackButton` from the page component that already knows the user's role. Skeletons are component-local — each analytics section handles its own loading skeleton internally using the existing shadcn `Skeleton` primitive.

**Tech Stack:** Next.js 16, React 19, shadcn/ui `Skeleton`, lucide-react `ArrowLeft`, next-intl

---

### File Map

| File | Action | Responsibility |
|------|--------|----------------|
| `web/src/messages/en.json` | Modify | Add `analytics.backToOverview` key |
| `web/src/messages/vi.json` | Modify | Add `analytics.backToOverview` key |
| `web/src/features/analytics/store-analytics.tsx` | Modify | Add `showBackButton` prop, back button in header |
| `web/src/app/[locale]/dashboard/analytics/[storeId]/page.tsx` | Modify | Pass `showBackButton={true}` |
| `web/src/features/analytics/period-stats.tsx` | Modify | Skeleton loading for `StorePeriodStats` |
| `web/src/features/analytics/summary-cards.tsx` | Modify | Skeleton loading for `SummaryCards` |
| `web/src/features/analytics/outcome-chart.tsx` | Modify | Skeleton loading for `OutcomeChart` |
| `web/src/features/analytics/wait-distribution-chart.tsx` | Modify | Skeleton loading for `WaitDistributionChart` |
| `web/src/features/analytics/peak-hours-chart.tsx` | Modify | Skeleton loading for `PeakHoursChart` |
| `web/src/features/analytics/throughput-chart.tsx` | Modify | Skeleton loading for `ThroughputChart` |
| `web/src/features/analytics/hourly-heatmap.tsx` | Modify | Skeleton loading for `HourlyHeatmap` |
| `web/src/features/analytics/overview-throughput-chart.tsx` | Modify | Skeleton loading for `OverviewThroughputChart` |
| `web/src/features/analytics/store-comparison-chart.tsx` | Modify | Skeleton loading for `StoreComparisonChart` |
| `web/src/features/analytics/store-wait-chart.tsx` | Modify | Skeleton loading for `StoreWaitChart` |
| `web/src/features/analytics/store-ranking-table.tsx` | Modify | Skeleton loading for `StoreRankingTable` |
| `docs/CHANGELOGS.md` | Modify | Log changes |

---

### Task 1: i18n — Add back button translation key

**Files:**
- Modify: `web/src/messages/en.json:345`
- Modify: `web/src/messages/vi.json:345`

- [ ] **Step 1: Add key to en.json**

In `web/src/messages/en.json`, inside the `"analytics"` object, after the `"storeAnalyticsWithName"` line (line 345), add:

```json
    "backToOverview": "Back to overview",
```

- [ ] **Step 2: Add key to vi.json**

In `web/src/messages/vi.json`, inside the `"analytics"` object, after the `"storeAnalyticsWithName"` line (line 345), add:

```json
    "backToOverview": "Quay lại tổng quan",
```

---

### Task 2: Back button — StoreAnalytics header

**Files:**
- Modify: `web/src/features/analytics/store-analytics.tsx`
- Modify: `web/src/app/[locale]/dashboard/analytics/[storeId]/page.tsx`

- [ ] **Step 1: Add `showBackButton` prop and back button to StoreAnalytics**

In `web/src/features/analytics/store-analytics.tsx`:

Add two imports at the top:

```tsx
import { ArrowLeft } from "lucide-react";
import { Link } from "@/i18n/navigation";
```

Update the interface:

```tsx
interface StoreAnalyticsProps {
  storeId: string;
  showBackButton?: boolean;
}
```

Update the destructuring:

```tsx
export function StoreAnalytics({ storeId, showBackButton }: StoreAnalyticsProps) {
```

Replace the `<h1>` block (lines 86–90):

```tsx
      <h1 className="text-xl font-bold l:text-2xl">
        {storeName
          ? t("storeAnalyticsWithName", { storeName })
          : t("storeAnalytics")}
      </h1>
```

With:

```tsx
      <div className="flex items-center gap-3">
        {showBackButton && (
          <Link
            href="/dashboard/analytics"
            aria-label={t("backToOverview")}
            className="flex size-8 items-center justify-center rounded-lg text-muted-foreground transition-colors hover:bg-muted hover:text-foreground"
          >
            <ArrowLeft aria-hidden="true" className="size-4" />
          </Link>
        )}
        <h1 className="text-xl font-bold l:text-2xl">
          {storeName
            ? t("storeAnalyticsWithName", { storeName })
            : t("storeAnalytics")}
        </h1>
      </div>
```

- [ ] **Step 2: Pass `showBackButton` from the drill-down page**

In `web/src/app/[locale]/dashboard/analytics/[storeId]/page.tsx`, change line 32:

```tsx
  return <StoreAnalytics storeId={params.storeId} />;
```

To:

```tsx
  return <StoreAnalytics storeId={params.storeId} showBackButton />;
```

- [ ] **Step 3: Verify the main analytics page does NOT pass the prop**

Open `web/src/app/[locale]/dashboard/analytics/page.tsx` and confirm line 26 is:

```tsx
  return <StoreAnalytics storeId={storeId} />;
```

No change needed — `showBackButton` defaults to `undefined` (falsy).

---

### Task 3: Skeleton — Period stat cards

**Files:**
- Modify: `web/src/features/analytics/period-stats.tsx`

- [ ] **Step 1: Add Skeleton import**

Add to imports in `web/src/features/analytics/period-stats.tsx`:

```tsx
import { Skeleton } from "@/components/ui/skeleton";
```

- [ ] **Step 2: Replace the StatCard loading state**

Replace the `StatCard` function (lines 129–155) with:

```tsx
function StatCard({
  label,
  value,
  loading,
  isText = false,
}: {
  label: string;
  value: number | string | null | undefined;
  loading: boolean;
  isText?: boolean;
}) {
  return (
    <div className="analytics-stat-card glass-card rounded-xl">
      {loading ? (
        <>
          <Skeleton className="h-7 w-16 rounded" />
          <Skeleton className="h-3 w-24 rounded" />
        </>
      ) : (
        <>
          <span className="analytics-stat-value text-foreground">
            {value != null ? (isText ? value : value.toLocaleString()) : "—"}
          </span>
          <span className="analytics-stat-label">{label}</span>
        </>
      )}
    </div>
  );
}
```

This removes the `loadingText` prop. Update the callers inside `OverviewPeriodStats` and `StorePeriodStats` — remove every `loadingText={tCommon("loading")}` prop from all `<StatCard>` calls, and remove the `const tCommon = useTranslations("common");` line from both functions since it's no longer used.

`OverviewPeriodStats` becomes:

```tsx
export function OverviewPeriodStats({
  data,
  period,
  loading,
}: OverviewPeriodStatsProps) {
  const t = useTranslations("analytics.periodStats");
  const periodLabel = usePeriodLabel(period);

  return (
    <div className="analytics-stats-grid">
      <StatCard
        label={t("storesActive")}
        value={!loading && data ? data.totalStores : null}
        loading={loading}
      />
      <StatCard
        label={t("issued", { period: periodLabel })}
        value={!loading && data ? data.totalIssued : null}
        loading={loading}
      />
      <StatCard
        label={t("completed", { period: periodLabel })}
        value={!loading && data ? data.totalCompleted : null}
        loading={loading}
      />
      <StatCard
        label={t("avgWait")}
        value={!loading && data ? formatWait(data.avgWaitSeconds) : null}
        loading={loading}
        isText
      />
    </div>
  );
}
```

`StorePeriodStats` follows the same pattern — remove `tCommon` and all `loadingText` props.

---

### Task 4: Skeleton — Summary cards

**Files:**
- Modify: `web/src/features/analytics/summary-cards.tsx`

- [ ] **Step 1: Add Skeleton import**

Add to imports in `web/src/features/analytics/summary-cards.tsx`:

```tsx
import { Skeleton } from "@/components/ui/skeleton";
```

- [ ] **Step 2: Replace the `loading` early return**

Replace lines 30–36:

```tsx
export function SummaryCards({ summary, loading }: SummaryCardsProps) {
  const t = useTranslations("analytics.summary");
  const tAnalytics = useTranslations("analytics");

  if (loading) return null;

  if (!summary) {
```

With:

```tsx
export function SummaryCards({ summary, loading }: SummaryCardsProps) {
  const t = useTranslations("analytics.summary");
  const tAnalytics = useTranslations("analytics");

  if (loading) {
    return (
      <div className="glass-card rounded-xl p-4 l:p-5">
        <Skeleton className="mb-3 h-4 w-20 l:mb-4" />
        <div className="grid grid-cols-2 gap-3 l:grid-cols-4 l:gap-4">
          {Array.from({ length: 8 }, (_, i) => (
            <div key={i}>
              <Skeleton className="mb-1 h-3 w-20" />
              <Skeleton className="h-6 w-14" />
            </div>
          ))}
        </div>
      </div>
    );
  }

  if (!summary) {
```

---

### Task 5: Skeleton — Chart components (4 charts)

All four chart components share the same skeleton pattern: glass-card with a title skeleton + a tall body skeleton matching chart height. Apply the same change to each.

**Files:**
- Modify: `web/src/features/analytics/outcome-chart.tsx`
- Modify: `web/src/features/analytics/wait-distribution-chart.tsx`
- Modify: `web/src/features/analytics/peak-hours-chart.tsx`
- Modify: `web/src/features/analytics/throughput-chart.tsx`

- [ ] **Step 1: Add Skeleton import to all four files**

Add to imports in each of the four files:

```tsx
import { Skeleton } from "@/components/ui/skeleton";
```

- [ ] **Step 2: Replace loading fallback in OutcomeChart**

In `web/src/features/analytics/outcome-chart.tsx`, replace lines 41–44:

```tsx
      {loading || total === 0 ? (
        <p className="py-8 text-center text-sm text-muted-foreground">
          {tAnalytics("noData")}
        </p>
```

With:

```tsx
      {loading ? (
        <Skeleton className="h-56 w-full rounded l:h-64" />
      ) : total === 0 ? (
        <p className="py-8 text-center text-sm text-muted-foreground">
          {tAnalytics("noData")}
        </p>
```

This separates the `loading` state (skeleton) from the `no data` state (text fallback). The closing `)` and rest of the JSX remains unchanged.

- [ ] **Step 3: Replace loading fallback in WaitDistributionChart**

In `web/src/features/analytics/wait-distribution-chart.tsx`, replace lines 34–38:

```tsx
      {loading || !hasData ? (
        <p className="py-8 text-center text-sm text-muted-foreground">
          {tAnalytics("noData")}
        </p>
```

With:

```tsx
      {loading ? (
        <Skeleton className="h-56 w-full rounded l:h-64" />
      ) : !hasData ? (
        <p className="py-8 text-center text-sm text-muted-foreground">
          {tAnalytics("noData")}
        </p>
```

- [ ] **Step 4: Replace loading fallback in PeakHoursChart**

In `web/src/features/analytics/peak-hours-chart.tsx`, replace lines 41–44:

```tsx
      {loading || !data ? (
        <p className="py-8 text-center text-sm text-muted-foreground">
          {tAnalytics("noData")}
        </p>
```

With:

```tsx
      {loading ? (
        <Skeleton className="h-56 w-full rounded l:h-64" />
      ) : !data ? (
        <p className="py-8 text-center text-sm text-muted-foreground">
          {tAnalytics("noData")}
        </p>
```

- [ ] **Step 5: Replace loading fallback in ThroughputChart**

In `web/src/features/analytics/throughput-chart.tsx`, replace lines 35–38:

```tsx
      {loading || !data || data.days.length === 0 ? (
        <p className="py-8 text-center text-sm text-muted-foreground">
          {tAnalytics("noData")}
        </p>
```

With:

```tsx
      {loading ? (
        <Skeleton className="h-56 w-full rounded l:h-64" />
      ) : !data || data.days.length === 0 ? (
        <p className="py-8 text-center text-sm text-muted-foreground">
          {tAnalytics("noData")}
        </p>
```

---

### Task 6: Skeleton — Hourly heatmap

**Files:**
- Modify: `web/src/features/analytics/hourly-heatmap.tsx`

- [ ] **Step 1: Add Skeleton import**

Add to imports in `web/src/features/analytics/hourly-heatmap.tsx`:

```tsx
import { Skeleton } from "@/components/ui/skeleton";
```

- [ ] **Step 2: Replace loading fallback**

Replace lines 52–55:

```tsx
      {loading || !hasData ? (
        <p className="py-8 text-center text-sm text-muted-foreground">
          {tAnalytics("noData")}
        </p>
```

With:

```tsx
      {loading ? (
        <Skeleton className="h-48 w-full rounded" />
      ) : !hasData ? (
        <p className="py-8 text-center text-sm text-muted-foreground">
          {tAnalytics("noData")}
        </p>
```

---

### Task 7: Changelog

**Files:**
- Modify: `docs/CHANGELOGS.md`

- [ ] **Step 1: Add changelog entry**

Add a new entry at the top of the changelog (below the header) in `docs/CHANGELOGS.md`:

```markdown
## 2026-05-19 — Analytics Back Button & Skeleton Loading

**Back button (SUPER_ADMIN only)**
- Added `showBackButton` prop to `StoreAnalytics` component
- `[storeId]/page.tsx` passes `showBackButton` for SUPER_ADMIN drill-down
- Uses same ArrowLeft icon-link pattern as devices page
- Links back to `/dashboard/analytics` (the overview)
- Regular admins viewing their own store analytics do not see the button

**Skeleton loading**
- Replaced all "Loading" text / `null` / "No Data" loading fallbacks with section-shaped skeletons
- `StorePeriodStats`: 4 glass-card skeletons with value+label pulse lines
- `SummaryCards`: glass-card with title skeleton + 2×4 grid of skeleton pairs
- `OutcomeChart`, `WaitDistributionChart`, `PeakHoursChart`, `ThroughputChart`: glass-card with full-height skeleton block
- `HourlyHeatmap`: glass-card with wide skeleton block
- All use existing shadcn `Skeleton` component (`animate-pulse bg-muted`)
- Loading and no-data states are now separate (skeleton vs "No Data" text)

**i18n**
- Added `analytics.backToOverview`: EN "Back to overview" / VI "Quay lại tổng quan"
```

---

### Task 8: Skeleton — Overview charts (OverviewThroughputChart, StoreComparisonChart, StoreWaitChart)

**Files:**
- Modify: `web/src/features/analytics/overview-throughput-chart.tsx`
- Modify: `web/src/features/analytics/store-comparison-chart.tsx`
- Modify: `web/src/features/analytics/store-wait-chart.tsx`

All three follow the same pattern as the per-store charts: separate `loading` from `no-data`.

- [ ] **Step 1: Add Skeleton import to all three files**

Add to imports in each file:

```tsx
import { Skeleton } from "@/components/ui/skeleton";
```

- [ ] **Step 2: Replace loading fallback in OverviewThroughputChart**

In `overview-throughput-chart.tsx`, replace:

```tsx
      {loading || !data || data.days.length === 0 ? (
        <p className="py-8 text-center text-sm text-muted-foreground">
          {tAnalytics("noData")}
        </p>
```

With:

```tsx
      {loading ? (
        <Skeleton className="h-56 w-full rounded l:h-64" />
      ) : !data || data.days.length === 0 ? (
        <p className="py-8 text-center text-sm text-muted-foreground">
          {tAnalytics("noData")}
        </p>
```

- [ ] **Step 3: Replace loading fallback in StoreComparisonChart**

In `store-comparison-chart.tsx`, replace:

```tsx
      {loading || chartData.length === 0 ? (
        <p className="py-8 text-center text-sm text-muted-foreground">
          {tAnalytics("noData")}
        </p>
```

With:

```tsx
      {loading ? (
        <Skeleton className="h-56 w-full rounded" />
      ) : chartData.length === 0 ? (
        <p className="py-8 text-center text-sm text-muted-foreground">
          {tAnalytics("noData")}
        </p>
```

- [ ] **Step 4: Replace loading fallback in StoreWaitChart**

In `store-wait-chart.tsx`, replace:

```tsx
      {loading || chartData.length === 0 ? (
        <p className="py-8 text-center text-sm text-muted-foreground">
          {tAnalytics("noData")}
        </p>
```

With:

```tsx
      {loading ? (
        <Skeleton className="h-56 w-full rounded" />
      ) : chartData.length === 0 ? (
        <p className="py-8 text-center text-sm text-muted-foreground">
          {tAnalytics("noData")}
        </p>
```

---

### Task 9: Skeleton — StoreRankingTable

**Files:**
- Modify: `web/src/features/analytics/store-ranking-table.tsx`

- [ ] **Step 1: Add Skeleton import**

Add to imports:

```tsx
import { Skeleton } from "@/components/ui/skeleton";
```

- [ ] **Step 2: Replace the loading early return**

Replace the existing loading block (lines 65–74):

```tsx
  if (loading) {
    return (
      <div className="glass-card rounded-xl p-4 l:p-5">
        <h3 className="mb-3 text-sm font-semibold text-foreground">
          {t("title")}
        </h3>
        <p className="text-sm text-muted-foreground">{tAnalytics("noData")}</p>
      </div>
    );
  }
```

With:

```tsx
  if (loading) {
    return (
      <div className="glass-card rounded-xl p-4 l:p-5">
        <div className="mb-3 flex items-center justify-between gap-2 l:mb-4">
          <Skeleton className="h-4 w-28" />
          <Skeleton className="h-7 w-32 rounded" />
        </div>
        <div className="space-y-3">
          <Skeleton className="h-3 w-full" />
          {Array.from({ length: 3 }, (_, idx) => (
            // biome-ignore lint/suspicious/noArrayIndexKey: static skeleton placeholders never reorder
            <Skeleton key={idx} className="h-8 w-full rounded" />
          ))}
        </div>
      </div>
    );
  }
```

