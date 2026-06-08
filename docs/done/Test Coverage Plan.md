# Test Coverage Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a professional, fully DB-free unit-test suite across the Spring Boot backend and both Next.js apps (`web`, `client-web`), plus coverage tooling — completing the SDLC without touching any runtime code.

**Architecture:** Backend uses JUnit5 + MockK (+ springmockk for `@MockkBean`) in three layers — pure-logic unit tests, service tests with mocked repositories/Redis, and `@WebFluxTest` controller slices — plus one JWT/security integration slice. Frontends use Vitest (node environment, no DOM) for pure modules, zustand stores, and an `en`↔`vi` i18n parity invariant. No PostgreSQL/Redis/TestContainers — all I/O is mocked.

**Tech Stack:** Kotlin 2.3 / Spring Boot 3.5 WebFlux (coroutines), MockK 1.13.x, springmockk 5.0.1, Kover 0.9.x, Vitest 3.x + `@vitest/coverage-v8` + `vite-tsconfig-paths`.

**Source spec:** `docs/spec/Test Coverage Spec.md` (read it first — this plan implements it).

---

## Ground Rules (read once, apply throughout)

- **No git commits.** This project's convention is that the executor decides when to commit — there are **no commit steps** in this plan. After each task, the "Checkpoint" step just confirms the test passes.
- **The production code already exists.** This is not red-green TDD of new features — you are writing tests for working code. So a newly written test should **PASS on first run**. If it FAILS, the cause is almost always (a) the test, or (b) a verified-signature mismatch — fix the test. Only if you are certain the test is correct does a failure indicate a real bug, which you **report** (do not patch runtime code under this plan).
- **Lint before test on the frontend** (project convention): run `yarn lint` before `yarn vitest run` in every frontend verification step.
- **Do not run a full build.** Run only the specific test(s) for the task you just wrote (commands given per task). The full-suite + coverage run lives in Phase 8 and the project's separate audit flow.
- **All signatures in this plan were verified against source.** Where a method body could not be fully read, the task says so and tells you to confirm a stub against the body — treat those as the only places to double-check.
- **Assertion library:** backend uses AssertJ (`org.assertj.core.api.Assertions.assertThat`), bundled in `spring-boot-starter-test`. Coroutine tests use `kotlinx.coroutines.test.runTest`; suspend mocks use `coEvery`/`coVerify`.

---

## File Structure

**Backend** — `backend/src/test/kotlin/com/thomas/notiguide/` mirrors `main`:
- `core/redis/RedisKeyManagerTest.kt`, `core/redis/RedisTTLPolicyTest.kt`
- `core/jwt/TokenHashUtilTest.kt`, `core/jwt/JWTManagerTest.kt`
- `core/security/Argon2PwdEncoderTest.kt`
- `core/tenant/JoinCodeGeneratorTest.kt`
- `core/device/DeviceCanonicalTest.kt`, `core/device/DevicePublicIdMinterTest.kt`, `core/device/DeviceCommandSignerTest.kt`
- `domain/store/service/SlugValidatorTest.kt`
- `domain/queue/service/NoShowPolicyTest.kt`, `domain/queue/service/QueueServiceTest.kt`
- `domain/admin/service/AdminAuthServiceTest.kt`, `domain/admin/service/SessionServiceTest.kt`
- `domain/device/service/HubDiagnosticsServiceTest.kt`, `domain/analytics/service/AnalyticsQueryServiceTest.kt`
- `support/TestSecurityConfig.kt`, `support/TestPrincipals.kt` (shared slice helpers)
- `domain/**/controller/*ControllerTest.kt` (web slices)
- `security/JwtSecurityIntegrationTest.kt`
- `src/test/resources/rsa/test-sign.pem`, `test-sign-pub.pem` (EC test key for the signer test only)

**Frontend** — co-located `*.test.ts`:
- `web/src/lib/{password-validation,user-agent,api-error}.test.ts`, `web/src/store/{auth,queue,layout}.test.ts`, `web/src/messages/parity.test.ts`, plus `web/vitest.config.ts`
- `client-web/src/store/ticket.test.ts`, `client-web/src/types/api.test.ts`, `client-web/src/messages/parity.test.ts`, plus `client-web/vitest.config.ts`

---

# Phase 0 — Backend test tooling

### Task 0.1: Add MockK, springmockk, and Kover to the backend build

**Files:**
- Modify: `backend/build.gradle.kts`

- [ ] **Step 1: Add the Kover plugin to the `plugins { }` block**

In `backend/build.gradle.kts`, inside the existing `plugins { }` block, add:

```kotlin
    id("org.jetbrains.kotlinx.kover") version "0.9.8"
```

- [ ] **Step 2: Add the test dependencies to the `dependencies { }` block**

Alongside the existing `testImplementation(...)` lines, add:

```kotlin
    testImplementation("io.mockk:mockk:1.13.13")
    testImplementation("com.ninja-squad:springmockk:5.0.1")
```

> If Gradle reports a version-not-found, pin to the latest stable: MockK `1.13.x`, springmockk `5.x` (the 5.x line targets Spring Boot 3.4+/3.5 — do not use 4.x).

- [ ] **Step 3: Add the Kover report-filter block at the end of the file**

Append to `backend/build.gradle.kts`:

```kotlin
kover {
    reports {
        filters {
            excludes {
                classes(
                    "com.thomas.notiguide.*.entity.*",
                    "com.thomas.notiguide.*.dto.*",
                    "com.thomas.notiguide.*.request.*",
                    "com.thomas.notiguide.*.response.*",
                )
                annotatedBy("org.springframework.context.annotation.Configuration")
            }
        }
    }
}
```

- [ ] **Step 4: Verify dependencies resolve**

Run: `cd backend && ./gradlew help --task koverHtmlReport`
Expected: task is recognised (Kover applied). No dependency-resolution errors.

- [ ] **Checkpoint:** Build file updated; Kover task exists. Do not commit.

---

### Task 0.2: Prove the test harness runs (smoke test)

**Files:**
- Test: `backend/src/test/kotlin/com/thomas/notiguide/HarnessSmokeTest.kt`

- [ ] **Step 1: Write a trivial MockK + coroutine smoke test**

```kotlin
package com.thomas.notiguide

import io.mockk.coEvery
import io.mockk.mockk
import kotlinx.coroutines.test.runTest
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test

class HarnessSmokeTest {
    private fun interface Repo { suspend fun load(id: Int): String }

    @Test
    fun `mockk and runTest are wired correctly`() = runTest {
        val repo = mockk<Repo>()
        coEvery { repo.load(1) } returns "ok"
        assertThat(repo.load(1)).isEqualTo("ok")
    }
}
```

- [ ] **Step 2: Run it**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.HarnessSmokeTest"`
Expected: PASS (1 test). Confirms JUnit5 + MockK + coroutines-test all work with **no infrastructure running**.

- [ ] **Checkpoint:** Harness validated.

---

# Phase 1 — Backend pure-logic tests

All classes here are `object`s or simple classes; no Spring context.

### Task 1.1: RedisKeyManager key formats

**Files:**
- Test: `backend/src/test/kotlin/com/thomas/notiguide/core/redis/RedisKeyManagerTest.kt`

- [ ] **Step 1: Write the test**

```kotlin
package com.thomas.notiguide.core.redis

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import java.time.LocalDate
import java.util.UUID

class RedisKeyManagerTest {
    private val storeId = UUID.fromString("11111111-1111-1111-1111-111111111111")
    private val ticketId = UUID.fromString("22222222-2222-2222-2222-222222222222")

    @Test
    fun `queue key has expected format`() {
        assertThat(RedisKeyManager.queue(storeId)).isEqualTo("store:$storeId:queue")
    }

    @Test
    fun `serving key has expected format`() {
        assertThat(RedisKeyManager.serving(storeId)).isEqualTo("store:$storeId:serving")
    }

    @Test
    fun `ticket key has expected format`() {
        assertThat(RedisKeyManager.ticket(storeId, ticketId)).isEqualTo("ticket:$storeId:$ticketId")
    }

    @Test
    fun `isTicketKey recognises ticket keys only`() {
        assertThat(RedisKeyManager.isTicketKey(RedisKeyManager.ticket(storeId, ticketId))).isTrue()
        assertThat(RedisKeyManager.isTicketKey(RedisKeyManager.queue(storeId))).isFalse()
    }

    @Test
    fun `parseTicketKey round-trips a ticket key`() {
        val key = RedisKeyManager.ticket(storeId, ticketId)
        assertThat(RedisKeyManager.parseTicketKey(key)).isEqualTo(storeId to ticketId)
    }

    @Test
    fun `counter key contains store id and date`() {
        val key = RedisKeyManager.counter(storeId, LocalDate.of(2026, 6, 8))
        assertThat(key).contains(storeId.toString()).contains("2026-06-08")
    }
}
```

- [ ] **Step 2: Run** — `cd backend && ./gradlew test --tests "*RedisKeyManagerTest"` — Expected: PASS.
- [ ] **Checkpoint.**

---

### Task 1.2: RedisTTLPolicy constants

**Files:**
- Test: `backend/src/test/kotlin/com/thomas/notiguide/core/redis/RedisTTLPolicyTest.kt`

- [ ] **Step 1: Write the test**

```kotlin
package com.thomas.notiguide.core.redis

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import java.time.Duration

class RedisTTLPolicyTest {
    @Test
    fun `ticket TTLs match the documented policy`() {
        assertThat(RedisTTLPolicy.TICKET_WAITING).isEqualTo(Duration.ofHours(12))
        assertThat(RedisTTLPolicy.TICKET_CALLED).isEqualTo(Duration.ofMinutes(30))
        assertThat(RedisTTLPolicy.TICKET_TERMINAL).isEqualTo(Duration.ofHours(2))
    }

    @Test
    fun `token and join-request TTLs match the documented policy`() {
        assertThat(RedisTTLPolicy.FCM_TOKEN).isEqualTo(Duration.ofHours(12))
        assertThat(RedisTTLPolicy.REFRESH_TOKEN).isEqualTo(Duration.ofDays(7))
        assertThat(RedisTTLPolicy.JOIN_REQUEST).isEqualTo(Duration.ofDays(7))
    }
}
```

- [ ] **Step 2: Run** — `./gradlew test --tests "*RedisTTLPolicyTest"` — Expected: PASS.
- [ ] **Checkpoint.**

---

### Task 1.3: TokenHashUtil

**Files:**
- Test: `backend/src/test/kotlin/com/thomas/notiguide/core/jwt/TokenHashUtilTest.kt`

- [ ] **Step 1: Write the test**

```kotlin
package com.thomas.notiguide.core.jwt

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test

class TokenHashUtilTest {
    @Test
    fun `sha256 is stable for equal input`() {
        assertThat(TokenHashUtil.sha256("token-abc")).isEqualTo(TokenHashUtil.sha256("token-abc"))
    }

    @Test
    fun `sha256 differs for different input`() {
        assertThat(TokenHashUtil.sha256("a")).isNotEqualTo(TokenHashUtil.sha256("b"))
    }

    @Test
    fun `sha256 returns lowercase hex of length 64`() {
        // Assumes lowercase hex SHA-256 (consistent with key names like enrollmentToken(sha256Hex)).
        assertThat(TokenHashUtil.sha256("x")).matches("[0-9a-f]{64}")
    }
}
```

