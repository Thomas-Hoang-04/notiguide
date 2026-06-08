# Custom Store Slugs Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let store admins attach up to 4 case-preserving, case-insensitively-unique vanity slugs (plus the immutable default) to their store, with a 30-day grace retirement and a confirmed hard-delete, resolvable by customers through any slug.

**Architecture:** A unified `store_public_id` table holds every identifier (default + aliases) with lifecycle columns; `store.public_id` is kept as an immutable write-once mirror so the existing surface is untouched. Resolution happens server-side in `StoreService` (no edge middleware). The admin dashboard gets a reusable `StoreSlugsPanel` consumed by both the ADMIN settings page and the SUPER_ADMIN store-management panel. `client-web` keeps the alias in the URL and emits the default identifier as a canonical link through Next.js metadata.

**Tech Stack:** Kotlin 2.3 / Spring Boot 3.5 WebFlux (coroutines) + Spring Data R2DBC + PostgreSQL enum (`slug_status`); Next.js 16 / React 19 / next-intl / shadcn-ui (admin `web` + `client-web`).

---

## Conventions & adaptations for this plan (read first)

This codebase deviates from the default writing-plans assumptions; these adaptations are deliberate and match the project's standing rules:

- **No automated tests / no TDD loop.** `backend/src/test` is empty and there is no frontend test runner. Verification steps are therefore **inspection + lint**, not `write-failing-test → run`.
- **Do NOT build/run the backend** after implementation (`./gradlew build`/`bootRun`/`test`). The project has a separate audit flow for compilation. Backend verification = code inspection only.
- **Frontend verification = lint first** (`yarn lint`, which is `biome check`). No build step.
- **No git commits.** Do not add `git add`/`git commit` steps; the human decides when to commit.
- **GitNexus discipline.** Before modifying an existing symbol (`StoreService.getStoreByPublicId`, `StoreService.createStore` if store creation logic is touched beyond the SQL generator, `StorePublicInfoResponse`, `R2DBCConfig`, `StoreSettingsPanel`, `client-web` store page), run `gitnexus_impact({target, direction:"upstream"})` and report the blast radius, per `CLAUDE.md`.
- **CHANGELOGS.** Every file change (including anything skipped) is logged to `docs/CHANGELOGS.md` — see Task 16.
- **Vietnamese copy.** The `vi.json` strings in Task 11 are **proposals**. Per `CLAUDE.md`, surface them to the user in chat for approval before writing, and never iterate VI edits blindly.
- **Phasing.** Phase 1–2 (DB + backend) must land before Phase 3–4 (frontends), which depend on the new endpoints/response fields.

Named query parameters in `@Query` use bare names (no `@Param`) — the project relies on Kotlin `-parameters`, matching `StoreRepository`/`AdminRepository`.

---

## File structure

**Backend (`backend/src/main/...`)**
- Create `kotlin/com/thomas/notiguide/domain/store/types/SlugStatus.kt` — enum.
- Create `kotlin/com/thomas/notiguide/domain/store/entity/StorePublicId.kt` — entity.
- Create `kotlin/com/thomas/notiguide/domain/store/repository/StorePublicIdRepository.kt` — queries.
- Create `kotlin/com/thomas/notiguide/domain/store/service/SlugValidator.kt` — format/blocklist.
- Create `kotlin/com/thomas/notiguide/domain/store/dto/StoreSlugDto.kt` — DTO + list response.
- Create `kotlin/com/thomas/notiguide/domain/store/request/CreateSlugRequest.kt` — request.
- Create `kotlin/com/thomas/notiguide/domain/store/service/StoreSlugService.kt` — lifecycle.
- Create `kotlin/com/thomas/notiguide/domain/store/controller/StoreSlugController.kt` — API.
- Create `kotlin/com/thomas/notiguide/domain/store/service/SlugGracePurgeScheduler.kt` — purge job.
- Modify `kotlin/com/thomas/notiguide/core/database/R2DBCConfig.kt` — register `slug_status` enum (codec + write converter).
- Modify `kotlin/com/thomas/notiguide/domain/store/service/StoreService.kt` — resolution via new repo + `resolvePublicId`.
- Modify `kotlin/com/thomas/notiguide/domain/queue/response/StorePublicInfoResponse.kt` — add `canonicalId`, `matchedSlug`.
- Modify `kotlin/com/thomas/notiguide/domain/queue/controller/QueuePublicController.kt` — populate new fields.
- Modify `resources/db/schema.sql` — new objects.
- Create `resources/db/migration/001_store_public_id.sql` — standalone backfill migration.

**Admin web (`web/src/...`)**
- Modify `types/store.ts`, `lib/constants.ts`, `features/store/api.ts`.
- Modify `messages/en.json`, `messages/vi.json`.
- Create `features/store/store-slugs-panel.tsx`, `features/store/slug-add-dialog.tsx`, `features/store/slug-retire-dialog.tsx`, `features/store/slug-remove-dialog.tsx`, `features/store/slugs-list.tsx`.
- Create `app/[locale]/dashboard/settings/slugs/page.tsx`; modify `app/[locale]/dashboard/settings/layout.tsx`.
- Modify `features/store/store-settings-panel.tsx` (SUPER_ADMIN tab).

**Client web (`client-web/src/...`)**
- Modify `types/store.ts`, `lib/constants.ts`, `app/[locale]/layout.tsx`, `app/[locale]/store/[storeId]/page.tsx`.
- Create `app/[locale]/store/[storeId]/store-page-content.tsx`.

---

## Phase 1 — Database

### Task 1: Schema + standalone backfill migration

**Files:**
- Modify: `backend/src/main/resources/db/schema.sql` (replace `generate_store_public_id()`, then add new objects after the `CREATE INDEX idx_store_public_id ON store(public_id);` line)
- Create: `backend/src/main/resources/db/migration/001_store_public_id.sql`

- [ ] **Step 1: Replace `generate_store_public_id()` in `schema.sql`**

Replace the existing function near the top of `schema.sql` with this collision-aware version. This keeps the existing DB-default store-creation path intact while ensuring generated defaults cannot collide case-insensitively with an alias already in `store_public_id`:

```sql
CREATE OR REPLACE FUNCTION generate_store_public_id() RETURNS TEXT AS $$
DECLARE
    candidate TEXT;
BEGIN
    LOOP
        -- 6 random bytes → 8 base64 chars; strip non-alphanumeric padding chars.
        candidate := regexp_replace(encode(gen_random_bytes(6), 'base64'), '[^A-Za-z0-9]', '', 'g');

        -- base64 of 6 bytes is always 8 chars but may contain +/=; keep looping until clean 8 chars.
        IF length(candidate) = 8 THEN
            IF to_regclass('store_public_id') IS NULL THEN
                IF NOT EXISTS (
                    SELECT 1 FROM store WHERE lower(public_id) = lower(candidate)
                ) THEN
                    RETURN candidate;
                END IF;
            ELSE
                IF NOT EXISTS (
                    SELECT 1 FROM store WHERE lower(public_id) = lower(candidate)
                ) AND NOT EXISTS (
                    SELECT 1 FROM store_public_id WHERE lower(slug) = lower(candidate)
                ) THEN
                    RETURN candidate;
                END IF;
            END IF;
        END IF;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

> `to_regclass('store_public_id')` lets the function exist before the new table is declared in fresh `schema.sql`; app-created stores happen after the full schema has initialized, so future defaults check both namespaces.

- [ ] **Step 2: Add the new objects to `schema.sql`**

Insert immediately after the existing `CREATE INDEX idx_store_public_id ON store(public_id);` line:

```sql
-- ===== Custom store slugs =====
CREATE TYPE slug_status AS ENUM ('ACTIVE', 'GRACE');

CREATE TABLE store_public_id (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    store_id    UUID NOT NULL REFERENCES store(id) ON DELETE CASCADE,
    slug        VARCHAR(128) NOT NULL,
    is_default  BOOLEAN NOT NULL DEFAULT FALSE,
    status      slug_status NOT NULL DEFAULT 'ACTIVE',
    retired_at  TIMESTAMP WITH TIME ZONE,
    expires_at  TIMESTAMP WITH TIME ZONE,
    created_at  TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT chk_default_active   CHECK (NOT is_default OR status = 'ACTIVE'),
    CONSTRAINT chk_grace_timestamps CHECK (
        (status = 'GRACE' AND retired_at IS NOT NULL AND expires_at IS NOT NULL)
        OR (status = 'ACTIVE' AND retired_at IS NULL AND expires_at IS NULL)
    )
);

-- Global uniqueness, case-insensitive — matches the resolver's lower(slug) key.
CREATE UNIQUE INDEX uq_store_public_id_slug ON store_public_id (lower(slug));
-- Exactly one default per store.
CREATE UNIQUE INDEX uq_store_public_id_default ON store_public_id (store_id) WHERE is_default;
-- Cap counting and purge scans.
CREATE INDEX idx_store_public_id_store_status ON store_public_id (store_id, status);
CREATE INDEX idx_store_public_id_expiry ON store_public_id (expires_at) WHERE status = 'GRACE';

-- Mirror each store's default public_id into store_public_id on insert.
CREATE OR REPLACE FUNCTION mirror_store_default_public_id() RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO store_public_id (store_id, slug, is_default, status)
    VALUES (NEW.id, NEW.public_id, TRUE, 'ACTIVE');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_mirror_store_default_public_id
    AFTER INSERT ON store
    FOR EACH ROW EXECUTE FUNCTION mirror_store_default_public_id();
```

- [ ] **Step 3: Create the standalone backfill migration** for already-running databases (schema.sql only runs on a fresh DB).

Create `backend/src/main/resources/db/migration/001_store_public_id.sql`:

```sql
-- Custom store slugs — apply once to an existing database.
-- Idempotent: safe to re-run.

DO $$
BEGIN
    CREATE TYPE slug_status AS ENUM ('ACTIVE', 'GRACE');
EXCEPTION
    WHEN duplicate_object THEN null;
END $$;

