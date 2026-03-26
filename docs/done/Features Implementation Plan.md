# NotiGuide — Features Implementation Plan

Concrete, step-by-step implementation plan for all features defined in `Features Ideas.md`. Covers backend, admin-web (`web/`), and client-web (`client-web/`). Each step specifies exact files, method signatures, class names, Redis keys, SQL statements, translation keys, and UI layout — ready to execute without ambiguity.

**Prerequisite**: The Analytics Implementation Plan (Phases 1–6) must be completed first. Features 1 and 7 are fully covered by that plan — **both are now DONE** (implemented 2026-03-23). This document covers Features 2, 4, 5, 6, 9, and 10.

**Also completed**: The Jump-Call feature (`allowJumpCall` store toggle + `callSpecificTicket` endpoint + stacked serving display) was implemented on 2026-03-25. This is not a numbered feature but is referenced by Feature 9 Phase 2 (store settings surface). The `allowJumpCall` field lives directly on the `store` table, not in `store_settings`.

---

## Dependency Graph & Implementation Order

```
Analytics Plan (Phases 1–6)   ✅ DONE (2026-03-23)
Jump-Call Feature             ✅ DONE (2026-03-25)
        │
        ├─── Feature 9 Phase 1  (frontend-only, no deps)
        ├─── Feature 10 Phase 1 (login history)
        ├─── Feature 9 Phase 2  (queue pausing & capping) ── includes Feature 4
        ├─── Feature 2          (proactive FCM alerts) ── requires existing FCM infra
        ├─── Feature 9 Phase 3  (no-show & grace period) ── includes Feature 5
        ├─── Feature 10 Phase 2 (active sessions)
        └─── Feature 6          (multi-counter, last — largest scope)
```

**Why this order:**
1. Feature 9 Phase 1 is frontend-only — ships immediately, no backend blocking.
2. Feature 10 Phase 1 is a simple DB + endpoint addition — low risk, builds the Security settings tab.
3. Feature 9 Phase 2 introduces `store_settings` table — required by Phases 3 and Feature 2 (alert thresholds). Note: `allowJumpCall` is already on the `store` table (not `store_settings`) — Phase 2 should add a toggle for it in the Store settings UI alongside the queue cap setting.
4. Feature 2 uses `store_settings` for alert thresholds and builds on existing FCM infra.
5. Feature 9 Phase 3 extends `store_settings` + `RedisKeyExpirationListener` — depends on Phase 2 table.
6. Feature 10 Phase 2 is self-contained but large — JWT blacklist touches `JWTAuthFilter`.
7. Feature 6 is the largest scope change (multi-queue Redis restructure) — last to avoid destabilizing earlier features.

---

## Feature 9 Phase 1: Default Counter ID (Frontend Only)

**Scope**: Admin-web only. No backend changes.

### Step 1.1 — Settings Tab: "Store" Page

**New file**: `web/src/app/[locale]/dashboard/settings/store/page.tsx`

```tsx
"use client";

import { useTranslations } from "next-intl";
import { useEffect, useState } from "react";
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from "@/components/ui/card";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { useAuthStore } from "@/store/auth";

export default function StoreSettingsPage() {
  const tSettings = useTranslations("settings");
  const { storeId } = useAuthStore();
  const [counterId, setCounterId] = useState("");

  useEffect(() => {
    if (!storeId) return;
    const stored = localStorage.getItem(`store:${storeId}:defaultCounterId`);
    if (stored) setCounterId(stored);
  }, [storeId]);

  function handleChange(value: string) {
    const trimmed = value.slice(0, 100);
    setCounterId(trimmed);
    if (storeId) {
      if (trimmed) {
        localStorage.setItem(`store:${storeId}:defaultCounterId`, trimmed);
      } else {
        localStorage.removeItem(`store:${storeId}:defaultCounterId`);
      }
    }
  }

  return (
    <Card>
      <CardHeader>
        <CardTitle>{tSettings("store.queueDefaults")}</CardTitle>
        <CardDescription>{tSettings("store.queueDefaultsDescription")}</CardDescription>
      </CardHeader>
      <CardContent>
        <div className="space-y-2">
          <Label htmlFor="defaultCounterId">{tSettings("store.defaultCounterIdLabel")}</Label>
          <Input
            id="defaultCounterId"
            value={counterId}
            onChange={(e) => handleChange(e.target.value)}
            placeholder={tSettings("store.defaultCounterIdPlaceholder")}
            maxLength={100}
            className="max-w-xs"
          />
          <p className="text-xs text-muted-foreground">{tSettings("store.defaultCounterIdCaption")}</p>
        </div>
      </CardContent>
    </Card>
  );
}
```

### Step 1.2 — Update Settings Layout Tab Navigation

**File**: `web/src/app/[locale]/dashboard/settings/layout.tsx`

Add "Store" tab between "Password" and any future tabs. Tab is **hidden** for `ROLE_SUPER_ADMIN`:

```tsx
// Add to tabs array (conditionally):
const tabs = [
  { key: "account", href: "/dashboard/settings/account" },
  { key: "password", href: "/dashboard/settings/password" },
  // Only show for ADMIN role (not SUPER_ADMIN):
  ...(isSuperAdmin ? [] : [{ key: "store", href: "/dashboard/settings/store" }]),
];
```

### Step 1.3 — Queue Page: Read Default Counter ID

**File**: `web/src/app/[locale]/dashboard/queue/page.tsx`

Change the `counterId` initial state from `""` to read from localStorage:

```tsx
const [counterId, setCounterId] = useState(() => {
  if (typeof window === "undefined" || !storeId) return "";
  return localStorage.getItem(`store:${storeId}:defaultCounterId`) ?? "";
});
```

### Step 1.4 — Translation Keys

**File**: `web/src/messages/en.json` — add under `"settings"`:

```json
"store": {
  "title": "Store",
  "queueDefaults": "Queue Defaults",
  "queueDefaultsDescription": "Configure default values for queue operations",
  "defaultCounterIdLabel": "Default Counter ID",
  "defaultCounterIdPlaceholder": "e.g. Counter 1",
  "defaultCounterIdCaption": "Pre-fills the counter field when calling the next ticket"
}
```

**File**: `web/src/messages/vi.json` — add under `"settings"`:

```json
"store": {
  "title": "Cửa hàng",
  "queueDefaults": "Mặc định hàng đợi",
  "queueDefaultsDescription": "Cấu hình giá trị mặc định cho thao tác hàng đợi",
  "defaultCounterIdLabel": "Mã quầy mặc định",
  "defaultCounterIdPlaceholder": "VD: Quầy 1",
  "defaultCounterIdCaption": "Tự động điền vào trường quầy khi gọi khách tiếp theo"
}
```

---

## Feature 10 Phase 1: Login History

### Step 2.1 — Database Schema

**File**: `backend/src/main/resources/db/schema.sql` — append:

```sql
CREATE TABLE login_history (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    admin_id    UUID NOT NULL REFERENCES admin(id) ON DELETE CASCADE,
    ip_address  VARCHAR(45) NOT NULL,
    success     BOOLEAN NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_login_history_admin_created
    ON login_history (admin_id, created_at DESC);
```

### Step 2.2 — Entity

**New file**: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/entity/LoginHistory.kt`

```kotlin
package com.thomas.notiguide.domain.admin.entity

import org.springframework.data.annotation.Id
import org.springframework.data.relational.core.mapping.Table
import java.time.OffsetDateTime
import java.util.UUID

@Table("login_history")
data class LoginHistory(
    @Id val id: UUID? = null,
    val adminId: UUID,
    val ipAddress: String,
    val success: Boolean,
    val createdAt: OffsetDateTime? = null
)
```

### Step 2.3 — DTO

**New file**: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/dto/LoginHistoryDto.kt`

```kotlin
package com.thomas.notiguide.domain.admin.dto

import java.time.OffsetDateTime

data class LoginHistoryDto(
    val id: String,
    val ipAddress: String,
    val success: Boolean,
    val createdAt: OffsetDateTime?
)

data class LoginHistoryPageResponse(
    val items: List<LoginHistoryDto>,
    val hasMore: Boolean
)
```

### Step 2.4 — Repository

**New file**: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/repository/LoginHistoryRepository.kt`

```kotlin
package com.thomas.notiguide.domain.admin.repository

import com.thomas.notiguide.domain.admin.entity.LoginHistory
import kotlinx.coroutines.flow.Flow
import org.springframework.data.repository.kotlin.CoroutineCrudRepository
import java.util.UUID

interface LoginHistoryRepository : CoroutineCrudRepository<LoginHistory, UUID> {
    fun findByAdminIdOrderByCreatedAtDesc(adminId: UUID): Flow<LoginHistory>
}
```

Query method: use R2DBC `DatabaseClient` for the paginated variant (limit + 1 for `hasMore`):

```kotlin
// Add to repository or a separate query class:
suspend fun findRecentByAdminId(adminId: UUID, limit: Int): List<LoginHistory>
```

Implementation uses `DatabaseClient`:

```kotlin
suspend fun findRecentByAdminId(adminId: UUID, limit: Int): List<LoginHistory> {
    return client.sql("""
        SELECT id, admin_id, ip_address, success, created_at
        FROM login_history
        WHERE admin_id = :adminId
        ORDER BY created_at DESC
        LIMIT :limit
    """)
    .bind("adminId", adminId)
    .bind("limit", limit + 1)  // fetch one extra to determine hasMore
    .map { row, _ -> ... }
    .flow()
    .toList()
}
```

### Step 2.5 — Service Method

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/service/AdminService.kt`

