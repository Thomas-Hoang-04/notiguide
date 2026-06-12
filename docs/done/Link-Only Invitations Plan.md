# Link-Only Invitations (Join-Code Purge) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove join codes (`o_…` / `s_…`) from the entire product — backend, database schema, web UI, and i18n — leaving expiring invite links as the only invitation mechanism.

**Architecture:** Single-pass purge per `docs/spec/Link-Only Invitations Spec.md`. The JOIN registration mode collapses to one credential (`inviteToken`); four join-code endpoints, two DB columns + unique indexes, the `JoinCodePanel`, and all join-code i18n die. `JoinCodeGenerator` is renamed to `InviteTokenGenerator` with a single-purpose API. The pending-join-request approval flow and all invite-link mechanics are untouched.

**Tech Stack:** Kotlin 2.3 / Spring Boot 3.5 WebFlux (coroutines, R2DBC, Redis) on the backend; Next.js 16 / React 19 / TypeScript / next-intl / vitest on the web.

---

**Conventions for every task:**

- Backend paths are relative to `backend/` (its own git repo). Web paths are relative to `web/` (its own git repo). Docs paths are workspace-relative.
- Per project rules: do **not** create git commits (the user commits); log every change — including anything skipped — in `docs/CHANGELOGS.md` (Task 10); per CLAUDE.md run `gitnexus_impact({target: "<symbol>", direction: "upstream"})` before editing each named symbol and surface HIGH/CRITICAL findings before proceeding (the spec already maps the blast radius — impact results should match § "File change summary").
- Task ordering is deliberate: each task leaves both repos compiling and type-clean. On the backend, the generator rename (Task 5) must come **last** because Tasks 1–4 remove the other `JoinCodeGenerator` call sites first. On the web, the wizard rework (Task 7) must precede the form rework (Task 8) — see the audit log at the bottom.
- The Vietnamese copy in Task 7 was drafted in the spec **subject to user approval at plan review**. Do not silently reword it; if the user amended any VI string during review, use the amended version.

---

### Task 1: Backend — link-only `join()` in RegistrationService

**Files:**
- Modify: `src/test/kotlin/com/thomas/notiguide/domain/admin/service/RegistrationServiceTest.kt`
- Modify: `src/main/kotlin/com/thomas/notiguide/domain/admin/request/RegisterRequest.kt`
- Modify: `src/main/kotlin/com/thomas/notiguide/domain/admin/service/RegistrationService.kt`

- [ ] **Step 1: Rework the test file first (red)**

In `RegistrationServiceTest.kt`:

1. Change the `joinRequest` helper (line ~54) to drop the `joinCode` parameter:

```kotlin
    private fun joinRequest(inviteToken: String? = null) = RegisterRequest(
        mode = RegisterMode.JOIN,
        username = "newjoiner",
        password = "Sup3rSecret!1",
        inviteToken = inviteToken
    )
```

2. **Delete** these two tests entirely:
   - `` `join with both a code and an invite token is rejected with 400` `` (line ~119)
   - `` `join via a join code records nothing in the usage trail` `` (line ~178, the last test — it mocks `organizationRepository.findByJoinCode`)

3. **Replace** the test `` `join with neither a code nor an invite token is rejected with 400` `` (line ~129) with a version that pins the new message:

```kotlin
    @Test
    fun `join without an invite token is rejected with 400`() = runTest {
        val ex = runCatching { service.register(joinRequest()) }.exceptionOrNull()

        assertThat(ex).isInstanceOf(HttpException::class.java)
        assertThat((ex as HttpException).status).isEqualTo(HttpStatus.BAD_REQUEST)
        assertThat(ex.message).isEqualTo("An invite link is required")
    }
```

Leave the `Organization(id = orgId, name = "Acme", joinCode = "o_x")` constructions (lines ~72 and ~169) **as they are** — the entity still has the parameter until Task 4, which removes it and updates these fixtures.

- [ ] **Step 2: Run the test class to verify it fails**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.admin.service.RegistrationServiceTest"`
Expected: FAIL — `join without an invite token is rejected with 400` fails on the message assertion (actual message is the old `"Provide a join code or an invite link"`). All other tests pass.

- [ ] **Step 3: Remove `joinCode` from RegisterRequest**

In `RegisterRequest.kt`, delete this field (lines ~38–39); `inviteToken` stays as-is:

```kotlin
    @field:Size(max = 64)
    val joinCode: String? = null,
```

- [ ] **Step 4: Rewrite `join()`**

In `RegistrationService.kt`, replace the whole `join(...)` method (lines ~97–125) with:

```kotlin
    private suspend fun join(username: String, passwordHash: String, request: RegisterRequest): RegisterResponse {
        val inviteToken = request.inviteToken?.trim()?.takeIf { it.isNotBlank() }
            ?: throw HttpException(HttpStatus.BAD_REQUEST, "An invite link is required")
        return joinViaInviteToken(username, passwordHash, inviteToken)
    }
```

This removes the `code` local, the both-provided guard, the `when` over code prefixes, and all four `"Invalid join code"` 400s. `joinViaInviteToken` and `recordUseSafely` are untouched.

Do **not** yet remove `generateUniqueOrgCode()` or the `JoinCodeGenerator` import — `createOrg` still mints the org code until the entity column drops in Task 4. (The `StoreRepository` import stays too: `joinViaInviteToken` uses `storeRepository.findById`.)

- [ ] **Step 5: Run the test class to verify it passes**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.admin.service.RegistrationServiceTest"`
Expected: PASS (all remaining tests green).

---