CREATE TABLE IF NOT EXISTS store_public_id (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    store_id    UUID NOT NULL REFERENCES store(id) ON DELETE CASCADE,
    slug        VARCHAR(128) NOT NULL,
    is_default  BOOLEAN NOT NULL DEFAULT FALSE,
    status      slug_status NOT NULL DEFAULT 'ACTIVE',
    retired_at  TIMESTAMP WITH TIME ZONE,
    expires_at  TIMESTAMP WITH TIME ZONE,
    created_at  TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT chk_default_active   CHECK (NOT is_default OR status = 'ACTIVE'),
    CONSTRAINT chk_grace_timestamps CHECK (
        (status = 'GRACE' AND retired_at IS NOT NULL AND expires_at IS NOT NULL)
        OR (status = 'ACTIVE' AND retired_at IS NULL AND expires_at IS NULL)
    )
);

CREATE UNIQUE INDEX IF NOT EXISTS uq_store_public_id_slug ON store_public_id (lower(slug));
CREATE UNIQUE INDEX IF NOT EXISTS uq_store_public_id_default ON store_public_id (store_id) WHERE is_default;
CREATE INDEX IF NOT EXISTS idx_store_public_id_store_status ON store_public_id (store_id, status);
CREATE INDEX IF NOT EXISTS idx_store_public_id_expiry ON store_public_id (expires_at) WHERE status = 'GRACE';

-- Future generated defaults must avoid aliases too, not only store.public_id.
CREATE OR REPLACE FUNCTION generate_store_public_id() RETURNS TEXT AS $$
DECLARE
    candidate TEXT;
BEGIN
    LOOP
        candidate := regexp_replace(encode(gen_random_bytes(6), 'base64'), '[^A-Za-z0-9]', '', 'g');
        IF length(candidate) = 8
            AND NOT EXISTS (SELECT 1 FROM store WHERE lower(public_id) = lower(candidate))
            AND NOT EXISTS (SELECT 1 FROM store_public_id WHERE lower(slug) = lower(candidate))
        THEN
            RETURN candidate;
        END IF;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION mirror_store_default_public_id() RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO store_public_id (store_id, slug, is_default, status)
    VALUES (NEW.id, NEW.public_id, TRUE, 'ACTIVE');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

DROP TRIGGER IF EXISTS trg_mirror_store_default_public_id ON store;
CREATE TRIGGER trg_mirror_store_default_public_id
    AFTER INSERT ON store
    FOR EACH ROW EXECUTE FUNCTION mirror_store_default_public_id();

-- Backfill existing stores' defaults.
INSERT INTO store_public_id (store_id, slug, is_default, status)
SELECT id, public_id, TRUE, 'ACTIVE' FROM store
ON CONFLICT (lower(slug)) DO NOTHING;
```

> Note: PostgreSQL `CREATE TYPE` has no `IF NOT EXISTS`; the migration wraps it in a `DO` block so the file is genuinely safe to re-run. `CREATE INDEX IF NOT EXISTS` and `ON CONFLICT (lower(slug)) DO NOTHING` cover the remaining replay cases.

- [ ] **Step 4: Verify by inspection** — `slug` is `VARCHAR(128)`; the unique index is on `lower(slug)`; the `ON CONFLICT (lower(slug))` target matches that functional index; the trigger inserts the default with `is_default = TRUE`; the standalone migration's `slug_status` creation is wrapped in a duplicate-safe `DO` block; both fresh schema and migration replace `generate_store_public_id()` so generated defaults avoid `store_public_id` aliases case-insensitively.

---

## Phase 2 — Backend

### Task 2: `SlugStatus` enum, `StorePublicId` entity, and R2DBC enum wiring

**Files:**
- Create: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/types/SlugStatus.kt`
- Create: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/entity/StorePublicId.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/core/database/R2DBCConfig.kt`

- [ ] **Step 1: Create the enum**

```kotlin
package com.thomas.notiguide.domain.store.types

enum class SlugStatus {
    ACTIVE,
    GRACE
}
```

- [ ] **Step 2: Create the entity** (matches the `ServiceType`/`Store` mapping style; generated UUID id → `save()` inserts when id is null, updates otherwise)

```kotlin
package com.thomas.notiguide.domain.store.entity

import com.thomas.notiguide.domain.store.dto.StoreSlugDto
import com.thomas.notiguide.domain.store.types.SlugStatus
import org.springframework.data.annotation.CreatedDate
import org.springframework.data.annotation.Id
import org.springframework.data.relational.core.mapping.Column
import org.springframework.data.relational.core.mapping.Table
import java.time.OffsetDateTime
import java.util.UUID

@Table("store_public_id")
data class StorePublicId(
    @Id
    @Column("id")
    val id: UUID? = null,

    @Column("store_id")
    val storeId: UUID,

    @Column("slug")
    val slug: String,

    @Column("is_default")
    val isDefault: Boolean = false,

    @Column("status")
    val status: SlugStatus = SlugStatus.ACTIVE,

    @Column("retired_at")
    val retiredAt: OffsetDateTime? = null,

    @Column("expires_at")
    val expiresAt: OffsetDateTime? = null,

    @CreatedDate
    @Column("created_at")
    val createdAt: OffsetDateTime? = null
) {
    fun toDto(): StoreSlugDto = StoreSlugDto(
        slug = slug,
        isDefault = isDefault,
        status = status.name,
        retiredAt = retiredAt,
        expiresAt = expiresAt,
        createdAt = createdAt
    )
}
```

- [ ] **Step 3: Register the `slug_status` enum in `R2DBCConfig.kt`** — three edits, mirroring the existing `admin_role`/`device_*` wiring.

Add the import alongside the other enum imports:

```kotlin
import com.thomas.notiguide.domain.store.types.SlugStatus
```

In `connectionFactory()`, add the `slug_status` line to the `EnumCodec.builder()` chain (after the `device_rf_ack_status` line):

```kotlin
                    .withEnum("device_rf_ack_status", DeviceRfAckStatus::class.java)
                    .withEnum("slug_status", SlugStatus::class.java)
                    .build()
```

Add the converter to `getCustomConverters()` (append to the list):

```kotlin
    @Bean
    override fun getCustomConverters(): List<Any?> = listOf(
        AdminRoleWriteConverter,
        DeviceKindWriteConverter,
        DeviceStatusWriteConverter,
        DeviceRfAckStatusWriteConverter,
        SlugStatusWriteConverter
    )
```

Add the converter object next to the other `*WriteConverter` objects:

```kotlin
    @WritingConverter
    object SlugStatusWriteConverter : EnumWriteSupport<SlugStatus>()
```

- [ ] **Step 4: Verify by inspection** — the enum is registered in all three places (codec, converter list, converter object); without the write converter, `CAST(:status AS slug_status)` param binding in Task 3 will fail.

---

### Task 3: `StorePublicIdRepository`

**Files:**
- Create: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/repository/StorePublicIdRepository.kt`

