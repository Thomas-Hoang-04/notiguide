# Web Prerequisites — Backend Implementation Plan

3 backend prerequisites still needed before the web frontend can be built. Items #1, #4, #5 from the original list are already done; #6-#7 are deferred.

---

## Prerequisite Status

| # | Item | Status |
|---|------|--------|
| 1 | `GET /api/stores` list-all (paginated, SUPER_ADMIN) | ✅ Already exists in [StoreController](file:///home/thomas/Coding/notiguide/backend/src/main/kotlin/com/thomas/notiguide/domain/store/controller/StoreController.kt#27-90) |
| 2 | `GET /api/admins` global list (without required `storeId`) | ❌ **Needs work** |
| 3 | CORS configuration | ❌ **Needs work** |
| 4 | [UpdateStoreRequest](file:///home/thomas/Coding/notiguide/backend/src/main/kotlin/com/thomas/notiguide/domain/store/request/UpdateStoreRequest.kt#6-22) address null-vs-absent | ✅ Already fixed (audit round) |
| 5 | Backend audit bugs | ✅ All fixed (4 audit rounds) |
| 6 | `GET /api/queue/admin/{storeId}/serving` | 📋 Deferred |
| 7 | `GET /api/queue/admin/{storeId}/size` — queue size | ❌ **Needs work** (trivial) |

---

## Proposed Changes

### A. CORS Configuration (Required for Phase 1)

#### [NEW] [CorsConfig.kt](file:///home/thomas/Coding/notiguide/backend/src/main/kotlin/com/thomas/notiguide/core/config/CorsConfig.kt)
- Create a `CorsWebFilter` bean that allows requests from the frontend origin
- Allow origins from a configurable `app.cors.allowed-origins` property (default: `http://localhost:3000`)
- Allow methods: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `OPTIONS`
- Allow headers: `Authorization`, `Content-Type`
- Expose headers: none needed currently
- Allow credentials: `true` (for `Authorization` header)
- Max age: `3600` seconds

#### [MODIFY] [application.yaml](file:///home/thomas/Coding/notiguide/backend/src/main/resources/application.yaml)
- Add `app.cors.allowed-origins` property (list)

#### [MODIFY] [AppProperties.kt](file:///home/thomas/Coding/notiguide/backend/src/main/kotlin/com/thomas/notiguide/core/config/AppProperties.kt)
- Add `cors.allowedOrigins: List<String>` under the existing `app` prefix

---

### B. Global Admin Listing Endpoint (Required for Phase 4)

Currently `GET /api/admins` requires `storeId`. The plan calls for Super Admins to list all admins across stores.

#### [MODIFY] [AdminRepository.kt](file:///home/thomas/Coding/notiguide/backend/src/main/kotlin/com/thomas/notiguide/domain/admin/repository/AdminRepository.kt)
- Add [findAllPaged(limit: Long, offset: Long): Flow<Admin>](file:///home/thomas/Coding/notiguide/backend/src/main/kotlin/com/thomas/notiguide/domain/store/repository/StoreRepository.kt#13-15) — `SELECT * FROM admin ORDER BY created_at DESC LIMIT :limit OFFSET :offset`
- Add `countAll(): Long` — leverages `CoroutineCrudRepository.count()` which already exists, but an explicit alias keeps it symmetrical with [countByStoreId](file:///home/thomas/Coding/notiguide/backend/src/main/kotlin/com/thomas/notiguide/domain/admin/repository/AdminRepository.kt#18-20)
  - Actually, `CoroutineCrudRepository` already provides [count()](file:///home/thomas/Coding/notiguide/backend/src/main/kotlin/com/thomas/notiguide/core/redis/RedisKeyManager.kt#11-12), so no new method needed for the total count

#### [MODIFY] [AdminService.kt](file:///home/thomas/Coding/notiguide/backend/src/main/kotlin/com/thomas/notiguide/domain/admin/service/AdminService.kt)
- Add `listAllAdmins(page: Int, size: Int): AdminPageResponse` method
- Same validation and pagination logic as [listAdminsByStore](file:///home/thomas/Coding/notiguide/backend/src/main/kotlin/com/thomas/notiguide/domain/admin/controller/AdminController.kt#81-92) but using [findAllPaged](file:///home/thomas/Coding/notiguide/backend/src/main/kotlin/com/thomas/notiguide/domain/store/repository/StoreRepository.kt#13-15) and [count()](file:///home/thomas/Coding/notiguide/backend/src/main/kotlin/com/thomas/notiguide/core/redis/RedisKeyManager.kt#11-12)

#### [MODIFY] [AdminController.kt](file:///home/thomas/Coding/notiguide/backend/src/main/kotlin/com/thomas/notiguide/domain/admin/controller/AdminController.kt)
- Make `storeId` optional on the existing `GET /api/admins` endpoint: `@RequestParam(required = false) storeId: UUID?`
- When `storeId` is `null`: require `ROLE_SUPER_ADMIN`, call `adminService.listAllAdmins(page, size)`
- When `storeId` is provided: keep existing behavior ([requireStoreAccess](file:///home/thomas/Coding/notiguide/backend/src/main/kotlin/com/thomas/notiguide/shared/principal/StoreAccessUtil.kt#9-14) + [listAdminsByStore](file:///home/thomas/Coding/notiguide/backend/src/main/kotlin/com/thomas/notiguide/domain/admin/controller/AdminController.kt#81-92))

---

### C. Queue Size Endpoint (Needed for Phase 5)

`RedisQueueRepository.getQueueSize(storeId)` already exists but has no controller endpoint.

#### [MODIFY] [QueueAdminController.kt](file:///home/thomas/Coding/notiguide/backend/src/main/kotlin/com/thomas/notiguide/domain/queue/controller/QueueAdminController.kt)
- Add `GET /api/queue/admin/{storeId}/size` endpoint
- Returns `{ "queueSize": Long }` from `redisQueueRepository.getQueueSize(storeId)`
- Uses `StoreAccessUtil.requireStoreAccess` for authorization

#### [MODIFY] [QueueService.kt](file:///home/thomas/Coding/notiguide/backend/src/main/kotlin/com/thomas/notiguide/domain/queue/service/QueueService.kt)
- Add [getQueueSize(storeId: UUID): Long](file:///home/thomas/Coding/notiguide/backend/src/main/kotlin/com/thomas/notiguide/domain/queue/repository/RedisQueueRepository.kt#19-22) that delegates to `redisQueueRepository.getQueueSize(storeId)`

---

### D. Changelog

#### [MODIFY] [CHANGELOGS.md](file:///home/thomas/Coding/notiguide/docs/CHANGELOGS.md)
- Prepend new changelog entry for web prerequisites

---

## Verification Plan

### Manual Verification
Per user rules, the user will handle building and running the application to verify correctness.