Add method:

```kotlin
suspend fun recordLoginAttempt(adminId: UUID, ipAddress: String, success: Boolean) {
    loginHistoryRepository.save(
        LoginHistory(adminId = adminId, ipAddress = ipAddress, success = success)
    )
}

suspend fun getLoginHistory(adminId: UUID, limit: Int = 20): LoginHistoryPageResponse {
    val rows = loginHistoryRepository.findRecentByAdminId(adminId, limit)
    val hasMore = rows.size > limit
    val items = rows.take(limit).map { it.toDto() }
    return LoginHistoryPageResponse(items = items, hasMore = hasMore)
}
```

### Step 2.6 — Integration into AuthController Login Flow

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/controller/AuthController.kt`

In the `login()` method, extract IP from `ServerHttpRequest` and record the attempt. Add `request: ServerHttpRequest` parameter:

```kotlin
@PostMapping("/login")
suspend fun login(
    @Valid @RequestBody body: LoginRequest,
    request: ServerHttpRequest
): ResponseEntity<LoginResponse> {
    val ip = extractClientIp(request)
    // ... existing login logic ...
    // On success:
    adminService.recordLoginAttempt(admin.id!!, ip, success = true)
    // On BadCredentialsException catch:
    // adminService.recordLoginAttempt(foundAdmin.id!!, ip, success = false)
}
```

IP extraction — reuse the same pattern as `RateLimitFilter`:

```kotlin
private fun extractClientIp(request: ServerHttpRequest): String {
    val forwarded = request.headers.getFirst("X-Forwarded-For")
    if (!forwarded.isNullOrBlank()) {
        return forwarded.split(",").first().trim()
    }
    return request.remoteAddress?.address?.hostAddress ?: "unknown"
}
```

**Important**: For failed login attempts, the admin ID may not be available (wrong username). In that case, skip recording. Only record when the admin entity was found (wrong password or unverified account = record with `success = false`).

### Step 2.7 — Endpoint

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/controller/AdminController.kt`

Add endpoint:

```kotlin
@GetMapping("/{id}/login-history")
suspend fun getLoginHistory(
    @PathVariable id: UUID,
    @RequestParam(defaultValue = "20") limit: Int,
    @AuthenticationPrincipal principal: AdminPrincipal
): ResponseEntity<LoginHistoryPageResponse> {
    if (principal.id != id) throw ForbiddenException("Can only view own login history")
    return ResponseEntity.ok(adminService.getLoginHistory(id, limit.coerceIn(1, 100)))
}
```

### Step 2.8 — Retention Scheduler

**New file**: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/service/LoginHistoryCleanupScheduler.kt`

```kotlin
@Component
class LoginHistoryCleanupScheduler(
    private val client: DatabaseClient
) {
    @Scheduled(cron = "0 0 3 * * *")  // daily at 3 AM
    fun cleanupOldEntries() = runBlocking {
        client.sql("DELETE FROM login_history WHERE created_at < now() - INTERVAL '90 days'")
            .fetch().rowsUpdated().awaitSingle()
    }
}
```

### Step 2.9 — Admin-Web: Security Settings Page

**New file**: `web/src/app/[locale]/dashboard/settings/security/page.tsx`

```tsx
// Client component
// Uses: useTranslations("settings"), useAuthStore()
// API: GET /api/admins/{id}/login-history?limit=20
// Renders: Card "Login History" with table
// Columns: Date & Time | IP Address | Status (Success/Failed badge)
// "Load more" button when hasMore = true (fetches with limit=40, 60, etc.)
```

### Step 2.10 — Admin-Web Types & API

**File**: `web/src/types/admin.ts` — add:

```typescript
interface LoginHistoryDto {
  id: string;
  ipAddress: string;
  success: boolean;
  createdAt: string | null;
}

