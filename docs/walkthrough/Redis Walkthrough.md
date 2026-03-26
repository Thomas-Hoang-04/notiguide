# Implementation Walkthrough

## Phase 1: Schema & Redis Setup

### Files Created/Modified
| File | Purpose |
|---|---|
| [schema.sql](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/resources/db/schema.sql) | 4 tables, 2 enums, constraints, indexes, TimescaleDB hypertable |
| [redis.conf](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/resources/redis/redis.conf) | AOF persistence + keyspace notifications |
| [RedisConfig.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/core/redis/RedisConfig.kt) | `ReactiveRedisTemplate` bean |
| [compose.yaml](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/compose.yaml) | TimescaleDB + Redis services |

---

## Phase 2: Redis Repository Layer

### Files Created/Modified

| File | Purpose |
|---|---|
| [RedisKeyManager.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/core/redis/RedisKeyManager.kt) | Shared: all key patterns + parsing |
| [RedisTTLPolicy.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/core/redis/RedisTTLPolicy.kt) | Shared: TTL durations (12h, 30min, 48h) |
| [RedisQueueRepository.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/queue/repository/RedisQueueRepository.kt) | ZSET queue + SET serving |
| [RedisTicketRepository.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/queue/repository/RedisTicketRepository.kt) | HASH ticket sessions, dynamic TTL |
| [RedisCounterRepository.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/queue/repository/RedisCounterRepository.kt) | Atomic INCR daily counter |
| [RedisKeyExpirationListener.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/core/redis/RedisKeyExpirationListener.kt) | Wired listener with error-resilient cleanup |

### Audit Results

| # | Check | Result |
|---|---|---|
| 1 | All key patterns routed through `RedisKeyManager` | ✅ |
| 2 | All TTL durations centralized in `RedisTTLPolicy` | ✅ |
| 3 | No hardcoded key strings in repositories | ✅ |
| 4 | [popNext()](file:///home/thomas/Coding/Java%20%28Kotlin%29/notiguide/src/main/kotlin/com/thomas/notiguide/domain/queue/repository/RedisQueueRepository.kt#27-31) null safety | ✅ Fixed (`!!` → `?.`) |
| 5 | Listener error resilience | ✅ Fixed (`onErrorResume` added) |
| 6 | Listener uses `RedisKeyManager` for parsing, not repo | ✅ |
| 7 | Ticket key format `ticket:{storeId}:{ticketId}` consistent | ✅ |
| 8 | Import consistency (no wildcard imports) | ✅ |

### Future Note

> [!IMPORTANT]
> [markServed()](file:///home/thomas/Coding/Java%20%28Kotlin%29/notiguide/src/main/kotlin/com/thomas/notiguide/domain/queue/repository/RedisTicketRepository.kt#39-41) and [markCancelled()](file:///home/thomas/Coding/Java%20%28Kotlin%29/notiguide/src/main/kotlin/com/thomas/notiguide/domain/queue/repository/RedisTicketRepository.kt#42-44) in [RedisTicketRepository](file:///home/thomas/Coding/Java%20%28Kotlin%29/notiguide/src/main/kotlin/com/thomas/notiguide/domain/queue/repository/RedisTicketRepository.kt#11-57) immediately delete the ticket HASH. The **service layer** calling these must [getTicket()](file:///home/thomas/Coding/Java%20%28Kotlin%29/notiguide/src/main/kotlin/com/thomas/notiguide/domain/queue/repository/RedisTicketRepository.kt#45-48) first to read `issued_at`/`called_at` for `wait_duration_seconds` analytics computation before triggering delete.