- [ ] **Step 1: Create the repository** (enum-filtered queries use `CAST(:status AS slug_status)`, exactly like `AdminRepository.countByStoreIdAndRole`; status-literal queries don't bind an enum so need no cast)

```kotlin
package com.thomas.notiguide.domain.store.repository

import com.thomas.notiguide.domain.store.entity.StorePublicId
import com.thomas.notiguide.domain.store.types.SlugStatus
import kotlinx.coroutines.flow.Flow
import org.springframework.data.r2dbc.repository.Modifying
import org.springframework.data.r2dbc.repository.Query
import org.springframework.data.repository.kotlin.CoroutineCrudRepository
import org.springframework.stereotype.Repository
import java.time.OffsetDateTime
import java.util.UUID

@Repository
interface StorePublicIdRepository : CoroutineCrudRepository<StorePublicId, UUID> {

    // Resolution: active + grace resolve transparently. Status literals need no cast.
    @Query("SELECT * FROM store_public_id WHERE lower(slug) = lower(:slug) AND status IN ('ACTIVE', 'GRACE')")
    suspend fun findResolvableBySlug(slug: String): StorePublicId?

    // Global uniqueness pre-check (any status, any store).
    @Query("SELECT * FROM store_public_id WHERE lower(slug) = lower(:slug)")
    suspend fun findAnyBySlug(slug: String): StorePublicId?

    // Locate a specific slug within a store for retire/delete (case-insensitive).
    @Query("SELECT * FROM store_public_id WHERE store_id = :storeId AND lower(slug) = lower(:slug)")
    suspend fun findByStoreIdAndSlug(storeId: UUID, slug: String): StorePublicId?

    // Listing: default first, then oldest alias first.
    @Query("SELECT * FROM store_public_id WHERE store_id = :storeId ORDER BY is_default DESC, created_at ASC")
    fun findByStoreId(storeId: UUID): Flow<StorePublicId>

    // Oldest active non-default alias — the auto-retire victim when at the active cap.
    @Query("SELECT * FROM store_public_id WHERE store_id = :storeId AND is_default = FALSE AND status = 'ACTIVE' ORDER BY created_at ASC LIMIT 1")
    suspend fun findOldestActiveAlias(storeId: UUID): StorePublicId?

    // Cap counting — enum param requires CAST to the Postgres enum type.
    @Query("SELECT COUNT(*) FROM store_public_id WHERE store_id = :storeId AND status = CAST(:status AS slug_status)")
    suspend fun countByStoreIdAndStatus(storeId: UUID, status: SlugStatus): Long

    // Grace purge — status literal, binds only the timestamp.
    @Modifying
    @Query("DELETE FROM store_public_id WHERE status = 'GRACE' AND expires_at <= :now")
    suspend fun deleteExpiredGrace(now: OffsetDateTime): Int
}
```

- [ ] **Step 2: Verify by inspection** — `@Modifying` imported from `org.springframework.data.r2dbc.repository.Modifying`; `@Query` from `org.springframework.data.r2dbc.repository.Query`; the only enum-bound query uses `CAST(:status AS slug_status)`.

---

### Task 4: `SlugValidator`

**Files:**
- Create: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/service/SlugValidator.kt`

- [ ] **Step 1: Create the validator** (throws `IllegalArgumentException` → 400 via `ExceptionHandler`; blocklist compared case-insensitively)

```kotlin
package com.thomas.notiguide.domain.store.service

object SlugValidator {
    const val MIN_LENGTH = 3
    const val MAX_LENGTH = 128

    // Alphanumeric segments joined by single hyphens; no leading/trailing/double hyphen.
    private val PATTERN = Regex("^[A-Za-z0-9]+(?:-[A-Za-z0-9]+)*$")

    // Compared case-insensitively. Extend with brand/profanity lists as needed.
    private val RESERVED: Set<String> = setOf(
        "api", "admin", "public", "store", "stores", "queue", "queues",
        "dashboard", "login", "logout", "auth", "www", "app", "assets",
        "static", "_next", "health", "actuator", "favicon", "robots", "sitemap"
    )

    /** @throws IllegalArgumentException with a user-facing message on any violation. */
    fun validate(slug: String) {
        require(slug.length in MIN_LENGTH..MAX_LENGTH) {
            "Slug must be between $MIN_LENGTH and $MAX_LENGTH characters"
        }
        require(PATTERN.matches(slug)) {
            "Slug may contain only letters, digits, and single hyphens (no leading, trailing, or repeated hyphens)"
        }
        require(slug.lowercase() !in RESERVED) {
            "This slug is reserved and cannot be used"
        }
    }
}
```

- [ ] **Step 2: Verify by inspection** — `validate` is pure and throws `IllegalArgumentException`; the caller (Task 6) calls it before the uniqueness/cap checks.

---

### Task 5: DTOs, request, and `StorePublicInfoResponse` update

**Files:**
- Create: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/dto/StoreSlugDto.kt`
- Create: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/request/CreateSlugRequest.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/queue/response/StorePublicInfoResponse.kt`

- [ ] **Step 1: Create the DTO + list response**

```kotlin
package com.thomas.notiguide.domain.store.dto

import java.time.OffsetDateTime

data class StoreSlugDto(
    val slug: String,
    val isDefault: Boolean,
    val status: String,
    val retiredAt: OffsetDateTime?,
    val expiresAt: OffsetDateTime?,
    val createdAt: OffsetDateTime?
)

data class StoreSlugListResponse(
    val items: List<StoreSlugDto>,
    val activeCount: Int,
    val activeMax: Int,
    val graceCount: Int,
    val graceMax: Int
)
```

- [ ] **Step 2: Create the request** (DTO-level format/length guard → 400 with field `details`; service re-validates via `SlugValidator`)

```kotlin
package com.thomas.notiguide.domain.store.request

import jakarta.validation.constraints.NotBlank
import jakarta.validation.constraints.Pattern
import jakarta.validation.constraints.Size

data class CreateSlugRequest(
    @field:NotBlank
    @field:Size(min = 3, max = 128)
    @field:Pattern(
        regexp = "^[A-Za-z0-9]+(?:-[A-Za-z0-9]+)*$",
        message = "Slug may contain only letters, digits, and single hyphens"
    )
    val slug: String,

    // Authorizes auto-retiring the oldest active alias when the store is at the
    // active cap and grace has room (see StoreSlugService.createAlias).
    val confirmAutoRetire: Boolean = false
)
```

- [ ] **Step 3: Add `canonicalId` + `matchedSlug` to `StorePublicInfoResponse`** (additive; defaults keep existing call sites compiling)

```kotlin
package com.thomas.notiguide.domain.queue.response

data class StorePublicInfoResponse(
    val publicId: String,
    val name: String,
    val address: String?,
    val isActive: Boolean,
    val queueState: String = "ACTIVE",
    val maxQueueSize: Int = 0,
    val canonicalId: String = "",
    val matchedSlug: String = ""
)
```

- [ ] **Step 4: Verify by inspection** — `StoreSlugDto.status` is a `String` (the enum name), so the frontend compares against `"ACTIVE"`/`"GRACE"`.

---

### Task 6: `StoreSlugService`

**Files:**
- Create: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/service/StoreSlugService.kt`

- [ ] **Step 1: Create the service** (caps: 5 active incl. default, 5 grace; grace = `now + gracePeriodDays`)

```kotlin
package com.thomas.notiguide.domain.store.service

import com.thomas.notiguide.core.exception.ConflictException
import com.thomas.notiguide.core.exception.NotFoundException
import com.thomas.notiguide.domain.store.dto.StoreSlugDto
import com.thomas.notiguide.domain.store.dto.StoreSlugListResponse
import com.thomas.notiguide.domain.store.entity.StorePublicId
import com.thomas.notiguide.domain.store.repository.StorePublicIdRepository
import com.thomas.notiguide.domain.store.repository.StoreRepository
import com.thomas.notiguide.domain.store.request.CreateSlugRequest
import com.thomas.notiguide.domain.store.types.SlugStatus
import kotlinx.coroutines.flow.toList
import org.springframework.beans.factory.annotation.Value
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.time.OffsetDateTime
import java.time.ZoneOffset
import java.util.UUID

@Service
class StoreSlugService(
    private val storePublicIdRepository: StorePublicIdRepository,
    private val storeRepository: StoreRepository,
    @Value($$"${slug.grace-period-days:30}") private val gracePeriodDays: Long
) {
    companion object {
        const val ACTIVE_CAP = 5  // includes the immutable default → 4 active aliases
        const val GRACE_CAP = 5
    }

    @Transactional(readOnly = true)
    suspend fun listSlugs(storeId: UUID): StoreSlugListResponse {
        storeRepository.findById(storeId) ?: throw NotFoundException("Store", "id", storeId.toString())
        val items = storePublicIdRepository.findByStoreId(storeId).toList().map { it.toDto() }
        return StoreSlugListResponse(
            items = items,
            activeCount = items.count { it.status == SlugStatus.ACTIVE.name },
            activeMax = ACTIVE_CAP,
            graceCount = items.count { it.status == SlugStatus.GRACE.name },
            graceMax = GRACE_CAP
        )
    }

    @Transactional
    suspend fun createAlias(storeId: UUID, request: CreateSlugRequest): StoreSlugDto {
        storeRepository.findById(storeId) ?: throw NotFoundException("Store", "id", storeId.toString())
        val slug = request.slug.trim()

        // Validate + uniqueness BEFORE any retire, so a bad/taken new slug can
        // never cost the admin an existing one.
        SlugValidator.validate(slug)
        if (storePublicIdRepository.findAnyBySlug(slug) != null) {
            throw ConflictException("Slug '$slug' is already taken")
        }

        val activeCount = storePublicIdRepository.countByStoreIdAndStatus(storeId, SlugStatus.ACTIVE)
        if (activeCount >= ACTIVE_CAP) {
            val graceCount = storePublicIdRepository.countByStoreIdAndStatus(storeId, SlugStatus.GRACE)
            if (graceCount >= GRACE_CAP) {
                throw ConflictException(
                    "Both the active and retiring limits are full. Remove a link immediately or wait for one to expire."
                )
            }
            if (!request.confirmAutoRetire) {
                throw ConflictException(
                    "Adding a link here will retire the oldest one. Confirm to proceed."
                )
            }
            // Atomically retire the oldest active alias (recomputed server-side).
            val oldest = storePublicIdRepository.findOldestActiveAlias(storeId)
                ?: throw ConflictException("No retireable link is available.")
            val now = OffsetDateTime.now(ZoneOffset.UTC)
            storePublicIdRepository.save(
                oldest.copy(
                    status = SlugStatus.GRACE,
                    retiredAt = now,
                    expiresAt = now.plusDays(gracePeriodDays)
                )
            )
        }

        val saved = storePublicIdRepository.save(
            StorePublicId(storeId = storeId, slug = slug, isDefault = false, status = SlugStatus.ACTIVE)
        )
        return saved.toDto()
    }

    @Transactional
    suspend fun retireAlias(storeId: UUID, slug: String): StoreSlugDto {
        val row = storePublicIdRepository.findByStoreIdAndSlug(storeId, slug.trim())
            ?: throw NotFoundException("Slug", "slug", slug)
        if (row.isDefault) throw ConflictException("The default identifier cannot be retired")
        if (row.status == SlugStatus.GRACE) throw ConflictException("Slug is already retiring")
        if (storePublicIdRepository.countByStoreIdAndStatus(storeId, SlugStatus.GRACE) >= GRACE_CAP) {
            throw ConflictException("Retiring slug limit reached ($GRACE_CAP). Remove one immediately first.")
        }

        val now = OffsetDateTime.now(ZoneOffset.UTC)
        val updated = row.copy(
            status = SlugStatus.GRACE,
            retiredAt = now,
            expiresAt = now.plusDays(gracePeriodDays)
        )
        return storePublicIdRepository.save(updated).toDto()
    }

    @Transactional
    suspend fun hardDeleteAlias(storeId: UUID, slug: String) {
        val row = storePublicIdRepository.findByStoreIdAndSlug(storeId, slug.trim())
            ?: throw NotFoundException("Slug", "slug", slug)
        if (row.isDefault) throw ConflictException("The default identifier cannot be deleted")
        storePublicIdRepository.delete(row)
    }
}
```

- [ ] **Step 2: Verify by inspection** — `@Value($$"${slug.grace-period-days:30}")` uses the Kotlin multi-dollar escape (matches `ServingSetCleanupScheduler`'s property string); `createAlias` validates + checks uniqueness *before* any retire; at the active cap it requires `confirmAutoRetire`, rejects when both buckets are full, and the retire + insert run in one `@Transactional`; retire checks the GRACE cap; default rows are protected in retire and delete (and never selected by `findOldestActiveAlias`, which filters `is_default = FALSE`).

---

### Task 7: Resolution refactor in `StoreService` + `QueuePublicController.getStoreInfo`

> Run `gitnexus_impact({target:"getStoreByPublicId", direction:"upstream"})` first — it is called by six `QueuePublicController` handlers. The signature is unchanged; only internals change.

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/service/StoreService.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/queue/controller/QueuePublicController.kt`

- [ ] **Step 1: Inject the repository** — add `storePublicIdRepository` to the `StoreService` constructor and the import.

Add import:
```kotlin
import com.thomas.notiguide.domain.store.repository.StorePublicIdRepository
```
Add the constructor parameter (e.g., after `storeRepository`):
```kotlin
    private val storeRepository: StoreRepository,
    private val storePublicIdRepository: StorePublicIdRepository,
```

- [ ] **Step 2: Add a `StoreResolution` holder + `resolvePublicId`, and rewrite `getStoreByPublicId` to delegate.** Replace the existing `getStoreByPublicId` method body with:

```kotlin
    data class StoreResolution(val store: StoreDto, val matchedSlug: String, val isDefault: Boolean)

    @Transactional(readOnly = true)
    suspend fun resolvePublicId(input: String): StoreResolution? {
        val normalized = input.trim()

        storePublicIdRepository.findResolvableBySlug(normalized)?.let { row ->
            val store = storeRepository.findById(row.storeId) ?: return null
            return StoreResolution(store.toDto(), row.slug, row.isDefault)
        }

        runCatching { UUID.fromString(normalized) }.getOrNull()
            ?.let { storeRepository.findById(it) }
            ?.let { return StoreResolution(it.toDto(), it.publicId!!, true) }

        return null
    }

    @Transactional(readOnly = true)
    suspend fun getStoreByPublicId(publicId: String): StoreDto =
        resolvePublicId(publicId)?.store
            ?: throw NotFoundException("Store", "publicId", publicId)
```

- [ ] **Step 3: Populate the new fields in `QueuePublicController.getStoreInfo`.** Replace the existing `getStoreInfo` handler with:

```kotlin
    @GetMapping("/info")
    suspend fun getStoreInfo(@PathVariable publicId: String): StorePublicInfoResponse {
        val resolution = storeService.resolvePublicId(publicId)
            ?: throw NotFoundException("Store", "publicId", publicId)
        val store = resolution.store
        val queueState = queueService.getQueueState(store.id)
        val settings = try {
            storeService.getStoreSettings(store.id)
        } catch (_: NotFoundException) { null }
        return StorePublicInfoResponse(
            publicId = store.publicId,
            name = store.name,
            address = store.address,
            isActive = store.isActive,
            queueState = queueState.name,
            maxQueueSize = settings?.maxQueueSize ?: 0,
            canonicalId = store.publicId,
            matchedSlug = resolution.matchedSlug
        )
    }
```

- [ ] **Step 4: Verify by inspection** — the other five public handlers still call `storeService.getStoreByPublicId(publicId)` unchanged; `resolvePublicId` is `null`-safe (returns `null` → handlers throw `NotFoundException` as before).

---

### Task 8: `StoreSlugController`

**Files:**
- Create: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/controller/StoreSlugController.kt`

- [ ] **Step 1: Create the controller** (mounted under the real `/api/stores` surface; scoped by `StoreAccessUtil` — ADMIN own store, SUPER_ADMIN any)

```kotlin
package com.thomas.notiguide.domain.store.controller

import com.thomas.notiguide.domain.store.dto.StoreSlugDto
import com.thomas.notiguide.domain.store.dto.StoreSlugListResponse
import com.thomas.notiguide.domain.store.request.CreateSlugRequest
import com.thomas.notiguide.domain.store.service.StoreSlugService
import com.thomas.notiguide.shared.principal.AdminPrincipal
import com.thomas.notiguide.shared.principal.StoreAccessUtil
import jakarta.validation.Valid
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.security.core.annotation.AuthenticationPrincipal
import org.springframework.web.bind.annotation.DeleteMapping
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.PathVariable
import org.springframework.web.bind.annotation.PostMapping
import org.springframework.web.bind.annotation.RequestBody
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController
import java.util.UUID

@RestController
@RequestMapping("/api/stores/{storeId}/slugs")
class StoreSlugController(
    private val storeSlugService: StoreSlugService
) {

    @GetMapping
    suspend fun list(
        @PathVariable storeId: UUID,
        @AuthenticationPrincipal principal: AdminPrincipal
    ): ResponseEntity<StoreSlugListResponse> {
        StoreAccessUtil.requireStoreAccess(principal, storeId)
        return ResponseEntity.ok(storeSlugService.listSlugs(storeId))
    }

    @PostMapping
    suspend fun create(
        @PathVariable storeId: UUID,
        @Valid @RequestBody request: CreateSlugRequest,
        @AuthenticationPrincipal principal: AdminPrincipal
    ): ResponseEntity<StoreSlugDto> {
        StoreAccessUtil.requireStoreAccess(principal, storeId)
        return ResponseEntity.status(HttpStatus.CREATED).body(storeSlugService.createAlias(storeId, request))
    }

    @PostMapping("/{slug}/retire")
    suspend fun retire(
        @PathVariable storeId: UUID,
        @PathVariable slug: String,
        @AuthenticationPrincipal principal: AdminPrincipal
    ): ResponseEntity<StoreSlugDto> {
        StoreAccessUtil.requireStoreAccess(principal, storeId)
        return ResponseEntity.ok(storeSlugService.retireAlias(storeId, slug))
    }

    @DeleteMapping("/{slug}")
    suspend fun delete(
        @PathVariable storeId: UUID,
        @PathVariable slug: String,
        @AuthenticationPrincipal principal: AdminPrincipal
    ): ResponseEntity<Void> {
        StoreAccessUtil.requireStoreAccess(principal, storeId)
        storeSlugService.hardDeleteAlias(storeId, slug)
        return ResponseEntity.noContent().build()
    }
}
```

- [ ] **Step 2: Verify by inspection** — base path is `/api/stores/...` (not `/api/admin/...`); `{slug}` only ever contains `[A-Za-z0-9-]` (no dots), so no Spring path-extension issues; SecurityConfig already authenticates `/api/stores/**` (no change needed).

---

### Task 9: `SlugGracePurgeScheduler`

**Files:**
- Create: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/service/SlugGracePurgeScheduler.kt`

- [ ] **Step 1: Create the scheduler** (mirrors `ServingSetCleanupScheduler`; `@EnableScheduling` is already on `NotiguideApplication`)

```kotlin
package com.thomas.notiguide.domain.store.service

import com.thomas.notiguide.domain.store.repository.StorePublicIdRepository
import kotlinx.coroutines.runBlocking
import org.slf4j.LoggerFactory
import org.springframework.scheduling.annotation.Scheduled
import org.springframework.stereotype.Component
import java.time.OffsetDateTime
import java.time.ZoneOffset

@Component
class SlugGracePurgeScheduler(
    private val storePublicIdRepository: StorePublicIdRepository
) {
    private val log = LoggerFactory.getLogger(this::class.java)

    @Scheduled(fixedDelayString = $$"${slug.grace-purge.fixed-delay-ms:300000}")
    fun purgeExpiredGrace() = runBlocking {
        try {
            val purged = storePublicIdRepository.deleteExpiredGrace(OffsetDateTime.now(ZoneOffset.UTC))
            if (purged > 0) {
                log.info("Slug grace purge: deleted {} expired slug(s)", purged)
            }
        } catch (ex: Exception) {
            log.warn("Slug grace purge failed", ex)
        }
    }
}
```

- [ ] **Step 2: Verify by inspection** — uses the `$$"..."` property-string escape and `runBlocking`, identical to `ServingSetCleanupScheduler`; logs with SLF4J parameterized format.

---

## Phase 3 — Admin web (`web`)

### Task 10: Types, route constants, and API functions

**Files:**
- Modify: `web/src/types/store.ts`
- Modify: `web/src/lib/constants.ts`
- Modify: `web/src/features/store/api.ts`

- [ ] **Step 1: Add slug types** to the end of `web/src/types/store.ts`:

```typescript
export interface StoreSlugDto {
  slug: string;
  isDefault: boolean;
  status: "ACTIVE" | "GRACE";
  retiredAt: string | null;
  expiresAt: string | null;
  createdAt: string | null;
}

export interface StoreSlugListResponse {
  items: StoreSlugDto[];
  activeCount: number;
  activeMax: number;
  graceCount: number;
  graceMax: number;
}

export interface CreateSlugRequest {
  slug: string;
  confirmAutoRetire?: boolean;
}
```

- [ ] **Step 2: Add route constants** inside the `STORES` object in `web/src/lib/constants.ts` (after the `SERVICE_TYPE` entry):

```typescript
    SLUGS: (id: string) => `/api/stores/${id}/slugs`,
    SLUG_RETIRE: (storeId: string, slug: string) =>
      `/api/stores/${storeId}/slugs/${encodeURIComponent(slug)}/retire`,
    SLUG: (storeId: string, slug: string) =>
      `/api/stores/${storeId}/slugs/${encodeURIComponent(slug)}`,
```

- [ ] **Step 3: Add API functions** to `web/src/features/store/api.ts`. Extend the type import and append the functions:

Add to the existing `import type { ... } from "@/types/store";` block: `CreateSlugRequest`, `StoreSlugDto`, `StoreSlugListResponse`.

```typescript
export function listSlugs(storeId: string) {
  return get<StoreSlugListResponse>(API_ROUTES.STORES.SLUGS(storeId));
}

export function createSlug(storeId: string, request: CreateSlugRequest) {
  return post<StoreSlugDto>(API_ROUTES.STORES.SLUGS(storeId), request);
}

export function retireSlug(storeId: string, slug: string) {
  return post<StoreSlugDto>(API_ROUTES.STORES.SLUG_RETIRE(storeId, slug), {});
}

export function removeSlug(storeId: string, slug: string) {
  return del<void>(API_ROUTES.STORES.SLUG(storeId, slug));
}
```

- [ ] **Step 4: Verify** — `cd web && yarn lint`. Expected: no new errors.

---

### Task 11: i18n keys (EN + proposed VI)

**Files:**
- Modify: `web/src/messages/en.json`
- Modify: `web/src/messages/vi.json`

> **VI strings below are proposals.** Per `CLAUDE.md`, surface them in chat for approval before writing `vi.json`. `en.json` and `vi.json` must stay structurally mirrored.

- [ ] **Step 1: Add the settings tab label.** In `settings` (both files), after `serviceTypesTab`:
  - `en.json`: `"slugsTab": "Custom links",`
  - `vi.json`: `"slugsTab": "Đường dẫn",`

- [ ] **Step 2: Add slug strings to the `stores` namespace** (both files).

`en.json` — add to `stores`:
```json
    "slugs": "Custom links",
    "slugsDescription": "Share your store through a custom link. The default link is permanent and can't be changed.",
    "slugColumn": "Link",
    "slugStatus": "Status",
    "addSlug": "Add link",
    "slugPlaceholder": "e.g. joes-coffee",
    "slugLabel": "Custom link",
    "slugDefault": "Default",
    "slugStatusActive": "Active",
    "slugStatusRetiring": "Retiring",
    "slugExpiresIn": "Expires in {days, plural, one {# day} other {# days}}",
    "slugCounts": "{active}/{activeMax} active · {grace}/{graceMax} retiring",
    "slugDefaultLocked": "Default link — can't be changed",
    "noSlugs": "No custom links yet",
    "retireSlug": "Retire",
    "retireSlugTitle": "Retire this link?",
    "retireSlugConfirm": "It keeps working for 30 days so existing links and QR codes stay valid, then it's removed automatically.",
    "removeSlug": "Remove now",
    "removeSlugTitle": "Remove this link immediately?",
    "removeSlugConfirm": "Any link or QR code using it breaks immediately. This can't be undone.",
    "slugCreated": "Link added",
    "slugRetired": "Link retired",
    "slugRemoved": "Link removed",
    "slugTaken": "This link is already taken",
    "slugReserved": "This link is reserved and can't be used",
    "slugInvalid": "Use 3–128 letters, digits, and hyphens",
    "slugConflictRefresh": "This link couldn't be added because the link state changed. Review the current limits and try again.",
    "slugAutoRetireWarning": "You're at the {max}-link limit. Adding this will start retiring the oldest link, \"{slug}\", which keeps working for 30 days before it's removed.",
    "slugBothFull": "You're at the limit for both active and retiring links. Remove one immediately, or wait for a retiring link to expire.",
    "slugGraceLimit": "You've reached the limit of {max} retiring links",
    "sectionSlugs": "Links",
```

`vi.json` — add to `stores` (proposed, native phrasing):
```json
    "slugs": "Đường dẫn tùy chỉnh",
    "slugsDescription": "Chia sẻ cửa hàng qua đường dẫn riêng. Đường dẫn mặc định là cố định, không thể thay đổi.",
    "slugColumn": "Đường dẫn",
    "slugStatus": "Trạng thái",
    "addSlug": "Thêm đường dẫn",
    "slugPlaceholder": "vd: quan-cafe-cua-toi",
    "slugLabel": "Đường dẫn tùy chỉnh",
    "slugDefault": "Mặc định",
    "slugStatusActive": "Đang dùng",
    "slugStatusRetiring": "Sắp ngừng",
    "slugExpiresIn": "Còn {days} ngày",
    "slugCounts": "{active}/{activeMax} đang dùng · {grace}/{graceMax} sắp ngừng",
    "slugDefaultLocked": "Đường dẫn mặc định — không thể thay đổi",
    "noSlugs": "Chưa có đường dẫn tùy chỉnh",
    "retireSlug": "Ngừng dùng",
    "retireSlugTitle": "Ngừng dùng đường dẫn này?",
    "retireSlugConfirm": "Đường dẫn vẫn hoạt động trong 30 ngày để các liên kết và mã QR cũ còn dùng được, sau đó tự động bị gỡ.",
    "removeSlug": "Gỡ ngay",
    "removeSlugTitle": "Gỡ đường dẫn này ngay?",
    "removeSlugConfirm": "Mọi liên kết hay mã QR đang dùng đường dẫn này sẽ hỏng ngay. Không thể hoàn tác.",
    "slugCreated": "Đã thêm đường dẫn",
    "slugRetired": "Đã ngừng đường dẫn",
    "slugRemoved": "Đã gỡ đường dẫn",
    "slugTaken": "Đường dẫn này đã có người dùng",
    "slugReserved": "Đường dẫn này không được phép dùng",
    "slugInvalid": "Dùng 3–128 ký tự gồm chữ, số và dấu gạch ngang",
    "slugConflictRefresh": "Chưa thể thêm đường dẫn vì trạng thái đường dẫn vừa thay đổi. Kiểm tra giới hạn hiện tại rồi thử lại.",
    "slugAutoRetireWarning": "Đã đạt giới hạn {max} đường dẫn. Thêm đường dẫn này sẽ bắt đầu ngừng đường dẫn cũ nhất, \"{slug}\" — vẫn hoạt động trong 30 ngày trước khi bị gỡ.",
    "slugBothFull": "Đã đạt giới hạn cả đường dẫn đang dùng lẫn sắp ngừng. Gỡ bớt một đường dẫn ngay, hoặc đợi một đường dẫn sắp ngừng hết hạn.",
    "slugGraceLimit": "Đã đạt giới hạn {max} đường dẫn sắp ngừng",
    "sectionSlugs": "Đường dẫn",
```

- [ ] **Step 2b: Add the `slugExpiresIn` plural form to `vi.json` only if confirmed** — Vietnamese has no plural inflection, so the simple `"Còn {days} ngày"` is correct; do not copy EN's ICU plural syntax.

- [ ] **Step 3: Verify** — both JSON files parse (`python3 -c "import json; json.load(open('web/src/messages/en.json')); json.load(open('web/src/messages/vi.json'))"`) and the two `stores` blocks contain the same keys.

---

### Task 12: Reusable slug components

**Files:**
- Create: `web/src/features/store/slugs-list.tsx`
- Create: `web/src/features/store/slug-add-dialog.tsx`
- Create: `web/src/features/store/slug-retire-dialog.tsx`
- Create: `web/src/features/store/slug-remove-dialog.tsx`
- Create: `web/src/features/store/store-slugs-panel.tsx`

- [ ] **Step 1: `slugs-list.tsx`** — presentational list of slug rows with status badges. Mirrors the table/badge idioms used elsewhere.

```tsx
"use client";

import { useTranslations } from "next-intl";
import { Badge } from "@/components/ui/badge";
import { Button } from "@/components/ui/button";
import { Skeleton } from "@/components/ui/skeleton";
import type { StoreSlugDto } from "@/types/store";

function daysUntil(expiresAt: string | null): number {
  if (!expiresAt) return 0;
  const ms = new Date(expiresAt).getTime() - Date.now();
  return Math.max(0, Math.ceil(ms / 86_400_000));
}

interface SlugsListProps {
  items: StoreSlugDto[];
  loading: boolean;
  atGraceCap: boolean;
  onRetire: (slug: StoreSlugDto) => void;
  onRemove: (slug: StoreSlugDto) => void;
}

export function SlugsList({
  items,
  loading,
  atGraceCap,
  onRetire,
  onRemove,
}: SlugsListProps) {
  const tStores = useTranslations("stores");

  if (loading) {
    return (
      <div className="space-y-2">
        <Skeleton className="h-12 w-full rounded-lg" />
        <Skeleton className="h-12 w-full rounded-lg" />
      </div>
    );
  }

  if (items.length === 0) {
    return (
      <p className="py-6 text-center text-sm text-muted-foreground">
        {tStores("noSlugs")}
      </p>
    );
  }

  return (
    <ul className="divide-y divide-border rounded-lg border border-border">
      {items.map((item) => (
        <li
          key={item.slug}
          className="flex flex-wrap items-center justify-between gap-2 px-3 py-2.5"
        >
          <div className="flex min-w-0 flex-col gap-1">
            <span className="truncate font-mono text-sm">{item.slug}</span>
            <div className="flex flex-wrap items-center gap-1.5">
              {item.isDefault && (
                <Badge variant="secondary">{tStores("slugDefault")}</Badge>
              )}
              {item.status === "ACTIVE" && !item.isDefault && (
                <Badge variant="outline">{tStores("slugStatusActive")}</Badge>
              )}
              {item.status === "GRACE" && (
                <Badge
                  variant="outline"
                  className="border-warning/40 bg-warning/15 text-warning dark:border-warning/50 dark:bg-warning/20"
                >
                  {tStores("slugStatusRetiring")} ·{" "}
                  {tStores("slugExpiresIn", { days: daysUntil(item.expiresAt) })}
                </Badge>
              )}
            </div>
          </div>

          {item.isDefault ? (
            <span className="text-xs text-muted-foreground">
              {tStores("slugDefaultLocked")}
            </span>
          ) : (
            <div className="flex shrink-0 gap-1.5">
              {item.status === "ACTIVE" && (
                <Button
                  variant="outline"
                  size="sm"
                  onClick={() => onRetire(item)}
                  disabled={atGraceCap}
                  title={atGraceCap ? tStores("slugGraceLimit", { max: 5 }) : undefined}
                >
                  {tStores("retireSlug")}
                </Button>
              )}
              <Button
                variant="ghost"
                size="sm"
                className="text-destructive hover:text-destructive"
                onClick={() => onRemove(item)}
              >
                {tStores("removeSlug")}
              </Button>
            </div>
          )}
        </li>
      ))}
    </ul>
  );
}
```

- [ ] **Step 2: `slug-add-dialog.tsx`** — Dialog + form, mirroring `service-type-form-dialog.tsx` (`details.slug` → field error, 409 → localized conflict toast + panel refresh, client-side format guard).

```tsx
"use client";

import { AlertTriangle, Loader2 } from "lucide-react";
import { useTranslations } from "next-intl";
import { type SyntheticEvent, useEffect, useState } from "react";
import { toast } from "sonner";
import { Button } from "@/components/ui/button";
import {
  Dialog,
  DialogContent,
  DialogFooter,
  DialogHeader,
  DialogTitle,
} from "@/components/ui/dialog";
import { InlineError } from "@/components/ui/inline-error";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { createSlug } from "@/features/store/api";
import {
  translateCommonApiError,
  translateNetworkError,
} from "@/lib/api-error";
import { ApiError } from "@/types/api";

const SLUG_PATTERN = /^[A-Za-z0-9]+(?:-[A-Za-z0-9]+)*$/;
const SLUG_MIN = 3;
const SLUG_MAX = 128;

interface SlugAddDialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  storeId: string;
  willAutoRetire: boolean;
  oldestActiveSlug: string | null;
  onSuccess: () => void;
}

export function SlugAddDialog({
  open,
  onOpenChange,
  storeId,
  willAutoRetire,
  oldestActiveSlug,
  onSuccess,
}: SlugAddDialogProps) {
  const tCommon = useTranslations("common");
  const tErrors = useTranslations("errors");
  const tStores = useTranslations("stores");

  const [slug, setSlug] = useState("");
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    if (open) {
      setSlug("");
      setError(null);
      setLoading(false);
    }
  }, [open]);

  function validate(value: string): boolean {
    if (value.length < SLUG_MIN || value.length > SLUG_MAX || !SLUG_PATTERN.test(value)) {
      setError(tStores("slugInvalid"));
      return false;
    }
    return true;
  }

  async function handleSubmit(e: SyntheticEvent<HTMLFormElement, SubmitEvent>) {
    e.preventDefault();
    if (loading) return;
    const trimmed = slug.trim();
    if (!validate(trimmed)) return;

    setLoading(true);
    setError(null);
    try {
      // Submitting with the auto-retire warning visible IS the "ask first"
      // consent, so authorize the server-side auto-retire.
      await createSlug(storeId, { slug: trimmed, confirmAutoRetire: willAutoRetire });
      toast.success(tStores("slugCreated"));
      onOpenChange(false);
      void onSuccess();
    } catch (err) {
      if (err instanceof ApiError) {
        if (err.details?.slug) {
          setError(err.details.slug);
        } else if (err.code === 409) {
          // A 409 can mean slug taken, confirmation required after stale data,
          // or full buckets after a concurrent change. Refresh the panel so the
          // visible limits and auto-retire warning catch up before retry.
          toast.error(tStores("slugConflictRefresh"));
          void onSuccess();
        } else {
          toast.error(translateCommonApiError(err, tErrors));
        }
      } else {
        toast.error(translateNetworkError(tErrors));
      }
    } finally {
      setLoading(false);
    }
  }

  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent>
        <DialogHeader>
          <DialogTitle>{tStores("addSlug")}</DialogTitle>
        </DialogHeader>

        <form onSubmit={handleSubmit} noValidate className="space-y-4">
          {willAutoRetire && oldestActiveSlug && (
            <div className="flex items-start gap-2.5 rounded-xl border border-warning/40 bg-warning/15 p-3 text-xs text-warning dark:border-warning/50 dark:bg-warning/20">
              <AlertTriangle aria-hidden="true" className="mt-0.5 size-4 shrink-0" />
              <span>
                {tStores("slugAutoRetireWarning", {
                  max: 5,
                  slug: oldestActiveSlug,
                })}
              </span>
            </div>
          )}
          <div className="space-y-1.5">
            <Label htmlFor="slug-input">{tStores("slugLabel")}</Label>
            <Input
              id="slug-input"
              value={slug}
              onChange={(e) => setSlug(e.target.value)}
              placeholder={tStores("slugPlaceholder")}
              maxLength={SLUG_MAX}
              aria-invalid={!!error}
              disabled={loading}
            />
            {error && <InlineError message={error} />}
          </div>

          <DialogFooter>
            <Button
              type="button"
              variant="outline"
              onClick={() => onOpenChange(false)}
              disabled={loading}
            >
              {tCommon("cancel")}
            </Button>
            <Button type="submit" disabled={loading}>
              {loading && <Loader2 className="mr-2 size-4 animate-spin" />}
              {tCommon("create")}
            </Button>
          </DialogFooter>
        </form>
      </DialogContent>
    </Dialog>
  );
}
```

- [ ] **Step 3: `slug-retire-dialog.tsx`** — AlertDialog confirming the 30-day retirement (non-destructive but meaningful).

```tsx
"use client";