- [ ] **Step 2: Run** — `./gradlew test --tests "*TokenHashUtilTest"` — Expected: PASS. (If the 3rd test fails, inspect the actual encoding — base64 vs hex — and adjust the regex; the first two tests are encoding-agnostic.)
- [ ] **Checkpoint.**

---

### Task 1.4: SlugValidator

**Files:**
- Test: `backend/src/test/kotlin/com/thomas/notiguide/domain/store/service/SlugValidatorTest.kt`

- [ ] **Step 1: Write the test** (pattern `^[A-Za-z0-9]+(?:-[A-Za-z0-9]+)*$`, length 3..128, reserved set verified in source)

```kotlin
package com.thomas.notiguide.domain.store.service

import org.assertj.core.api.Assertions.assertThatCode
import org.assertj.core.api.Assertions.assertThatThrownBy
import org.junit.jupiter.api.Test

class SlugValidatorTest {
    @Test
    fun `accepts a valid hyphenated slug`() {
        assertThatCode { SlugValidator.validate("my-store-2") }.doesNotThrowAnyException()
    }

    @Test
    fun `rejects a slug shorter than the minimum`() {
        assertThatThrownBy { SlugValidator.validate("ab") }
            .isInstanceOf(IllegalArgumentException::class.java)
    }

    @Test
    fun `rejects a slug longer than the maximum`() {
        assertThatThrownBy { SlugValidator.validate("a".repeat(SlugValidator.MAX_LENGTH + 1)) }
            .isInstanceOf(IllegalArgumentException::class.java)
    }

    @Test
    fun `rejects leading, trailing, and doubled hyphens`() {
        assertThatThrownBy { SlugValidator.validate("-store") }.isInstanceOf(IllegalArgumentException::class.java)
        assertThatThrownBy { SlugValidator.validate("store-") }.isInstanceOf(IllegalArgumentException::class.java)
        assertThatThrownBy { SlugValidator.validate("a--b") }.isInstanceOf(IllegalArgumentException::class.java)
    }

    @Test
    fun `rejects a reserved word (case-insensitive)`() {
        assertThatThrownBy { SlugValidator.validate("Admin") }.isInstanceOf(IllegalArgumentException::class.java)
    }
}
```

- [ ] **Step 2: Run** — `./gradlew test --tests "*SlugValidatorTest"` — Expected: PASS.
- [ ] **Checkpoint.**

---

### Task 1.5: JoinCodeGenerator

**Files:**
- Test: `backend/src/test/kotlin/com/thomas/notiguide/core/tenant/JoinCodeGeneratorTest.kt`

- [ ] **Step 1: Write the test**

```kotlin
package com.thomas.notiguide.core.tenant

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test

class JoinCodeGeneratorTest {
    @Test
    fun `org code carries the org prefix`() {
        assertThat(JoinCodeGenerator.generate(JoinCodeGenerator.ORG_PREFIX)).startsWith("o_")
    }

    @Test
    fun `store code carries the store prefix`() {
        assertThat(JoinCodeGenerator.generate(JoinCodeGenerator.STORE_PREFIX)).startsWith("s_")
    }

    @Test
    fun `codes are distinct across calls`() {
        val a = JoinCodeGenerator.generate(JoinCodeGenerator.ORG_PREFIX)
        val b = JoinCodeGenerator.generate(JoinCodeGenerator.ORG_PREFIX)
        assertThat(a).isNotEqualTo(b)
    }

    @Test
    fun `suffix is url-safe`() {
        val suffix = JoinCodeGenerator.generate(JoinCodeGenerator.ORG_PREFIX).removePrefix("o_")
        assertThat(suffix).matches("[A-Za-z0-9_-]+")
    }
}
```

- [ ] **Step 2: Run** — `./gradlew test --tests "*JoinCodeGeneratorTest"` — Expected: PASS.
- [ ] **Checkpoint.**

---

### Task 1.6: NoShowPolicy settings resolution

**Files:**
- Test: `backend/src/test/kotlin/com/thomas/notiguide/domain/queue/service/NoShowPolicyTest.kt`

> The target is the package-level `internal fun resolveApplicableNoShowSettings(store, settings) = settings?.takeIf { store?.allowNoShow == true }`. `internal` is visible from the test source set.

- [ ] **Step 1: Write the test**

```kotlin
package com.thomas.notiguide.domain.queue.service

import com.thomas.notiguide.domain.store.entity.Store
import com.thomas.notiguide.domain.store.entity.StoreSettings
import io.mockk.every
import io.mockk.mockk
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import java.util.UUID

class NoShowPolicyTest {
    private val settings = StoreSettings(storeId = UUID.randomUUID())

    @Test
    fun `returns settings when the store allows no-show`() {
        val store = mockk<Store> { every { allowNoShow } returns true }
        assertThat(resolveApplicableNoShowSettings(store, settings)).isSameAs(settings)
    }

    @Test
    fun `returns null when the store disallows no-show`() {
        val store = mockk<Store> { every { allowNoShow } returns false }
        assertThat(resolveApplicableNoShowSettings(store, settings)).isNull()
    }

    @Test
    fun `returns null when the store is null`() {
        assertThat(resolveApplicableNoShowSettings(null, settings)).isNull()
    }

    @Test
    fun `returns null when settings is null`() {
        val store = mockk<Store> { every { allowNoShow } returns true }
        assertThat(resolveApplicableNoShowSettings(store, null)).isNull()
    }
}
```

- [ ] **Step 2: Run** — `./gradlew test --tests "*NoShowPolicyTest"` — Expected: PASS.
- [ ] **Checkpoint.**

---

### Task 1.7: DeviceCanonical exact formats

**Files:**
- Test: `backend/src/test/kotlin/com/thomas/notiguide/core/device/DeviceCanonicalTest.kt`

- [ ] **Step 1: Write the test** (format verified: `activate-v1|$challengeId|$nonce|$issuedAtInstant|$expiresAtInstant`)

```kotlin
package com.thomas.notiguide.core.device

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import java.time.OffsetDateTime
import java.time.ZoneOffset
import java.util.UUID

class DeviceCanonicalTest {
    private val challengeId = UUID.fromString("33333333-3333-3333-3333-333333333333")
    private val issued = OffsetDateTime.of(2026, 6, 8, 10, 0, 0, 0, ZoneOffset.UTC)
    private val expires = OffsetDateTime.of(2026, 6, 8, 11, 0, 0, 0, ZoneOffset.UTC)

    @Test
    fun `activate canonical has the exact documented format`() {
        assertThat(DeviceCanonical.activate(challengeId, "nonce-1", issued, expires))
            .isEqualTo("activate-v1|$challengeId|nonce-1|${issued.toInstant()}|${expires.toInstant()}")
    }

    @Test
    fun `activate canonical is stable for equal input`() {
        assertThat(DeviceCanonical.activate(challengeId, "n", issued, expires))
            .isEqualTo(DeviceCanonical.activate(challengeId, "n", issued, expires))
    }

    @Test
    fun `activate canonical differs when the nonce changes`() {
        assertThat(DeviceCanonical.activate(challengeId, "n1", issued, expires))
            .isNotEqualTo(DeviceCanonical.activate(challengeId, "n2", issued, expires))
    }
}
```

- [ ] **Step 2: Run** — `./gradlew test --tests "*DeviceCanonicalTest"` — Expected: PASS.
- [ ] **Checkpoint.**

---

### Task 1.8: DevicePublicIdMinter

**Files:**
- Test: `backend/src/test/kotlin/com/thomas/notiguide/core/device/DevicePublicIdMinterTest.kt`

> Verified: prefix `rcv` (RECEIVER_433M/2_4G), `pas` (RECEIVER_433M_PASSIVE), `hub` (TRANSMITTER_HUB); 5-char Crockford suffix; checks `lookup.existsByPublicId(candidate)` (suspend → Boolean); throws `IllegalStateException` if all 32 attempts collide.

- [ ] **Step 1: Write the test**

```kotlin
package com.thomas.notiguide.core.device

import com.thomas.notiguide.domain.device.repository.DeviceRepository
import com.thomas.notiguide.domain.device.types.DeviceKind
import io.mockk.coEvery
import io.mockk.mockk
import kotlinx.coroutines.test.runTest
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test

class DevicePublicIdMinterTest {
    private val repo = mockk<DeviceRepository>()
    private val minter = DevicePublicIdMinter(repo)

    @Test
    fun `mint produces a rcv-prefixed Crockford id for a receiver`() = runTest {
        coEvery { repo.existsByPublicId(any()) } returns false
        assertThat(minter.mint(DeviceKind.RECEIVER_433M)).matches("rcv-[0-9ABCDEFGHJKMNPQRSTVWXYZ]{5}")
    }

    @Test
    fun `mint uses the hub prefix for a transmitter hub`() = runTest {
        coEvery { repo.existsByPublicId(any()) } returns false
        assertThat(minter.mint(DeviceKind.TRANSMITTER_HUB)).startsWith("hub-")
    }

    @Test
    fun `mint uses the passive prefix for a passive receiver`() = runTest {
        coEvery { repo.existsByPublicId(any()) } returns false
        assertThat(minter.mint(DeviceKind.RECEIVER_433M_PASSIVE)).startsWith("pas-")
    }

    @Test
    fun `mint throws when every candidate collides`() = runTest {
        coEvery { repo.existsByPublicId(any()) } returns true
        val thrown = runCatching { minter.mint(DeviceKind.RECEIVER_2_4G) }.exceptionOrNull()
        assertThat(thrown).isInstanceOf(IllegalStateException::class.java)
    }
}
```

- [ ] **Step 2: Run** — `./gradlew test --tests "*DevicePublicIdMinterTest"` — Expected: PASS.
- [ ] **Checkpoint.**

---

### Task 1.9: Argon2PwdEncoder bean

**Files:**
- Test: `backend/src/test/kotlin/com/thomas/notiguide/core/security/Argon2PwdEncoderTest.kt`

> `Argon2PwdEncoder` is a `@Configuration` with `passwordEncoder(): Argon2PasswordEncoder`. We test the bean it produces.

- [ ] **Step 1: Write the test**

```kotlin
package com.thomas.notiguide.core.security

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test

class Argon2PwdEncoderTest {
    private val encoder = Argon2PwdEncoder().passwordEncoder()

    @Test
    fun `encodes and verifies the same password`() {
        val hash = encoder.encode("s3cret!")
        assertThat(encoder.matches("s3cret!", hash)).isTrue()
    }

    @Test
    fun `rejects a wrong password`() {
        val hash = encoder.encode("s3cret!")
        assertThat(encoder.matches("wrong", hash)).isFalse()
    }
}
```

- [ ] **Step 2: Run** — `./gradlew test --tests "*Argon2PwdEncoderTest"` — Expected: PASS (a few seconds; Argon2 is intentionally slow).
- [ ] **Checkpoint.**

---

### Task 1.10: JWTManager issue/verify (mocked RSAKeyProperties)

**Files:**
- Test: `backend/src/test/kotlin/com/thomas/notiguide/core/jwt/JWTManagerTest.kt`

> Verified: `JWTManager(jwtProperties: JWTProperties, rsaKeys: RSAKeyProperties)`; `suspend fun issue(id: UUID, roles: List<String>): String`; `suspend fun verify(token: String): DecodedJWT` (RS512, public-key-only verify). We mock `RSAKeyProperties` with an in-test RSA-2048 keypair — no PEM files.

- [ ] **Step 1: Write the test**

