# Analytics Charts Expansion Plan

Additional charts for the Overview dashboard (Super Admin) and Store Analytics detail page. Built on top of existing analytics infrastructure (TimescaleDB hypertable, continuous aggregates, existing API layer).

---

## Current State

### Overview Dashboard (`analytics-overview.tsx`) — Super Admin
1. **DateRangePicker** — period selector (TODAY / WEEK / MONTH / QUARTER / custom range)
2. **OverviewPeriodStats** — 4 stat cards (Stores Active, Issued, Completed, Avg Wait)
3. **OverviewThroughputChart** — line chart (issued & completed daily, store dropdown)
4. **StoreRankingTable** — sortable table (by issued, wait, cancel rate)

### Store Analytics (`store-analytics.tsx`) — per-store detail
1. **DateRangePicker** — period selector
2. **StorePeriodStats** — 4 stat cards (Issued, Completed, Cancelled, Avg Wait)
3. **SummaryCards** — 8 metric cards (totals, averages, rates, peak hour)
4. **PeakHoursChart** — bar chart (avg tickets per hour, 0–23)
5. **ThroughputChart** — line chart (issued & completed daily)

### Available Data (already collected)
- `analytics_event` table: `TICKET_ISSUED`, `TICKET_CALLED`, `TICKET_COMPLETED`, `TICKET_CANCELLED`, `TICKET_SKIPPED`, `DEVICE_TRIGGERED`
- Each event has: `time`, `store_id`, `event_type`, `ticket_id`, `wait_duration_seconds`, `device_id`, `metadata` (JSONB with `ticket_number`, `counter_id`, `previous_status`)
- `StoreAnalyticsSummary` per store: `issued`, `completed`, `cancelled`, `skipped`, `avgWaitSeconds`
- `StoreSummaryResponse`: all counts + `avgWaitSeconds`, `avgServiceSeconds`, `medianWaitSeconds`, `peakHour`, `cancelRate`, `skipRate`

---

## New Charts

### Chart 1: Store Comparison — Cancel & Skip Rate (Overview)

**What it shows**: Horizontal bar chart comparing each store's cancel rate and skip rate side by side. Surfaces problem stores — high cancel rate → wait times too long; high skip rate → notification issues.

**Backend work**: None. Data already in `OverviewResponse.stores[]` — compute `cancelRate = cancelled / issued * 100` and `skipRate = skipped / issued * 100` on the frontend (same as `StoreRankingTable` already does).

**Frontend**:

#### Step 1.1 — New component: `store-comparison-chart.tsx`

**File**: `web/src/features/analytics/store-comparison-chart.tsx`

```tsx
"use client";

import { useTranslations } from "next-intl";
import {
  Bar,
  BarChart,
  CartesianGrid,
  Legend,
  ResponsiveContainer,
  Tooltip,
  XAxis,
  YAxis,
} from "recharts";
import type { StoreAnalyticsSummary } from "./types";

interface StoreComparisonChartProps {
  stores: StoreAnalyticsSummary[];
  loading: boolean;
}

export function StoreComparisonChart({ stores, loading }: StoreComparisonChartProps) {
  const t = useTranslations("analytics.storeComparison");
  const tAnalytics = useTranslations("analytics");

  // chart data: store name + cancel % + skip %
  const chartData = stores
    .filter((s) => s.issued > 0)
    .map((s) => ({
      name: s.storeName,
      cancelRate: Number(((s.cancelled / s.issued) * 100).toFixed(1)),
      skipRate: Number(((s.skipped / s.issued) * 100).toFixed(1)),
    }));

  // recharts horizontal bar chart (layout="vertical")
  // XAxis type="number" (the %) → YAxis type="category" (store names)
  // Two Bar series: cancelRate (warm color) and skipRate (muted color)
  // Tooltip shows "Store: XX%, YY%"
  // glass-card wrapper matching existing chart style
}
```

