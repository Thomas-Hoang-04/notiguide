# Queue + Store Domain — Implementation Walkthrough

## Summary

Implemented the Queue Service and Controller layer along with the Store domain, resolving **19 audit findings** across 5 rounds.

---

## Files Modified (4)

### [RedisConfig.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/core/redis/RedisConfig.kt)
render_diffs(file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/core/redis/RedisConfig.kt)

### [SecurityConfig.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/core/security/SecurityConfig.kt)
render_diffs(file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/core/security/SecurityConfig.kt)

### [RedisCounterRepository.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/queue/repository/RedisCounterRepository.kt)
render_diffs(file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/queue/repository/RedisCounterRepository.kt)

### [AdminController.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/admin/controller/AdminController.kt)
render_diffs(file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/admin/controller/AdminController.kt)

---

## New Files (11)

### Shared Utility
| File | Purpose |
|---|---|
| [StoreAccessUtil.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/shared/principal/StoreAccessUtil.kt) | [requireStoreAccess()](file:///home/thomas/Coding/Java%20%28Kotlin%29/notiguide/src/main/kotlin/com/thomas/notiguide/shared/principal/StoreAccessUtil.kt#9-14) — SUPER_ADMIN bypass, ROLE_ADMIN store match |

### Store Domain
| File | Purpose |
|---|---|
| [Store.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/store/entity/Store.kt) | R2DBC entity mapped to `store` table |
| [StoreDto.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/store/dto/StoreDto.kt) | API response DTO |
| [StoreRepository.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/store/repository/StoreRepository.kt) | `CoroutineCrudRepository<Store, UUID>` |
| [StoreRequest.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/store/request/StoreRequest.kt) | Create/Update request DTOs |
| [StoreService.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/store/service/StoreService.kt) | CRUD + [deleteStore](file:///home/thomas/Coding/Java%20%28Kotlin%29/notiguide/src/main/kotlin/com/thomas/notiguide/domain/store/controller/StoreController.kt#62-71) guards active queue |
| [StoreController.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/store/controller/StoreController.kt) | `/api/stores` — all authenticated |

### Queue Domain
| File | Purpose |
|---|---|
| [TicketDto.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/queue/dto/TicketDto.kt) | DTOs: TicketDto, IssueTicketResponse, TicketStatusResponse, NextTicketResponse |
| [CallNextResult.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/queue/types/CallNextResult.kt) | Sealed class: Success / QueueEmpty / GhostTicketSkipped |
| [QueueService.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/queue/service/QueueService.kt) | Core service with Lua scripts |
| [QueuePublicController.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/queue/controller/QueuePublicController.kt) | `POST /tickets`, `GET /tickets/{id}` |
| [QueueAdminController.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/queue/controller/QueueAdminController.kt) | `GET /tickets/{id}`, `POST /next, /serve, /cancel, /cleanup` |

---

## Key Design Highlights

### Lua Scripts (Atomicity)
- **[issueTicket](file:///home/thomas/Coding/Java%20%28Kotlin%29/notiguide/src/main/kotlin/com/thomas/notiguide/domain/queue/controller/QueuePublicController.kt#21-28)**: `ZADD` + `HSET` + `EXPIRE` in one script — prevents ghost tickets and partial failures
- **[callNext](file:///home/thomas/Coding/Java%20%28Kotlin%29/notiguide/src/main/kotlin/com/thomas/notiguide/domain/queue/controller/QueueAdminController.kt#38-51)**: `ZPOPMIN` + `EXISTS` check + `SADD` serving + `HSET` status=CALLED + `EXPIRE` TTL downgrade (12h→30min) — with ghost-skip loop inside the script

### Security
- Public: `/api/queue/public/**` — permit all (issue + status only, no cancel)
- Admin: `/api/queue/admin/**` + `/api/stores/**` — JWT authenticated
- `StoreAccessUtil.requireStoreAccess()` on every admin queue and store GET endpoint

### Data Lifecycle
- `TICKET_WAITING` 12h → `TICKET_CALLED` 30min TTL downgrade in Lua
- Counter `EXPIREAT` midnight (not rolling 24h)
- [cleanupServingSet](file:///home/thomas/Coding/Java%20%28Kotlin%29/notiguide/src/main/kotlin/com/thomas/notiguide/domain/queue/service/QueueService.kt#213-225) removes orphaned entries
- [clearStoreData](file:///home/thomas/Coding/Java%20%28Kotlin%29/notiguide/src/main/kotlin/com/thomas/notiguide/domain/queue/service/QueueService.kt#226-240) uses `SCAN` for ticket HASH cleanup on store deletion