```kotlin
package com.thomas.notiguide.core.jwt

import com.thomas.notiguide.core.config.JWTProperties
import com.thomas.notiguide.core.security.RSAKeyProperties
import io.mockk.every
import io.mockk.mockk
import kotlinx.coroutines.test.runTest
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import java.security.KeyPairGenerator
import java.security.interfaces.RSAPrivateKey
import java.security.interfaces.RSAPublicKey
import java.util.UUID

class JWTManagerTest {
    private val props = JWTProperties(
        accessExpirySeconds = 900,
        refreshExpirySeconds = 604800,
        privateKey = "unused",
        privateKeyPassword = "",
        publicKey = "unused",
    )

    private fun managerWithFreshKeys(): JWTManager {
        val pair = KeyPairGenerator.getInstance("RSA").apply { initialize(2048) }.generateKeyPair()
        val keys = mockk<RSAKeyProperties> {
            every { publicKey } returns pair.public as RSAPublicKey
            every { privateKey } returns pair.private as RSAPrivateKey
        }
        return JWTManager(props, keys)
    }

    @Test
    fun `issue then verify round-trips the subject`() = runTest {
        val manager = managerWithFreshKeys()
        val id = UUID.randomUUID()
        val token = manager.issue(id, listOf("ROLE_ADMIN"))
        val decoded = manager.verify(token)
        assertThat(decoded.subject).isEqualTo(id.toString())
    }

    @Test
    fun `verify rejects a token signed by a different key`() = runTest {
        val foreignToken = managerWithFreshKeys().issue(UUID.randomUUID(), listOf("ROLE_ADMIN"))
        val thrown = runCatching { managerWithFreshKeys().verify(foreignToken) }.exceptionOrNull()
        assertThat(thrown).isNotNull()
    }

    @Test
    fun `verify rejects a malformed token`() = runTest {
        val thrown = runCatching { managerWithFreshKeys().verify("not-a-jwt") }.exceptionOrNull()
        assertThat(thrown).isNotNull()
    }
}
```

- [ ] **Step 2: Run** — `./gradlew test --tests "*JWTManagerTest"` — Expected: PASS. (Verified: the token **subject** is the admin id — `JWTToPrincipal.convert` does `UUID.fromString(jwt.subject)`, so this assertion holds.)
- [ ] **Checkpoint.**

---

### Task 1.11: DeviceCommandSigner (EC test key)

**Files:**
- Create: `backend/src/test/resources/rsa/test-sign.pem`, `backend/src/test/resources/rsa/test-sign-pub.pem`
- Test: `backend/src/test/kotlin/com/thomas/notiguide/core/device/DeviceCommandSignerTest.kt`

> Verified: `DeviceCommandSigner(keyLocation: String, resourceLoader: ResourceLoader)`; `fun sign(canonical): String` → Base64 `SHA256withECDSA` over an **EC P-256** private key; **non-deterministic** (assert verify, not equality).

- [ ] **Step 1: Generate a test EC keypair**