import { Loader2 } from "lucide-react";
import { useTranslations } from "next-intl";
import { useState } from "react";
import { toast } from "sonner";
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
} from "@/components/ui/alert-dialog";
import { retireSlug } from "@/features/store/api";
import {
  translateCommonApiError,
  translateNetworkError,
} from "@/lib/api-error";
import { ApiError } from "@/types/api";
import type { StoreSlugDto } from "@/types/store";

interface SlugRetireDialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  slug: StoreSlugDto | null;
  storeId: string;
  onSuccess: () => void;
}

export function SlugRetireDialog({
  open,
  onOpenChange,
  slug,
  storeId,
  onSuccess,
}: SlugRetireDialogProps) {
  const tCommon = useTranslations("common");
  const tErrors = useTranslations("errors");
  const tStores = useTranslations("stores");
  const [loading, setLoading] = useState(false);

  async function handleRetire() {
    if (!slug) return;
    setLoading(true);
    try {
      await retireSlug(storeId, slug.slug);
      toast.success(tStores("slugRetired"));
      onOpenChange(false);
      void onSuccess();
    } catch (err) {
      if (err instanceof ApiError && err.code === 409) {
        // Retire is only offered for active, non-default slugs, so a 409 here
        // means the retiring (grace) limit is full.
        toast.error(tStores("slugGraceLimit", { max: 5 }));
      } else if (err instanceof ApiError) {
        toast.error(translateCommonApiError(err, tErrors));
      } else {
        toast.error(translateNetworkError(tErrors));
      }
    } finally {
      setLoading(false);
    }
  }

  return (
    <AlertDialog open={open} onOpenChange={onOpenChange}>
      <AlertDialogContent>
        <AlertDialogHeader>
          <AlertDialogTitle>{tStores("retireSlugTitle")}</AlertDialogTitle>
          <AlertDialogDescription>
            {tStores("retireSlugConfirm")}
          </AlertDialogDescription>
        </AlertDialogHeader>
        <AlertDialogFooter>
          <AlertDialogCancel disabled={loading}>
            {tCommon("cancel")}
          </AlertDialogCancel>
          <AlertDialogAction
            onClick={(e) => {
              e.preventDefault();
              void handleRetire();
            }}
            disabled={loading}
          >
            {loading && <Loader2 className="mr-2 size-4 animate-spin" />}
            {tStores("retireSlug")}
          </AlertDialogAction>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  );
}
```

- [ ] **Step 4: `slug-remove-dialog.tsx`** — destructive AlertDialog (hard-delete), mirroring `delete-service-type-dialog.tsx` styling.

```tsx
"use client";

