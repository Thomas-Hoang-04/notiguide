# Organization Tenancy & Self-Registration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Introduce a top-level Organization tenant and public self-registration (4 paths), fencing every SUPER_ADMIN to its own org while keeping ADMIN device/store management intact.

**Architecture:** A new `organization` table sits above `store` (nullable `org_id` → independent stores). SUPER_ADMIN binds to one org; ADMIN binds to one store. Authorization moves from a static `StoreAccessUtil` (SUPER_ADMIN sees all) to an injectable, org-aware `StoreAccessService` (SUPER_ADMIN sees only `store.org_id == principal.orgId`). Self-registration posts to a public `/api/auth/register`; "create org"/"create store" activate immediately, while "join" paths live as ephemeral **Redis 7-day join requests** that materialize an `admin` row only on owner approval.

**Tech Stack:** Kotlin 2.3 / Spring Boot 3.5 WebFlux (coroutines) · R2DBC (TimescaleDB) · reactive Redis (Lettuce) · Next.js 16 / React 19 / next-intl / shadcn-ui / Tailwind 4 / Biome.

**Source spec:** `docs/spec/Organization Tenancy and Self-Registration Spec.md`

---

## Conventions for this plan (read first)

These override the generic skill template to match this repo's established workflow:

- **No git commits.** Per project rule, the executor decides when to commit — **do not** add `git commit` steps. (Standing user preference.)
- **No build after implementation.** Per CLAUDE.md, building/compiling is handled by a separate audit flow. **Do not run `./gradlew build`/`test`.** Backend verification = a logical review checklist (imports resolve, types/signatures match, no unused imports).
- **Lint before anything, frontend only.** Frontend verification = `cd web && yarn lint` (Biome). Fix all reported issues before moving on.
- **Changelog.** Every phase ends with a `docs/CHANGELOGS.md` entry step (including anything intentionally skipped).
- **Named imports only** at the top of each file (no fully-qualified inline references).
- **R2DBC writes use entity `.save()`** (mirroring `StoreService`). Never write a bare `@Query("UPDATE …")`/`INSERT`/`DELETE` — Context7 confirmed those require `@Modifying`, and we avoid them entirely. Custom `@Query` is **SELECT-only**.
- **UI rules** (from `docs/walkthrough/Web Styles.md` + actual codebase usage; user directive: follow the codebase where the doc is stale):
  - **Warning/alert boxes:** dashboard surfaces use `rounded-xl border border-warning/40 bg-warning/15 text-warning dark:border-warning/50 dark:bg-warning/20`. **Auth pages (login/register) use `rounded-lg`** to match the existing login banners (`mb-5 rounded-lg border border-warning/40 bg-warning/15 p-4 text-sm font-medium text-warning dark:border-warning/50 dark:bg-warning/20`). Always use `text-warning` (never `text-warning-foreground`) and always include the `dark:` overrides.
  - **Hyperlinks:** never bare `hover:underline`. Use `text-primary underline decoration-primary/30 underline-offset-4 transition-colors hover:decoration-primary/70` (add `font-medium text-sm` for nav/breadcrumb links).
  - **shadcn-ui only.** Reuse existing components in `web/src/components/ui/`. Do not introduce other UI libraries.
  - **Bilingual.** Every user-facing string goes through next-intl with mirrored keys in `web/src/messages/en.json` and `vi.json` (same keys, same `<tag>` structure). Per CLAUDE.md, **propose non-trivial Vietnamese phrasings in chat for approval before writing them to `vi.json`** — the VI values in this plan are drafts to confirm, not final.
- **GitNexus impact** must be run before editing `StoreAccessUtil` and `AdminPrincipal` (see Task 3.1).

---

## File map

**Backend — new files**
- `domain/organization/entity/Organization.kt`
- `domain/organization/dto/OrganizationDto.kt`
- `domain/organization/repository/OrganizationRepository.kt`
- `domain/organization/service/OrganizationService.kt`
- `domain/organization/controller/OrganizationController.kt`
- `domain/organization/response/JoinCodeResponse.kt`
- `core/tenant/JoinCodeGenerator.kt`
- `shared/principal/StoreAccessService.kt` (replaces `StoreAccessUtil.kt`)
- `domain/admin/service/JoinRequestService.kt`
- `domain/admin/service/RegistrationService.kt`
- `domain/admin/controller/JoinRequestController.kt`
- `domain/admin/request/RegisterRequest.kt`
- `domain/admin/response/RegisterResponse.kt`
- `domain/admin/dto/JoinRequestDto.kt`
- `core/exception/` → `PendingJoinRequestException` (added to existing `HttpException.kt`)

**Backend — modified files**
- `src/main/resources/db/schema.sql`
- `src/main/resources/db/migration/002_organization_tenancy.sql` (new migration)
- `domain/admin/entity/Admin.kt`, `domain/admin/dto/AdminDto.kt`
- `shared/principal/AdminPrincipal.kt`, `core/jwt/JWTToPrincipal.kt`
- `domain/store/entity/Store.kt`, `domain/store/dto/StoreDto.kt`, `domain/store/repository/StoreRepository.kt`, `domain/store/service/StoreService.kt`, `domain/store/controller/StoreController.kt`
- `domain/admin/service/AdminService.kt`, `domain/admin/controller/AdminController.kt`, `domain/admin/controller/AuthController.kt`
- `domain/device/service/DeviceQueryService.kt`, `domain/device/controller/DeviceAdminController.kt`, `domain/device/service/HubDiagnosticsService.kt`
- `domain/device/service/EnrollmentTokenService.kt`, `domain/device/controller/EnrollmentTokenController.kt`
- `domain/analytics/repository/AnalyticsEventRepository.kt`, `domain/analytics/service/AnalyticsQueryService.kt`, `domain/analytics/controller/AnalyticsController.kt`
- `core/redis/RedisKeyManager.kt`, `core/redis/RedisTTLPolicy.kt`
- All 13 `StoreAccessUtil` call-site files (Task 3.3)

**Frontend — new files**
- `web/src/app/[locale]/(auth)/register/page.tsx`
- `web/src/features/auth/register-path-selector.tsx`
- `web/src/features/auth/register-create-org-form.tsx`
- `web/src/features/auth/register-create-store-form.tsx`
- `web/src/features/auth/register-join-form.tsx`
- `web/src/features/auth/register-pending-screen.tsx`
- `web/src/features/admin/join-requests-panel.tsx`
- `web/src/features/admin/approve-join-request-dialog.tsx`
- `web/src/features/organization/join-code-panel.tsx`
- `web/src/features/organization/api.ts`
- `web/src/app/[locale]/dashboard/settings/organization/page.tsx`
- `web/src/types/organization.ts`

**Frontend — modified files**
- `web/src/types/admin.ts`, `web/src/lib/constants.ts`, `web/src/store/auth.ts`
- `web/src/features/auth/api.ts`
- `web/src/app/[locale]/(auth)/login/page.tsx`
- `web/src/app/[locale]/dashboard/admins/page.tsx`
- `web/src/app/[locale]/dashboard/settings/store/page.tsx`, `web/src/components/layout/sidebar.tsx`
- `web/src/messages/en.json`, `web/src/messages/vi.json`

---

# Phase 1 — Database

### Task 1.1: Update `schema.sql` for fresh installs

**Files:**
- Modify: `backend/src/main/resources/db/schema.sql`

- [ ] **Step 1: Add the `organization` table before the `store` table.** Insert immediately after the `CREATE TYPE admin_role …` line (currently line 16) and before `CREATE TABLE store`. `created_by` has NO inline FK (the `admin` table is defined later — the FK is added at the end to break the circular dependency).

```sql
-- Organizations: top-level tenant. A SUPER_ADMIN owns exactly one org and
-- governs every store inside it. Independent stores have org_id = NULL.
CREATE TABLE organization (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        TEXT NOT NULL,
    join_code   TEXT NOT NULL UNIQUE,
    created_by  UUID,  -- FK to admin(id) added at end of file (circular dep)
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

- [ ] **Step 2: Add `org_id` + `join_code` columns to the `store` table.** Inside the `CREATE TABLE store (…)` body (currently starts line 76), add these two columns:

```sql
    org_id      UUID REFERENCES organization(id) ON DELETE RESTRICT,
    join_code   TEXT,
```

After the `CREATE TABLE store` statement, add:

```sql
CREATE UNIQUE INDEX uq_store_join_code ON store(join_code) WHERE join_code IS NOT NULL;
CREATE INDEX idx_store_org ON store(org_id);
```

- [ ] **Step 3: Add `org_id` to the `admin` table and swap the CHECK constraint.** In the `CREATE TABLE admin (…)` body (currently line 130), add the column:

```sql
    org_id UUID REFERENCES organization(id) ON DELETE RESTRICT,
```

Replace the existing constraint:

```sql
    CONSTRAINT chk_superadmin_no_store CHECK (
        (role = 'ROLE_SUPER_ADMIN' AND store_id IS NULL) OR
        (role = 'ROLE_ADMIN')
    )
```

with:

```sql
    CONSTRAINT chk_admin_tenancy CHECK (
        (role = 'ROLE_SUPER_ADMIN' AND org_id IS NOT NULL AND store_id IS NULL) OR
        (role = 'ROLE_ADMIN' AND org_id IS NULL)
    )
```

After the admin table's indexes (near line 149), add:

```sql
CREATE INDEX idx_admin_org ON admin(org_id);
```

- [ ] **Step 4: Add the deferred `organization.created_by` FK at the very end of the file** (after the `admin` table exists):

```sql
ALTER TABLE organization
    ADD CONSTRAINT fk_organization_created_by
    FOREIGN KEY (created_by) REFERENCES admin(id) ON DELETE SET NULL;
```

- [ ] **Step 5: Verify** — re-read the file top-to-bottom: `organization` is created before `store`; `store.org_id`/`admin.org_id` reference `organization`; the `created_by` FK is the last statement. A fresh DB has no rows, so no backfill is needed here.

---

### Task 1.2: Standalone idempotent migration for existing environments

**Files:**
- Create: `backend/src/main/resources/db/migration/002_organization_tenancy.sql`

> Numbering note: the `db/migration/` folder is currently empty; `002` follows the slug spec's documented `001_*` convention. If `001` is not present in the target environment, this script is still self-contained — just set the prefix to the next free integer.

- [ ] **Step 1: Write the migration.** Order is critical: **backfill runs before the CHECK swap**, because existing SUPER_ADMINs have `org_id = NULL` until backfilled and would otherwise violate the new constraint.

```sql
-- 002_organization_tenancy.sql
-- Run ONCE per existing environment. Idempotent and safe to re-run:
-- the data backfill is guarded so a second run is a no-op and never folds
-- later-created independent stores into the default org.
BEGIN;