**Key design decisions**:
- `layout="vertical"` on BarChart for horizontal bars — store names on Y-axis, percentages on X-axis
- Sort stores by total `cancelRate + skipRate` descending so worst performers appear at top
- If only 1 store exists, still show it (chart degrades gracefully)
- Uses `var(--chart-accent)` for cancel rate, `var(--chart-muted)` (or `var(--destructive)` / `var(--warning)`) for skip rate — check `Web Styles.md` for palette
- Max 10 stores displayed to keep chart readable; if more, show top 10 by combined rate

#### Step 1.2 — Add to overview layout

**File**: `web/src/features/analytics/analytics-overview.tsx`

Insert between `OverviewThroughputChart` and `StoreRankingTable`:

```tsx
<StoreComparisonChart stores={overview?.stores ?? []} loading={loading} />
```

#### Step 1.3 — Translation keys

**EN** (`web/src/messages/en.json`) — add under `analytics`:
```json
"storeComparison": {
  "title": "Cancel & Skip Rate by Store",
  "cancelRate": "Cancel Rate",
  "skipRate": "Skip Rate"
}
```

**VI** (`web/src/messages/vi.json`):
```json
"storeComparison": {
  "title": "Tỷ lệ hủy & bỏ lỡ theo cửa hàng",
  "cancelRate": "Tỷ lệ hủy",
  "skipRate": "Tỷ lệ bỏ lỡ"
}
```

---

### Chart 2: Avg Wait Time by Store (Overview)

**What it shows**: Horizontal bar chart comparing average wait times across stores. Identifies underperforming stores or staffing imbalances at a glance.

**Backend work**: None. Data already in `OverviewResponse.stores[].avgWaitSeconds`.

**Frontend**:

#### Step 2.1 — New component: `store-wait-chart.tsx`

**File**: `web/src/features/analytics/store-wait-chart.tsx`

```tsx
"use client";

import { useTranslations } from "next-intl";
import {
  Bar,
  BarChart,
  CartesianGrid,
  ResponsiveContainer,
  Tooltip,
  XAxis,
  YAxis,
} from "recharts";
import type { StoreAnalyticsSummary } from "./types";

interface StoreWaitChartProps {
  stores: StoreAnalyticsSummary[];
  loading: boolean;
}

export function StoreWaitChart({ stores, loading }: StoreWaitChartProps) {
  const t = useTranslations("analytics.storeWait");
  const tAnalytics = useTranslations("analytics");

  // chart data: store name + avgWaitMinutes (convert seconds → minutes for readability)
  const chartData = stores
    .filter((s) => s.avgWaitSeconds != null)
    .map((s) => ({
      name: s.storeName,
      avgWait: Number((s.avgWaitSeconds! / 60).toFixed(1)),
    }))
    .sort((a, b) => b.avgWait - a.avgWait);

  // layout="vertical" horizontal BarChart, single Bar series
  // XAxis label shows minutes (e.g., "5.2 min")
  // Tooltip formatter: "{value} min"
  // Single color: var(--chart-accent)
}
```

**Key design decisions**:
- Convert `avgWaitSeconds` → minutes for human readability
- Sort descending (longest wait at top)
- Stores with no wait data (`avgWaitSeconds === null`) are excluded
- Tooltip shows `"X.X min"` format
- Same max-10 cap as Chart 1

#### Step 2.2 — Add to overview layout

**File**: `web/src/features/analytics/analytics-overview.tsx`

Place Charts 1 and 2 in a 2-column grid below the throughput chart:

```tsx
<div className="grid gap-4 l:grid-cols-2 l:gap-6">
  <StoreComparisonChart stores={overview?.stores ?? []} loading={loading} />
  <StoreWaitChart stores={overview?.stores ?? []} loading={loading} />
</div>
```

Final overview layout order:
1. DateRangePicker
2. OverviewPeriodStats (4 stat cards)
3. OverviewThroughputChart (line chart with store dropdown)
4. **StoreComparisonChart + StoreWaitChart** (2-column grid) ← NEW
5. StoreRankingTable

#### Step 2.3 — Translation keys

**EN**:
```json
"storeWait": {
  "title": "Avg Wait Time by Store",
  "avgWait": "Avg Wait",
  "minutes": "{value} min"
}
```

