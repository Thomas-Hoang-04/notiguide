# Join Request Role Selection Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let an organization's SUPER_ADMIN choose, when approving an org join request, whether the joiner becomes a store-scoped `ROLE_ADMIN` or a peer org-wide `ROLE_SUPER_ADMIN` — without any schema change.

**Architecture:** Backend uses a typed `Approval` sealed type (Approach A) so the controller resolves authorization + role/tenancy and the service persists the right columns; the DB `chk_admin_tenancy` constraint is satisfied by construction. Frontend extracts the dialog's gating logic into pure, unit-testable helpers and adds a role picker + conditional store select + warning box. The SUPER_ADMIN option is offered only for ORG-target requests and re-validated server-side.

**Tech Stack:** Kotlin/Spring WebFlux (coroutines, R2DBC), MockK + WebTestClient tests; Next.js/React + next-intl + shadcn/ui, vitest (pure-logic tests only — no jsdom/testing-library).

**Spec:** `docs/spec/Join Request Role Selection Spec.md`

## Conventions for this plan

- **No commit steps.** Per project convention, the executor decides when to commit — this plan never instructs a commit.
- **No production build.** Do not run `./gradlew build` or `yarn build`; the dedicated audit flow owns full build/type-check.
- **Verification = lint first, then tests.** Frontend tasks run `yarn lint` before `yarn test`. Backend tasks run the relevant `./gradlew test --tests` selector.
- **Log to CHANGELOGS.** Task 7 records every change (including the `requireStore`→`allowRoleChoice` rename and the deliberately-unchanged manual "Add admin" path).

## File structure

**Backend (`backend/src/main/kotlin/com/thomas/notiguide/domain/admin/`)**
- `request/ApproveJoinRequest.kt` — gains a `role` field (default `ROLE_ADMIN`).
- `service/JoinRequestService.kt` — gains the `Approval` sealed type; `approve` takes `Approval` instead of a bare store id.
- `controller/JoinRequestController.kt` — maps the request body → `Approval`, enforces the ORG-only-SUPER_ADMIN invariant.

**Backend tests**
- `service/JoinRequestServiceTest.kt` — add `approve` tests (existing file).
- `controller/JoinRequestControllerTest.kt` — new WebTestClient controller test.

**Frontend (`web/src/features/admin/`)**
- `approve-join-request-logic.ts` — **new** pure helpers (`isStoreRequired`, `isConfirmEnabled`).
- `approve-join-request-logic.test.ts` — **new** vitest unit tests.
- `approve-join-request-dialog.tsx` — role picker, conditional store, warning box, new `onConfirm`/`allowRoleChoice`.
- `join-requests-panel.tsx` — pass `allowRoleChoice`, thread `role` through `doApprove`.
- `api.ts` — `approveJoinRequest(requestId, role, storeId?)`.
- `api.test.ts` — **new** vitest test mocking `@/lib/api`.
- `web/src/messages/en.json`, `web/src/messages/vi.json` — add `admins.requestApproveSuperAdminNote`.

**Docs**
- `docs/CHANGELOGS.md`.

---

### Task 1: Backend — typed `Approval` and role/tenancy-aware `approve`

Introduces the `role` field, the `Approval` sealed type, and the new `approve` body. The controller's single call site is updated minimally (`Approval.AsAdmin`) so the module keeps compiling and behavior is unchanged until Task 2 adds role-awareness.

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/request/ApproveJoinRequest.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/service/JoinRequestService.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/controller/JoinRequestController.kt:74`
- Test: `backend/src/test/kotlin/com/thomas/notiguide/domain/admin/service/JoinRequestServiceTest.kt`

- [ ] **Step 1: Add the `role` field to the request DTO**

Replace the entire contents of `ApproveJoinRequest.kt`:

```kotlin
package com.thomas.notiguide.domain.admin.request

import com.thomas.notiguide.domain.admin.types.AdminRole
import java.util.UUID

data class ApproveJoinRequest(
    val storeId: UUID? = null,
    val role: AdminRole = AdminRole.ROLE_ADMIN,
)
```

- [ ] **Step 2: Add the `Approval` sealed type to `JoinRequestService`**

Add this nested type inside the `JoinRequestService` class, next to the existing `enum class TargetType` (around `JoinRequestService.kt:30`):

```kotlin
    sealed interface Approval {
        data class AsAdmin(val storeId: UUID) : Approval
        data class AsSuperAdmin(val orgId: UUID) : Approval
    }
```

- [ ] **Step 3: Write the failing service tests**

Replace the entire contents of `JoinRequestServiceTest.kt`:

```kotlin
package com.thomas.notiguide.domain.admin.service

import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import com.thomas.notiguide.core.redis.RedisKeyManager
import com.thomas.notiguide.domain.admin.entity.Admin
import com.thomas.notiguide.domain.admin.repository.AdminRepository
import com.thomas.notiguide.domain.admin.types.AdminRole
import io.mockk.CapturingSlot
import io.mockk.Called
import io.mockk.coEvery
import io.mockk.every
import io.mockk.mockk
import io.mockk.slot
import io.mockk.verify
import kotlinx.coroutines.test.runTest
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.data.redis.core.ReactiveRedisTemplate
import org.springframework.data.redis.core.ReactiveValueOperations
import org.springframework.data.redis.core.ReactiveZSetOperations
import reactor.core.publisher.Mono
import java.time.Duration
import java.time.OffsetDateTime
import java.util.UUID