interface LoginHistoryPageResponse {
  items: LoginHistoryDto[];
  hasMore: boolean;
}
```

**File**: `web/src/features/admin/api.ts` — add:

```typescript
export async function getLoginHistory(adminId: string, limit = 20): Promise<LoginHistoryPageResponse> {
  return get(`/api/admins/${adminId}/login-history?limit=${limit}`);
}
```

### Step 2.11 — Settings Layout: Add Security Tab

**File**: `web/src/app/[locale]/dashboard/settings/layout.tsx`

Add "Security" tab (visible to all roles):

```tsx
const tabs = [
  { key: "account", href: "/dashboard/settings/account" },
  { key: "password", href: "/dashboard/settings/password" },
  ...(isSuperAdmin ? [] : [{ key: "store", href: "/dashboard/settings/store" }]),
  { key: "security", href: "/dashboard/settings/security" },
];
```

### Step 2.12 — Translation Keys

**File**: `web/src/messages/en.json` — add under `"settings"`:

```json
"security": {
  "title": "Security",
  "loginHistory": "Login History",
  "loginHistoryDescription": "Recent login attempts for your account",
  "date": "Date & Time",
  "ipAddress": "IP Address",
  "status": "Status",
  "loginSuccess": "Success",
  "loginFailed": "Failed",
  "loadMore": "Load more",
  "noHistory": "No login history yet"
}
```

**File**: `web/src/messages/vi.json`:

```json
"security": {
  "title": "Bảo mật",
  "loginHistory": "Lịch sử đăng nhập",
  "loginHistoryDescription": "Các lần đăng nhập gần đây của tài khoản",
  "date": "Thời gian",
  "ipAddress": "Địa chỉ IP",
  "status": "Trạng thái",
  "loginSuccess": "Thành công",
  "loginFailed": "Thất bại",
  "loadMore": "Xem thêm",
  "noHistory": "Chưa có lịch sử đăng nhập"
}
```

---

## Feature 9 Phase 2 + Feature 4: Queue Pausing & Capping

### Step 3.1 — Database Schema

**File**: `backend/src/main/resources/db/schema.sql` — append:

```sql
CREATE TABLE store_settings (
    store_id          UUID PRIMARY KEY REFERENCES store(id) ON DELETE CASCADE,
    max_queue_size    INT NOT NULL DEFAULT 0,
    grace_period_sec  INT NOT NULL DEFAULT 0,
    no_show_action    VARCHAR(10) NOT NULL DEFAULT 'SKIP',
    max_requeues      INT NOT NULL DEFAULT 1,
    requeue_offset    INT NOT NULL DEFAULT 3,
    alert_threshold   INT NOT NULL DEFAULT 2,
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Note: `alert_threshold` is added here for Feature 2 (proactive alerts) — the position at which "You're next" alerts fire. Default `2` means alert at position 2 and position 0 (called).

### Step 3.2 — Entity

**New file**: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/entity/StoreSettings.kt`

```kotlin
package com.thomas.notiguide.domain.store.entity

import org.springframework.data.annotation.Id
import org.springframework.data.annotation.LastModifiedDate
import org.springframework.data.relational.core.mapping.Table
import java.time.OffsetDateTime
import java.util.UUID

@Table("store_settings")
data class StoreSettings(
    @Id val storeId: UUID,
    val maxQueueSize: Int = 0,
    val gracePeriodSec: Int = 0,
    val noShowAction: String = "SKIP",
    val maxRequeues: Int = 1,
    val requeueOffset: Int = 3,
    val alertThreshold: Int = 2,
    @LastModifiedDate val updatedAt: OffsetDateTime? = null
)
```

### Step 3.3 — DTO & Request

**New file**: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/dto/StoreSettingsDto.kt`

```kotlin
data class StoreSettingsDto(
    val storeId: String,
    val maxQueueSize: Int,
    val gracePeriodSec: Int,
    val noShowAction: String,
    val maxRequeues: Int,
    val requeueOffset: Int,
    val alertThreshold: Int,
    val updatedAt: OffsetDateTime?
)
```

**New file**: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/request/UpdateStoreSettingsRequest.kt`

```kotlin
data class UpdateStoreSettingsRequest(
    @field:Min(0) val maxQueueSize: Int? = null,
    @field:Min(0) @field:Max(600) val gracePeriodSec: Int? = null,
    val noShowAction: String? = null,           // "SKIP" or "REQUEUE"
    @field:Min(0) @field:Max(5) val maxRequeues: Int? = null,
    @field:Min(1) @field:Max(20) val requeueOffset: Int? = null,
    @field:Min(1) @field:Max(10) val alertThreshold: Int? = null
)
```

### Step 3.4 — Repository

**New file**: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/repository/StoreSettingsRepository.kt`

```kotlin
interface StoreSettingsRepository : CoroutineCrudRepository<StoreSettings, UUID>
```

### Step 3.5 — Redis Key Additions

**File**: `backend/src/main/kotlin/com/thomas/notiguide/core/redis/RedisKeyManager.kt`

Add:

```kotlin
fun queueState(storeId: UUID) = "store:$storeId:queue_state"
fun storeSettings(storeId: UUID) = "store:$storeId:settings"
```

### Step 3.6 — Queue State Enum

**New file**: `backend/src/main/kotlin/com/thomas/notiguide/domain/queue/types/QueueState.kt`

```kotlin
enum class QueueState {
    ACTIVE, PAUSED;

    companion object {
        fun from(raw: String?): QueueState = when (raw) {
            "PAUSED" -> PAUSED
            else -> ACTIVE
        }
    }
}
```

### Step 3.7 — StoreService: Auto-Create Settings on Store Creation

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/service/StoreService.kt`

Inject `StoreSettingsRepository`. In `createStore()`, after saving the store entity:

```kotlin
storeSettingsRepository.save(StoreSettings(storeId = saved.id!!))
```

In `deleteStore()`, the settings row is deleted automatically via `ON DELETE CASCADE`.

### Step 3.8 — Store Settings Service Methods

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/service/StoreService.kt`

Add methods:

```kotlin
suspend fun getStoreSettings(storeId: UUID): StoreSettingsDto {
    val settings = storeSettingsRepository.findById(storeId)
        ?: throw NotFoundException("StoreSettings", "storeId", storeId.toString())
    return settings.toDto()
}

suspend fun updateStoreSettings(storeId: UUID, request: UpdateStoreSettingsRequest): StoreSettingsDto {
    val existing = storeSettingsRepository.findById(storeId)
        ?: throw NotFoundException("StoreSettings", "storeId", storeId.toString())

    if (request.noShowAction != null && request.noShowAction !in listOf("SKIP", "REQUEUE")) {
        throw IllegalArgumentException("noShowAction must be SKIP or REQUEUE")
    }

    val updated = existing.copy(
        maxQueueSize = request.maxQueueSize ?: existing.maxQueueSize,
        gracePeriodSec = request.gracePeriodSec ?: existing.gracePeriodSec,
        noShowAction = request.noShowAction ?: existing.noShowAction,
        maxRequeues = request.maxRequeues ?: existing.maxRequeues,
        requeueOffset = request.requeueOffset ?: existing.requeueOffset,
        alertThreshold = request.alertThreshold ?: existing.alertThreshold
    )
    val saved = storeSettingsRepository.save(updated)

    // Invalidate Redis cache
    redis.delete(RedisKeyManager.storeSettings(storeId)).awaitSingleOrNull()

    return saved.toDto()
}
```

### Step 3.9 — Store Settings Endpoints

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/controller/StoreController.kt`

Add:

```kotlin
@GetMapping("/{id}/settings")
suspend fun getStoreSettings(
    @PathVariable id: UUID,
    @AuthenticationPrincipal principal: AdminPrincipal
): ResponseEntity<StoreSettingsDto> {
    StoreAccessUtil.requireStoreAccess(principal, id)
    return ResponseEntity.ok(storeService.getStoreSettings(id))
}

@PutMapping("/{id}/settings")
suspend fun updateStoreSettings(
    @PathVariable id: UUID,
    @Valid @RequestBody request: UpdateStoreSettingsRequest,
    @AuthenticationPrincipal principal: AdminPrincipal
): ResponseEntity<StoreSettingsDto> {
    StoreAccessUtil.requireStoreAccess(principal, id)
    return ResponseEntity.ok(storeService.updateStoreSettings(id, request))
}
```

### Step 3.10 — Queue Pause/Resume Endpoints

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/queue/controller/QueueAdminController.kt`

Add:

```kotlin
@PostMapping("/pause")
suspend fun pauseQueue(
    @PathVariable storeId: UUID,
    @AuthenticationPrincipal principal: AdminPrincipal
): ResponseEntity<Void> {
    StoreAccessUtil.requireStoreAccess(principal, storeId)
    queueService.setQueueState(storeId, QueueState.PAUSED)
    return ResponseEntity.noContent().build()
}

@PostMapping("/resume")
suspend fun resumeQueue(
    @PathVariable storeId: UUID,
    @AuthenticationPrincipal principal: AdminPrincipal
): ResponseEntity<Void> {
    StoreAccessUtil.requireStoreAccess(principal, storeId)
    queueService.setQueueState(storeId, QueueState.ACTIVE)
    return ResponseEntity.noContent().build()
}
```

### Step 3.11 — QueueService: State Management

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/queue/service/QueueService.kt`

Inject `StoreSettingsRepository`. Add methods:

```kotlin
suspend fun setQueueState(storeId: UUID, state: QueueState) {
    redis.opsForValue()
        .set(RedisKeyManager.queueState(storeId), state.name)
        .awaitSingle()
}

suspend fun getQueueState(storeId: UUID): QueueState {
    val raw = redis.opsForValue()
        .get(RedisKeyManager.queueState(storeId))
        .awaitSingleOrNull()
    return QueueState.from(raw)
}
```

### Step 3.12 — QueueService.issueTicket(): Enforce Pause & Cap

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/queue/service/QueueService.kt`

At the top of `issueTicket()`, before the existing store check, add:

```kotlin
// Check queue state
val queueState = getQueueState(storeId)
if (queueState == QueueState.PAUSED) {
    throw ConflictException("Queue is paused")
}

// Check queue cap
val settings = storeSettingsRepository.findById(storeId)
if (settings != null && settings.maxQueueSize > 0) {
    val currentSize = redisQueueRepository.getQueueSize(storeId)
    if (currentSize >= settings.maxQueueSize) {
        throw ConflictException("Queue is full")
    }
}
```

### Step 3.13 — QueuePublicController: Return Queue State in Store Info

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/queue/dto/StorePublicInfoResponse.kt`

Add field:

```kotlin
data class StorePublicInfoResponse(
    val id: UUID,
    val name: String,
    val address: String?,
    val isActive: Boolean,
    val queueState: String = "ACTIVE",    // NEW
    val maxQueueSize: Int = 0             // NEW
)
```

Update `QueueService` or the controller to populate these from Redis/DB when building the response.

### Step 3.14 — Client-Web: Handle Paused/Full Queue

**File**: `client-web/src/types/store.ts` — add fields:

```typescript
interface StorePublicInfoResponse {
  id: string;
  name: string;
  address: string | null;
  isActive: boolean;
  queueState: string;    // "ACTIVE" | "PAUSED"
  maxQueueSize: number;  // 0 = unlimited
}
```

**File**: `client-web/src/app/[locale]/store/[storeId]/page.tsx`

Add handling for paused queue (show banner similar to `StoreClosedBanner`) and full queue (show banner when queue size >= max). Disable `JoinQueueButton` when paused or full.

**New component**: `client-web/src/components/store/queue-paused-banner.tsx`

Glass card with `PauseCircle` icon. Message: "This queue is temporarily paused. Please check back shortly."

**New component**: `client-web/src/components/store/queue-full-banner.tsx`

Glass card with `UsersRound` icon. Message: "Queue is full. Please try again later."

### Step 3.15 — Admin-Web: Queue Page Pause/Resume Button

**File**: `web/src/app/[locale]/dashboard/queue/page.tsx`

Add a toggle button next to the header. When queue is paused, show a banner across the queue page. The button calls `POST /pause` or `POST /resume`.

**New file**: `web/src/features/queue/queue-state-toggle.tsx`

```tsx
// Props: storeId, currentState, onStateChange
// Renders: Button with Play/Pause icon
// Confirmation dialog before pausing
// Calls: POST /api/queue/admin/{storeId}/pause or /resume
// Toast on success
```

**File**: `web/src/features/queue/api.ts` — add:

```typescript
export async function pauseQueue(storeId: string): Promise<void> {
  return post(`/api/queue/admin/${storeId}/pause`);
}

export async function resumeQueue(storeId: string): Promise<void> {
  return post(`/api/queue/admin/${storeId}/resume`);
}

export async function getStoreSettings(storeId: string): Promise<StoreSettingsDto> {
  return get(`/api/stores/${storeId}/settings`);
}

export async function updateStoreSettings(storeId: string, request: UpdateStoreSettingsRequest): Promise<StoreSettingsDto> {
  return put(`/api/stores/${storeId}/settings`, request);
}
```

### Step 3.16 — Admin-Web: Store Settings Page (Queue Limits Card)

**File**: `web/src/app/[locale]/dashboard/settings/store/page.tsx`

Add cards below "Queue Defaults":

```
Card: "Queue Behavior"
  ├─ Switch: Allow jump-call (toggles allowJumpCall on the store)
  │  Caption: "Show a Call button on waiting list tickets to call them out of order"
  └─ Note: This calls PUT /api/stores/{storeId} (existing endpoint, allowJumpCall field already exists on store entity)

Card: "Queue Limits"
  └─ Input: Max queue size (number, 0 = unlimited)
     Caption: "New tickets are rejected when the queue reaches this size"
     Save button (calls PUT /api/stores/{storeId}/settings)
```

This section is only visible when `storeId` is present. The `allowJumpCall` toggle uses the existing store update endpoint. The queue limits card fetches current settings on mount via `GET /api/stores/{storeId}/settings`.

### Step 3.17 — Translation Keys (Queue State)

**File**: `web/src/messages/en.json` — add under `"queue"`:

```json
"pauseQueue": "Pause queue",
"resumeQueue": "Resume queue",
"queuePaused": "Queue is paused",
"queuePausedDescription": "No new tickets are being accepted. Existing tickets can still be served.",
"confirmPause": "Are you sure you want to pause the queue? No new tickets will be accepted.",
"pauseSuccess": "Queue paused",
"resumeSuccess": "Queue resumed"
```

**File**: `client-web/src/messages/en.json` — add under `"store"`:

```json
"queuePaused": "Queue is temporarily paused",
"queuePausedDescription": "Please check back shortly.",
"queueFull": "Queue is full",
"queueFullDescription": "Please try again later."
```

Add Vietnamese equivalents to both `vi.json` files.

---

## Feature 2: "You're Next" Proactive Alerts (FCM)

### Step 4.1 — Redis: Alert Tracking in Ticket Hash

When a proactive alert is sent, store a flag in the ticket Redis hash to prevent duplicate sends:

**Key pattern**: Existing ticket hash `ticket:{storeId}:{ticketId}` — add field `alert_pos_{N}: 1`

No new Redis keys needed — this uses the existing ticket hash.

### Step 4.2 — FcmNotificationService: New Methods

**File**: `backend/src/main/kotlin/com/thomas/notiguide/core/firebase/FcmNotificationService.kt`

Add methods:

```kotlin
suspend fun sendProactiveAlert(
    storeId: UUID,
    ticketId: UUID,
    ticketNumber: String?,
    position: Long,
    storeName: String
) {
    val tokenKey = RedisKeyManager.fcmToken(storeId, ticketId)
    val token = redis.opsForValue().get(tokenKey).awaitSingleOrNull() ?: return

    // Check if already sent for this position
    val ticketKey = RedisKeyManager.ticket(storeId, ticketId)
    val alertFlag = "alert_pos_$position"
    val alreadySent = redis.opsForHash<String, String>()
        .get(ticketKey, alertFlag).awaitSingleOrNull()
    if (alreadySent != null) return

    // Mark as sent
    redis.opsForHash<String, String>()
        .put(ticketKey, alertFlag, "1").awaitSingle()

    // Build and send FCM message — data-only
    val data = mapOf(
        "type" to "POSITION_ALERT",
        "storeId" to storeId.toString(),
        "ticketId" to ticketId.toString(),
        "ticketNumber" to (ticketNumber ?: ""),
        "position" to position.toString(),
        "storeName" to storeName
    )

    val message = Message.builder()
        .setToken(token)
        .putAllData(data)
        .setWebpushConfig(WebpushConfig.builder()
            .setFcmOptions(WebpushFcmOptions.builder()
                .setLink("/store/$storeId/ticket/$ticketId")
                .build())
            .build())
        .build()

    withContext(Dispatchers.IO) {
        try { firebaseMessaging.send(message) }
        catch (e: FirebaseMessagingException) { removeToken(storeId, ticketId) }
    }
}
```

### Step 4.3 — QueueService.callNext() & callSpecificTicket(): Trigger Proactive Alerts

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/queue/service/QueueService.kt`

After a successful `callNext()` or `callSpecificTicket()` (a ticket is called), check the front of the queue and send alerts to tickets at positions within the store's `alertThreshold`. Note: `callSpecificTicket()` already exists (jump-call feature, 2026-03-25) — add the proactive alert dispatch there too.

```kotlin
// After successful CALL_NEXT_SCRIPT and event publishing:
if (fcmNotificationService != null) {
    val settings = storeSettingsRepository.findById(storeId)
    val threshold = settings?.alertThreshold ?: 2
    val storeName = storeRepository.findById(storeId)?.name ?: ""

    // Get tickets at positions 0..(threshold-1) in the queue
    val waitingIds = redisQueueRepository.getWaitingTicketIds(storeId)
    for ((index, waitingTicketId) in waitingIds.take(threshold).withIndex()) {
        val position = index + 1L  // 1-based position
        val waitingTicketUuid = UUID.fromString(waitingTicketId)
        val ticketData = redisTicketRepository.getTicket(storeId, waitingTicketUuid)
        val ticketNumber = ticketData["number"]
        fcmNotificationService.sendProactiveAlert(
            storeId, waitingTicketUuid, ticketNumber, position, storeName
        )
    }
}
```

### Step 4.4 — Client-Web: Service Worker Update

**File**: `client-web/public/firebase-messaging-sw.js`

Add handler for `POSITION_ALERT` message type:

```javascript
// In onBackgroundMessage handler, add case for POSITION_ALERT:
if (data.type === "POSITION_ALERT") {
    const position = parseInt(data.position);
    const storeName = data.storeName || "";
    let title, body;
    if (locale === "vi") {
        title = position === 1 ? "Sắp đến lượt bạn!" : `Còn ${position} người phía trước`;
        body = position === 1
            ? `Hãy chuẩn bị tại ${storeName}.`
            : `Bạn đang ở vị trí ${position} tại ${storeName}.`;
    } else {
        title = position === 1 ? "You're next!" : `${position} ahead of you`;
        body = position === 1
            ? `Please head to ${storeName}.`
            : `You're #${position} in line at ${storeName}.`;
    }
    return self.registration.showNotification(title, {
        body,
        icon: "/notiguide-192.png",
        data: { url: `/store/${data.storeId}/ticket/${data.ticketId}` }
    });
}
```

### Step 4.5 — Translation Keys (Client-Web)

**File**: `client-web/src/messages/en.json`:

```json
"notification": {
  ...existing keys...,
  "positionAlertTitle": "Almost there!",
  "positionAlertBody": "You're #{position} in line at {storeName}.",
  "youreNextTitle": "You're next!",
  "youreNextBody": "Please head to {storeName}."
}
```

Vietnamese equivalents in `vi.json`.

---

## Feature 9 Phase 3 + Feature 5: No-Show & Grace Period

### Step 5.1 — New Ticket Statuses

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/queue/types/TicketStatus.kt`