import { Loader2 } from "lucide-react";
import { useTranslations } from "next-intl";
import { useState } from "react";
import { toast } from "sonner";
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
} from "@/components/ui/alert-dialog";
import { removeSlug } from "@/features/store/api";
import {
  translateCommonApiError,
  translateNetworkError,
} from "@/lib/api-error";
import { ApiError } from "@/types/api";
import type { StoreSlugDto } from "@/types/store";

interface SlugRemoveDialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  slug: StoreSlugDto | null;
  storeId: string;
  onSuccess: () => void;
}

export function SlugRemoveDialog({
  open,
  onOpenChange,
  slug,
  storeId,
  onSuccess,
}: SlugRemoveDialogProps) {
  const tCommon = useTranslations("common");
  const tErrors = useTranslations("errors");
  const tStores = useTranslations("stores");
  const [loading, setLoading] = useState(false);

  async function handleRemove() {
    if (!slug) return;
    setLoading(true);
    try {
      await removeSlug(storeId, slug.slug);
      toast.success(tStores("slugRemoved"));
      onOpenChange(false);
      void onSuccess();
    } catch (err) {
      if (err instanceof ApiError) {
        toast.error(translateCommonApiError(err, tErrors));
      } else {
        toast.error(translateNetworkError(tErrors));
      }
    } finally {
      setLoading(false);
    }
  }

  return (
    <AlertDialog open={open} onOpenChange={onOpenChange}>
      <AlertDialogContent>
        <AlertDialogHeader>
          <AlertDialogTitle>{tStores("removeSlugTitle")}</AlertDialogTitle>
          <AlertDialogDescription>
            {tStores("removeSlugConfirm")}
          </AlertDialogDescription>
        </AlertDialogHeader>
        <AlertDialogFooter>
          <AlertDialogCancel disabled={loading}>
            {tCommon("cancel")}
          </AlertDialogCancel>
          <AlertDialogAction
            onClick={(e) => {
              e.preventDefault();
              void handleRemove();
            }}
            disabled={loading}
            className="bg-destructive text-destructive-foreground hover:bg-destructive/90"
          >
            {loading && <Loader2 className="mr-2 size-4 animate-spin" />}
            {tCommon("delete")}
          </AlertDialogAction>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  );
}
```

- [ ] **Step 5: `store-slugs-panel.tsx`** — the reusable orchestrator consumed by both surfaces (takes `storeId`).

```tsx
"use client";