class JoinRequestServiceTest {
    private val redis = mockk<ReactiveRedisTemplate<String, String>>()
    private val valueOps = mockk<ReactiveValueOperations<String, String>>()
    private val zSetOps = mockk<ReactiveZSetOperations<String, String>>()
    private val adminRepository = mockk<AdminRepository>(relaxed = true)
    private val service = JoinRequestService(redis, jacksonObjectMapper(), adminRepository)

    // usernameReserved consults only the Redis username index (redis.hasKey), not the admin table.
    @Test
    fun `usernameReserved is true when the redis username index holds the key`() = runTest {
        every { redis.hasKey(any()) } returns Mono.just(true)
        assertThat(service.usernameReserved("taken")).isTrue()
        verify { adminRepository wasNot Called }
    }

    @Test
    fun `usernameReserved is false when the redis username index is empty`() = runTest {
        every { redis.hasKey(any()) } returns Mono.just(false)
        assertThat(service.usernameReserved("free")).isFalse()
        verify { adminRepository wasNot Called }
    }

    // --- approve ---

    // Stubs the full approve() Redis flow for an ORG-target request and returns the slot
    // that captures the Admin handed to adminRepository.save.
    private fun stubApproveForOrgRequest(requestId: String, username: String, orgId: UUID): CapturingSlot<Admin> {
        val payloadJson = jacksonObjectMapper().writeValueAsString(
            JoinRequestService.JoinRequestPayload(
                username = username,
                passwordHash = "hash",
                targetType = JoinRequestService.TargetType.ORG,
                targetId = orgId.toString(),
                createdAt = OffsetDateTime.now().toString(),
            ),
        )
        every { redis.opsForValue() } returns valueOps
        every { redis.opsForZSet() } returns zSetOps
        every { valueOps.setIfAbsent(RedisKeyManager.joinRequestLock(requestId), any(), Duration.ofSeconds(30)) } returns Mono.just(true)
        every { valueOps.get(RedisKeyManager.joinRequest(requestId)) } returns Mono.just(payloadJson)
        every { redis.delete(RedisKeyManager.joinRequest(requestId)) } returns Mono.just(1L)
        every { redis.delete(RedisKeyManager.joinRequestUsername(username)) } returns Mono.just(1L)
        every { redis.delete(RedisKeyManager.joinRequestLock(requestId)) } returns Mono.just(1L)
        every { zSetOps.remove(RedisKeyManager.joinRequestOrgIndex(orgId), requestId) } returns Mono.just(1L)
        coEvery { adminRepository.existsByUsername(any()) } returns false
        val saved = slot<Admin>()
        coEvery { adminRepository.save(capture(saved)) } answers { saved.captured }
        return saved
    }

    @Test
    fun `approve as super admin creates an org-wide owner with no store`() = runTest {
        val requestId = "req-super"
        val orgId = UUID.randomUUID()
        val verifierId = UUID.randomUUID()
        val saved = stubApproveForOrgRequest(requestId, "newowner", orgId)

        service.approve(requestId, JoinRequestService.Approval.AsSuperAdmin(orgId), verifierId)

        assertThat(saved.captured.role).isEqualTo(AdminRole.ROLE_SUPER_ADMIN)
        assertThat(saved.captured.orgId).isEqualTo(orgId)
        assertThat(saved.captured.storeId).isNull()
        assertThat(saved.captured.isVerified).isTrue()
        assertThat(saved.captured.verifiedBy).isEqualTo(verifierId)
    }

    @Test
    fun `approve as admin assigns the store and leaves org null`() = runTest {
        val requestId = "req-admin"
        val orgId = UUID.randomUUID()
        val storeId = UUID.randomUUID()
        val verifierId = UUID.randomUUID()
        val saved = stubApproveForOrgRequest(requestId, "newstaff", orgId)

        service.approve(requestId, JoinRequestService.Approval.AsAdmin(storeId), verifierId)

        assertThat(saved.captured.role).isEqualTo(AdminRole.ROLE_ADMIN)
        assertThat(saved.captured.storeId).isEqualTo(storeId)
        assertThat(saved.captured.orgId).isNull()
        assertThat(saved.captured.isVerified).isTrue()
        assertThat(saved.captured.verifiedBy).isEqualTo(verifierId)
    }
}
```

- [ ] **Step 4: Run the tests to verify they fail**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.admin.service.JoinRequestServiceTest"`
Expected: FAIL — compilation error, because `approve` still has the old `(String, UUID, UUID)` signature and does not accept an `Approval`.

- [ ] **Step 5: Rewrite `approve` to take an `Approval`**

In `JoinRequestService.kt`, replace the existing `approve` function (currently `JoinRequestService.kt:108-137`) with:

```kotlin
    /** Materialize an admin row from a request per the approval decision, then delete the request. */
    @Transactional
    suspend fun approve(requestId: String, approval: Approval, verifierId: UUID): Admin {
        val lockKey = RedisKeyManager.joinRequestLock(requestId)
        val locked = redis.opsForValue().setIfAbsent(lockKey, "1", Duration.ofSeconds(30)).awaitSingle()
        if (!locked) throw ConflictException("Join request is already being processed")
        val payload = get(requestId) ?: throw ConflictException("Join request not found or expired")
        try {
            if (adminRepository.existsByUsername(payload.username)) {
                // Per spec, keep the request so the owner can retry/reject or let it expire.
                throw ConflictException("Username '${payload.username}' is already taken")
            }
            val now = OffsetDateTime.now()
            val admin = adminRepository.save(
                when (approval) {
                    is Approval.AsAdmin -> Admin(
                        username = payload.username,
                        passwordHash = payload.passwordHash,
                        role = AdminRole.ROLE_ADMIN,
                        storeId = approval.storeId,
                        orgId = null,
                        isVerified = true,
                        verifiedBy = verifierId,
                        verifiedAt = now,
                    )
                    is Approval.AsSuperAdmin -> Admin(
                        username = payload.username,
                        passwordHash = payload.passwordHash,
                        role = AdminRole.ROLE_SUPER_ADMIN,
                        storeId = null,
                        orgId = approval.orgId,
                        isVerified = true,
                        verifiedBy = verifierId,
                        verifiedAt = now,
                    )
                },
            )
            deleteKeys(requestId, payload)
            return admin
        } finally {
            redis.delete(lockKey).awaitSingleOrNull()
        }
    }
```

No import changes are needed: `Approval`, `Admin`, `AdminRole`, `ConflictException`, `RedisKeyManager`, `Duration`, `OffsetDateTime`, `awaitSingle`, and `awaitSingleOrNull` are all already imported in `JoinRequestService.kt`. After this step the *main* source will not compile until Step 6 updates the one caller — that is expected; do not run tests until Step 7.

- [ ] **Step 6: Fix the controller call site so the module compiles**

In `JoinRequestController.kt:74`, change the call (the surrounding `approve` method body is replaced in Task 2; this is the only change for now):

```kotlin
        joinRequestService.approve(requestId, JoinRequestService.Approval.AsAdmin(assignStoreId), principal.id)
```

- [ ] **Step 7: Run the tests to verify they pass**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.admin.service.JoinRequestServiceTest"`
Expected: PASS — all four tests green.

---

### Task 2: Backend — controller maps role → `Approval` and enforces the ORG-only invariant

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/controller/JoinRequestController.kt`
- Test: `backend/src/test/kotlin/com/thomas/notiguide/domain/admin/controller/JoinRequestControllerTest.kt` (new)

- [ ] **Step 1: Write the failing controller test**

Create `backend/src/test/kotlin/com/thomas/notiguide/domain/admin/controller/JoinRequestControllerTest.kt`:

```kotlin
package com.thomas.notiguide.domain.admin.controller

import com.ninjasquad.springmockk.MockkBean
import com.thomas.notiguide.core.jwt.JWTAuthFilter
import com.thomas.notiguide.core.ratelimit.RateLimitFilter
import com.thomas.notiguide.domain.admin.entity.Admin
import com.thomas.notiguide.domain.admin.service.JoinRequestService
import com.thomas.notiguide.domain.admin.types.AdminRole
import com.thomas.notiguide.domain.store.entity.Store
import com.thomas.notiguide.domain.store.repository.StoreRepository
import com.thomas.notiguide.support.TestPrincipals
import com.thomas.notiguide.support.TestSecurityConfig
import io.mockk.coEvery
import io.mockk.coVerify
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.reactive.WebFluxTest
import org.springframework.context.annotation.ComponentScan.Filter
import org.springframework.context.annotation.FilterType
import org.springframework.context.annotation.Import
import org.springframework.http.MediaType
import org.springframework.security.test.web.reactive.server.SecurityMockServerConfigurers.mockAuthentication
import org.springframework.test.web.reactive.server.WebTestClient
import java.util.UUID

@WebFluxTest(
    controllers = [JoinRequestController::class],
    excludeFilters = [Filter(type = FilterType.ASSIGNABLE_TYPE, classes = [JWTAuthFilter::class, RateLimitFilter::class])],
)
@Import(TestSecurityConfig::class)
class JoinRequestControllerTest {
    @MockkBean lateinit var joinRequestService: JoinRequestService
    @MockkBean(relaxed = true) lateinit var storeRepository: StoreRepository

    @Autowired lateinit var client: WebTestClient

    private val requestId = "req-1"
    private val savedAdmin = Admin(username = "joiner", passwordHash = "hash")

    private fun orgPayload(orgId: UUID) = JoinRequestService.JoinRequestPayload(
        username = "joiner",
        passwordHash = "hash",
        targetType = JoinRequestService.TargetType.ORG,
        targetId = orgId.toString(),
        createdAt = "2026-06-13T00:00:00Z",
    )

    private fun storePayload(storeId: UUID) = JoinRequestService.JoinRequestPayload(
        username = "joiner",
        passwordHash = "hash",
        targetType = JoinRequestService.TargetType.STORE,
        targetId = storeId.toString(),
        createdAt = "2026-06-13T00:00:00Z",
    )