### Task 2: Backend — remove the four join-code endpoints and their service methods

**Files:**
- Modify: `src/main/kotlin/com/thomas/notiguide/domain/organization/controller/OrganizationController.kt`
- Modify: `src/main/kotlin/com/thomas/notiguide/domain/organization/service/OrganizationService.kt`
- Delete: `src/main/kotlin/com/thomas/notiguide/domain/organization/response/JoinCodeResponse.kt`
- Modify: `src/main/kotlin/com/thomas/notiguide/domain/store/controller/StoreController.kt`
- Modify: `src/main/kotlin/com/thomas/notiguide/domain/store/service/StoreService.kt`
- Modify: `src/main/kotlin/com/thomas/notiguide/domain/store/repository/StoreRepository.kt`

No new tests: `OrganizationControllerTest.kt` and `StoreControllerTest.kt` contain no join-code endpoint tests and never stub the deleted service methods (verified — `OrganizationControllerTest` does not reference `OrganizationService` at all; `StoreControllerTest` stubs only `storeService.getStore`), so this task is deletion + compile + existing suites. No `SecurityConfig` change is needed: no matcher references these routes specifically (spec § 7).

- [ ] **Step 1: OrganizationController — drop 2 endpoints, reword the 403**

In `OrganizationController.kt`:

1. Delete the `getJoinCode` endpoint (lines ~36–40) and the `rotateJoinCode` endpoint (lines ~42–46) — both `/me/join-code` routes.
2. Delete the import `com.thomas.notiguide.domain.organization.response.JoinCodeResponse` (line 10).
3. In `requireOrgOwner` (line ~76), reword the 403 message:

```kotlin
    private fun requireOrgOwner(principal: AdminPrincipal) =
        principal.takeIf { it.authorities.any { a -> a.authority == AdminRole.ROLE_SUPER_ADMIN.name } }
            ?.orgId
            ?: throw ForbiddenException("Only organization owners can manage organization invites")
```

The `/me`, `/me/invite-link`, and `/me/invite-link/rotate` endpoints, `resolveOrgId`, and `composeLinkResponse` are untouched.

- [ ] **Step 2: OrganizationService — keep only `getOrganizationPublic`**