Add:

```kotlin
enum class TicketStatus {
    WAITING, CALLED, SERVED, CANCELLED, SKIPPED, REQUEUED, UNKNOWN;
    // ...existing from() companion
}
```

### Step 5.2 — Redis Key Additions

**File**: `backend/src/main/kotlin/com/thomas/notiguide/core/redis/RedisKeyManager.kt`

Add:

```kotlin
fun graceExpiry(storeId: UUID, ticketId: UUID) = "grace:$storeId:$ticketId"

fun isGraceExpiryKey(key: String) = key.startsWith("grace:")

fun parseGraceExpiryKey(key: String): Pair<UUID, UUID>? {
    val parts = key.removePrefix("grace:").split(":")
    if (parts.size != 2) return null
    return try { UUID.fromString(parts[0]) to UUID.fromString(parts[1]) }
    catch (_: Exception) { null }
}
```

### Step 5.3 — QueueService.callNext(): Set Grace Expiry

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/queue/service/QueueService.kt`

After a successful call (ticket status set to CALLED), check store settings and set grace expiry if enabled:

```kotlin
// After CALL_NEXT_SCRIPT succeeds:
val settings = storeSettingsRepository.findById(storeId)
if (settings != null && settings.gracePeriodSec > 0) {
    redis.opsForValue()
        .set(
            RedisKeyManager.graceExpiry(storeId, ticketUuid),
            "1",
            Duration.ofSeconds(settings.gracePeriodSec.toLong())
        )
        .awaitSingle()
}
```

### Step 5.4 — Requeue Lua Script

**New constant** in `QueueService.kt`:

```kotlin
private val REQUEUE_TICKET_SCRIPT = RedisScript.of("""
    local servingKey = KEYS[1]
    local queueKey = KEYS[2]
    local ticketKey = KEYS[3]
    local ticketId = ARGV[1]
    local offset = tonumber(ARGV[2])
    local waitingTtlSeconds = ARGV[3]

    -- Remove from serving
    redis.call('SREM', servingKey, ticketId)

    -- Get the score of the front ticket (lowest score = oldest)
    local front = redis.call('ZRANGE', queueKey, 0, 0, 'WITHSCORES')
    local insertScore
    if #front == 0 then
        -- Empty queue: insert at current time
        insertScore = tonumber(ARGV[4])
    else
        -- Insert at front score minus offset (so it's placed 'offset' positions behind front)
        -- Use ZRANGEBYSCORE to find the Nth ticket's score
        local range = redis.call('ZRANGE', queueKey, 0, offset - 1, 'WITHSCORES')
        if #range >= offset * 2 then
            -- Position exists: place just after the Nth ticket
            insertScore = tonumber(range[offset * 2]) + 0.001
        else
            -- Queue shorter than offset: place at end
            local last = redis.call('ZRANGE', queueKey, -1, -1, 'WITHSCORES')
            if #last >= 2 then
                insertScore = tonumber(last[2]) + 0.001
            else
                insertScore = tonumber(ARGV[4])
            end
        end
    end

    redis.call('ZADD', queueKey, insertScore, ticketId)
    redis.call('HSET', ticketKey, 'status', 'REQUEUED')
    redis.call('EXPIRE', ticketKey, waitingTtlSeconds)

    return 1
""".trimIndent(), Long::class.java)
```

### Step 5.5 — QueueService: No-Show Handling Methods

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/queue/service/QueueService.kt`