    @Test
    fun `org approval as super admin maps to AsSuperAdmin`() {
        val orgId = UUID.randomUUID()
        coEvery { joinRequestService.get(requestId) } returns orgPayload(orgId)
        coEvery { joinRequestService.approve(any(), any(), any()) } returns savedAdmin

        client.mutateWith(mockAuthentication(TestPrincipals.authToken(role = AdminRole.ROLE_SUPER_ADMIN, orgId = orgId)))
            .post().uri("/api/admins/requests/$requestId/approve")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue("""{"role":"ROLE_SUPER_ADMIN"}""")
            .exchange()
            .expectStatus().isNoContent

        coVerify { joinRequestService.approve(requestId, JoinRequestService.Approval.AsSuperAdmin(orgId), any()) }
    }

    @Test
    fun `org approval as admin with a valid store maps to AsAdmin`() {
        val orgId = UUID.randomUUID()
        val storeId = UUID.randomUUID()
        coEvery { joinRequestService.get(requestId) } returns orgPayload(orgId)
        coEvery { storeRepository.findById(storeId) } returns Store(id = storeId, orgId = orgId, name = "Main")
        coEvery { joinRequestService.approve(any(), any(), any()) } returns savedAdmin

        client.mutateWith(mockAuthentication(TestPrincipals.authToken(role = AdminRole.ROLE_SUPER_ADMIN, orgId = orgId)))
            .post().uri("/api/admins/requests/$requestId/approve")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue("""{"role":"ROLE_ADMIN","storeId":"$storeId"}""")
            .exchange()
            .expectStatus().isNoContent

        coVerify { joinRequestService.approve(requestId, JoinRequestService.Approval.AsAdmin(storeId), any()) }
    }

    @Test
    fun `org approval as admin without a store is 409`() {
        val orgId = UUID.randomUUID()
        coEvery { joinRequestService.get(requestId) } returns orgPayload(orgId)

        client.mutateWith(mockAuthentication(TestPrincipals.authToken(role = AdminRole.ROLE_SUPER_ADMIN, orgId = orgId)))
            .post().uri("/api/admins/requests/$requestId/approve")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue("""{"role":"ROLE_ADMIN"}""")
            .exchange()
            .expectStatus().isEqualTo(409)
    }

    @Test
    fun `super admin role on a store target is forbidden`() {
        val storeId = UUID.randomUUID()
        coEvery { joinRequestService.get(requestId) } returns storePayload(storeId)

        client.mutateWith(mockAuthentication(TestPrincipals.authToken(role = AdminRole.ROLE_ADMIN, storeId = storeId)))
            .post().uri("/api/admins/requests/$requestId/approve")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue("""{"role":"ROLE_SUPER_ADMIN"}""")
            .exchange()
            .expectStatus().isForbidden
    }

    @Test
    fun `non super admin cannot approve an org request`() {
        val orgId = UUID.randomUUID()
        coEvery { joinRequestService.get(requestId) } returns orgPayload(orgId)

        client.mutateWith(mockAuthentication(TestPrincipals.authToken(role = AdminRole.ROLE_ADMIN, storeId = UUID.randomUUID())))
            .post().uri("/api/admins/requests/$requestId/approve")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue("""{"role":"ROLE_ADMIN","storeId":"${UUID.randomUUID()}"}""")
            .exchange()
            .expectStatus().isForbidden
    }