Replace the entire body of `OrganizationService.kt` with (deletes `getOrganization`, `rotateJoinCode`, `generateUniqueJoinCode`, and the `JoinCodeGenerator` + `OrganizationDto` imports — `getOrganization`'s only caller was the deleted `getJoinCode` endpoint):

```kotlin
package com.thomas.notiguide.domain.organization.service

import com.thomas.notiguide.core.exception.NotFoundException
import com.thomas.notiguide.domain.organization.dto.OrganizationPublicDto
import com.thomas.notiguide.domain.organization.repository.OrganizationRepository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.util.UUID

@Service
class OrganizationService(
    private val organizationRepository: OrganizationRepository
) {
    @Transactional(readOnly = true)
    suspend fun getOrganizationPublic(id: UUID): OrganizationPublicDto =
        organizationRepository.findById(id)?.toPublicDto()
            ?: throw NotFoundException("Organization", "id", id.toString())
}
```

- [ ] **Step 3: Delete the JoinCodeResponse DTO**

Run: `cd backend && git rm src/main/kotlin/com/thomas/notiguide/domain/organization/response/JoinCodeResponse.kt`
(Its only consumers were the four endpoints removed in this task. `InviteLinkResponse.kt` in the same package stays.)

- [ ] **Step 4: StoreController — drop 2 endpoints, reword the 409 and its KDoc**

In `StoreController.kt`:

1. Delete the `getStoreJoinCode` endpoint (lines ~124–131) and the `rotateStoreJoinCode` endpoint (lines ~133–140) — both `/{id}/join-code` routes.
2. Delete the import `com.thomas.notiguide.domain.organization.response.JoinCodeResponse` (line 9).
3. Replace `requireIndependentStore` and its KDoc (lines ~166–170) — the old KDoc referenced the now-deleted join-code endpoints:

```kotlin
    /** Org-owned stores are joined via the organization — invites for them are managed at the org level. */
    private suspend fun requireIndependentStore(id: UUID) {
        if (storeService.getStore(id).orgId != null)
            throw ConflictException("Org-owned stores are joined via the organization")
    }
```

The `/{id}/invite-link*` endpoints and `composeLinkResponse` are untouched.

- [ ] **Step 5: StoreService — drop the three join-code methods and create-time minting**

In `StoreService.kt`:

1. Delete `rotateStoreJoinCode(storeId)` (lines ~199–208), `getStoreJoinCode(storeId)` including its lazy-mint branch (lines ~210–218), and `generateUniqueStoreJoinCode()` (lines ~220–226). (The two `"Org-owned stores are joined via the organization code"` `ConflictException` strings die with their methods.)
2. Delete the import `com.thomas.notiguide.core.tenant.JoinCodeGenerator` (line 6).
3. In `createStore` (line ~104), delete the line `val joinCode = if (orgId == null) generateUniqueStoreJoinCode() else null` and the constructor argument `joinCode = joinCode,` (line ~110). The `Store(...)` construction becomes:

```kotlin
        val store = Store(
            name = request.name,
            address = request.address?.takeIf { it.isNotBlank() },
            orgId = orgId,
            allowJumpCall = request.allowJumpCall,
            allowNoShow = request.allowNoShow
        )
```

(This compiles now because `Store.joinCode` has a default of `null`; the property itself is removed in Task 4.)

- [ ] **Step 6: StoreRepository — drop `findByJoinCode`**

In `StoreRepository.kt`, delete the last method (lines ~31–32):

```kotlin
    @Query("SELECT * FROM store WHERE join_code = :joinCode")
    suspend fun findByJoinCode(joinCode: String): Store?
```

(Its two callers — `StoreService.generateUniqueStoreJoinCode` and the old `RegistrationService` code branch — are gone after Step 5 and Task 1.)

- [ ] **Step 7: Compile and run the touched suites**

Run: `cd backend && ./gradlew compileKotlin compileTestKotlin --console=plain`
Expected: BUILD SUCCESSFUL.

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.organization.controller.OrganizationControllerTest" --tests "com.thomas.notiguide.domain.store.controller.StoreControllerTest" --tests "com.thomas.notiguide.domain.store.controller.StoreSlugControllerTest"`
Expected: PASS.

---

### Task 3: Backend — drop `join_code` from schema.sql + document the manual migration

**Files:**
- Modify: `src/main/resources/db/schema.sql`

`schema.sql` is init-only (mounted into the Postgres init dir; no migration framework). Existing deployments need the manual `ALTER`s, which are recorded in CHANGELOGS in Task 10.

- [ ] **Step 1: Remove the organization column + index**

In the `CREATE TABLE organization` block (line ~23), delete the line:

```sql
    join_code   TEXT NOT NULL,
```

and delete the index line (~29):

```sql
CREATE UNIQUE INDEX uq_organization_join_code ON organization(join_code);
```

- [ ] **Step 2: Remove the store column + partial index**

In the `CREATE TABLE store` block (line ~99), delete the line:

```sql
    join_code TEXT,
```

and delete the partial index line (~105):

```sql
CREATE UNIQUE INDEX uq_store_join_code ON store(join_code) WHERE join_code IS NOT NULL;
```

- [ ] **Step 3: Verify no `join_code` remains in the schema**

Run: `grep -n "join_code" backend/src/main/resources/db/schema.sql`
Expected: no output.

The documented manual migration for existing deployments (Postgres drops dependent indexes with the column — goes into CHANGELOGS verbatim in Task 10):

```sql
ALTER TABLE organization DROP COLUMN join_code;
ALTER TABLE store DROP COLUMN join_code;
```

---

### Task 4: Backend — remove `joinCode` from the entities, delete `OrganizationDto`, prune repositories, update test fixtures

**Files:**
- Modify: `src/main/kotlin/com/thomas/notiguide/domain/organization/entity/Organization.kt`
- Delete: `src/main/kotlin/com/thomas/notiguide/domain/organization/dto/OrganizationDto.kt`
- Modify: `src/main/kotlin/com/thomas/notiguide/domain/organization/repository/OrganizationRepository.kt`
- Modify: `src/main/kotlin/com/thomas/notiguide/domain/store/entity/Store.kt`
- Modify: `src/main/kotlin/com/thomas/notiguide/domain/admin/service/RegistrationService.kt`
- Modify (fixtures): `src/test/kotlin/com/thomas/notiguide/domain/admin/service/RegistrationServiceTest.kt`, `src/test/kotlin/com/thomas/notiguide/domain/admin/service/InviteLinkServiceTest.kt`

These changes are interdependent (entity ⇄ service ⇄ fixtures), so they land as one task with the compiler as the failing test.

- [ ] **Step 1: Organization entity — drop `joinCode` and `toDto()`**

Replace the full content of `Organization.kt` with (removes the `join_code` column property, the `toDto()` mapping, and the `OrganizationDto` import; `toPublicDto()` — the only mapping with a live consumer — stays):

```kotlin
package com.thomas.notiguide.domain.organization.entity

import com.thomas.notiguide.domain.organization.dto.OrganizationPublicDto
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

    @Column("created_by")
    val createdBy: UUID? = null,

    @CreatedDate
    @Column("created_at")
    val createdAt: OffsetDateTime? = null,

    @LastModifiedDate
    @Column("updated_at")
    val updatedAt: OffsetDateTime? = null
) {
    fun toPublicDto(): OrganizationPublicDto = OrganizationPublicDto(
        id = id!!,
        name = name,
        createdAt = createdAt
    )
}
```

- [ ] **Step 2: Delete OrganizationDto**

Run: `cd backend && git rm src/main/kotlin/com/thomas/notiguide/domain/organization/dto/OrganizationDto.kt`
(Orphaned: its producers — `toDto()`, `getOrganization`, `rotateJoinCode` — are all gone. The live `getMyOrg` endpoint returns `OrganizationPublicDto`, untouched.)

- [ ] **Step 3: OrganizationRepository — drop both join-code queries**

Replace the full content of `OrganizationRepository.kt` with:

```kotlin
package com.thomas.notiguide.domain.organization.repository

import com.thomas.notiguide.domain.organization.entity.Organization
import org.springframework.data.repository.kotlin.CoroutineCrudRepository
import org.springframework.stereotype.Repository
import java.util.UUID

@Repository
interface OrganizationRepository : CoroutineCrudRepository<Organization, UUID>
```

(`findByJoinCode`'s caller died in Task 1; `existsByJoinCode`'s last caller, `generateUniqueOrgCode`, dies in Step 5 below.)

- [ ] **Step 4: Store entity — drop `joinCode`**

In `Store.kt`, delete the property (lines ~24–25):

```kotlin
    @Column("join_code")
    val joinCode: String? = null,
```

`Store.toDto()` never exposed `joinCode`, so no DTO change follows.

- [ ] **Step 5: RegistrationService — drop org-code minting**

In `RegistrationService.kt`:

1. In `createOrg` (line ~60), change the org construction to:

```kotlin
        val org = organizationRepository.save(
            Organization(name = orgName)
        )
```

2. Delete the whole `generateUniqueOrgCode()` method (lines ~171–177).
3. Delete the import `com.thomas.notiguide.core.tenant.JoinCodeGenerator` (line 5) — now unused in this file.

- [ ] **Step 6: Update the test fixtures that pass `joinCode` to `Organization(...)`**

In `RegistrationServiceTest.kt`, both remaining constructions (in `` `join via a valid org invite token files a pending request and records the use` `` and `` `a recordUse failure does not fail the registration` ``) become:

```kotlin
            Organization(id = orgId, name = "Acme")
```

In `InviteLinkServiceTest.kt` (in `` `resolveForDisplay returns the org name for an org target` ``, line ~152):

```kotlin
            Organization(id = orgId, name = "Acme Group")
```

- [ ] **Step 7: Compile and run the touched suites**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.admin.service.RegistrationServiceTest" --tests "com.thomas.notiguide.domain.admin.service.InviteLinkServiceTest" --tests "com.thomas.notiguide.domain.organization.controller.OrganizationControllerTest"`
Expected: PASS.

---

### Task 5: Backend — rename `JoinCodeGenerator` → `InviteTokenGenerator`

**Files:**
- Rename: `src/main/kotlin/com/thomas/notiguide/core/tenant/JoinCodeGenerator.kt` → `src/main/kotlin/com/thomas/notiguide/core/tenant/InviteTokenGenerator.kt`
- Modify: `src/main/kotlin/com/thomas/notiguide/domain/admin/service/InviteLinkService.kt`
- Rename: `src/test/kotlin/com/thomas/notiguide/core/tenant/JoinCodeGeneratorTest.kt` → `src/test/kotlin/com/thomas/notiguide/core/tenant/InviteTokenGeneratorTest.kt`

- [ ] **Step 1: Verify `InviteLinkService` is the sole remaining caller**

Run: `grep -rn "JoinCodeGenerator" backend/src/`
Expected: matches only in `core/tenant/JoinCodeGenerator.kt`, `domain/admin/service/InviteLinkService.kt`, and `core/tenant/JoinCodeGeneratorTest.kt`. **If anything else matches, stop — a Task 1–4 step was missed; fix that first.**

- [ ] **Step 2: Rename and rewrite the generator**

Run: `cd backend && git mv src/main/kotlin/com/thomas/notiguide/core/tenant/JoinCodeGenerator.kt src/main/kotlin/com/thomas/notiguide/core/tenant/InviteTokenGenerator.kt`

Replace its full content with (removes `ORG_PREFIX`, `STORE_PREFIX`, `INVITE_PREFIX` — folded into `PREFIX` — the prefix parameter, the single-arg legacy overload, and `DEFAULT_BYTE_COUNT`):

```kotlin
package com.thomas.notiguide.core.tenant

import java.security.SecureRandom
import java.util.Base64

object InviteTokenGenerator {
    const val PREFIX = "i_"

    private val secureRandom = SecureRandom()
    private val encoder = Base64.getUrlEncoder().withoutPadding()

    /** e.g. "i_…" — byteCount random bytes, base64url without padding, after the prefix. */
    fun generate(byteCount: Int): String {
        val bytes = ByteArray(byteCount)
        secureRandom.nextBytes(bytes)
        return PREFIX + encoder.encodeToString(bytes)
    }
}
```

- [ ] **Step 3: Update the two call sites in InviteLinkService**

In `InviteLinkService.kt`:

1. Change the import (line 7): `com.thomas.notiguide.core.tenant.JoinCodeGenerator` → `com.thomas.notiguide.core.tenant.InviteTokenGenerator`.
2. In `regenerate` (line ~86): `JoinCodeGenerator.generate(JoinCodeGenerator.INVITE_PREFIX, TOKEN_BYTE_COUNT)` → `InviteTokenGenerator.generate(TOKEN_BYTE_COUNT)`.
3. In `resolve` (line ~119): `token.startsWith(JoinCodeGenerator.INVITE_PREFIX)` → `token.startsWith(InviteTokenGenerator.PREFIX)`.

- [ ] **Step 4: Rename and rewrite the generator test**

Run: `cd backend && git mv src/test/kotlin/com/thomas/notiguide/core/tenant/JoinCodeGeneratorTest.kt src/test/kotlin/com/thomas/notiguide/core/tenant/InviteTokenGeneratorTest.kt`

Replace its full content with. Disposition of the old eight tests: org-prefix, store-prefix, and legacy-9-byte tests **deleted** (symbols gone); the uniqueness test and non-invite url-safe test **ported** to `generate(byteCount)` (the url-safe one folds into the invite url-safe test); the three invite-token tests **kept**, retargeted:

```kotlin
package com.thomas.notiguide.core.tenant

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test

class InviteTokenGeneratorTest {
    @Test
    fun `token carries the invite prefix`() {
        assertThat(InviteTokenGenerator.generate(16)).startsWith("i_")
    }

    @Test
    fun `a 16-byte token is 24 chars total`() {
        // 16 bytes → 22 base64url chars (no padding) + the 2-char prefix
        assertThat(InviteTokenGenerator.generate(16)).hasSize(24)
    }

    @Test
    fun `tokens are distinct across calls`() {
        val a = InviteTokenGenerator.generate(16)
        val b = InviteTokenGenerator.generate(16)
        assertThat(a).isNotEqualTo(b)
    }

    @Test
    fun `token suffix is url-safe`() {
        val suffix = InviteTokenGenerator.generate(16).removePrefix("i_")
        assertThat(suffix).matches("[A-Za-z0-9_-]+")
    }
}
```

- [ ] **Step 5: Full backend suite — the gate**

Run: `cd backend && ./gradlew test`
Expected: BUILD SUCCESSFUL, all tests green.

- [ ] **Step 6: Final backend grep for survivors**

Run: `grep -rni "joincode\|join_code\|join-code\|JoinCodeGenerator" backend/src/`
Expected: no output. (If anything surfaces, purge it and re-run Step 5.)

---

### Task 6: Web — purge route constants, API functions, response type, and the JoinCodePanel

**Files:**
- Modify: `src/types/organization.ts`
- Modify: `src/lib/constants.ts`
- Modify: `src/features/organization/api.ts`
- Delete: `src/features/organization/join-code-panel.tsx`
- Modify: `src/app/[locale]/dashboard/settings/organization/page.tsx`
- Modify: `src/app/[locale]/dashboard/settings/store/page.tsx`

All commands in Tasks 6–9 run from `web/`. (`types/admin.ts` is deliberately **not** in this task — `RegisterRequest.joinCode` is removed in Task 8 together with its last consumer, keeping the tree type-clean at every task boundary.)

- [ ] **Step 1: types/organization.ts — delete the JoinCodeResponse interface**

Delete (lines ~7–9):

```typescript
export interface JoinCodeResponse {
  joinCode: string;
}
```

- [ ] **Step 2: lib/constants.ts — drop the four join-code routes**

In `API_ROUTES.STORES` delete (lines ~43–44):

```typescript
    JOIN_CODE: (id: string) => `/api/stores/${id}/join-code`,
    JOIN_CODE_ROTATE: (id: string) => `/api/stores/${id}/join-code/rotate`,
```

In `API_ROUTES.ORGS` delete (lines ~107–108):

```typescript
    JOIN_CODE: "/api/orgs/me/join-code",
    JOIN_CODE_ROTATE: "/api/orgs/me/join-code/rotate",
```

The `INVITE_LINK` / `INVITE_LINK_ROTATE` entries in both objects stay.

- [ ] **Step 3: features/organization/api.ts — drop the four join-code functions**

Delete `getOrgJoinCode`, `rotateOrgJoinCode`, `getStoreJoinCode`, `rotateStoreJoinCode` and remove `JoinCodeResponse` from the type-import block. The file becomes:

```typescript
import { get, post } from "@/lib/api";
import { API_ROUTES } from "@/lib/constants";
import type { InviteLinkState, OrganizationDto } from "@/types/organization";

export function getMyOrg() {
  return get<OrganizationDto>(API_ROUTES.ORGS.ME);
}

export function getOrgInviteLink() {
  return get<InviteLinkState>(API_ROUTES.ORGS.INVITE_LINK);
}

export function rotateOrgInviteLink() {
  return post<InviteLinkState>(API_ROUTES.ORGS.INVITE_LINK_ROTATE, undefined);
}

export function getStoreInviteLink(storeId: string) {
  return get<InviteLinkState>(API_ROUTES.STORES.INVITE_LINK(storeId));
}

export function rotateStoreInviteLink(storeId: string) {
  return post<InviteLinkState>(
    API_ROUTES.STORES.INVITE_LINK_ROTATE(storeId),
    undefined,
  );
}
```

- [ ] **Step 4: Delete the panel**

Run: `git rm src/features/organization/join-code-panel.tsx`
(The recent error-toast amendments to this file die with it; the identical pattern already lives in `invite-link-panel.tsx`.)

- [ ] **Step 5: Organization settings page — InviteLinkPanel only**

Replace the full content of `src/app/[locale]/dashboard/settings/organization/page.tsx` with:

```tsx
"use client";

import {
  getOrgInviteLink,
  rotateOrgInviteLink,
} from "@/features/organization/api";
import { InviteLinkPanel } from "@/features/organization/invite-link-panel";

export default function OrganizationSettingsPage() {
  return (
    <div className="mx-auto max-w-2xl space-y-6">
      <InviteLinkPanel
        fetchLink={getOrgInviteLink}
        generateLink={rotateOrgInviteLink}
      />
    </div>
  );
}
```

- [ ] **Step 6: Store settings page — drop JoinCodePanel, collapse the fragment**

In `src/app/[locale]/dashboard/settings/store/page.tsx`:

1. In the `@/features/organization/api` import, remove `getStoreJoinCode` and `rotateStoreJoinCode` (keep `getStoreInviteLink`, `rotateStoreInviteLink`).
2. Remove the `import { JoinCodePanel } from "@/features/organization/join-code-panel";` line.
3. Replace the gated block at the bottom (lines ~187–198) with — the gate itself stays, the fragment collapses to the single panel:

```tsx
      {storeId && storeOrgId === null && (
        <InviteLinkPanel
          fetchLink={() => getStoreInviteLink(storeId)}
          generateLink={() => rotateStoreInviteLink(storeId)}
        />
      )}
```

(`dashboard/admins/page.tsx` is unaffected — it never rendered `JoinCodePanel`.)

- [ ] **Step 7: Lint**

Run: `yarn lint`
Expected: clean. (Biome will also catch any now-unused import this task should have removed.)

---

### Task 7: Web — wizard guidance states + i18n additions/reworks (EN + VI)

**Files:**
- Create: `src/features/auth/register-join-guidance.tsx`
- Modify: `src/features/auth/register-wizard.tsx`
- Modify: `src/features/auth/register-path-selector.tsx`
- Modify: `src/messages/en.json`
- Modify: `src/messages/vi.json`

This task intentionally precedes the form rework (Task 8): the new wizard always passes the `invite` prop when mounting `RegisterJoinForm`, which type-checks against both the current optional prop and Task 8's required prop. Doing it the other way around would leave the wizard passing `undefined` to a required prop between tasks.

- [ ] **Step 1: Add the reworked + new i18n keys (EN)**

In `src/messages/en.json`, `register` block — keys must change in this step because the components below consume them (structural mirror is kept by Step 2):

| Key | New EN value |
|---|---|
| `pathJoinTitle` (rework, line ~69) | `"Join with an invite"` |
| `pathJoinDesc` (rework, line ~70) | `"Join an existing store or organization through an invite link."` |
| `inviteInvalid` (rework, line ~91) | `"This invite link is invalid or has expired. Ask your admin for a new one."` |
| `inviteRequiredTitle` (**new**, insert after `inviteInvalid`) | `"Joining needs an invite link"` |
| `inviteRequiredDesc` (**new**, insert after `inviteRequiredTitle`) | `"Ask an admin of the store or organization to send you an invite link, then open it to continue."` |

`register.joiningTarget` is unchanged. The dead `joinCode*` keys are removed in Task 9, after nothing references them.

- [ ] **Step 2: Mirror in VI — use the user-approved drafts**

In `src/messages/vi.json`, `register` block, same positions as EN (per-spec drafts approved at plan review; if the user amended any wording during review, use the amended version):

| Key | New VI value |
|---|---|
| `pathJoinTitle` | `"Tham gia qua link mời"` |
| `pathJoinDesc` | `"Tham gia cửa hàng hoặc tổ chức có sẵn qua link mời."` |
| `inviteInvalid` | `"Link mời không hợp lệ hoặc đã hết hạn. Hãy xin link mới từ quản trị viên."` |
| `inviteRequiredTitle` (**new**) | `"Cần link mời để tham gia"` |
| `inviteRequiredDesc` (**new**) | `"Hãy xin link mời từ quản trị viên của cửa hàng hoặc tổ chức, sau đó mở link để tiếp tục."` |

- [ ] **Step 3: Create the guidance component**

Create `src/features/auth/register-join-guidance.tsx`. The guidance card is a neutral informational state, not a warning — it uses the standard `border-border bg-card` surface (the canonical severity boxes in `docs/walkthrough/Web Styles.md` are for warning/destructive/success only). The back-button row replicates the path-selector Back affordance from `RegisterFormActions` minus the submit button:

```tsx
"use client";

import { Link2 } from "lucide-react";
import { useTranslations } from "next-intl";
import { Button } from "@/components/ui/button";

interface RegisterJoinGuidanceProps {
  onBack: () => void;
}

/** "Joining needs an invite link" info state — the JOIN view without a
 * resolved token never renders credential fields (spec § 3). */
export function RegisterJoinGuidance({ onBack }: RegisterJoinGuidanceProps) {
  const t = useTranslations("register");
  return (
    <div className="space-y-4">
      <div className="flex items-start gap-2.5 rounded-xl border border-border bg-card px-3.5 py-3">
        <Link2
          aria-hidden="true"
          className="mt-0.5 size-4 shrink-0 text-primary"
        />
        <div className="space-y-1">
          <p className="text-sm font-semibold">{t("inviteRequiredTitle")}</p>
          <p className="text-sm text-muted-foreground">
            {t("inviteRequiredDesc")}
          </p>
        </div>
      </div>
      <Button
        type="button"
        variant="outline"
        className="w-full"
        onClick={onBack}
      >
        {t("back")}
      </Button>
    </div>
  );
}
```

- [ ] **Step 4: Wizard — render the JOIN view by `invite.kind`**

In `src/features/auth/register-wizard.tsx`:

1. Add the import: `import { RegisterJoinGuidance } from "@/features/auth/register-join-guidance";`
2. Replace the resolve-failure `.catch` comment (lines ~50–52) — the old one references the purged manual join-code form:

```tsx
      .catch(() => {
        // 404 (unknown/expired/deleted target) and network failures share the
        // same fallback: the invalid warning above the guidance state (spec § 6).
        if (active) setInvite({ kind: "invalid" });
      });
```

3. Replace the mid-flow comment inside `submit`'s catch (lines ~67–68) — the handler code itself stays as-is:

```tsx
      // The token died between page load and submit: flip the invite state to
      // invalid so the warning + guidance replace the form (spec § 5.3).
```

4. Replace the `view === "JOIN" && !resolvingInvite` block (lines ~111–133) with — `loading` keeps the existing spinner, `resolved` mounts the form (prop now always present; the form itself is reworked in Task 8 and accepts this in both its current and final signature), `none`/`invalid` fall through to guidance, `invalid` adds the warning box above it:

```tsx
      {view === "JOIN" && !resolvingInvite && (
        <div className="space-y-4">
          {invite.kind === "invalid" && (
            <div className="flex items-start gap-2.5 rounded-xl border border-warning/40 bg-warning/15 px-3.5 py-3 text-sm text-warning dark:border-warning/50 dark:bg-warning/20">
              <AlertTriangle
                aria-hidden="true"
                className="mt-0.5 size-4 shrink-0"
              />
              <span>{t("inviteInvalid")}</span>
            </div>
          )}
          {invite.kind === "resolved" ? (
            <RegisterJoinForm
              submitting={submitting}
              onBack={() => setView("select")}
              onSubmit={submit}
              invite={{ token: invite.token, targetName: invite.targetName }}
            />
          ) : (
            <RegisterJoinGuidance onBack={() => setView("select")} />
          )}
        </div>
      )}
```

- [ ] **Step 5: Path selector icon — key → link**

In `src/features/auth/register-path-selector.tsx`, the JOIN card's `KeyRound` icon depicts a code/key and dies with the metaphor. Change the import (line 3) to `import { Building2, Link2, Store } from "lucide-react";` and the JOIN entry's `icon: KeyRound` to `icon: Link2`.

- [ ] **Step 6: Lint + tests**

Run: `yarn lint`
Expected: clean.

Run: `yarn test`
Expected: PASS — `src/messages/parity.test.ts` confirms EN/VI key sets still mirror (both sides gained `inviteRequiredTitle`/`inviteRequiredDesc` in the same block).

---

### Task 8: Web — register-join-form: invite becomes required, code field dies

**Files:**
- Modify: `src/types/admin.ts`
- Modify: `src/features/auth/register-join-form.tsx`

- [ ] **Step 1: types/admin.ts — drop `joinCode` from RegisterRequest**

Delete the line (~88) `joinCode?: string;` so the interface reads:

```typescript
export interface RegisterRequest {
  mode: RegisterMode;
  username: string;
  password: string;
  orgName?: string;
  storeName?: string;
  storeAddress?: string;
  inviteToken?: string;
}
```

(This lands in the same task as the form rewrite below — the form's old `buildRequest` else-branch is the field's last producer.)

- [ ] **Step 2: Rewrite the form**

Replace the full content of `src/features/auth/register-join-form.tsx` with:

```tsx
"use client";

import { useTranslations } from "next-intl";
import { InlineError } from "@/components/ui/inline-error";
import {
  RegisterFormActions,
  type RegisterFormProps,
  RegisterPasswordField,
  RegisterUsernameField,
  useRegisterForm,
} from "@/features/auth/register-form-shared";

interface RegisterJoinFormProps extends RegisterFormProps {
  /** The invite resolved in this page session — joining has no other path,
   * so the wizard only mounts this form once a token resolved. */
  invite: { token: string; targetName: string };
}

export function RegisterJoinForm({
  submitting,
  onBack,
  onSubmit,
  invite,
}: RegisterJoinFormProps) {
  const t = useTranslations("register");
  const {
    username,
    setUsername,
    password,
    setPassword,
    errors,
    buildSubmitHandler,
  } = useRegisterForm(submitting, onSubmit);

  const handleSubmit = buildSubmitHandler({
    buildRequest: () => ({
      mode: "JOIN",
      username,
      password,
      inviteToken: invite.token,
    }),
  });

  return (
    <form onSubmit={handleSubmit} noValidate className="space-y-4">
      <div className="rounded-xl border border-success/40 bg-success/10 px-3.5 py-3 text-sm text-success dark:border-success/50 dark:bg-success/15">
        {t.rich("joiningTarget", {
          name: invite.targetName,
          bold: (chunks) => <span className="font-semibold">{chunks}</span>,
        })}
      </div>
      <RegisterUsernameField
        value={username}
        onChange={setUsername}
        error={errors.username}
      />
      <RegisterPasswordField
        value={password}
        onChange={setPassword}
        error={errors.password}
      />
      {errors.form && <InlineError message={errors.form} />}
      <RegisterFormActions
        submitting={submitting}
        onBack={onBack}
        submitLabel={t("submitJoin")}
      />
    </form>
  );
}
```

What changed and why (spec § 5.3):

- `invite` prop is **required**; the `invite &&` guard around the `joiningTarget` box is gone (always rendered).
- The `joinCode` state, the code `Input`/`Label` block, and the `validateExtra` joinCode check are removed — along with the now-unused `useState`, `Input`, and `Label` imports.
- The **entire `mapApiError` override** is removed: the `"join code"` branch lost its error source, and the `"invite"` branch is unreachable UI — on that same 400 the wizard's `submit` catch flips the invite state to `invalid`, which unmounts this form before any field error could render; the wizard's warning box is the user-visible message. `useRegisterForm`'s built-in 409/`errors.form` fallback still covers username conflicts and unexpected errors.

- [ ] **Step 3: Lint**

Run: `yarn lint`
Expected: clean.

---

### Task 9: Web — remove the dead i18n keys (EN + VI)

**Files:**
- Modify: `src/messages/en.json`
- Modify: `src/messages/vi.json`

- [ ] **Step 1: Verify the keys are referenced nowhere**

Run: `grep -rn "joinCodeLabel\|joinCodePlaceholder\|joinCodeRequired\|invalidJoinCode\|joinCodeTitle\|joinCodeDesc\|joinCodeCopy\|joinCodeCopied\|joinCodeRotate\|joinCodeRotateConfirm" src --include=*.ts --include=*.tsx`
Expected: matches only inside `src/messages/en.json` and `src/messages/vi.json`. (Their consumers died in Tasks 6–8.)

- [ ] **Step 2: Delete from BOTH files, same keys, same blocks**

From the `register` block (EN lines ~77–78, ~88–89; same keys in VI): `joinCodeLabel`, `joinCodePlaceholder`, `joinCodeRequired`, `invalidJoinCode`.

From the `admins` block (EN lines ~345–350; same keys in VI): `joinCodeTitle`, `joinCodeDesc`, `joinCodeCopy`, `joinCodeCopied`, `joinCodeRotate`, `joinCodeRotateConfirm`.

`admins.copyFailed` **stays** — `invite-link-panel.tsx` uses it. `register.joiningTarget`, `register.inviteInvalid` (reworked in Task 7), and all `admins.inviteLink*` keys stay.

- [ ] **Step 3: Lint + tests + final web grep**

Run: `yarn lint && yarn test`
Expected: clean lint; parity test PASS (identical key sets in both languages).

Run: `grep -rni "joincode\|join-code" src`
Expected: no output.

---

### Task 10: Docs — README reword + CHANGELOGS entry

**Files:**
- Modify: `backend/README.md`
- Modify: `docs/CHANGELOGS.md`

- [ ] **Step 1: backend/README.md — reword the core/tenant line**

Line ~134, change:

```
│   └── tenant/ database/ config/ exception/ — org/store join codes, R2DBC, app config, error handling
```

to:

```
│   └── tenant/ database/ config/ exception/ — invite tokens, R2DBC, app config, error handling
```

Then run `grep -ni "join code\|join-code\|join_code" backend/README.md` — expected: no output (mentions of "join requests" / "join queues" are unrelated and stay).

- [ ] **Step 2: docs/CHANGELOGS.md — append the sprint entry**

Append a new dated section following the file's existing format (a heading, a summary paragraph, and a file table). It must contain:

1. Every file change from Tasks 1–10 (create/modify/delete/rename), including the four removed API routes and the ten removed i18n keys.
2. The **manual migration for existing deployments**, verbatim (schema.sql is init-only; Postgres drops dependent indexes with the column):

```sql
ALTER TABLE organization DROP COLUMN join_code;
ALTER TABLE store DROP COLUMN join_code;
```

3. The **skipped/no-change notes** (the project logs skips too):
   - `OrganizationControllerTest.kt` / `StoreControllerTest.kt` — no change beyond compilation; they contain no join-code endpoint tests.
   - No `SecurityConfig` change — no matcher referenced the removed routes.
   - No client-web changes — it has no invitation surface.
   - In-flight join codes are not honored post-deploy (spec § 1 decision); stale clients submitting `joinCode` get 400 `"An invite link is required"` (unknown JSON fields are ignored by Jackson).
   - `register.pathJoinTitle` reworked ("Join with a code" → "Join with an invite") and the path-selector icon swapped (`KeyRound` → `Link2`) — purge-completeness additions found at plan audit, beyond the spec's § 5.4 list.

- [ ] **Step 3: Final whole-workspace verification**

Run, in order (lint before tests per project rule; no production builds — the audit flow covers that):

```bash
cd backend && ./gradlew test
cd ../web && yarn lint && yarn test
grep -rni "joincode\|join_code\|join-code" ../backend/src ../web/src
```

Expected: backend suite green; web lint clean; web tests (incl. i18n parity) green; the grep returns nothing.

Manual scenarios for the later audit flow (spec § 8): direct Join navigation → guidance state; valid link → unchanged happy path; dead link → warning + guidance (no code field anywhere); org/store settings pages show only the invite-link panel.

---

## Audit log (plan reviewed against spec before hand-off)

Findings raised and amended during the post-write audit:

1. **Spec gap — `register.pathJoinTitle`:** the spec's § 5.4 key list left `"Join with a code"` alive, violating its own § 2 goal ("no i18n string related to join codes survives"). Amended: Task 7 reworks it to "Join with an invite" (VI draft `"Tham gia qua link mời"`, flagged for user approval) and swaps the `KeyRound` icon for `Link2`. Logged as a spec-plus addition in CHANGELOGS (Task 10).
2. **Backend compile-order hazard:** the spec's § 4 ordering (registration → endpoints → entities → rename) would break compilation mid-stream if `RegisterRequest.joinCode` or the entity columns were dropped before their last readers. Amended: Task 1 keeps `generateUniqueOrgCode` alive until Task 4; Task 2 removes `generateUniqueStoreJoinCode` together with its only remaining caller (`createStore` minting); repository methods are removed only after their final callers (org-repo methods wait for Task 4, store-repo method goes in Task 2). Every task ends on a green compile.
3. **Web type-safety ordering (two findings):** (a) removing `RegisterRequest.joinCode` from `types/admin.ts` in the first web task would type-break the still-unreworked join form's `buildRequest` else-branch — moved into Task 8 beside its last consumer; (b) making the form's `invite` prop required before the wizard guarantees passing it would type-break the wizard — the wizard task (7) now precedes the form task (8), and the new wizard only mounts the form with the prop present, which satisfies both the old and new signatures.
4. **Fixture timing:** `Organization(...)` test fixtures keep `joinCode = "o_x"` until the constructor parameter actually disappears (Task 4 Step 6) — updating them in Task 1, as a literal reading of spec § 8 might suggest, would not compile.
5. **i18n sequencing:** new/reworked keys land in the same task as their first consumer (Task 7); dead keys are removed only after their last consumer died (Task 9), with a reference-grep guard. This keeps the app renderable and the parity test green at every task boundary.
6. **Guidance-box style:** the spec's "info box" has no canonical severity in `docs/walkthrough/Web Styles.md` (only warning/destructive/success exist). The plan uses the neutral `border-border bg-card` surface with an icon — no invented severity variant, per the style guide's "never invent a new variant" rule.
7. **Integration checks run against the codebase:** `OrganizationControllerTest` never references `OrganizationService` (deleting `getOrganization` cannot break it); `StoreControllerTest` stubs only `storeService.getStore`; `JoinCodeGenerator` is referenced by exactly the five files the plan touches; no web test file references join codes; `dashboard/admins/page.tsx` and `client-web` confirmed untouched.