**VI**:
```json
"storeWait": {
  "title": "Thời gian chờ TB theo cửa hàng",
  "avgWait": "Chờ TB",
  "minutes": "{value} phút"
}
```

---

### Chart 3: Outcome Breakdown — Donut Chart (Store Detail)

**What it shows**: Donut/pie chart showing the proportion of Completed vs Cancelled vs Skipped tickets. Quick health check of a store's queue efficiency.

**Backend work**: None. Data already in `StoreSummaryResponse.totalCompleted / totalCancelled / totalSkipped`.

**Frontend**:

#### Step 3.1 — New component: `outcome-chart.tsx`

**File**: `web/src/features/analytics/outcome-chart.tsx`

```tsx
"use client";

import { useTranslations } from "next-intl";
import { Cell, Legend, Pie, PieChart, ResponsiveContainer, Tooltip } from "recharts";
import type { StoreSummaryResponse } from "./types";

interface OutcomeChartProps {
  summary: StoreSummaryResponse | null;
  loading: boolean;
}

export function OutcomeChart({ summary, loading }: OutcomeChartProps) {
  const t = useTranslations("analytics.outcome");
  const tAnalytics = useTranslations("analytics");

  // Three segments: Completed, Cancelled, Skipped
  // Colors: Completed → var(--primary) or green-ish
  //         Cancelled → var(--destructive) or warm
  //         Skipped → var(--warning) or muted
  //
  // recharts PieChart with <Pie innerRadius="60%" outerRadius="80%"> for donut style
  // Center label showing total tickets
  // Legend below the chart
  // Tooltip shows count + percentage
  //
  // If all values are 0, show noData placeholder
}
```

**Key design decisions**:
- Donut style (`innerRadius={60} outerRadius={80}`) not solid pie — cleaner look
- Center text label showing total count (Completed + Cancelled + Skipped)
- 3 segments only (not Issued — Issued = sum of all outcomes + still-waiting, which is confusing on a donut)
- Custom tooltip showing both absolute count and percentage
- Responsive: `<ResponsiveContainer>` with `initialDimension={{ width: 1, height: 1 }}`

#### Step 3.2 — Add to store detail layout

**File**: `web/src/features/analytics/store-analytics.tsx`

Change the 2-column grid to include OutcomeChart:

```tsx
{/* Before (2 charts): */}
<div className="grid gap-4 l:grid-cols-2 l:gap-6">
  <PeakHoursChart ... />
  <ThroughputChart ... />
</div>

{/* After (3 charts — outcome donut + peak hours on row 1, throughput full-width on row 2): */}
<div className="grid gap-4 l:grid-cols-2 l:gap-6">
  <OutcomeChart summary={summary} loading={loading} />
  <PeakHoursChart data={peakHours} loading={loading} />
</div>
<ThroughputChart data={throughput} loading={loading} />
```

Alternative layout (if outcome chart is too small in half-width): place it above the 2-column grid as a standalone card. Decide during implementation based on visual balance.

#### Step 3.3 — Translation keys

**EN**:
```json
"outcome": {
  "title": "Ticket Outcomes",
  "completed": "Completed",
  "cancelled": "Cancelled",
  "skipped": "Skipped",
  "total": "Total"
}
```

**VI**:
```json
"outcome": {
  "title": "Kết quả vé",
  "completed": "Hoàn thành",
  "cancelled": "Đã hủy",
  "skipped": "Bỏ lỡ",
  "total": "Tổng"
}
```

---

### Chart 4: Wait Time Distribution — Histogram (Store Detail)

**What it shows**: Histogram showing how wait times are distributed in buckets (0–5 min, 5–10 min, 10–15 min, 15–20 min, 20+ min). More informative than just "avg wait" — reveals whether most customers are served quickly with a few outliers, or if wait times are uniformly long.

**Backend work**: **New endpoint + repository method required**.

#### Step 4.1 — Backend: New DTO

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/analytics/dto/AnalyticsDtos.kt`

Add:
```kotlin
data class WaitDistributionResponse(
    val period: String,
    val buckets: List<WaitBucket>
)