import { Plus } from "lucide-react";
import { useTranslations } from "next-intl";
import { useCallback, useEffect, useState } from "react";
import { Button } from "@/components/ui/button";
import { listSlugs } from "@/features/store/api";
import { SlugAddDialog } from "@/features/store/slug-add-dialog";
import { SlugRemoveDialog } from "@/features/store/slug-remove-dialog";
import { SlugRetireDialog } from "@/features/store/slug-retire-dialog";
import { SlugsList } from "@/features/store/slugs-list";
import { StoreManagementErrorBanner } from "@/features/store/store-management-error-banner";
import {
  translateCommonApiError,
  translateNetworkError,
} from "@/lib/api-error";
import { ApiError } from "@/types/api";
import type { StoreSlugDto, StoreSlugListResponse } from "@/types/store";

interface StoreSlugsPanelProps {
  storeId: string;
}

export function StoreSlugsPanel({ storeId }: StoreSlugsPanelProps) {
  const tStores = useTranslations("stores");

  const [data, setData] = useState<StoreSlugListResponse | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const [addOpen, setAddOpen] = useState(false);
  const [retireTarget, setRetireTarget] = useState<StoreSlugDto | null>(null);
  const [removeTarget, setRemoveTarget] = useState<StoreSlugDto | null>(null);

  const t = useTranslations("errors");

  const fetchSlugs = useCallback(async () => {
    setLoading(true);
    setError(null);
    try {
      setData(await listSlugs(storeId));
    } catch (err) {
      setError(
        err instanceof ApiError
          ? translateCommonApiError(err, t)
          : translateNetworkError(t),
      );
    } finally {
      setLoading(false);
    }
  }, [storeId, t]);

  useEffect(() => {
    void fetchSlugs();
  }, [fetchSlugs]);

  const atActiveCap = data ? data.activeCount >= data.activeMax : false;
  const atGraceCap = data ? data.graceCount >= data.graceMax : false;
  const bothFull = atActiveCap && atGraceCap;
  const willAutoRetire = atActiveCap && !atGraceCap;
  // items are ordered default-first then created_at ASC, so the first ACTIVE
  // non-default is the oldest active alias (the auto-retire victim).
  const oldestActiveSlug =
    data?.items.find((i) => i.status === "ACTIVE" && !i.isDefault)?.slug ?? null;

  return (
    <div className="space-y-4">
      <div className="flex items-start justify-between gap-3">
        <div className="space-y-1">
          <h3 className="text-sm font-semibold">{tStores("slugs")}</h3>
          <p className="text-xs text-muted-foreground">
            {tStores("slugsDescription")}
          </p>
        </div>
        <Button size="sm" onClick={() => setAddOpen(true)} disabled={bothFull}>
          <Plus className="mr-1.5 size-4" />
          {tStores("addSlug")}
        </Button>
      </div>

      {data && (
        <p className="text-xs text-muted-foreground">
          {tStores("slugCounts", {
            active: data.activeCount,
            activeMax: data.activeMax,
            grace: data.graceCount,
            graceMax: data.graceMax,
          })}
        </p>
      )}

      {bothFull && (
        <p className="rounded-xl border border-warning/40 bg-warning/15 p-3 text-xs text-warning dark:border-warning/50 dark:bg-warning/20">
          {tStores("slugBothFull")}
        </p>
      )}

      {error && (
        <StoreManagementErrorBanner error={error} onRetry={() => void fetchSlugs()} />
      )}

      <SlugsList
        items={data?.items ?? []}
        loading={loading}
        atGraceCap={atGraceCap}
        onRetire={setRetireTarget}
        onRemove={setRemoveTarget}
      />

      <SlugAddDialog
        open={addOpen}
        onOpenChange={setAddOpen}
        storeId={storeId}
        willAutoRetire={willAutoRetire}
        oldestActiveSlug={oldestActiveSlug}
        onSuccess={() => void fetchSlugs()}
      />
      <SlugRetireDialog
        open={retireTarget != null}
        onOpenChange={(v) => !v && setRetireTarget(null)}
        slug={retireTarget}
        storeId={storeId}
        onSuccess={() => void fetchSlugs()}
      />
      <SlugRemoveDialog
        open={removeTarget != null}
        onOpenChange={(v) => !v && setRemoveTarget(null)}
        slug={removeTarget}
        storeId={storeId}
        onSuccess={() => void fetchSlugs()}
      />
    </div>
  );
}
```

- [ ] **Step 6: Verify** — `cd web && yarn lint`. Confirm all five files lint clean and imports resolve.

---

### Task 13: ADMIN settings page + nav tab

**Files:**
- Create: `web/src/app/[locale]/dashboard/settings/slugs/page.tsx`
- Modify: `web/src/app/[locale]/dashboard/settings/layout.tsx`

- [ ] **Step 1: Create the settings page** (mirrors `service-types/page.tsx`; `storeId` from `useAuthStore`)

```tsx
"use client";

