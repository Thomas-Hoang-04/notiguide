# Unused Repository Methods — Kept for Future Use

Methods that currently have no callers but are intentionally retained for planned features.

## RedisCounterRepository

| Method | Reason Kept |
|---|---|
| `getCurrentCount(storeId)` | Dashboard feature: display "tickets issued today" per store. Will be used by a store stats endpoint or admin dashboard API. |

---

## Removed Methods (2026-03-12)

Methods superseded by Lua scripts in `QueueService` or redundant with other methods. Removed to reduce dead code.

### RedisQueueRepository
| Method | Reason Removed |
|---|---|
| `addToQueue(storeId, ticketId)` | Replaced by `ISSUE_TICKET_SCRIPT` Lua (atomic ZADD + HSET + EXPIRE) |
| `peekNext(storeId)` | Replaced by `CALL_NEXT_SCRIPT` Lua (atomic ZPOPMIN + EXISTS + SADD + HSET) |
| `popNext(storeId)` | Replaced by `CALL_NEXT_SCRIPT` Lua |
| `addToServing(storeId, ticketId)` | Replaced by `CALL_NEXT_SCRIPT` Lua |

### RedisTicketRepository
| Method | Reason Removed |
|---|---|
| `createTicket(storeId, ticketId, number)` | Replaced by `ISSUE_TICKET_SCRIPT` Lua |
| `markCalled(storeId, ticketId, counterId)` | Replaced by `CALL_NEXT_SCRIPT` Lua |
| `getStatus(storeId, ticketId)` | Redundant — `getTicket()` is always used instead (fetches full hash for additional fields) |