data class WaitBucket(
    val label: String,    // e.g. "0-5", "5-10", "20+"
    val minMinutes: Int,  // inclusive lower bound
    val maxMinutes: Int?, // exclusive upper bound, null = unbounded
    val count: Long
)
```

#### Step 4.2 — Backend: New repository method

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/analytics/repository/AnalyticsEventRepository.kt`

Add row type:
```kotlin
data class WaitBucketRow(
    val bucket: Int,  // 0, 1, 2, 3, 4 (corresponding to 0-5, 5-10, 10-15, 15-20, 20+)
    val count: Long
)
```

Add method:
```kotlin
suspend fun getWaitDistribution(storeId: UUID, from: OffsetDateTime, to: OffsetDateTime): List<WaitBucketRow> {
    return client.sql("""
        SELECT
            CASE
                WHEN wait_duration_seconds < 300 THEN 0
                WHEN wait_duration_seconds < 600 THEN 1
                WHEN wait_duration_seconds < 900 THEN 2
                WHEN wait_duration_seconds < 1200 THEN 3
                ELSE 4
            END AS bucket,
            COUNT(*) AS count
        FROM analytics_event
        WHERE store_id = :storeId
          AND time >= :from AND time < :to
          AND event_type = 'TICKET_CALLED'
          AND wait_duration_seconds IS NOT NULL
        GROUP BY bucket
        ORDER BY bucket
    """.trimIndent())
    .bind("storeId", storeId)
    .bind("from", from)
    .bind("to", to)
    .map { row: Readable, _ ->
        WaitBucketRow(
            bucket = row.get("bucket", Int::class.javaObjectType) ?: 0,
            count = row.get("count", Long::class.javaObjectType) ?: 0L
        )
    }
    .all()
    .asFlow()
    .toList()
}
```

**Why `TICKET_CALLED`**: The `wait_duration_seconds` on `TICKET_CALLED` events represents queue wait time (time from issued to called). This is what customers experience. `TICKET_COMPLETED` events' `wait_duration_seconds` represents service time (called → completed), which is a different metric.

#### Step 4.3 — Backend: Service method

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/analytics/service/AnalyticsQueryService.kt`

Add:
```kotlin
suspend fun getWaitDistribution(storeId: UUID, period: Period): WaitDistributionResponse {
    val (from, to) = periodToRange(period)
    val rows = analyticsEventRepository.getWaitDistribution(storeId, from, to)

    val labels = listOf("0-5", "5-10", "10-15", "15-20", "20+")
    val bounds = listOf(0 to 5, 5 to 10, 10 to 15, 15 to 20, 20 to null)
    val rowMap = rows.associateBy { it.bucket }

    val buckets = labels.mapIndexed { i, label ->
        WaitBucket(
            label = label,
            minMinutes = bounds[i].first,
            maxMinutes = bounds[i].second,
            count = rowMap[i]?.count ?: 0L
        )
    }

    return WaitDistributionResponse(
        period = period.name.lowercase(),
        buckets = buckets
    )
}
```

#### Step 4.4 — Backend: Controller endpoint

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/analytics/controller/AnalyticsController.kt`

Add:
```kotlin
@GetMapping("/{storeId}/wait-distribution")
suspend fun getWaitDistribution(
    @PathVariable storeId: UUID,
    @RequestParam period: Period,
    @AuthenticationPrincipal principal: AdminPrincipal
): ResponseEntity<WaitDistributionResponse> {
    StoreAccessUtil.requireStoreAccess(principal, storeId)
    return ResponseEntity.ok(analyticsQueryService.getWaitDistribution(storeId, period))
}
```

#### Step 4.5 — Frontend: API + types

**File**: `web/src/features/analytics/types.ts`

Add:
```typescript
export interface WaitDistributionResponse {
  period: string;
  buckets: WaitBucket[];
}

export interface WaitBucket {
  label: string;
  minMinutes: number;
  maxMinutes: number | null;
  count: number;
}
```

**File**: `web/src/lib/constants.ts`

Add route:
```typescript
WAIT_DISTRIBUTION: (storeId: string, period: string) =>
  `/api/analytics/${storeId}/wait-distribution?period=${period}`,
```