import { StoreSlugsPanel } from "@/features/store/store-slugs-panel";
import { useAuthStore } from "@/store/auth";

export default function SlugsSettingsPage() {
  const { storeId } = useAuthStore();

  if (!storeId) return null;

  return (
    <div className="mx-auto max-w-2xl">
      <StoreSlugsPanel storeId={storeId} />
    </div>
  );
}
```

> The `StoreSlugsPanel` already renders its own heading, description, counts, and Add button, so the page is a thin host — no duplicate `h2`.

- [ ] **Step 2: Add the nav tab** to `settings/layout.tsx`. Add `Link2` to the existing `lucide-react` import, then add this entry inside the `!isSuperAdmin` array, after the `service-types` entry:

```tsx
          {
            href: "/dashboard/settings/slugs",
            label: "slugsTab" as const,
            icon: Link2,
          },
```

- [ ] **Step 3: Verify** — `cd web && yarn lint`; confirm the tab renders only for non-super-admins (it's inside the `!isSuperAdmin` block) and routes to `/dashboard/settings/slugs`.

---

### Task 14: SUPER_ADMIN slugs tab in `StoreSettingsPanel`

**Files:**
- Modify: `web/src/features/store/store-settings-panel.tsx`

- [ ] **Step 1: Add a `slugs` tab** to the panel. Apply these edits:

Add the import:
```tsx
import { Link2, Settings, ShieldCheck, Users } from "lucide-react";
import { StoreSlugsPanel } from "@/features/store/store-slugs-panel";
```

Extend the `TabId` type and `TABS`:
```tsx
type TabId = "general" | "admins" | "queue" | "slugs";

const TABS: { id: TabId; icon: typeof Settings }[] = [
  { id: "general", icon: ShieldCheck },
  { id: "admins", icon: Users },
  { id: "queue", icon: Settings },
  { id: "slugs", icon: Link2 },
];
```

Add the label case in `getTabLabel`:
```tsx
      case "slugs":
        return tStores("sectionSlugs");
```

Add the tab content (after the `queue` block):
```tsx
        {activeTab === "slugs" && <StoreSlugsPanel storeId={store.id} />}
```

- [ ] **Step 2: Verify** — `cd web && yarn lint`; SUPER_ADMIN can now reach slug management for any store via the expandable store-management row, reusing the same panel as the ADMIN page.

---

## Phase 4 — Client web (`client-web`)

### Task 15: Keep alias in URL + canonical link

> Run `gitnexus_context({name:"StorePage", file_path:"client-web/src/app/[locale]/store/[storeId]/page.tsx"})` to disambiguate, then `gitnexus_impact({target:"StorePage", direction:"upstream"})` before modifying the current route. Current audit result: LOW risk, 0 upstream dependents. After the split, also run impact on the new `StorePageContent` symbol before further edits.

**Files:**
- Modify: `client-web/src/types/store.ts`
- Modify: `client-web/src/lib/constants.ts`
- Modify: `client-web/src/app/[locale]/layout.tsx`
- Modify: `client-web/src/app/[locale]/store/[storeId]/page.tsx`
- Create: `client-web/src/app/[locale]/store/[storeId]/store-page-content.tsx`

- [ ] **Step 1: Add the new response fields** to `StorePublicInfoResponse` in `client-web/src/types/store.ts`:

```typescript
export interface StorePublicInfoResponse {
  publicId: string;
  name: string;
  address: string | null;
  isActive: boolean;
  queueState: string;
  maxQueueSize: number;
  canonicalId: string;
  matchedSlug: string;
}
```

- [ ] **Step 2: Relax the redirect.** After moving the client code to `store-page-content.tsx` in Step 4, replace the existing effect from the current `page.tsx` implementation (currently at lines ~106–108):

```tsx
  useEffect(() => {
    if (storeInfo && storeInfo.publicId !== storeId) {
      router.replace(`/store/${storeInfo.publicId}`);
    }
  }, [storeInfo, storeId, router]);
```

with:

```tsx
  // Keep the alias in the URL. Only snap raw UUID access to the canonical id,
  // and cosmetically normalize a case-mismatched alias to its registered casing.
  useEffect(() => {
    if (!storeInfo) return;
    if (UUID_RE.test(storeId)) {
      router.replace(`/store/${storeInfo.canonicalId}`);
      return;
    }
    if (storeInfo.matchedSlug && storeInfo.matchedSlug !== storeId) {
      router.replace(`/store/${storeInfo.matchedSlug}`);
    }
  }, [storeInfo, storeId, router]);
```

- [ ] **Step 3: Add the UUID regex** near the top of the client component file (module scope, after imports):

```tsx
const UUID_RE =
  /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;
```

- [ ] **Step 4: Split the current page into a server route file plus a client content component.**

Create `store-page-content.tsx` as a mechanical move of the current `page.tsx` client implementation:
- Keep the `"use client"` directive and all current hook/component imports in `store-page-content.tsx`.
- Move `UUID_RE` from Step 3 into `store-page-content.tsx`.
- Remove the current `StorePageProps` interface, `StorePage` default export, and `use(params)` wrapper from the moved file.
- Export the current inner content function as `StorePageContent({ storeId }: { storeId: string })`.
- Apply the redirect replacement from Step 2 inside this moved client component.

Replace `page.tsx` with a Server Component that renders the client content and exports `generateMetadata`:

```tsx
import type { Metadata } from "next";
import { API_BASE_URL } from "@/lib/constants";
import type { StorePublicInfoResponse } from "@/types/store";
import { StorePageContent } from "./store-page-content";

interface StorePageProps {
  params: Promise<{ locale: string; storeId: string }>;
}

async function fetchStoreInfoForMetadata(
  storeId: string,
): Promise<StorePublicInfoResponse | null> {
  try {
    const response = await fetch(
      `${API_BASE_URL}/api/queue/public/${encodeURIComponent(storeId)}/info`,
      { cache: "no-store" },
    );
    if (!response.ok) return null;
    return (await response.json()) as StorePublicInfoResponse;
  } catch {
    return null;
  }
}

