# Expiring Invite Links Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement expiring (7-day), multi-use, revocable invite links on top of the existing join-code system, including a 30-day per-tenant usage audit trail, per `docs/spec/Expiring Invite Links Spec.md`.

**Architecture:** Backend stores invite links entirely in Redis (token key + per-tenant active key + per-tenant audit ZSet), following `JoinRequestService`'s exact patterns (`ReactiveRedisTemplate<String, String>` + `ObjectMapper`, `RedisKeyManager`, `RedisTTLPolicy`, `setIfAbsent` mutex). Five new endpoints (4 owner-scoped, 1 public resolve). The web gains an `InviteLinkPanel` (sibling of `JoinCodePanel`), a register-page deep link via an extracted `RegisterWizard` wrapped in `<Suspense>` (required by Next.js 16 for `useSearchParams` in statically rendered routes), and bilingual i18n.

**Tech Stack:** Kotlin 2.3 / Spring Boot 3.5 WebFlux coroutines, Spring Data Redis 3.5 reactive, MockK + springmockk + WebTestClient (backend tests); Next.js 16 / React 19 / next-intl / shadcn/ui / Biome / Vitest (web).

---

## Ground Rules (project conventions — binding for the executor)

1. **No git commits.** The user decides when to commit. Plan steps therefore contain **no commit steps** (explicit user preference overriding the writing-plans skill default).
2. **No builds.** Do not run `./gradlew build` or `yarn build` — verification is tests + lint only; builds happen in the project's separate audit flow.
3. **Lint before tests** on the web side: `yarn lint` first, then `yarn test`.
4. **GitNexus:** before editing any *existing* symbol (e.g. `RegistrationService.join`, `OrganizationController`, `StoreController`, `AuthController`, `RegisterJoinForm`), run `gitnexus_impact({target: "<symbol>", direction: "upstream"})` and report the blast radius. Surface (do not silently proceed past) HIGH/CRITICAL warnings.
5. **Named imports only** — no fully-qualified inline types, including in new test files. (Some existing controller tests use FQN `@MockkBean` declarations; new files must NOT copy that — CLAUDE.md wins. When *modifying* `AuthControllerTest`, match that file's existing style for the added `@MockkBean` line but use named imports for everything else.)
6. **Log everything to `docs/CHANGELOGS.md`** at the end, including anything skipped (Task 15).
7. **Vietnamese copy** in this plan was drafted for approval at plan review. If the user amends any string in chat, use the amended version, not the one printed here.
8. Frontend: only shadcn/ui components already in the project (`Input`, `Button`, `Badge`, `AlertDialog` — all exist). Alert boxes use the canonical patterns from `docs/walkthrough/Web Styles.md` § "Status Alert Boxes". Time display is language-agnostic (fixed-locale `Intl.DateTimeFormat("en-US", …)`, the `waiting-list.tsx` pattern).

## Spec Ambiguities Resolved (audit log — decided before any code)

| # | Spec point | Resolution in this plan |
|---|-----------|------------------------|
| A1 | § 4.3 calls `InviteTarget` a **private** data class, but `resolve(): InviteTarget?` is consumed by `RegistrationService` — a private type cannot cross that boundary. | `InviteTarget` is a **public nested** data class of `InviteLinkService`, mirroring the existing `JoinRequestService.JoinRequestPayload` pattern. |
| A2 | § 4.4 says *AuthController gains an `OrganizationRepository` injection*, but § 4.3 puts the repository lookups inside `InviteLinkService.resolveForDisplay` (repos injected into the **service**). | Repos are injected into `InviteLinkService` (§ 4.3 wins — it is the explicit design). `AuthController` gains an **`InviteLinkService`** injection instead. § 4.4's parenthetical is treated as vestigial. |
| A3 | § 5.2 says trail timestamps use "fixed-locale formatting, same as `JoinRequestsPanel`" — but `JoinRequestsPanel` renders no timestamps. | Use the actual codebase pattern: a module-level `Intl.DateTimeFormat("en-US", …)` constant, as in `web/src/features/queue/waiting-list.tsx:27`. |
| A4 | § 8 says "no test framework exists in `web/`" — stale; Vitest now exists but only covers `src/**/*.test.ts` in a node environment (no component tests). | No new web component tests. The existing `src/messages/parity.test.ts` automatically guards the new i18n keys (key parity + rich-text tag parity). Manual scenarios remain for the audit flow. |
| A5 | § 4.4 requires 409 for org-owned stores on the two store endpoints, but § 10's file list does not touch `StoreService`. | The check lives in a private `StoreController.requireIndependentStore(id)` helper built on the existing `storeService.getStore(id)` (`StoreDto.orgId`), throwing the same `ConflictException` message as `StoreService.getStoreJoinCode`. `StoreService` stays untouched. |
| A6 | `RedisKeyManager` lives in `core/` — it must not depend on `domain/admin`'s `TargetType` enum. | The four new key functions take `targetType: String`; callers pass `TargetType.name` (`"ORG"`/`"STORE"`), matching the spec's key layout exactly. |
| A7 | § 8 lists "recordUse failure does not fail registration" as a **RegistrationService** test, while § 4.6 makes `recordUse` itself never throw. | Defense in depth: `recordUse` never throws (per § 4.6) **and** `RegistrationService` wraps the call in a try/catch that rethrows only `CancellationException`. Both layers are tested. |
| A8 | Web `RegisterOutcome` type alias name after the `outcome` → `status` field rename. | Keep the alias name `RegisterOutcome` (1 usage); rename only the **field** to `status` as the spec instructs. |

## File Map

**Backend — modified:** `core/tenant/JoinCodeGenerator.kt`, `core/redis/RedisKeyManager.kt`, `core/redis/RedisTTLPolicy.kt`, `domain/admin/request/RegisterRequest.kt`, `domain/admin/service/RegistrationService.kt`, `domain/organization/controller/OrganizationController.kt`, `domain/store/controller/StoreController.kt`, `domain/admin/controller/AuthController.kt`
**Backend — created:** `domain/admin/service/InviteLinkService.kt`, `domain/organization/response/InviteLinkResponse.kt`, `domain/admin/response/InviteResolveResponse.kt`
**Backend tests — modified:** `JoinCodeGeneratorTest.kt`, `RedisKeyManagerTest.kt`, `RedisTTLPolicyTest.kt`, `AuthControllerTest.kt`
**Backend tests — created:** `InviteLinkServiceTest.kt`, `RegistrationServiceTest.kt`, `OrganizationControllerTest.kt`, `StoreControllerTest.kt`
**Web — modified:** `types/admin.ts`, `types/organization.ts`, `lib/constants.ts`, `features/organization/api.ts`, `features/auth/api.ts`, `features/auth/register-join-form.tsx`, `app/[locale]/(auth)/register/page.tsx`, `app/[locale]/dashboard/settings/organization/page.tsx`, `app/[locale]/dashboard/settings/store/page.tsx`, `app/[locale]/dashboard/admins/page.tsx`, `messages/en.json`, `messages/vi.json`
**Web — created:** `features/auth/register-wizard.tsx`, `features/organization/invite-link-panel.tsx`
**Docs — modified:** `docs/CHANGELOGS.md`

All backend paths below are relative to `backend/src/main/kotlin/com/thomas/notiguide/` (main) and `backend/src/test/kotlin/com/thomas/notiguide/` (test). All web paths are relative to `web/src/`.

---

### Task 1: Token generator — `INVITE_PREFIX` + byte-count overload

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/core/tenant/JoinCodeGenerator.kt`
- Test: `backend/src/test/kotlin/com/thomas/notiguide/core/tenant/JoinCodeGeneratorTest.kt`

- [x] **Step 1: Add the failing tests** — append inside the existing `JoinCodeGeneratorTest` class:

```kotlin
    @Test
    fun `invite token carries the invite prefix`() {
        assertThat(JoinCodeGenerator.generate(JoinCodeGenerator.INVITE_PREFIX, 16)).startsWith("i_")
    }

    @Test
    fun `a 16-byte invite token is 24 chars total`() {
        // 16 bytes → 22 base64url chars (no padding) + the 2-char prefix
        assertThat(JoinCodeGenerator.generate(JoinCodeGenerator.INVITE_PREFIX, 16)).hasSize(24)
    }

    @Test
    fun `the single-arg overload keeps the legacy 9-byte length`() {
        // 9 bytes → 12 base64url chars + the 2-char prefix
        assertThat(JoinCodeGenerator.generate(JoinCodeGenerator.ORG_PREFIX)).hasSize(14)
    }

    @Test
    fun `invite token suffix is url-safe`() {
        val suffix = JoinCodeGenerator.generate(JoinCodeGenerator.INVITE_PREFIX, 16).removePrefix("i_")
        assertThat(suffix).matches("[A-Za-z0-9_-]+")
    }
```

- [x] **Step 2: Run the test class — expect failure**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.core.tenant.JoinCodeGeneratorTest"`
Expected: compilation FAILS with `unresolved reference 'INVITE_PREFIX'` (and the 2-arg `generate`).

- [x] **Step 3: Implement** — replace the body of `JoinCodeGenerator.kt` with:

```kotlin
package com.thomas.notiguide.core.tenant

import java.security.SecureRandom
import java.util.Base64

object JoinCodeGenerator {
    const val ORG_PREFIX = "o_"
    const val STORE_PREFIX = "s_"
    const val INVITE_PREFIX = "i_"

    private const val DEFAULT_BYTE_COUNT = 9

    private val secureRandom = SecureRandom()
    private val encoder = Base64.getUrlEncoder().withoutPadding()

    /** e.g. "o_X1k9..." — 9 random bytes → 12 url-safe chars after the prefix. */
    fun generate(prefix: String): String = generate(prefix, DEFAULT_BYTE_COUNT)

    /** e.g. "i_…" — byteCount random bytes, base64url without padding, after the prefix. */
    fun generate(prefix: String, byteCount: Int): String {
        val bytes = ByteArray(byteCount)
        secureRandom.nextBytes(bytes)
        return prefix + encoder.encodeToString(bytes)
    }
}
```

- [x] **Step 4: Run the test class — expect pass**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.core.tenant.JoinCodeGeneratorTest"`
Expected: `BUILD SUCCESSFUL`, 7 tests pass (3 pre-existing + 4 new).

---

### Task 2: Redis keys & TTL policy

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/core/redis/RedisKeyManager.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/core/redis/RedisTTLPolicy.kt`
- Test: `backend/src/test/kotlin/com/thomas/notiguide/core/redis/RedisKeyManagerTest.kt`
- Test: `backend/src/test/kotlin/com/thomas/notiguide/core/redis/RedisTTLPolicyTest.kt`

- [x] **Step 1: Add the failing tests.** In `RedisKeyManagerTest` append:

```kotlin
    @Test
    fun `invite keys have expected formats`() {
        assertThat(RedisKeyManager.inviteToken("i_abc")).isEqualTo("invite:token:i_abc")
        assertThat(RedisKeyManager.inviteActive("ORG", storeId)).isEqualTo("invite:active:ORG:$storeId")
        assertThat(RedisKeyManager.inviteLock("STORE", storeId)).isEqualTo("invite:lock:STORE:$storeId")
        assertThat(RedisKeyManager.inviteAudit("STORE", storeId)).isEqualTo("invite:audit:STORE:$storeId")
    }
```

In `RedisTTLPolicyTest` append:

```kotlin
    @Test
    fun `invite link TTLs match the documented policy`() {
        assertThat(RedisTTLPolicy.INVITE_LINK).isEqualTo(Duration.ofDays(7))
        assertThat(RedisTTLPolicy.INVITE_AUDIT).isEqualTo(Duration.ofDays(30))
    }
```

- [x] **Step 2: Run both test classes — expect compilation failure**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.core.redis.RedisKeyManagerTest" --tests "com.thomas.notiguide.core.redis.RedisTTLPolicyTest"`
Expected: FAILS — `unresolved reference 'inviteToken'` / `'INVITE_LINK'`.

- [x] **Step 3: Implement.** In `RedisKeyManager.kt`, directly below the `joinRequestLock` line (the join-request group, line ~38), add:

```kotlin
    fun inviteToken(token: String) = "invite:token:$token"
    fun inviteActive(targetType: String, targetId: UUID) = "invite:active:$targetType:$targetId"
    fun inviteLock(targetType: String, targetId: UUID) = "invite:lock:$targetType:$targetId"
    fun inviteAudit(targetType: String, targetId: UUID) = "invite:audit:$targetType:$targetId"
```

(`targetType` is a `String` — callers pass `TargetType.name` — so `core/` gains no dependency on `domain/`.)

In `RedisTTLPolicy.kt` add two values at the end of the object:

```kotlin
    val INVITE_LINK: Duration = Duration.ofDays(7)
    val INVITE_AUDIT: Duration = Duration.ofDays(30)
```

- [x] **Step 4: Run both test classes — expect pass**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.core.redis.RedisKeyManagerTest" --tests "com.thomas.notiguide.core.redis.RedisTTLPolicyTest"`
Expected: `BUILD SUCCESSFUL`.

---

### Task 3: Response DTOs

Pure data classes — no test steps (they are exercised by every later task's tests).

**Files:**
- Create: `backend/src/main/kotlin/com/thomas/notiguide/domain/organization/response/InviteLinkResponse.kt`
- Create: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/response/InviteResolveResponse.kt`

- [x] **Step 1:** Create `InviteLinkResponse.kt` (sibling of `JoinCodeResponse.kt`). `InviteLinkUse` lives beside it per spec § 4.4; the Jackson-friendly defaults mirror `JoinRequestService.JoinRequestPayload` because `InviteLinkService.getRecentUses` deserializes trail members back into this DTO:

```kotlin
package com.thomas.notiguide.domain.organization.response

/** One usage-trail entry: a join submission made through an invite link. */
data class InviteLinkUse(
    val username: String = "",
    val usedAt: String = "",
    val linkId: String = ""
)

/** `token`/`expiresAt` both null = "no active link" (a normal state, always HTTP 200). */
data class InviteLinkResponse(
    val token: String?,
    val expiresAt: String?,
    val recentUses: List<InviteLinkUse> = emptyList()
)
```

- [x] **Step 2:** Create `InviteResolveResponse.kt` (sibling of `RegisterResponse.kt`):

```kotlin
package com.thomas.notiguide.domain.admin.response

data class InviteResolveResponse(
    val targetType: String,
    val name: String
)
```

- [x] **Step 3: Compile check**

Run: `cd backend && ./gradlew compileKotlin`
Expected: `BUILD SUCCESSFUL`.

---

### Task 4: `InviteLinkService` — link lifecycle (getActive / regenerate / resolve / resolveForDisplay)

**Files:**
- Create: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/service/InviteLinkService.kt`
- Create: `backend/src/test/kotlin/com/thomas/notiguide/domain/admin/service/InviteLinkServiceTest.kt`

- [x] **Step 1: Write the failing tests.** Create `InviteLinkServiceTest.kt` with the lifecycle tests (trail tests are added in Task 5):

```kotlin
package com.thomas.notiguide.domain.admin.service

import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import com.thomas.notiguide.core.exception.ConflictException
import com.thomas.notiguide.core.redis.RedisTTLPolicy
import com.thomas.notiguide.domain.organization.entity.Organization
import com.thomas.notiguide.domain.organization.repository.OrganizationRepository
import com.thomas.notiguide.domain.store.entity.Store
import com.thomas.notiguide.domain.store.repository.StoreRepository
import io.mockk.coEvery
import io.mockk.every
import io.mockk.mockk
import io.mockk.verify
import io.mockk.verifyOrder
import kotlinx.coroutines.test.runTest
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.data.redis.core.ReactiveRedisTemplate
import org.springframework.data.redis.core.ReactiveValueOperations
import org.springframework.data.redis.core.ReactiveZSetOperations
import reactor.core.publisher.Mono
import java.time.Duration
import java.util.UUID

class InviteLinkServiceTest {
    private val redis = mockk<ReactiveRedisTemplate<String, String>>()
    private val valueOps = mockk<ReactiveValueOperations<String, String>>()
    private val zSetOps = mockk<ReactiveZSetOperations<String, String>>()
    private val organizationRepository = mockk<OrganizationRepository>()
    private val storeRepository = mockk<StoreRepository>()
    private val service =
        InviteLinkService(redis, jacksonObjectMapper(), organizationRepository, storeRepository)

    private val orgId = UUID.fromString("11111111-1111-1111-1111-111111111111")
    private val storeId = UUID.fromString("22222222-2222-2222-2222-222222222222")

    init {
        every { redis.opsForValue() } returns valueOps
        every { redis.opsForZSet() } returns zSetOps
    }

    @Test
    fun `getActive returns null when no active entry exists`() = runTest {
        every { valueOps.get("invite:active:ORG:$orgId") } returns Mono.empty()
        assertThat(service.getActive(JoinRequestService.TargetType.ORG, orgId)).isNull()
    }

    @Test
    fun `getActive returns the active link state without minting`() = runTest {
        every { valueOps.get("invite:active:ORG:$orgId") } returns
            Mono.just("""{"token":"i_active123","expiresAt":"2026-06-18T10:00:00Z"}""")

        val link = service.getActive(JoinRequestService.TargetType.ORG, orgId)

        assertThat(link?.token).isEqualTo("i_active123")
        assertThat(link?.expiresAt).isEqualTo("2026-06-18T10:00:00Z")
        verify(exactly = 0) { valueOps.set(any(), any(), any<Duration>()) }
    }

    @Test
    fun `regenerate deletes the old token before writing the new pair with TTLs`() = runTest {
        every { valueOps.setIfAbsent("invite:lock:ORG:$orgId", any(), Duration.ofSeconds(10)) } returns
            Mono.just(true)
        every { valueOps.get("invite:active:ORG:$orgId") } returns
            Mono.just("""{"token":"i_old","expiresAt":"2026-06-12T00:00:00Z"}""")
        every { redis.delete("invite:token:i_old") } returns Mono.just(1L)
        every { valueOps.set(any(), any(), RedisTTLPolicy.INVITE_LINK) } returns Mono.just(true)
        every { redis.delete("invite:lock:ORG:$orgId") } returns Mono.just(1L)

        val link = service.regenerate(JoinRequestService.TargetType.ORG, orgId)

        assertThat(link.token).startsWith("i_").isNotEqualTo("i_old")
        assertThat(link.expiresAt).isNotNull()
        verifyOrder {
            redis.delete("invite:token:i_old")
            valueOps.set(match { it.startsWith("invite:token:i_") }, any(), RedisTTLPolicy.INVITE_LINK)
            valueOps.set("invite:active:ORG:$orgId", any(), RedisTTLPolicy.INVITE_LINK)
            redis.delete("invite:lock:ORG:$orgId")
        }
    }

    @Test
    fun `regenerate without an existing link skips the revoke delete`() = runTest {
        every { valueOps.setIfAbsent("invite:lock:STORE:$storeId", any(), any<Duration>()) } returns
            Mono.just(true)
        every { valueOps.get("invite:active:STORE:$storeId") } returns Mono.empty()
        every { valueOps.set(any(), any(), RedisTTLPolicy.INVITE_LINK) } returns Mono.just(true)
        every { redis.delete("invite:lock:STORE:$storeId") } returns Mono.just(1L)

        val link = service.regenerate(JoinRequestService.TargetType.STORE, storeId)

        assertThat(link.token).startsWith("i_")
        verify(exactly = 0) { redis.delete(match<String> { it.startsWith("invite:token:") }) }
    }

    @Test
    fun `regenerate is rejected with a conflict while the per-tenant lock is held`() = runTest {
        every { valueOps.setIfAbsent("invite:lock:ORG:$orgId", any(), any<Duration>()) } returns
            Mono.just(false)

        val ex = runCatching { service.regenerate(JoinRequestService.TargetType.ORG, orgId) }
            .exceptionOrNull()

        assertThat(ex).isInstanceOf(ConflictException::class.java)
        verify(exactly = 0) { valueOps.set(any(), any(), any<Duration>()) }
    }

    @Test
    fun `regenerate releases the lock even when a redis write fails`() = runTest {
        every { valueOps.setIfAbsent("invite:lock:ORG:$orgId", any(), any<Duration>()) } returns
            Mono.just(true)
        every { valueOps.get("invite:active:ORG:$orgId") } returns Mono.empty()
        every { valueOps.set(any(), any(), any<Duration>()) } returns
            Mono.error(RuntimeException("redis down"))
        every { redis.delete("invite:lock:ORG:$orgId") } returns Mono.just(1L)

        runCatching { service.regenerate(JoinRequestService.TargetType.ORG, orgId) }

        verify { redis.delete("invite:lock:ORG:$orgId") }
    }

    @Test
    fun `resolve returns the target and never consumes the token`() = runTest {
        every { valueOps.get("invite:token:i_tok") } returns
            Mono.just("""{"targetType":"STORE","targetId":"$storeId","expiresAt":"2026-06-18T10:00:00Z"}""")

        val first = service.resolve("i_tok")
        val second = service.resolve("i_tok")

        assertThat(first?.targetId).isEqualTo(storeId.toString())
        assertThat(second?.targetType).isEqualTo(JoinRequestService.TargetType.STORE)
        verify(exactly = 0) { redis.delete(any<String>()) }
    }

    @Test
    fun `resolve returns null for unknown and non-invite tokens`() = runTest {
        every { valueOps.get(any<String>()) } returns Mono.empty()

        assertThat(service.resolve("i_unknown")).isNull()
        assertThat(service.resolve("o_notAnInviteToken")).isNull()
        // the non-invite prefix short-circuits before touching Redis
        verify(exactly = 1) { valueOps.get(any<String>()) }
    }

    @Test
    fun `resolveForDisplay returns the org name for an org target`() = runTest {
        every { valueOps.get("invite:token:i_tok") } returns
            Mono.just("""{"targetType":"ORG","targetId":"$orgId","expiresAt":"2026-06-18T10:00:00Z"}""")
        coEvery { organizationRepository.findById(orgId) } returns
            Organization(id = orgId, name = "Acme Group", joinCode = "o_x")

        val res = service.resolveForDisplay("i_tok")

        assertThat(res?.targetType).isEqualTo("ORG")
        assertThat(res?.name).isEqualTo("Acme Group")
    }

    @Test
    fun `resolveForDisplay returns null when the target was deleted`() = runTest {
        every { valueOps.get("invite:token:i_tok") } returns
            Mono.just("""{"targetType":"ORG","targetId":"$orgId","expiresAt":"2026-06-18T10:00:00Z"}""")
        coEvery { organizationRepository.findById(orgId) } returns null

        assertThat(service.resolveForDisplay("i_tok")).isNull()
    }

    @Test
    fun `resolveForDisplay returns null when the store became org-owned`() = runTest {
        every { valueOps.get("invite:token:i_tok") } returns
            Mono.just("""{"targetType":"STORE","targetId":"$storeId","expiresAt":"2026-06-18T10:00:00Z"}""")
        coEvery { storeRepository.findById(storeId) } returns
            Store(id = storeId, orgId = orgId, name = "Acme Store")

        assertThat(service.resolveForDisplay("i_tok")).isNull()
    }
}
```

- [x] **Step 2: Run — expect compilation failure**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.admin.service.InviteLinkServiceTest"`
Expected: FAILS — `unresolved reference 'InviteLinkService'`.

- [x] **Step 3: Implement the service.** Create `InviteLinkService.kt`. Notes baked into the design: regenerate deletes the old token key **before** writing the new pair (spec § 4.3 failure-ordering argument); `expiresAt` is computed at mint time and stored in both payloads (spec § 4.2); `resolve` is read-only and also rejects non-`i_` prefixes and oversized strings before touching Redis. `recordUse`/`getRecentUses` bodies are included now (they are tested in Task 5 — keep the file complete in one write):

```kotlin
package com.thomas.notiguide.domain.admin.service

import com.fasterxml.jackson.databind.ObjectMapper
import com.thomas.notiguide.core.exception.ConflictException
import com.thomas.notiguide.core.redis.RedisKeyManager
import com.thomas.notiguide.core.redis.RedisTTLPolicy
import com.thomas.notiguide.core.tenant.JoinCodeGenerator
import com.thomas.notiguide.domain.admin.response.InviteResolveResponse
import com.thomas.notiguide.domain.organization.repository.OrganizationRepository
import com.thomas.notiguide.domain.organization.response.InviteLinkResponse
import com.thomas.notiguide.domain.organization.response.InviteLinkUse
import com.thomas.notiguide.domain.store.repository.StoreRepository
import kotlinx.coroutines.CancellationException
import kotlinx.coroutines.reactor.awaitSingle
import kotlinx.coroutines.reactor.awaitSingleOrNull
import org.slf4j.LoggerFactory
import org.springframework.data.domain.Range
import org.springframework.data.redis.core.ReactiveRedisTemplate
import org.springframework.stereotype.Service
import java.time.Duration
import java.time.OffsetDateTime
import java.util.UUID

@Service
class InviteLinkService(
    private val redis: ReactiveRedisTemplate<String, String>,
    private val objectMapper: ObjectMapper,
    private val organizationRepository: OrganizationRepository,
    private val storeRepository: StoreRepository
) {
    private val log = LoggerFactory.getLogger(InviteLinkService::class.java)

    /** Token payload stored at invite:token:{token}. */
    data class InviteTarget(
        val targetType: JoinRequestService.TargetType = JoinRequestService.TargetType.ORG,
        val targetId: String = "",
        val expiresAt: String = ""
    )

    /** Active-link payload stored at invite:active:{type}:{id}. */
    data class ActiveLink(
        val token: String = "",
        val expiresAt: String = ""
    )

    companion object {
        private const val TOKEN_BYTE_COUNT = 16
        private const val MAX_TOKEN_LENGTH = 64
        private const val MAX_AUDIT_ENTRIES = 200L
        private const val RECENT_USES_LIMIT = 20L
        private val LOCK_TTL: Duration = Duration.ofSeconds(10)
    }

    /** Current link state, or null when no active link exists. Never mints. */
    suspend fun getActive(targetType: JoinRequestService.TargetType, targetId: UUID): InviteLinkResponse? {
        val json = redis.opsForValue()
            .get(RedisKeyManager.inviteActive(targetType.name, targetId))
            .awaitSingleOrNull() ?: return null
        val active = runCatching { objectMapper.readValue(json, ActiveLink::class.java) }.getOrNull()
            ?: return null
        if (active.token.isBlank()) return null
        return InviteLinkResponse(token = active.token, expiresAt = active.expiresAt)
    }

    /**
     * Mints a new link, revoking any previous one. Serialized per tenant by the
     * invite:lock mutex. The old token key is deleted BEFORE the new pair is
     * written so no partial failure can leave a revoked token resolvable.
     */
    suspend fun regenerate(targetType: JoinRequestService.TargetType, targetId: UUID): InviteLinkResponse {
        val lockKey = RedisKeyManager.inviteLock(targetType.name, targetId)
        val locked = redis.opsForValue().setIfAbsent(lockKey, "1", LOCK_TTL).awaitSingle()
        if (!locked) throw ConflictException("Invite link is already being regenerated")
        try {
            val activeKey = RedisKeyManager.inviteActive(targetType.name, targetId)
            val oldToken = redis.opsForValue().get(activeKey).awaitSingleOrNull()
                ?.let { runCatching { objectMapper.readValue(it, ActiveLink::class.java) }.getOrNull() }
                ?.token
                ?.takeIf { it.isNotBlank() }
            if (oldToken != null) {
                redis.delete(RedisKeyManager.inviteToken(oldToken)).awaitSingleOrNull()
            }

            val token = JoinCodeGenerator.generate(JoinCodeGenerator.INVITE_PREFIX, TOKEN_BYTE_COUNT)
            val expiresAt = OffsetDateTime.now().plus(RedisTTLPolicy.INVITE_LINK).toString()
            val payload = InviteTarget(
                targetType = targetType,
                targetId = targetId.toString(),
                expiresAt = expiresAt
            )
            redis.opsForValue()
                .set(
                    RedisKeyManager.inviteToken(token),
                    objectMapper.writeValueAsString(payload),
                    RedisTTLPolicy.INVITE_LINK
                )
                .awaitSingle()
            redis.opsForValue()
                .set(
                    activeKey,
                    objectMapper.writeValueAsString(ActiveLink(token = token, expiresAt = expiresAt)),
                    RedisTTLPolicy.INVITE_LINK
                )
                .awaitSingle()
            return InviteLinkResponse(token = token, expiresAt = expiresAt)
        } finally {
            redis.delete(lockKey).awaitSingleOrNull()
        }
    }

    /** Read-only token resolution — never consumes. Null when unknown/expired. */
    suspend fun resolve(token: String): InviteTarget? {
        if (!token.startsWith(JoinCodeGenerator.INVITE_PREFIX) || token.length > MAX_TOKEN_LENGTH) return null
        val json = redis.opsForValue().get(RedisKeyManager.inviteToken(token)).awaitSingleOrNull()
            ?: return null
        return runCatching { objectMapper.readValue(json, InviteTarget::class.java) }.getOrNull()
    }

    /** resolve() + display-name lookup. Null on ANY failure — backs the public endpoint's 404. */
    suspend fun resolveForDisplay(token: String): InviteResolveResponse? {
        val target = resolve(token) ?: return null
        val targetId = runCatching { UUID.fromString(target.targetId) }.getOrNull() ?: return null
        return when (target.targetType) {
            JoinRequestService.TargetType.ORG ->
                organizationRepository.findById(targetId)
                    ?.let { InviteResolveResponse(targetType = "ORG", name = it.name) }
            JoinRequestService.TargetType.STORE ->
                storeRepository.findById(targetId)
                    ?.takeIf { it.orgId == null }
                    ?.let { InviteResolveResponse(targetType = "STORE", name = it.name) }
        }
    }

    /** Best-effort usage-trail append — catches and logs every failure, never throws. */
    suspend fun recordUse(
        targetType: JoinRequestService.TargetType,
        targetId: UUID,
        username: String,
        token: String
    ) {
        try {
            val key = RedisKeyManager.inviteAudit(targetType.name, targetId)
            val now = OffsetDateTime.now()
            val entry = InviteLinkUse(
                username = username,
                usedAt = now.toString(),
                linkId = token.takeLast(4)
            )
            redis.opsForZSet()
                .add(key, objectMapper.writeValueAsString(entry), now.toInstant().toEpochMilli().toDouble())
                .awaitSingle()
            val cutoff = now.minus(RedisTTLPolicy.INVITE_AUDIT).toInstant().toEpochMilli().toDouble()
            redis.opsForZSet().removeRangeByScore(key, Range.closed(0.0, cutoff)).awaitSingle()
            redis.opsForZSet().removeRange(key, Range.closed(0L, -(MAX_AUDIT_ENTRIES + 1))).awaitSingle()
            redis.expire(key, RedisTTLPolicy.INVITE_AUDIT).awaitSingle()
        } catch (ex: CancellationException) {
            throw ex
        } catch (ex: Exception) {
            log.warn("Failed to record invite link use for {} {}: {}", targetType, targetId, ex.message)
        }
    }

    /** Newest ≤ 20 trail entries; lazily prunes >30-day entries; skips malformed members. */
    suspend fun getRecentUses(targetType: JoinRequestService.TargetType, targetId: UUID): List<InviteLinkUse> {
        val key = RedisKeyManager.inviteAudit(targetType.name, targetId)
        val cutoff = OffsetDateTime.now().minus(RedisTTLPolicy.INVITE_AUDIT).toInstant().toEpochMilli().toDouble()
        redis.opsForZSet().removeRangeByScore(key, Range.closed(0.0, cutoff)).awaitSingleOrNull()
        val members = redis.opsForZSet()
            .reverseRange(key, Range.closed(0L, RECENT_USES_LIMIT - 1))
            .collectList()
            .awaitSingle()
        return members.mapNotNull { member ->
            runCatching { objectMapper.readValue(member, InviteLinkUse::class.java) }.getOrNull()
        }
    }
}
```

Verified API facts (Context7 / spring-data-redis 3.5.9 jar): `opsForZSet().add(K, V, double): Mono<Boolean>`, `removeRangeByScore(K, Range<Double>): Mono<Long>` (ZREMRANGEBYSCORE), `removeRange(K, Range<Long>): Mono<Long>` (ZREMRANGEBYRANK — negative indices pass through; `Range.closed` does not validate bound ordering), `reverseRange(K, Range<Long>): Flux<V>` (ZREVRANGE), `ReactiveRedisTemplate.expire(K, Duration): Mono<Boolean>`, `opsForValue().setIfAbsent(K, V, Duration): Mono<Boolean>`. `Range.closed(0L, -201L)` = `ZREMRANGEBYRANK key 0 -201` = keep the 200 newest.

- [x] **Step 4: Run — expect pass**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.admin.service.InviteLinkServiceTest"`
Expected: `BUILD SUCCESSFUL`, 11 tests pass.

---

### Task 5: `InviteLinkService` — usage-trail tests (recordUse / getRecentUses)

The implementation already landed in Task 4 Step 3; this task pins its behavior with tests. (Test-after here is a deliberate trade-off to keep the service file written once; the behavior contract below is from spec § 4.6 and was fixed before implementation.)

**Files:**
- Test: `backend/src/test/kotlin/com/thomas/notiguide/domain/admin/service/InviteLinkServiceTest.kt`

- [x] **Step 1: Append the trail tests** to `InviteLinkServiceTest` (add `import org.springframework.data.domain.Range` and `import reactor.core.publisher.Flux` to the imports):

```kotlin
    @Test
    fun `recordUse appends then prunes then caps then refreshes the key TTL`() = runTest {
        val key = "invite:audit:STORE:$storeId"
        every { zSetOps.add(key, any(), any()) } returns Mono.just(true)
        every { zSetOps.removeRangeByScore(key, any()) } returns Mono.just(0L)
        every { zSetOps.removeRange(key, Range.closed(0L, -201L)) } returns Mono.just(0L)
        every { redis.expire(key, RedisTTLPolicy.INVITE_AUDIT) } returns Mono.just(true)

        service.recordUse(JoinRequestService.TargetType.STORE, storeId, "newjoiner", "i_abcd1234")

        verifyOrder {
            zSetOps.add(
                key,
                match { it.contains("\"username\":\"newjoiner\"") && it.contains("\"linkId\":\"1234\"") },
                any()
            )
            zSetOps.removeRangeByScore(key, any())
            zSetOps.removeRange(key, Range.closed(0L, -201L))
            redis.expire(key, RedisTTLPolicy.INVITE_AUDIT)
        }
    }

    @Test
    fun `recordUse never throws when redis fails`() = runTest {
        every { zSetOps.add(any(), any(), any()) } returns Mono.error(RuntimeException("redis down"))

        // must complete normally — a trail failure must never fail a registration
        service.recordUse(JoinRequestService.TargetType.ORG, orgId, "joiner", "i_abcd1234")
    }

    @Test
    fun `getRecentUses returns newest-first capped at 20 and skips malformed entries`() = runTest {
        val key = "invite:audit:ORG:$orgId"
        every { zSetOps.removeRangeByScore(key, any()) } returns Mono.just(0L)
        every { zSetOps.reverseRange(key, Range.closed(0L, 19L)) } returns Flux.just(
            """{"username":"newest","usedAt":"2026-06-11T10:00:00Z","linkId":"1234"}""",
            "not-json",
            """{"username":"older","usedAt":"2026-06-10T10:00:00Z","linkId":"5678"}"""
        )

        val uses = service.getRecentUses(JoinRequestService.TargetType.ORG, orgId)

        assertThat(uses).extracting("username").containsExactly("newest", "older")
        assertThat(uses).extracting("linkId").containsExactly("1234", "5678")
    }
```

- [x] **Step 2: Run — expect pass**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.admin.service.InviteLinkServiceTest"`
Expected: `BUILD SUCCESSFUL`, 14 tests pass.

---

### Task 6: Registration via invite token

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/request/RegisterRequest.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/service/RegistrationService.kt`
- Create: `backend/src/test/kotlin/com/thomas/notiguide/domain/admin/service/RegistrationServiceTest.kt`

Run GitNexus impact first: `gitnexus_impact({target: "RegistrationService", direction: "upstream"})` (expected direct caller: `AuthController.register`).

- [x] **Step 1: Write the failing tests.** Create `RegistrationServiceTest.kt`:

```kotlin
package com.thomas.notiguide.domain.admin.service

import com.thomas.notiguide.core.exception.HttpException
import com.thomas.notiguide.domain.admin.repository.AdminRepository
import com.thomas.notiguide.domain.admin.request.RegisterRequest
import com.thomas.notiguide.domain.admin.types.RegisterMode
import com.thomas.notiguide.domain.admin.types.RegisterStatus
import com.thomas.notiguide.domain.organization.entity.Organization
import com.thomas.notiguide.domain.organization.repository.OrganizationRepository
import com.thomas.notiguide.domain.store.entity.Store
import com.thomas.notiguide.domain.store.repository.StoreRepository
import com.thomas.notiguide.domain.store.service.StoreService
import io.mockk.coEvery
import io.mockk.coVerify
import io.mockk.every
import io.mockk.mockk
import kotlinx.coroutines.test.runTest
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.http.HttpStatus
import org.springframework.security.crypto.password.PasswordEncoder
import java.util.UUID

class RegistrationServiceTest {
    private val adminRepository = mockk<AdminRepository> {
        coEvery { existsByUsername(any()) } returns false
    }
    private val organizationRepository = mockk<OrganizationRepository>()
    private val storeRepository = mockk<StoreRepository>()
    private val storeService = mockk<StoreService>()
    private val passwordEncoder = mockk<PasswordEncoder> {
        every { encode(any()) } returns "hashed"
    }
    private val joinRequestService = mockk<JoinRequestService> {
        coEvery { usernameReserved(any()) } returns false
        coEvery { create(any(), any(), any(), any()) } returns "req-1"
    }
    private val inviteLinkService = mockk<InviteLinkService> {
        coEvery { recordUse(any(), any(), any(), any()) } returns Unit
    }
    private val service = RegistrationService(
        adminRepository,
        organizationRepository,
        storeRepository,
        storeService,
        passwordEncoder,
        joinRequestService,
        inviteLinkService
    )

    private val orgId = UUID.fromString("11111111-1111-1111-1111-111111111111")
    private val storeId = UUID.fromString("22222222-2222-2222-2222-222222222222")

    private fun joinRequest(joinCode: String? = null, inviteToken: String? = null) = RegisterRequest(
        mode = RegisterMode.JOIN,
        username = "newjoiner",
        password = "Sup3rSecret!1",
        joinCode = joinCode,
        inviteToken = inviteToken
    )

    private fun orgTarget() = InviteLinkService.InviteTarget(
        targetType = JoinRequestService.TargetType.ORG,
        targetId = orgId.toString(),
        expiresAt = "2026-06-18T10:00:00Z"
    )

    @Test
    fun `join via a valid org invite token files a pending request and records the use`() = runTest {
        coEvery { inviteLinkService.resolve("i_tok") } returns orgTarget()
        coEvery { organizationRepository.findById(orgId) } returns
            Organization(id = orgId, name = "Acme", joinCode = "o_x")

        val response = service.register(joinRequest(inviteToken = "i_tok"))

        assertThat(response.status).isEqualTo(RegisterStatus.PENDING)
        assertThat(response.targetType).isEqualTo("ORG")
        coVerify {
            joinRequestService.create("newjoiner", "hashed", JoinRequestService.TargetType.ORG, orgId)
        }
        coVerify {
            inviteLinkService.recordUse(JoinRequestService.TargetType.ORG, orgId, "newjoiner", "i_tok")
        }
    }

    @Test
    fun `join via a valid store invite token files a pending store request`() = runTest {
        coEvery { inviteLinkService.resolve("i_tok") } returns InviteLinkService.InviteTarget(
            targetType = JoinRequestService.TargetType.STORE,
            targetId = storeId.toString(),
            expiresAt = "2026-06-18T10:00:00Z"
        )
        coEvery { storeRepository.findById(storeId) } returns
            Store(id = storeId, orgId = null, name = "Indie Store")

        val response = service.register(joinRequest(inviteToken = "i_tok"))

        assertThat(response.status).isEqualTo(RegisterStatus.PENDING)
        assertThat(response.targetType).isEqualTo("STORE")
        coVerify {
            joinRequestService.create("newjoiner", "hashed", JoinRequestService.TargetType.STORE, storeId)
        }
    }

    @Test
    fun `join via an unknown or expired invite token is rejected with 400`() = runTest {
        coEvery { inviteLinkService.resolve("i_dead") } returns null

        val ex = runCatching { service.register(joinRequest(inviteToken = "i_dead")) }.exceptionOrNull()

        assertThat(ex).isInstanceOf(HttpException::class.java)
        assertThat((ex as HttpException).status).isEqualTo(HttpStatus.BAD_REQUEST)
        assertThat(ex.message).contains("invite link")
    }

    @Test
    fun `join with both a code and an invite token is rejected with 400`() = runTest {
        val ex = runCatching {
            service.register(joinRequest(joinCode = "s_code", inviteToken = "i_tok"))
        }.exceptionOrNull()

        assertThat(ex).isInstanceOf(HttpException::class.java)
        assertThat((ex as HttpException).status).isEqualTo(HttpStatus.BAD_REQUEST)
    }

    @Test
    fun `join with neither a code nor an invite token is rejected with 400`() = runTest {
        val ex = runCatching { service.register(joinRequest()) }.exceptionOrNull()

        assertThat(ex).isInstanceOf(HttpException::class.java)
        assertThat((ex as HttpException).status).isEqualTo(HttpStatus.BAD_REQUEST)
    }

    @Test
    fun `join via a store invite token whose store became org-owned is rejected with 400`() = runTest {
        coEvery { inviteLinkService.resolve("i_tok") } returns InviteLinkService.InviteTarget(
            targetType = JoinRequestService.TargetType.STORE,
            targetId = storeId.toString(),
            expiresAt = "2026-06-18T10:00:00Z"
        )
        coEvery { storeRepository.findById(storeId) } returns
            Store(id = storeId, orgId = orgId, name = "Captured Store")

        val ex = runCatching { service.register(joinRequest(inviteToken = "i_tok")) }.exceptionOrNull()

        assertThat(ex).isInstanceOf(HttpException::class.java)
        assertThat((ex as HttpException).status).isEqualTo(HttpStatus.BAD_REQUEST)
        coVerify(exactly = 0) { joinRequestService.create(any(), any(), any(), any()) }
    }

    @Test
    fun `join via an invite token whose target was deleted is rejected with 400`() = runTest {
        coEvery { inviteLinkService.resolve("i_tok") } returns orgTarget()
        coEvery { organizationRepository.findById(orgId) } returns null

        val ex = runCatching { service.register(joinRequest(inviteToken = "i_tok")) }.exceptionOrNull()

        assertThat(ex).isInstanceOf(HttpException::class.java)
        assertThat((ex as HttpException).status).isEqualTo(HttpStatus.BAD_REQUEST)
    }

    @Test
    fun `a recordUse failure does not fail the registration`() = runTest {
        coEvery { inviteLinkService.resolve("i_tok") } returns orgTarget()
        coEvery { organizationRepository.findById(orgId) } returns
            Organization(id = orgId, name = "Acme", joinCode = "o_x")
        coEvery { inviteLinkService.recordUse(any(), any(), any(), any()) } throws
            RuntimeException("redis down")

        val response = service.register(joinRequest(inviteToken = "i_tok"))

        assertThat(response.status).isEqualTo(RegisterStatus.PENDING)
    }

    @Test
    fun `join via a join code records nothing in the usage trail`() = runTest {
        coEvery { organizationRepository.findByJoinCode("o_code") } returns
            Organization(id = orgId, name = "Acme", joinCode = "o_code")

        val response = service.register(joinRequest(joinCode = "o_code"))

        assertThat(response.status).isEqualTo(RegisterStatus.PENDING)
        coVerify(exactly = 0) { inviteLinkService.recordUse(any(), any(), any(), any()) }
    }
}
```

(Token non-consumption on successful registration is pinned at the `InviteLinkService` layer — `resolve returns the target and never consumes the token`, Task 4 — since `RegistrationService` only sees a mock.)

- [x] **Step 2: Run — expect compilation failure**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.admin.service.RegistrationServiceTest"`
Expected: FAILS — `RegistrationService` has no 7th constructor parameter; `RegisterRequest` has no `inviteToken`.

- [x] **Step 3: Implement.** In `RegisterRequest.kt`, add a field after `joinCode`:

```kotlin
    @field:Size(max = 64)
    val inviteToken: String? = null
```

In `RegistrationService.kt`:

1. Add constructor parameter (after `joinRequestService`): `private val inviteLinkService: InviteLinkService`
2. Add imports: `kotlinx.coroutines.CancellationException`, `java.util.UUID`
3. Replace the `join` function and add two private helpers:

```kotlin
    private suspend fun join(username: String, passwordHash: String, request: RegisterRequest): RegisterResponse {
        val code = request.joinCode?.trim()?.takeIf { it.isNotBlank() }
        val inviteToken = request.inviteToken?.trim()?.takeIf { it.isNotBlank() }

        if (code != null && inviteToken != null)
            throw HttpException(HttpStatus.BAD_REQUEST, "Provide a join code or an invite link, not both")
        if (inviteToken != null)
            return joinViaInviteToken(username, passwordHash, inviteToken)
        if (code == null)
            throw HttpException(HttpStatus.BAD_REQUEST, "Provide a join code or an invite link")

        when {
            code.startsWith(JoinCodeGenerator.ORG_PREFIX) -> {
                val org = organizationRepository.findByJoinCode(code)
                    ?: throw HttpException(HttpStatus.BAD_REQUEST, "Invalid join code")
                joinRequestService.create(username, passwordHash, JoinRequestService.TargetType.ORG, org.id!!)
                return RegisterResponse(RegisterStatus.PENDING, AdminRole.ROLE_ADMIN, "ORG")
            }
            code.startsWith(JoinCodeGenerator.STORE_PREFIX) -> {
                val store = storeRepository.findByJoinCode(code)
                    ?: throw HttpException(HttpStatus.BAD_REQUEST, "Invalid join code")
                if (store.orgId != null)
                    throw HttpException(HttpStatus.BAD_REQUEST, "Invalid join code")
                joinRequestService.create(username, passwordHash, JoinRequestService.TargetType.STORE, store.id!!)
                return RegisterResponse(RegisterStatus.PENDING, AdminRole.ROLE_ADMIN, "STORE")
            }
            else -> throw HttpException(HttpStatus.BAD_REQUEST, "Invalid join code")
        }
    }

    private suspend fun joinViaInviteToken(username: String, passwordHash: String, token: String): RegisterResponse {
        val target = inviteLinkService.resolve(token)
            ?: throw HttpException(HttpStatus.BAD_REQUEST, "Invalid or expired invite link")
        val targetId = runCatching { UUID.fromString(target.targetId) }.getOrNull()
            ?: throw HttpException(HttpStatus.BAD_REQUEST, "Invalid or expired invite link")

        // findById(targetId) guarantees the entity id equals targetId, so the
        // non-null targetId is used for create/recordUse — no `!!` on entity ids.
        return when (target.targetType) {
            JoinRequestService.TargetType.ORG -> {
                organizationRepository.findById(targetId)
                    ?: throw HttpException(HttpStatus.BAD_REQUEST, "Invalid or expired invite link")
                joinRequestService.create(username, passwordHash, JoinRequestService.TargetType.ORG, targetId)
                recordUseSafely(JoinRequestService.TargetType.ORG, targetId, username, token)
                RegisterResponse(RegisterStatus.PENDING, AdminRole.ROLE_ADMIN, "ORG")
            }
            JoinRequestService.TargetType.STORE -> {
                val store = storeRepository.findById(targetId)
                    ?: throw HttpException(HttpStatus.BAD_REQUEST, "Invalid or expired invite link")
                if (store.orgId != null)
                    throw HttpException(HttpStatus.BAD_REQUEST, "Invalid or expired invite link")
                joinRequestService.create(username, passwordHash, JoinRequestService.TargetType.STORE, targetId)
                recordUseSafely(JoinRequestService.TargetType.STORE, targetId, username, token)
                RegisterResponse(RegisterStatus.PENDING, AdminRole.ROLE_ADMIN, "STORE")
            }
        }
    }

    /** Best-effort: the usage trail must never fail a registration (defense in depth on top of recordUse's own guarantee). */
    private suspend fun recordUseSafely(
        targetType: JoinRequestService.TargetType,
        targetId: UUID,
        username: String,
        token: String
    ) {
        try {
            inviteLinkService.recordUse(targetType, targetId, username, token)
        } catch (ex: CancellationException) {
            throw ex
        } catch (_: Exception) {
            // logged inside recordUse's own guard when it is the source; swallowed here by design
        }
    }
```

The token is never deleted or touched by registration (spec § 4.5).

- [x] **Step 4: Run — expect pass**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.admin.service.RegistrationServiceTest"`
Expected: `BUILD SUCCESSFUL`, 9 tests pass.

---

### Task 7: Organization invite-link endpoints

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/organization/controller/OrganizationController.kt`
- Create: `backend/src/test/kotlin/com/thomas/notiguide/domain/organization/controller/OrganizationControllerTest.kt`

Run GitNexus impact first: `gitnexus_impact({target: "OrganizationController", direction: "upstream"})`.

- [x] **Step 1: Write the failing tests.** Create `OrganizationControllerTest.kt` (first test file for this controller — follows the `QueueAdminControllerTest` slice pattern, but with named imports per CLAUDE.md):

```kotlin
package com.thomas.notiguide.domain.organization.controller

import com.ninjasquad.springmockk.MockkBean
import com.thomas.notiguide.core.jwt.JWTAuthFilter
import com.thomas.notiguide.core.ratelimit.RateLimitFilter
import com.thomas.notiguide.domain.admin.service.InviteLinkService
import com.thomas.notiguide.domain.admin.service.JoinRequestService
import com.thomas.notiguide.domain.admin.types.AdminRole
import com.thomas.notiguide.domain.organization.response.InviteLinkResponse
import com.thomas.notiguide.domain.organization.response.InviteLinkUse
import com.thomas.notiguide.domain.organization.service.OrganizationService
import com.thomas.notiguide.domain.store.repository.StoreRepository
import com.thomas.notiguide.support.TestPrincipals
import com.thomas.notiguide.support.TestSecurityConfig
import io.mockk.coEvery
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.reactive.WebFluxTest
import org.springframework.context.annotation.ComponentScan.Filter
import org.springframework.context.annotation.FilterType
import org.springframework.context.annotation.Import
import org.springframework.security.test.web.reactive.server.SecurityMockServerConfigurers.mockAuthentication
import org.springframework.test.web.reactive.server.WebTestClient
import java.util.UUID

@WebFluxTest(
    controllers = [OrganizationController::class],
    excludeFilters = [Filter(type = FilterType.ASSIGNABLE_TYPE, classes = [JWTAuthFilter::class, RateLimitFilter::class])],
)
@Import(TestSecurityConfig::class)
class OrganizationControllerTest {
    @MockkBean(relaxed = true) lateinit var organizationService: OrganizationService
    @MockkBean(relaxed = true) lateinit var storeRepository: StoreRepository
    @MockkBean lateinit var inviteLinkService: InviteLinkService

    @Autowired lateinit var client: WebTestClient

    private val orgId = UUID.randomUUID()

    @Test
    fun `GET invite-link returns the no-active-link state carrying the usage trail`() {
        coEvery { inviteLinkService.getActive(JoinRequestService.TargetType.ORG, orgId) } returns null
        coEvery { inviteLinkService.getRecentUses(JoinRequestService.TargetType.ORG, orgId) } returns listOf(
            InviteLinkUse(username = "joiner", usedAt = "2026-06-10T09:00:00Z", linkId = "abcd")
        )

        client.mutateWith(mockAuthentication(TestPrincipals.authToken(role = AdminRole.ROLE_SUPER_ADMIN, orgId = orgId)))
            .get().uri("/api/orgs/me/invite-link")
            .exchange()
            .expectStatus().isOk
            .expectBody()
            .jsonPath("$.token").isEmpty
            .jsonPath("$.expiresAt").isEmpty
            .jsonPath("$.recentUses[0].username").isEqualTo("joiner")
            .jsonPath("$.recentUses[0].linkId").isEqualTo("abcd")
    }

    @Test
    fun `GET invite-link is forbidden for a role-ADMIN principal`() {
        client.mutateWith(mockAuthentication(TestPrincipals.authToken(role = AdminRole.ROLE_ADMIN, storeId = UUID.randomUUID())))
            .get().uri("/api/orgs/me/invite-link")
            .exchange()
            .expectStatus().isForbidden
    }

    @Test
    fun `POST rotate returns the freshly minted link with the unchanged trail`() {
        coEvery { inviteLinkService.regenerate(JoinRequestService.TargetType.ORG, orgId) } returns
            InviteLinkResponse(token = "i_fresh", expiresAt = "2026-06-18T10:00:00Z")
        coEvery { inviteLinkService.getRecentUses(JoinRequestService.TargetType.ORG, orgId) } returns emptyList()

        client.mutateWith(mockAuthentication(TestPrincipals.authToken(role = AdminRole.ROLE_SUPER_ADMIN, orgId = orgId)))
            .post().uri("/api/orgs/me/invite-link/rotate")
            .exchange()
            .expectStatus().isOk
            .expectBody()
            .jsonPath("$.token").isEqualTo("i_fresh")
            .jsonPath("$.expiresAt").isEqualTo("2026-06-18T10:00:00Z")
    }

    @Test
    fun `POST rotate is forbidden for a role-ADMIN principal`() {
        client.mutateWith(mockAuthentication(TestPrincipals.authToken(role = AdminRole.ROLE_ADMIN, storeId = UUID.randomUUID())))
            .post().uri("/api/orgs/me/invite-link/rotate")
            .exchange()
            .expectStatus().isForbidden
    }
}
```

- [x] **Step 2: Run — expect failure**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.organization.controller.OrganizationControllerTest"`
Expected: FAILS — all four tests get 404 (the routes don't exist yet; the slice context itself starts fine because `@MockkBean` supplies the `InviteLinkService` bean).

- [x] **Step 3: Implement.** In `OrganizationController.kt`:

1. Constructor gains `private val inviteLinkService: InviteLinkService`.
2. Add named imports: `com.thomas.notiguide.domain.admin.service.InviteLinkService`, `com.thomas.notiguide.domain.admin.service.JoinRequestService`, `com.thomas.notiguide.domain.organization.response.InviteLinkResponse`.
3. Add below the join-code endpoints:

```kotlin
    @GetMapping("/me/invite-link")
    suspend fun getInviteLink(@AuthenticationPrincipal principal: AdminPrincipal): ResponseEntity<InviteLinkResponse> {
        val orgId = requireOrgOwner(principal)
        return ResponseEntity.ok(composeLinkResponse(orgId, inviteLinkService.getActive(JoinRequestService.TargetType.ORG, orgId)))
    }

    @PostMapping("/me/invite-link/rotate")
    suspend fun rotateInviteLink(@AuthenticationPrincipal principal: AdminPrincipal): ResponseEntity<InviteLinkResponse> {
        val orgId = requireOrgOwner(principal)
        return ResponseEntity.ok(composeLinkResponse(orgId, inviteLinkService.regenerate(JoinRequestService.TargetType.ORG, orgId)))
    }

    private suspend fun composeLinkResponse(orgId: UUID, link: InviteLinkResponse?): InviteLinkResponse {
        val base = link ?: InviteLinkResponse(token = null, expiresAt = null)
        return base.copy(recentUses = inviteLinkService.getRecentUses(JoinRequestService.TargetType.ORG, orgId))
    }
```

(`requireOrgOwner` already exists and returns the non-null `orgId`; its 403 message mentions "join codes" — left unchanged deliberately, the web only branches on the status code.)

- [x] **Step 4: Run — expect pass**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.organization.controller.OrganizationControllerTest"`
Expected: `BUILD SUCCESSFUL`, 4 tests pass.

---

### Task 8: Store invite-link endpoints

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/store/controller/StoreController.kt`
- Create: `backend/src/test/kotlin/com/thomas/notiguide/domain/store/controller/StoreControllerTest.kt`

Run GitNexus impact first: `gitnexus_impact({target: "StoreController", direction: "upstream"})`.

- [x] **Step 1: Write the failing tests.** Create `StoreControllerTest.kt`:

```kotlin
package com.thomas.notiguide.domain.store.controller

import com.ninjasquad.springmockk.MockkBean
import com.thomas.notiguide.core.exception.ForbiddenException
import com.thomas.notiguide.core.jwt.JWTAuthFilter
import com.thomas.notiguide.core.ratelimit.RateLimitFilter
import com.thomas.notiguide.domain.admin.service.InviteLinkService
import com.thomas.notiguide.domain.admin.service.JoinRequestService
import com.thomas.notiguide.domain.organization.response.InviteLinkResponse
import com.thomas.notiguide.domain.store.dto.StoreDto
import com.thomas.notiguide.domain.store.service.StoreService
import com.thomas.notiguide.shared.principal.StoreAccessService
import com.thomas.notiguide.support.TestPrincipals
import com.thomas.notiguide.support.TestSecurityConfig
import io.mockk.coEvery
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.reactive.WebFluxTest
import org.springframework.context.annotation.ComponentScan.Filter
import org.springframework.context.annotation.FilterType
import org.springframework.context.annotation.Import
import org.springframework.security.test.web.reactive.server.SecurityMockServerConfigurers.mockAuthentication
import org.springframework.test.web.reactive.server.WebTestClient
import java.util.UUID

@WebFluxTest(
    controllers = [StoreController::class],
    excludeFilters = [Filter(type = FilterType.ASSIGNABLE_TYPE, classes = [JWTAuthFilter::class, RateLimitFilter::class])],
)
@Import(TestSecurityConfig::class)
class StoreControllerTest {
    @MockkBean lateinit var storeService: StoreService
    @MockkBean lateinit var storeAccess: StoreAccessService
    @MockkBean lateinit var inviteLinkService: InviteLinkService

    @Autowired lateinit var client: WebTestClient

    private val storeId = UUID.randomUUID()

    private fun storeDto(orgId: UUID?) = StoreDto(
        id = storeId,
        publicId = "st_pub",
        orgId = orgId,
        name = "Store",
        address = null,
        isActive = true,
        allowJumpCall = false,
        allowNoShow = false,
        createdAt = null,
        updatedAt = null
    )

    @Test
    fun `GET invite-link returns 200 for an independent store`() {
        coEvery { storeAccess.requireStoreAccess(any(), storeId) } returns Unit
        coEvery { storeService.getStore(storeId) } returns storeDto(orgId = null)
        coEvery { inviteLinkService.getActive(JoinRequestService.TargetType.STORE, storeId) } returns
            InviteLinkResponse(token = "i_live", expiresAt = "2026-06-18T10:00:00Z")
        coEvery { inviteLinkService.getRecentUses(JoinRequestService.TargetType.STORE, storeId) } returns
            emptyList()

        client.mutateWith(mockAuthentication(TestPrincipals.authToken(storeId = storeId)))
            .get().uri("/api/stores/$storeId/invite-link")
            .exchange()
            .expectStatus().isOk
            .expectBody()
            .jsonPath("$.token").isEqualTo("i_live")
    }

    @Test
    fun `GET invite-link returns 409 for an org-owned store`() {
        coEvery { storeAccess.requireStoreAccess(any(), storeId) } returns Unit
        coEvery { storeService.getStore(storeId) } returns storeDto(orgId = UUID.randomUUID())

        client.mutateWith(mockAuthentication(TestPrincipals.authToken(storeId = storeId)))
            .get().uri("/api/stores/$storeId/invite-link")
            .exchange()
            .expectStatus().isEqualTo(409)
    }

    @Test
    fun `POST rotate returns 409 for an org-owned store`() {
        coEvery { storeAccess.requireStoreAccess(any(), storeId) } returns Unit
        coEvery { storeService.getStore(storeId) } returns storeDto(orgId = UUID.randomUUID())

        client.mutateWith(mockAuthentication(TestPrincipals.authToken(storeId = storeId)))
            .post().uri("/api/stores/$storeId/invite-link/rotate")
            .exchange()
            .expectStatus().isEqualTo(409)
    }

    @Test
    fun `POST rotate returns 403 when store access is denied`() {
        coEvery { storeAccess.requireStoreAccess(any(), storeId) } throws ForbiddenException("forbidden")

        client.mutateWith(mockAuthentication(TestPrincipals.authToken(storeId = storeId)))
            .post().uri("/api/stores/$storeId/invite-link/rotate")
            .exchange()
            .expectStatus().isForbidden
    }

    @Test
    fun `POST rotate returns the freshly minted link for an independent store`() {
        coEvery { storeAccess.requireStoreAccess(any(), storeId) } returns Unit
        coEvery { storeService.getStore(storeId) } returns storeDto(orgId = null)
        coEvery { inviteLinkService.regenerate(JoinRequestService.TargetType.STORE, storeId) } returns
            InviteLinkResponse(token = "i_fresh", expiresAt = "2026-06-18T10:00:00Z")
        coEvery { inviteLinkService.getRecentUses(JoinRequestService.TargetType.STORE, storeId) } returns
            emptyList()

        client.mutateWith(mockAuthentication(TestPrincipals.authToken(storeId = storeId)))
            .post().uri("/api/stores/$storeId/invite-link/rotate")
            .exchange()
            .expectStatus().isOk
            .expectBody()
            .jsonPath("$.token").isEqualTo("i_fresh")
    }
}
```

- [x] **Step 2: Run — expect failure**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.store.controller.StoreControllerTest"`
Expected: FAILS — all five tests get 404 (routes not mapped yet; the context starts since `@MockkBean` supplies the bean).

- [x] **Step 3: Implement.** In `StoreController.kt`:

1. Constructor gains `private val inviteLinkService: InviteLinkService`.
2. Add named imports: `com.thomas.notiguide.core.exception.ConflictException`, `com.thomas.notiguide.domain.admin.service.InviteLinkService`, `com.thomas.notiguide.domain.admin.service.JoinRequestService`, `com.thomas.notiguide.domain.organization.response.InviteLinkResponse`.
3. Add below the join-code endpoints:

```kotlin
    @GetMapping("/{id}/invite-link")
    suspend fun getStoreInviteLink(
        @PathVariable id: UUID,
        @AuthenticationPrincipal principal: AdminPrincipal
    ): ResponseEntity<InviteLinkResponse> {
        storeAccess.requireStoreAccess(principal, id)
        requireIndependentStore(id)
        return ResponseEntity.ok(
            composeLinkResponse(id, inviteLinkService.getActive(JoinRequestService.TargetType.STORE, id))
        )
    }

    @PostMapping("/{id}/invite-link/rotate")
    suspend fun rotateStoreInviteLink(
        @PathVariable id: UUID,
        @AuthenticationPrincipal principal: AdminPrincipal
    ): ResponseEntity<InviteLinkResponse> {
        storeAccess.requireStoreAccess(principal, id)
        requireIndependentStore(id)
        return ResponseEntity.ok(
            composeLinkResponse(id, inviteLinkService.regenerate(JoinRequestService.TargetType.STORE, id))
        )
    }

    /** Org-owned stores are joined via the organization — same 409 as the join-code endpoints. */
    private suspend fun requireIndependentStore(id: UUID) {
        if (storeService.getStore(id).orgId != null)
            throw ConflictException("Org-owned stores are joined via the organization code")
    }

    private suspend fun composeLinkResponse(storeId: UUID, link: InviteLinkResponse?): InviteLinkResponse {
        val base = link ?: InviteLinkResponse(token = null, expiresAt = null)
        return base.copy(recentUses = inviteLinkService.getRecentUses(JoinRequestService.TargetType.STORE, storeId))
    }
```

- [x] **Step 4: Run — expect pass**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.store.controller.StoreControllerTest"`
Expected: `BUILD SUCCESSFUL`, 5 tests pass.

---

### Task 9: Public resolve endpoint + full backend test sweep

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/admin/controller/AuthController.kt`
- Modify: `backend/src/test/kotlin/com/thomas/notiguide/domain/admin/controller/AuthControllerTest.kt`

Run GitNexus impact first: `gitnexus_impact({target: "AuthController", direction: "upstream"})`.

The endpoint is public automatically: `SecurityConfig` permits `/api/auth/**`, and `RateLimitFilter.resolveTier` already puts every `/api/auth/…` path in the auth tier — no security or rate-limit changes needed.

- [x] **Step 1: Write the failing tests.** In `AuthControllerTest.kt`, add a 13th `@MockkBean` line beneath the existing twelve (matching that file's existing declaration style):

```kotlin
    @MockkBean(relaxed = true) lateinit var inviteLinkService: com.thomas.notiguide.domain.admin.service.InviteLinkService
```

and append two tests (add named imports `io.mockk.coEvery` and `com.thomas.notiguide.domain.admin.response.InviteResolveResponse`):

```kotlin
    @Test
    fun `GET invite resolve returns the target display info for a valid token`() {
        coEvery { inviteLinkService.resolveForDisplay("i_tok") } returns
            InviteResolveResponse(targetType = "STORE", name = "Acme Store")

        client.get().uri("/api/auth/invite/i_tok")
            .exchange()
            .expectStatus().isOk
            .expectBody()
            .jsonPath("$.targetType").isEqualTo("STORE")
            .jsonPath("$.name").isEqualTo("Acme Store")
    }

    @Test
    fun `GET invite resolve returns 404 for an unknown or expired token`() {
        coEvery { inviteLinkService.resolveForDisplay("i_dead") } returns null

        client.get().uri("/api/auth/invite/i_dead")
            .exchange()
            .expectStatus().isNotFound
    }
```

- [x] **Step 2: Run — expect failure**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.admin.controller.AuthControllerTest"`
Expected: FAILS — both new tests get 404 with an error body that doesn't match the expectations (route not mapped yet); the pre-existing login test still passes.

- [x] **Step 3: Implement.** In `AuthController.kt`:

1. Constructor gains `private val inviteLinkService: InviteLinkService`.
2. Add named imports: `com.thomas.notiguide.core.exception.HttpException`, `com.thomas.notiguide.domain.admin.response.InviteResolveResponse`, `com.thomas.notiguide.domain.admin.service.InviteLinkService`, `org.springframework.web.bind.annotation.GetMapping`, `org.springframework.web.bind.annotation.PathVariable`.
3. Add the endpoint after `register`:

```kotlin
    @GetMapping("/invite/{token}")
    suspend fun resolveInvite(@PathVariable token: String): ResponseEntity<InviteResolveResponse> {
        // Public, read-only, never mints or consumes. Unknown and expired are
        // indistinguishable by design; the message never echoes the token.
        val resolved = inviteLinkService.resolveForDisplay(token)
            ?: throw HttpException(HttpStatus.NOT_FOUND, "Invite link is invalid or has expired")
        return ResponseEntity.ok(resolved)
    }
```

- [x] **Step 4: Run the class, then the full backend suite — expect pass**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.admin.controller.AuthControllerTest"`
Expected: `BUILD SUCCESSFUL`.

Run: `cd backend && ./gradlew test`
Expected: `BUILD SUCCESSFUL` — the whole suite, proving no existing test regressed (constructor changes to `RegistrationService`/controllers are covered by springmockk in the slices).

---

### Task 10: Web — types, API routes, API functions, `status` fix

**Files:**
- Modify: `web/src/types/admin.ts`
- Modify: `web/src/types/organization.ts`
- Modify: `web/src/lib/constants.ts`
- Modify: `web/src/features/organization/api.ts`
- Modify: `web/src/features/auth/api.ts`
- Modify: `web/src/app/[locale]/(auth)/register/page.tsx` (one line — keeps the tree type-correct until Task 12 replaces it)

- [x] **Step 1:** In `types/admin.ts`:

1. Add to `RegisterRequest` (after `joinCode?: string;`):

```ts
  inviteToken?: string;
```

2. **Pre-existing fix (spec § 5.1):** rename the `RegisterResponse` field `outcome` → `status` (the backend has always serialized `status`; the `outcome` field never matched and the PENDING branch never fired). Keep the `RegisterOutcome` alias name:

```ts
export interface RegisterResponse {
  status: RegisterOutcome;
  role: AdminRole | null;
  targetType: "ORG" | "STORE" | null;
}
```

3. Add beside `RegisterResponse`:

```ts
export interface InviteResolveResponse {
  targetType: "ORG" | "STORE";
  name: string;
}
```

- [x] **Step 2:** In `app/[locale]/(auth)/register/page.tsx` line 31, change `res.outcome === "PENDING"` to `res.status === "PENDING"`.

- [x] **Step 3:** In `types/organization.ts`, add below `JoinCodeResponse`:

```ts
export interface InviteLinkUse {
  username: string;
  usedAt: string;
  linkId: string;
}

export interface InviteLinkState {
  token: string | null;
  expiresAt: string | null;
  recentUses: InviteLinkUse[];
}
```

- [x] **Step 4:** In `lib/constants.ts`:

1. In `AUTH`, after `REGISTER`:

```ts
    INVITE: (token: string) => `/api/auth/invite/${encodeURIComponent(token)}`,
```

2. In `ORGS`, after `JOIN_CODE_ROTATE`:

```ts
    INVITE_LINK: "/api/orgs/me/invite-link",
    INVITE_LINK_ROTATE: "/api/orgs/me/invite-link/rotate",
```

3. In `STORES`, after `JOIN_CODE_ROTATE`:

```ts
    INVITE_LINK: (id: string) => `/api/stores/${id}/invite-link`,
    INVITE_LINK_ROTATE: (id: string) => `/api/stores/${id}/invite-link/rotate`,
```

- [x] **Step 5:** In `features/organization/api.ts`, extend the type import to include `InviteLinkState` and append:

```ts
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

- [x] **Step 6:** In `features/auth/api.ts`, add `get` to the `@/lib/api` import, `InviteResolveResponse` to the `@/types/admin` import, and append (after `register`):

```ts
export function resolveInvite(token: string) {
  return get<InviteResolveResponse>(API_ROUTES.AUTH.INVITE(token), {
    skipAuth: true,
  });
}
```

(`skipAuth: true` — public endpoint; throws `ApiError` with `code: 404` for unknown/expired tokens, which the wizard catches.)

- [x] **Step 7: Verify**

Run: `cd web && yarn lint`
Expected: no new errors. Then `yarn format` to normalize import order.

---

### Task 11: Web — i18n keys (EN + VI) and terminology cleanup

**Files:**
- Modify: `web/src/messages/en.json`
- Modify: `web/src/messages/vi.json`

Both files must change in the same task — `src/messages/parity.test.ts` fails on any key or rich-tag mismatch. The VI copy below was drafted for approval at plan review; use the user-approved wording if it differs.

- [x] **Step 1:** In `en.json`, `register` namespace — rework the four "invite code" strings to say "join code" (keys unchanged, spec § 5.5 terminology cleanup) and add two keys:

```json
"pathJoinDesc": "Join an existing store or organization using a join code.",
"joinCodeLabel": "Join code",
"joinCodeRequired": "A join code is required",
"invalidJoinCode": "That join code is invalid",
"joiningTarget": "You're joining <bold>{name}</bold>",
"inviteInvalid": "This invite link is invalid or has expired. You can still join with a join code."
```

- [x] **Step 2:** In `en.json`, `admins` namespace — rework one string and add thirteen:

```json
"joinCodeTitle": "Join code",
"inviteLinkTitle": "Invite link",
"inviteLinkDesc": "Generate a shareable link that lets people request to join without typing a code. Links expire after 7 days.",
"inviteLinkEmpty": "No active invite link. Generate one to share a tappable invitation.",
"inviteLinkGenerate": "Generate invite link",
"inviteLinkRegenerate": "Regenerate link",
"inviteLinkRegenerateConfirm": "Regenerating immediately invalidates the current link for everyone holding it. Continue?",
"inviteLinkCopy": "Copy link",
"inviteLinkCopied": "Link copied",
"inviteLinkValidUntil": "Valid until {date}",
"inviteLinkCaution": "Anyone with this link can request to join until it expires.",
"inviteLinkUsageTitle": "Recent joins via invite link",
"inviteLinkUsageDesc": "Join requests submitted through invite links in the last 30 days.",
"inviteLinkUsageCurrent": "Current link"
```

- [x] **Step 3:** In `vi.json`, mirror structurally (same keys, same `<bold>` tag in `joiningTarget`). Reworked terminology — "mã tham gia" (join code) vs "link mời" (invite link):

`register` namespace:

```json
"pathJoinDesc": "Tham gia cửa hàng hoặc tổ chức có sẵn bằng mã tham gia.",
"joinCodeLabel": "Mã tham gia",
"joinCodeRequired": "Cần nhập mã tham gia",
"invalidJoinCode": "Mã tham gia không hợp lệ",
"joiningTarget": "Bạn đang tham gia <bold>{name}</bold>",
"inviteInvalid": "Link mời không hợp lệ hoặc đã hết hạn. Bạn vẫn có thể tham gia bằng mã tham gia."
```

`admins` namespace:

```json
"joinCodeTitle": "Mã tham gia",
"inviteLinkTitle": "Link mời",
"inviteLinkDesc": "Tạo link chia sẻ để mọi người gửi yêu cầu tham gia mà không cần nhập mã. Link hết hạn sau 7 ngày.",
"inviteLinkEmpty": "Chưa có link mời nào đang hoạt động. Tạo link để chia sẻ lời mời.",
"inviteLinkGenerate": "Tạo link mời",
"inviteLinkRegenerate": "Tạo link mới",
"inviteLinkRegenerateConfirm": "Tạo link mới sẽ vô hiệu hóa link hiện tại ngay lập tức với mọi người đang giữ link. Tiếp tục?",
"inviteLinkCopy": "Sao chép link",
"inviteLinkCopied": "Đã sao chép link",
"inviteLinkValidUntil": "Có hiệu lực đến {date}",
"inviteLinkCaution": "Ai có link này đều có thể gửi yêu cầu tham gia cho đến khi link hết hạn.",
"inviteLinkUsageTitle": "Lượt tham gia gần đây qua link mời",
"inviteLinkUsageDesc": "Các yêu cầu tham gia gửi qua link mời trong 30 ngày qua.",
"inviteLinkUsageCurrent": "Link hiện tại"
```

- [x] **Step 4: Verify parity**

Run: `cd web && yarn test`
Expected: `parity.test.ts` passes (identical key sets, matching `<bold>` tags for `joiningTarget`).

---

### Task 12: Web — `RegisterWizard` extraction, Suspense, invite mode, join-form `invite` prop

**Files:**
- Create: `web/src/features/auth/register-wizard.tsx`
- Modify: `web/src/app/[locale]/(auth)/register/page.tsx`
- Modify: `web/src/features/auth/register-join-form.tsx`

Run GitNexus impact first: `gitnexus_impact({target: "RegisterJoinForm", direction: "upstream"})`.

Why the extraction (verified against Next.js 16.1 docs via Context7): in a statically rendered route, `useSearchParams` client-side-renders the tree up to the closest `<Suspense>` boundary, and without one the build fails. The page keeps the decorative shell (statically renderable); the wizard reads `?invite=`.

- [x] **Step 1:** Create `features/auth/register-wizard.tsx` — the page's view state machine moved verbatim, plus invite mode:

```tsx
"use client";

import { AlertTriangle, Loader2 } from "lucide-react";
import { useLocale, useTranslations } from "next-intl";
import { useSearchParams } from "next/navigation";
import { useEffect, useState } from "react";
import { register, resolveInvite } from "@/features/auth/api";
import { RegisterCreateOrgForm } from "@/features/auth/register-create-org-form";
import { RegisterCreateStoreForm } from "@/features/auth/register-create-store-form";
import { RegisterJoinForm } from "@/features/auth/register-join-form";
import { RegisterPathSelector } from "@/features/auth/register-path-selector";
import { RegisterPendingScreen } from "@/features/auth/register-pending-screen";
import { Link } from "@/i18n/navigation";
import type { RegisterMode, RegisterRequest } from "@/types/admin";
import { ApiError } from "@/types/api";

type View = "select" | RegisterMode | "active" | "pending";

type InviteState =
  | { kind: "none" }
  | { kind: "loading"; token: string }
  | { kind: "resolved"; token: string; targetName: string }
  | { kind: "invalid" };

export function RegisterWizard() {
  const locale = useLocale();
  const t = useTranslations("register");
  const tAuth = useTranslations("auth");
  const searchParams = useSearchParams();
  const inviteToken = searchParams.get("invite");

  const [view, setView] = useState<View>(inviteToken ? "JOIN" : "select");
  const [submitting, setSubmitting] = useState(false);
  const [invite, setInvite] = useState<InviteState>(
    inviteToken ? { kind: "loading", token: inviteToken } : { kind: "none" },
  );

  useEffect(() => {
    if (!inviteToken) return;
    let active = true;
    resolveInvite(inviteToken)
      .then((res) => {
        if (active)
          setInvite({
            kind: "resolved",
            token: inviteToken,
            targetName: res.name,
          });
      })
      .catch(() => {
        // 404 (unknown/expired/deleted target) and network failures share the
        // same fallback: warning + manual join-code form (spec § 6).
        if (active) setInvite({ kind: "invalid" });
      });
    return () => {
      active = false;
    };
  }, [inviteToken]);

  async function submit(request: RegisterRequest) {
    setSubmitting(true);
    try {
      const res = await register(request);
      setView(res.status === "PENDING" ? "pending" : "active");
      return null;
    } catch (err) {
      // The token died between page load and submit: clear the invite state so
      // every subsequent submit is a plain manual-code JOIN submit (spec § 5.4).
      if (
        err instanceof ApiError &&
        err.code === 400 &&
        err.message.toLowerCase().includes("invite")
      ) {
        setInvite({ kind: "invalid" });
      }
      throw err;
    } finally {
      setSubmitting(false);
    }
  }

  const resolvingInvite = view === "JOIN" && invite.kind === "loading";

  return (
    <>
      {view === "select" && (
        <RegisterPathSelector onSelect={(m) => setView(m)} />
      )}
      {view === "CREATE_ORG" && (
        <RegisterCreateOrgForm
          submitting={submitting}
          onBack={() => setView("select")}
          onSubmit={submit}
        />
      )}
      {view === "CREATE_STORE" && (
        <RegisterCreateStoreForm
          submitting={submitting}
          onBack={() => setView("select")}
          onSubmit={submit}
        />
      )}
      {resolvingInvite && (
        <div className="flex justify-center py-8">
          <Loader2
            aria-hidden="true"
            className="size-5 animate-spin text-muted-foreground"
          />
        </div>
      )}
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
          <RegisterJoinForm
            submitting={submitting}
            onBack={() => setView("select")}
            onSubmit={submit}
            invite={
              invite.kind === "resolved"
                ? { token: invite.token, targetName: invite.targetName }
                : undefined
            }
          />
        </div>
      )}
      {view === "active" && <RegisterPendingScreen variant="active" />}
      {view === "pending" && <RegisterPendingScreen variant="pending" />}

      {(view === "select" ||
        view === "CREATE_ORG" ||
        view === "CREATE_STORE" ||
        view === "JOIN") && (
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
    </>
  );
}
```

- [x] **Step 2:** Replace `app/[locale]/(auth)/register/page.tsx` — shell only, wizard inside `<Suspense>` (fallback `null` per spec § 5.4, it flashes for milliseconds):

```tsx
"use client";

import { Ticket } from "lucide-react";
import { useTranslations } from "next-intl";
import { Suspense } from "react";
import { LanguageSwitcher } from "@/components/layout/language-switcher";
import { ThemeToggle } from "@/components/layout/theme-toggle";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { RegisterWizard } from "@/features/auth/register-wizard";

export default function RegisterPage() {
  const t = useTranslations("register");

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
          <Suspense fallback={null}>
            <RegisterWizard />
          </Suspense>
        </CardContent>
      </Card>
    </div>
  );
}
```

- [x] **Step 3:** Replace `features/auth/register-join-form.tsx` — adds the optional `invite` prop (code field hidden, token submitted, "You're joining {name}" success-pattern alert):

```tsx
"use client";

import { useTranslations } from "next-intl";
import { useState } from "react";
import { InlineError } from "@/components/ui/inline-error";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import {
  RegisterFormActions,
  type RegisterFormProps,
  RegisterPasswordField,
  RegisterUsernameField,
  useRegisterForm,
} from "@/features/auth/register-form-shared";

interface RegisterJoinFormProps extends RegisterFormProps {
  /** Set when the joiner arrived through an invite link: hides the code
   * field and submits the token instead of a join code. */
  invite?: { token: string; targetName: string };
}

export function RegisterJoinForm({
  submitting,
  onBack,
  onSubmit,
  invite,
}: RegisterJoinFormProps) {
  const t = useTranslations("register");
  const [joinCode, setJoinCode] = useState("");
  const {
    username,
    setUsername,
    password,
    setPassword,
    errors,
    buildSubmitHandler,
  } = useRegisterForm(submitting, onSubmit);

  const handleSubmit = buildSubmitHandler({
    buildRequest: () =>
      invite
        ? { mode: "JOIN", username, password, inviteToken: invite.token }
        : { mode: "JOIN", username, password, joinCode: joinCode.trim() },
    validateExtra: (e) => {
      if (!invite && !joinCode.trim()) e.joinCode = t("joinCodeRequired");
    },
    mapApiError: (err) => {
      const message = err.message.toLowerCase();
      if (err.code === 400 && message.includes("invite"))
        return { joinCode: t("inviteInvalid") };
      if (message.includes("join code"))
        return { joinCode: t("invalidJoinCode") };
      return undefined;
    },
  });

  return (
    <form onSubmit={handleSubmit} noValidate className="space-y-4">
      {invite && (
        <div className="rounded-xl border border-success/40 bg-success/10 px-3.5 py-3 text-sm text-success dark:border-success/50 dark:bg-success/15">
          {t.rich("joiningTarget", {
            name: invite.targetName,
            bold: (chunks) => <span className="font-semibold">{chunks}</span>,
          })}
        </div>
      )}
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
      {!invite && (
        <div className="space-y-2">
          <Label htmlFor="reg-joincode">{t("joinCodeLabel")}</Label>
          <Input
            id="reg-joincode"
            value={joinCode}
            maxLength={64}
            placeholder={t("joinCodePlaceholder")}
            className="font-mono"
            onChange={(e) => setJoinCode(e.target.value)}
            aria-invalid={!!errors.joinCode}
          />
          {errors.joinCode && (
            <InlineError message={errors.joinCode} className="mt-1" />
          )}
        </div>
      )}
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

Expired-mid-flow flow check (spec § 5.4 + § 6): submit with a dead token → backend 400 "Invalid or expired invite link" → wizard's `submit` catch clears invite state (prop becomes `undefined`) → form's `mapApiError` maps the same error to `errors.joinCode = inviteInvalid` → re-render shows the manual code field with the inline error beneath it. Both checks use the identical condition (`code === 400 && includes("invite")`).

- [x] **Step 4: Verify**

Run: `cd web && yarn lint && yarn format`
Expected: clean. (No `yarn build` — the Suspense requirement is exercised by the project's audit flow.)

---

### Task 13: Web — `InviteLinkPanel` component

**Files:**
- Create: `web/src/features/organization/invite-link-panel.tsx`

Sibling of `JoinCodePanel` (same card chrome, same dependency-injection props shape, same `AlertDialog` confirm pattern), plus the usage-trail block. Uses only existing project components (`Input`, `Button`, `Badge`, `AlertDialog`). Warning boxes follow the canonical pattern from Web Styles § Status Alert Boxes.

- [x] **Step 1:** Create the component:

```tsx
"use client";

import { AlertTriangle, Copy, Loader2, RefreshCw } from "lucide-react";
import { useLocale, useTranslations } from "next-intl";
import { useEffect, useState } from "react";
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
import { Badge } from "@/components/ui/badge";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import type { InviteLinkState } from "@/types/organization";

interface InviteLinkPanelProps {
  fetchLink: () => Promise<InviteLinkState>;
  generateLink: () => Promise<InviteLinkState>;
}

// Time display is language-agnostic across the app: fixed locale, not next-intl
// (same pattern as waiting-list.tsx / serving-display.tsx).
const expiryFormatter = new Intl.DateTimeFormat("en-US", {
  year: "numeric",
  month: "short",
  day: "numeric",
  hour: "2-digit",
  minute: "2-digit",
  hour12: true,
});

const usedAtFormatter = new Intl.DateTimeFormat("en-US", {
  month: "short",
  day: "numeric",
  hour: "2-digit",
  minute: "2-digit",
  hour12: true,
});

export function InviteLinkPanel({
  fetchLink,
  generateLink,
}: InviteLinkPanelProps) {
  const tAdmins = useTranslations("admins");
  const tCommon = useTranslations("common");
  const locale = useLocale();
  const [origin, setOrigin] = useState("");
  const [link, setLink] = useState<InviteLinkState | null>(null);
  const [loading, setLoading] = useState(true);
  const [working, setWorking] = useState(false);
  const [confirmOpen, setConfirmOpen] = useState(false);

  useEffect(() => {
    // window is unavailable during prerender; resolve the origin after mount.
    setOrigin(window.location.origin);
  }, []);

  useEffect(() => {
    fetchLink()
      .then(setLink)
      .catch(() => {})
      .finally(() => setLoading(false));
  }, [fetchLink]);

  const inviteUrl =
    link?.token && origin
      ? `${origin}/${locale}/register?invite=${link.token}`
      : "";

  async function copy() {
    await navigator.clipboard.writeText(inviteUrl);
    toast.success(tAdmins("inviteLinkCopied"));
  }

  async function generate() {
    setWorking(true);
    try {
      setLink(await generateLink());
    } finally {
      setWorking(false);
      setConfirmOpen(false);
    }
  }

  const currentLinkId = link?.token ? link.token.slice(-4) : null;

  return (
    <div className="rounded-xl border border-border bg-card p-4">
      <h2 className="text-sm font-semibold">{tAdmins("inviteLinkTitle")}</h2>
      <p className="mt-1 mb-3 text-sm text-muted-foreground">
        {tAdmins("inviteLinkDesc")}
      </p>

      {loading ? (
        <div className="flex justify-center py-4">
          <Loader2
            aria-hidden="true"
            className="size-4 animate-spin text-muted-foreground"
          />
        </div>
      ) : link?.token && link.expiresAt ? (
        <>
          <div className="flex gap-2">
            <Input readOnly value={inviteUrl} className="font-mono" />
            <Button
              type="button"
              variant="outline"
              size="icon"
              onClick={copy}
              aria-label={tAdmins("inviteLinkCopy")}
            >
              <Copy className="size-4" />
            </Button>
            <Button
              type="button"
              variant="outline"
              size="icon"
              onClick={() => setConfirmOpen(true)}
              aria-label={tAdmins("inviteLinkRegenerate")}
            >
              <RefreshCw className="size-4" />
            </Button>
          </div>
          <p className="mt-2 text-xs text-muted-foreground">
            {tAdmins("inviteLinkValidUntil", {
              date: expiryFormatter.format(new Date(link.expiresAt)),
            })}
          </p>
          <div className="mt-3 flex items-start gap-2.5 rounded-xl border border-warning/40 bg-warning/15 px-3.5 py-3 text-sm text-warning dark:border-warning/50 dark:bg-warning/20">
            <AlertTriangle
              aria-hidden="true"
              className="mt-0.5 size-4 shrink-0"
            />
            <span>{tAdmins("inviteLinkCaution")}</span>
          </div>
        </>
      ) : (
        <div className="flex flex-col items-start gap-3">
          <p className="text-sm text-muted-foreground">
            {tAdmins("inviteLinkEmpty")}
          </p>
          <Button
            type="button"
            disabled={working}
            onClick={() => void generate()}
            className="bg-primary text-primary-foreground hover:bg-primary-hover"
          >
            {working && (
              <Loader2
                aria-hidden="true"
                className="mr-2 size-4 animate-spin"
              />
            )}
            {tAdmins("inviteLinkGenerate")}
          </Button>
        </div>
      )}

      {!loading && link && link.recentUses.length > 0 && (
        <div className="mt-4 border-t border-border pt-3">
          <h3 className="text-sm font-semibold">
            {tAdmins("inviteLinkUsageTitle")}
          </h3>
          <p className="mt-0.5 mb-2 text-xs text-muted-foreground">
            {tAdmins("inviteLinkUsageDesc")}
          </p>
          <div className="space-y-1.5">
            {link.recentUses.map((use) => {
              const isCurrent =
                currentLinkId !== null && use.linkId === currentLinkId;
              return (
                <div
                  key={`${use.usedAt}-${use.username}-${use.linkId}`}
                  className="flex items-center justify-between gap-3 text-sm"
                >
                  <span className="truncate font-medium">{use.username}</span>
                  <span className="flex shrink-0 items-center gap-2">
                    <span className="text-xs text-muted-foreground">
                      {usedAtFormatter.format(new Date(use.usedAt))}
                    </span>
                    <Badge
                      variant={isCurrent ? "default" : "outline"}
                      className="font-mono"
                      title={
                        isCurrent
                          ? tAdmins("inviteLinkUsageCurrent")
                          : undefined
                      }
                    >
                      {use.linkId}
                    </Badge>
                  </span>
                </div>
              );
            })}
          </div>
        </div>
      )}

      <AlertDialog
        open={confirmOpen}
        onOpenChange={(next) => !working && setConfirmOpen(next)}
      >
        <AlertDialogContent>
          <AlertDialogHeader>
            <AlertDialogTitle>
              {tAdmins("inviteLinkRegenerate")}
            </AlertDialogTitle>
            <AlertDialogDescription>
              {tAdmins("inviteLinkRegenerateConfirm")}
            </AlertDialogDescription>
          </AlertDialogHeader>
          <AlertDialogFooter>
            <AlertDialogCancel disabled={working}>
              {tCommon("cancel")}
            </AlertDialogCancel>
            <AlertDialogAction
              disabled={working}
              onClick={(e) => {
                e.preventDefault();
                void generate();
              }}
              className="bg-warning text-warning-foreground hover:bg-warning/90"
            >
              {working && <Loader2 className="mr-2 size-4 animate-spin" />}
              {tAdmins("inviteLinkRegenerate")}
            </AlertDialogAction>
          </AlertDialogFooter>
        </AlertDialogContent>
      </AlertDialog>
    </div>
  );
}
```

Design notes pinned to spec/styles:
- The usage-trail block renders in **both** link states (only gated on `recentUses` being non-empty) — history survives expiry/regeneration (spec § 5.2).
- The "current link" chip accent is `Badge variant="default"` (primary) vs `outline` for old generations; `linkId` chips are monospace.
- `bg-warning text-warning-foreground` on the confirm **button** is correct (solid warning background → dark-brown foreground token); the transparent caution **box** uses `text-warning` per the canonical alert pattern.
- `origin` is read in a mount effect because client components still prerender on the server (no `window` at render time).

- [x] **Step 2: Verify**

Run: `cd web && yarn lint && yarn format`
Expected: clean.

---

### Task 14: Web — panel placements (3 pages)

**Files:**
- Modify: `web/src/app/[locale]/dashboard/settings/organization/page.tsx`
- Modify: `web/src/app/[locale]/dashboard/settings/store/page.tsx`
- Modify: `web/src/app/[locale]/dashboard/admins/page.tsx`

- [x] **Step 1:** Replace `settings/organization/page.tsx` (page is SUPER_ADMIN-scoped — always shows the org panel):

```tsx
"use client";

import {
  getOrgInviteLink,
  getOrgJoinCode,
  rotateOrgInviteLink,
  rotateOrgJoinCode,
} from "@/features/organization/api";
import { InviteLinkPanel } from "@/features/organization/invite-link-panel";
import { JoinCodePanel } from "@/features/organization/join-code-panel";

export default function OrganizationSettingsPage() {
  return (
    <div className="mx-auto max-w-2xl space-y-6">
      <JoinCodePanel
        fetchCode={getOrgJoinCode}
        rotateCode={rotateOrgJoinCode}
      />
      <InviteLinkPanel
        fetchLink={getOrgInviteLink}
        generateLink={rotateOrgInviteLink}
      />
    </div>
  );
}
```

- [x] **Step 2:** In `settings/store/page.tsx`: extend the `@/features/organization/api` import with `getStoreInviteLink, rotateStoreInviteLink`, add `import { InviteLinkPanel } from "@/features/organization/invite-link-panel";`, and replace the gated `JoinCodePanel` block at the bottom with:

```tsx
      {storeId && storeOrgId === null && (
        <>
          <JoinCodePanel
            fetchCode={() => getStoreJoinCode(storeId)}
            rotateCode={() => rotateStoreJoinCode(storeId)}
          />
          <InviteLinkPanel
            fetchLink={() => getStoreInviteLink(storeId)}
            generateLink={() => rotateStoreInviteLink(storeId)}
          />
        </>
      )}
```

(The fragment's children become direct children of the `space-y-6` root, so spacing applies. The inline-arrow props are safe — the React Compiler memoizes them, the exact pattern `JoinCodePanel` already relies on here.)

- [x] **Step 3:** In `dashboard/admins/page.tsx` — the panel renders **above `JoinRequestsPanel`** (invite and approve in one workflow). SUPER_ADMIN → org link. Role-ADMIN → fetch the admin's store and show the store panel only when the **store's** `orgId === null` (`admin.orgId` is null for every role-ADMIN, including admins of org-owned stores — the gate cannot come from the auth store; spec § 5.3).

1. Add imports:

```tsx
import {
  getOrgInviteLink,
  getStoreInviteLink,
  rotateOrgInviteLink,
  rotateStoreInviteLink,
} from "@/features/organization/api";
import { InviteLinkPanel } from "@/features/organization/invite-link-panel";
import { getStore } from "@/features/store/api";
```

2. Add state + effect inside `AdminsPage` (near the other state declarations):

```tsx
  const adminStoreId = currentAdmin?.storeId ?? null;
  const [storeOrgId, setStoreOrgId] = useState<string | null | undefined>(
    undefined,
  );

  useEffect(() => {
    if (isSuperAdmin || !adminStoreId) return;
    getStore(adminStoreId)
      .then((s) => setStoreOrgId(s.orgId))
      .catch(() => setStoreOrgId(undefined));
  }, [isSuperAdmin, adminStoreId]);
```

3. In the returned JSX, immediately before `<JoinRequestsPanel …>`:

```tsx
      {isSuperAdmin && (
        <div className="mb-6">
          <InviteLinkPanel
            fetchLink={getOrgInviteLink}
            generateLink={rotateOrgInviteLink}
          />
        </div>
      )}
      {!isSuperAdmin && adminStoreId && storeOrgId === null && (
        <div className="mb-6">
          <InviteLinkPanel
            fetchLink={() => getStoreInviteLink(adminStoreId)}
            generateLink={() => rotateStoreInviteLink(adminStoreId)}
          />
        </div>
      )}
```

(`adminStoreId` is a `const`, so TypeScript's narrowing holds inside the closures — no non-null assertions needed.)

- [x] **Step 4: Verify**

Run: `cd web && yarn lint && yarn format`
Expected: clean.

---

### Task 15: Final verification + CHANGELOGS

**Files:**
- Modify: `docs/CHANGELOGS.md`

- [x] **Step 1: Full backend suite**

Run: `cd backend && ./gradlew test`
Expected: `BUILD SUCCESSFUL`, zero failures.

- [x] **Step 2: Web lint, then tests** (lint before everything else, per preference; **no builds** — the audit flow owns that):

Run: `cd web && yarn lint && yarn test`
Expected: biome clean; all vitest suites pass including `parity.test.ts`.

- [x] **Step 3: GitNexus pre-finish check**

Run `gitnexus_detect_changes({scope: "all"})` and confirm the affected symbols match this plan's File Map (no stray edits).

- [x] **Step 4: CHANGELOGS entry.** Prepend a `## 2026-06-XX — Expiring Invite Links: backend + web implementation` section to `docs/CHANGELOGS.md` (use the actual date), following the existing format: a summary paragraph, a `### Files Changed` table listing **every** file from the File Map with NEW/MODIFIED actions and one-line summaries, and a `### Skipped` section. Items known to be skipped/deferred at planning time (list them, per project rules):

- No use-count caps, no per-invitee invitations, no recording of link opens or lifecycle events (spec § 9 — future work).
- No web component tests (Vitest covers only `src/**/*.test.ts`, node env) — manual scenarios from spec § 8 go to the audit flow.
- `requireOrgOwner`'s 403 message still says "join codes" though it now also guards invite-link endpoints (cosmetic, server-internal).
- No `yarn build` / `./gradlew build` (project rule — audit flow).

- [x] **Step 5: Manual scenario handoff.** Surface spec § 8's seven manual scenarios to the user as the checklist for the audit flow (do not run them — they need a live stack).

---

## Plan Self-Audit (spec adherence — performed at planning time)

| Spec § | Requirement | Where in plan |
|--------|-------------|---------------|
| 4.1 | `i_` + 16 bytes base64url, `SecureRandom`, `JoinCodeGenerator` overload, raw token storage | Task 1, Task 4 |
| 4.2 | 4 Redis keys + 7d/30d/10s TTLs, `expiresAt` stored in payload at mint time | Tasks 2, 4 |
| 4.3 | Service methods ×6, per-tenant lock via `setIfAbsent`, delete-old-before-write ordering, 409 on held lock | Tasks 4, 5 |
| 4.4 | 4 owner endpoints + public resolve, `requireOrgOwner`/`requireStoreAccess`, org-owned 409, always-200 owner DTO with nullable pair + `recentUses` on all four, 404 public semantics | Tasks 3, 7, 8, 9 |
| 4.5 | `inviteToken` field (`@Size(max=64)`), exactly-one-of validation, resolve→400, deleted/org-owned re-checks, same `joinRequestService.create`, token never consumed, best-effort `recordUse` after create | Task 6 |
| 4.6 | ZSet trail, member shape, `linkId` = last 4 chars, ZADD→ZREMRANGEBYSCORE→ZREMRANGEBYRANK(200)→EXPIRE, TTL refreshed each write, never throws, lazy prune + newest-20 + `runCatching` hydrate on read, per-tenant lifetime | Tasks 4, 5 |
| 5.1 | 4 + 1 web API functions, `API_ROUTES` entries, types incl. `InviteLinkUse`, `status`/`outcome` pre-existing fix | Task 10 |
| 5.2 | Panel: two link states + Copy + AlertDialog regenerate + valid-until + caution sentence + usage-trail block (both states, non-empty gate, current-chip accent, fixed-locale time, monospace chip) | Task 13 |
| 5.3 | 3 placements, admins-page store fetch for the gate, panel above JoinRequestsPanel | Task 14 |
| 5.4 | Suspense-wrapped wizard extraction, invite mode, resolve on mount, valid/invalid branches, expired-mid-flow clears invite state and falls back to manual code | Task 12 |
| 5.5 | 13 admins + 2 register keys, EN/VI structural mirror, "invite code"→"join code" rewording (5 strings), VI drafted for chat approval | Task 11 |
| 6 | All 11 error rows covered (404 resolve, 400 register paths, 409 org-owned/lock, partial-failure ordering, concurrent joiners via username reservation, network fallback, best-effort audit, malformed-entry skip) | Tasks 4–9, 12 |
| 7 | No mint on GET, entropy, owner-only trail, no IP storage, token never echoed in errors | Tasks 4, 9 |
| 8 | Backend: service + registration + controller tests as listed; web: manual scenarios handed to audit flow | Tasks 4–9, 15 |
| 10 | File table — every row mapped (with audit-resolved deviations A2/A5 noted above) | File Map |

### Post-write audit — issues found and amended in this document

The finished plan was audited against the spec for adherence, logical errors, flow inconsistencies, and codebase-integration problems. Four issues were found; all are already amended in the task bodies above:

1. **Task 6 (logical / compile robustness):** `joinViaInviteToken` originally passed the entity's nullable `id` (`org.id` / `store.id`) into `recordUseSafely`, relying on smart-cast after a `!!` elsewhere. Since `findById(targetId)` guarantees the entity id equals `targetId`, both `joinRequestService.create` and `recordUseSafely` now use the non-null `targetId` directly — no `!!` and no smart-cast dependence.
2. **Tasks 7/8, Step 2 (flow inconsistency):** the expected-failure descriptions claimed the test run would also fail on a "missing `inviteLinkService` bean dependency". Incorrect — `@MockkBean` registers the bean regardless of whether the controller consumes it, so the slice context starts and the only failure signal is 404s on the unmapped routes. Wording corrected.
3. **Task 9, Step 2 (same class):** same correction for `AuthControllerTest` — the new tests fail with 404; the pre-existing login test keeps passing.
4. **Tasks 12/13 (codebase integration):** the warning icon was written as `TriangleAlert`; the codebase convention is `AlertTriangle` (`slug-add-dialog.tsx`, `device-dispatch-panel.tsx`). Both components amended.

Verified clean during the same audit: every API signature used is real (`ReactiveValueOperations.get(Object)/set(K,V,Duration)/setIfAbsent(K,V,Duration)`, `ReactiveZSetOperations.add(K,V,double)/removeRangeByScore(K,Range<Double>)/removeRange(K,Range<Long>)/reverseRange(K,Range<Long>)`, `ReactiveRedisTemplate.expire(K,Duration)` — inspected in the spring-data-redis 3.5.9 jar; Next.js 16 `useSearchParams`-needs-`Suspense` confirmed against v16.1 docs via Context7); type/name consistency holds across tasks (service methods, DTOs, web types, `API_ROUTES` entries, i18n keys all cross-referenced); no placeholder patterns remain; the expired-mid-flow fallback uses the identical error condition (`code === 400 && message includes "invite"`) in both the wizard and the form so the two state updates can never diverge.