**File**: `web/src/features/analytics/api.ts`

Add:
```typescript
export function getWaitDistribution(storeId: string, period: PeriodOrRange) {
  return get<WaitDistributionResponse>(
    API_ROUTES.ANALYTICS.WAIT_DISTRIBUTION(storeId, periodParam(period)),
  );
}
```

#### Step 4.6 — Frontend: New component `wait-distribution-chart.tsx`

**File**: `web/src/features/analytics/wait-distribution-chart.tsx`

```tsx
"use client";

import { useTranslations } from "next-intl";
import {
  Bar,
  BarChart,
  CartesianGrid,
  ResponsiveContainer,
  Tooltip,
  XAxis,
  YAxis,
} from "recharts";
import type { WaitDistributionResponse } from "./types";

interface WaitDistributionChartProps {
  data: WaitDistributionResponse | null;
  loading: boolean;
}

export function WaitDistributionChart({ data, loading }: WaitDistributionChartProps) {
  const t = useTranslations("analytics.waitDistribution");
  const tAnalytics = useTranslations("analytics");

  // Vertical bar chart (standard layout)
  // X-axis: bucket labels ("0-5 min", "5-10 min", etc.)
  // Y-axis: count of tickets
  // Single Bar series with var(--chart-accent) color
  // Tooltip shows exact count
  //
  // Label formatter on X-axis: append " min" to each label
  // glass-card wrapper
}
```

#### Step 4.7 — Add to store detail layout

**File**: `web/src/features/analytics/store-analytics.tsx`

Add `getWaitDistribution` to the `Promise.all` fetch:
```tsx
const [summaryRes, peakRes, throughputRes, waitDistRes] = await Promise.all([
  getStoreSummary(storeId, p),
  getPeakHours(storeId, p),
  getDailyThroughput(storeId, p),
  getWaitDistribution(storeId, p),
]);
```

Add state:
```tsx
const [waitDist, setWaitDist] = useState<WaitDistributionResponse | null>(null);
```

Place in layout — 2×2 grid with the other charts:
```tsx
<div className="grid gap-4 l:grid-cols-2 l:gap-6">
  <OutcomeChart summary={summary} loading={loading} />
  <WaitDistributionChart data={waitDist} loading={loading} />
  <PeakHoursChart data={peakHours} loading={loading} />
  <ThroughputChart data={throughput} loading={loading} />
</div>
```

#### Step 4.8 — Translation keys

**EN**:
```json
"waitDistribution": {
  "title": "Wait Time Distribution",
  "tickets": "Tickets",
  "minutes": "min"
}
```

**VI**:
```json
"waitDistribution": {
  "title": "Phân bố thời gian chờ",
  "tickets": "Số vé",
  "minutes": "phút"
}
```

---

### Chart 5: Hourly Heatmap — Day × Hour (Store Detail)

**What it shows**: A 7×24 grid heatmap where rows are days of the week (Mon–Sun), columns are hours (0–23), and cell color intensity represents ticket volume. Reveals recurring patterns like "Saturdays at 2 PM are the busiest" — much richer than the current flat "Tickets by Hour" bar chart.

**Backend work**: **New endpoint + repository method required**.

#### Step 5.1 — Backend: New DTO

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/analytics/dto/AnalyticsDtos.kt`

Add:
```kotlin
data class HourlyHeatmapResponse(
    val range: String,
    val cells: List<HeatmapCell>
)

data class HeatmapCell(
    val dayOfWeek: Int,  // 1 = Monday, 7 = Sunday (ISO)
    val hour: Int,       // 0–23
    val avgTickets: Double
)
```

#### Step 5.2 — Backend: New repository method

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/analytics/repository/AnalyticsEventRepository.kt`

Add row type:
```kotlin
data class HeatmapRow(
    val dayOfWeek: Int,
    val hour: Int,
    val avgTickets: Double
)
```