-- 1. DDL (idempotent) -------------------------------------------------------
CREATE TABLE IF NOT EXISTS organization (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        TEXT NOT NULL,
    join_code   TEXT NOT NULL UNIQUE,
    created_by  UUID,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE store ADD COLUMN IF NOT EXISTS org_id    UUID REFERENCES organization(id) ON DELETE RESTRICT;
ALTER TABLE store ADD COLUMN IF NOT EXISTS join_code TEXT;
ALTER TABLE admin ADD COLUMN IF NOT EXISTS org_id    UUID REFERENCES organization(id) ON DELETE RESTRICT;

CREATE UNIQUE INDEX IF NOT EXISTS uq_store_join_code ON store(join_code) WHERE join_code IS NOT NULL;
CREATE INDEX IF NOT EXISTS idx_store_org ON store(org_id);
CREATE INDEX IF NOT EXISTS idx_admin_org ON admin(org_id);

DO $$
BEGIN
    IF NOT EXISTS (SELECT 1 FROM pg_constraint WHERE conname = 'fk_organization_created_by') THEN
        ALTER TABLE organization
            ADD CONSTRAINT fk_organization_created_by
            FOREIGN KEY (created_by) REFERENCES admin(id) ON DELETE SET NULL;
    END IF;
END $$;

-- 2. One-time guarded backfill (BEFORE the CHECK swap) ----------------------
DO $$
DECLARE
    default_org_id UUID;
BEGIN
    IF NOT EXISTS (SELECT 1 FROM organization) THEN
        INSERT INTO organization (name, join_code)
        VALUES ('Default Organization', 'o_' || substr(md5(random()::text), 1, 12))
        RETURNING id INTO default_org_id;

        UPDATE store SET org_id = default_org_id WHERE org_id IS NULL;
        UPDATE admin SET org_id = default_org_id
            WHERE role = 'ROLE_SUPER_ADMIN' AND org_id IS NULL;
    END IF;
END $$;

-- 3. Swap the CHECK constraint (AFTER backfill so all rows satisfy it) ------
ALTER TABLE admin DROP CONSTRAINT IF EXISTS chk_superadmin_no_store;
ALTER TABLE admin DROP CONSTRAINT IF EXISTS chk_admin_tenancy;
ALTER TABLE admin ADD CONSTRAINT chk_admin_tenancy CHECK (
    (role = 'ROLE_SUPER_ADMIN' AND org_id IS NOT NULL AND store_id IS NULL) OR
    (role = 'ROLE_ADMIN' AND org_id IS NULL)
);

COMMIT;
```

- [ ] **Step 2: Verify** — confirm the three sections are ordered DDL → backfill → CHECK swap, every DDL statement is `IF NOT EXISTS`/`IF EXISTS`-guarded, and the backfill is wrapped in `IF NOT EXISTS (SELECT 1 FROM organization)`.

- [ ] **Step 3: `docs/CHANGELOGS.md`** — add a "Phase 1 — DB tenancy schema" entry summarizing the `organization` table, `store`/`admin` `org_id` columns, the CHECK swap, and the new migration file.

---

# Phase 2 — Domain entities, DTOs, repositories

### Task 2.1: Organization entity, DTO, repository + join-code generator

**Files:**
- Create: `backend/src/main/kotlin/com/thomas/notiguide/domain/organization/entity/Organization.kt`
- Create: `backend/src/main/kotlin/com/thomas/notiguide/domain/organization/dto/OrganizationDto.kt`
- Create: `backend/src/main/kotlin/com/thomas/notiguide/domain/organization/repository/OrganizationRepository.kt`
- Create: `backend/src/main/kotlin/com/thomas/notiguide/core/tenant/JoinCodeGenerator.kt`

- [ ] **Step 1: `Organization.kt`.** `created_by` is a plain column (set explicitly by the service — there is no security context during public registration, so `@CreatedBy` would not populate). `created_at`/`updated_at` use auditing, which is enabled (`@EnableR2dbcAuditing` in `R2DBCConfig`).

```kotlin
package com.thomas.notiguide.domain.organization.entity

import com.thomas.notiguide.domain.organization.dto.OrganizationDto
import org.springframework.data.annotation.CreatedDate
import org.springframework.data.annotation.Id
import org.springframework.data.annotation.LastModifiedDate
import org.springframework.data.relational.core.mapping.Column
import org.springframework.data.relational.core.mapping.Table
import java.time.OffsetDateTime
import java.util.UUID

@Table("organization")
data class Organization(
    @Id
    @Column("id")
    val id: UUID? = null,

    @Column("name")
    val name: String,

    @Column("join_code")
    val joinCode: String,

    @Column("created_by")
    val createdBy: UUID? = null,

    @CreatedDate
    @Column("created_at")
    val createdAt: OffsetDateTime? = null,

    @LastModifiedDate
    @Column("updated_at")
    val updatedAt: OffsetDateTime? = null
) {
    fun toDto(): OrganizationDto = OrganizationDto(
        id = id!!,
        name = name,
        joinCode = joinCode,
        createdBy = createdBy,
        createdAt = createdAt,
        updatedAt = updatedAt
    )
}
```

- [ ] **Step 2: `OrganizationDto.kt`.**

```kotlin
package com.thomas.notiguide.domain.organization.dto

import java.time.OffsetDateTime
import java.util.UUID

data class OrganizationDto(
    val id: UUID,
    val name: String,
    val joinCode: String,
    val createdBy: UUID?,
    val createdAt: OffsetDateTime?,
    val updatedAt: OffsetDateTime?
)
```

- [ ] **Step 3: `OrganizationRepository.kt`** (SELECT-only custom queries, mirroring `AdminRepository`).

```kotlin
package com.thomas.notiguide.domain.organization.repository

import com.thomas.notiguide.domain.organization.entity.Organization
import org.springframework.data.r2dbc.repository.Query
import org.springframework.data.repository.kotlin.CoroutineCrudRepository
import org.springframework.stereotype.Repository
import java.util.UUID

@Repository
interface OrganizationRepository : CoroutineCrudRepository<Organization, UUID> {
    @Query("SELECT * FROM organization WHERE join_code = :joinCode")
    suspend fun findByJoinCode(joinCode: String): Organization?

    @Query("SELECT EXISTS(SELECT 1 FROM organization WHERE join_code = :joinCode)")
    suspend fun existsByJoinCode(joinCode: String): Boolean
}
```

- [ ] **Step 4: `JoinCodeGenerator.kt`** (opaque, URL-safe, prefixed — mirrors the `SecureRandom` token style in `RefreshTokenService`).

```kotlin
package com.thomas.notiguide.core.tenant

import java.security.SecureRandom
import java.util.Base64

object JoinCodeGenerator {
    const val ORG_PREFIX = "o_"
    const val STORE_PREFIX = "s_"

    private val secureRandom = SecureRandom()
    private val encoder = Base64.getUrlEncoder().withoutPadding()

    /** e.g. "o_X1k9..." — 9 random bytes → 12 url-safe chars after the prefix. */
    fun generate(prefix: String): String {
        val bytes = ByteArray(9)
        secureRandom.nextBytes(bytes)
        return prefix + encoder.encodeToString(bytes)
    }
}
```

- [ ] **Step 5: Verify** — package paths match `domain/organization/...` and `core/tenant/...`; all imports are named; no `@Modifying`/write `@Query`.

---

### Task 2.2: Add `orgId` to Admin, AdminDto, AdminPrincipal, JWTToPrincipal

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/entity/Admin.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/dto/AdminDto.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/shared/principal/AdminPrincipal.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/core/jwt/JWTToPrincipal.kt`

- [ ] **Step 1: `Admin.kt`** — add the `orgId` column (after `storeId`, line 32):

```kotlin
    @Column("org_id")
    val orgId: UUID? = null,
```

Update `toDto(...)` to pass `orgId = orgId`, and update `toPrincipal()` to pass `orgId = orgId`:

```kotlin
    fun toDto(storeName: String? = null): AdminDto = AdminDto(
        id = id!!,
        username = username,
        role = role,
        orgId = orgId,
        storeId = storeId,
        storeName = storeName,
        isVerified = isVerified,
        createdBy = createdBy,
        verifiedBy = verifiedBy,
        verifiedAt = verifiedAt,
        createdAt = createdAt,
        updatedAt = updatedAt
    )

    fun toPrincipal(): AdminPrincipal = AdminPrincipal(
        _id = id!!,
        _username = username,
        _password = passwordHash,
        _authorities = listOf(SimpleGrantedAuthority(role.name)),
        orgId = orgId,
        storeId = storeId,
        isVerified = isVerified
    )
```

- [ ] **Step 2: `AdminDto.kt`** — add `val orgId: UUID?` immediately before `val storeId: UUID?`.

- [ ] **Step 3: `AdminPrincipal.kt`** — add `val orgId: UUID?` to the constructor (before `storeId`):

```kotlin
class AdminPrincipal(
    private val _id: UUID,
    private val _username: String,
    private val _password: String,
    private val _authorities: Collection<GrantedAuthority>,
    val orgId: UUID?,
    val storeId: UUID?,
    val isVerified: Boolean
) : UserDetails {
```

- [ ] **Step 4: `JWTToPrincipal.kt`** — in `convert`, pass `orgId = admin.orgId` in the `AdminPrincipal(...)` construction:

```kotlin
        return AdminPrincipal(
            _id = admin.id!!,
            _username = admin.username,
            _password = admin.passwordHash,
            _authorities = listOf(SimpleGrantedAuthority(admin.role.name)),
            orgId = admin.orgId,
            storeId = admin.storeId,
            isVerified = true
        )
```

- [ ] **Step 5: Verify** — every `AdminPrincipal(...)` constructor call now passes `orgId`. Grep: `grep -rn "AdminPrincipal(" backend/src/main/kotlin` and confirm only `Admin.toPrincipal` and `JWTToPrincipal.convert` construct it (both updated).

---

### Task 2.3: Add `orgId`/`joinCode` to Store + new SELECT queries

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/entity/Store.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/dto/StoreDto.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/repository/StoreRepository.kt`

- [ ] **Step 1: `Store.kt`** — add two columns after `publicId` (line 19):

```kotlin
    @Column("org_id")
    val orgId: UUID? = null,

    @Column("join_code")
    val joinCode: String? = null,
```

Update `toDto()` to include `orgId = orgId` (do **not** expose `joinCode` in `StoreDto` — it is served only via the dedicated join-code endpoints).

- [ ] **Step 2: `StoreDto.kt`** — add `val orgId: UUID?` (place after `publicId`). Do not add `joinCode`.

- [ ] **Step 3: `StoreRepository.kt`** — add SELECT-only queries:

```kotlin
    @Query("SELECT org_id FROM store WHERE id = :storeId")
    suspend fun findOrgIdByStoreId(storeId: UUID): UUID?

    @Query("SELECT * FROM store WHERE org_id = :orgId ORDER BY created_at DESC LIMIT :size OFFSET :offset")
    fun findByOrgIdPaged(orgId: UUID, size: Long, offset: Long): Flow<Store>

    @Query("SELECT COUNT(*) FROM store WHERE org_id = :orgId")
    suspend fun countByOrgId(orgId: UUID): Long

    @Query("SELECT * FROM store WHERE org_id = :orgId AND is_active = true ORDER BY created_at DESC")
    fun findActiveByOrgId(orgId: UUID): Flow<Store>

    @Query("SELECT id FROM store WHERE org_id = :orgId")
    fun findIdsByOrgId(orgId: UUID): Flow<UUID>

    @Query("SELECT * FROM store WHERE join_code = :joinCode")
    suspend fun findByJoinCode(joinCode: String): Store?
```

- [ ] **Step 4: Verify** — `findOrgIdByStoreId` returns `UUID?` (null for missing rows OR independent stores — both correctly deny SUPER_ADMIN access). No write queries added.

- [ ] **Step 5: `docs/CHANGELOGS.md`** — add a "Phase 2 — domain model" entry (Organization domain, `orgId` on Admin/Store + principal, join-code generator, new store queries).

---

# Phase 3 — Authorization core

### Task 3.1: Impact analysis (mandatory before editing)

**Files:** none (analysis only)

- [ ] **Step 1:** Run impact analysis and report the blast radius to the reviewer before any edit:

```
gitnexus_impact({target: "StoreAccessUtil", direction: "upstream"})
gitnexus_impact({target: "AdminPrincipal", direction: "upstream"})
```

- [ ] **Step 2:** Confirm the call-site list matches the 13 files enumerated in Task 3.3. If GitNexus reports additional callers, add them to Task 3.3's checklist. If risk is HIGH/CRITICAL, surface it to the user before proceeding.

  Audit baseline from this plan review: `StoreAccessUtil` impact is **MEDIUM** (13 impacted symbols, 10 direct importers); `AdminPrincipal` impact is **HIGH** (27 impacted symbols, 17 direct importers, including `EnrollmentTokenService.kt` / `EnrollmentTokenController.kt` because they import/use the principal). Treat `AdminPrincipal` constructor changes as high-risk and re-run the analysis in the implementation branch before editing.

---

### Task 3.2: Replace `StoreAccessUtil` with org-aware `StoreAccessService`

**Files:**
- Create: `backend/src/main/kotlin/com/thomas/notiguide/shared/principal/StoreAccessService.kt`
- Delete: `backend/src/main/kotlin/com/thomas/notiguide/shared/principal/StoreAccessUtil.kt`

- [ ] **Step 1: Create `StoreAccessService.kt`.** ADMIN keeps the cheap in-memory check (the queue hot path — no DB lookup). SUPER_ADMIN does one indexed `org_id` lookup. (No cache in v1 — YAGNI; ADMIN path is unaffected and SUPER_ADMIN checks are infrequent. Caching is a noted future optimization.)

```kotlin
package com.thomas.notiguide.shared.principal

import com.thomas.notiguide.core.exception.ForbiddenException
import com.thomas.notiguide.domain.admin.types.AdminRole
import com.thomas.notiguide.domain.store.repository.StoreRepository
import org.springframework.stereotype.Service
import java.util.UUID

@Service
class StoreAccessService(
    private val storeRepository: StoreRepository
) {
    /**
     * SUPER_ADMIN may act on a store only when it belongs to their org.
     * ADMIN may act only on their single assigned store.
     */
    suspend fun requireStoreAccess(principal: AdminPrincipal, storeId: UUID) {
        if (isSuperAdmin(principal)) {
            val orgId = principal.orgId
                ?: throw ForbiddenException("You do not have access to this store")
            val storeOrgId = storeRepository.findOrgIdByStoreId(storeId)
            if (storeOrgId == null || storeOrgId != orgId)
                throw ForbiddenException("You do not have access to this store")
            return
        }
        if (principal.storeId != storeId)
            throw ForbiddenException("You do not have access to this store")
    }

    private fun isSuperAdmin(principal: AdminPrincipal): Boolean =
        principal.authorities.any { it.authority == AdminRole.ROLE_SUPER_ADMIN.name }
}
```

- [ ] **Step 2: Delete `StoreAccessUtil.kt`.**

- [ ] **Step 3: Verify** — `requireStoreAccess` is now `suspend` (all current callers already run in suspend functions, confirmed in Task 3.3).
  Do not keep any old SUPER_ADMIN short-circuit around this call. Existing helper methods such as `DeviceLifecycleService.loadAccessibleDevice`, `RfCodeService.loadAccessibleDevice`, `DeviceApprovalService.requireDeviceAccess`, and `DeviceQueryService.renameDevice` currently return early or skip checks for SUPER_ADMIN; those branches must be removed or rewritten so a device with a `storeId` always flows through `storeAccess.requireStoreAccess(principal, storeId)`.

- [ ] **Step 3: Verify** — `requireStoreAccess` is now `suspend` (all current direct call sites run in suspend functions, confirmed in Task 3.3), and every former SUPER_ADMIN bypass around store-owned devices now calls the service.

---

### Task 3.3: Inject `StoreAccessService` at all 13 call sites

The change is **identical** in every file: (a) replace the import `com.thomas.notiguide.shared.principal.StoreAccessUtil` with `com.thomas.notiguide.shared.principal.StoreAccessService`; (b) add a constructor parameter `private val storeAccess: StoreAccessService`; (c) replace each `StoreAccessUtil.requireStoreAccess(principal, X)` with `storeAccess.requireStoreAccess(principal, X)`. Every call already sits in a `suspend fun`.

**Representative example — `StoreController.kt`:**

Constructor:
```kotlin
class StoreController(
    private val storeService: StoreService,
    private val storeAccess: StoreAccessService
) {
```
Call site (×4 in this file):
```kotlin
        storeAccess.requireStoreAccess(principal, id)
```

- [ ] **Step 1: Controllers (7 files)** — apply the change to:
  - `domain/store/controller/StoreController.kt` (4 calls)
  - `domain/store/controller/ServiceTypeController.kt` (4 calls)
  - `domain/store/controller/StoreSlugController.kt` (4 calls)
  - `domain/queue/controller/QueueAdminController.kt` (15 calls)
  - `domain/analytics/controller/AnalyticsController.kt` (6 calls)
  - `domain/admin/controller/AdminController.kt` (1 call, line ~195)
  - `domain/device/controller/DeviceAdminController.kt` (3 calls)

- [ ] **Step 2: Services (6 files)** — apply the same change to:
  - `domain/device/service/RfCodeService.kt` (1 call)
  - `domain/device/service/UsbDispatchPayloadService.kt` (1 call)
  - `domain/device/service/DeviceApprovalService.kt` (2 calls)
  - `domain/device/service/DeviceLifecycleService.kt` (1 call)
  - `domain/device/service/PassiveDeviceRegistrationService.kt` (1 call)
  - `domain/device/service/DeviceQueryService.kt` (1 call, in `renameDevice`)

- [ ] **Step 2b: Remove SUPER_ADMIN bypasses in device service helpers.** These are not visible as plain `StoreAccessUtil` replacements unless the surrounding control flow is audited:
  - `DeviceLifecycleService.loadAccessibleDevice`: delete the `if (isSuperAdmin(principal)) return device` early return; if `device.storeId != null`, call `storeAccess.requireStoreAccess(principal, storeId)`. Keep the existing "store-scoped admins need an assigned store" error for ADMIN when `storeId == null`; a SUPER_ADMIN may only access a storeless/pending device when no store fence exists yet.
  - `RfCodeService.loadAccessibleDevice`: apply the same change as `DeviceLifecycleService`; SUPER_ADMIN must not bypass the store fence for an RF-code device already assigned to a store.
  - `DeviceApprovalService.requireDeviceAccess`: remove the `if (isSuperAdmin(principal)) return` branch so rejecting or moving an already store-assigned pending device is fenced to the caller's org. **Also change its signature from `private fun requireDeviceAccess(...)` to `private suspend fun requireDeviceAccess(...)`** — it now calls the `suspend` `storeAccess.requireStoreAccess(...)`. Its callers `approve`/`reject` are already `suspend`, so no further changes are needed. (The other three helpers — `DeviceLifecycleService.loadAccessibleDevice`, `RfCodeService.loadAccessibleDevice`, `DeviceQueryService.renameDevice` — are already `suspend`; if any helper you touch is not, add `suspend`.)
  - `DeviceQueryService.renameDevice`: remove the `if (!isSuperAdmin(principal))` wrapper; if the device has a `storeId`, call `storeAccess.requireStoreAccess(principal, storeId)` for both roles. Keep the existing ADMIN error when the device has no store.

- [ ] **Step 3: Verify** — `grep -rn "StoreAccessUtil" backend/src/main/kotlin` returns **zero** results. `grep -rn "storeAccess.requireStoreAccess" backend/src/main/kotlin` returns the controller/service replacements plus the four audited device-helper calls above.

- [ ] **Step 4: `docs/CHANGELOGS.md`** — add a "Phase 3 — org-aware authorization" entry (StoreAccessUtil → StoreAccessService, SUPER_ADMIN now fenced to its org, 13 call sites updated).

---

# Phase 4 — Org-scoped listings, store creation & admin management

### Task 4.1: Store creation + listing scope to org

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/service/StoreService.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/controller/StoreController.kt`

- [ ] **Step 1: `StoreService.createStore` — add an `orgId` param and generate a join code only for independent stores.** Inject `OrganizationRepository`? No — only `StoreRepository` is needed for uniqueness. Add the import `com.thomas.notiguide.core.tenant.JoinCodeGenerator`. New signature + body:

```kotlin
    @Transactional
    suspend fun createStore(request: CreateStoreRequest, orgId: UUID?): StoreDto {
        require(request.name.isNotBlank()) { "Store name must not be blank" }
        validateNoShowAction(request.noShowAction)

        val joinCode = if (orgId == null) generateUniqueStoreJoinCode() else null

        val store = Store(
            name = request.name,
            address = request.address?.takeIf { it.isNotBlank() },
            orgId = orgId,
            joinCode = joinCode,
            allowJumpCall = request.allowJumpCall,
            allowNoShow = request.allowNoShow
        )
        val saved = storeRepository.save(store)
        // ... (rest unchanged: storeSettingsRepository.save, serviceTypeRepository.save, return)
```

Add a private helper to `StoreService`:

```kotlin
    private suspend fun generateUniqueStoreJoinCode(): String {
        repeat(5) {
            val code = JoinCodeGenerator.generate(JoinCodeGenerator.STORE_PREFIX)
            if (storeRepository.findByJoinCode(code) == null) return code
        }
        throw IllegalStateException("Could not generate a unique store join code")
    }
```

- [ ] **Step 2: `StoreService.listStores` — accept an `orgId` and scope to it.** Change the signature to `listStores(orgId: UUID, page: Int, size: Int)` and use `storeRepository.countByOrgId(orgId)` + `storeRepository.findByOrgIdPaged(orgId, size.toLong(), offset)` instead of `count()`/`findAllPaged(...)`.

- [ ] **Step 3: `StoreController.createStore`** — pass the org:

```kotlin
        requireSuperAdmin(principal)
        val orgId = principal.orgId
            ?: throw ForbiddenException("Organization owner has no organization assigned")
        val dto = storeService.createStore(request, orgId)
```

- [ ] **Step 4: `StoreController.listStores`** — pass the org:

```kotlin
        requireSuperAdmin(principal)
        val orgId = principal.orgId
            ?: throw ForbiddenException("Organization owner has no organization assigned")
        val response = storeService.listStores(orgId, page, size)
```

- [ ] **Step 5: Org-fence `deleteStore`.** `StoreController.deleteStore` currently calls only `requireSuperAdmin(principal)`, which would let a SUPER_ADMIN delete another org's store. Add the org check right after it:

```kotlin
        requireSuperAdmin(principal)
        storeAccess.requireStoreAccess(principal, id)
        storeService.deleteStore(id)
```
(`getStore`/`updateStore`/settings already route through `requireStoreAccess` via Task 3.3.)

- [ ] **Step 6: Verify** — independent stores get a join code; org-owned stores do not; listing returns only the caller's org stores; cross-org delete now returns 403.

---

### Task 4.2: Admin listing scoped to org

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/repository/AdminRepository.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/service/AdminService.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/controller/AdminController.kt`

- [ ] **Step 1: `AdminRepository.kt`** — add org-scoped queries. "Admins in an org" = the org's SUPER_ADMINs (`a.org_id = :orgId`) plus the admins of the org's stores (`s.org_id = :orgId`):

```kotlin
    @Query(
        """
        SELECT a.* FROM admin a
        LEFT JOIN store s ON a.store_id = s.id
        WHERE a.org_id = :orgId OR s.org_id = :orgId
        ORDER BY a.created_at DESC LIMIT :limit OFFSET :offset
        """
    )
    fun findByOrgPaged(orgId: UUID, limit: Long, offset: Long): Flow<Admin>

    @Query(
        """
        SELECT COUNT(*) FROM admin a
        LEFT JOIN store s ON a.store_id = s.id
        WHERE a.org_id = :orgId OR s.org_id = :orgId
        """
    )
    suspend fun countByOrg(orgId: UUID): Long
```

- [ ] **Step 2: `AdminService.kt`** — add `listAdminsByOrg(orgId, page, size)` mirroring `listAllAdmins` but using `countByOrg`/`findByOrgPaged`, and resolving store names via the existing `resolveStoreNames(...)` helper:

```kotlin
    @Transactional(readOnly = true)
    suspend fun listAdminsByOrg(orgId: UUID, page: Int, size: Int): AdminPageResponse {
        require(page >= 0) { "Page must be greater than or equal to 0" }
        require(size in 1..100) { "Size must be between 1 and 100" }

        val totalItems = adminRepository.countByOrg(orgId)
        val totalPages = if (totalItems == 0L) 0 else ((totalItems + size - 1) / size).toInt()
        val offset = page.toLong() * size

        val items = if (offset >= totalItems) {
            emptyList()
        } else {
            val admins = adminRepository.findByOrgPaged(orgId, size.toLong(), offset).toList()
            val storeNames = resolveStoreNames(admins)
            admins.map { it.toDto(it.storeId?.let { id -> storeNames[id] }) }
        }
        return AdminPageResponse(items = items, page = page, size = size, totalItems = totalItems, totalPages = totalPages)
    }
```

- [ ] **Step 3: `AdminController.listAdmins`** — when `storeId` is null, scope to the SUPER_ADMIN's org instead of all admins:

```kotlin
        val admins = if (storeId == null) {
            requireSuperAdmin(principal)
            val orgId = principal.orgId
                ?: throw ForbiddenException("Organization owner has no organization assigned")
            adminService.listAdminsByOrg(orgId, page, size)
        } else {
            storeAccess.requireStoreAccess(principal, storeId)
            adminService.listAdminsByStore(storeId, page, size, role)
        }
```
(Keep the existing `role` filtering for the store branch; the org branch ignores `role` for simplicity in v1.)

- [ ] **Step 4: Verify** — a SUPER_ADMIN listing admins sees only their org's admins (their stores' admins + co-owner SUPER_ADMINs of the same org).

---

### Task 4.3: Device listing & hub-health scoped to org

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/service/DeviceQueryService.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/service/HubDiagnosticsService.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/controller/DeviceAdminController.kt`

- [ ] **Step 1: `DeviceQueryService.listDevices`** — add an `orgId: UUID?` parameter and one WHERE clause (the query already `LEFT JOIN store s`):

```kotlin
    suspend fun listDevices(
        kind: DeviceKind?,
        storeId: UUID?,
        orgId: UUID?
    ): DeviceListResponse {
```
In the SQL, add after the existing `AND (:storeId IS NULL OR d.store_id = :storeId)` line:
```
              AND (:orgId IS NULL OR s.org_id = :orgId)
```
And add the binding alongside the others:
```kotlin
            .bindNullable("orgId", orgId, UUID::class.java)
```

- [ ] **Step 1b: Fix the OTHER `listDevices` caller.** `DeviceDispatchService.kt:93` calls `deviceQueryService.listDevices(kind = null, storeId = storeId)` — the new arity breaks it. Update that call to pass `orgId = null` (internal dispatch is already store-scoped):

```kotlin
        val devices = deviceQueryService.listDevices(kind = null, storeId = storeId, orgId = null).devices
```

- [ ] **Step 2: `HubDiagnosticsService.getHubHealthSummary`** — add an `orgId: UUID?` parameter and an org filter (current code conditionally appends `AND store_id = :storeId`):

```kotlin
    suspend fun getHubHealthSummary(storeId: UUID?, orgId: UUID?): HubHealthSummaryResponse {
```
In the SQL builder, when `orgId != null` (and `storeId == null`) append:
```
              AND store_id IN (SELECT id FROM store WHERE org_id = :orgId)
```
and bind `:orgId` in that branch (mirror the existing `.let { if (storeId != null) it.bind("storeId", storeId) else it }` pattern with an analogous `orgId` branch).

- [ ] **Step 3: `DeviceAdminController`** — update both list call sites to pass the org. For `listDevices`:

```kotlin
        val effectiveStoreId = when {
            storeId != null -> {
                storeAccess.requireStoreAccess(principal, storeId)
                storeId
            }
            isSuperAdmin(principal) -> null
            else -> principal.storeId
                ?: throw ForbiddenException("Store-scoped admins need an assigned store to view devices")
        }
        val orgScope = if (storeId == null && isSuperAdmin(principal)) principal.orgId else null
        return ResponseEntity.ok(deviceQueryService.listDevices(kind, effectiveStoreId, orgScope))
```
For `getHubHealth`:
```kotlin
        val orgScope = if (isSuperAdmin(principal)) principal.orgId else null
        val effectiveStoreId = if (isSuperAdmin(principal)) null
            else principal.storeId
                ?: throw ForbiddenException("Store-scoped admins need an assigned store to view hub health")
        return ResponseEntity.ok(hubDiagnosticsService.getHubHealthSummary(effectiveStoreId, orgScope))
```

- [ ] **Step 4: Verify** — a SUPER_ADMIN sees devices/hub-health only for stores in their org; an independent-store ADMIN still sees only their store's devices (unchanged path).

---

### Task 4.4: Analytics overview scoped to org

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/analytics/repository/AnalyticsEventRepository.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/analytics/service/AnalyticsQueryService.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/analytics/controller/AnalyticsController.kt`

- [ ] **Step 1: `AnalyticsEventRepository.getOverview`** — add an `orgId: UUID?` parameter; the query already `JOIN store s ON ae.store_id = s.id`, so add the clause `AND (:orgId IS NULL OR s.org_id = :orgId)` to the WHERE, and bind it (`bindNullable("orgId", orgId, UUID::class.java)`).

- [ ] **Step 2: `AnalyticsEventRepository.getOverviewDailyThroughput`** — add an `orgId: UUID?` parameter; this query has no store join, so add the clause `AND (:orgId IS NULL OR store_id IN (SELECT id FROM store WHERE org_id = :orgId))` to the WHERE, and bind it.

- [ ] **Step 3: `AnalyticsQueryService`** — thread `orgId` through the three overview methods:
  - `getOverviewRealtime(orgId: UUID?)`: replace `storeRepository.findAllActive()` with `if (orgId != null) storeRepository.findActiveByOrgId(orgId) else storeRepository.findAllActive()`.
  - `getOverview(orgId: UUID?, period, customFrom, customTo)`: pass `orgId` to `analyticsEventRepository.getOverview(from, to, orgId)`.
  - `getOverviewThroughput(orgId: UUID?, range, customFrom, customTo)`: pass `orgId` to `analyticsEventRepository.getOverviewDailyThroughput(from, to, orgId)`.

- [ ] **Step 4: `AnalyticsController`** — pass `principal.orgId` in all three overview endpoints (e.g. `analyticsQueryService.getOverviewRealtime(principal.orgId)`). Keep `requireSuperAdmin(principal)`. The store-specific endpoints already route through `storeAccess.requireStoreAccess` (Task 3.3) and need no change.

- [ ] **Step 5: Verify** — overview aggregates only the caller-org's stores. Independent stores never appear in any SUPER_ADMIN overview.

---

### Task 4.5: Org-scope existing admin management (verify / assign-store / create / delete)

These existing SUPER_ADMIN actions must be fenced to the caller's org so a SUPER_ADMIN cannot touch another org's admins/stores.
The existing verify endpoint also widens per spec: a verified independent-store ADMIN may verify another ADMIN in that same independent store.

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/repository/AdminRepository.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/service/AdminService.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/controller/AdminController.kt`

- [ ] **Step 1: Add repository helpers.**

```kotlin
    @Query("SELECT COUNT(*) FROM admin WHERE org_id = :orgId AND role = CAST(:role AS admin_role)")
    suspend fun countByOrgIdAndRole(orgId: UUID, role: AdminRole): Long
```

- [ ] **Step 2: Add private membership guards** to `AdminService` (an admin belongs to org X if its `org_id == X` or its store's `org_id == X`; an independent-store admin belongs to store Y if `store_id == Y` and the store has `org_id IS NULL`):

```kotlin
    private suspend fun requireSameOrg(target: Admin, orgId: UUID) {
        val belongs = target.orgId == orgId ||
            (target.storeId?.let { storeRepository.findById(it)?.orgId } == orgId)
        if (!belongs) throw ForbiddenException("Admin is not in your organization")
    }

    private suspend fun requireSameIndependentStore(target: Admin, storeId: UUID) {
        if (target.storeId != storeId) throw ForbiddenException("Admin is not in your store")
        val store = storeRepository.findById(storeId)
            ?: throw NotFoundException("Store", "id", storeId.toString())
        if (store.orgId != null)
            throw ForbiddenException("Store admins cannot verify organization-owned store members")
    }
```

- [ ] **Step 3: Gate `verifyAdmin`, `updateAdminStore`, `deleteAdmin`** with explicit actor scope. `updateAdminStore` and `deleteAdmin` stay SUPER_ADMIN-only and take `actorOrgId: UUID`; call `requireSameOrg(target, actorOrgId)` after loading the target. For `updateAdminStore`, also verify the *new* store belongs to the actor's org:

```kotlin
        if (storeId != null) {
            val store = storeRepository.findById(storeId)
                ?: throw NotFoundException("Store", "id", storeId.toString())
            if (store.orgId != actorOrgId)
                throw ForbiddenException("Store is not in your organization")
            // ... existing storeName resolution
        }
```

For `verifyAdmin`, add a scope enum or two explicit methods so both allowed approvers are represented:

```kotlin
    sealed interface VerifyScope {
        data class Org(val orgId: UUID) : VerifyScope
        data class IndependentStore(val storeId: UUID) : VerifyScope
    }
```

`verifyAdmin(adminId, verifierId, scope)` loads the target and:
  - for `VerifyScope.Org`, calls `requireSameOrg(target, orgId)`;
  - for `VerifyScope.IndependentStore`, calls `requireSameIndependentStore(target, storeId)` and rejects targets whose role is `ROLE_SUPER_ADMIN`.

- [ ] **Step 4: Gate `createAdmin`** — a SUPER_ADMIN may only create an admin attached to a store in their own org (or an unassigned ADMIN). Add `actorOrgId: UUID`; if `request.storeId != null`, require `storeRepository.findById(request.storeId)?.orgId == actorOrgId`. The new admin's `orgId` stays null (it's an ADMIN). Reject `role == ROLE_SUPER_ADMIN` here (super-admins are created only via self-registration `CREATE_ORG`). This rejection **replaces** the existing partial guard at `AdminService.kt:64` (`if (request.role == ROLE_SUPER_ADMIN && request.storeId != null) throw …`) — delete that old line so there is no dead/contradictory check, and place the new guard before username/store resolution:

```kotlin
        if (request.role == AdminRole.ROLE_SUPER_ADMIN)
            throw HttpException(HttpStatus.BAD_REQUEST, "Organization owners are created via registration, not here")
```

- [ ] **Step 5: Fix the last-SUPER_ADMIN guard to be per org.** In `deleteAdmin`, after `requireSameOrg`, if the target is `ROLE_SUPER_ADMIN`, use `adminRepository.countByOrgIdAndRole(actorOrgId, AdminRole.ROLE_SUPER_ADMIN)`. Do **not** keep the old global `countByRole(ROLE_SUPER_ADMIN)` guard; it would allow deleting the last owner in one org whenever another org still has an owner.

- [ ] **Step 6: `AdminController`** — `createAdmin`, `updateAdminStore`, and `deleteAdmin` remain SUPER_ADMIN-only. After `requireSuperAdmin(principal)`, resolve the org and pass it (new last param `actorOrgId`). Add a private helper and use it in those handlers:

```kotlin
    private fun requireOrgId(principal: AdminPrincipal): UUID =
        principal.orgId ?: throw ForbiddenException("Organization owner has no organization assigned")
```
```kotlin
        // createAdmin
        requireSuperAdmin(principal)
        val dto = adminService.createAdmin(request, principal.id, requireOrgId(principal))
```
```kotlin
        // updateAdminStore
        requireSuperAdmin(principal)
        val dto = adminService.updateAdminStore(id, request.storeId, requireOrgId(principal))
```
```kotlin
        // deleteAdmin
        requireSuperAdmin(principal)
        adminService.deleteAdmin(id, principal.id, requireOrgId(principal))
```
Update the matching `AdminService` method signatures to take the trailing `actorOrgId: UUID` (Steps 1–3).

- [ ] **Step 7: `AdminController.verifyAdmin` widened authority.** Replace its unconditional `requireSuperAdmin(principal)` with:
  - if SUPER_ADMIN: `adminService.verifyAdmin(id, principal.id, VerifyScope.Org(requireOrgId(principal)))`;
  - else: require `principal.isVerified`, require `principal.storeId != null`, then call `adminService.verifyAdmin(id, principal.id, VerifyScope.IndependentStore(principal.storeId))`.

- [ ] **Step 8: Verify** — cross-org verify/assign/delete now returns 403; deleting the last owner in the caller's org returns 409 even if other orgs have owners; a verified independent-store ADMIN can verify a co-owner in the same independent store and cannot verify org-owned store members.

- [ ] **Step 9: `docs/CHANGELOGS.md`** — add a "Phase 4 — org-scoped reads/writes" entry (store/admin/device/analytics listings + admin management fenced to org).

---

### Task 4.6: Enrollment-token scope follows organization tenancy

Enrollment tokens are device-management surface area. Today SUPER_ADMIN list/revoke/issue paths are global or may create storeless tokens; after org tenancy they must be store-scoped to the caller's org.

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/service/EnrollmentTokenService.kt`

- [ ] **Step 1: Inject `StoreAccessService`.** Add `private val storeAccess: StoreAccessService` and replace local SUPER_ADMIN/global checks with the same org-aware store guard used by other device surfaces.

- [ ] **Step 2: Issue requires a concrete accessible store.** In `resolveStoreIdForIssue`, require `requestedStoreId` for SUPER_ADMIN and call `storeAccess.requireStoreAccess(principal, requestedStoreId)`. Keep ADMIN behavior unchanged (`requestedStoreId` omitted or equal to `principal.storeId`). Do not allow new storeless enrollment tokens in v1.

- [ ] **Step 3: List filters by accessible stores.** For ADMIN, continue returning only `principal.storeId`. For SUPER_ADMIN, resolve `principal.orgId`, load org store ids via `storeRepository.findIdsByOrgId(orgId).toList()`, and return only token records whose `storeId` is in that set. Ignore legacy/storeless token records for SUPER_ADMIN rather than exposing them globally.

- [ ] **Step 4: Revoke fences by the token's store.** If the token record has `storeId == null`, allow no SUPER_ADMIN shortcut; reject with `ForbiddenException` unless a deliberate legacy-cleanup endpoint is added later. Otherwise call `storeAccess.requireStoreAccess(principal, record.storeId)` before deleting.

- [ ] **Step 5: Verify** — a SUPER_ADMIN can issue/list/revoke enrollment tokens only for stores in their org; independent-store ADMIN behavior stays store-local; no new storeless enrollment token can be issued.

---

# Phase 5 — Organization service/controller & join codes

### Task 5.1: OrganizationService + OrganizationController (view/rotate org join code)

**Files:**
- Create: `backend/src/main/kotlin/com/thomas/notiguide/domain/organization/service/OrganizationService.kt`
- Create: `backend/src/main/kotlin/com/thomas/notiguide/domain/organization/controller/OrganizationController.kt`
- Create: `backend/src/main/kotlin/com/thomas/notiguide/domain/organization/response/JoinCodeResponse.kt`

- [ ] **Step 1: `JoinCodeResponse.kt`.**

```kotlin
package com.thomas.notiguide.domain.organization.response

data class JoinCodeResponse(val joinCode: String)
```

- [ ] **Step 2: `OrganizationService.kt`.**

```kotlin
package com.thomas.notiguide.domain.organization.service

import com.thomas.notiguide.core.exception.NotFoundException
import com.thomas.notiguide.core.tenant.JoinCodeGenerator
import com.thomas.notiguide.domain.organization.dto.OrganizationDto
import com.thomas.notiguide.domain.organization.repository.OrganizationRepository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.util.UUID

@Service
class OrganizationService(
    private val organizationRepository: OrganizationRepository
) {
    @Transactional(readOnly = true)
    suspend fun getOrganization(id: UUID): OrganizationDto =
        organizationRepository.findById(id)?.toDto()
            ?: throw NotFoundException("Organization", "id", id.toString())

    @Transactional
    suspend fun rotateJoinCode(id: UUID): OrganizationDto {
        val org = organizationRepository.findById(id)
            ?: throw NotFoundException("Organization", "id", id.toString())
        val newCode = generateUniqueJoinCode()
        return organizationRepository.save(org.copy(joinCode = newCode)).toDto()
    }

    private suspend fun generateUniqueJoinCode(): String {
        repeat(5) {
            val code = JoinCodeGenerator.generate(JoinCodeGenerator.ORG_PREFIX)
            if (!organizationRepository.existsByJoinCode(code)) return code
        }
        throw IllegalStateException("Could not generate a unique organization join code")
    }
}
```

- [ ] **Step 3: `OrganizationController.kt`** — only the org's own SUPER_ADMIN may view/rotate. Routes live under `/api/orgs`; resolve the org from `principal.orgId` (never trust a path id for the owner's own org).

```kotlin
package com.thomas.notiguide.domain.organization.controller

import com.thomas.notiguide.core.exception.ForbiddenException
import com.thomas.notiguide.domain.admin.types.AdminRole
import com.thomas.notiguide.domain.organization.dto.OrganizationDto
import com.thomas.notiguide.domain.organization.response.JoinCodeResponse
import com.thomas.notiguide.domain.organization.service.OrganizationService
import com.thomas.notiguide.shared.principal.AdminPrincipal
import org.springframework.http.ResponseEntity
import org.springframework.security.core.annotation.AuthenticationPrincipal
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.PostMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController

@RestController
@RequestMapping("/api/orgs")
class OrganizationController(
    private val organizationService: OrganizationService
) {
    @GetMapping("/me")
    suspend fun getMyOrg(@AuthenticationPrincipal principal: AdminPrincipal): ResponseEntity<OrganizationDto> {
        val orgId = requireOrgOwner(principal)
        return ResponseEntity.ok(organizationService.getOrganization(orgId))
    }

    @GetMapping("/me/join-code")
    suspend fun getJoinCode(@AuthenticationPrincipal principal: AdminPrincipal): ResponseEntity<JoinCodeResponse> {
        val orgId = requireOrgOwner(principal)
        return ResponseEntity.ok(JoinCodeResponse(organizationService.getOrganization(orgId).joinCode))
    }

    @PostMapping("/me/join-code/rotate")
    suspend fun rotateJoinCode(@AuthenticationPrincipal principal: AdminPrincipal): ResponseEntity<JoinCodeResponse> {
        val orgId = requireOrgOwner(principal)
        return ResponseEntity.ok(JoinCodeResponse(organizationService.rotateJoinCode(orgId).joinCode))
    }

    private fun requireOrgOwner(principal: AdminPrincipal) =
        principal.takeIf { it.authorities.any { a -> a.authority == AdminRole.ROLE_SUPER_ADMIN.name } }
            ?.orgId
            ?: throw ForbiddenException("Only organization owners can manage organization join codes")
}
```

- [ ] **Step 4: Verify** — only SUPER_ADMIN with an `orgId` reaches these; rotation invalidates the old code for new joins (existing in-flight Redis requests reference the resolved target id, not the code).

---

### Task 5.2: Independent-store join-code endpoints

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/service/StoreService.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/controller/StoreController.kt`

- [ ] **Step 1: `StoreService.rotateStoreJoinCode`** — only valid for independent stores:

```kotlin
    @Transactional
    suspend fun rotateStoreJoinCode(storeId: UUID): String {
        val store = storeRepository.findById(storeId)
            ?: throw NotFoundException("Store", "id", storeId.toString())
        if (store.orgId != null)
            throw ConflictException("Org-owned stores are joined via the organization code")
        val code = generateUniqueStoreJoinCode()
        storeRepository.save(store.copy(joinCode = code))
        return code
    }

    @Transactional
    suspend fun getStoreJoinCode(storeId: UUID): String {
        val store = storeRepository.findById(storeId)
            ?: throw NotFoundException("Store", "id", storeId.toString())
        if (store.orgId != null)
            throw ConflictException("Org-owned stores are joined via the organization code")
        return store.joinCode
            ?: generateUniqueStoreJoinCode().also { storeRepository.save(store.copy(joinCode = it)) }
    }
```
(Add the `ConflictException` import.)

- [ ] **Step 2: `StoreController`** — add two routes, both gated by `storeAccess.requireStoreAccess(principal, id)` (an independent store's ADMIN passes because `principal.storeId == id`):

```kotlin
    @GetMapping("/{id}/join-code")
    suspend fun getStoreJoinCode(
        @PathVariable id: UUID,
        @AuthenticationPrincipal principal: AdminPrincipal
    ): ResponseEntity<JoinCodeResponse> {
        storeAccess.requireStoreAccess(principal, id)
        return ResponseEntity.ok(JoinCodeResponse(storeService.getStoreJoinCode(id)))
    }

    @PostMapping("/{id}/join-code/rotate")
    suspend fun rotateStoreJoinCode(
        @PathVariable id: UUID,
        @AuthenticationPrincipal principal: AdminPrincipal
    ): ResponseEntity<JoinCodeResponse> {
        storeAccess.requireStoreAccess(principal, id)
        return ResponseEntity.ok(JoinCodeResponse(storeService.rotateStoreJoinCode(id)))
    }
```
(Import `JoinCodeResponse` from `domain.organization.response`.)

- [ ] **Step 3: Verify** — only an independent store's own ADMIN can view/rotate. A SUPER_ADMIN can pass `storeAccess` only for org-owned stores, and org-owned stores reject with 409; independent stores have `org_id = NULL`, so they are intentionally invisible to every SUPER_ADMIN. `getStoreJoinCode` is a write transaction because it may backfill a missing legacy join code.

- [ ] **Step 4: `docs/CHANGELOGS.md`** — add a "Phase 5 — join-code management" entry.

---

# Phase 6 — Join requests (Redis), registration, login

### Task 6.1: Redis keys/TTL + JoinRequestService

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/core/redis/RedisKeyManager.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/core/redis/RedisTTLPolicy.kt`
- Create: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/service/JoinRequestService.kt`

- [ ] **Step 1: `RedisKeyManager.kt`** — add key builders:

```kotlin
    fun joinRequest(requestId: String) = "join_request:$requestId"
    fun joinRequestOrgIndex(orgId: UUID) = "join_request:index:org:$orgId"
    fun joinRequestStoreIndex(storeId: UUID) = "join_request:index:store:$storeId"
    fun joinRequestUsername(usernameLower: String) = "join_request:username:$usernameLower"
    fun joinRequestLock(requestId: String) = "join_request:lock:$requestId"
```

- [ ] **Step 2: `RedisTTLPolicy.kt`** — add:

```kotlin
    val JOIN_REQUEST: Duration = Duration.ofDays(7)
```

- [ ] **Step 3: `JoinRequestService.kt`** — Redis CRUD + materialize-on-approve. Mirrors `RefreshTokenService`/`LoginAbortService` for String-serialized JSON payloads, but uses the spec's ZSET indexes (`score = createdAt epoch millis`) so listings are naturally newest-first and expired request hashes are lazily pruned.

```kotlin
package com.thomas.notiguide.domain.admin.service

import com.fasterxml.jackson.databind.ObjectMapper
import com.thomas.notiguide.core.exception.ConflictException
import com.thomas.notiguide.core.redis.RedisKeyManager
import com.thomas.notiguide.core.redis.RedisTTLPolicy
import com.thomas.notiguide.domain.admin.dto.JoinRequestDto
import com.thomas.notiguide.domain.admin.entity.Admin
import com.thomas.notiguide.domain.admin.repository.AdminRepository
import com.thomas.notiguide.domain.admin.types.AdminRole
import kotlinx.coroutines.flow.toList
import kotlinx.coroutines.reactor.awaitSingle
import kotlinx.coroutines.reactor.awaitSingleOrNull
import org.springframework.data.domain.Range
import org.springframework.data.redis.core.ReactiveRedisTemplate
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.security.SecureRandom
import java.time.Duration
import java.time.OffsetDateTime
import java.util.UUID

@Service
class JoinRequestService(
    private val redis: ReactiveRedisTemplate<String, String>,
    private val objectMapper: ObjectMapper,
    private val adminRepository: AdminRepository
) {
    private val secureRandom = SecureRandom()

    enum class TargetType { ORG, STORE }

    data class JoinRequestPayload(
        val username: String = "",
        val passwordHash: String = "",
        val targetType: TargetType = TargetType.ORG,
        val targetId: String = "",
        val createdAt: String = ""
    )

    /** Creates a pending join request. Returns the opaque requestId. */
    suspend fun create(username: String, passwordHash: String, targetType: TargetType, targetId: UUID): String {
        val requestId = generateOpaqueToken()
        val usernameKey = RedisKeyManager.joinRequestUsername(username.lowercase())
        val reserved = redis.opsForValue().setIfAbsent(usernameKey, requestId, RedisTTLPolicy.JOIN_REQUEST).awaitSingle()
        if (!reserved) throw ConflictException("Username '$username' is already taken")

        val createdAt = OffsetDateTime.now()
        val payload = JoinRequestPayload(
            username = username,
            passwordHash = passwordHash,
            targetType = targetType,
            targetId = targetId.toString(),
            createdAt = createdAt.toString()
        )
        val ttl = RedisTTLPolicy.JOIN_REQUEST

        try {
            redis.opsForValue().set(RedisKeyManager.joinRequest(requestId), objectMapper.writeValueAsString(payload), ttl).awaitSingle()

            val indexKey = when (targetType) {
                TargetType.ORG -> RedisKeyManager.joinRequestOrgIndex(targetId)
                TargetType.STORE -> RedisKeyManager.joinRequestStoreIndex(targetId)
            }
            redis.opsForZSet().add(indexKey, requestId, createdAt.toInstant().toEpochMilli().toDouble()).awaitSingle()
        } catch (ex: Exception) {
            redis.delete(usernameKey).awaitSingleOrNull()
            redis.delete(RedisKeyManager.joinRequest(requestId)).awaitSingleOrNull()
            throw ex
        }
        return requestId
    }

    suspend fun usernameReserved(usernameLower: String): Boolean =
        redis.hasKey(RedisKeyManager.joinRequestUsername(usernameLower)).awaitSingle()

    /** For the login "pending" hint: resolve a pending request by username. */
    suspend fun findPendingByUsername(usernameLower: String): JoinRequestPayload? {
        val requestId = redis.opsForValue().get(RedisKeyManager.joinRequestUsername(usernameLower)).awaitSingleOrNull()
            ?: return null
        return get(requestId)
    }

    suspend fun get(requestId: String): JoinRequestPayload? {
        val json = redis.opsForValue().get(RedisKeyManager.joinRequest(requestId)).awaitSingleOrNull() ?: return null
        return runCatching { objectMapper.readValue(json, JoinRequestPayload::class.java) }.getOrNull()
    }

    suspend fun listByOrg(orgId: UUID): List<JoinRequestDto> =
        hydrate(RedisKeyManager.joinRequestOrgIndex(orgId))

    suspend fun listByStore(storeId: UUID): List<JoinRequestDto> =
        hydrate(RedisKeyManager.joinRequestStoreIndex(storeId))

    private suspend fun hydrate(indexKey: String): List<JoinRequestDto> {
        val ids = redis.opsForZSet().reverseRange(indexKey, Range.unbounded<Long>()).collectList().awaitSingle()
        val result = mutableListOf<JoinRequestDto>()
        for (id in ids) {
            val payload = get(id)
            if (payload == null) {
                redis.opsForZSet().remove(indexKey, id).awaitSingleOrNull() // lazy prune expired
                continue
            }
            result.add(JoinRequestDto(requestId = id, username = payload.username, createdAt = payload.createdAt))
        }
        return result
    }

    /** Materialize an ADMIN row from a request, then delete the request. */
    @Transactional
    suspend fun approve(requestId: String, assignStoreId: UUID, verifierId: UUID): Admin {
        val lockKey = RedisKeyManager.joinRequestLock(requestId)
        val locked = redis.opsForValue().setIfAbsent(lockKey, "1", Duration.ofSeconds(30)).awaitSingle()
        if (!locked) throw ConflictException("Join request is already being processed")
        val payload = get(requestId) ?: throw ConflictException("Join request not found or expired")
        try {
            if (adminRepository.existsByUsername(payload.username)) {
                // Per spec, keep the request so the owner can retry/reject or let it expire.
                throw ConflictException("Username '${payload.username}' is already taken")
            }
            val admin = adminRepository.save(
                Admin(
                    username = payload.username,
                    passwordHash = payload.passwordHash,
                    role = AdminRole.ROLE_ADMIN,
                    storeId = assignStoreId,
                    orgId = null,
                    isVerified = true,
                    verifiedBy = verifierId,
                    verifiedAt = OffsetDateTime.now()
                )
            )
            deleteKeys(requestId, payload)
            return admin
        } finally {
            redis.delete(lockKey).awaitSingleOrNull()
        }
    }

    suspend fun reject(requestId: String) {
        val payload = get(requestId) ?: return
        deleteKeys(requestId, payload)
    }

    private suspend fun deleteKeys(requestId: String, payload: JoinRequestPayload) {
        redis.delete(RedisKeyManager.joinRequest(requestId)).awaitSingleOrNull()
        redis.delete(RedisKeyManager.joinRequestUsername(payload.username.lowercase())).awaitSingleOrNull()
        val indexKey = when (payload.targetType) {
            TargetType.ORG -> RedisKeyManager.joinRequestOrgIndex(UUID.fromString(payload.targetId))
            TargetType.STORE -> RedisKeyManager.joinRequestStoreIndex(UUID.fromString(payload.targetId))
        }
        redis.opsForZSet().remove(indexKey, requestId).awaitSingleOrNull()
    }

    private fun generateOpaqueToken(): String {
        val bytes = ByteArray(24)
        secureRandom.nextBytes(bytes)
        return bytes.joinToString("") { "%02x".format(it) }
    }
}
```

> Note: `approve` sets `isVerified = true`, `verifiedBy = verifierId`, and `verifiedAt = now` on the materialized row. No follow-up save is required.

- [ ] **Step 4: `JoinRequestDto.kt`.**

**Files:** Create `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/dto/JoinRequestDto.kt`

```kotlin
package com.thomas.notiguide.domain.admin.dto

data class JoinRequestDto(
    val requestId: String,
    val username: String,
    val createdAt: String
)
```

- [ ] **Step 5: Verify** — all Redis ops use `String` serializer (the project's `RedisConfig` uses `StringRedisSerializer` everywhere — payload is a JSON string, never a JSON-wrapped value). Username reservation uses `setIfAbsent`; index ZSETs are lazily pruned when member hashes expire; approval is protected by a short Redis lock so double-clicks/concurrent approvers cannot materialize the same request twice.

---

### Task 6.2: Registration request/response + RegistrationService + endpoint

**Files:**
- Create: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/request/RegisterRequest.kt`
- Create: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/response/RegisterResponse.kt`
- Create: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/service/RegistrationService.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/controller/AuthController.kt`

- [ ] **Step 1: `RegisterRequest.kt`** (reuses the username/password constraints from `CreateAdminRequest`):

```kotlin
package com.thomas.notiguide.domain.admin.request

import jakarta.validation.constraints.NotBlank
import jakarta.validation.constraints.NotNull
import jakarta.validation.constraints.Pattern
import jakarta.validation.constraints.Size

enum class RegisterMode { CREATE_ORG, CREATE_STORE, JOIN }

data class RegisterRequest(
    @field:NotNull
    val mode: RegisterMode,

    @field:NotBlank
    @field:Size(min = 3, max = 100)
    @field:Pattern(
        regexp = "^[a-zA-Z0-9_]+$",
        message = "Username must contain only letters, digits, and underscores"
    )
    val username: String,

    @field:NotBlank
    @field:Size(min = 8, max = 128)
    @field:Pattern(
        regexp = "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[^a-zA-Z0-9\\s]).+$",
        message = "Password must contain at least one uppercase letter, one lowercase letter, one digit, and one special character"
    )
    val password: String,

    @field:Size(max = 255)
    val orgName: String? = null,

    @field:Size(max = 255)
    val storeName: String? = null,

    @field:Size(max = 1000)
    val storeAddress: String? = null,

    @field:Size(max = 64)
    val joinCode: String? = null
)
```

- [ ] **Step 2: `RegisterResponse.kt`.**

```kotlin
package com.thomas.notiguide.domain.admin.response

import com.thomas.notiguide.domain.admin.types.AdminRole

enum class RegisterOutcome { ACTIVE, PENDING }

data class RegisterResponse(
    val outcome: RegisterOutcome,
    val role: AdminRole?,
    val targetType: String? = null
)
```

- [ ] **Step 3: `RegistrationService.kt`.**

```kotlin
package com.thomas.notiguide.domain.admin.service

import com.thomas.notiguide.core.exception.ConflictException
import com.thomas.notiguide.core.exception.HttpException
import com.thomas.notiguide.core.tenant.JoinCodeGenerator
import com.thomas.notiguide.domain.admin.entity.Admin
import com.thomas.notiguide.domain.admin.repository.AdminRepository
import com.thomas.notiguide.domain.admin.request.RegisterMode
import com.thomas.notiguide.domain.admin.request.RegisterRequest
import com.thomas.notiguide.domain.admin.response.RegisterOutcome
import com.thomas.notiguide.domain.admin.response.RegisterResponse
import com.thomas.notiguide.domain.admin.types.AdminRole
import com.thomas.notiguide.domain.organization.entity.Organization
import com.thomas.notiguide.domain.organization.repository.OrganizationRepository
import com.thomas.notiguide.domain.store.repository.StoreRepository
import com.thomas.notiguide.domain.store.request.CreateStoreRequest
import com.thomas.notiguide.domain.store.service.StoreService
import org.springframework.http.HttpStatus
import org.springframework.security.crypto.password.PasswordEncoder
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class RegistrationService(
    private val adminRepository: AdminRepository,
    private val organizationRepository: OrganizationRepository,
    private val storeRepository: StoreRepository,
    private val storeService: StoreService,
    private val passwordEncoder: PasswordEncoder,
    private val joinRequestService: JoinRequestService
) {
    @Transactional
    suspend fun register(request: RegisterRequest): RegisterResponse {
        val username = request.username.trim()
        requireUsernameAvailable(username)
        val passwordHash = passwordEncoder.encode(request.password)

        return when (request.mode) {
            RegisterMode.CREATE_ORG -> createOrg(username, passwordHash, request)
            RegisterMode.CREATE_STORE -> createStore(username, passwordHash, request)
            RegisterMode.JOIN -> join(username, passwordHash, request)
        }
    }

    private suspend fun requireUsernameAvailable(username: String) {
        if (adminRepository.existsByUsername(username))
            throw ConflictException("Username '$username' is already taken")
        if (joinRequestService.usernameReserved(username.lowercase()))
            throw ConflictException("Username '$username' is already taken")
    }

    private suspend fun createOrg(username: String, passwordHash: String, request: RegisterRequest): RegisterResponse {
        val orgName = request.orgName?.trim()?.takeIf { it.isNotBlank() }
            ?: throw HttpException(HttpStatus.BAD_REQUEST, "Organization name is required")

        val org = organizationRepository.save(
            Organization(name = orgName, joinCode = generateUniqueOrgCode())
        )
        val admin = adminRepository.save(
            Admin(
                username = username,
                passwordHash = passwordHash,
                role = AdminRole.ROLE_SUPER_ADMIN,
                orgId = org.id,
                storeId = null,
                isVerified = true
            )
        )
        organizationRepository.save(org.copy(createdBy = admin.id))
        return RegisterResponse(outcome = RegisterOutcome.ACTIVE, role = AdminRole.ROLE_SUPER_ADMIN)
    }

    private suspend fun createStore(username: String, passwordHash: String, request: RegisterRequest): RegisterResponse {
        val storeName = request.storeName?.trim()?.takeIf { it.isNotBlank() }
            ?: throw HttpException(HttpStatus.BAD_REQUEST, "Store name is required")

        val store = storeService.createStore(
            CreateStoreRequest(name = storeName, address = request.storeAddress?.takeIf { it.isNotBlank() }),
            orgId = null
        )
        adminRepository.save(
            Admin(
                username = username,
                passwordHash = passwordHash,
                role = AdminRole.ROLE_ADMIN,
                orgId = null,
                storeId = store.id,
                isVerified = true
            )
        )
        return RegisterResponse(outcome = RegisterOutcome.ACTIVE, role = AdminRole.ROLE_ADMIN)
    }

    private suspend fun join(username: String, passwordHash: String, request: RegisterRequest): RegisterResponse {
        val code = request.joinCode?.trim()?.takeIf { it.isNotBlank() }
            ?: throw HttpException(HttpStatus.BAD_REQUEST, "Join code is required")

        when {
            code.startsWith(JoinCodeGenerator.ORG_PREFIX) -> {
                val org = organizationRepository.findByJoinCode(code)
                    ?: throw HttpException(HttpStatus.BAD_REQUEST, "Invalid join code")
                joinRequestService.create(username, passwordHash, JoinRequestService.TargetType.ORG, org.id!!)
                return RegisterResponse(RegisterOutcome.PENDING, AdminRole.ROLE_ADMIN, "ORG")
            }
            code.startsWith(JoinCodeGenerator.STORE_PREFIX) -> {
                val store = storeRepository.findByJoinCode(code)
                    ?: throw HttpException(HttpStatus.BAD_REQUEST, "Invalid join code")
                if (store.orgId != null)
                    throw HttpException(HttpStatus.BAD_REQUEST, "Invalid join code")
                joinRequestService.create(username, passwordHash, JoinRequestService.TargetType.STORE, store.id!!)
                return RegisterResponse(RegisterOutcome.PENDING, AdminRole.ROLE_ADMIN, "STORE")
            }
            else -> throw HttpException(HttpStatus.BAD_REQUEST, "Invalid join code")
        }
    }

    private suspend fun generateUniqueOrgCode(): String {
        repeat(5) {
            val code = JoinCodeGenerator.generate(JoinCodeGenerator.ORG_PREFIX)
            if (!organizationRepository.existsByJoinCode(code)) return code
        }
        throw IllegalStateException("Could not generate a unique organization join code")
    }
}
```

- [ ] **Step 4: `AuthController` — add the public `register` endpoint.** Inject `RegistrationService` into the constructor, add the import, and add:

```kotlin
    @PostMapping("/register")
    suspend fun register(@Valid @RequestBody request: RegisterRequest): ResponseEntity<RegisterResponse> {
        val response = registrationService.register(request)
        val status = if (response.outcome == RegisterOutcome.PENDING) HttpStatus.ACCEPTED else HttpStatus.CREATED
        return ResponseEntity.status(status).body(response)
    }
```
(Add imports for `RegisterOutcome`, `RegisterRequest`, `RegisterResponse`, `RegistrationService`, and `HttpStatus`.)

> **Rate limiting:** no filter change needed. `/api/auth/register` is already matched by the `/api/auth/` prefix in `RateLimitFilter.resolveTier`, which applies the **`auth` tier (10 req/min/IP)** — the most restrictive existing tier (stricter than "strict" at 20/min). This corrects the spec's "strict tier" wording.

- [ ] **Step 5: Verify** — registration writes are inside `@Transactional`; `CREATE_ORG` ordering (org → admin → org.createdBy) satisfies both the FK and the new CHECK; `JOIN` creates no admin row and returns `202 Accepted`; active registrations return `201 Created`; the endpoint is public (already covered by `permitAll("/api/auth/**")` in `SecurityConfig`).

---

### Task 6.3: Login "awaiting approval" feedback

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/core/exception/HttpException.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/controller/AuthController.kt`

- [ ] **Step 1: Add `PendingJoinRequestException`** to `HttpException.kt`:

```kotlin
class PendingJoinRequestException
    : HttpException(HttpStatus.FORBIDDEN, "Your join request is awaiting approval")
```

- [ ] **Step 2: `AuthController.login`** — when the username isn't a real admin, check for a pending join request and verify the password against the stored hash before revealing the pending state (password-gated → no enumeration). Inject `JoinRequestService`. Replace the current "admin not found → BadCredentials" line:

```kotlin
        val admin = adminRepository.findByUsername(request.username.trim())
            ?: run {
                val pending = joinRequestService.findPendingByUsername(request.username.trim().lowercase())
                if (pending != null && passwordEncoder.matches(request.password, pending.passwordHash)) {
                    throw PendingJoinRequestException()
                }
                throw BadCredentialsException("Invalid username or password")
            }
```
(Add imports for `PendingJoinRequestException` and `JoinRequestService`; add `joinRequestService` to the constructor.)

- [ ] **Step 3: Verify** — a wrong password (or a non-pending username) still yields the generic 401 `BadCredentialsException`; only the genuine requester gets the distinct 403. No cookie/session/abort artifacts are issued on this path.

- [ ] **Step 4: `docs/CHANGELOGS.md`** — add a "Phase 6 — registration + join requests + login feedback" entry.

---

# Phase 7 — Approval endpoints

### Task 7.1: List / approve / reject join requests

**Files:**
- Create: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/controller/JoinRequestController.kt`

- [ ] **Step 1: Create `JoinRequestController`.** Authorization: a SUPER_ADMIN sees/approves requests targeting their org (and assigns a store in that org); an independent-store ADMIN sees/approves requests targeting their store. Approval validates that the request's target matches the caller's scope before materializing.

```kotlin
package com.thomas.notiguide.domain.admin.controller

import com.thomas.notiguide.core.exception.ConflictException
import com.thomas.notiguide.core.exception.ForbiddenException
import com.thomas.notiguide.core.exception.NotFoundException
import com.thomas.notiguide.domain.admin.dto.JoinRequestDto
import com.thomas.notiguide.domain.admin.service.JoinRequestService
import com.thomas.notiguide.domain.admin.types.AdminRole
import com.thomas.notiguide.domain.store.repository.StoreRepository
import com.thomas.notiguide.shared.principal.AdminPrincipal
import org.springframework.http.ResponseEntity
import org.springframework.security.core.annotation.AuthenticationPrincipal
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.PathVariable
import org.springframework.web.bind.annotation.PostMapping
import org.springframework.web.bind.annotation.RequestBody
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController
import java.util.UUID

data class ApproveJoinRequest(val storeId: UUID? = null)

@RestController
@RequestMapping("/api/admins/requests")
class JoinRequestController(
    private val joinRequestService: JoinRequestService,
    private val storeRepository: StoreRepository
) {
    @GetMapping
    suspend fun list(@AuthenticationPrincipal principal: AdminPrincipal): ResponseEntity<List<JoinRequestDto>> {
        val items = if (isSuperAdmin(principal)) {
            val orgId = principal.orgId ?: throw ForbiddenException("No organization assigned")
            joinRequestService.listByOrg(orgId)
        } else {
            val storeId = principal.storeId
                ?: throw ForbiddenException("Store-scoped admins need an assigned store")
            joinRequestService.listByStore(storeId)
        }
        return ResponseEntity.ok(items)
    }

    @PostMapping("/{requestId}/approve")
    suspend fun approve(
        @PathVariable requestId: String,
        @RequestBody(required = false) body: ApproveJoinRequest?,
        @AuthenticationPrincipal principal: AdminPrincipal
    ): ResponseEntity<Void> {
        val payload = joinRequestService.get(requestId)
            ?: throw NotFoundException("JoinRequest", "id", requestId)

        val assignStoreId: UUID = when (payload.targetType) {
            JoinRequestService.TargetType.ORG -> {
                val orgId = principal.orgId?.takeIf { isSuperAdmin(principal) }
                    ?: throw ForbiddenException("Only the organization owner can approve this request")
                if (UUID.fromString(payload.targetId) != orgId)
                    throw ForbiddenException("Request is not for your organization")
                val storeId = body?.storeId
                    ?: throw ConflictException("Select a store in your organization to assign")
                val store = storeRepository.findById(storeId)
                    ?: throw NotFoundException("Store", "id", storeId.toString())
                if (store.orgId != orgId)
                    throw ForbiddenException("Store is not in your organization")
                storeId
            }
            JoinRequestService.TargetType.STORE -> {
                val storeId = principal.storeId
                    ?: throw ForbiddenException("Only the store's admin can approve this request")
                if (UUID.fromString(payload.targetId) != storeId)
                    throw ForbiddenException("Request is not for your store")
                if (!principal.isVerified)
                    throw ForbiddenException("Only a verified admin can approve co-owners")
                storeId
            }
        }
        joinRequestService.approve(requestId, assignStoreId, principal.id)
        return ResponseEntity.noContent().build()
    }

    @PostMapping("/{requestId}/reject")
    suspend fun reject(
        @PathVariable requestId: String,
        @AuthenticationPrincipal principal: AdminPrincipal
    ): ResponseEntity<Void> {
        val payload = joinRequestService.get(requestId)
            ?: return ResponseEntity.noContent().build()
        // Same scope check as approve
        when (payload.targetType) {
            JoinRequestService.TargetType.ORG -> {
                val orgId = principal.orgId?.takeIf { isSuperAdmin(principal) }
                    ?: throw ForbiddenException("Only the organization owner can reject this request")
                if (UUID.fromString(payload.targetId) != orgId)
                    throw ForbiddenException("Request is not for your organization")
            }
            JoinRequestService.TargetType.STORE -> {
                val storeId = principal.storeId
                    ?: throw ForbiddenException("Only the store's admin can reject this request")
                if (UUID.fromString(payload.targetId) != storeId)
                    throw ForbiddenException("Request is not for your store")
            }
        }
        joinRequestService.reject(requestId)
        return ResponseEntity.noContent().build()
    }

    private fun isSuperAdmin(principal: AdminPrincipal): Boolean =
        principal.authorities.any { it.authority == AdminRole.ROLE_SUPER_ADMIN.name }
}
```

- [ ] **Step 2: Verify** — ORG requests are approvable only by the org's SUPER_ADMIN (who must supply a store in that org); STORE requests only by the store's verified ADMIN. Approval materializes a verified ADMIN row (Task 6.1) and clears the Redis keys.

- [ ] **Step 3: `docs/CHANGELOGS.md`** — add a "Phase 7 — join-request approval" entry.

---

# Phase 8 — Frontend foundation

### Task 8.1: Types, API routes, auth store, register API

**Files:**
- Modify: `web/src/types/admin.ts`
- Create: `web/src/types/organization.ts`
- Modify: `web/src/lib/constants.ts`
- Modify: `web/src/store/auth.ts`
- Modify: `web/src/features/auth/api.ts`

- [ ] **Step 1: `types/admin.ts`** — add `orgId` to `AdminDto` (after `role`): `orgId: string | null;`. Add registration + join-request types at the end:

```typescript
export type RegisterMode = "CREATE_ORG" | "CREATE_STORE" | "JOIN";
export type RegisterOutcome = "ACTIVE" | "PENDING";

export interface RegisterRequest {
  mode: RegisterMode;
  username: string;
  password: string;
  orgName?: string;
  storeName?: string;
  storeAddress?: string;
  joinCode?: string;
}

export interface RegisterResponse {
  outcome: RegisterOutcome;
  role: AdminRole | null;
  targetType: "ORG" | "STORE" | null;
}

export interface JoinRequestDto {
  requestId: string;
  username: string;
  createdAt: string;
}
```

- [ ] **Step 2: `types/organization.ts`.**

```typescript
export interface OrganizationDto {
  id: string;
  name: string;
  joinCode: string;
  createdBy: string | null;
  createdAt: string | null;
  updatedAt: string | null;
}

export interface JoinCodeResponse {
  joinCode: string;
}
```

- [ ] **Step 3: `lib/constants.ts`** — add `REGISTER: "/api/auth/register"` to `API_ROUTES.AUTH`; add `REQUESTS: "/api/admins/requests"`, `APPROVE_REQUEST: (id) => ...`, `REJECT_REQUEST: (id) => ...` to `ADMINS`; add `JOIN_CODE`/`JOIN_CODE_ROTATE` to `STORES`; add a new `ORGS` group:

```typescript
  AUTH: {
    LOGIN: "/api/auth/login",
    LOGOUT: "/api/auth/logout",
    REFRESH: "/api/auth/refresh",
    ABORT: "/api/auth/abort",
    REGISTER: "/api/auth/register",
  },
```
```typescript
    REQUESTS: "/api/admins/requests",
    APPROVE_REQUEST: (id: string) => `/api/admins/requests/${id}/approve`,
    REJECT_REQUEST: (id: string) => `/api/admins/requests/${id}/reject`,
```
```typescript
    JOIN_CODE: (id: string) => `/api/stores/${id}/join-code`,
    JOIN_CODE_ROTATE: (id: string) => `/api/stores/${id}/join-code/rotate`,
```
```typescript
  ORGS: {
    ME: "/api/orgs/me",
    JOIN_CODE: "/api/orgs/me/join-code",
    JOIN_CODE_ROTATE: "/api/orgs/me/join-code/rotate",
  },
```

- [ ] **Step 4: `store/auth.ts`** — add `orgId` to derived state. In `deriveState`: add `orgId: admin?.orgId ?? null,`; add `orgId: string | null;` to the `AuthState` interface and the initial state (`orgId: null`).

- [ ] **Step 5: `features/auth/api.ts`** — add the register call (public, `skipAuth`):

```typescript
import { post } from "@/lib/api";
import { API_ROUTES } from "@/lib/constants";
import type { RegisterRequest, RegisterResponse } from "@/types/admin";

export function register(request: RegisterRequest) {
  return post<RegisterResponse>(API_ROUTES.AUTH.REGISTER, request, {
    skipAuth: true,
  });
}
```
(Add to the existing file; keep existing exports.)

- [ ] **Step 6: Verify** — `cd web && yarn lint`. Expected: clean.

---

### Task 8.2: i18n keys (en + vi, mirrored)

**Files:**
- Modify: `web/src/messages/en.json`
- Modify: `web/src/messages/vi.json`

> Per CLAUDE.md: confirm the Vietnamese values below in chat before committing them to `vi.json`. EN is authoritative for meaning; VI must read natively. Keys/structure must mirror exactly.

- [ ] **Step 1: Add to the `auth` namespace** (both files). EN:

```json
    "createAccountLink": "Create an account",
    "haveAccountLink": "Already have an account? Sign in",
    "awaitingApproval": "Your request to join is awaiting approval. You'll be able to sign in once an owner approves it."
```
VI (draft to confirm):
```json
    "createAccountLink": "Tạo tài khoản",
    "haveAccountLink": "Đã có tài khoản? Đăng nhập",
    "awaitingApproval": "Yêu cầu tham gia của bạn đang chờ duyệt. Bạn có thể đăng nhập sau khi chủ sở hữu phê duyệt."
```

- [ ] **Step 2: Add a new top-level `register` namespace** (both files). EN:

```json
  "register": {
    "title": "Create your account",
    "subtitle": "Choose how you want to get started",
    "pathOrgTitle": "Start an organization",
    "pathOrgDesc": "Run multiple stores under one organization you own.",
    "pathStoreTitle": "Open my own store",
    "pathStoreDesc": "Run a single independent store.",
    "pathJoinTitle": "Join with a code",
    "pathJoinDesc": "Join an existing store or organization using an invite code.",
    "orgNameLabel": "Organization name",
    "orgNamePlaceholder": "e.g. Acme Coffee Group",
    "storeNameLabel": "Store name",
    "storeNamePlaceholder": "e.g. Acme Coffee — Downtown",
    "storeAddressLabel": "Store address (optional)",
    "storeAddressPlaceholder": "123 Main St",
    "joinCodeLabel": "Invite code",
    "joinCodePlaceholder": "o_… or s_…",
    "submitOrg": "Create organization",
    "submitStore": "Create store",
    "submitJoin": "Send join request",
    "back": "Back",
    "createdActiveTitle": "Account created",
    "createdActiveDesc": "Your account is ready. Please sign in.",
    "pendingTitle": "Request sent",
    "pendingDesc": "We've sent your join request. An owner must approve it before you can sign in.",
    "goToLogin": "Go to sign in",
    "joinCodeRequired": "An invite code is required",
    "invalidJoinCode": "That invite code is invalid",
    "orgNameRequired": "Organization name is required",
    "storeNameRequired": "Store name is required"
  }
```
VI (draft to confirm — keep concise/native):
```json
  "register": {
    "title": "Tạo tài khoản",
    "subtitle": "Chọn cách bạn muốn bắt đầu",
    "pathOrgTitle": "Lập tổ chức",
    "pathOrgDesc": "Quản lý nhiều cửa hàng trong một tổ chức của bạn.",
    "pathStoreTitle": "Mở cửa hàng riêng",
    "pathStoreDesc": "Quản lý một cửa hàng độc lập.",
    "pathJoinTitle": "Tham gia bằng mã",
    "pathJoinDesc": "Tham gia cửa hàng hoặc tổ chức có sẵn bằng mã mời.",
    "orgNameLabel": "Tên tổ chức",
    "orgNamePlaceholder": "vd: Acme Coffee Group",
    "storeNameLabel": "Tên cửa hàng",
    "storeNamePlaceholder": "vd: Acme Coffee — Quận 1",
    "storeAddressLabel": "Địa chỉ cửa hàng (không bắt buộc)",
    "storeAddressPlaceholder": "123 Đường Lê Lợi",
    "joinCodeLabel": "Mã mời",
    "joinCodePlaceholder": "o_… hoặc s_…",
    "submitOrg": "Tạo tổ chức",
    "submitStore": "Tạo cửa hàng",
    "submitJoin": "Gửi yêu cầu tham gia",
    "back": "Quay lại",
    "createdActiveTitle": "Đã tạo tài khoản",
    "createdActiveDesc": "Tài khoản đã sẵn sàng. Vui lòng đăng nhập.",
    "pendingTitle": "Đã gửi yêu cầu",
    "pendingDesc": "Yêu cầu tham gia đã được gửi. Chủ sở hữu cần duyệt trước khi bạn đăng nhập.",
    "goToLogin": "Đến trang đăng nhập",
    "joinCodeRequired": "Cần nhập mã mời",
    "invalidJoinCode": "Mã mời không hợp lệ",
    "orgNameRequired": "Cần nhập tên tổ chức",
    "storeNameRequired": "Cần nhập tên cửa hàng"
  }
```

- [ ] **Step 3: Add to the `admins` namespace** (join-request UI). EN:

```json
    "requestsTitle": "Pending join requests",
    "requestsEmpty": "No pending requests",
    "requestApprove": "Approve",
    "requestReject": "Reject",
    "requestApproveTitle": "Approve join request",
    "requestApproveStoreLabel": "Assign to store",
    "requestApprovedToast": "{username} approved",
    "requestRejectedToast": "Request rejected",
    "joinCodeTitle": "Invite code",
    "joinCodeDesc": "Share this code so others can request to join.",
    "joinCodeCopy": "Copy",
    "joinCodeCopied": "Copied",
    "joinCodeRotate": "Rotate code",
    "joinCodeRotateConfirm": "Rotating invalidates the current code for new requests. Continue?"
```
VI (draft to confirm):
```json
    "requestsTitle": "Yêu cầu tham gia",
    "requestsEmpty": "Không có yêu cầu nào",
    "requestApprove": "Duyệt",
    "requestReject": "Từ chối",
    "requestApproveTitle": "Duyệt yêu cầu tham gia",
    "requestApproveStoreLabel": "Gán vào cửa hàng",
    "requestApprovedToast": "Đã duyệt {username}",
    "requestRejectedToast": "Đã từ chối yêu cầu",
    "joinCodeTitle": "Mã mời",
    "joinCodeDesc": "Chia sẻ mã này để người khác gửi yêu cầu tham gia.",
    "joinCodeCopy": "Sao chép",
    "joinCodeCopied": "Đã sao chép",
    "joinCodeRotate": "Đổi mã",
    "joinCodeRotateConfirm": "Đổi mã sẽ vô hiệu hóa mã hiện tại cho yêu cầu mới. Tiếp tục?"
```

- [ ] **Step 4: Relabel role display keys per spec §4.4.** Update every existing `roleSuperAdmin` / `roleAdmin` pair in `navigation`, `settings`, and `admins` namespaces so users no longer see raw platform-role language. EN:

```json
    "roleSuperAdmin": "Organization owner",
    "roleAdmin": "Store manager"
```
VI (draft to confirm):
```json
    "roleSuperAdmin": "Chủ tổ chức",
    "roleAdmin": "Quản lý cửa hàng"
```

- [ ] **Step 5: Add organization keys to the `settings` namespace** (the settings sub-nav reads `useTranslations("settings")`, and current tab labels use `*Tab` keys). EN:

```json
    "organizationTab": "Organization",
    "organizationTitle": "Organization"
```
VI:
```json
    "organizationTab": "Tổ chức",
    "organizationTitle": "Tổ chức"
```
(No `navigation`-namespace key is needed — the Organization link lives in the settings sub-nav, not the main sidebar.)

- [ ] **Step 6: Verify** — `cd web && yarn lint`. Confirm `en.json` and `vi.json` have identical key sets (diff the key trees).

- [ ] **Step 7: `docs/CHANGELOGS.md`** — add a "Phase 8 — frontend foundation (types, routes, i18n)" entry.

---

# Phase 9 — Register page

### Task 9.1: Register route shell + path selector

**Files:**
- Create: `web/src/app/[locale]/(auth)/register/page.tsx`
- Create: `web/src/features/auth/register-path-selector.tsx`

- [ ] **Step 1: `register-path-selector.tsx`** — three selectable option rows implemented as buttons with icons. Do not use shadcn `Card` inside the auth card; this avoids a card-in-card layout while keeping the options easy to scan. Uses the existing spacing conventions; no warning boxes here.

```tsx
"use client";

import { Building2, KeyRound, Store } from "lucide-react";
import { useTranslations } from "next-intl";
import type { RegisterMode } from "@/types/admin";

interface RegisterPathSelectorProps {
  onSelect: (mode: RegisterMode) => void;
}

const PATHS: { mode: RegisterMode; icon: typeof Building2; titleKey: string; descKey: string }[] = [
  { mode: "CREATE_ORG", icon: Building2, titleKey: "pathOrgTitle", descKey: "pathOrgDesc" },
  { mode: "CREATE_STORE", icon: Store, titleKey: "pathStoreTitle", descKey: "pathStoreDesc" },
  { mode: "JOIN", icon: KeyRound, titleKey: "pathJoinTitle", descKey: "pathJoinDesc" },
];

export function RegisterPathSelector({ onSelect }: RegisterPathSelectorProps) {
  const t = useTranslations("register");
  return (
    <div className="space-y-3">
      {PATHS.map(({ mode, icon: Icon, titleKey, descKey }) => (
        <button
          key={mode}
          type="button"
          onClick={() => onSelect(mode)}
          className="flex w-full items-start gap-3 rounded-xl border border-border bg-card p-4 text-left transition-colors hover:border-primary/50 hover:bg-accent"
        >
          <span className="mt-0.5 flex size-9 shrink-0 items-center justify-center rounded-lg bg-primary/10 text-primary">
            <Icon aria-hidden="true" className="size-5" />
          </span>
          <span className="space-y-0.5">
            <span className="block text-sm font-semibold">{t(titleKey)}</span>
            <span className="block text-sm text-muted-foreground">{t(descKey)}</span>
          </span>
        </button>
      ))}
    </div>
  );
}
```

- [ ] **Step 2: `register/page.tsx`** — mirrors the login page shell (glass card, orbs, language/theme toggles), holds the `mode` state, and routes to the right form / pending screen. (Forms imported in Task 9.2; pending screen in Task 9.3 — write the page now importing all of them.)

```tsx
"use client";

import { Ticket } from "lucide-react";
import { useLocale, useTranslations } from "next-intl";
import { useState } from "react";
import { LanguageSwitcher } from "@/components/layout/language-switcher";
import { ThemeToggle } from "@/components/layout/theme-toggle";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { register } from "@/features/auth/api";
import { RegisterCreateOrgForm } from "@/features/auth/register-create-org-form";
import { RegisterCreateStoreForm } from "@/features/auth/register-create-store-form";
import { RegisterJoinForm } from "@/features/auth/register-join-form";
import { RegisterPathSelector } from "@/features/auth/register-path-selector";
import { RegisterPendingScreen } from "@/features/auth/register-pending-screen";
import { Link } from "@/i18n/navigation";
import type { RegisterMode, RegisterRequest } from "@/types/admin";

type View = "select" | RegisterMode | "active" | "pending";

export default function RegisterPage() {
  const locale = useLocale();
  const t = useTranslations("register");
  const tAuth = useTranslations("auth");
  const [view, setView] = useState<View>("select");
  const [submitting, setSubmitting] = useState(false);

  async function submit(request: RegisterRequest) {
    setSubmitting(true);
    try {
      const res = await register(request);
      setView(res.outcome === "PENDING" ? "pending" : "active");
      return null;
    } finally {
      setSubmitting(false);
    }
  }

  return (
    <div className="bg-gradient-auth relative flex min-h-screen items-center justify-center overflow-hidden p-3 s:p-4 l:p-6">
      <div className="absolute top-3 right-3 z-20 flex items-center gap-2 s:top-4 s:right-4">
        <LanguageSwitcher />
        <ThemeToggle />
      </div>
      <div className="login-orb login-orb-primary" />
      <div className="login-orb login-orb-secondary" />
      <Card className="glass-card relative z-10 w-full max-w-md pb-5 pt-6">
        <CardHeader className="gap-2 px-6 text-center">
          <div className="mx-auto mb-3 flex size-12 items-center justify-center rounded-xl bg-primary text-primary-foreground">
            <Ticket aria-hidden="true" className="size-6" />
          </div>
          <CardTitle className="text-2xl font-bold">{t("title")}</CardTitle>
          <p className="text-sm text-muted-foreground">{t("subtitle")}</p>
        </CardHeader>
        <CardContent className="px-6 pb-4">
          {view === "select" && <RegisterPathSelector onSelect={(m) => setView(m)} />}
          {view === "CREATE_ORG" && (
            <RegisterCreateOrgForm submitting={submitting} onBack={() => setView("select")} onSubmit={submit} />
          )}
          {view === "CREATE_STORE" && (
            <RegisterCreateStoreForm submitting={submitting} onBack={() => setView("select")} onSubmit={submit} />
          )}
          {view === "JOIN" && (
            <RegisterJoinForm submitting={submitting} onBack={() => setView("select")} onSubmit={submit} />
          )}
          {view === "active" && <RegisterPendingScreen variant="active" />}
          {view === "pending" && <RegisterPendingScreen variant="pending" />}

          {(view === "select" || view === "CREATE_ORG" || view === "CREATE_STORE" || view === "JOIN") && (
            <p className="mt-6 text-center text-sm text-muted-foreground">
              <Link
                href="/login"
                locale={locale}
                className="text-primary underline decoration-primary/30 underline-offset-4 transition-colors hover:decoration-primary/70"
              >
                {tAuth("haveAccountLink")}
              </Link>
            </p>
          )}
        </CardContent>
      </Card>
    </div>
  );
}
```

- [ ] **Step 3: Verify** — `cd web && yarn lint`.

---

### Task 9.2: The three registration forms

**Files:**
- Create: `web/src/features/auth/register-create-org-form.tsx`
- Create: `web/src/features/auth/register-create-store-form.tsx`
- Create: `web/src/features/auth/register-join-form.tsx`

Shared props contract (define inline in each file):
```tsx
interface RegisterFormProps {
  submitting: boolean;
  onBack: () => void;
  onSubmit: (request: RegisterRequest) => Promise<string | null>;
}
```

All three reuse: `Input`, `Label`, `Button`, `InlineError`, and (org/store forms) `PasswordValidationChecklist`, the show/hide password toggle, and the username/password validation pattern from `create-admin-dialog.tsx` (`USERNAME_RULES`, `PASSWORD_RULES`, `getFirstMissingPasswordRequirementKey`). On `ApiError` 409 → set username error to `tAdmins("usernameTaken")`; on 400 with `invalid join code` message (join form) → set join-code error.

- [ ] **Step 1: `register-create-org-form.tsx`** — fields: username, password (with checklist + toggle), orgName. Validates locally, then `onSubmit({ mode: "CREATE_ORG", username, password, orgName })`.

```tsx
"use client";

import { Eye, EyeOff, Loader2 } from "lucide-react";
import { useTranslations } from "next-intl";
import type React from "react";
import { useState } from "react";
import { Button } from "@/components/ui/button";
import { InlineError } from "@/components/ui/inline-error";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { PasswordValidationChecklist } from "@/components/ui/password-validation-checklist";
import { PASSWORD_RULES, USERNAME_RULES } from "@/lib/constants";
import { getFirstMissingPasswordRequirementKey } from "@/lib/password-validation";
import { ApiError } from "@/types/api";
import type { RegisterRequest } from "@/types/admin";

interface RegisterFormProps {
  submitting: boolean;
  onBack: () => void;
  onSubmit: (request: RegisterRequest) => Promise<string | null>;
}

export function RegisterCreateOrgForm({ submitting, onBack, onSubmit }: RegisterFormProps) {
  const t = useTranslations("register");
  const tAuth = useTranslations("auth");
  const tAdmins = useTranslations("admins");
  const tValidation = useTranslations("validation");
  const [username, setUsername] = useState("");
  const [password, setPassword] = useState("");
  const [orgName, setOrgName] = useState("");
  const [showPassword, setShowPassword] = useState(false);
  const [errors, setErrors] = useState<Record<string, string>>({});

  function validate(): boolean {
    const e: Record<string, string> = {};
    if (!username.trim()) e.username = tAuth("usernameRequired");
    else if (username.length < USERNAME_RULES.MIN_LENGTH) e.username = tValidation("usernameMin", { min: USERNAME_RULES.MIN_LENGTH });
    else if (!USERNAME_RULES.PATTERN.test(username)) e.username = tValidation("usernamePattern");
    if (!password) e.password = tAuth("passwordRequired");
    else {
      const missing = getFirstMissingPasswordRequirementKey(password);
      if (missing) e.password = tValidation("missingRequirement", { requirement: tValidation(missing) });
    }
    if (!orgName.trim()) e.orgName = t("orgNameRequired");
    setErrors(e);
    return Object.keys(e).length === 0;
  }

  async function handleSubmit(ev: React.SyntheticEvent) {
    ev.preventDefault();
    if (submitting || !validate()) return;
    try {
      await onSubmit({ mode: "CREATE_ORG", username, password, orgName });
    } catch (err) {
      if (err instanceof ApiError && err.code === 409) setErrors({ username: tAdmins("usernameTaken") });
      else if (err instanceof ApiError) setErrors({ form: err.message });
      else setErrors({ form: tAuth("connectionLost") });
    }
  }

  return (
    <form onSubmit={handleSubmit} noValidate className="space-y-4">
      <div className="space-y-2">
        <Label htmlFor="reg-username">{tAdmins("usernameLabel")}</Label>
        <Input id="reg-username" value={username} maxLength={USERNAME_RULES.MAX_LENGTH}
          onChange={(e) => setUsername(e.target.value)} aria-invalid={!!errors.username} autoComplete="username" />
        {errors.username && <InlineError message={errors.username} className="mt-1" />}
      </div>
      <div className="space-y-2">
        <Label htmlFor="reg-password">{tAdmins("passwordLabel")}</Label>
        <div className="relative">
          <Input id="reg-password" type={showPassword ? "text" : "password"} value={password}
            maxLength={PASSWORD_RULES.MAX_LENGTH} onChange={(e) => setPassword(e.target.value)}
            aria-invalid={!!errors.password} autoComplete="new-password" />
          <button type="button" onClick={() => setShowPassword((v) => !v)}
            className="absolute top-1/2 right-3 -translate-y-1/2 text-muted-foreground hover:text-foreground"
            aria-label={showPassword ? tAuth("hidePassword") : tAuth("showPassword")}>
            {showPassword ? <EyeOff aria-hidden="true" className="size-4" /> : <Eye aria-hidden="true" className="size-4" />}
          </button>
        </div>
        <PasswordValidationChecklist password={password} className="mt-2" />
        {errors.password && <InlineError message={errors.password} className="mt-1" />}
      </div>
      <div className="space-y-2">
        <Label htmlFor="reg-orgname">{t("orgNameLabel")}</Label>
        <Input id="reg-orgname" value={orgName} maxLength={255} placeholder={t("orgNamePlaceholder")}
          onChange={(e) => setOrgName(e.target.value)} aria-invalid={!!errors.orgName} />
        {errors.orgName && <InlineError message={errors.orgName} className="mt-1" />}
      </div>
      {errors.form && <InlineError message={errors.form} />}
      <div className="flex gap-2">
        <Button type="button" variant="outline" className="flex-1" onClick={onBack}>{t("back")}</Button>
        <Button type="submit" disabled={submitting}
          className="flex-1 bg-primary text-primary-foreground hover:bg-primary-hover">
          {submitting && <Loader2 aria-hidden="true" className="mr-2 size-4 animate-spin" />}
          {t("submitOrg")}
        </Button>
      </div>
    </form>
  );
}
```

- [ ] **Step 2: `register-create-store-form.tsx`** — full code (username + password as in Step 1, plus required `storeName` and optional `storeAddress`):

```tsx
"use client";

import { Eye, EyeOff, Loader2 } from "lucide-react";
import { useTranslations } from "next-intl";
import type React from "react";
import { useState } from "react";
import { Button } from "@/components/ui/button";
import { InlineError } from "@/components/ui/inline-error";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { PasswordValidationChecklist } from "@/components/ui/password-validation-checklist";
import { PASSWORD_RULES, USERNAME_RULES } from "@/lib/constants";
import { getFirstMissingPasswordRequirementKey } from "@/lib/password-validation";
import { ApiError } from "@/types/api";
import type { RegisterRequest } from "@/types/admin";

interface RegisterFormProps {
  submitting: boolean;
  onBack: () => void;
  onSubmit: (request: RegisterRequest) => Promise<string | null>;
}

export function RegisterCreateStoreForm({ submitting, onBack, onSubmit }: RegisterFormProps) {
  const t = useTranslations("register");
  const tAuth = useTranslations("auth");
  const tAdmins = useTranslations("admins");
  const tValidation = useTranslations("validation");
  const [username, setUsername] = useState("");
  const [password, setPassword] = useState("");
  const [storeName, setStoreName] = useState("");
  const [storeAddress, setStoreAddress] = useState("");
  const [showPassword, setShowPassword] = useState(false);
  const [errors, setErrors] = useState<Record<string, string>>({});

  function validate(): boolean {
    const e: Record<string, string> = {};
    if (!username.trim()) e.username = tAuth("usernameRequired");
    else if (username.length < USERNAME_RULES.MIN_LENGTH) e.username = tValidation("usernameMin", { min: USERNAME_RULES.MIN_LENGTH });
    else if (!USERNAME_RULES.PATTERN.test(username)) e.username = tValidation("usernamePattern");
    if (!password) e.password = tAuth("passwordRequired");
    else {
      const missing = getFirstMissingPasswordRequirementKey(password);
      if (missing) e.password = tValidation("missingRequirement", { requirement: tValidation(missing) });
    }
    if (!storeName.trim()) e.storeName = t("storeNameRequired");
    setErrors(e);
    return Object.keys(e).length === 0;
  }

  async function handleSubmit(ev: React.SyntheticEvent) {
    ev.preventDefault();
    if (submitting || !validate()) return;
    try {
      await onSubmit({
        mode: "CREATE_STORE",
        username,
        password,
        storeName,
        storeAddress: storeAddress.trim() || undefined,
      });
    } catch (err) {
      if (err instanceof ApiError && err.code === 409) setErrors({ username: tAdmins("usernameTaken") });
      else if (err instanceof ApiError) setErrors({ form: err.message });
      else setErrors({ form: tAuth("connectionLost") });
    }
  }

  return (
    <form onSubmit={handleSubmit} noValidate className="space-y-4">
      <div className="space-y-2">
        <Label htmlFor="reg-username">{tAdmins("usernameLabel")}</Label>
        <Input id="reg-username" value={username} maxLength={USERNAME_RULES.MAX_LENGTH}
          onChange={(e) => setUsername(e.target.value)} aria-invalid={!!errors.username} autoComplete="username" />
        {errors.username && <InlineError message={errors.username} className="mt-1" />}
      </div>
      <div className="space-y-2">
        <Label htmlFor="reg-password">{tAdmins("passwordLabel")}</Label>
        <div className="relative">
          <Input id="reg-password" type={showPassword ? "text" : "password"} value={password}
            maxLength={PASSWORD_RULES.MAX_LENGTH} onChange={(e) => setPassword(e.target.value)}
            aria-invalid={!!errors.password} autoComplete="new-password" />
          <button type="button" onClick={() => setShowPassword((v) => !v)}
            className="absolute top-1/2 right-3 -translate-y-1/2 text-muted-foreground hover:text-foreground"
            aria-label={showPassword ? tAuth("hidePassword") : tAuth("showPassword")}>
            {showPassword ? <EyeOff aria-hidden="true" className="size-4" /> : <Eye aria-hidden="true" className="size-4" />}
          </button>
        </div>
        <PasswordValidationChecklist password={password} className="mt-2" />
        {errors.password && <InlineError message={errors.password} className="mt-1" />}
      </div>
      <div className="space-y-2">
        <Label htmlFor="reg-storename">{t("storeNameLabel")}</Label>
        <Input id="reg-storename" value={storeName} maxLength={255} placeholder={t("storeNamePlaceholder")}
          onChange={(e) => setStoreName(e.target.value)} aria-invalid={!!errors.storeName} />
        {errors.storeName && <InlineError message={errors.storeName} className="mt-1" />}
      </div>
      <div className="space-y-2">
        <Label htmlFor="reg-storeaddr">{t("storeAddressLabel")}</Label>
        <Input id="reg-storeaddr" value={storeAddress} maxLength={1000} placeholder={t("storeAddressPlaceholder")}
          onChange={(e) => setStoreAddress(e.target.value)} />
      </div>
      {errors.form && <InlineError message={errors.form} />}
      <div className="flex gap-2">
        <Button type="button" variant="outline" className="flex-1" onClick={onBack}>{t("back")}</Button>
        <Button type="submit" disabled={submitting}
          className="flex-1 bg-primary text-primary-foreground hover:bg-primary-hover">
          {submitting && <Loader2 aria-hidden="true" className="mr-2 size-4 animate-spin" />}
          {t("submitStore")}
        </Button>
      </div>
    </form>
  );
}
```

- [ ] **Step 3: `register-join-form.tsx`** — full code (username + password as in Step 1, plus a required `joinCode`; on a 400 mentioning "join code" set the field error):

```tsx
"use client";

import { Eye, EyeOff, Loader2 } from "lucide-react";
import { useTranslations } from "next-intl";
import type React from "react";
import { useState } from "react";
import { Button } from "@/components/ui/button";
import { InlineError } from "@/components/ui/inline-error";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { PasswordValidationChecklist } from "@/components/ui/password-validation-checklist";
import { PASSWORD_RULES, USERNAME_RULES } from "@/lib/constants";
import { getFirstMissingPasswordRequirementKey } from "@/lib/password-validation";
import { ApiError } from "@/types/api";
import type { RegisterRequest } from "@/types/admin";

interface RegisterFormProps {
  submitting: boolean;
  onBack: () => void;
  onSubmit: (request: RegisterRequest) => Promise<string | null>;
}

export function RegisterJoinForm({ submitting, onBack, onSubmit }: RegisterFormProps) {
  const t = useTranslations("register");
  const tAuth = useTranslations("auth");
  const tAdmins = useTranslations("admins");
  const tValidation = useTranslations("validation");
  const [username, setUsername] = useState("");
  const [password, setPassword] = useState("");
  const [joinCode, setJoinCode] = useState("");
  const [showPassword, setShowPassword] = useState(false);
  const [errors, setErrors] = useState<Record<string, string>>({});

  function validate(): boolean {
    const e: Record<string, string> = {};
    if (!username.trim()) e.username = tAuth("usernameRequired");
    else if (username.length < USERNAME_RULES.MIN_LENGTH) e.username = tValidation("usernameMin", { min: USERNAME_RULES.MIN_LENGTH });
    else if (!USERNAME_RULES.PATTERN.test(username)) e.username = tValidation("usernamePattern");
    if (!password) e.password = tAuth("passwordRequired");
    else {
      const missing = getFirstMissingPasswordRequirementKey(password);
      if (missing) e.password = tValidation("missingRequirement", { requirement: tValidation(missing) });
    }
    if (!joinCode.trim()) e.joinCode = t("joinCodeRequired");
    setErrors(e);
    return Object.keys(e).length === 0;
  }

  async function handleSubmit(ev: React.SyntheticEvent) {
    ev.preventDefault();
    if (submitting || !validate()) return;
    try {
      await onSubmit({ mode: "JOIN", username, password, joinCode: joinCode.trim() });
    } catch (err) {
      if (err instanceof ApiError && err.code === 409) setErrors({ username: tAdmins("usernameTaken") });
      else if (err instanceof ApiError && err.message.toLowerCase().includes("join code")) setErrors({ joinCode: t("invalidJoinCode") });
      else if (err instanceof ApiError) setErrors({ form: err.message });
      else setErrors({ form: tAuth("connectionLost") });
    }
  }

  return (
    <form onSubmit={handleSubmit} noValidate className="space-y-4">
      <div className="space-y-2">
        <Label htmlFor="reg-username">{tAdmins("usernameLabel")}</Label>
        <Input id="reg-username" value={username} maxLength={USERNAME_RULES.MAX_LENGTH}
          onChange={(e) => setUsername(e.target.value)} aria-invalid={!!errors.username} autoComplete="username" />
        {errors.username && <InlineError message={errors.username} className="mt-1" />}
      </div>
      <div className="space-y-2">
        <Label htmlFor="reg-password">{tAdmins("passwordLabel")}</Label>
        <div className="relative">
          <Input id="reg-password" type={showPassword ? "text" : "password"} value={password}
            maxLength={PASSWORD_RULES.MAX_LENGTH} onChange={(e) => setPassword(e.target.value)}
            aria-invalid={!!errors.password} autoComplete="new-password" />
          <button type="button" onClick={() => setShowPassword((v) => !v)}
            className="absolute top-1/2 right-3 -translate-y-1/2 text-muted-foreground hover:text-foreground"
            aria-label={showPassword ? tAuth("hidePassword") : tAuth("showPassword")}>
            {showPassword ? <EyeOff aria-hidden="true" className="size-4" /> : <Eye aria-hidden="true" className="size-4" />}
          </button>
        </div>
        <PasswordValidationChecklist password={password} className="mt-2" />
        {errors.password && <InlineError message={errors.password} className="mt-1" />}
      </div>
      <div className="space-y-2">
        <Label htmlFor="reg-joincode">{t("joinCodeLabel")}</Label>
        <Input id="reg-joincode" value={joinCode} maxLength={64} placeholder={t("joinCodePlaceholder")}
          className="font-mono" onChange={(e) => setJoinCode(e.target.value)} aria-invalid={!!errors.joinCode} />
        {errors.joinCode && <InlineError message={errors.joinCode} className="mt-1" />}
      </div>
      {errors.form && <InlineError message={errors.form} />}
      <div className="flex gap-2">
        <Button type="button" variant="outline" className="flex-1" onClick={onBack}>{t("back")}</Button>
        <Button type="submit" disabled={submitting}
          className="flex-1 bg-primary text-primary-foreground hover:bg-primary-hover">
          {submitting && <Loader2 aria-hidden="true" className="mr-2 size-4 animate-spin" />}
          {t("submitJoin")}
        </Button>
      </div>
    </form>
  );
}
```

- [ ] **Step 4: Verify** — `cd web && yarn lint`.

---

### Task 9.3: Pending/active screen + login "Create an account" link

**Files:**
- Create: `web/src/features/auth/register-pending-screen.tsx`
- Modify: `web/src/app/[locale]/(auth)/login/page.tsx`

- [ ] **Step 1: `register-pending-screen.tsx`** — both the "active" (created, go sign in) and "pending" (awaiting approval) terminal states. The pending state uses the **auth-page warning box** (`rounded-lg`, per conventions).

```tsx
"use client";

import { CheckCircle2, Clock } from "lucide-react";
import { useLocale, useTranslations } from "next-intl";
import { Button } from "@/components/ui/button";
import { Link } from "@/i18n/navigation";

interface RegisterPendingScreenProps {
  variant: "active" | "pending";
}

export function RegisterPendingScreen({ variant }: RegisterPendingScreenProps) {
  const t = useTranslations("register");
  const locale = useLocale();
  const isPending = variant === "pending";

  return (
    <div className="space-y-5 text-center">
      {isPending ? (
        <div className="rounded-lg border border-warning/40 bg-warning/15 p-4 text-sm font-medium text-warning dark:border-warning/50 dark:bg-warning/20">
          <Clock aria-hidden="true" className="mx-auto mb-2 size-6" />
          <p className="text-base font-semibold">{t("pendingTitle")}</p>
          <p className="mt-1 font-normal">{t("pendingDesc")}</p>
        </div>
      ) : (
        <div className="rounded-lg border border-success/40 bg-success/10 p-4 text-sm font-medium text-success dark:border-success/50 dark:bg-success/15">
          <CheckCircle2 aria-hidden="true" className="mx-auto mb-2 size-6" />
          <p className="text-base font-semibold">{t("createdActiveTitle")}</p>
          <p className="mt-1 font-normal">{t("createdActiveDesc")}</p>
        </div>
      )}
      <Button
        render={<Link href="/login" locale={locale} />}
        className="w-full bg-primary text-primary-foreground hover:bg-primary-hover"
      >
        {t("goToLogin")}
      </Button>
    </div>
  );
}
```

- [ ] **Step 2: `login/page.tsx`** — (a) add a "Create an account" link below the sign-in button; (b) add an "awaiting approval" banner driven by a new `pendingBanner` state set when login throws the pending-join 403.

In the `catch (err)` block, add a branch before the generic 403 unverified check:
```tsx
        if (
          err.code === 403 &&
          err.message.toLowerCase().includes("awaiting approval")
        ) {
          setPendingBanner(true);
        } else if (
          err.code === 403 &&
          err.message.toLowerCase().includes("not been verified")
        ) {
          setUnverifiedBanner(true);
        } else if (err.code === 401) {
```
Add state: `const [pendingBanner, setPendingBanner] = useState(false);` and reset it (`setPendingBanner(false)`) at the top of `handleSubmit` alongside `setUnverifiedBanner(false)`.

Add the banner near the `unverifiedBanner` block (auth-page `rounded-lg` warning box):
```tsx
          {pendingBanner && (
            <div className="mb-5 rounded-lg border border-warning/40 bg-warning/15 p-4 text-sm font-medium text-warning dark:border-warning/50 dark:bg-warning/20">
              {tAuth("awaitingApproval")}
            </div>
          )}
```

Add the link below the form's submit `Button` (inside the `<form>`, after the submit button), using `Link` + `useLocale` (already importable):
```tsx
            <p className="text-center text-sm text-muted-foreground">
              <Link
                href="/register"
                locale={locale}
                className="text-primary underline decoration-primary/30 underline-offset-4 transition-colors hover:decoration-primary/70"
              >
                {tAuth("createAccountLink")}
              </Link>
            </p>
```
(`Link` import: `import { Link, useRouter } from "@/i18n/navigation";` — extend the existing `useRouter` import.)

- [ ] **Step 3: Verify** — `cd web && yarn lint`. Manually trace: pending 403 → pending banner; unverified 403 → unverified banner; 401 → invalid credentials.

- [ ] **Step 4: `docs/CHANGELOGS.md`** — add a "Phase 9 — register page + login link/banner" entry.

---

# Phase 10 — Approval UI & join-code panels

### Task 10.1: Join-requests panel + approve dialog (admins page)

**Files:**
- Create: `web/src/features/admin/join-requests-panel.tsx`
- Create: `web/src/features/admin/approve-join-request-dialog.tsx`
- Create: `web/src/features/admin/reject-join-request-dialog.tsx`
- Modify: `web/src/features/admin/api.ts`
- Modify: `web/src/app/[locale]/dashboard/admins/page.tsx`

- [ ] **Step 1: `features/admin/api.ts`** — add request endpoints:

```typescript
import type { JoinRequestDto } from "@/types/admin";

export function listJoinRequests() {
  return get<JoinRequestDto[]>(API_ROUTES.ADMINS.REQUESTS);
}

export function approveJoinRequest(requestId: string, storeId?: string) {
  return post<void>(API_ROUTES.ADMINS.APPROVE_REQUEST(requestId), storeId ? { storeId } : undefined);
}

export function rejectJoinRequest(requestId: string) {
  return post<void>(API_ROUTES.ADMINS.REJECT_REQUEST(requestId), undefined);
}
```
(Extend the existing import of `@/types/admin` to include `JoinRequestDto`; `get`/`post` are already imported.)

- [ ] **Step 2: `approve-join-request-dialog.tsx`** — SUPER_ADMIN must pick a store (Select, mirroring `create-admin-dialog.tsx`); store-ADMIN approval needs no store (the dialog shows just a confirm). Props: `{ open, onOpenChange, request, requireStore, stores, onConfirm }`. Use the repo's existing consequential-action convention: shadcn `AlertDialog` + `Select`, with `AlertDialogCancel`, `AlertDialogAction`, `preventDefault()`, and loading-disabled actions. Do **not** use plain `Dialog` for this approval flow.

```tsx
"use client";

import { Loader2 } from "lucide-react";
import { useTranslations } from "next-intl";
import { useState } from "react";
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
import { Label } from "@/components/ui/label";
import { Select, SelectContent, SelectItem, SelectTrigger } from "@/components/ui/select";
import type { JoinRequestDto } from "@/types/admin";
import type { StoreDto } from "@/types/store";

interface ApproveJoinRequestDialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  request: JoinRequestDto | null;
  requireStore: boolean;
  stores: StoreDto[];
  onConfirm: (storeId?: string) => Promise<void>;
}

export function ApproveJoinRequestDialog({ open, onOpenChange, request, requireStore, stores, onConfirm }: ApproveJoinRequestDialogProps) {
  const tAdmins = useTranslations("admins");
  const tCommon = useTranslations("common");
  const [storeId, setStoreId] = useState("");
  const [loading, setLoading] = useState(false);

  async function confirm() {
    if (requireStore && !storeId) return;
    setLoading(true);
    try {
      await onConfirm(requireStore ? storeId : undefined);
      onOpenChange(false);
      setStoreId("");
    } finally {
      setLoading(false);
    }
  }

  return (
    <AlertDialog open={open} onOpenChange={(next) => !loading && onOpenChange(next)}>
      <AlertDialogContent>
        <AlertDialogHeader>
          <AlertDialogTitle>{tAdmins("requestApproveTitle")}</AlertDialogTitle>
          <AlertDialogDescription>{request?.username}</AlertDialogDescription>
        </AlertDialogHeader>
        <div className="space-y-4">
          {requireStore && (
            <div className="space-y-2">
              <Label>{tAdmins("requestApproveStoreLabel")}</Label>
              <Select value={storeId} onValueChange={(v) => v && setStoreId(v)}>
                <SelectTrigger className="h-10 w-full gap-2 px-3">
                  <span>{storeId ? stores.find((s) => s.id === storeId)?.name : tAdmins("storePlaceholder")}</span>
                </SelectTrigger>
                <SelectContent align="start" alignItemWithTrigger={false} className="p-1.5">
                  {stores.map((s) => (
                    <SelectItem key={s.id} value={s.id} className="py-2">{s.name}</SelectItem>
                  ))}
                </SelectContent>
              </Select>
            </div>
          )}
        </div>
        <AlertDialogFooter>
          <AlertDialogCancel disabled={loading}>{tCommon("cancel")}</AlertDialogCancel>
          <AlertDialogAction
            disabled={loading || (requireStore && !storeId)}
            onClick={(e) => {
              e.preventDefault();
              void confirm();
            }}
            className="bg-primary text-primary-foreground hover:bg-primary-hover"
          >
            {loading && <Loader2 aria-hidden="true" className="mr-2 size-4 animate-spin" />}
            {tAdmins("requestApprove")}
          </AlertDialogAction>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  );
}
```

- [ ] **Step 3: `reject-join-request-dialog.tsx`** — create a small destructive confirmation dialog using the same `AlertDialog` convention. Props: `{ open, onOpenChange, request, onConfirm }`. The confirm action calls `preventDefault()`, awaits `onConfirm()`, disables both actions while loading, and uses `bg-destructive text-destructive-foreground hover:bg-destructive/90`.

- [ ] **Step 4: `join-requests-panel.tsx`** — fetches `listJoinRequests()`, renders a list (reusing the row style from `store-admins-content.tsx`), with Approve/Reject. `requireStore` = `isSuperAdmin`. Renders nothing when the list is empty (keeps the admins page clean). Approve opens `ApproveJoinRequestDialog`; reject opens `RejectJoinRequestDialog` and must not submit on a single click.

```tsx
"use client";

import { Loader2 } from "lucide-react";
import { useTranslations } from "next-intl";
import { useCallback, useEffect, useState } from "react";
import { toast } from "sonner";
import { Button } from "@/components/ui/button";
import { approveJoinRequest, listJoinRequests, rejectJoinRequest } from "@/features/admin/api";
import { ApproveJoinRequestDialog } from "@/features/admin/approve-join-request-dialog";
import { RejectJoinRequestDialog } from "@/features/admin/reject-join-request-dialog";
import { translateCommonApiError, translateNetworkError } from "@/lib/api-error";
import { ApiError } from "@/types/api";
import type { JoinRequestDto } from "@/types/admin";
import type { StoreDto } from "@/types/store";

interface JoinRequestsPanelProps {
  isSuperAdmin: boolean;
  stores: StoreDto[];
}

export function JoinRequestsPanel({ isSuperAdmin, stores }: JoinRequestsPanelProps) {
  const tAdmins = useTranslations("admins");
  const tErrors = useTranslations("errors");
  const [requests, setRequests] = useState<JoinRequestDto[]>([]);
  const [actionId, setActionId] = useState<string | null>(null);
  const [approveTarget, setApproveTarget] = useState<JoinRequestDto | null>(null);
  const [rejectTarget, setRejectTarget] = useState<JoinRequestDto | null>(null);

  const fetchRequests = useCallback(async () => {
    try {
      setRequests(await listJoinRequests());
    } catch {
      // silent: panel is supplemental
    }
  }, []);

  useEffect(() => {
    void fetchRequests();
  }, [fetchRequests]);

  async function doApprove(storeId?: string) {
    if (!approveTarget) return;
    try {
      await approveJoinRequest(approveTarget.requestId, storeId);
      toast.success(tAdmins("requestApprovedToast", { username: approveTarget.username }));
      await fetchRequests();
    } catch (err) {
      toast.error(err instanceof ApiError ? translateCommonApiError(err, tErrors) : translateNetworkError(tErrors));
    }
  }

  async function doReject() {
    if (!rejectTarget) return;
    setActionId(rejectTarget.requestId);
    try {
      await rejectJoinRequest(rejectTarget.requestId);
      toast.success(tAdmins("requestRejectedToast"));
      await fetchRequests();
    } catch (err) {
      toast.error(err instanceof ApiError ? translateCommonApiError(err, tErrors) : translateNetworkError(tErrors));
    } finally {
      setActionId(null);
    }
  }

  if (requests.length === 0) return null;

  return (
    <div className="mb-6 rounded-xl border border-border bg-card p-4">
      <h2 className="mb-3 text-sm font-semibold">{tAdmins("requestsTitle")}</h2>
      <div className="space-y-2.5">
        {requests.map((req) => (
          <div key={req.requestId} className="flex items-center justify-between gap-4 rounded-lg border border-border px-4 py-3">
            <p className="truncate text-sm font-medium">{req.username}</p>
            <div className="flex shrink-0 gap-2">
              <Button size="sm" variant="outline" disabled={actionId === req.requestId} onClick={() => setRejectTarget(req)}>
                {actionId === req.requestId ? <Loader2 className="size-4 animate-spin" /> : tAdmins("requestReject")}
              </Button>
              <Button size="sm" className="bg-primary text-primary-foreground hover:bg-primary-hover"
                onClick={() => setApproveTarget(req)}>
                {tAdmins("requestApprove")}
              </Button>
            </div>
          </div>
        ))}
      </div>
      <ApproveJoinRequestDialog
        open={!!approveTarget}
        onOpenChange={(o) => !o && setApproveTarget(null)}
        request={approveTarget}
        requireStore={isSuperAdmin}
        stores={stores}
        onConfirm={doApprove}
      />
      <RejectJoinRequestDialog
        open={!!rejectTarget}
        onOpenChange={(o) => !o && setRejectTarget(null)}
        request={rejectTarget}
        onConfirm={doReject}
      />
    </div>
  );
}
```

- [ ] **Step 5: `admins/page.tsx`** — render the panel above the toolbar. Add import and place `<JoinRequestsPanel isSuperAdmin={isSuperAdmin} stores={stores} />` as the first child inside the top-level `<div>` (before `<AdminDirectoryToolbar />`).

- [ ] **Step 6: Verify** — `cd web && yarn lint`. The panel hides when empty; SUPER_ADMIN sees a store picker on approve; store-ADMIN approves directly after confirmation; reject always requires confirmation.

---

### Task 10.2: Join-code panel + organization settings page + store settings

**Files:**
- Create: `web/src/features/organization/api.ts`
- Create: `web/src/features/organization/join-code-panel.tsx`
- Create: `web/src/app/[locale]/dashboard/settings/organization/page.tsx`
- Modify: `web/src/app/[locale]/dashboard/settings/store/page.tsx`
- Modify: `web/src/components/layout/sidebar.tsx`

- [ ] **Step 1: `features/organization/api.ts`.**

```typescript
import { get, post } from "@/lib/api";
import { API_ROUTES } from "@/lib/constants";
import type { JoinCodeResponse, OrganizationDto } from "@/types/organization";

export function getMyOrg() {
  return get<OrganizationDto>(API_ROUTES.ORGS.ME);
}

export function getOrgJoinCode() {
  return get<JoinCodeResponse>(API_ROUTES.ORGS.JOIN_CODE);
}

export function rotateOrgJoinCode() {
  return post<JoinCodeResponse>(API_ROUTES.ORGS.JOIN_CODE_ROTATE, undefined);
}

export function getStoreJoinCode(storeId: string) {
  return get<JoinCodeResponse>(API_ROUTES.STORES.JOIN_CODE(storeId));
}

export function rotateStoreJoinCode(storeId: string) {
  return post<JoinCodeResponse>(API_ROUTES.STORES.JOIN_CODE_ROTATE(storeId), undefined);
}
```

- [ ] **Step 2: `join-code-panel.tsx`** — generic panel taking `fetchCode`/`rotateCode` callbacks; shows the code read-only with Copy + Rotate. Rotation is a warning confirm because it invalidates the current shared code for new requests; use the existing repo `AlertDialog` convention and a warning action button (`bg-warning text-warning-foreground hover:bg-warning/90`) rather than inventing a new dialog style.

```tsx
"use client";

import { Copy, Loader2, RefreshCw } from "lucide-react";
import { useTranslations } from "next-intl";
import { useEffect, useState } from "react";
import { toast } from "sonner";
import {
  AlertDialog, AlertDialogAction, AlertDialogCancel, AlertDialogContent,
  AlertDialogDescription, AlertDialogFooter, AlertDialogHeader, AlertDialogTitle,
} from "@/components/ui/alert-dialog";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";

interface JoinCodePanelProps {
  fetchCode: () => Promise<{ joinCode: string }>;
  rotateCode: () => Promise<{ joinCode: string }>;
}

export function JoinCodePanel({ fetchCode, rotateCode }: JoinCodePanelProps) {
  const tAdmins = useTranslations("admins");
  const tCommon = useTranslations("common");
  const [code, setCode] = useState("");
  const [loading, setLoading] = useState(true);
  const [rotating, setRotating] = useState(false);
  const [confirmOpen, setConfirmOpen] = useState(false);

  useEffect(() => {
    fetchCode().then((r) => setCode(r.joinCode)).catch(() => {}).finally(() => setLoading(false));
  }, [fetchCode]);

  async function copy() {
    await navigator.clipboard.writeText(code);
    toast.success(tAdmins("joinCodeCopied"));
  }

  async function rotate() {
    setRotating(true);
    try {
      setCode((await rotateCode()).joinCode);
    } finally {
      setRotating(false);
      setConfirmOpen(false);
    }
  }

  return (
    <div className="rounded-xl border border-border bg-card p-4">
      <h2 className="text-sm font-semibold">{tAdmins("joinCodeTitle")}</h2>
      <p className="mt-1 mb-3 text-sm text-muted-foreground">{tAdmins("joinCodeDesc")}</p>
      <div className="flex gap-2">
        <Input readOnly value={loading ? "…" : code} className="font-mono" />
        <Button type="button" variant="outline" size="icon" onClick={copy} aria-label={tAdmins("joinCodeCopy")}>
          <Copy className="size-4" />
        </Button>
        <Button type="button" variant="outline" size="icon" onClick={() => setConfirmOpen(true)} aria-label={tAdmins("joinCodeRotate")}>
          <RefreshCw className="size-4" />
        </Button>
      </div>
      <AlertDialog open={confirmOpen} onOpenChange={setConfirmOpen}>
        <AlertDialogContent>
          <AlertDialogHeader>
            <AlertDialogTitle>{tAdmins("joinCodeRotate")}</AlertDialogTitle>
            <AlertDialogDescription>{tAdmins("joinCodeRotateConfirm")}</AlertDialogDescription>
          </AlertDialogHeader>
          <AlertDialogFooter>
            <AlertDialogCancel>{tCommon("cancel")}</AlertDialogCancel>
            <AlertDialogAction onClick={(e) => { e.preventDefault(); void rotate(); }}
              className="bg-warning text-warning-foreground hover:bg-warning/90">
              {rotating && <Loader2 className="mr-2 size-4 animate-spin" />}
              {tAdmins("joinCodeRotate")}
            </AlertDialogAction>
          </AlertDialogFooter>
        </AlertDialogContent>
      </AlertDialog>
    </div>
  );
}
```

- [ ] **Step 3: `settings/organization/page.tsx`** — SUPER_ADMIN-only; shows org name + the org join-code panel.

```tsx
"use client";

import { useTranslations } from "next-intl";
import { getOrgJoinCode, rotateOrgJoinCode } from "@/features/organization/api";
import { JoinCodePanel } from "@/features/organization/join-code-panel";

export default function OrganizationSettingsPage() {
  const tSettings = useTranslations("settings");
  return (
    <div className="space-y-6">
      <h1 className="text-lg font-semibold">{tSettings("organizationTitle")}</h1>
      <JoinCodePanel fetchCode={getOrgJoinCode} rotateCode={rotateOrgJoinCode} />
    </div>
  );
}
```

- [ ] **Step 4: `settings/store/page.tsx`** — show the store join-code panel only for an **independent** store. ⚠️ An ADMIN's `admin.orgId` is **always null** (an ADMIN's org is derived via its store), so it cannot distinguish independent from org-owned — you must read the **store's** `orgId`. This page currently only reads `storeId` from `useAuthStore()` and does not fetch the store, so add a fetch:

```tsx
import { getStore } from "@/features/store/api"; // add export below if missing
import { getStoreJoinCode, rotateStoreJoinCode } from "@/features/organization/api";
import { JoinCodePanel } from "@/features/organization/join-code-panel";
// ...
const [storeOrgId, setStoreOrgId] = useState<string | null | undefined>(undefined);
useEffect(() => {
  if (!storeId) return;
  getStore(storeId).then((s) => setStoreOrgId(s.orgId)).catch(() => setStoreOrgId(undefined));
}, [storeId]);
// ...render, only when independent:
{storeId && storeOrgId === null && (
  <JoinCodePanel
    fetchCode={() => getStoreJoinCode(storeId)}
    rotateCode={() => rotateStoreJoinCode(storeId)}
  />
)}
```
If `getStore` is not already exported from `web/src/features/store/api.ts`, add it: `export function getStore(id: string) { return get<StoreDto>(API_ROUTES.STORES.BY_ID(id)); }` (import `get`, `API_ROUTES`, and `StoreDto`). `StoreDto` now carries `orgId` (Phase 2 / Task 8.1).

- [ ] **Step 5: `settings/layout.tsx`** — the settings sub-nav (not the main sidebar) holds these tabs. It already conditionally includes store/service-types/slugs via `...(!isSuperAdmin ? [...] : [])`. Add the inverse for the Organization tab (SUPER_ADMIN only). Import `Building2` from `lucide-react` and append to `settingsTabs`:

```tsx
    ...(isSuperAdmin
      ? [
          {
            href: "/dashboard/settings/organization",
            label: "organizationTab" as const,
            icon: Building2,
          },
        ]
      : []),
```
Match the existing `{ href, label: "...Tab" as const, icon }` object shape used by the current tabs.

- [ ] **Step 6: Verify** — `cd web && yarn lint`. SUPER_ADMIN sees org join code in settings; independent-store ADMIN sees store join code; org-owned-store ADMIN sees neither.

- [ ] **Step 7: `docs/CHANGELOGS.md`** — add a "Phase 10 — approval UI + join-code management" entry.

---

### Task 10.3: Restrict the create-admin dialog to ADMIN-only

Backend `createAdmin` now rejects `ROLE_SUPER_ADMIN` (Task 4.5 Step 3) — SUPER_ADMINs are created only via self-registration `CREATE_ORG`. The existing dialog still offers a SUPER_ADMIN role option, which would now 400.

**Files:**
- Modify: `web/src/features/admin/create-admin-dialog.tsx`

- [ ] **Step 1: Remove the role picker.** Delete the `role` state (`const [role, setRole] = useState<AdminRole>(ROLES.ADMIN);`), the entire role `Select` block (the `<div>` containing `tAdmins("roleLabel")`), and the `isSuperAdminRole` constant. Remove now-unused imports (`Select*` components if used nowhere else in the file, and `AdminRole` type if unused).

- [ ] **Step 2: Always create an ADMIN.** In `handleSubmit`, change the `createAdmin({...})` call to:

```tsx
      await createAdmin({
        username,
        password,
        role: ROLES.ADMIN,
        storeId: storeId || null,
      });
```

- [ ] **Step 3: Always show the optional store field + admin note.** Remove the `{!isSuperAdminRole && (...)}` wrapper so the store `Select` always renders, and replace the note block with the non-super-admin branch only:

```tsx
          <div className="rounded-lg border border-border bg-muted p-3 text-sm text-muted-foreground">
            {storeId ? tAdmins("noteAdmin") : tAdmins("noteAdminPending")}
          </div>
```
(`tAdmins("noteSuperAdmin")` / `tAdmins("roleSuperAdmin")` keys stay in the message files — they're still used by `admin-directory-table` role badges.)

- [ ] **Step 4: Verify** — `cd web && yarn lint`. The dialog creates only ADMINs; the store dropdown shows the SUPER_ADMIN's org stores (auto-scoped by the org-aware `listStores`).

- [ ] **Step 5: `docs/CHANGELOGS.md`** — extend the "Phase 10" entry to note the create-admin dialog is now ADMIN-only.

---

## Final self-review (run after all phases)

- [ ] **Spec coverage:** organization tenancy (P1–P2), org-aware authz (P3), org-scoped reads/writes (P4), join codes (P5), Redis join requests + registration + login feedback (P6), approval (P7), frontend register/approval/join-code (P8–P10). Migration + schema both delivered (P1).
- [ ] **Device guarantee:** confirm an ADMIN still manages their own store's devices end-to-end (Task 4.3 keeps the ADMIN path unchanged; `requireStoreAccess` ADMIN branch is `principal.storeId == storeId`).
- [ ] **Type consistency:** `AdminPrincipal(orgId, storeId, isVerified)` constructor order is consistent across `Admin.toPrincipal` and `JWTToPrincipal`; `listDevices(kind, storeId, orgId)` arity matches all callers; `createStore(request, orgId)` arity matches both callers (StoreController + RegistrationService).
- [ ] **No placeholders:** every code step either has complete code or points to an existing repo pattern with the exact files/components to mirror.
- [ ] **GitNexus:** Task 3.1 runs impact before the StoreAccessService refactor; run `gitnexus_detect_changes()` before any commit.

## Audit log (amendments applied to this plan)

This plan was audited for spec adherence, logical errors, flow inconsistency, UI-rule compliance, and codebase-integration risks. Fixes applied inline:

1. **Integration:** `DeviceDispatchService.kt:93` is a second caller of `deviceQueryService.listDevices(...)` — the new `orgId` arity would break it; Task 4.3 Step 1b updates it (`orgId = null`).
2. **Integration:** the settings sub-nav lives in `settings/layout.tsx` (not `sidebar.tsx`); Task 10.2 Step 5 edits the correct file, and the Organization label moved to the `settings` i18n namespace (Amendment to Task 8.2 Step 4 / Task 10.2 Step 3).
3. **Logical error:** an ADMIN's `admin.orgId` is always null, so it cannot detect an independent store; Task 10.2 Step 4 now reads the **store's** `orgId` via `getStore(storeId)`.
4. **Flow inconsistency:** backend `createAdmin` now rejects `ROLE_SUPER_ADMIN`, but the create-admin dialog still offered it → new Task 10.3 restricts the dialog to ADMIN-only.
5. **Logical error:** dangling "Task 4.6" reference replaced with a concrete `deleteStore` org-guard step (Task 4.1 Step 5).
6. **Spec fidelity:** join-request approval now records `verifiedBy`/`verifiedAt` (Task 6.1/7.1).
7. **Correctness:** the `admin`-management methods (`createAdmin`/`verifyAdmin`/`updateAdminStore`/`deleteAdmin`) now take an explicit `actorOrgId` with exact `AdminController` call sites (Task 4.5 Step 4) — closes a cross-org tampering hole.
8. **Lint:** removed a hook-call-in-JSX from `JoinCodePanel` (hoisted `tCommon`).
9. **No-placeholder:** the create-store and join registration forms are now written in full (Task 9.2 Steps 2–3).
10. **Authorization:** device helper methods that previously short-circuited SUPER_ADMIN (`DeviceLifecycleService`, `RfCodeService`, `DeviceApprovalService`, `DeviceQueryService`) are explicitly routed through `StoreAccessService` when a device has a store.
11. **Authorization:** enrollment-token issue/list/revoke now has a dedicated org-scope task; SUPER_ADMIN can only manage tokens for stores in their org, and new storeless enrollment tokens are rejected.
12. **Spec fidelity:** `verifyAdmin` now supports both SUPER_ADMIN-of-org and verified independent-store ADMIN approvals; last-SUPER_ADMIN deletion is per org, not global.
13. **Race safety:** pending username reservation uses Redis `setIfAbsent`, approval uses a short Redis lock, and approval username conflicts leave the request intact for retry/reject/expiry.
14. **HTTP contract:** pending JOIN registration returns `202 Accepted`; active CREATE_ORG/CREATE_STORE registrations return `201 Created`.
15. **UI convention:** join-request approve/reject and join-code rotation use the repo's `AlertDialog` confirmation convention; reject is no longer one-click.
16. **Spec compliance:** role display labels are included in i18n work as "Organization owner" / "Store manager" instead of being deferred.

### Verified-correct (no change needed)
- `Button` is Base UI-backed and **supports `render`**, so `<Button render={<Link/>}>` is valid.
- Only two `AdminPrincipal(...)` constructors exist (`Admin.toPrincipal`, `JWTToPrincipal`) — both updated.
- `register` auto-inherits the `auth` rate-limit tier via the `/api/auth/` prefix (no filter edit).
- All `@Query` writes avoided (entity `.save()` only) — no `@Modifying` needed (Context7-confirmed requirement).