    @Test
    fun `body omitting role defaults to admin`() {
        val orgId = UUID.randomUUID()
        val storeId = UUID.randomUUID()
        coEvery { joinRequestService.get(requestId) } returns orgPayload(orgId)
        coEvery { storeRepository.findById(storeId) } returns Store(id = storeId, orgId = orgId, name = "Main")
        coEvery { joinRequestService.approve(any(), any(), any()) } returns savedAdmin

        client.mutateWith(mockAuthentication(TestPrincipals.authToken(role = AdminRole.ROLE_SUPER_ADMIN, orgId = orgId)))
            .post().uri("/api/admins/requests/$requestId/approve")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue("""{"storeId":"$storeId"}""")
            .exchange()
            .expectStatus().isNoContent

        coVerify { joinRequestService.approve(requestId, JoinRequestService.Approval.AsAdmin(storeId), any()) }
    }
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.admin.controller.JoinRequestControllerTest"`
Expected: FAIL — the current controller (Task 1's minimal `AsAdmin` mapping) still requires a store for ORG requests and never produces `AsSuperAdmin`, so `org approval as super admin maps to AsSuperAdmin` and `super admin role on a store target is forbidden` fail.

- [ ] **Step 3: Replace the controller's `approve` body with role-aware mapping**

In `JoinRequestController.kt`, replace the entire `approve` method (currently `JoinRequestController.kt:41-76`) with:

```kotlin
    @PostMapping("/{requestId}/approve")
    suspend fun approve(
        @PathVariable requestId: String,
        @RequestBody(required = false) body: ApproveJoinRequest?,
        @AuthenticationPrincipal principal: AdminPrincipal,
    ): ResponseEntity<Void> {
        val payload = joinRequestService.get(requestId)
            ?: throw NotFoundException("JoinRequest", "id", requestId)
        val role = body?.role ?: AdminRole.ROLE_ADMIN

        val approval: JoinRequestService.Approval = when (payload.targetType) {
            JoinRequestService.TargetType.ORG -> {
                val orgId = principal.orgId?.takeIf { isSuperAdmin(principal) }
                    ?: throw ForbiddenException("Only the organization owner can approve this request")
                if (UUID.fromString(payload.targetId) != orgId)
                    throw ForbiddenException("Request is not for your organization")
                when (role) {
                    AdminRole.ROLE_SUPER_ADMIN -> JoinRequestService.Approval.AsSuperAdmin(orgId)
                    AdminRole.ROLE_ADMIN -> {
                        val storeId = body?.storeId
                            ?: throw ConflictException("Select a store in your organization to assign")
                        val store = storeRepository.findById(storeId)
                            ?: throw NotFoundException("Store", "id", storeId.toString())
                        if (store.orgId != orgId)
                            throw ForbiddenException("Store is not in your organization")
                        JoinRequestService.Approval.AsAdmin(storeId)
                    }
                }
            }
            JoinRequestService.TargetType.STORE -> {
                // Invariant: an independent store has no org, so SUPER_ADMIN is impossible here.
                if (role == AdminRole.ROLE_SUPER_ADMIN)
                    throw ForbiddenException("An independent store has no organization to own")
                val storeId = principal.storeId
                    ?: throw ForbiddenException("Only the store's admin can approve this request")
                if (UUID.fromString(payload.targetId) != storeId)
                    throw ForbiddenException("Request is not for your store")
                if (!principal.isVerified)
                    throw ForbiddenException("Only a verified admin can approve co-owners")
                JoinRequestService.Approval.AsAdmin(storeId)
            }
        }
        joinRequestService.approve(requestId, approval, principal.id)
        return ResponseEntity.noContent().build()
    }
```

Note: the imports `AdminRole`, `ConflictException`, `ForbiddenException`, `NotFoundException`, `StoreRepository`, and `UUID` are already present in this file. No import changes are required.

- [ ] **Step 4: Run the test to verify it passes**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.admin.controller.JoinRequestControllerTest"`
Expected: PASS — all six tests green.

- [ ] **Step 5: Run the whole admin domain test suite for regressions**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.admin.*"`
Expected: PASS — no regressions in `AdminControllerTest`, `RegistrationServiceTest`, `JoinRequestServiceTest`, etc.

---

### Task 3: Frontend — add the SUPER_ADMIN warning copy (EN + VI)

Done first so the message key exists before the dialog references it, and so `messages/parity.test.ts` stays green.

**Files:**
- Modify: `web/src/messages/en.json`
- Modify: `web/src/messages/vi.json`
- Test: `web/src/messages/parity.test.ts` (existing — no edit, just runs)

- [ ] **Step 1: Add the key to `en.json`**

In `web/src/messages/en.json`, insert the new key immediately after `"requestApproveStoreLabel"` (currently `en.json:338`):

```json
    "requestApproveStoreLabel": "Assign to store",
    "requestApproveSuperAdminNote": "Makes this person an organization owner — full access to every store and the ability to manage other admins.",
```

- [ ] **Step 2: Add the mirrored key to `vi.json`**

In `web/src/messages/vi.json`, insert the new key immediately after `"requestApproveStoreLabel"`:

```json
    "requestApproveStoreLabel": "Gán vào cửa hàng",
    "requestApproveSuperAdminNote": "Cấp quyền chủ tổ chức — có toàn quyền với mọi cửa hàng và quản lý các quản trị viên khác.",
```

- [ ] **Step 3: Lint, then run the parity test**

Run: `cd web && yarn lint && yarn test src/messages/parity.test.ts`
Expected: lint clean (JSON formatting matches Biome); parity test PASS (both locales have identical keys).

---

### Task 4: Frontend — pure gating helpers + unit tests

Extracts the dialog's "is the store required / is confirm enabled" logic into a pure module so it can be unit-tested with vitest (the project has no jsdom/component-test harness).

**Files:**
- Create: `web/src/features/admin/approve-join-request-logic.ts`
- Test: `web/src/features/admin/approve-join-request-logic.test.ts`

- [ ] **Step 1: Write the failing test**

Create `web/src/features/admin/approve-join-request-logic.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import {
  isConfirmEnabled,
  isStoreRequired,
} from "@/features/admin/approve-join-request-logic";

describe("isStoreRequired", () => {
  it("requires a store for an org admin", () => {
    expect(isStoreRequired(true, "ROLE_ADMIN")).toBe(true);
  });
  it("does not require a store for a super admin", () => {
    expect(isStoreRequired(true, "ROLE_SUPER_ADMIN")).toBe(false);
  });
  it("does not require a store for an independent-store admin", () => {
    expect(isStoreRequired(false, "ROLE_ADMIN")).toBe(false);
  });
});

describe("isConfirmEnabled", () => {
  it("disables confirm for an org admin until a store is chosen", () => {
    expect(isConfirmEnabled(true, "ROLE_ADMIN", "")).toBe(false);
    expect(isConfirmEnabled(true, "ROLE_ADMIN", "store-1")).toBe(true);
  });
  it("enables confirm for a super admin without a store", () => {
    expect(isConfirmEnabled(true, "ROLE_SUPER_ADMIN", "")).toBe(true);
  });
  it("enables confirm for an independent-store admin without a store", () => {
    expect(isConfirmEnabled(false, "ROLE_ADMIN", "")).toBe(true);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd web && yarn test src/features/admin/approve-join-request-logic.test.ts`
Expected: FAIL — module `approve-join-request-logic` does not exist.

- [ ] **Step 3: Implement the helpers**

Create `web/src/features/admin/approve-join-request-logic.ts`:

```ts
import type { AdminRole } from "@/types/admin";

/**
 * A store must be chosen only when an org owner approves someone as a store-scoped
 * admin. Super admins are org-wide (no store); independent-store approvals assign the
 * approver's own store server-side.
 */
export function isStoreRequired(
  allowRoleChoice: boolean,
  role: AdminRole,
): boolean {
  return allowRoleChoice && role === "ROLE_ADMIN";
}

/** The approve button is enabled once any required store has been selected. */
export function isConfirmEnabled(
  allowRoleChoice: boolean,
  role: AdminRole,
  storeId: string,
): boolean {
  if (role === "ROLE_SUPER_ADMIN") return true;
  return !isStoreRequired(allowRoleChoice, role) || storeId.length > 0;
}
```

- [ ] **Step 4: Lint, then run the test to verify it passes**

Run: `cd web && yarn lint && yarn test src/features/admin/approve-join-request-logic.test.ts`
Expected: lint clean; tests PASS.

---

### Task 5: Frontend — dialog, panel, and api wiring

These three files are type-coupled (the dialog's `onConfirm` signature, the panel's `doApprove`, and `approveJoinRequest`'s arguments must all match), so they change together to keep the project type-checking.

**Files:**
- Create: `web/src/features/admin/api.test.ts`
- Modify: `web/src/features/admin/api.ts`
- Modify: `web/src/features/admin/approve-join-request-dialog.tsx`
- Modify: `web/src/features/admin/join-requests-panel.tsx`

- [ ] **Step 1: Write the failing api test**

Create `web/src/features/admin/api.test.ts`:

```ts
import { describe, expect, it, vi } from "vitest";

vi.mock("@/lib/api", () => ({
  get: vi.fn(),
  post: vi.fn(),
  patch: vi.fn(),
  del: vi.fn(),
}));

import { approveJoinRequest } from "@/features/admin/api";
import { post } from "@/lib/api";

describe("approveJoinRequest", () => {
  it("posts the role and store id to the approve route", () => {
    approveJoinRequest("req-1", "ROLE_ADMIN", "store-9");
    expect(post).toHaveBeenCalledWith("/api/admins/requests/req-1/approve", {
      role: "ROLE_ADMIN",
      storeId: "store-9",
    });
  });

  it("omits the store id for a super-admin approval", () => {
    approveJoinRequest("req-2", "ROLE_SUPER_ADMIN");
    expect(post).toHaveBeenCalledWith("/api/admins/requests/req-2/approve", {
      role: "ROLE_SUPER_ADMIN",
      storeId: undefined,
    });
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd web && yarn test src/features/admin/api.test.ts`
Expected: FAIL — `approveJoinRequest` currently has signature `(requestId, storeId?)` and posts `{ storeId }` (or `undefined`), so neither assertion matches.

- [ ] **Step 3: Update `approveJoinRequest`**

In `web/src/features/admin/api.ts`:

First, add `AdminRole` to the type import block (it currently imports from `@/types/admin`):

```ts
import type {
  AdminDto,
  AdminPageResponse,
  AdminRole,
  AdminSessionDto,
  CreateAdminRequest,
  JoinRequestDto,
  LoginHistoryPageResponse,
  UpdatePasswordRequest,
  UpdateUsernameRequest,
} from "@/types/admin";
```

Then replace the `approveJoinRequest` function (currently `api.ts:88-93`):

```ts
export function approveJoinRequest(
  requestId: string,
  role: AdminRole,
  storeId?: string,
) {
  return post<void>(API_ROUTES.ADMINS.APPROVE_REQUEST(requestId), {
    role,
    storeId,
  });
}
```

- [ ] **Step 4: Run the api test to verify it passes**

Run: `cd web && yarn test src/features/admin/api.test.ts`
Expected: PASS — both assertions match (`toHaveBeenCalledWith` ignores the `undefined` `storeId`).

- [ ] **Step 5: Rewrite the approve dialog**

Replace the entire contents of `web/src/features/admin/approve-join-request-dialog.tsx`:

```tsx
"use client";

import { AlertTriangle, Loader2 } from "lucide-react";
import { useTranslations } from "next-intl";
import { useEffect, useState } from "react";
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
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
} from "@/components/ui/select";
import {
  isConfirmEnabled,
  isStoreRequired,
} from "@/features/admin/approve-join-request-logic";
import type { AdminRole, JoinRequestDto } from "@/types/admin";
import type { StoreDto } from "@/types/store";

interface ApproveJoinRequestDialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  request: JoinRequestDto | null;
  // True only for org-owner approvals (ORG-target requests): the approver may pick the
  // role, and a store is required only when ROLE_ADMIN is chosen.
  allowRoleChoice: boolean;
  stores: StoreDto[];
  onConfirm: (role: AdminRole, storeId?: string) => Promise<void>;
}

export function ApproveJoinRequestDialog({
  open,
  onOpenChange,
  request,
  allowRoleChoice,
  stores,
  onConfirm,
}: ApproveJoinRequestDialogProps) {
  const tAdmins = useTranslations("admins");
  const tCommon = useTranslations("common");
  const [role, setRole] = useState<AdminRole>("ROLE_ADMIN");
  const [storeId, setStoreId] = useState("");
  const [loading, setLoading] = useState(false);

  // Reset the selection each time the dialog opens for a fresh request. A failed approve
  // keeps the dialog open (open stays true), so the admin's selection is preserved for
  // retry; only a new open resets it.
  useEffect(() => {
    if (open) {
      setRole("ROLE_ADMIN");
      setStoreId("");
    }
  }, [open]);

  const storeRequired = isStoreRequired(allowRoleChoice, role);
  const confirmEnabled = isConfirmEnabled(allowRoleChoice, role, storeId);

  async function confirm() {
    if (!confirmEnabled) return;
    setLoading(true);
    try {
      await onConfirm(role, storeRequired ? storeId : undefined);
      onOpenChange(false);
      setRole("ROLE_ADMIN");
      setStoreId("");
    } finally {
      setLoading(false);
    }
  }

  return (
    <AlertDialog
      open={open}
      onOpenChange={(next) => !loading && onOpenChange(next)}
    >
      <AlertDialogContent>
        <AlertDialogHeader>
          <AlertDialogTitle>{tAdmins("requestApproveTitle")}</AlertDialogTitle>
          <AlertDialogDescription>{request?.username}</AlertDialogDescription>
        </AlertDialogHeader>
        <div className="space-y-4">
          {allowRoleChoice && (
            <div className="space-y-2">
              <Label>{tAdmins("roleLabel")}</Label>
              <Select
                value={role}
                onValueChange={(v) => v && setRole(v as AdminRole)}
              >
                <SelectTrigger className="h-10 w-full gap-2 px-3">
                  <span>
                    {role === "ROLE_SUPER_ADMIN"
                      ? tAdmins("roleSuperAdmin")
                      : tAdmins("roleAdmin")}
                  </span>
                </SelectTrigger>
                <SelectContent
                  align="start"
                  alignItemWithTrigger={false}
                  className="p-1.5"
                >
                  <SelectItem value="ROLE_ADMIN" className="py-2">
                    {tAdmins("roleAdmin")}
                  </SelectItem>
                  <SelectItem value="ROLE_SUPER_ADMIN" className="py-2">
                    {tAdmins("roleSuperAdmin")}
                  </SelectItem>
                </SelectContent>
              </Select>
            </div>
          )}
          {storeRequired && (
            <div className="space-y-2">
              <Label>{tAdmins("requestApproveStoreLabel")}</Label>
              <Select value={storeId} onValueChange={(v) => v && setStoreId(v)}>
                <SelectTrigger className="h-10 w-full gap-2 px-3">
                  <span>
                    {storeId
                      ? stores.find((s) => s.id === storeId)?.name
                      : tAdmins("storePlaceholder")}
                  </span>
                </SelectTrigger>
                <SelectContent
                  align="start"
                  alignItemWithTrigger={false}
                  className="p-1.5"
                >
                  {stores.map((s) => (
                    <SelectItem key={s.id} value={s.id} className="py-2">
                      {s.name}
                    </SelectItem>
                  ))}
                </SelectContent>
              </Select>
            </div>
          )}
          {role === "ROLE_SUPER_ADMIN" && (
            <div className="flex items-start gap-2.5 rounded-xl border border-warning/40 bg-warning/15 px-3.5 py-3 text-warning dark:border-warning/50 dark:bg-warning/20">
              <AlertTriangle aria-hidden="true" className="size-4 shrink-0" />
              <p className="text-sm">
                {tAdmins("requestApproveSuperAdminNote")}
              </p>
            </div>
          )}
        </div>
        <AlertDialogFooter>
          <AlertDialogCancel disabled={loading}>
            {tCommon("cancel")}
          </AlertDialogCancel>
          <AlertDialogAction
            disabled={loading || !confirmEnabled}
            onClick={(e) => {
              e.preventDefault();
              void confirm();
            }}
            className="bg-primary text-primary-foreground hover:bg-primary-hover"
          >
            {loading && (
              <Loader2
                aria-hidden="true"
                className="mr-2 size-4 animate-spin"
              />
            )}
            {tAdmins("requestApprove")}
          </AlertDialogAction>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  );
}
```

- [ ] **Step 6: Update the panel to thread `role` and pass `allowRoleChoice`**

In `web/src/features/admin/join-requests-panel.tsx`:

Add `AdminRole` to the `@/types/admin` import (currently `import type { JoinRequestDto } from "@/types/admin";`):

```tsx
import type { AdminRole, JoinRequestDto } from "@/types/admin";
```

Replace the `doApprove` function (currently `join-requests-panel.tsx:53-70`):

```tsx
  async function doApprove(role: AdminRole, storeId?: string) {
    if (!approveTarget) return;
    try {
      await approveJoinRequest(approveTarget.requestId, role, storeId);
      toast.success(
        tAdmins("requestApprovedToast", { username: approveTarget.username }),
      );
      await fetchRequests();
    } catch (err) {
      toast.error(
        err instanceof ApiError
          ? translateCommonApiError(err, tErrors)
          : translateNetworkError(tErrors),
      );
      // Re-throw so the dialog stays open on failure and the admin can retry.
      throw err;
    }
  }
```

Change the dialog prop (currently `requireStore={isSuperAdmin}` at `join-requests-panel.tsx:132`):

```tsx
        allowRoleChoice={isSuperAdmin}
```

- [ ] **Step 7: Lint, then run the full admin feature + messages tests**

Run: `cd web && yarn lint && yarn test src/features/admin src/messages/parity.test.ts`
Expected: lint clean; all tests PASS (helper tests, api test, parity test). The dialog/panel changes are covered by type-checking + lint + the helper tests; there is no render test (no jsdom harness — see spec).

---

### Task 6: Frontend — full lint + test sweep

A final guard that the whole web project is consistent after the coupled changes.

- [ ] **Step 1: Lint the whole project**

Run: `cd web && yarn lint`
Expected: clean.

- [ ] **Step 2: Run the whole vitest suite**

Run: `cd web && yarn test`
Expected: all tests PASS (no regressions in stores, lib, or message parity).

---

### Task 7: Documentation — CHANGELOGS

**Files:**
- Modify: `docs/CHANGELOGS.md`

- [ ] **Step 1: Append a changelog entry**

Add an entry under the current date (2026-06-13) summarizing:

- **Backend:** `ApproveJoinRequest` gains `role` (default `ROLE_ADMIN`); `JoinRequestService` gains the `Approval` sealed type and `approve(requestId, Approval, verifierId)`; `JoinRequestController.approve` maps role → `Approval` and enforces the invariant that `ROLE_SUPER_ADMIN` is only reachable from ORG-target requests (STORE targets reject it with 403). New tests: `JoinRequestControllerTest`; `approve` tests added to `JoinRequestServiceTest`.
- **Frontend:** approve dialog gains a role picker (default `ROLE_ADMIN`), a conditional store select, and a SUPER_ADMIN warning box; `requireStore` prop renamed to `allowRoleChoice`; `approveJoinRequest(requestId, role, storeId?)`. New pure helpers `approve-join-request-logic.ts` + tests; new `api.test.ts`. New i18n key `admins.requestApproveSuperAdminNote` (EN + VI).
- **Deliberately unchanged (record per repo convention):** the manual "Add admin" path (`POST /api/admins` / `AdminService.createAdmin`) remains `ROLE_ADMIN`-only; no schema change (existing `chk_admin_tenancy` honored by construction).

- [ ] **Step 2: Verify the changelog renders**

Open `docs/CHANGELOGS.md` and confirm the entry is well-formed Markdown and placed under the correct date heading.

---

## Self-Review

**Spec coverage** (every spec section maps to a task):

| Spec requirement | Task |
|---|---|
| `ApproveJoinRequest.role` field | Task 1 |
| `Approval` sealed type + `approve` rewrite | Task 1 |
| Controller role→Approval mapping | Task 2 |
| ORG-only SUPER_ADMIN invariant (STORE → 403) | Task 2 |
| Backend service tests (AsSuperAdmin / AsAdmin) | Task 1 |
| Backend controller tests (mapping, 409, invariant, authz, backward-compat) | Task 2 |
| Dialog: role picker, conditional store, warning, `allowRoleChoice`, `onConfirm` | Task 5 |
| Pure gating helpers + tests | Task 4 |
| Panel wiring | Task 5 |
| `approveJoinRequest(requestId, role, storeId?)` + test | Task 5 |
| i18n `requestApproveSuperAdminNote` (EN + VI), parity | Task 3 |
| CHANGELOGS | Task 7 |

**Type/signature consistency:** `Approval` / `Approval.AsAdmin` / `Approval.AsSuperAdmin` used identically across Tasks 1–2; `approve(requestId, Approval, verifierId)` signature consistent; `isStoreRequired(allowRoleChoice, role)` / `isConfirmEnabled(allowRoleChoice, role, storeId)` defined in Task 4 and consumed in Task 5; `approveJoinRequest(requestId, role, storeId?)` defined and consumed consistently; `onConfirm(role, storeId?)` matches `doApprove(role, storeId?)`.

**Ordering keeps the build green:** Task 1 changes the service signature *and* the one controller call site together (compiles, behavior unchanged); Task 2 swaps the controller body (compiles); Task 3 (i18n) precedes Task 5 (dialog references the key); Task 4 (helpers) precedes Task 5 (dialog imports them); Task 5 changes the three type-coupled frontend files together.