Add method:
```kotlin
suspend fun getHourlyHeatmap(storeId: UUID, from: OffsetDateTime, to: OffsetDateTime): List<HeatmapRow> {
    val weekCount = Duration.between(from, to).toDays().coerceAtLeast(1) / 7.0

    return client.sql("""
        SELECT
            EXTRACT(ISODOW FROM time AT TIME ZONE 'Asia/Ho_Chi_Minh')::int AS dow,
            EXTRACT(HOUR FROM time AT TIME ZONE 'Asia/Ho_Chi_Minh')::int AS hour,
            COUNT(*)::double precision / :weekCount AS avg_tickets
        FROM analytics_event
        WHERE store_id = :storeId
          AND time >= :from AND time < :to
          AND event_type = 'TICKET_ISSUED'
        GROUP BY dow, hour
        ORDER BY dow, hour
    """.trimIndent())
    .bind("storeId", storeId)
    .bind("from", from)
    .bind("to", to)
    .bind("weekCount", weekCount.coerceAtLeast(1.0))
    .map { row: Readable, _ ->
        HeatmapRow(
            dayOfWeek = row.get("dow", Int::class.javaObjectType) ?: 1,
            hour = row.get("hour", Int::class.javaObjectType) ?: 0,
            avgTickets = row.get("avg_tickets", Double::class.javaObjectType) ?: 0.0
        )
    }
    .all()
    .asFlow()
    .toList()
}
```

**Why `ISODOW`**: PostgreSQL's `ISODOW` returns 1=Monday through 7=Sunday (ISO 8601), which is the standard in Vietnam and most non-US locales.

**Why divide by `weekCount`**: Normalizes to "average tickets per week" so a 90-day query doesn't look 13× busier than a 7-day query. Same approach used in existing `getHourlyDistribution`.

#### Step 5.3 — Backend: Service method

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/analytics/service/AnalyticsQueryService.kt`

Add:
```kotlin
suspend fun getHourlyHeatmap(storeId: UUID, range: Range): HourlyHeatmapResponse {
    val (from, to) = rangeToRange(range)
    val rows = analyticsEventRepository.getHourlyHeatmap(storeId, from, to)

    // Fill in missing (dow, hour) pairs with 0
    val cellMap = rows.associateBy { it.dayOfWeek to it.hour }
    val cells = (1..7).flatMap { dow ->
        (0..23).map { hour ->
            HeatmapCell(
                dayOfWeek = dow,
                hour = hour,
                avgTickets = cellMap[dow to hour]?.avgTickets ?: 0.0
            )
        }
    }

    return HourlyHeatmapResponse(
        range = range.name.lowercase(),
        cells = cells
    )
}
```

#### Step 5.4 — Backend: Controller endpoint

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/analytics/controller/AnalyticsController.kt`

Add:
```kotlin
@GetMapping("/{storeId}/heatmap")
suspend fun getHourlyHeatmap(
    @PathVariable storeId: UUID,
    @RequestParam range: Range,
    @AuthenticationPrincipal principal: AdminPrincipal
): ResponseEntity<HourlyHeatmapResponse> {
    StoreAccessUtil.requireStoreAccess(principal, storeId)
    return ResponseEntity.ok(analyticsQueryService.getHourlyHeatmap(storeId, range))
}
```

#### Step 5.5 — Frontend: API + types

**File**: `web/src/features/analytics/types.ts`

Add:
```typescript
export interface HourlyHeatmapResponse {
  range: string;
  cells: HeatmapCell[];
}

export interface HeatmapCell {
  dayOfWeek: number;  // 1=Mon, 7=Sun
  hour: number;       // 0-23
  avgTickets: number;
}
```

**File**: `web/src/lib/constants.ts`

Add route:
```typescript
HEATMAP: (storeId: string, range: string) =>
  `/api/analytics/${storeId}/heatmap?range=${range}`,
```

**File**: `web/src/features/analytics/api.ts`

Add:
```typescript
export function getHourlyHeatmap(storeId: string, period: PeriodOrRange) {
  return get<HourlyHeatmapResponse>(
    API_ROUTES.ANALYTICS.HEATMAP(storeId, rangeParam(period)),
  );
}
```

#### Step 5.6 — Frontend: New component `hourly-heatmap.tsx`

**File**: `web/src/features/analytics/hourly-heatmap.tsx`