Run (creates the test-only key fixtures):
```bash
cd backend/src/test/resources/rsa
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 -out test-sign.pem
openssl pkey -in test-sign.pem -pubout -out test-sign-pub.pem
```
Expected: two PEM files created (`test-sign.pem` = PKCS#8 EC private key, `test-sign-pub.pem` = SPKI public key).

- [ ] **Step 2: Write the test**

```kotlin
package com.thomas.notiguide.core.device

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.core.io.DefaultResourceLoader
import java.security.KeyFactory
import java.security.Signature
import java.security.spec.X509EncodedKeySpec
import java.util.Base64

class DeviceCommandSignerTest {
    private val loader = DefaultResourceLoader()
    private val signer = DeviceCommandSigner("classpath:rsa/test-sign.pem", loader)

    private fun loadPublicKey() = loader.getResource("classpath:rsa/test-sign-pub.pem")
        .inputStream.bufferedReader().readText()
        .replace("-----BEGIN PUBLIC KEY-----", "")
        .replace("-----END PUBLIC KEY-----", "")
        .replace(Regex("\\s"), "")
        .let { Base64.getDecoder().decode(it) }
        .let { KeyFactory.getInstance("EC").generatePublic(X509EncodedKeySpec(it)) }

    @Test
    fun `sign produces a signature that verifies with the matching public key`() {
        val canonical = "activate-v1|abc|nonce|2026-06-08T10:00:00Z|2026-06-08T11:00:00Z"
        val signatureB64 = signer.sign(canonical)
        assertThat(signatureB64).isNotBlank()

        val verifier = Signature.getInstance("SHA256withECDSA").apply {
            initVerify(loadPublicKey())
            update(canonical.toByteArray(Charsets.UTF_8))
        }
        assertThat(verifier.verify(Base64.getDecoder().decode(signatureB64))).isTrue()
    }

    @Test
    fun `a signature does not verify against a different message`() {
        val signatureB64 = signer.sign("message-one")
        val verifier = Signature.getInstance("SHA256withECDSA").apply {
            initVerify(loadPublicKey())
            update("message-two".toByteArray(Charsets.UTF_8))
        }
        assertThat(verifier.verify(Base64.getDecoder().decode(signatureB64))).isFalse()
    }
}
```

- [ ] **Step 3: Run** — `./gradlew test --tests "*DeviceCommandSignerTest"` — Expected: PASS. (If `DeviceCommandSigner`'s PEM parser rejects the openssl PKCS#8 output, regenerate with `openssl ec -in test-sign.pem -out test-sign.pem` to SEC1 form — the verified loader supports `PEMKeyPair` and `PrivateKeyInfo`.)
- [ ] **Checkpoint.**

---

# Phase 2 — Backend service tests (MockK)

Construct each service with `mockk(...)` collaborators (manual style — clean when some deps are real).

### Task 2.1: AdminAuthService.findByUsername

**Files:**
- Test: `backend/src/test/kotlin/com/thomas/notiguide/domain/admin/service/AdminAuthServiceTest.kt`

> Verified: `AdminAuthService(adminRepository) : ReactiveUserDetailsService`; `findByUsername(username): Mono<UserDetails>`; calls `adminRepository.findByUsername(username): Admin?`. Returns the principal regardless of `isVerified`.

- [ ] **Step 1: Write the test**

```kotlin
package com.thomas.notiguide.domain.admin.service

import com.thomas.notiguide.domain.admin.entity.Admin
import com.thomas.notiguide.domain.admin.repository.AdminRepository
import com.thomas.notiguide.domain.admin.types.AdminRole
import io.mockk.coEvery
import io.mockk.mockk
import kotlinx.coroutines.reactive.awaitSingleOrNull
import kotlinx.coroutines.test.runTest
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import java.util.UUID

class AdminAuthServiceTest {
    private val adminRepository = mockk<AdminRepository>()
    private val service = AdminAuthService(adminRepository)

    @Test
    fun `findByUsername returns a principal for an existing admin`() = runTest {
        val admin = Admin(
            id = UUID.randomUUID(),
            username = "alice",
            passwordHash = "hash",
            role = AdminRole.ROLE_ADMIN,
            isVerified = true,
        )
        coEvery { adminRepository.findByUsername("alice") } returns admin

        val details = service.findByUsername("alice").awaitSingleOrNull()

        assertThat(details).isNotNull
        assertThat(details!!.username).isEqualTo("alice")
    }

    @Test
    fun `findByUsername yields no user for an unknown admin`() = runTest {
        coEvery { adminRepository.findByUsername("ghost") } returns null
        // Empty Mono (or an error) both mean "no user"; runCatching tolerates either.
        val result = runCatching { service.findByUsername("ghost").awaitSingleOrNull() }
        assertThat(result.getOrNull()).isNull()
    }
}
```

- [ ] **Step 2: Run** — `./gradlew test --tests "*AdminAuthServiceTest"` — Expected: PASS.
- [ ] **Checkpoint.**

---

### Task 2.2: SessionService.createSession

**Files:**
- Test: `backend/src/test/kotlin/com/thomas/notiguide/domain/admin/service/SessionServiceTest.kt`

> Verified: `SessionService(sessionRepository: AdminSessionRepository, redis: ReactiveRedisTemplate<String,String>, jwtProperties: JWTProperties)`; `suspend fun createSession(adminId, tokenHash, ip, userAgent: String?): AdminSession` → `sessionRepository.save(...)`.

- [ ] **Step 1: Write the test**

```kotlin
package com.thomas.notiguide.domain.admin.service

import com.thomas.notiguide.core.config.JWTProperties
import com.thomas.notiguide.domain.admin.entity.AdminSession
import com.thomas.notiguide.domain.admin.repository.AdminSessionRepository
import io.mockk.coEvery
import io.mockk.coVerify
import io.mockk.mockk
import kotlinx.coroutines.test.runTest
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.data.redis.core.ReactiveRedisTemplate
import java.util.UUID

class SessionServiceTest {
    private val sessionRepository = mockk<AdminSessionRepository>()
    private val redis = mockk<ReactiveRedisTemplate<String, String>>()
    private val service = SessionService(
        sessionRepository,
        redis,
        JWTProperties(900, 604800, "k", "", "k"),
    )

    @Test
    fun `createSession persists and returns the session`() = runTest {
        val adminId = UUID.randomUUID()
        val saved = AdminSession(
            id = UUID.randomUUID(),
            adminId = adminId,
            tokenHash = "hash",
            ipAddress = "1.2.3.4",
            userAgent = "UA",
        )
        coEvery { sessionRepository.save(any()) } returns saved

        val result = service.createSession(adminId, "hash", "1.2.3.4", "UA")

        assertThat(result.adminId).isEqualTo(adminId)
        assertThat(result.tokenHash).isEqualTo("hash")
        coVerify(exactly = 1) { sessionRepository.save(any()) }
    }
}
```

- [ ] **Step 2: Run** — `./gradlew test --tests "*SessionServiceTest"` — Expected: PASS.
- [ ] **Checkpoint.**

---

### Task 2.3: QueueService non-Lua methods

**Files:**
- Test: `backend/src/test/kotlin/com/thomas/notiguide/domain/queue/service/QueueServiceTest.kt`

> Verified: 15-arg constructor (order below). We test only the **non-Lua** methods `cleanupServingSet` and `getQueueState`. `getServingTickets` returns `Flow<String>` (not suspend → stub with `every`).

- [ ] **Step 1: Write the test**

```kotlin
package com.thomas.notiguide.domain.queue.service

import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import com.thomas.notiguide.core.redis.RedisKeyManager
import com.thomas.notiguide.domain.queue.repository.RedisCounterRepository
import com.thomas.notiguide.domain.queue.repository.RedisQueueRepository
import com.thomas.notiguide.domain.queue.repository.RedisTicketRepository
import com.thomas.notiguide.domain.queue.types.QueueState
import io.mockk.coEvery
import io.mockk.coVerify
import io.mockk.every
import io.mockk.mockk
import kotlinx.coroutines.flow.flowOf
import kotlinx.coroutines.test.runTest
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.data.redis.core.ReactiveRedisTemplate
import org.springframework.data.redis.core.ReactiveValueOperations
import reactor.core.publisher.Mono
import java.util.UUID

class QueueServiceTest {
    private val storeRepository = mockk<com.thomas.notiguide.domain.store.repository.StoreRepository>(relaxed = true)
    private val redisQueueRepository = mockk<RedisQueueRepository>()
    private val redisTicketRepository = mockk<RedisTicketRepository>()
    private val redisCounterRepository = mockk<RedisCounterRepository>(relaxed = true)
    private val redis = mockk<ReactiveRedisTemplate<String, String>>()
    private val queueEventBroadcaster = mockk<com.thomas.notiguide.core.sse.QueueEventBroadcaster>(relaxed = true)
    private val deviceDispatchEventBroadcaster = mockk<com.thomas.notiguide.core.sse.DeviceDispatchEventBroadcaster>(relaxed = true)
    private val deviceQueryService = mockk<com.thomas.notiguide.domain.device.service.DeviceQueryService>(relaxed = true)
    private val storeSettingsRepository = mockk<com.thomas.notiguide.domain.store.repository.StoreSettingsRepository>(relaxed = true)
    private val serviceTypeRepository = mockk<com.thomas.notiguide.domain.store.repository.ServiceTypeRepository>(relaxed = true)

    private val service = QueueService(
        storeRepository, redisQueueRepository, redisTicketRepository, redisCounterRepository,
        redis, jacksonObjectMapper(), null, null, null, null,
        queueEventBroadcaster, deviceDispatchEventBroadcaster, deviceQueryService,
        storeSettingsRepository, serviceTypeRepository,
    )

    private val storeId = UUID.randomUUID()

    @Test
    fun `cleanupServingSet removes orphaned tickets and counts them`() = runTest {
        val ghost = UUID.randomUUID()
        every { redisQueueRepository.getServingTickets(storeId) } returns flowOf(ghost.toString())
        coEvery { redisTicketRepository.exists(storeId, ghost) } returns false
        coEvery { redisQueueRepository.removeFromServing(storeId, ghost) } returns 1L

        val cleaned = service.cleanupServingSet(storeId)

        assertThat(cleaned).isEqualTo(1)
        coVerify { redisQueueRepository.removeFromServing(storeId, ghost) }
    }

    @Test
    fun `getQueueState returns PAUSED when redis holds PAUSED`() = runTest {
        val valueOps = mockk<ReactiveValueOperations<String, String>>()
        every { redis.opsForValue() } returns valueOps
        every { valueOps.get(RedisKeyManager.queueState(storeId)) } returns Mono.just("PAUSED")

        assertThat(service.getQueueState(storeId)).isEqualTo(QueueState.PAUSED)
    }

    @Test
    fun `getQueueState defaults to ACTIVE when nothing is stored`() = runTest {
        val valueOps = mockk<ReactiveValueOperations<String, String>>()
        every { redis.opsForValue() } returns valueOps
        every { valueOps.get(any()) } returns Mono.empty()

        assertThat(service.getQueueState(storeId)).isEqualTo(QueueState.ACTIVE)
    }
}
```

- [ ] **Step 2: Run** — `./gradlew test --tests "*QueueServiceTest"` — Expected: PASS. (If a constructor type/package differs, fix the import — the constructor **order** is verified. If `getQueueState`'s default is not `ACTIVE`, adjust the third test to the actual default.)
- [ ] **Checkpoint.**

---

### Task 2.4: HubDiagnosticsService.loadDiagnostics (cache-miss path)

**Files:**
- Test: `backend/src/test/kotlin/com/thomas/notiguide/domain/device/service/HubDiagnosticsServiceTest.kt`

> Verified: `HubDiagnosticsService(redis, objectMapper, properties: DeviceTransmitterProperties, client: DatabaseClient)`; `suspend fun loadDiagnostics(deviceId): HubDiagnosticsDto?` reads a Redis value and JSON-deserializes it. The cache-miss path returns `null` without needing the DTO shape. (`getHubHealthSummary` is `DatabaseClient`-backed and out of scope — spec §3.7.)

- [ ] **Step 1: Write the test**

```kotlin
package com.thomas.notiguide.domain.device.service

import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import io.mockk.every
import io.mockk.mockk
import kotlinx.coroutines.test.runTest
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.data.redis.core.ReactiveRedisTemplate
import org.springframework.data.redis.core.ReactiveValueOperations
import org.springframework.r2dbc.core.DatabaseClient
import reactor.core.publisher.Mono
import java.util.UUID

class HubDiagnosticsServiceTest {
    private val redis = mockk<ReactiveRedisTemplate<String, String>>()
    private val properties = mockk<com.thomas.notiguide.core.device.DeviceTransmitterProperties>(relaxed = true)
    private val client = mockk<DatabaseClient>(relaxed = true)
    private val service = HubDiagnosticsService(redis, jacksonObjectMapper(), properties, client)

    @Test
    fun `loadDiagnostics returns null when nothing is cached`() = runTest {
        val valueOps = mockk<ReactiveValueOperations<String, String>>()
        every { redis.opsForValue() } returns valueOps
        every { valueOps.get(any()) } returns Mono.empty()

        assertThat(service.loadDiagnostics(UUID.randomUUID())).isNull()
    }
}
```

- [ ] **Step 2: Run** — `./gradlew test --tests "*HubDiagnosticsServiceTest"` — Expected: PASS.
- [ ] **Checkpoint.**

---

### Task 2.5: AnalyticsQueryService.getRealtimeStats (concrete spec — confirm stubs against body)

**Files:**
- Test: `backend/src/test/kotlin/com/thomas/notiguide/domain/analytics/service/AnalyticsQueryServiceTest.kt`

> Verified constructor: `AnalyticsQueryService(analyticsEventRepository, redisQueueRepository, redisCounterRepository, storeRepository, redis, appProperties: AppProperties)`. `getRealtimeStats(storeId): RealtimeStatsResponse(currentQueueSize, currentServingCount, ticketsIssuedToday, estimatedAvgWaitMinutes)`. The method's exact Redis reads for `estimatedAvgWaitMinutes` were not fully read — **before running, open `getRealtimeStats` and stub whatever Redis key/op it reads** (most likely `redis.opsForValue().get(avgServiceDuration)` → return `Mono.empty()`).

- [ ] **Step 1: Write the test**

```kotlin
package com.thomas.notiguide.domain.analytics.service

import com.thomas.notiguide.core.config.AppProperties
import com.thomas.notiguide.domain.analytics.repository.AnalyticsEventRepository
import com.thomas.notiguide.domain.queue.repository.RedisCounterRepository
import com.thomas.notiguide.domain.queue.repository.RedisQueueRepository
import com.thomas.notiguide.domain.store.repository.StoreRepository
import io.mockk.coEvery
import io.mockk.every
import io.mockk.mockk
import kotlinx.coroutines.test.runTest
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.data.redis.core.ReactiveRedisTemplate
import org.springframework.data.redis.core.ReactiveValueOperations
import reactor.core.publisher.Mono
import java.util.UUID

class AnalyticsQueryServiceTest {
    private val analyticsEventRepository = mockk<AnalyticsEventRepository>(relaxed = true)
    private val redisQueueRepository = mockk<RedisQueueRepository>()
    private val redisCounterRepository = mockk<RedisCounterRepository>()
    private val storeRepository = mockk<StoreRepository>(relaxed = true)
    private val redis = mockk<ReactiveRedisTemplate<String, String>>()
    private val appProperties = mockk<AppProperties>(relaxed = true)
    private val service = AnalyticsQueryService(
        analyticsEventRepository, redisQueueRepository, redisCounterRepository,
        storeRepository, redis, appProperties,
    )

    @Test
    fun `getRealtimeStats maps queue, serving, and issued-today counts`() = runTest {
        val storeId = UUID.randomUUID()
        coEvery { redisQueueRepository.getQueueSize(storeId) } returns 5L
        coEvery { redisQueueRepository.getServingCount(storeId) } returns 2L
        coEvery { redisCounterRepository.getCurrentCount(storeId) } returns 10L
        val valueOps = mockk<ReactiveValueOperations<String, String>>()
        every { redis.opsForValue() } returns valueOps
        every { valueOps.get(any()) } returns Mono.empty() // no avg-service-duration cached

        val stats = service.getRealtimeStats(storeId)

        assertThat(stats.currentQueueSize).isEqualTo(5)
        assertThat(stats.currentServingCount).isEqualTo(2)
        assertThat(stats.ticketsIssuedToday).isEqualTo(10)
    }
}
```

- [ ] **Step 2: Run** — `./gradlew test --tests "*AnalyticsQueryServiceTest"` — Expected: PASS. If it fails with a missing stub (e.g. an unstubbed `appProperties.timezone` NPE or an extra Redis read), add the stub the body requires and re-run. This is the one service test whose exact internal reads must be confirmed against the method body.
- [ ] **Checkpoint.**

---

### Task 2.6: Remaining services — EnrollmentTokenService + JoinRequestService

> Closes the spec §3.4 must-cover list. `RegistrationService.register` is a 6-collaborator `@Transactional` orchestrator whose happy path is heavy to unit-test in isolation; it is exercised at the controller layer (AuthController `POST /register` validation) and **deferred** for deeper unit coverage — log the deferral in Task 8.2.

- [ ] **Step 1: Write `EnrollmentTokenServiceTest.kt`** (cache-miss path — tractable, no DTO shape needed)

`backend/src/test/kotlin/com/thomas/notiguide/domain/device/service/EnrollmentTokenServiceTest.kt`:

```kotlin
package com.thomas.notiguide.domain.device.service

import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import io.mockk.every
import io.mockk.mockk
import kotlinx.coroutines.test.runTest
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.data.redis.core.ReactiveRedisTemplate
import org.springframework.data.redis.core.ReactiveValueOperations
import reactor.core.publisher.Mono

class EnrollmentTokenServiceTest {
    private val redis = mockk<ReactiveRedisTemplate<String, String>>()
    private val storeRepository = mockk<com.thomas.notiguide.domain.store.repository.StoreRepository>(relaxed = true)
    private val storeAccess = mockk<com.thomas.notiguide.shared.principal.StoreAccessService>(relaxed = true)
    private val properties = mockk<com.thomas.notiguide.core.device.DeviceCommandSigningProperties>(relaxed = true)
    private val service = EnrollmentTokenService(redis, storeRepository, storeAccess, jacksonObjectMapper(), properties)

    @Test
    fun `consume returns null for an unknown token`() = runTest {
        val valueOps = mockk<ReactiveValueOperations<String, String>>()
        every { redis.opsForValue() } returns valueOps
        every { valueOps.get(any()) } returns Mono.empty()

        assertThat(service.consume("nonexistent-token")).isNull()
    }
}
```

- [ ] **Step 2: Run** — `./gradlew test --tests "*EnrollmentTokenServiceTest"` — Expected: PASS.

- [ ] **Step 3: Write `JoinRequestServiceTest.kt`** (concrete spec — confirm the Redis read against the body)

`backend/src/test/kotlin/com/thomas/notiguide/domain/admin/service/JoinRequestServiceTest.kt`. Verified ctor `JoinRequestService(redis, objectMapper, adminRepository)`; `suspend fun usernameReserved(usernameLower): Boolean` consults `adminRepository.existsByUsername(...)` and a Redis username index. Construct `JoinRequestService(mockk<ReactiveRedisTemplate<String,String>>(), jacksonObjectMapper(), mockk<AdminRepository>())` and:
  - **Reserved case:** `coEvery { adminRepository.existsByUsername("taken") } returns true` (stub the Redis index read too if the method reaches it before returning) → `assertThat(service.usernameReserved("taken")).isTrue()`.
  - **Free case:** `coEvery { adminRepository.existsByUsername("free") } returns false` + stub the Redis index empty (`every { redis.opsForValue().get(any()) } returns Mono.empty()` or `every { redis.hasKey(any()) } returns Mono.just(false)`, per the body) → `assertThat(service.usernameReserved("free")).isFalse()`.
  - Open `usernameReserved` first and stub exactly what it reads (the only confirm-against-body step here).

- [ ] **Step 4: Run** — `./gradlew test --tests "*JoinRequestServiceTest"` — Expected: PASS.
- [ ] **Checkpoint:** services fan-out complete; note the `RegistrationService` deferral for Task 8.2.

---

# Phase 3 — Backend controller web slices

All controllers use **manual** gating (no `@PreAuthorize`): `@AuthenticationPrincipal AdminPrincipal`, a private `requireSuperAdmin(principal)` driven by `principal.authorities`, and `storeAccess.requireStoreAccess(principal, storeId)` (suspend). We exclude the two custom `CoWebFilter`s and inject auth via `mockAuthentication`.

### Task 3.0: Shared slice support (TestSecurityConfig + principals)

**Files:**
- Create: `backend/src/test/kotlin/com/thomas/notiguide/support/TestSecurityConfig.kt`
- Create: `backend/src/test/kotlin/com/thomas/notiguide/support/TestPrincipals.kt`

- [ ] **Step 1: Write a permissive test security chain**

```kotlin
package com.thomas.notiguide.support

import org.springframework.boot.test.context.TestConfiguration
import org.springframework.context.annotation.Bean
import org.springframework.security.config.annotation.web.reactive.EnableWebFluxSecurity
import org.springframework.security.config.web.server.ServerHttpSecurity
import org.springframework.security.web.server.SecurityWebFilterChain

@TestConfiguration
@EnableWebFluxSecurity
class TestSecurityConfig {
    @Bean
    fun testSecurityWebFilterChain(http: ServerHttpSecurity): SecurityWebFilterChain = http
        .csrf { it.disable() }
        .authorizeExchange { it.anyExchange().permitAll() }
        .build()
}
```

- [ ] **Step 2: Write a principal/auth-token factory** (verified `AdminPrincipal` + `AdminPrincipalAuthToken` constructors)

```kotlin
package com.thomas.notiguide.support

import com.thomas.notiguide.domain.admin.types.AdminRole
import com.thomas.notiguide.shared.principal.AdminPrincipal
import com.thomas.notiguide.shared.principal.AdminPrincipalAuthToken
import org.springframework.security.core.authority.SimpleGrantedAuthority
import java.util.UUID

object TestPrincipals {
    fun authToken(
        role: AdminRole = AdminRole.ROLE_ADMIN,
        id: UUID = UUID.randomUUID(),
        storeId: UUID? = null,
        orgId: UUID? = null,
    ): AdminPrincipalAuthToken {
        val principal = AdminPrincipal(
            id,
            "tester",
            "",
            listOf(SimpleGrantedAuthority(role.name)),
            orgId,
            storeId,
            true,
        )
        return AdminPrincipalAuthToken(principal)
    }
}
```

- [ ] **Step 3: Compile check** — `cd backend && ./gradlew compileTestKotlin` — Expected: BUILD SUCCESSFUL (no test run yet).
- [ ] **Checkpoint.** These helpers are reused by every slice task below.

---

### Task 3.1: AuthController slice (public + validation)

**Files:**
- Test: `backend/src/test/kotlin/com/thomas/notiguide/domain/admin/controller/AuthControllerTest.kt`

> `@RequestMapping("/api/auth")`. `POST /login` takes `@Valid @RequestBody LoginRequest(username, password)` + `ServerHttpRequest`. Since `LoginRequest` fields are non-null Kotlin, an empty body fails deserialization → 400. This validates routing + the global advice without stubbing the 12-service login orchestration.

- [ ] **Step 1: Write the test**

```kotlin
package com.thomas.notiguide.domain.admin.controller

import com.ninjasquad.springmockk.MockkBean
import com.thomas.notiguide.core.ratelimit.RateLimitFilter
import com.thomas.notiguide.core.jwt.JWTAuthFilter
import com.thomas.notiguide.support.TestSecurityConfig
import org.junit.jupiter.api.Test
import org.springframework.boot.test.autoconfigure.web.reactive.WebFluxTest
import org.springframework.context.annotation.ComponentScan.Filter
import org.springframework.context.annotation.FilterType
import org.springframework.context.annotation.Import
import org.springframework.http.MediaType
import org.springframework.test.web.reactive.server.WebTestClient

@WebFluxTest(
    controllers = [AuthController::class],
    excludeFilters = [Filter(type = FilterType.ASSIGNABLE_TYPE, classes = [JWTAuthFilter::class, RateLimitFilter::class])],
)
@Import(TestSecurityConfig::class)
class AuthControllerTest {

    // AuthController's collaborators — mocked so the slice context starts.
    @MockkBean(relaxed = true) lateinit var adminRepository: com.thomas.notiguide.domain.admin.repository.AdminRepository
    @MockkBean(relaxed = true) lateinit var passwordEncoder: org.springframework.security.crypto.password.PasswordEncoder
    @MockkBean(relaxed = true) lateinit var jwtManager: com.thomas.notiguide.core.jwt.JWTManager
    @MockkBean(relaxed = true) lateinit var refreshTokenService: com.thomas.notiguide.core.jwt.RefreshTokenService
    @MockkBean(relaxed = true) lateinit var storeRepository: com.thomas.notiguide.domain.store.repository.StoreRepository
    @MockkBean(relaxed = true) lateinit var jwtProperties: com.thomas.notiguide.core.config.JWTProperties
    @MockkBean(relaxed = true) lateinit var appProperties: com.thomas.notiguide.core.config.AppProperties
    @MockkBean(relaxed = true) lateinit var adminService: com.thomas.notiguide.domain.admin.service.AdminService
    @MockkBean(relaxed = true) lateinit var sessionService: com.thomas.notiguide.domain.admin.service.SessionService
    @MockkBean(relaxed = true) lateinit var loginAbortService: com.thomas.notiguide.core.jwt.LoginAbortService
    @MockkBean(relaxed = true) lateinit var registrationService: com.thomas.notiguide.domain.admin.service.RegistrationService
    @MockkBean(relaxed = true) lateinit var joinRequestService: com.thomas.notiguide.domain.admin.service.JoinRequestService

    @org.springframework.beans.factory.annotation.Autowired
    lateinit var client: WebTestClient

    @Test
    fun `POST login with an empty body is rejected with 400`() {
        client.post().uri("/api/auth/login")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue("{}")
            .exchange()
            .expectStatus().isBadRequest
    }
}
```

- [ ] **Step 2: Run** — `cd backend && ./gradlew test --tests "*AuthControllerTest"` — Expected: PASS. (If the slice fails to start because a collaborator package differs, fix the `@MockkBean` type — the **list** of 12 collaborators is verified.)
- [ ] **Checkpoint.** This task also proves the slice harness (exclude-filters + TestSecurityConfig) works — validate it here before fanning out.

---

### Task 3.2: QueueAdminController slice (store-access gating)

**Files:**
- Test: `backend/src/test/kotlin/com/thomas/notiguide/domain/queue/controller/QueueAdminControllerTest.kt`

> `@RequestMapping("/api/queue/admin/{storeId}")`, constructor `(QueueService, QueueEventBroadcaster, DeviceDispatchService, StoreAccessService)`. `GET /size` → `storeAccess.requireStoreAccess(principal, storeId)` (suspend) then `queueService.getQueueSize`.

- [ ] **Step 1: Write the test**

```kotlin
package com.thomas.notiguide.domain.queue.controller

import com.ninjasquad.springmockk.MockkBean
import com.thomas.notiguide.core.jwt.JWTAuthFilter
import com.thomas.notiguide.core.ratelimit.RateLimitFilter
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
import org.springframework.security.test.web.reactive.server.SecurityMockServerConfigurers.mockAuthentication
import org.springframework.test.web.reactive.server.WebTestClient
import java.util.UUID

@WebFluxTest(
    controllers = [QueueAdminController::class],
    excludeFilters = [Filter(type = FilterType.ASSIGNABLE_TYPE, classes = [JWTAuthFilter::class, RateLimitFilter::class])],
)
@Import(TestSecurityConfig::class)
class QueueAdminControllerTest {
    @MockkBean lateinit var queueService: com.thomas.notiguide.domain.queue.service.QueueService
    @MockkBean(relaxed = true) lateinit var broadcaster: com.thomas.notiguide.core.sse.QueueEventBroadcaster
    @MockkBean(relaxed = true) lateinit var dispatchService: com.thomas.notiguide.domain.device.service.DeviceDispatchService
    @MockkBean lateinit var storeAccess: com.thomas.notiguide.shared.principal.StoreAccessService

    @Autowired lateinit var client: WebTestClient

    private val storeId = UUID.randomUUID()

    @Test
    fun `GET size returns 200 when store access is granted`() {
        coEvery { storeAccess.requireStoreAccess(any(), storeId) } returns Unit
        coEvery { queueService.getQueueSize(storeId) } returns 7L

        client.mutateWith(mockAuthentication(TestPrincipals.authToken(storeId = storeId)))
            .get().uri("/api/queue/admin/$storeId/size")
            .exchange()
            .expectStatus().isOk

        coVerify { storeAccess.requireStoreAccess(any(), storeId) }
    }

    @Test
    fun `GET size returns 403 when store access is denied`() {
        // ForbiddenException is the project's HttpException(FORBIDDEN, ...) subclass; the global
        // @RestControllerAdvice maps it to 403. (This is what StoreAccessService.requireStoreAccess throws on denial.)
        coEvery { storeAccess.requireStoreAccess(any(), storeId) } throws
            com.thomas.notiguide.core.exception.ForbiddenException("forbidden")

        client.mutateWith(mockAuthentication(TestPrincipals.authToken(storeId = storeId)))
            .get().uri("/api/queue/admin/$storeId/size")
            .exchange()
            .expectStatus().isForbidden
    }
}
```

- [ ] **Step 2: Run** — `./gradlew test --tests "*QueueAdminControllerTest"` — Expected: PASS. (Confirm `HttpException`'s constructor signature in `core/exception/HttpException.kt`; adjust the throw to the real `requireStoreAccess` failure type if different.)
- [ ] **Checkpoint.**

---

### Task 3.3: AnalyticsController slice (super-admin gating)

**Files:**
- Test: `backend/src/test/kotlin/com/thomas/notiguide/domain/analytics/controller/AnalyticsControllerTest.kt`

> `@RequestMapping("/api/analytics")`, constructor `(AnalyticsQueryService, StoreAccessService)`. `GET /overview/realtime` → private `requireSuperAdmin(principal)` (checks `principal.authorities` for `ROLE_SUPER_ADMIN`). Gating is driven purely by the injected principal — no service stub needed for the 403 case.

- [ ] **Step 1: Write the test**

```kotlin
package com.thomas.notiguide.domain.analytics.controller

import com.ninjasquad.springmockk.MockkBean
import com.thomas.notiguide.core.jwt.JWTAuthFilter
import com.thomas.notiguide.core.ratelimit.RateLimitFilter
import com.thomas.notiguide.domain.admin.types.AdminRole
import com.thomas.notiguide.domain.analytics.response.OverviewRealtimeResponse
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

@WebFluxTest(
    controllers = [AnalyticsController::class],
    excludeFilters = [Filter(type = FilterType.ASSIGNABLE_TYPE, classes = [JWTAuthFilter::class, RateLimitFilter::class])],
)
@Import(TestSecurityConfig::class)
class AnalyticsControllerTest {
    @MockkBean lateinit var analyticsQueryService: com.thomas.notiguide.domain.analytics.service.AnalyticsQueryService
    @MockkBean(relaxed = true) lateinit var storeAccess: com.thomas.notiguide.shared.principal.StoreAccessService

    @Autowired lateinit var client: WebTestClient

    @Test
    fun `overview realtime is forbidden for a non-super admin`() {
        client.mutateWith(mockAuthentication(TestPrincipals.authToken(role = AdminRole.ROLE_ADMIN)))
            .get().uri("/api/analytics/overview/realtime")
            .exchange()
            .expectStatus().isForbidden
    }

    @Test
    fun `overview realtime is allowed for a super admin`() {
        coEvery { analyticsQueryService.getOverviewRealtime(any()) } returns
            OverviewRealtimeResponse(0, 0, 0, 0, null)

        client.mutateWith(mockAuthentication(TestPrincipals.authToken(role = AdminRole.ROLE_SUPER_ADMIN)))
            .get().uri("/api/analytics/overview/realtime")
            .exchange()
            .expectStatus().isOk
    }
}
```

- [ ] **Step 2: Run** — `./gradlew test --tests "*AnalyticsControllerTest"` — Expected: PASS. (`getOverviewRealtime` takes `orgId: UUID?`; the controller passes the super-admin's `orgId` — `any()` covers it.)
- [ ] **Checkpoint.**

---

### Task 3.4: Remaining controller slices (QueuePublic, StoreSlug, Admin, DeviceAdmin)

> These follow the **identical harness** established in Tasks 3.0–3.3 (`@WebFluxTest(controllers=[X], excludeFilters=[...JWTAuthFilter, RateLimitFilter])` + `@Import(TestSecurityConfig)` + `@MockkBean` collaborators + `mockAuthentication(TestPrincipals.authToken(...))`). To avoid re-pasting the boilerplate four times, each is specified concretely below — copy the harness from the nearest exemplar and supply the verified collaborators/endpoint/assertion listed for each. Each is its own file and its own checkpoint.

- [ ] **3.4a — `QueuePublicControllerTest.kt`** (`domain/queue/controller/`). Exemplar: Task 3.1 (public, no auth). Collaborators (`@MockkBean`): `QueueService`, `StoreService`, `ServiceTypeService`, and `FcmNotificationService` (nullable — `@MockkBean(relaxed = true)`). `@RequestMapping("/api/queue/public/{publicId}")`.
  - Test 1: `GET /api/queue/public/{publicId}/size` with `coEvery { queueService.getQueueSize(any()) } returns 3L` (and stub `storeService` lookup of publicId→storeId as the method requires) → `expectStatus().isOk`.
  - Test 2: `POST /api/queue/public/{publicId}/tickets/{ticketId}/fcm-token` with an empty JSON body → `expectStatus().isBadRequest` (validates `@Valid RegisterFcmTokenRequest`).
  - Run: `./gradlew test --tests "*QueuePublicControllerTest"` — Expected: PASS.

- [ ] **3.4b — `StoreSlugControllerTest.kt`** (`domain/store/controller/`). Exemplar: Task 3.2 (store-access gating). Collaborators: `StoreSlugService`, `StoreAccessService`. `@RequestMapping("/api/stores/{storeId}/slugs")`.
  - Test 1: `GET /api/stores/{storeId}/slugs` with `coEvery { storeAccess.requireStoreAccess(any(), any()) } returns Unit` and `coEvery { storeSlugService.list(any()) } returns <StoreSlugListResponse>` (confirm `list` method name/return in `StoreSlugService.kt`) + `mockAuthentication(...)` → `isOk`.
  - Test 2: same endpoint, `requireStoreAccess` throws `HttpException(FORBIDDEN, ...)` → `isForbidden`.
  - Run: `./gradlew test --tests "*StoreSlugControllerTest"` — Expected: PASS.

- [ ] **3.4c — `AdminControllerTest.kt`** (`domain/admin/controller/`). Exemplar: Task 3.3 (super-admin gating) + Task 3.2. Collaborators: `AdminService`, `SessionService`, `RefreshTokenService`, `AppProperties`, `StoreAccessService` (all `@MockkBean`, relaxed where convenient). `@RequestMapping("/api/admins")`.
  - Test 1: `POST /api/admins` (createAdmin) with `mockAuthentication(authToken(role = ROLE_ADMIN))` and a valid `CreateAdminRequest` body → `isForbidden` (manual `requireSuperAdmin`).
  - Test 2: `GET /api/admins/me` with `mockAuthentication(authToken())` and `coEvery { adminService.<meMapping>(any()) } returns <AdminDto>` (confirm the method `GET /me` calls) → `isOk`.
  - Run: `./gradlew test --tests "*AdminControllerTest"` — Expected: PASS.

- [ ] **3.4d — `DeviceAdminControllerTest.kt`** (`domain/device/controller/`). Exemplar: Task 3.2. Collaborators (8): `DeviceQueryService`, `PassiveDeviceRegistrationService`, `DeviceApprovalService`, `RfCodeService`, `DeviceLifecycleService`, `UsbDispatchPayloadService`, `HubDiagnosticsService`, `StoreAccessService` (all `@MockkBean(relaxed = true)` except the one you assert on). `@RequestMapping("/api/devices")`. Note this controller also has its own `DeviceControllerExceptionHandler` advice (loaded by the slice).
  - Test 1: `GET /api/devices?storeId={id}` with `coEvery { storeAccess.requireStoreAccess(any(), any()) } returns Unit` and `coEvery { deviceQueryService.<listMapping>(...) } returns <DeviceListResponse>` (confirm the list method) + `mockAuthentication(...)` → `isOk`.
  - Test 2: `POST /api/devices/passive` with an empty body → `isBadRequest` (`@Valid PassiveDeviceRegistrationRequest`).
  - Run: `./gradlew test --tests "*DeviceAdminControllerTest"` — Expected: PASS.

- [ ] **Checkpoint:** four controller-slice files added, each green.

---

# Phase 4 — JWT / security integration slice

### Task 4.1: Real JWT filter end-to-end (no DB, no Redis)

**Files:**
- Test: `backend/src/test/kotlin/com/thomas/notiguide/security/JwtSecurityIntegrationTest.kt`

> **Harness-defining task — expect to iterate once.** Wires the real `JWTManager` (mocked `RSAKeyProperties`), real `JWTToPrincipal` (mocked `AdminRepository`), and real `JWTAuthFilter` (mocked `SessionService`) into a test `SecurityWebFilterChain` that mirrors `SecurityConfig` (we do **not** import the real `SecurityConfig`, because it also needs `RateLimitFilter`→Redis). `JWTAuthFilter` constructor is `(AppProperties, JWTManager, JWTToPrincipal, ObjectMapper, SessionService)` and it skips `/api/auth/login|logout|refresh`.

- [ ] **Step 1: Write the integration test (`@WebFluxTest` + inline `@TestConfiguration` + stub controller)**

```kotlin
package com.thomas.notiguide.security

import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import com.thomas.notiguide.core.config.AppProperties
import com.thomas.notiguide.core.config.JWTProperties
import com.thomas.notiguide.core.jwt.JWTAuthFilter
import com.thomas.notiguide.core.jwt.JWTManager
import com.thomas.notiguide.core.jwt.JWTToPrincipal
import com.thomas.notiguide.core.ratelimit.RateLimitFilter
import com.thomas.notiguide.core.security.RSAKeyProperties
import com.thomas.notiguide.domain.admin.entity.Admin
import com.thomas.notiguide.domain.admin.repository.AdminRepository
import com.thomas.notiguide.domain.admin.service.SessionService
import com.thomas.notiguide.domain.admin.types.AdminRole
import com.thomas.notiguide.shared.principal.AdminPrincipal
import io.mockk.coEvery
import io.mockk.every
import io.mockk.mockk
import kotlinx.coroutines.runBlocking
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.reactive.WebFluxTest
import org.springframework.boot.test.context.TestConfiguration
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.ComponentScan.Filter
import org.springframework.context.annotation.FilterType
import org.springframework.context.annotation.Import
import org.springframework.security.config.web.server.SecurityWebFiltersOrder
import org.springframework.security.config.web.server.ServerHttpSecurity
import org.springframework.security.core.annotation.AuthenticationPrincipal
import org.springframework.security.web.server.SecurityWebFilterChain
import org.springframework.test.web.reactive.server.WebTestClient
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController
import java.security.KeyPairGenerator
import java.security.interfaces.RSAPrivateKey
import java.security.interfaces.RSAPublicKey
import java.util.UUID

@WebFluxTest(
    controllers = [JwtSecurityIntegrationTest.StubController::class],
    excludeFilters = [Filter(type = FilterType.ASSIGNABLE_TYPE, classes = [JWTAuthFilter::class, RateLimitFilter::class])],
)
@Import(JwtSecurityIntegrationTest.JwtTestConfig::class)
class JwtSecurityIntegrationTest {

    @RestController
    class StubController {
        @GetMapping("/api/queue/public/ping")
        fun publicPing(): String = "public"

        // Echoes the authenticated principal's authorities, so we can assert they came from the DB role.
        @GetMapping("/secure/ping")
        suspend fun securePing(@AuthenticationPrincipal principal: AdminPrincipal): String =
            principal.authorities.joinToString(",") { it.authority }
    }

    @TestConfiguration
    class JwtTestConfig {
        // One fixed keypair shared by the JWTManager bean — used by both the filter and the test.
        private val pair = KeyPairGenerator.getInstance("RSA").apply { initialize(2048) }.generateKeyPair()

        @Bean fun jwtProperties() = JWTProperties(900, 604800, "unused", "", "unused")

        @Bean fun rsaKeys(): RSAKeyProperties = mockk {
            every { publicKey } returns pair.public as RSAPublicKey
            every { privateKey } returns pair.private as RSAPrivateKey
        }

        @Bean fun jwtManager(props: JWTProperties, keys: RSAKeyProperties) = JWTManager(props, keys)

        @Bean fun adminRepository(): AdminRepository = mockk(relaxed = true)

        @Bean fun jwtToPrincipal(repo: AdminRepository) = JWTToPrincipal(repo)

        @Bean fun sessionService(): SessionService = mockk(relaxed = true) {
            coEvery { isRevoked(any()) } returns false
        }

        @Bean fun appProperties(): AppProperties = mockk(relaxed = true)

        @Bean fun jwtAuthFilter(
            appProperties: AppProperties,
            jwtManager: JWTManager,
            jwtToPrincipal: JWTToPrincipal,
            sessionService: SessionService,
        ) = JWTAuthFilter(appProperties, jwtManager, jwtToPrincipal, jacksonObjectMapper(), sessionService)

        // Mirrors SecurityConfig's rules (minus RateLimitFilter, which would need Redis).
        @Bean fun securityWebFilterChain(http: ServerHttpSecurity, jwtAuthFilter: JWTAuthFilter): SecurityWebFilterChain =
            http
                .csrf { it.disable() }
                .authorizeExchange {
                    it.pathMatchers("/api/auth/**", "/api/queue/public/**").permitAll()
                    it.anyExchange().authenticated()
                }
                .addFilterAt(jwtAuthFilter, SecurityWebFiltersOrder.AUTHENTICATION)
                .build()
    }

    @Autowired lateinit var client: WebTestClient
    @Autowired lateinit var jwtManager: JWTManager
    @Autowired lateinit var adminRepository: AdminRepository

    @Test
    fun `public route is reachable without a token`() {
        client.get().uri("/api/queue/public/ping").exchange().expectStatus().isOk
    }

    @Test
    fun `secure route is 401 without a token`() {
        client.get().uri("/secure/ping").exchange().expectStatus().isUnauthorized
    }

    @Test
    fun `secure route authorizes via the DB role, not the JWT claim`() = runBlocking {
        val id = UUID.randomUUID()
        val admin = Admin(id = id, username = "alice", passwordHash = "h", role = AdminRole.ROLE_ADMIN, isVerified = true)
        coEvery { adminRepository.findById(id) } returns admin  // convert() loads by id from the token subject

        // Token claims ROLE_SUPER_ADMIN, but convert() ignores the claim and uses admin.role.
        val token = jwtManager.issue(id, listOf("ROLE_SUPER_ADMIN"))

        client.get().uri("/secure/ping")
            .header("Authorization", "Bearer $token")
            .exchange()
            .expectStatus().isOk
            .expectBody(String::class.java).isEqualTo("ROLE_ADMIN")
    }
}
```

- [ ] **Step 2: Run** — `cd backend && ./gradlew test --tests "*JwtSecurityIntegrationTest"` — Expected: PASS (3 tests). With **no DB/Redis**, this exercises the verified end-to-end path: `extractToken()` reads `Authorization: Bearer`; `JWTToPrincipal.convert()` parses the admin id from the token **subject**, loads via `adminRepository.findById(id)`, requires `isVerified`, and derives authorities from `admin.role` — so `/secure/ping` returns `ROLE_ADMIN` (DB) even though the token's `roles` claim said `ROLE_SUPER_ADMIN` (audit-round-4 regression guard).
- [ ] **Step 3 (only if the chain isn't applied):** some Boot versions need security enabled explicitly in a slice — if the secure route returns 200 without a token, add `@EnableWebFluxSecurity` (`org.springframework.security.config.annotation.web.reactive.EnableWebFluxSecurity`) to `JwtTestConfig`. This is the single expected wiring iteration (spec §8).
- [ ] **Checkpoint.** Highest-complexity task complete.

---

# Phase 5 — Frontend tooling: `web`

### Task 5.1: Install Vitest and configure `web`

**Files:**
- Modify: `web/package.json`
- Create: `web/vitest.config.ts`

- [ ] **Step 1: Add dev dependencies**

Run: `cd web && yarn add -D vitest@^3 @vitest/coverage-v8@^3 vite-tsconfig-paths@^5`
Expected: added to `devDependencies`. (`@vitest/coverage-v8` MUST share the `vitest` major version.)

- [ ] **Step 2: Add test scripts to `web/package.json`**

In the `"scripts"` block, add:
```jsonc
"test": "vitest run",
"test:watch": "vitest",
"test:coverage": "vitest run --coverage"
```

- [ ] **Step 3: Create `web/vitest.config.ts`**

```ts
import tsconfigPaths from "vite-tsconfig-paths";
import { defineConfig } from "vitest/config";

export default defineConfig({
  plugins: [tsconfigPaths()],
  test: {
    environment: "node",
    include: ["src/**/*.test.ts"],
    coverage: {
      provider: "v8",
      include: ["src/lib/**", "src/store/**", "src/types/**"],
    },
  },
});
```

- [ ] **Step 4: Smoke-check the runner** — create `web/src/smoke.test.ts` with `import { expect, it } from "vitest"; it("runs", () => expect(1 + 1).toBe(2));`, then run `yarn lint && yarn vitest run src/smoke.test.ts` — Expected: PASS. **Delete `web/src/smoke.test.ts` afterward.**
- [ ] **Checkpoint.**

---

# Phase 6 — Frontend tests: `web`

### Task 6.1: password-validation

**Files:**
- Test: `web/src/lib/password-validation.test.ts`

> Verified exports: `getPasswordRequirementStatuses(password): PasswordRequirementStatus[]` (keys `reqUppercase`/`reqLowercase`/`reqDigit`/`reqSpecial`), `getFirstMissingPasswordRequirementKey(password): PasswordRequirementKey | null`.

- [ ] **Step 1: Write the test**

```ts
import { describe, expect, it } from "vitest";
import {
  getFirstMissingPasswordRequirementKey,
  getPasswordRequirementStatuses,
} from "@/lib/password-validation";

describe("password-validation", () => {
  it("marks every requirement passed for a strong password", () => {
    const statuses = getPasswordRequirementStatuses("Abcdef1!");
    expect(statuses.every((s) => s.passed)).toBe(true);
    expect(getFirstMissingPasswordRequirementKey("Abcdef1!")).toBeNull();
  });

  it("flags the missing uppercase requirement", () => {
    const statuses = getPasswordRequirementStatuses("abcdef1!");
    expect(statuses.find((s) => s.requirementKey === "reqUppercase")?.passed).toBe(false);
  });

  it("returns the first missing requirement key", () => {
    expect(getFirstMissingPasswordRequirementKey("abcdef1!")).toBe("reqUppercase");
  });
});
```

- [ ] **Step 2: Run** — `cd web && yarn lint && yarn vitest run src/lib/password-validation.test.ts` — Expected: PASS. (If the requirement ordering differs, adjust which key is "first" — the per-key assertions are order-independent.)
- [ ] **Checkpoint.**

---

### Task 6.2: user-agent

**Files:**
- Test: `web/src/lib/user-agent.test.ts`

> Verified: `parseUserAgent(ua: string | null): { browser: string; os: string; isMobile: boolean }`.

- [ ] **Step 1: Write the test**

```ts
import { describe, expect, it } from "vitest";
import { parseUserAgent } from "@/lib/user-agent";

describe("parseUserAgent", () => {
  it("returns a shape with browser, os, and isMobile", () => {
    const result = parseUserAgent(
      "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120 Safari/537.36",
    );
    expect(result).toHaveProperty("browser");
    expect(result).toHaveProperty("os");
    expect(typeof result.isMobile).toBe("boolean");
  });

  it("flags a mobile user agent", () => {
    const result = parseUserAgent(
      "Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X) AppleWebKit/605.1.15 Mobile/15E148 Safari/604.1",
    );
    expect(result.isMobile).toBe(true);
  });

  it("handles a null user agent without throwing", () => {
    expect(() => parseUserAgent(null)).not.toThrow();
  });
});
```

- [ ] **Step 2: Run** — `yarn lint && yarn vitest run src/lib/user-agent.test.ts` — Expected: PASS.
- [ ] **Checkpoint.**

---

### Task 6.3: api-error (plumbing)

**Files:**
- Test: `web/src/lib/api-error.test.ts`

> Verified: `translateCommonApiError(error: ApiError, tErrors): string`, `translateNetworkError(tErrors): string`. We pass a `tErrors` spy `(key) => key` and assert a non-empty string comes back (the exact key→message map is intentionally not asserted; that is i18n content, not logic).

- [ ] **Step 1: Write the test**

```ts
import { describe, expect, it, vi } from "vitest";
import { translateCommonApiError, translateNetworkError } from "@/lib/api-error";
import type { ApiError } from "@/types/api";

const tErrors = (key: string) => key;

describe("api-error", () => {
  it("translates a common API error to a non-empty string and consults tErrors", () => {
    const spy = vi.fn(tErrors);
    const error = { code: 429, error: "RATE_LIMITED" } as unknown as ApiError;
    const message = translateCommonApiError(error, spy as never);
    expect(typeof message).toBe("string");
    expect(message.length).toBeGreaterThan(0);
    expect(spy).toHaveBeenCalled();
  });

  it("translates a network error to a non-empty string", () => {
    expect(translateNetworkError(tErrors as never).length).toBeGreaterThan(0);
  });
});
```

- [ ] **Step 2: Run** — `yarn lint && yarn vitest run src/lib/api-error.test.ts` — Expected: PASS. (Once green, optionally strengthen by asserting the specific key returned for each `code`, after reading the mapping in `api-error.ts`.)
- [ ] **Checkpoint.**

---

### Task 6.4: zustand stores (auth, queue, layout)

**Files:**
- Test: `web/src/store/auth.test.ts`, `web/src/store/queue.test.ts`, `web/src/store/layout.test.ts`

> Verified exports `useAuthStore`/`useQueueStore`/`useLayoutStore` with the state/actions listed below. Reset state between tests with `setState(initial, true)`.

- [ ] **Step 1: Write `auth.test.ts`**

```ts
import { afterEach, describe, expect, it } from "vitest";
import { useAuthStore } from "@/store/auth";
import type { AdminDto } from "@/types/store";

const initial = useAuthStore.getState();
afterEach(() => useAuthStore.setState(initial, true));

describe("useAuthStore", () => {
  it("starts unauthenticated", () => {
    expect(useAuthStore.getState().isAuthenticated).toBe(false);
  });

  it("login sets authenticated + super-admin flags from the response", () => {
    const response = {
      admin: { id: "1", username: "root", role: "ROLE_SUPER_ADMIN", storeId: null, orgId: "o1" } as unknown as AdminDto,
      abortToken: "abort",
    };
    useAuthStore.getState().login(response as never);
    const state = useAuthStore.getState();
    expect(state.isAuthenticated).toBe(true);
    expect(state.isSuperAdmin).toBe(true);
  });
});
```

- [ ] **Step 2: Write `queue.test.ts`**

```ts
import { afterEach, describe, expect, it } from "vitest";
import { useQueueStore } from "@/store/queue";
import type { TicketDto } from "@/types/store";

const initial = useQueueStore.getState();
afterEach(() => useQueueStore.setState(initial, true));

const ticket = (id: string) => ({ id, number: "A1", status: "CALLED" }) as unknown as TicketDto;

describe("useQueueStore", () => {
  it("adds and removes serving tickets", () => {
    useQueueStore.getState().addServingTicket(ticket("t1"));
    expect(useQueueStore.getState().servingTickets).toHaveLength(1);
    useQueueStore.getState().removeServingTicket("t1");
    expect(useQueueStore.getState().servingTickets).toHaveLength(0);
  });

  it("clearServing empties the list", () => {
    useQueueStore.getState().addServingTicket(ticket("t1"));
    useQueueStore.getState().clearServing();
    expect(useQueueStore.getState().servingTickets).toHaveLength(0);
  });
});
```

- [ ] **Step 3: Write `layout.test.ts`**

```ts
import { afterEach, describe, expect, it } from "vitest";
import { useLayoutStore } from "@/store/layout";

const initial = useLayoutStore.getState();
afterEach(() => useLayoutStore.setState(initial, true));

describe("useLayoutStore", () => {
  it("sets and clears the page gradient class", () => {
    useLayoutStore.getState().setPageGradientClass("from-x to-y");
    expect(useLayoutStore.getState().pageGradientClass).toBe("from-x to-y");
    useLayoutStore.getState().clearPageGradientClass();
    expect(useLayoutStore.getState().pageGradientClass).toBe("");
  });
});
```

- [ ] **Step 4: Run** — `yarn lint && yarn vitest run src/store` — Expected: PASS (all three files). (Confirm `AdminDto`/`TicketDto` import from `@/types/store`; if the type lives elsewhere, fix the import — the casts make the runtime behavior independent of the exact type.)
- [ ] **Checkpoint.**

---

### Task 6.5: i18n parity (`web` — keys + rich-text tags)

**Files:**
- Test: `web/src/messages/parity.test.ts`

> `web` messages use rich-text tags (`<bold>`, `<p>`, `<shield>`). Top-level keys match across en/vi (verified).

- [ ] **Step 1: Write the test**

```ts
import { describe, expect, it } from "vitest";
import en from "@/messages/en.json";
import vi from "@/messages/vi.json";

type Json = string | number | boolean | null | { [k: string]: Json } | Json[];

function leafEntries(obj: Json, prefix = ""): [string, string][] {
  if (typeof obj === "string") return [[prefix, obj]];
  if (obj && typeof obj === "object") {
    return Object.entries(obj).flatMap(([k, v]) => leafEntries(v as Json, prefix ? `${prefix}.${k}` : k));
  }
  return [];
}

const tagsOf = (s: string) => (s.match(/<\/?[a-zA-Z]+>/g) ?? []).map((t) => t.replace("/", "")).sort();
const countP = (s: string) => (s.match(/<p>/g) ?? []).length;

const enLeaves = new Map(leafEntries(en as Json));
const viLeaves = new Map(leafEntries(vi as Json));

describe("web i18n parity", () => {
  it("en and vi share identical key sets", () => {
    expect([...viLeaves.keys()].sort()).toEqual([...enLeaves.keys()].sort());
  });

  it("each leaf has matching <p> counts and rich-text tags across languages", () => {
    for (const [key, enValue] of enLeaves) {
      const viValue = viLeaves.get(key);
      if (viValue === undefined) continue;
      expect(countP(viValue), `<p> count for ${key}`).toBe(countP(enValue));
      expect(tagsOf(viValue), `tags for ${key}`).toEqual(tagsOf(enValue));
    }
  });
});
```

- [ ] **Step 2: Run** — `yarn lint && yarn vitest run src/messages/parity.test.ts` — Expected: PASS.
- [ ] **Step 3: Prove it fails on divergence** — temporarily delete one key from `web/src/messages/vi.json`, run the test → Expected: FAIL on the key-set assertion. **Revert the deletion** and re-run → PASS. (Satisfies success criterion §9.3.)
- [ ] **Checkpoint.**

---

### Task 6.6: serial support helper (only if present)

**Files:**
- Test: `web/src/lib/serial/support.test.ts`

> `serial-protocol.ts` exposes the `SerialProtocol` class whose methods need a `SerialPort` (descoped). The pure-logic target is `serial/support.ts` (a capability check). If it exports a function (e.g. `isWebSerialSupported()`), test it; **if it only reads `navigator.serial`, skip this task and note the skip in CHANGELOGS** (node env has no `navigator`).

- [ ] **Step 1:** Read `web/src/lib/serial/support.ts`. If it exports a pure predicate, write a small test asserting it returns `false`/`true` for a stubbed input. If it depends on `navigator`, **skip** (log the skip).
- [ ] **Step 2: Run (if written)** — `yarn lint && yarn vitest run src/lib/serial/support.test.ts` — Expected: PASS.
- [ ] **Checkpoint.**

---

# Phase 7 — Frontend tooling + tests: `client-web`

### Task 7.1: Install Vitest and configure `client-web`

**Files:**
- Modify: `client-web/package.json`
- Create: `client-web/vitest.config.ts`

- [ ] **Step 1:** `cd client-web && yarn add -D vitest@^3 @vitest/coverage-v8@^3 vite-tsconfig-paths@^5`
- [ ] **Step 2:** Add the same `"test"`/`"test:watch"`/`"test:coverage"` scripts to `client-web/package.json` as in Task 5.1 Step 2.
- [ ] **Step 3:** Create `client-web/vitest.config.ts` — identical to `web/vitest.config.ts` (Task 5.1 Step 3).
- [ ] **Step 4:** Smoke-check: create `client-web/src/smoke.test.ts` (`import { expect, it } from "vitest"; it("runs", () => expect(true).toBe(true));`), run `yarn lint && yarn vitest run src/smoke.test.ts` → PASS, then **delete it**.
- [ ] **Checkpoint.**

---

### Task 7.2: ticket store

**Files:**
- Test: `client-web/src/store/ticket.test.ts`

> Verified: `useTicketStore` (zustand `persist`, name `"notiguide-ticket"`). State: `storeId`, `ticketId`, `ticket`, `status`. Actions: `setTicket(storeId, ticket)`, `updateStatus(status)`, `clearTicket()`, `hasActiveTicket(storeId)`.

- [ ] **Step 1: Write the test**

```ts
import { afterEach, describe, expect, it } from "vitest";
import { useTicketStore } from "@/store/ticket";
import type { TicketDto } from "@/types/store";

const initial = useTicketStore.getState();
afterEach(() => useTicketStore.setState(initial, true));

const ticket = (id: string) => ({ id, number: "A7", status: "WAITING" }) as unknown as TicketDto;

describe("useTicketStore", () => {
  it("setTicket stores the store id and ticket", () => {
    useTicketStore.getState().setTicket("store-1", ticket("t1"));
    const state = useTicketStore.getState();
    expect(state.storeId).toBe("store-1");
    expect(state.ticketId).toBe("t1");
  });

  it("hasActiveTicket is true only for the matching store", () => {
    useTicketStore.getState().setTicket("store-1", ticket("t1"));
    expect(useTicketStore.getState().hasActiveTicket("store-1")).toBe(true);
    expect(useTicketStore.getState().hasActiveTicket("store-2")).toBe(false);
  });

  it("clearTicket resets the ticket fields", () => {
    useTicketStore.getState().setTicket("store-1", ticket("t1"));
    useTicketStore.getState().clearTicket();
    expect(useTicketStore.getState().ticket).toBeNull();
  });
});
```

- [ ] **Step 2: Run** — `cd client-web && yarn lint && yarn vitest run src/store/ticket.test.ts` — Expected: PASS. (Confirm `TicketDto` import path in `client-web/src/types`; adjust if needed — casts keep runtime behavior independent of the type.)
- [ ] **Checkpoint.**

---

### Task 7.3: types/api error classes

**Files:**
- Test: `client-web/src/types/api.test.ts`

> Verified: `ApiError(code, error, message, path?)`, `NotFoundError(message, path?)`, `RateLimitError(message, retryAfterSeconds, path?)`, `NetworkError(message?)`.

- [ ] **Step 1: Write the test**

```ts
import { describe, expect, it } from "vitest";
import { ApiError, NetworkError, NotFoundError, RateLimitError } from "@/types/api";

describe("api error classes", () => {
  it("ApiError carries its fields", () => {
    const e = new ApiError(400, "BAD_REQUEST", "bad", "/x");
    expect(e.code).toBe(400);
    expect(e.error).toBe("BAD_REQUEST");
    expect(e.message).toBe("bad");
    expect(e).toBeInstanceOf(Error);
  });

  it("NotFoundError and RateLimitError extend ApiError", () => {
    expect(new NotFoundError("nope")).toBeInstanceOf(ApiError);
    const rl = new RateLimitError("slow", 30);
    expect(rl).toBeInstanceOf(ApiError);
    expect(rl.retryAfterSeconds).toBe(30);
  });

  it("NetworkError is a plain Error", () => {
    expect(new NetworkError()).toBeInstanceOf(Error);
  });
});
```

- [ ] **Step 2: Run** — `yarn lint && yarn vitest run src/types/api.test.ts` — Expected: PASS.
- [ ] **Checkpoint.**

---

### Task 7.4: i18n parity (`client-web` — keys + ICU placeholders, no tags)

**Files:**
- Test: `client-web/src/messages/parity.test.ts`

> `client-web` messages are plain strings with `{placeholder}` tokens and **no** rich-text tags — so this variant checks key-set equality and matching ICU placeholders per leaf (not `<p>`/tags).

- [ ] **Step 1: Write the test**

```ts
import { describe, expect, it } from "vitest";
import en from "@/messages/en.json";
import vi from "@/messages/vi.json";

type Json = string | number | boolean | null | { [k: string]: Json } | Json[];

function leafEntries(obj: Json, prefix = ""): [string, string][] {
  if (typeof obj === "string") return [[prefix, obj]];
  if (obj && typeof obj === "object") {
    return Object.entries(obj).flatMap(([k, v]) => leafEntries(v as Json, prefix ? `${prefix}.${k}` : k));
  }
  return [];
}

const placeholders = (s: string) => (s.match(/\{[a-zA-Z0-9_]+\}/g) ?? []).sort();

const enLeaves = new Map(leafEntries(en as Json));
const viLeaves = new Map(leafEntries(vi as Json));

describe("client-web i18n parity", () => {
  it("en and vi share identical key sets", () => {
    expect([...viLeaves.keys()].sort()).toEqual([...enLeaves.keys()].sort());
  });

  it("each leaf uses the same ICU placeholders across languages", () => {
    for (const [key, enValue] of enLeaves) {
      const viValue = viLeaves.get(key);
      if (viValue === undefined) continue;
      expect(placeholders(viValue), `placeholders for ${key}`).toEqual(placeholders(enValue));
    }
  });
});
```

- [ ] **Step 2: Run** — `yarn lint && yarn vitest run src/messages/parity.test.ts` — Expected: PASS.
- [ ] **Step 3: Prove failure on divergence** — temporarily change a `{count}` placeholder name in `client-web/src/messages/vi.json`, run → Expected: FAIL on the placeholder assertion. **Revert** and re-run → PASS.
- [ ] **Checkpoint.**

---

# Phase 8 — Coverage & changelog

### Task 8.1: Generate coverage reports (verification)

- [ ] **Step 1: Backend coverage** — `cd backend && ./gradlew koverHtmlReport` — Expected: report at `backend/build/reports/kover/html/index.html`, generated with no infrastructure running.
- [ ] **Step 2: web coverage** — `cd web && yarn lint && yarn test:coverage` — Expected: all tests PASS; coverage summary printed for `src/lib`/`src/store`/`src/types`.
- [ ] **Step 3: client-web coverage** — `cd client-web && yarn lint && yarn test:coverage` — Expected: all tests PASS; coverage summary printed.
- [ ] **Checkpoint:** full suite green, DB-free, reports generate on demand.

### Task 8.2: Record everything in CHANGELOGS

**Files:**
- Modify: `docs/CHANGELOGS.md`

- [ ] **Step 1:** Add a dated "Test Coverage" entry to `docs/CHANGELOGS.md` listing: the build/config additions (MockK, springmockk, Kover, Vitest×2 configs + scripts), every test file created (grouped by phase), the test EC key fixtures, and any **deliberately skipped** items with the reason (e.g. `AnalyticsQueryService` deeper methods, `DeviceDispatchService`/`getHubHealthSummary` DB-backed, `serial/support` if `navigator`-dependent, controllers beyond the seven).
- [ ] **Checkpoint:** Implementation complete. (Do not build/commit — the project's separate audit flow owns full verification; commit when you decide to.)

---

## Self-Review (run by the plan author after writing — see audit notes below)

This plan was checked against `docs/spec/Test Coverage Spec.md` for coverage, placeholder scan, type/name consistency, and codebase-integration accuracy. Findings and fixes are recorded in the chat audit summary accompanying this plan.