export async function generateMetadata({
  params,
}: StorePageProps): Promise<Metadata> {
  const { locale, storeId } = await params;
  const storeInfo = await fetchStoreInfoForMetadata(storeId);
  if (!storeInfo) return {};

  return {
    alternates: {
      canonical: `/${locale}/store/${storeInfo.canonicalId}`,
    },
  };
}

export default async function StorePage({ params }: StorePageProps) {
  const { storeId } = await params;

  return <StorePageContent storeId={storeId} />;
}
```

- [ ] **Step 5: Add a site metadata base** so the relative canonical path above is valid in Next.js.

In `client-web/src/lib/constants.ts`, add:

```typescript
export const SITE_URL =
  process.env.NEXT_PUBLIC_SITE_URL ?? "http://localhost:3000";
```

In `client-web/src/app/[locale]/layout.tsx`, import `SITE_URL` and add `metadataBase` to the existing `generateMetadata` return object:

```tsx
import { SITE_URL } from "@/lib/constants";
```

```tsx
  return {
    metadataBase: new URL(SITE_URL),
    title: t("title"),
    description: t("description"),
    // existing metadata fields...
  };
```

- [ ] **Step 6: Verify** — `cd client-web && yarn lint`. Confirm: visiting an alias no longer redirects to the default; visiting a UUID redirects to `canonicalId`; visiting a wrong-cased alias normalizes to `matchedSlug`; the initial HTML contains one canonical link for `/{locale}/store/{canonicalId}`.

---

## Phase 5 — Documentation

### Task 16: CHANGELOGS

**Files:**
- Modify: `docs/CHANGELOGS.md`

- [ ] **Step 1: Append a dated entry** summarizing every file created/modified across Tasks 1–15 (backend table/enum/repo/validator/service/controller/scheduler/resolution + R2DBC enum wiring + migration; admin-web types/constants/api/i18n/components/pages/panel tab; client-web types/constants/layout/page split/redirect/canonical metadata). Note explicitly anything skipped or deferred (e.g., Redis resolution caching, slug-reuse quarantine, slug analytics — all out of scope per the spec).

- [ ] **Step 2: Verify** — the entry lists the standalone migration file and notes the VI strings were confirmed with the user before writing.

---

## Self-review & audit record

### Spec coverage

| Spec section | Implemented by |
|---|---|
| §4 Data model (`store_public_id`, indexes, mirror trigger, backfill) | Task 1 |
| §4.2 `store.public_id` immutable mirror | Task 1 trigger + no entity change |
| §5 Lifecycle & caps (5 active / 5 grace, retire/hard-delete/purge) | Tasks 6, 9 |
| §5.3 Auto-retire oldest on create-at-cap, with `confirmAutoRetire` consent | Tasks 3, 5, 6, 12 |
| §6 Validation (format, 3–128, blocklist, case-insensitive uniqueness) | Tasks 4, 5 (DTO), 6 |
| §6 Case-preserving + case-insensitive uniqueness (`UNIQUE(lower(slug))`) | Tasks 1, 3 |
| §7 Resolution via new table, grace transparent, UUID fallback | Task 7 |
| §7.1 No edge middleware (server-side resolver) | Tasks 7, 15 |
| §8 Backend components (entity/repo/service/controller/scheduler/DTOs) | Tasks 2, 3, 5, 6, 8, 9 |
| §8 Postgres enum wiring (codec + write converter + CAST) | Tasks 2, 3 |
| §9 Admin API scoped by `StoreAccessUtil` | Task 8 |
| §10 client-web keep-alias + canonical + UUID/case redirect | Task 15 |
| §11 Admin dashboard Slugs panel (ADMIN + SUPER_ADMIN), bilingual | Tasks 11, 12, 13, 14 |
| §15 CHANGELOGS | Task 16 |

### Placeholder scan
No `TBD`/`TODO`/"handle errors"/"similar to". All code blocks are complete. The `vi.json` strings are concrete **proposals** flagged for user confirmation (per `CLAUDE.md`), not placeholders.

### Type consistency (cross-task)
`StoreSlugDto` / `StoreSlugListResponse` / `CreateSlugRequest` / `StorePublicInfoResponse(+canonicalId,+matchedSlug)` match byte-for-byte between backend (Tasks 5, 7) and frontend TS (Tasks 10, 15). Route constants (Task 10) match controller paths (Task 8). Service method names (`createAlias`/`retireAlias`/`hardDeleteAlias`/`listSlugs`) and repository methods (`findResolvableBySlug`/`findAnyBySlug`/`findByStoreIdAndSlug`/`findByStoreId`/`countByStoreIdAndStatus`/`deleteExpiredGrace`) are each defined where referenced.

### Post-review design change — auto-retire-with-confirmation (flow re-verification)
A flow re-verification revealed the original "block at the active cap" model was
less ergonomic than intended. The agreed behavior is now: at the active cap with
grace room, **adding auto-retires the oldest active alias** (lowest `created_at`,
never the default) into the 30-day grace — but only after the admin confirms (the
dialog names the slug). Implemented atomically server-side, gated by
`confirmAutoRetire`, with validation/uniqueness *before* the retire so a
bad/taken new slug never costs an existing one; blocked only when both buckets
are full. This stays loophole-free because the grace bucket is still capped at 5
(≤ 10 resolving total). Touched: Task 3 (`findOldestActiveAlias`), Task 5
(`confirmAutoRetire` field), Task 6 (`createAlias` rewrite), Tasks 11–12
(`slugAutoRetireWarning`/`slugBothFull`, panel gating on `bothFull`, add-dialog
warning). Spec §1/§5/§8.6/§9/§11 updated to match.

### Issues found during audit — amended inline
1. **`slugs-list.tsx` imported unused `useFormatter`** → would fail `biome check`. Removed; added an `atGraceCap` prop and disabled the Retire button (with a hint) when the grace bucket is full — closes a reachable failed-retire path.
2. **Add/Retire dialogs matched on the backend's English `err.message` substring (`"limit"`)** — brittle across wording/locale. Replaced Add's 409 handling with a localized conflict toast plus panel refresh, because a 409 can be slug-taken, confirmation-required after stale data, or bucket-full after a concurrent change; Retire's 409 still maps to "grace limit" because Retire is only offered for active non-default slugs. No dependency on message text.
3. **ADMIN settings page rendered a duplicate `h2`** (same text as the panel's own header) and left `tStores` unused → simplified to a thin host of `StoreSlugsPanel`.
4. **`store-slugs-panel.tsx` didn't compute/pass `atGraceCap`** required by the amended `SlugsList` → added.
5. **Spec ↔ codebase path drift** (`/api/admin/stores` vs real `/api/stores`) → plan uses the correct `/api/stores`; the spec §9 was corrected to match.
6. **Standalone migration claimed idempotency but used bare `CREATE TYPE slug_status`** → wrapped type creation in a duplicate-safe `DO` block; PostgreSQL supports `IF NOT EXISTS` for indexes but not for `CREATE TYPE`.
7. **Client canonical link used a client-rendered `<link>` / React hoisting approach** → replaced with Next.js `generateMetadata` + `alternates.canonical`, added `metadataBase`, and split the client page into a server route file plus `store-page-content.tsx`.
8. **Inline warning UI snippets used partial warning classes** → updated the retiring badge, auto-retire warning, and both-full warning to the canonical warning palette with required dark-mode overrides.
9. **Spec edge cases still said "Create when active full: rejected"** despite the confirmed auto-retire flow → corrected the spec edge case and aligned hard-delete dialog wording with the repo's `AlertDialog` convention.
10. **Task 15 incorrectly said `client-web` was not GitNexus-indexed** → verified `StorePage` is indexed, ran upstream impact (LOW, 0 dependents), and replaced the note with the valid `context`-then-`impact` sequence.
11. **Store creation/default-id collision path was missing** → replaced `generate_store_public_id()` in both fresh schema and standalone migration so generated defaults avoid existing aliases case-insensitively before the mirror trigger inserts the default row.
12. **Add-dialog 409 mapping still assumed every conflict meant "slug taken"** → changed Add 409 handling to show a localized conflict/refresh message and refetch counts; `details.slug` remains the only field-level path.

### Integration assumptions verified against the live codebase
- Postgres enum needs **three** wirings (`EnumCodec` read, `EnumWriteSupport` write, `CAST(:x AS slug_status)` in enum-param `@Query`) — confirmed against `admin_role`; Task 2/3 cover all three.
- `@Modifying` is **new** to this codebase but is the correct R2DBC mechanism (Context7-verified); package `org.springframework.data.r2dbc.repository.Modifying`, sibling of the already-used `@Query`.
- `Long` count vs `Int` cap (`count >= ACTIVE_CAP`) is idiomatic — `AdminService` does `superAdminCount <= 1`.
- web `del<void>` / 204 handled by `lib/api.ts` (line 101); `ApiError` exposes `.code`/`.details`; `Badge` has `secondary`/`outline`/`destructive`; `useAuthStore` exposes `storeId`.
- `@EnableScheduling` already present; SecurityConfig already authenticates `/api/stores/**`; the mirror trigger reads `NEW.public_id` after the DB default has populated it.
- Next.js `generateMetadata` + `alternates.canonical` are the correct canonical-link API (Context7-verified for Next.js 16); the client page must be split so metadata stays in a Server Component. `next-intl` `router.replace` remains the existing localized navigation API from `client-web/src/i18n/navigation.ts`.
- GitNexus indexes `client-web`; `StorePage` currently has LOW upstream impact with 0 direct callers/dependents.
- `generate_store_public_id()` currently only returns a clean random 8-character candidate; Task 1 replaces it so the DB-default store creation path remains compatible with `uq_store_public_id_slug`.

### Residual accepted limitations (by design, per spec)
- Cross-request cap race (rare, per-store, single-admin) — accepted, mirrors the documented SUPER_ADMIN deletion race.
- No Redis resolution cache, no slug-reuse quarantine, no slug analytics — explicitly out of scope (§14).
- Destructive hard-delete uses an `AlertDialog` (matching the existing `delete-service-type-dialog`) rather than an inline warning box; the canonical warning palette is used for non-dialog warning/status surfaces.