This is a custom-rendered heatmap (recharts does not have a native heatmap). Render as a CSS grid or HTML table:

```tsx
"use client";

import { useTranslations } from "next-intl";
import type { HourlyHeatmapResponse } from "./types";

interface HourlyHeatmapProps {
  data: HourlyHeatmapResponse | null;
  loading: boolean;
}

export function HourlyHeatmap({ data, loading }: HourlyHeatmapProps) {
  const t = useTranslations("analytics.heatmap");
  const tAnalytics = useTranslations("analytics");

  // Render a 7-row × 24-column grid
  // Row labels: Mon, Tue, Wed, Thu, Fri, Sat, Sun (use t("mon") etc.)
  // Column headers: 0, 1, 2, ..., 23 (or "12a", "6a", "12p", "6p" at intervals)
  //
  // Cell color: interpolate between transparent/muted (0 tickets) and var(--chart-accent) (max tickets)
  // Use inline style: `opacity: value / maxValue` or HSL lightness interpolation
  //
  // Hover tooltip showing: "Monday 14:00–15:00: 12.3 tickets/week"
  //
  // glass-card wrapper, full-width layout
  // On mobile: horizontal scroll with min-width
}
```

**Key design decisions**:
- **No recharts dependency** for this chart — pure CSS grid + divs with dynamic background opacity is simpler and more performant than trying to force recharts into a heatmap
- Color scale: compute `maxTickets = Math.max(...cells.map(c => c.avgTickets))`, then each cell's opacity is `avgTickets / maxTickets`
- Base color from CSS variable `var(--chart-accent)` with varying alpha
- Day labels localized via translation keys
- Hour headers: show every 3rd hour (0, 3, 6, 9, 12, 15, 18, 21) to avoid crowding
- Tooltip via `title` attribute or a lightweight hover state (no heavy tooltip library needed)
- Mobile: `overflow-x-auto` with `min-w-[600px]` on the grid container

#### Step 5.7 — Add to store detail layout

**File**: `web/src/features/analytics/store-analytics.tsx`

Add `getHourlyHeatmap` to the `Promise.all` fetch:
```tsx
const [summaryRes, peakRes, throughputRes, waitDistRes, heatmapRes] = await Promise.all([
  getStoreSummary(storeId, p),
  getPeakHours(storeId, p),
  getDailyThroughput(storeId, p),
  getWaitDistribution(storeId, p),
  getHourlyHeatmap(storeId, p),
]);
```

Place as full-width below the 2×2 chart grid:
```tsx
<div className="grid gap-4 l:grid-cols-2 l:gap-6">
  <OutcomeChart summary={summary} loading={loading} />
  <WaitDistributionChart data={waitDist} loading={loading} />
  <PeakHoursChart data={peakHours} loading={loading} />
  <ThroughputChart data={throughput} loading={loading} />
</div>

<HourlyHeatmap data={heatmap} loading={loading} />
```

#### Step 5.8 — Translation keys

**EN**:
```json
"heatmap": {
  "title": "Weekly Traffic Pattern",
  "mon": "Mon",
  "tue": "Tue",
  "wed": "Wed",
  "thu": "Thu",
  "fri": "Fri",
  "sat": "Sat",
  "sun": "Sun",
  "tooltip": "{day} {hourRange}: {value} tickets/week"
}
```

**VI**:
```json
"heatmap": {
  "title": "Mật độ khách theo tuần",
  "mon": "T2",
  "tue": "T3",
  "wed": "T4",
  "thu": "T5",
  "fri": "T6",
  "sat": "T7",
  "sun": "CN",
  "tooltip": "{day} {hourRange}: {value} vé/tuần"
}
```

---

## Final Layout Summary

### Overview Dashboard (Super Admin)