Add:

```kotlin
suspend fun handleNoShow(storeId: UUID, ticketId: UUID) {
    val settings = storeSettingsRepository.findById(storeId) ?: return
    val ticketData = redisTicketRepository.getTicket(storeId, ticketId)
    if (ticketData.isEmpty()) return

    val status = TicketStatus.from(ticketData["status"])
    if (status != TicketStatus.CALLED) return  // Only process CALLED tickets

    val requeueCount = ticketData["requeue_count"]?.toIntOrNull() ?: 0

    if (settings.noShowAction == "REQUEUE" && requeueCount < settings.maxRequeues) {
        // Requeue
        redis.execute(REQUEUE_TICKET_SCRIPT, listOf(
            RedisKeyManager.serving(storeId),
            RedisKeyManager.queue(storeId),
            RedisKeyManager.ticket(storeId, ticketId)
        ), listOf(
            ticketId.toString(),
            settings.requeueOffset.toString(),
            RedisTTLPolicy.TICKET_WAITING.seconds.toString(),
            System.currentTimeMillis().toString()
        )).awaitSingleOrNull()

        // Increment requeue count
        redis.opsForHash<String, String>()
            .put(RedisKeyManager.ticket(storeId, ticketId), "requeue_count", (requeueCount + 1).toString())
            .awaitSingle()

        // Broadcast requeue event
        queueEventBroadcaster.broadcast(QueueSseEvent(
            type = "TICKET_REQUEUED", storeId = storeId, ticketId = ticketId,
            ticketNumber = ticketData["number"]
        ))
    } else {
        // Skip
        redisQueueRepository.removeFromServing(storeId, ticketId)
        redisTicketRepository.markSkipped(storeId, ticketId)
        fcmNotificationService?.removeToken(storeId, ticketId)

        // Broadcast skip event
        queueEventBroadcaster.broadcast(QueueSseEvent(
            type = "TICKET_SKIPPED", storeId = storeId, ticketId = ticketId,
            ticketNumber = ticketData["number"]
        ))

        // Analytics event
        analyticsEventService?.recordTicketSkipped(
            storeId, ticketId, ticketData["number"] ?: "",
            ticketData["issued_at"]?.let { Instant.parse(it) }
        )
    }

    // Delete grace expiry key (may already be expired)
    redis.delete(RedisKeyManager.graceExpiry(storeId, ticketId)).awaitSingleOrNull()
}
```

### Step 5.6 — RedisTicketRepository: markSkipped

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/queue/repository/RedisTicketRepository.kt`

Add:

```kotlin
suspend fun markSkipped(storeId: UUID, ticketId: UUID): Boolean {
    val key = RedisKeyManager.ticket(storeId, ticketId)
    return redis.opsForHash<String, String>()
        .put(key, "status", TicketStatus.SKIPPED.name)
        .awaitSingle()
    // TTL: immediate deletion (same as SERVED/CANCELLED)
}
```

### Step 5.7 — RedisKeyExpirationListener: Handle Grace Expiry

**File**: `backend/src/main/kotlin/com/thomas/notiguide/core/redis/RedisKeyExpirationListener.kt`

In the `subscribe()` method, add a branch for grace expiry keys:

```kotlin
// Existing: if (RedisKeyManager.isTicketKey(expiredKey)) { ... }
// Add:
if (RedisKeyManager.isGraceExpiryKey(expiredKey)) {
    val (storeId, ticketId) = RedisKeyManager.parseGraceExpiryKey(expiredKey) ?: return@collect
    queueService.handleNoShow(storeId, ticketId)
}
```

Inject `QueueService` into the listener (lazy to avoid circular dependency):

```kotlin
class RedisKeyExpirationListener(
    private val connectionFactory: ReactiveRedisConnectionFactory,
    private val queueRepo: RedisQueueRepository,
    private val queueService: Lazy<QueueService>  // lazy injection
)
```

### Step 5.8 — Manual No-Show Endpoint

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/queue/controller/QueueAdminController.kt`

Add:

```kotlin
@PostMapping("/tickets/{ticketId}/no-show")
suspend fun triggerNoShow(
    @PathVariable storeId: UUID,
    @PathVariable ticketId: UUID,
    @AuthenticationPrincipal principal: AdminPrincipal
): ResponseEntity<Void> {
    StoreAccessUtil.requireStoreAccess(principal, storeId)
    queueService.handleNoShow(storeId, ticketId)
    return ResponseEntity.noContent().build()
}
```

### Step 5.9 — QueueService.serveTicket() & cancelTicket(): Clear Grace Expiry

In both `serveTicket()` and `cancelTicket()`, add at the end:

```kotlin
redis.delete(RedisKeyManager.graceExpiry(storeId, ticketId)).awaitSingleOrNull()
```

This prevents the grace expiry from firing after the ticket is already handled.

### Step 5.10 — Admin-Web: Serving Display — No-Show Button

**File**: `web/src/features/queue/serving-display.tsx`

Add a third action button "No-Show" alongside "Serve" and "Cancel". Only visible when store settings have `gracePeriodSec > 0`. Keyboard shortcut: `X`.

```tsx
// Add to the button row:
{settings?.gracePeriodSec > 0 && (
    <Button variant="outline" onClick={handleNoShow} disabled={noShowLoading}>
        {noShowLoading && <Loader2 aria-hidden="true" className="mr-2 size-4 animate-spin" />}
        {tQueue("noShowButton")}
        <Kbd aria-hidden="true" className="ml-2">X</Kbd>
    </Button>
)}
```

**File**: `web/src/features/queue/api.ts` — add:

```typescript
export async function triggerNoShow(storeId: string, ticketId: string): Promise<void> {
  return post(`/api/queue/admin/${storeId}/tickets/${ticketId}/no-show`);
}
```

### Step 5.11 — Admin-Web: Grace Period Timer Display

When a ticket is being served and grace period is enabled, show a countdown timer on the `ServingDisplay` component. The timer is purely client-side — calculated from `calledAt + gracePeriodSec`.

### Step 5.12 — Admin-Web: Store Settings Page — No-Show Handling Card

**File**: `web/src/app/[locale]/dashboard/settings/store/page.tsx`

Add third card:

```
Card: "No-Show Handling"
  ├─ Switch: Enable grace period (toggles gracePeriodSec between 0 and 180)
  │  Caption: "Automatically handle no-shows..."
  ├─ Input: Grace period seconds (30–600, visible when enabled)
  ├─ Separator
  ├─ Select: No-show action (Skip / Re-queue, visible when enabled)
  ├─ Input: Max re-queues (visible when action = Re-queue)
  └─ Input: Re-queue offset (visible when action = Re-queue)
```

Uses same `PUT /api/stores/{storeId}/settings` endpoint.

### Step 5.13 — Client-Web: Handle SKIPPED/REQUEUED Status

**File**: `client-web/src/types/queue.ts`

Add `"SKIPPED" | "REQUEUED"` to `TicketStatus` type.

**File**: `client-web/src/components/queue/ticket-status-badge.tsx`

Add badge variants for SKIPPED (destructive) and REQUEUED (warning).

**File**: `client-web/src/components/queue/ticket-card.tsx`

Handle SKIPPED as a terminal state (similar to CANCELLED). For REQUEUED, show a message: "You've been moved back in the queue. Please wait for your turn."

### Step 5.14 — Translation Keys

Both `en.json` and `vi.json` for admin-web and client-web:

Admin-web additions (under `"queue"` and `"settings"`):

```json
"noShowButton": "No-Show",
"ticketSkipped": "Ticket skipped",
"ticketRequeued": "Ticket re-queued",
"keyboardShortcutsDescription": "Keyboard shortcuts: N to call next, S to serve, C to cancel, X for no-show"
```

Settings additions (under `"settings.store"`):

```json
"noShowHandling": "No-Show Handling",
"noShowHandlingDescription": "Configure how to handle customers who don't arrive when called",
"enableGracePeriod": "Enable grace period",
"enableGracePeriodCaption": "Automatically handle no-shows after a timeout. Disable for stores where customers are expected to be present immediately.",
"gracePeriodLabel": "Grace period (seconds)",
"gracePeriodCaption": "How long to wait for a called customer before marking no-show",
"noShowActionLabel": "No-show action",
"noShowActionSkip": "Skip",
"noShowActionRequeue": "Re-queue",
"maxRequeuesLabel": "Max re-queues",
"requeueOffsetLabel": "Re-queue offset"
```

Client-web additions (under `"status"` and `"queue"`):

```json
"statusSkipped": "Skipped",
"statusRequeued": "Re-queued",
"ticketSkippedMessage": "Your ticket was skipped. You may re-join the queue.",
"ticketRequeuedMessage": "You've been moved back in the queue. Please wait for your turn."
```

Vietnamese equivalents for all.

---

## Feature 10 Phase 2: Active Sessions

### Step 6.1 — Database Schema

**File**: `backend/src/main/resources/db/schema.sql` — append:

```sql
CREATE TABLE admin_session (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    admin_id      UUID NOT NULL REFERENCES admin(id) ON DELETE CASCADE,
    token_hash    VARCHAR(64) NOT NULL UNIQUE,
    ip_address    VARCHAR(45) NOT NULL,
    user_agent    TEXT,
    last_active   TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_admin_session_admin ON admin_session (admin_id);
CREATE INDEX idx_admin_session_token_hash ON admin_session (token_hash);
```

### Step 6.2 — Entity & DTO

**New file**: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/entity/AdminSession.kt`

```kotlin
@Table("admin_session")
data class AdminSession(
    @Id val id: UUID? = null,
    val adminId: UUID,
    val tokenHash: String,
    val ipAddress: String,
    val userAgent: String?,
    val lastActive: OffsetDateTime? = null,
    val createdAt: OffsetDateTime? = null
)
```

**New file**: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/dto/AdminSessionDto.kt`

```kotlin
data class AdminSessionDto(
    val id: String,
    val ipAddress: String,
    val userAgent: String?,
    val lastActive: OffsetDateTime?,
    val createdAt: OffsetDateTime?,
    val isCurrent: Boolean
)
```

### Step 6.3 — Repository

**New file**: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/repository/AdminSessionRepository.kt`

```kotlin
interface AdminSessionRepository : CoroutineCrudRepository<AdminSession, UUID> {
    fun findByAdminIdOrderByLastActiveDesc(adminId: UUID): Flow<AdminSession>
    suspend fun findByTokenHash(tokenHash: String): AdminSession?
    suspend fun deleteByTokenHash(tokenHash: String)
    suspend fun deleteByAdminId(adminId: UUID)
}
```

### Step 6.4 — Redis Key Additions

**File**: `backend/src/main/kotlin/com/thomas/notiguide/core/redis/RedisKeyManager.kt`

Add:

```kotlin
fun revokedToken(tokenHash: String) = "revoked:$tokenHash"
fun sessionLastUpdate(tokenHash: String) = "session:$tokenHash:last_update"
```

### Step 6.5 — Session Service

**New file**: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/service/SessionService.kt`

```kotlin
@Service
class SessionService(
    private val sessionRepository: AdminSessionRepository,
    private val redis: ReactiveRedisTemplate<String, String>,
    private val jwtProperties: JWTProperties
) {
    suspend fun createSession(adminId: UUID, tokenHash: String, ip: String, userAgent: String?): AdminSession {
        return sessionRepository.save(
            AdminSession(adminId = adminId, tokenHash = tokenHash, ipAddress = ip, userAgent = userAgent)
        )
    }

    suspend fun listSessions(adminId: UUID, currentTokenHash: String): List<AdminSessionDto> {
        return sessionRepository.findByAdminIdOrderByLastActiveDesc(adminId)
            .toList()
            .map { it.toDto(isCurrent = it.tokenHash == currentTokenHash) }
    }

    suspend fun revokeSession(sessionId: UUID, adminId: UUID, currentTokenHash: String) {
        val session = sessionRepository.findById(sessionId)
            ?: throw NotFoundException("Session", "id", sessionId.toString())
        if (session.adminId != adminId) throw ForbiddenException("Cannot revoke another admin's session")
        if (session.tokenHash == currentTokenHash) throw ConflictException("Cannot revoke current session")

        // Add to blacklist with TTL = remaining JWT lifetime
        val ttl = Duration.ofSeconds(jwtProperties.accessExpirySeconds)
        redis.opsForValue()
            .set(RedisKeyManager.revokedToken(session.tokenHash), "1", ttl)
            .awaitSingle()

        sessionRepository.deleteById(sessionId)
    }

    suspend fun updateLastActive(tokenHash: String) {
        // Throttle: only update if Redis key doesn't exist (5 min TTL)
        val throttleKey = RedisKeyManager.sessionLastUpdate(tokenHash)
        val set = redis.opsForValue()
            .setIfAbsent(throttleKey, "1", Duration.ofMinutes(5))
            .awaitSingle()
        if (set) {
            val session = sessionRepository.findByTokenHash(tokenHash) ?: return
            sessionRepository.save(session.copy(lastActive = OffsetDateTime.now()))
        }
    }

    suspend fun isRevoked(tokenHash: String): Boolean {
        return redis.hasKey(RedisKeyManager.revokedToken(tokenHash)).awaitSingle()
    }

    suspend fun deleteAllForAdmin(adminId: UUID) {
        sessionRepository.deleteByAdminId(adminId)
    }

    suspend fun deleteByTokenHash(tokenHash: String) {
        sessionRepository.deleteByTokenHash(tokenHash)
    }
}
```

### Step 6.6 — Token Hashing Utility

**New file**: `backend/src/main/kotlin/com/thomas/notiguide/core/jwt/TokenHashUtil.kt`

```kotlin
object TokenHashUtil {
    fun sha256(token: String): String {
        val digest = MessageDigest.getInstance("SHA-256")
        return digest.digest(token.toByteArray()).joinToString("") { "%02x".format(it) }
    }
}
```

### Step 6.7 — AuthController: Create Session on Login

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/controller/AuthController.kt`

After issuing the JWT in `login()`:

```kotlin
val tokenHash = TokenHashUtil.sha256(accessToken)
val userAgent = request.headers.getFirst("User-Agent")
val session = sessionService.createSession(admin.id!!, tokenHash, ip, userAgent)
```

Update `LoginResponse` to include `sessionId`:

```kotlin
data class LoginResponse(
    val admin: AdminDto,
    val sessionId: String? = null  // NEW — used by frontend to identify "this device"
)
```

### Step 6.8 — AuthController: Delete Session on Logout

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/controller/AuthController.kt`

In `logout()`, extract the access token and delete the session:

```kotlin
val accessToken = extractToken(request)
if (accessToken != null) {
    val tokenHash = TokenHashUtil.sha256(accessToken)
    sessionService.deleteByTokenHash(tokenHash)
}
```

### Step 6.9 — JWTAuthFilter: Check Blacklist + Update Last Active

**File**: `backend/src/main/kotlin/com/thomas/notiguide/core/jwt/JWTAuthFilter.kt`

After successful JWT verification (before `chain.filter()`):

```kotlin
val tokenHash = TokenHashUtil.sha256(token)

// Check blacklist
if (sessionService.isRevoked(tokenHash)) {
    writeErrorResponse(exchange, HttpStatus.UNAUTHORIZED, "Session revoked")
    return
}

// Update last active (fire-and-forget — don't block the request)
// Store tokenHash on exchange attributes for downstream use
exchange.attributes["tokenHash"] = tokenHash
sessionService.updateLastActive(tokenHash)
```

Inject `SessionService` into `JWTAuthFilter`.

### Step 6.10 — Session Endpoints

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/controller/AdminController.kt`

Add:

```kotlin
@GetMapping("/{id}/sessions")
suspend fun listSessions(
    @PathVariable id: UUID,
    @AuthenticationPrincipal principal: AdminPrincipal,
    request: ServerHttpRequest
): ResponseEntity<List<AdminSessionDto>> {
    if (principal.id != id) throw ForbiddenException("Can only view own sessions")
    val token = extractToken(request)
    val tokenHash = if (token != null) TokenHashUtil.sha256(token) else ""
    return ResponseEntity.ok(sessionService.listSessions(id, tokenHash))
}

@DeleteMapping("/{id}/sessions/{sessionId}")
suspend fun revokeSession(
    @PathVariable id: UUID,
    @PathVariable sessionId: UUID,
    @AuthenticationPrincipal principal: AdminPrincipal,
    request: ServerHttpRequest
): ResponseEntity<Void> {
    if (principal.id != id) throw ForbiddenException("Can only revoke own sessions")
    val token = extractToken(request)
    val tokenHash = if (token != null) TokenHashUtil.sha256(token) else ""
    sessionService.revokeSession(sessionId, id, tokenHash)
    return ResponseEntity.noContent().build()
}
```

### Step 6.11 — Session Cleanup Scheduler

**New file**: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/service/SessionCleanupScheduler.kt`

```kotlin
@Component
class SessionCleanupScheduler(private val client: DatabaseClient) {
    // Delete sessions older than JWT max lifetime (access + refresh combined = ~7 days)
    @Scheduled(cron = "0 30 3 * * *")  // daily at 3:30 AM
    fun cleanupExpiredSessions() = runBlocking {
        client.sql("DELETE FROM admin_session WHERE created_at < now() - INTERVAL '8 days'")
            .fetch().rowsUpdated().awaitSingle()
    }
}
```

### Step 6.12 — Admin-Web: Session Management UI

**File**: `web/src/app/[locale]/dashboard/settings/security/page.tsx`

Add "Active Sessions" card **above** "Login History":

```
Card: "Active Sessions"
  └─ List of session cards:
     ├─ Icon (Monitor/Smartphone based on User-Agent parsing)
     ├─ Browser/OS label + "This device" badge (if current)
     ├─ IP address · Last active "2 hours ago"
     └─ [Revoke] button (hidden for current session)
```

### Step 6.13 — Admin-Web: User-Agent Parsing

**New file**: `web/src/lib/user-agent.ts`

