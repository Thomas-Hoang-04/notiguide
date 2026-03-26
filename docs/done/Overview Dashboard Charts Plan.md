# Overview Dashboard Charts Plan

Add summary charts to the main Overview page (`/dashboard`) for both Super Admin and Admin roles. These give at-a-glance insight without navigating to the full Analytics tab.

---

## Current State

### Super Admin Overview (`/dashboard`)

1. **RealtimeStats** — 4 live-polled cards (Active Stores, Issued Today, Serving, Avg Wait)
2. **OverviewThroughputChart** — line chart (issued + completed, hardcoded WEEK)
3. **StoreRankingTable** — sortable table with "View" links
4. "View full analytics →" link

### Admin Overview (`/dashboard`)

1. **RealtimeStats** — 4 live-polled cards (Queue Now, Serving Now, Issued Today, Avg Wait)
2. **ThroughputChart** — line chart (issued + completed, hardcoded WEEK)
3. "View full analytics →" link

---

## Changes

### Super Admin Overview — Add 2 charts

**Charts**: `StoreComparisonChart` (cancel/skip rate) + `StoreWaitChart` (avg wait by store)

**Why these two**: They answer the two most actionable Super Admin questions: "which stores are losing customers?" and "which stores are too slow?" — without any new API calls (data already in `OverviewResponse.stores`).

#### Step 1 — Import and wire into `page.tsx`

**File**: `web/src/app/[locale]/dashboard/page.tsx`

Add imports:

```tsx
import { StoreComparisonChart } from "@/features/analytics/store-comparison-chart";
import { StoreWaitChart } from "@/features/analytics/store-wait-chart";
```

In `SuperAdminDashboardCharts`, add between `OverviewThroughputChart` and `StoreRankingTable`:

```tsx
<div className="grid gap-4 l:grid-cols-2 l:gap-6">
  <StoreComparisonChart stores={overview?.stores ?? []} loading={loading} />
  <StoreWaitChart stores={overview?.stores ?? []} loading={loading} />
</div>
```

#### Step 2 — Final Super Admin Overview layout

```
┌──────────────────────────────────────┐
│ Overview                             │
├──────────────────────────────────────┤
│ [RealtimeStats — 4 live cards]       │
├──────────────────────────────────────┤
│ Daily Throughput [▼ All Stores]      │
│ ┌────────────────────────────────┐   │
│ │  📈 Line chart (WEEK)          │   │
│ └────────────────────────────────┘   │
├──────────────┬───────────────────────┤
│ Cancel/Skip  │ Avg Wait by Store     │  ← NEW
│ Rate by Store│ ┌─────────────────┐   │  ← NEW
│ ┌──────────┐ │ │  ▬ horizontal   │   │  ← NEW
│ │ ▬▬ horiz │ │ │    bars         │   │  ← NEW
│ └──────────┘ │ └─────────────────┘   │  ← NEW
├──────────────┴───────────────────────┤
│ Store Rankings [▼ Sort: Most Issued] │
│ ┌────────────────────────────────┐   │
│ │  📋 Table                      │   │
│ └────────────────────────────────┘   │
├──────────────────────────────────────┤
│ View full analytics →                │
└──────────────────────────────────────┘
```

**No new backend work, translations, or types needed** — both components and all their dependencies were added in the Analytics Charts Expansion.

---

### Admin Overview — Add 2 charts

**Charts**: `OutcomeChart` (ticket outcomes donut) + `PeakHoursChart` (tickets by hour)

**Why these two**: The donut gives a quick health pulse ("am I losing too many customers?"), and peak hours is the most operationally useful chart for planning staffing and breaks.

#### Step 3 — Add API calls to `AdminDashboardChart`

**File**: `web/src/app/[locale]/dashboard/page.tsx`

Add imports:

```tsx
import { getPeakHours, getStoreSummary } from "@/features/analytics/api";
import { OutcomeChart } from "@/features/analytics/outcome-chart";
import { PeakHoursChart } from "@/features/analytics/peak-hours-chart";
import type {
  PeakHoursResponse,
  StoreSummaryResponse,
} from "@/features/analytics/types";
```

Rename `AdminDashboardChart` to `AdminDashboardCharts` (plural, since it now renders multiple charts).

Add state and fetch calls:

```tsx
function AdminDashboardCharts({ storeId }: { storeId: string }) {
  const [throughput, setThroughput] = useState<DailyThroughputResponse | null>(null);
  const [summary, setSummary] = useState<StoreSummaryResponse | null>(null);
  const [peakHours, setPeakHours] = useState<PeakHoursResponse | null>(null);
  const [loading, setLoading] = useState(true);

  const fetchData = useCallback(async () => {
    try {
      const [throughputRes, summaryRes, peakRes] = await Promise.all([
        getDailyThroughput(storeId, "WEEK"),
        getStoreSummary(storeId, "WEEK"),
        getPeakHours(storeId, "WEEK"),
      ]);
      setThroughput(throughputRes);
      setSummary(summaryRes);
      setPeakHours(peakRes);
    } catch {
      // handled by api layer
    } finally {
      setLoading(false);
    }
  }, [storeId]);

  useEffect(() => {
    void fetchData();
  }, [fetchData]);

  return (
    <>
      <ThroughputChart data={throughput} loading={loading} />
      <div className="grid gap-4 l:grid-cols-2 l:gap-6">
        <OutcomeChart summary={summary} loading={loading} />
        <PeakHoursChart data={peakHours} loading={loading} />
      </div>
    </>
  );
}
```

Update the render call:

```tsx
{storeId && <AdminDashboardCharts storeId={storeId} />}
```

#### Step 4 — Final Admin Overview layout

```
┌──────────────────────────────────────┐
│ Store Name                           │
├──────────────────────────────────────┤
│ [RealtimeStats — 4 live cards]       │
├──────────────────────────────────────┤
│ Daily Throughput                     │
│ ┌────────────────────────────────┐   │
│ │  📈 Line chart (WEEK)          │   │
│ └────────────────────────────────┘   │
├──────────────┬───────────────────────┤
│ Ticket       │ Tickets by Hour       │  ← NEW
│ Outcomes     │ ┌─────────────────┐   │  ← NEW
│ ┌──────────┐ │ │  📊 Bar chart   │   │  ← NEW
│ │  🍩 Donut│ │ │                 │   │  ← NEW
│ └──────────┘ │ └─────────────────┘   │  ← NEW
├──────────────┴───────────────────────┤
│ View full analytics →                │
└──────────────────────────────────────┘
```

**No new backend work, translations, or types needed** — both components and all their dependencies already exist.

---

## Implementation Order


| Step | What                                                             | Backend?                | Effort |
| ---- | ---------------------------------------------------------------- | ----------------------- | ------ |
| 1    | Super Admin Overview — add StoreComparisonChart + StoreWaitChart | No                      | Small  |
| 2    | Admin Overview — add OutcomeChart + PeakHoursChart + API calls   | No (existing endpoints) | Small  |


Both steps modify only `web/src/app/[locale]/dashboard/page.tsx`. No new components, translations, endpoints, or types.

---

## CHANGELOGS.md Entry

```
### Overview Dashboard Charts
- **New**: Cancel & Skip Rate + Avg Wait Time charts on Super Admin Overview (data from existing OverviewResponse)
- **New**: Ticket Outcomes donut + Peak Hours chart on Admin Overview (new API calls to existing endpoints)
- **Updated**: Admin dashboard fetches `getStoreSummary` + `getPeakHours` alongside `getDailyThroughput`
- **Note**: Store Ranking "View" button → `/dashboard/analytics/:storeId` drill-down already functional (no changes)
```