```
┌──────────────────────────────────────┐
│ Analytics Overview                    │
│ [Today] [Week] [Month] [Quarter] [📅]│
├──────────────────────────────────────┤
│ ┌────────┐┌────────┐┌────────┐┌─────┐│
│ │ Stores ││ Issued ││Compltd ││ Avg ││
│ │ Active ││        ││        ││Wait ││
│ └────────┘└────────┘└────────┘└─────┘│
├──────────────────────────────────────┤
│ Daily Throughput [▼ All Stores]      │
│ ┌────────────────────────────────┐   │
│ │  📈 Line chart                 │   │
│ └────────────────────────────────┘   │
├──────────────┬───────────────────────┤
│ Cancel/Skip  │  Avg Wait by Store    │  ← NEW
│ Rate by Store│  ┌─────────────────┐  │  ← NEW
│ ┌──────────┐ │  │  ▬ horizontal   │  │  ← NEW
│ │ ▬▬ horiz │ │  │    bars         │  │  ← NEW
│ └──────────┘ │  └─────────────────┘  │  ← NEW
├──────────────┴───────────────────────┤
│ Store Rankings [▼ Sort: Most Issued] │
│ ┌────────────────────────────────┐   │
│ │  📋 Table                      │   │
│ └────────────────────────────────┘   │
└──────────────────────────────────────┘
```

### Store Analytics (Detail)

```
┌──────────────────────────────────────┐
│ Analytics — Store Name               │
│ [Today] [Week] [Month] [Quarter] [📅]│
├──────────────────────────────────────┤
│ ┌────────┐┌────────┐┌────────┐┌─────┐│
│ │ Issued ││Compltd ││Cancel'd││ Avg ││
│ │        ││        ││        ││Wait ││
│ └────────┘└────────┘└────────┘└─────┘│
├──────────────────────────────────────┤
│ Summary (8 metric cards)             │
├──────────────┬───────────────────────┤
│ Ticket       │  Wait Time            │  ← NEW
│ Outcomes     │  Distribution         │  ← NEW
│ ┌──────────┐ │  ┌─────────────────┐  │  ← NEW
│ │  🍩 Donut│ │  │  📊 Histogram   │  │  ← NEW
│ └──────────┘ │  └─────────────────┘  │  ← NEW
├──────────────┼───────────────────────┤
│ Tickets by   │  Daily Throughput     │
│ Hour         │  ┌─────────────────┐  │
│ ┌──────────┐ │  │  📈 Line chart  │  │
│ │  📊 Bars │ │  │                 │  │
│ └──────────┘ │  └─────────────────┘  │
├──────────────┴───────────────────────┤
│ Weekly Traffic Pattern               │  ← NEW
│ ┌────────────────────────────────┐   │  ← NEW
│ │  🗓️ 7×24 Heatmap              │   │  ← NEW
│ └────────────────────────────────┘   │  ← NEW
└──────────────────────────────────────┘
```

---

## Implementation Order

| Step | Chart | Scope | Backend? | Estimated Effort |
|------|-------|-------|----------|------------------|
| 1 | Store Comparison (cancel/skip) | Overview | No | Small |
| 2 | Store Wait Time | Overview | No | Small |
| 3 | Outcome Donut | Store Detail | No | Small |
| 4 | Wait Time Distribution | Store Detail | Yes — 1 endpoint | Medium |
| 5 | Hourly Heatmap | Store Detail | Yes — 1 endpoint | Medium-Large |

**Recommended order**: 1 → 2 → 3 → 4 → 5 (frontend-only first, then backend additions)

Charts 1–3 can be implemented in a single session (all frontend-only, data already available).
Charts 4–5 each require a backend endpoint + frontend component.

---

## CHANGELOGS.md Entry

Log under the implementation date:
```
### Analytics Charts Expansion
- **New**: Store Comparison chart (cancel/skip rate by store) on Overview dashboard
- **New**: Avg Wait Time by Store chart on Overview dashboard
- **New**: Outcome Breakdown donut chart on Store Analytics detail
- **New**: Wait Time Distribution histogram on Store Analytics detail (new backend endpoint)
- **New**: Weekly Traffic Pattern heatmap on Store Analytics detail (new backend endpoint)
- **Updated**: Store Analytics layout — 2×2 chart grid + full-width heatmap
- **Updated**: Overview layout — 2-column comparison charts between throughput and rankings
- **Added**: Translation keys for all new charts (EN + VI)
```