```typescript
export function parseUserAgent(ua: string | null): { browser: string; os: string; isMobile: boolean } {
    if (!ua) return { browser: "Unknown", os: "Unknown", isMobile: false };
    // Simple regex-based parsing:
    // Browser: Chrome, Firefox, Safari, Edge, Opera, etc.
    // OS: Windows, macOS, Linux, Android, iOS, etc.
    // isMobile: /Mobile|Android|iPhone/ test
}
```

### Step 6.14 — Admin-Web: Auth Store Update

**File**: `web/src/store/auth.ts`

Store `sessionId` from login response:

```typescript
interface AuthState {
    // ...existing...
    sessionId: string | null;
    login(response: LoginResponse): void;
}
```

In `login()`: `set({ sessionId: response.sessionId })`.

### Step 6.15 — Admin-Web Types & API

**File**: `web/src/types/admin.ts` — add:

```typescript
interface AdminSessionDto {
    id: string;
    ipAddress: string;
    userAgent: string | null;
    lastActive: string | null;
    createdAt: string | null;
    isCurrent: boolean;
}
```

**File**: `web/src/features/admin/api.ts` — add:

```typescript
export async function listSessions(adminId: string): Promise<AdminSessionDto[]> {
    return get(`/api/admins/${adminId}/sessions`);
}

export async function revokeSession(adminId: string, sessionId: string): Promise<void> {
    return del(`/api/admins/${adminId}/sessions/${sessionId}`);
}
```

### Step 6.16 — Translation Keys

**File**: `web/src/messages/en.json` — add under `"settings.security"`:

```json
"activeSessions": "Active Sessions",
"activeSessionsDescription": "Devices currently logged into your account",
"thisDevice": "This device",
"revoke": "Revoke",
"revokeConfirm": "Revoke this session? The device will be logged out.",
"revokeSuccess": "Session revoked",
"lastActive": "Last active {time}",
"unknownDevice": "Unknown device",
"unknownBrowser": "Unknown browser"
```

Vietnamese equivalents in `vi.json`.

---

## Feature 6: Multi-Counter / Service Type Routing

> This is the largest-scope feature. It restructures the Redis queue model from one queue per store to one queue per (store, service type).

### Step 7.1 — Database Schema

**File**: `backend/src/main/resources/db/schema.sql` — append:

```sql
CREATE TABLE service_type (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    store_id    UUID NOT NULL REFERENCES store(id) ON DELETE CASCADE,
    name        VARCHAR(100) NOT NULL,
    prefix      VARCHAR(5) NOT NULL,
    is_active   BOOLEAN NOT NULL DEFAULT true,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(store_id, prefix)
);

CREATE INDEX idx_service_type_store ON service_type (store_id);
```

### Step 7.2 — Auto-Create Default Service Type

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/service/StoreService.kt`

In `createStore()`, after saving the store and settings:

```kotlin
serviceTypeRepository.save(
    ServiceType(storeId = saved.id!!, name = "General", prefix = "A")
)
```

### Step 7.3 — Entity

**New file**: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/entity/ServiceType.kt`

```kotlin
@Table("service_type")
data class ServiceType(
    @Id val id: UUID? = null,
    val storeId: UUID,
    val name: String,
    val prefix: String,
    val isActive: Boolean = true,
    @CreatedDate val createdAt: OffsetDateTime? = null,
    @LastModifiedDate val updatedAt: OffsetDateTime? = null
)
```

### Step 7.4 — Repository

**New file**: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/repository/ServiceTypeRepository.kt`

```kotlin
interface ServiceTypeRepository : CoroutineCrudRepository<ServiceType, UUID> {
    fun findByStoreId(storeId: UUID): Flow<ServiceType>
    fun findByStoreIdAndIsActive(storeId: UUID, isActive: Boolean): Flow<ServiceType>
    suspend fun findByStoreIdAndPrefix(storeId: UUID, prefix: String): ServiceType?
}
```

### Step 7.5 — Service Type CRUD Endpoints

**New file**: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/controller/ServiceTypeController.kt`

```kotlin
@RestController
@RequestMapping("/api/stores/{storeId}/service-types")
class ServiceTypeController(
    private val serviceTypeService: ServiceTypeService
) {
    @GetMapping
    suspend fun list(@PathVariable storeId: UUID, @AuthenticationPrincipal principal: AdminPrincipal): ResponseEntity<List<ServiceTypeDto>>

    @PostMapping
    suspend fun create(@PathVariable storeId: UUID, @Valid @RequestBody request: CreateServiceTypeRequest, @AuthenticationPrincipal principal: AdminPrincipal): ResponseEntity<ServiceTypeDto>

    @PutMapping("/{id}")
    suspend fun update(@PathVariable storeId: UUID, @PathVariable id: UUID, @Valid @RequestBody request: UpdateServiceTypeRequest, @AuthenticationPrincipal principal: AdminPrincipal): ResponseEntity<ServiceTypeDto>

    @DeleteMapping("/{id}")
    suspend fun delete(@PathVariable storeId: UUID, @PathVariable id: UUID, @AuthenticationPrincipal principal: AdminPrincipal): ResponseEntity<Void>
}
```

### Step 7.6 — Redis Key Structure Change

**File**: `backend/src/main/kotlin/com/thomas/notiguide/core/redis/RedisKeyManager.kt`

Add service-type-scoped keys (keeping existing keys for backward compatibility during migration):

```kotlin
fun queue(storeId: UUID, serviceTypeId: UUID) = "store:$storeId:queue:$serviceTypeId"
fun serving(storeId: UUID, serviceTypeId: UUID) = "store:$storeId:serving:$serviceTypeId"
fun counterLanes(storeId: UUID, counterId: String) = "counter:$storeId:$counterId:lanes"
```

### Step 7.7 — QueueService: Service-Type-Aware Issue & Call

**Modify `issueTicket()`**: Accept optional `serviceTypeId` parameter. If not provided, use the store's default service type. Route to the service-type-specific queue ZSet.

```kotlin
suspend fun issueTicket(storeId: UUID, serviceTypeId: UUID? = null): TicketDto {
    val resolvedServiceType = if (serviceTypeId != null) {
        serviceTypeRepository.findById(serviceTypeId)
            ?: throw NotFoundException("ServiceType", "id", serviceTypeId.toString())
    } else {
        serviceTypeRepository.findByStoreIdAndIsActive(storeId, true).first()
    }
    // Use resolvedServiceType.prefix for ticket number prefix
    // Use queue(storeId, resolvedServiceType.id!!) for Redis queue key
}
```

**Modify `callNext()`**: Accept optional `serviceTypeId`. If admin's counter is assigned to specific lanes, pull from oldest across those lanes. If no lane restriction, pull from default.

```kotlin
suspend fun callNext(storeId: UUID, counterId: String?, serviceTypeId: UUID? = null): CallNextResult {
    // If counterId is set, check counter lane assignments
    // If assigned to specific lanes, find oldest ticket across those lanes
    // Otherwise, use the specified serviceTypeId or default
}
```

### Step 7.8 — QueuePublicController: Service Type Selection

**File**: `backend/src/main/kotlin/com/thomas/notiguide/domain/queue/controller/QueuePublicController.kt`

Update `POST /tickets` to accept optional `serviceTypeId`:

```kotlin
@PostMapping("/tickets")
suspend fun issueTicket(
    @PathVariable storeId: UUID,
    @RequestParam(required = false) serviceTypeId: UUID?
): ResponseEntity<IssueTicketResponse>
```

Add endpoint to list available service types:

```kotlin
@GetMapping("/service-types")
suspend fun listServiceTypes(@PathVariable storeId: UUID): ResponseEntity<List<ServiceTypePublicDto>>
```

### Step 7.9 — Ticket Transfer

**New endpoint** in `QueueAdminController`:

```kotlin
@PostMapping("/tickets/{ticketId}/transfer")
suspend fun transferTicket(
    @PathVariable storeId: UUID,
    @PathVariable ticketId: UUID,
    @RequestParam targetServiceTypeId: UUID,
    @AuthenticationPrincipal principal: AdminPrincipal
): ResponseEntity<Void>
```

Implementation: Atomically move ticket from current queue ZSet to target queue ZSet, preserving original `issued_at` score.

### Step 7.10 — Client-Web: Service Type Selection UI

**File**: `client-web/src/app/[locale]/store/[storeId]/page.tsx`

When a store has multiple active service types, show a selection screen before the "Join Queue" button:

```
Card: "Select Service"
  ├─ Button: "New Purchases" (Lane A)
  ├─ Button: "Returns & Exchanges" (Lane B)
  └─ Button: "Customer Service" (Lane C)
```

If the store has only one service type (the default), skip this screen entirely — maintain current behavior.

**New component**: `client-web/src/components/queue/service-type-selector.tsx`

```tsx
// Props: serviceTypes: ServiceTypePublicDto[], onSelect: (id: string) => void
// Renders glass cards for each service type
// Single service type: auto-selects, no UI shown
```

### Step 7.11 — Admin-Web: Service Type Management

**New file**: `web/src/features/store/service-type-management.tsx`

Card within the store edit dialog or stores page showing service types for a store. CRUD interface:

```
Card: "Service Types"
  ├─ Table: Name | Prefix | Status | Actions (Edit, Delete)
  └─ Button: Add Service Type
```

Only visible for stores, managed by SUPER_ADMIN.

### Step 7.12 — Admin-Web: Queue Page — Lane Awareness

**File**: `web/src/app/[locale]/dashboard/queue/page.tsx`

If the admin's store has multiple service types:
- Show a lane selector/filter above the waiting list
- "Call Next" pulls from the selected lane (or all lanes if "All" selected)
- Waiting list shows lane prefix on each ticket (e.g., "A-042", "B-015")

### Step 7.13 — Backward Compatibility

- Stores created before Feature 6 get a default service type via migration script
- All existing Redis keys (`store:{id}:queue`, `store:{id}:serving`) continue working for the default service type
- The migration creates a `service_type` row for each existing store and remaps Redis keys

**Migration script** (one-time, run after deployment):

```sql
-- Create default service type for existing stores that don't have one
INSERT INTO service_type (store_id, name, prefix, is_active)
SELECT id, 'General', 'A', true FROM store
WHERE id NOT IN (SELECT store_id FROM service_type);
```

### Step 7.14 — Translation Keys

Admin-web (under `"stores"`):

```json
"serviceTypes": "Service Types",
"serviceTypeName": "Service name",
"serviceTypePrefix": "Prefix",
"addServiceType": "Add service type",
"editServiceType": "Edit service type",
"deleteServiceType": "Delete service type",
"transferTicket": "Transfer ticket",
"transferTo": "Transfer to"
```

Client-web (under `"store"`):

```json
"selectService": "Select a service",
"selectServiceDescription": "Choose the type of service you need"
```

Vietnamese equivalents for all.

---

## Settings Tab Summary (Final State)

After all features are implemented:

```
Account | Password | Store | Security
```

- **Account**: Username, role, store, verification status (existing)
- **Password**: Change password (existing)
- **Store**: Default counter ID (Phase 1) + Queue limits (Phase 2) + No-show handling (Phase 3) — hidden for SUPER_ADMIN
- **Security**: Active sessions (10 Phase 2) + Login history (10 Phase 1)

---

## Global Cross-Cutting Concerns

### R2DBC EnumCodec Registration

**File**: `backend/src/main/kotlin/com/thomas/notiguide/core/database/R2DBCConfig.kt`

The existing `EnumCodec` registers `AdminRole`. If new enums are stored in Postgres (e.g., `analytics_event_type`), they must also be registered:

```kotlin
EnumCodec.builder()
    .withEnum("admin_role", AdminRole::class.java)
    .withEnum("analytics_event_type", AnalyticsEventType::class.java)
    .build()
```

### Security Config: No New Public Endpoints

Features 2, 4, 5, 9, 10 do NOT add new public endpoints. All new endpoints are under `/api/stores/**`, `/api/admins/**`, or `/api/queue/admin/**` — all already require authentication. The only public endpoint change is the `StorePublicInfoResponse` returning `queueState` and `maxQueueSize` (existing public endpoint, new fields only).

Feature 6 adds `GET /api/queue/public/{storeId}/service-types` — already under `/api/queue/public/**` which is permitted.

### SSE Event Types

**File**: `backend/src/main/kotlin/com/thomas/notiguide/core/sse/QueueEventBroadcaster.kt`

The `QueueSseEvent.type` field currently handles: `TICKET_ISSUED`, `TICKET_CALLED`, `TICKET_SERVED`, `TICKET_CANCELLED`.

New event types from Features 5 and 6:
- `TICKET_SKIPPED` — ticket was auto-skipped by grace period expiry
- `TICKET_REQUEUED` — ticket was re-queued after no-show
- `TICKET_TRANSFERRED` — ticket moved between service types

Admin-web `use-queue-events.ts` must handle these new types (trigger stats refresh + waiting list refresh).

Client-web `useTicketPolling` hook already polls for status — no SSE changes needed for client-web.

### Admin-Web Queue Event Types

**File**: `web/src/types/queue.ts`

Update `QueueEventType`:

```typescript
type QueueEventType =
    | "TICKET_ISSUED"
    | "TICKET_CALLED"
    | "TICKET_SERVED"
    | "TICKET_CANCELLED"
    | "TICKET_SKIPPED"
    | "TICKET_REQUEUED"
    | "TICKET_TRANSFERRED";
```

**File**: `web/src/hooks/use-queue-events.ts`

Add the new event types to the listener.

### Client-Web Ticket Status Type

**File**: `client-web/src/types/queue.ts`

```typescript
type TicketStatus = "WAITING" | "CALLED" | "SERVED" | "CANCELLED" | "SKIPPED" | "REQUEUED" | "UNKNOWN";
```

### CSS Patterns for New Components

All new cards/panels follow the existing glass morphism patterns:
- Cards use `.glass-card` class with `rounded-xl`
- Action buttons follow `.queue-action-btn` sizing pattern
- Stats use `.queue-stat-card` / `.queue-stat-value` / `.queue-stat-label` patterns
- Responsive spacing uses custom breakpoints: `s:`, `m:`, `l:`, `xl:`, `2xl:`, `3xl:`, `4xl:`
- Color tokens: `--primary` for teal, `--action` for orange, `--destructive` for red, `--success` for green
- Badge variants from existing `badge.tsx` CVA definitions

No new CSS files needed — all new components fit within existing design system patterns.

---

## Implementation Checklist

| # | Feature | Status | Backend Files | Admin-Web Files | Client-Web Files |
|---|---|---|---|---|---|
| 1 | F9 Ph1 (counter ID) | **TODO** | — | settings/store/page.tsx, queue/page.tsx, en.json, vi.json, settings layout | — |
| 2 | F10 Ph1 (login history) | **TODO** | LoginHistory entity/repo, AdminService, AuthController, AdminController, scheduler | settings/security/page.tsx, admin/api.ts, types/admin.ts, en.json, vi.json, settings layout | — |
| 3 | F9 Ph2 + F4 (pause/cap) | **TODO** | StoreSettings entity/repo, StoreService, StoreController, QueueService, QueueAdminController, RedisKeyManager, QueueState | queue-state-toggle.tsx, queue/api.ts, settings/store/page.tsx (+ allowJumpCall toggle), en.json, vi.json | store page (paused/full banners), queue-paused-banner.tsx, queue-full-banner.tsx, types/store.ts, en.json, vi.json |
| 4 | F2 (FCM alerts) | **TODO** | FcmNotificationService (proactive methods), QueueService (alert dispatch in callNext + callSpecificTicket) | — | firebase-messaging-sw.js (POSITION_ALERT handler), en.json, vi.json |
| 5 | F9 Ph3 + F5 (no-show) | **TODO** | TicketStatus (SKIPPED/REQUEUED), RedisKeyManager (grace expiry), QueueService (handleNoShow, requeue Lua), RedisTicketRepository (markSkipped), RedisKeyExpirationListener, QueueAdminController (/no-show) | serving-display.tsx (no-show button + timer), settings/store/page.tsx (no-show card), en.json, vi.json | ticket-status-badge.tsx, ticket-card.tsx (skipped/requeued), types/queue.ts, en.json, vi.json |
| 6 | F10 Ph2 (sessions) | **TODO** | AdminSession entity/repo, SessionService, TokenHashUtil, AuthController (session create/delete), JWTAuthFilter (blacklist check), AdminController (session endpoints), scheduler | settings/security/page.tsx (sessions card), lib/user-agent.ts, store/auth.ts (sessionId), admin/api.ts, types/admin.ts, en.json, vi.json | — |
| 7 | F6 (multi-counter) | **TODO** | ServiceType entity/repo, ServiceTypeController, ServiceTypeService, QueueService (service-type-aware issue/call/transfer), RedisKeyManager (scoped keys), QueuePublicController (service-types endpoint), migration | service-type-management.tsx, queue page (lane selector), store page updates, en.json, vi.json | service-type-selector.tsx, store page (service selection), types, en.json, vi.json |

---

## Risk Notes

1. **Feature 6 (Multi-Counter)** is the highest-risk change — it restructures Redis keys. Consider implementing behind a feature flag or as a separate deployment phase with rollback plan.
2. **Feature 10 Phase 2 (Sessions)** modifies `JWTAuthFilter` — the most security-critical code path. Test thoroughly with concurrent sessions and revocation.
3. **Feature 5 (Grace Period)** adds a circular dependency risk: `RedisKeyExpirationListener` → `QueueService` → (indirectly) `RedisKeyExpirationListener`. Use `Lazy<QueueService>` injection to break the cycle.
4. **Feature 2 (FCM Alerts)** in `callNext()` and `callSpecificTicket()` adds N additional Redis reads (one per threshold position). For typical thresholds (2–3), this is negligible. For very large thresholds, consider batching.
5. **Jump-Call + No-Show interaction**: When `allowJumpCall` is enabled (already implemented) and grace period is later enabled (Feature 5), multiple stacked called tickets could each have independent grace timers. The `handleNoShow()` method must handle individual ticket expiry without affecting other stacked tickets — this is naturally isolated since each grace key is ticket-specific.
