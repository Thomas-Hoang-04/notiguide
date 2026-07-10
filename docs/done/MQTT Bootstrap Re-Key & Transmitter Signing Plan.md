# MQTT Bootstrap Re-Key & Transmitter-Side Signing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Re-key the MQTT bootstrap reply topic from the backend-minted `challenge_id` to the hub-generated `registration_nonce` (removing `challenge_id` entirely), and make the transmitter hub sign its `roster/update`, `ack`, and `heartbeat` messages so the backend can cryptographically authenticate them.

**Architecture:** The hub subscribes to `bootstrap/{registration_nonce}` from the start (no wildcard). `challenge_id` is deleted from the topic, Redis keys, canonical string, and envelope bodies; `registration_nonce` absorbs its roles, guarded by a pre-token-consume `nonce_in_use` check with firmware auto-retry. Separately, a reusable `DeviceSignatureVerifier` verifies device→backend signatures against the per-device `public_key_der` the backend already stores; the firmware signs at each publish site with the existing `device_identity_sign_text()`.

**Tech Stack:** Kotlin / Spring WebFlux (coroutines) / R2DBC / reactive Lettuce Redis / Jackson / Auth0-style JDK crypto (`SHA256withECDSA`, EC P-256); ESP-IDF v6.0 C firmware (esp-mqtt v5, cJSON, mbedTLS via `device_identity`). Tests: JUnit5 + MockK + AssertJ + `kotlinx.coroutines.test`.

## Global Constraints

- **Hard cutover, no back-compat / no dual path.** Firmware + backend deploy in lockstep; every hub is reflashed. Unsigned/legacy device→backend messages are dropped.
- **No version bumps.** Canonical tags stay `-v1` (`activate-v1`, `roster-update-v1`, `ack-v1`, `heartbeat-v1`); `schema_version` stays `1` on all messages. Contracts tighten in place.
- `registration_nonce` = base64url (`[A-Za-z0-9-_]`, MQTT-topic-safe), backend validation floor **≥16 bytes decoded**. Firmware already emits 16 bytes — no firmware nonce-size change.
- **Byte-identical canonicals.** Firmware and backend must build every canonical string identically (field order, separators `|` and `:`, null handling). Device-generated timestamps (`heartbeat.issued_at`) are bound as the **raw wire string**, never re-serialized.
- **Named imports only** (no fully-qualified inline references). Match existing code style.
- **Commits:** the executor decides when to commit — **no commit steps are included** per project convention. Do not add `docs/CHANGELOGS.md` entries until implementation; log there at the end.
- **Firmware unit tests:** the repo has no ESP host-test harness. Firmware tasks are implementation + a **backend golden-canonical test** as the byte-contract cross-check + manual flash verification. Backend tasks are full TDD.

---



### Note on Part A backend build order (Tasks 1–8)

The `challenge_id` → `registration_nonce` change is an **atomic cross-file refactor**. **Tasks 1 and 2 are independently compilable and testable.** From **Task 3 onward the backend module does not compile** until every `challenge_id` reference is updated together, so the intermediate per-task tests (Tasks 3–7) are **authored in their task but executed together at the Task 8 compile + test gate**. Do not expect `./gradlew test` to pass mid-refactor between Tasks 3 and 8 — treat those "Run test" steps as *author the test now*; the failing→passing run happens at Task 8. (Firmware Tasks 9–10 and all of Part B are independent of this constraint and are individually testable/verifiable.)

---



## Task 1: `DeviceSignatureVerifier` (shared EC-verify)

**Files:**

- Create: `backend/src/main/kotlin/com/thomas/notiguide/core/device/DeviceSignatureVerifier.kt`
- Test: `backend/src/test/kotlin/com/thomas/notiguide/core/device/DeviceSignatureVerifierTest.kt`

**Interfaces:**

- Produces: `object DeviceSignatureVerifier { fun verify(publicKeyDer: ByteArray, canonical: String, signatureB64: String): Boolean }`

- [ ] **Step 1: Write the failing test**

```kotlin
package com.thomas.notiguide.core.device

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import java.security.KeyPairGenerator
import java.security.Signature
import java.security.spec.ECGenParameterSpec
import java.util.Base64

class DeviceSignatureVerifierTest {
    private fun sign(privateKey: java.security.PrivateKey, canonical: String): String {
        val sig = Signature.getInstance("SHA256withECDSA").apply {
            initSign(privateKey)
            update(canonical.toByteArray(Charsets.UTF_8))
        }.sign()
        return Base64.getEncoder().encodeToString(sig)
    }

    @Test
    fun `verify accepts a valid signature and rejects tampering`() {
        val kp = KeyPairGenerator.getInstance("EC")
            .apply { initialize(ECGenParameterSpec("secp256r1")) }
            .generateKeyPair()
        val canonical = "roster-update-v1|hub-1|7|1:433M:Table 1"
        val sigB64 = sign(kp.private, canonical)
        val der = kp.public.encoded // X.509 SubjectPublicKeyInfo DER

        assertThat(DeviceSignatureVerifier.verify(der, canonical, sigB64)).isTrue()
        assertThat(DeviceSignatureVerifier.verify(der, canonical + "x", sigB64)).isFalse()
        assertThat(DeviceSignatureVerifier.verify(der, canonical, "not-base64!!")).isFalse()
        assertThat(DeviceSignatureVerifier.verify(ByteArray(0), canonical, sigB64)).isFalse()
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.core.device.DeviceSignatureVerifierTest"`
Expected: FAIL — `DeviceSignatureVerifier` unresolved.

- [ ] **Step 3: Write the implementation**

```kotlin
package com.thomas.notiguide.core.device

import java.security.KeyFactory
import java.security.Signature
import java.security.interfaces.ECPublicKey
import java.security.spec.X509EncodedKeySpec
import java.util.Base64

object DeviceSignatureVerifier {
    fun verify(publicKeyDer: ByteArray, canonical: String, signatureB64: String): Boolean {
        val publicKey = runCatching {
            KeyFactory.getInstance("EC").generatePublic(X509EncodedKeySpec(publicKeyDer))
        }.getOrNull() as? ECPublicKey ?: return false
        val signatureBytes = runCatching { Base64.getDecoder().decode(signatureB64) }.getOrNull() ?: return false
        return runCatching {
            Signature.getInstance("SHA256withECDSA").run {
                initVerify(publicKey)
                update(canonical.toByteArray(Charsets.UTF_8))
                verify(signatureBytes)
            }
        }.getOrDefault(false)
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.core.device.DeviceSignatureVerifierTest"`
Expected: PASS.

---



## Task 2: `DeviceCanonical.activate` → `registration_nonce`

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/core/device/DeviceCanonical.kt:7-12`
- Test: `backend/src/test/kotlin/com/thomas/notiguide/core/device/DeviceCanonicalTest.kt`

**Interfaces:**

- Produces: `DeviceCanonical.activate(registrationNonce: String, nonce: String, issuedAt: OffsetDateTime, expiresAt: OffsetDateTime): String`

- [ ] **Step 1: Write the failing test**

```kotlin
package com.thomas.notiguide.core.device

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import java.time.OffsetDateTime

class DeviceCanonicalTest {
    @Test
    fun `activate binds registration_nonce as the first field`() {
        val issued = OffsetDateTime.parse("2026-07-09T09:00:00Z")
        val expires = OffsetDateTime.parse("2026-07-09T09:05:00Z")
        val result = DeviceCanonical.activate("Ab-_nonce123", "bWFjbm9uY2U", issued, expires)
        assertThat(result)
            .isEqualTo("activate-v1|Ab-_nonce123|bWFjbm9uY2U|2026-07-09T09:00:00Z|2026-07-09T09:05:00Z")
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.core.device.DeviceCanonicalTest"`
Expected: FAIL — signature still takes `challengeId: UUID`.

- [ ] **Step 3: Change the signature (keep the** `activate-v1` **tag)**

Replace lines 7-12 of `DeviceCanonical.kt`:

```kotlin
    fun activate(
        registrationNonce: String,
        nonce: String,
        issuedAt: OffsetDateTime,
        expiresAt: OffsetDateTime
    ): String = "activate-v1|$registrationNonce|$nonce|${issuedAt.toInstant()}|${expiresAt.toInstant()}"
```

(The `import java.util.UUID` line stays — other functions still use it.)

**Then update** `activate`**'s sole caller** so the module stays green — in `DeviceActivationService.verifyResponse` (around line 173), build the canonical from the record's nonce (already in scope; this is the correct final behavior since the record's `registrationNonce` equals the topic nonce the response arrives on — Task 7 only rewrites the routing around it):

```kotlin
        val canonical = DeviceCanonical.activate(
            registrationNonce = activationRecord.registrationNonce,
            nonce = activationRecord.nonce ?: return false,
            issuedAt = activationRecord.issuedAt,
            expiresAt = activationRecord.expiresAt
        )
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.core.device.DeviceCanonicalTest"`
Expected: PASS. The module still compiles because `activate`'s only caller was updated in Step 3. (`DeviceCanonical.activate` has exactly one caller — `DeviceActivationService.verifyResponse`.)

---



## Task 3: Redis key + activation-by-device record → `registration_nonce`

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/core/redis/RedisKeyManager.kt:23`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/redis/DeviceActivationByDeviceRecord.kt`

**Interfaces:**

- Produces: `RedisKeyManager.deviceActivation(registrationNonce: String): String` → `device:activation:{registrationNonce}`; `DeviceActivationByDeviceRecord(registrationNonce: String)`

- [ ] **Step 1: Change the key function signature**

`RedisKeyManager.kt` line 23:

```kotlin
    fun deviceActivation(registrationNonce: String) = "device:activation:$registrationNonce"
```

(`deviceActivationByDevice(deviceId: UUID)` on line 24 is unchanged.)

- [ ] **Step 2: Change the by-device record field**

Replace the body of `DeviceActivationByDeviceRecord.kt`:

```kotlin
package com.thomas.notiguide.domain.device.redis

data class DeviceActivationByDeviceRecord(
    val registrationNonce: String = ""
)
```

(Remove the now-unused `import java.util.UUID`.)

- [ ] **Step 3: Confirm scope**

No test here — this is a type/signature change consumed by Tasks 5–8. Callers will be fixed in those tasks; do not run a full build yet.

---



## Task 4: `DeviceMqttPublisher` — topic keyed by nonce, trimmed envelopes

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/core/device/DeviceMqttPublisher.kt`
- Test: `backend/src/test/kotlin/com/thomas/notiguide/core/device/DeviceMqttPublisherTest.kt`

**Interfaces:**

- Produces:
  - `publishPending(family: DeviceFamily, registrationNonce: String, issuedAt: OffsetDateTime)`
  - `publishRejected(family: DeviceFamily, registrationNonce: String, reason: String)`
  - `publishRejected(kind: DeviceKind, registrationNonce: String, reason: String)`
  - `publishChallenge(kind: DeviceKind, registrationNonce: String, nonce: String, issuedAt: OffsetDateTime, expiresAt: OffsetDateTime)`
  - `publishResult(kind: DeviceKind, registrationNonce: String, publicId: String, assignedDeviceName: String, storeId: String?)`
- Consumes: `MqttClientManager.publish(topic, ByteArray, qos, retained)` (existing)

- [ ] **Step 1: Write the failing test**

```kotlin
package com.thomas.notiguide.core.device

import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import com.thomas.notiguide.core.mqtt.MqttClientManager
import com.thomas.notiguide.core.mqtt.MqttProperties
import com.thomas.notiguide.domain.device.types.DeviceKind
import io.mockk.every
import io.mockk.mockk
import io.mockk.slot
import kotlinx.coroutines.test.runTest
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import java.time.OffsetDateTime

class DeviceMqttPublisherTest {
    private val mqtt = mockk<MqttClientManager>(relaxed = true)
    private val props = mockk<MqttProperties> { every { topicPrefix } returns "notiguide" }
    private val publisher = DeviceMqttPublisher(mqtt, props, jacksonObjectMapper())

    @Test
    fun `challenge publishes to the nonce topic and omits challenge_id and registration_nonce`() = runTest {
        val topic = slot<String>()
        val body = slot<ByteArray>()
        // MqttClientManager.publish(topic, payload, qos, retained) is a plain (non-suspend) fun.
        every { mqtt.publish(capture(topic), capture(body), any(), any()) } returns Unit

        publisher.publishChallenge(
            kind = DeviceKind.TRANSMITTER_HUB,
            registrationNonce = "NONCE_abc-1",
            nonce = "server16b",
            issuedAt = OffsetDateTime.parse("2026-07-09T09:00:00Z"),
            expiresAt = OffsetDateTime.parse("2026-07-09T09:05:00Z")
        )

        assertThat(topic.captured).isEqualTo("notiguide/transmitter/bootstrap/NONCE_abc-1")
        val json = String(body.captured)
        assertThat(json).contains("\"type\":\"challenge\"", "\"nonce\":\"server16b\"", "\"purpose\":\"activate-v1\"")
        assertThat(json).doesNotContain("challenge_id", "registration_nonce")
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.core.device.DeviceMqttPublisherTest"`
Expected: FAIL — `publishChallenge` still takes `challengeId`; body contains `challenge_id`.

- [ ] **Step 3: Rewrite the bootstrap publishers, envelopes, and topic builder**

In `DeviceMqttPublisher.kt` replace the five bootstrap methods (lines 20-100), the topic builders (lines 213-220), and the four envelope classes (lines 222-272) with:

```kotlin
    suspend fun publishPending(
        family: DeviceFamily,
        registrationNonce: String,
        issuedAt: OffsetDateTime
    ) {
        publishJson(
            topic = bootstrapTopic(family, registrationNonce),
            payload = PendingEnvelope(issuedAt = issuedAt.toInstant().toString())
        )
    }

    suspend fun publishRejected(
        family: DeviceFamily,
        registrationNonce: String,
        reason: String
    ) {
        publishJson(
            topic = bootstrapTopic(family, registrationNonce),
            payload = RejectedEnvelope(reason = reason)
        )
    }

    suspend fun publishRejected(
        kind: DeviceKind,
        registrationNonce: String,
        reason: String
    ) {
        publishJson(
            topic = bootstrapTopic(kind, registrationNonce),
            payload = RejectedEnvelope(reason = reason)
        )
    }

    suspend fun publishChallenge(
        kind: DeviceKind,
        registrationNonce: String,
        nonce: String,
        issuedAt: OffsetDateTime,
        expiresAt: OffsetDateTime
    ) {
        publishJson(
            topic = bootstrapTopic(kind, registrationNonce),
            payload = ChallengeEnvelope(
                nonce = nonce,
                issuedAt = issuedAt.toInstant().toString(),
                expiresAt = expiresAt.toInstant().toString()
            )
        )
    }

    suspend fun publishResult(
        kind: DeviceKind,
        registrationNonce: String,
        publicId: String,
        assignedDeviceName: String,
        storeId: String? = null
    ) {
        publishJson(
            topic = bootstrapTopic(kind, registrationNonce),
            payload = ResultEnvelope(
                publicId = publicId,
                assignedDeviceName = assignedDeviceName,
                storeId = storeId
            )
        )
    }
```

Topic builders (replace lines 213-220):

```kotlin
    private fun bootstrapTopic(family: DeviceFamily, registrationNonce: String): String =
        "${mqttProperties.topicPrefix}/${family.topicSegment}/bootstrap/$registrationNonce"

    private fun bootstrapTopic(kind: DeviceKind, registrationNonce: String): String =
        bootstrapTopic(
            family = if (kind.isHub()) DeviceFamily.TRANSMITTER else DeviceFamily.RECEIVER,
            registrationNonce = registrationNonce
        )
```

Envelopes (replace lines 222-272 — drop `challenge_id` everywhere; drop `registration_nonce` from pending + challenge):

```kotlin
    private data class PendingEnvelope(
        @field:JsonProperty("schema_version")
        val schemaVersion: Int = 1,
        val type: String = "pending",
        @field:JsonProperty("issued_at")
        val issuedAt: String
    )

    private data class RejectedEnvelope(
        @field:JsonProperty("schema_version")
        val schemaVersion: Int = 1,
        val type: String = "rejected",
        val reason: String
    )

    private data class ChallengeEnvelope(
        @field:JsonProperty("schema_version")
        val schemaVersion: Int = 1,
        val type: String = "challenge",
        val nonce: String,
        @field:JsonProperty("issued_at")
        val issuedAt: String,
        @field:JsonProperty("expires_at")
        val expiresAt: String,
        val purpose: String = "activate-v1"
    )

    private data class ResultEnvelope(
        @field:JsonProperty("schema_version")
        val schemaVersion: Int = 1,
        val type: String = "result",
        val status: String = "active",
        @field:JsonProperty("public_id")
        val publicId: String,
        @field:JsonProperty("assigned_device_name")
        val assignedDeviceName: String,
        @field:JsonProperty("store_id")
        val storeId: String? = null
    )
```

Remove the now-unused `import java.util.UUID`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.core.device.DeviceMqttPublisherTest"`
Expected: PASS.

---



## Task 5: `DeviceRegistrationService` — `nonce_in_use` guard + key by nonce

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/service/DeviceRegistrationService.kt`
- Test: `backend/src/test/kotlin/com/thomas/notiguide/domain/device/service/DeviceRegistrationServiceTest.kt`

**Interfaces:**

- Consumes: `RedisKeyManager.deviceActivation(String)` (Task 3), `DeviceMqttPublisher.publishRejected/publishPending` (Task 4), `DeviceActivationByDeviceRecord(registrationNonce)` (Task 3)
- Produces: `onRegister(payload, family)` unchanged signature; rejects `nonce_in_use` pre-consume; keys activation by `registration_nonce`

- [ ] **Step 1: Write the failing test**

```kotlin
package com.thomas.notiguide.domain.device.service

import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import com.thomas.notiguide.core.device.DeviceMqttPublisher
import com.thomas.notiguide.core.device.DeviceTransmitterProperties
import com.thomas.notiguide.core.redis.RedisKeyManager
import com.thomas.notiguide.domain.device.repository.DeviceRepository
import com.thomas.notiguide.domain.device.repository.DeviceRfCodeRepository
import com.thomas.notiguide.domain.device.types.DeviceFamily
import io.mockk.coEvery
import io.mockk.coVerify
import io.mockk.every
import io.mockk.mockk
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Test
import org.springframework.data.redis.core.ReactiveRedisTemplate
import reactor.core.publisher.Mono
import java.util.Base64

class DeviceRegistrationServiceTest {
    private val objectMapper = jacksonObjectMapper()
    private val enrollmentTokenService = mockk<EnrollmentTokenService>()
    private val deviceRepository = mockk<DeviceRepository>(relaxed = true)
    private val deviceRfCodeRepository = mockk<DeviceRfCodeRepository>(relaxed = true)
    private val publisher = mockk<DeviceMqttPublisher>(relaxed = true)
    private val txProps = mockk<DeviceTransmitterProperties>(relaxed = true)
    private val redis = mockk<ReactiveRedisTemplate<String, String>>(relaxed = true)
    private val service = DeviceRegistrationService(
        objectMapper, enrollmentTokenService, deviceRepository,
        deviceRfCodeRepository, publisher, txProps, redis
    )

    private fun nonce16(): String =
        Base64.getUrlEncoder().withoutPadding().encodeToString(ByteArray(16) { 1 })

    private fun registerPayload(nonce: String): String =
        """{"schema_version":1,"hardware_model":"ESP32-C3","kind":"TRANSMITTER_HUB",
           "firmware_version":"1.0.0","public_key_b64":"$VALID_P256_B64",
           "enrollment_token":"tok","registration_nonce":"$nonce"}""".trimIndent()

    @Test
    fun `rejects nonce_in_use before consuming the token`() = runTest {
        val nonce = nonce16()
        every { redis.hasKey(RedisKeyManager.deviceActivation(nonce)) } returns Mono.just(true)

        service.onRegister(registerPayload(nonce), DeviceFamily.TRANSMITTER)

        coVerify(exactly = 1) { publisher.publishRejected(DeviceFamily.TRANSMITTER, nonce, "nonce_in_use") }
        coVerify(exactly = 0) { enrollmentTokenService.consume(any()) }
    }

    companion object {
        // A real X.509 DER SubjectPublicKeyInfo for a P-256 key, base64 (generate once with
        // KeyPairGenerator("EC", "secp256r1") and paste .public.encoded base64 here).
        const val VALID_P256_B64 = "<PASTE_REAL_P256_X509_DER_BASE64>"
    }
}
```

> Note for the implementer: replace `VALID_P256_B64` with a real base64 X.509 P-256 public key. Generate it in a scratch test: `Base64.getEncoder().encodeToString(KeyPairGenerator.getInstance("EC").apply { initialize(ECGenParameterSpec("secp256r1")) }.generateKeyPair().public.encoded)`.

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.device.service.DeviceRegistrationServiceTest"`
Expected: FAIL — no `nonce_in_use` guard; `publishRejected` still takes `challengeId: UUID`.

- [ ] **Step 3: Rewrite** `onRegister` **and helpers**

Replace `onRegister` (lines 49-125) with:

```kotlin
    @Transactional
    suspend fun onRegister(
        payload: String,
        family: DeviceFamily
    ) {
        val registration = when (family) {
            DeviceFamily.RECEIVER -> parseReceiverRegistration(payload)
            DeviceFamily.TRANSMITTER -> parseTransmitterRegistration(payload)
        } ?: return

        // nonce_in_use guard — BEFORE consuming the token, so a rejected hub can retry with
        // a fresh nonce and re-present the same (unconsumed) token.
        val activationKey = RedisKeyManager.deviceActivation(registration.registrationNonce)
        if (redis.hasKey(activationKey).awaitSingle()) {
            deviceMqttPublisher.publishRejected(family, registration.registrationNonce, "nonce_in_use")
            return
        }

        val enrollmentRecord = enrollmentTokenService.consume(registration.enrollmentToken)
        if (enrollmentRecord == null) {
            deviceMqttPublisher.publishRejected(family, registration.registrationNonce, "invalid_token")
            return
        }

        if (family == DeviceFamily.TRANSMITTER && enrollmentRecord.storeId != null) {
            val count = deviceRepository.countRegisteredHubsByStoreExcludingPublicKey(
                storeId = enrollmentRecord.storeId,
                publicKeyDer = registration.publicKeyDer
            )
            if (count >= deviceTransmitterProperties.maxRegisteredPerStore) {
                deviceMqttPublisher.publishRejected(family, registration.registrationNonce, "hub_cap_reached")
                return
            }
        }

        val now = OffsetDateTime.now(ZoneOffset.UTC)
        val expiresAt = now.plus(Duration.ofMinutes(15))
        val existing = deviceRepository.findByPublicKeyDer(registration.publicKeyDer)
        val saved = deviceRepository.save(
            Device(
                id = existing?.id,
                publicId = null,
                publicKeyDer = registration.publicKeyDer,
                kind = registration.kind,
                status = DeviceStatus.PENDING,
                assignedName = existing?.assignedName,
                storeId = if (family == DeviceFamily.TRANSMITTER) {
                    enrollmentRecord.storeId ?: existing?.storeId
                } else {
                    existing?.storeId
                },
                firmwareVersion = registration.firmwareVersion,
                lastSeenAt = null,
                activatedAt = null,
                createdAt = existing?.createdAt
            )
        )

        if (existing != null) {
            deviceRfCodeRepository.deleteByDeviceId(saved.id!!)
        }

        val activationRecord = DeviceActivationRecord(
            deviceId = saved.id!!,
            publicKeyFingerprint = sha256Hex(registration.publicKeyDer),
            registrationNonce = registration.registrationNonce,
            nonce = null,
            issuedAt = now,
            expiresAt = expiresAt,
            status = DeviceActivationStatus.PENDING
        )
        val activationByDevice = DeviceActivationByDeviceRecord(registrationNonce = registration.registrationNonce)
        val activationJson = objectMapper.writeValueAsString(activationRecord)
        val activationByDeviceJson = objectMapper.writeValueAsString(activationByDevice)

        writeActivationKeys(saved.id, registration.registrationNonce, activationJson, activationByDeviceJson)
        deviceMqttPublisher.publishPending(
            family = family,
            registrationNonce = registration.registrationNonce,
            issuedAt = now
        )
    }
```

Update the two parse helpers to drop the `challengeId` parameter and use `request.registrationNonce` for their own rejections. Replace their signatures/rejection calls:

```kotlin
    private suspend fun parseReceiverRegistration(payload: String): ParsedRegistration? {
        val request = runCatching { objectMapper.readValue(payload, ReceiverBootstrapRegistration::class.java) }
            .getOrElse {
                log.warn("Ignoring malformed receiver registration payload", it)
                return null
            }
        if (request.schemaVersion != 1 || request.firmwareVersion.isBlank() || !isValidRegistrationNonce(request.registrationNonce)) {
            log.warn("Ignoring invalid receiver registration payload")
            return null
        }
        val kind = runCatching { DeviceKind.valueOf(request.receiverType) }.getOrNull()
        if (kind == null || kind == DeviceKind.TRANSMITTER_HUB || kind == DeviceKind.RECEIVER_433M_PASSIVE) {
            deviceMqttPublisher.publishRejected(DeviceFamily.RECEIVER, request.registrationNonce, "model_radio_mismatch")
            return null
        }
        val publicKeyDer = decodePublicKeyDer(request.publicKeyB64) ?: return null
        return ParsedRegistration(kind, request.firmwareVersion.trim(), publicKeyDer, request.enrollmentToken, request.registrationNonce)
    }

    private suspend fun parseTransmitterRegistration(payload: String): ParsedRegistration? {
        val request = runCatching { objectMapper.readValue(payload, TransmitterBootstrapRegistration::class.java) }
            .getOrElse {
                log.warn("Ignoring malformed transmitter registration payload", it)
                return null
            }
        if (request.schemaVersion != 1 || request.firmwareVersion.isBlank() || !isValidRegistrationNonce(request.registrationNonce)) {
            log.warn("Ignoring invalid transmitter registration payload")
            return null
        }
        val kind = runCatching { DeviceKind.valueOf(request.kind) }.getOrNull()
        if (kind != DeviceKind.TRANSMITTER_HUB) {
            deviceMqttPublisher.publishRejected(DeviceFamily.TRANSMITTER, request.registrationNonce, "model_radio_mismatch")
            return null
        }
        val publicKeyDer = decodePublicKeyDer(request.publicKeyB64) ?: return null
        return ParsedRegistration(DeviceKind.TRANSMITTER_HUB, request.firmwareVersion.trim(), publicKeyDer, request.enrollmentToken, request.registrationNonce)
    }
```



Change `writeActivationKeys` to take `registrationNonce: String` and key by it:

```kotlin
    private suspend fun writeActivationKeys(
        deviceId: UUID,
        registrationNonce: String,
        activationJson: String,
        activationByDeviceJson: String
    ) {
        val ttl = Duration.ofMinutes(15)
        val activationKey = RedisKeyManager.deviceActivation(registrationNonce)
        val activationByDeviceKey = RedisKeyManager.deviceActivationByDevice(deviceId)
        redis.opsForValue().set(activationKey, activationJson, ttl).awaitSingle()
        try {
            redis.opsForValue().set(activationByDeviceKey, activationByDeviceJson, ttl).awaitSingle()
        } catch (ex: Exception) {
            runCatching { redis.delete(activationKey).awaitSingleOrNull() }
            throw ex
        }
    }
```

Raise the nonce floor to 16 bytes:

```kotlin
    private fun isValidRegistrationNonce(value: String): Boolean {
        val decoded = runCatching { Base64.getUrlDecoder().decode(value) }.getOrNull() ?: return false
        return decoded.size >= 16
    }
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.device.service.DeviceRegistrationServiceTest"`
Expected: PASS.

---



## Task 6: `DeviceApprovalService` — `registration_nonce` session chain

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/service/DeviceApprovalService.kt`
- Test: `backend/src/test/kotlin/com/thomas/notiguide/domain/device/service/DeviceApprovalServiceTest.kt`

**Interfaces:**

- Consumes: `DeviceActivationByDeviceRecord(registrationNonce)` (Task 3), `RedisKeyManager.deviceActivation(String)` (Task 3), `DeviceMqttPublisher.publishChallenge/publishRejected(..., registrationNonce, ...)` (Task 4)
- Produces: `approve(deviceId, request, principal)` / `reject(deviceId, principal)` — unchanged public signatures; keyed by `deviceId` (Web Serial + manual both use these)

- [ ] **Step 1: Write the failing test (authored now; runs at the Task 8 gate)**

```kotlin
package com.thomas.notiguide.domain.device.service

import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import com.thomas.notiguide.core.device.DeviceMqttPublisher
import com.thomas.notiguide.core.device.DeviceTransmitterProperties
import com.thomas.notiguide.core.redis.RedisKeyManager
import com.thomas.notiguide.domain.device.entity.Device
import com.thomas.notiguide.domain.device.redis.DeviceActivationByDeviceRecord
import com.thomas.notiguide.domain.device.redis.DeviceActivationRecord
import com.thomas.notiguide.domain.device.repository.DeviceRepository
import com.thomas.notiguide.domain.device.request.ApproveDeviceRequest
import com.thomas.notiguide.domain.device.types.DeviceActivationStatus
import com.thomas.notiguide.domain.device.types.DeviceKind
import com.thomas.notiguide.domain.device.types.DeviceStatus
import com.thomas.notiguide.domain.store.repository.StoreRepository
import com.thomas.notiguide.shared.principal.AdminPrincipal
import com.thomas.notiguide.shared.principal.StoreAccessService
import io.mockk.coEvery
import io.mockk.coVerify
import io.mockk.every
import io.mockk.mockk
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.ObjectProvider
import org.springframework.data.redis.core.ReactiveRedisTemplate
import org.springframework.data.redis.core.ReactiveValueOperations
import reactor.core.publisher.Mono
import java.time.Duration
import java.time.OffsetDateTime
import java.time.ZoneOffset
import java.util.UUID

class DeviceApprovalServiceTest {
    private val deviceRepository = mockk<DeviceRepository>(relaxed = true)
    private val storeRepository = mockk<StoreRepository>(relaxed = true)
    private val deviceQueryService = mockk<DeviceQueryService>(relaxed = true)
    private val redis = mockk<ReactiveRedisTemplate<String, String>>(relaxed = true)
    private val publisher = mockk<DeviceMqttPublisher>(relaxed = true)
    private val publisherProvider = mockk<ObjectProvider<DeviceMqttPublisher>> { every { ifAvailable } returns publisher }
    private val txProps = mockk<DeviceTransmitterProperties> { every { maxRegisteredPerStore } returns 10 }
    private val storeAccess = mockk<StoreAccessService>(relaxed = true)
    private val objectMapper = jacksonObjectMapper()
    private val service = DeviceApprovalService(
        deviceRepository, storeRepository, deviceQueryService, objectMapper,
        redis, publisherProvider, txProps, storeAccess
    )

    @Test
    fun `approve issues a challenge on the record's registration nonce`() = runTest {
        val deviceId = UUID.randomUUID()
        val storeId = UUID.randomUUID()
        val nonce = "N16_urlsafe_nonce_x"
        val now = OffsetDateTime.now(ZoneOffset.UTC)
        val valueOps = mockk<ReactiveValueOperations<String, String>>()
        every { redis.opsForValue() } returns valueOps
        every { valueOps.get(RedisKeyManager.deviceActivationByDevice(deviceId)) } returns
            Mono.just(objectMapper.writeValueAsString(DeviceActivationByDeviceRecord(registrationNonce = nonce)))
        every { valueOps.get(RedisKeyManager.deviceActivation(nonce)) } returns
            Mono.just(objectMapper.writeValueAsString(DeviceActivationRecord(
                deviceId = deviceId, registrationNonce = nonce, status = DeviceActivationStatus.PENDING,
                issuedAt = now, expiresAt = now.plusMinutes(15))))
        every { valueOps.set(any(), any(), any<Duration>()) } returns Mono.just(true)
        coEvery { deviceRepository.findById(deviceId) } returns Device(
            id = deviceId, publicKeyDer = ByteArray(1), kind = DeviceKind.TRANSMITTER_HUB,
            status = DeviceStatus.PENDING, storeId = storeId)
        coEvery { deviceRepository.save(any()) } answers { firstArg() }
        coEvery { deviceRepository.countRegisteredHubsByStoreExcludingPublicKey(any(), any()) } returns 0L
        coEvery { storeRepository.findById(storeId) } returns mockk(relaxed = true)
        coEvery { deviceQueryService.getRequiredDeviceDetailById(any()) } returns mockk(relaxed = true)

        service.approve(deviceId, ApproveDeviceRequest(assignedName = "Hub A", storeId = storeId), mockk<AdminPrincipal>(relaxed = true))

        coVerify { publisher.publishChallenge(DeviceKind.TRANSMITTER_HUB, nonce, any(), any(), any()) }
    }
}
```

> If `ApproveDeviceRequest` has additional required fields, add them to the constructor call. `storeAccess`/`principal` are relaxed (no throw) so the org fence is a no-op in this unit test.

- [ ] **Step 2: (authored — executes at Task 8)**

Intent pre-impl: `challengeLink.challengeId` / `publishChallenge(challengeId = …)` no longer resolve after Tasks 3–4; the impl below restores them via `registrationNonce`.

- [ ] **Step 3: Update the record chain**

In `approve` (lines 83-108) replace the challenge-link block:

```kotlin
        val challengeLink = readActivationByDevice(device.id!!)
        val activationRecord = readActivationRecord(challengeLink.registrationNonce)
        val now = OffsetDateTime.now(ZoneOffset.UTC)
        val updatedActivation = activationRecord.copy(
            nonce = generateChallengeNonce(),
            issuedAt = now,
            expiresAt = now.plus(CHALLENGE_LIFETIME),
            status = DeviceActivationStatus.ISSUED
        )
        val saved = deviceRepository.save(device.copy(assignedName = assignedName, storeId = request.storeId))
        writeActivationState(saved.id!!, challengeLink.registrationNonce, updatedActivation)
        publisher.publishChallenge(
            kind = saved.kind,
            registrationNonce = challengeLink.registrationNonce,
            nonce = requireNotNull(updatedActivation.nonce),
            issuedAt = updatedActivation.issuedAt,
            expiresAt = updatedActivation.expiresAt
        )
```

In `reject` (lines 124-132):

```kotlin
        val challengeLink = readActivationByDevice(device.id!!)
        readActivationRecord(challengeLink.registrationNonce)
        val saved = deviceRepository.save(device.copy(status = DeviceStatus.REJECTED))
        publisher.publishRejected(saved.kind, challengeLink.registrationNonce, "admin_rejected")
        runCatching { deleteActivationState(saved.id!!, challengeLink.registrationNonce) }
            .onFailure { ex -> log.warn("Failed to clear activation state for rejected device {}", saved.id, ex) }
```

Change `readActivationRecord`, `writeActivationState`, `deleteActivationState` to take `registrationNonce: String` and key via `RedisKeyManager.deviceActivation(registrationNonce)`, and change the by-device write to `DeviceActivationByDeviceRecord(registrationNonce = registrationNonce)`:

```kotlin
    private suspend fun readActivationRecord(registrationNonce: String): DeviceActivationRecord {
        val key = RedisKeyManager.deviceActivation(registrationNonce)
        val payload = redis.opsForValue().get(key).awaitSingleOrNull()
            ?: throw DeviceConflictEnvelopeException("activation_session_missing")
        return runCatching { objectMapper.readValue(payload, DeviceActivationRecord::class.java) }
            .getOrElse { throw ConflictException("Device activation session is malformed") }
    }

    private suspend fun writeActivationState(deviceId: UUID, registrationNonce: String, activationRecord: DeviceActivationRecord) {
        val activationJson = objectMapper.writeValueAsString(activationRecord)
        val activationByDeviceJson = objectMapper.writeValueAsString(DeviceActivationByDeviceRecord(registrationNonce = registrationNonce))
        redis.opsForValue().set(RedisKeyManager.deviceActivation(registrationNonce), activationJson, ACTIVATION_TTL).awaitSingle()
        try {
            redis.opsForValue().set(RedisKeyManager.deviceActivationByDevice(deviceId), activationByDeviceJson, ACTIVATION_TTL).awaitSingle()
        } catch (ex: Exception) {
            redis.delete(RedisKeyManager.deviceActivation(registrationNonce)).awaitSingleOrNull()
            throw ex
        }
    }

    private suspend fun deleteActivationState(deviceId: UUID, registrationNonce: String) {
        redis.delete(RedisKeyManager.deviceActivation(registrationNonce)).awaitSingleOrNull()
        redis.delete(RedisKeyManager.deviceActivationByDevice(deviceId)).awaitSingleOrNull()
    }
```

- [ ] **Step 4: Run test to verify it passes**



Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.device.service.DeviceApprovalServiceTest"`
Expected: PASS.

---



## Task 7: `DeviceActivationService.onResponse` — key by nonce, verify via `DeviceSignatureVerifier`

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/service/DeviceActivationService.kt`
- Test: `backend/src/test/kotlin/com/thomas/notiguide/domain/device/service/DeviceActivationServiceTest.kt`

**Interfaces:**

- Consumes: `DeviceSignatureVerifier.verify(...)` (Task 1), `DeviceCanonical.activate(registrationNonce, ...)` (Task 2), `RedisKeyManager.deviceActivation(String)` (Task 3)
- Produces: `onResponse(payload: String, registrationNonce: String)`

- [ ] **Step 1: Write the failing test (authored now; runs at the Task 8 gate — see Part A note)**

```kotlin
package com.thomas.notiguide.domain.device.service

import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import com.thomas.notiguide.core.device.DeviceCanonical
import com.thomas.notiguide.core.device.DeviceMqttPublisher
import com.thomas.notiguide.core.device.DevicePublicIdMinter
import com.thomas.notiguide.core.redis.RedisKeyManager
import com.thomas.notiguide.domain.device.entity.Device
import com.thomas.notiguide.domain.device.redis.DeviceActivationRecord
import com.thomas.notiguide.domain.device.repository.DeviceRepository
import com.thomas.notiguide.domain.device.types.DeviceActivationStatus
import com.thomas.notiguide.domain.device.types.DeviceKind
import com.thomas.notiguide.domain.device.types.DeviceStatus
import io.mockk.coEvery
import io.mockk.coVerify
import io.mockk.every
import io.mockk.mockk
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Test
import org.springframework.data.redis.core.ReactiveRedisTemplate
import org.springframework.data.redis.core.ReactiveValueOperations
import reactor.core.publisher.Mono
import java.security.KeyPairGenerator
import java.security.Signature
import java.security.spec.ECGenParameterSpec
import java.time.OffsetDateTime
import java.time.ZoneOffset
import java.util.Base64
import java.util.UUID

class DeviceActivationServiceTest {
    private val objectMapper = jacksonObjectMapper()
    private val redis = mockk<ReactiveRedisTemplate<String, String>>(relaxed = true)
    private val deviceRepository = mockk<DeviceRepository>(relaxed = true)
    private val publisher = mockk<DeviceMqttPublisher>(relaxed = true)
    private val minter = mockk<DevicePublicIdMinter>()
    private val rfCodeService = mockk<RfCodeService>(relaxed = true)
    private val service = DeviceActivationService(objectMapper, redis, deviceRepository, publisher, minter, rfCodeService)

    private fun arrange(): Triple<java.security.KeyPair, String, DeviceActivationRecord> {
        val kp = KeyPairGenerator.getInstance("EC")
            .apply { initialize(ECGenParameterSpec("secp256r1")) }.generateKeyPair()
        val nonce = "reg_nonce_16b_urlsafe"
        val deviceId = UUID.randomUUID()
        val issued = OffsetDateTime.now(ZoneOffset.UTC)
        val record = DeviceActivationRecord(
            deviceId = deviceId, publicKeyFingerprint = "fp", registrationNonce = nonce,
            nonce = "server16", issuedAt = issued, expiresAt = issued.plusMinutes(5),
            status = DeviceActivationStatus.ISSUED
        )
        val valueOps = mockk<ReactiveValueOperations<String, String>>()
        every { redis.opsForValue() } returns valueOps
        every { valueOps.get(any()) } returns Mono.empty()
        every { valueOps.get(RedisKeyManager.deviceActivation(nonce)) } returns
            Mono.just(objectMapper.writeValueAsString(record))
        every { redis.delete(any<String>()) } returns Mono.just(1L)
        val device = Device(
            id = deviceId, publicKeyDer = kp.public.encoded, kind = DeviceKind.TRANSMITTER_HUB,
            status = DeviceStatus.PENDING, assignedName = "Hub A", storeId = UUID.randomUUID()
        )
        coEvery { deviceRepository.findById(deviceId) } returns device
        coEvery { deviceRepository.save(any()) } answers { firstArg() }
        coEvery { minter.mint(DeviceKind.TRANSMITTER_HUB) } returns "HUB-123" // mint is a suspend fun
        return Triple(kp, nonce, record)
    }

    private fun responsePayload(kp: java.security.KeyPair, canonical: String): String {
        val sig = Signature.getInstance("SHA256withECDSA")
            .apply { initSign(kp.private); update(canonical.toByteArray(Charsets.UTF_8)) }.sign()
        return """{"schema_version":1,"type":"response","signature_b64":"${Base64.getEncoder().encodeToString(sig)}"}"""
    }

    @Test
    fun `valid response activates the hub`() = runTest {
        val (kp, nonce, record) = arrange()
        val canonical = DeviceCanonical.activate(nonce, record.nonce!!, record.issuedAt, record.expiresAt)
        service.onResponse(responsePayload(kp, canonical), nonce)
        coVerify { deviceRepository.save(match { it.status == DeviceStatus.ACTIVE && it.publicId == "HUB-123" }) }
        coVerify { publisher.publishResult(DeviceKind.TRANSMITTER_HUB, nonce, "HUB-123", "Hub A", any()) }
    }

    @Test
    fun `tampered signature does not activate`() = runTest {
        val (kp, nonce, record) = arrange()
        val canonical = DeviceCanonical.activate(nonce, record.nonce!!, record.issuedAt, record.expiresAt)
        service.onResponse(responsePayload(kp, canonical + "TAMPER"), nonce)
        coVerify(exactly = 0) { deviceRepository.save(match { it.status == DeviceStatus.ACTIVE }) }
    }
}
```

- [ ] **Step 2: (authored — executes at Task 8) Confirm the test fails pre-refactor**

The module is mid-refactor; this test compiles and runs at the Task 8 gate. Its pre-implementation intent: `onResponse` still takes `challengeId: UUID` (won't compile) → the impl below fixes it.

- [ ] **Step 3: Rewrite** `onResponse`**,** `loadActivationRecord`**,** `verifyResponse`**, envelope**

Change the signature and routing (replace lines 43-73 region as follows):

```kotlin
    @Transactional
    suspend fun onResponse(
        payload: String,
        registrationNonce: String
    ) {
        val response = runCatching {
            objectMapper.readValue(payload, ActivationResponseEnvelope::class.java)
        }.getOrElse {
            log.warn("Ignoring malformed device activation response for nonce {}", registrationNonce, it)
            return
        }
        if (response.schemaVersion != 1 || response.type != "response") {
            return
        }

        val activationRecord = loadActivationRecord(registrationNonce) ?: return
        if (activationRecord.status != DeviceActivationStatus.ISSUED) return
        val now = OffsetDateTime.now(ZoneOffset.UTC)
        if (activationRecord.expiresAt.isBefore(now)) return

        val device = deviceRepository.findById(activationRecord.deviceId) ?: return
        if (device.status != DeviceStatus.PENDING) return
        if (!verifyResponse(device, registrationNonce, activationRecord, response.signatureB64)) {
            log.warn("Ignoring activation response with invalid signature for device {}", device.id)
            return
        }
        // ... unchanged: mint publicId, save ACTIVE, publishResult, clear state ...
```

Where the tail previously called `deleteActivationState(saved.id!!, challengeId)` and `publishResult(..., challengeId = challengeId, ...)`, replace with `registrationNonce`:

```kotlin
        deleteActivationState(saved.id!!, registrationNonce)
        // ...
        deviceMqttPublisher.publishResult(
            kind = saved.kind,
            registrationNonce = registrationNonce,
            publicId = newPublicId,
            assignedDeviceName = requireNotNull(saved.assignedName) { "Approved devices must have an assigned name before activation" },
            storeId = activationStoreId
        )
```

`loadActivationRecord` / `deleteActivationState` take `registrationNonce: String` and key via `RedisKeyManager.deviceActivation(registrationNonce)` (the by-device delete still uses `deviceActivationByDevice(deviceId)`).

`verifyResponse` uses the shared verifier and the nonce-based canonical:

```kotlin
    private fun verifyResponse(
        device: Device,
        registrationNonce: String,
        activationRecord: DeviceActivationRecord,
        signatureB64: String
    ): Boolean {
        val publicKeyDer = device.publicKeyDer ?: return false
        val canonical = DeviceCanonical.activate(
            registrationNonce = registrationNonce,
            nonce = activationRecord.nonce ?: return false,
            issuedAt = activationRecord.issuedAt,
            expiresAt = activationRecord.expiresAt
        )
        return DeviceSignatureVerifier.verify(publicKeyDer, canonical, signatureB64)
    }
```

Drop `challenge_id` from the response envelope:

```kotlin
private data class ActivationResponseEnvelope(
    @field:JsonProperty("schema_version")
    val schemaVersion: Int = 0,
    val type: String = "",
    @field:JsonProperty("signature_b64")
    val signatureB64: String = ""
)
```

Remove now-unused crypto imports (`KeyFactory`, `Signature`, `ECPublicKey`, `X509EncodedKeySpec`, `Base64`) if no longer referenced.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.device.service.DeviceActivationServiceTest"`
Expected: PASS.

---



## Task 8: Bootstrap listeners — route `response` by nonce from topic

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/listener/TransmitterBootstrapListener.kt:80-95`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/listener/DeviceBootstrapListener.kt` (mirror; unused receiver path, kept consistent)

**Interfaces:**

- Consumes: `DeviceActivationService.onResponse(payload, registrationNonce: String)` (Task 7)

- [ ] **Step 1: Update the response routing**

In `TransmitterBootstrapListener.kt`, replace the challenge-id extraction (lines 81-94) with nonce extraction from the topic:

```kotlin
        if (envelopeType != "response") {
            return@MqttMessageHandler
        }

        val registrationNonce = topic
            .removePrefix("${mqttProperties.topicPrefix}/transmitter/bootstrap/")
            .takeIf { it.isNotBlank() && it != topic } ?: return@MqttMessageHandler

        scope.launch {
            runCatching {
                deviceActivationService.onResponse(payload = payloadStr, registrationNonce = registrationNonce)
            }.onFailure { ex ->
                log.warn("Transmitter activation response handling failed", ex)
            }
        }
```

Remove the now-unused `import java.util.UUID`.

- [ ] **Step 2: Mirror the change in** `DeviceBootstrapListener.kt` **(receiver segment)**

`DeviceBootstrapListener` currently calls `onResponse(payload, challengeId = UUID.fromString(topic.removePrefix(".../receiver/bootstrap/")))`. Replace that block with:

```kotlin
        val registrationNonce = topic
            .removePrefix("${mqttProperties.topicPrefix}/receiver/bootstrap/")
            .takeIf { it.isNotBlank() && it != topic } ?: return@MqttMessageHandler

        scope.launch {
            runCatching {
                deviceActivationService.onResponse(payload = payloadStr, registrationNonce = registrationNonce)
            }.onFailure { ex ->
                log.warn("Receiver activation response handling failed", ex)
            }
        }
```

Remove the now-unused `import java.util.UUID`. (This path is unused in this deployment — receivers are hub-paired — but it shares `DeviceActivationService`, so it must compile.)

- [ ] **Step 3: Part A compile gate**

This is the point where the atomic `challenge_id` → `registration_nonce` refactor (Tasks 3–8) first compiles.

Run: `cd backend && ./gradlew compileKotlin compileTestKotlin`
Expected: SUCCESS — all `challenge_id` references gone; Tasks 2–8 type-check together. If this fails, a caller was missed — grep `challengeId` / `challenge_id` under `backend/src/main` and fix before proceeding.

- [ ] **Step 4: Part A test gate — run every test authored in Tasks 3–8**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.device.*" --tests "com.thomas.notiguide.core.device.*"`
Expected: PASS — this executes the DeviceMqttPublisher (Task 4), DeviceRegistrationService (Task 5), DeviceApprovalService (Task 6), and DeviceActivationService (Task 7) tests that could not run mid-refactor, plus Tasks 1–2.

- [ ] **Step 5: Confirm no** `challenge_id` **remains**

Run: `grep -rn "challengeId\|challenge_id" backend/src/main/kotlin/com/thomas/notiguide/{core/device,core/redis,domain/device}`
Expected: no matches (the entire identifier is retired).

---



## Task 9: Firmware — bootstrap re-key (topic + handlers + `activate-v1`)

**Files:**

- Modify: `transmitter/main/network/mqtt.c` (`bootstrap_session_t`, `mqtt_begin_bootstrap`, `mqtt_subscribe_current_phase`, `handle_bootstrap_pending/challenge/result`)

*(No firmware unit harness — verified via the Task 2/11 backend golden canonicals + manual flash. No commit step.)*

- [ ] **Step 1: Drop** `challenge_id` **from the session; keep** `challenge_topic` **= nonce topic**

In `bootstrap_session_t` (lines 42-48) remove the `challenge_id` member (keep `registration_nonce`, `challenge_topic`, add `retry_count` used in Task 10):

```c
typedef struct {
    bool active;
    bool register_published;
    uint8_t retry_count;
    char registration_nonce[DEVICE_IDENTITY_REGISTRATION_NONCE_MAX_LEN];
    char challenge_topic[128];
} bootstrap_session_t;
```

- [ ] **Step 2: Set the nonce topic at session start and subscribe to it directly**

In `mqtt_begin_bootstrap` (lines 657-668), after generating the nonce, build `challenge_topic` and subscribe to it instead of the wildcard:

```c
    mqtt_reset_bootstrap_session();
    ESP_RETURN_ON_ERROR(device_identity_generate_registration_nonce(s_bootstrap.registration_nonce,
                                                                   sizeof(s_bootstrap.registration_nonce)),
                        TAG, "registration nonce");
    (void)snprintf(s_bootstrap.challenge_topic, sizeof(s_bootstrap.challenge_topic),
                   TOPIC_PREFIX "transmitter/bootstrap/%s", s_bootstrap.registration_nonce);
    s_bootstrap.active = true;
    xEventGroupClearBits(s_mqtt_events, MQTT_BIT_BOOTSTRAP_DONE | MQTT_BIT_BOOTSTRAP_FAILED);

    if (mqtt_is_connected()) {
        (void)esp_mqtt_client_subscribe(s_client, s_bootstrap.challenge_topic, 1);
        return mqtt_publish_register();
    }
    return ESP_OK;
```

In `mqtt_subscribe_current_phase` (lines 170-180) replace the `challenge_topic`/`+` branch — `challenge_topic` is always set now:

```c
    if (!s_bootstrap.active) {
        return;
    }
    (void)esp_mqtt_client_subscribe(s_client, s_bootstrap.challenge_topic, 1);
    ESP_LOGI(TAG, "Subscribed to bootstrap topic");
    (void)mqtt_publish_register();
```

- [ ] **Step 3: Simplify** `handle_bootstrap_pending` **(no challenge_id, no nonce echo)**

The message already arrived on the hub's own topic; `pending` now carries only `issued_at`:

```c
static void handle_bootstrap_pending(cJSON *root)
{
    char issued_at[40];
    if (json_copy_string(root, "issued_at", issued_at, sizeof(issued_at))) {
        (void)time_utils_sync_from_iso8601(issued_at);
    }
    ESP_LOGI(TAG, "Bootstrap pending");
}
```

- [ ] **Step 4: Sign** `activate-v1|{registration_nonce}|…` **in** `handle_bootstrap_challenge`

`challenge` now carries only `nonce`, `issued_at`, `expires_at`, `purpose`. Replace lines 232-291:

```c
static void handle_bootstrap_challenge(cJSON *root)
{
    char nonce[64];
    char issued_at[40];
    char expires_at[40];
    char purpose[24];
    if (!json_copy_string(root, "nonce", nonce, sizeof(nonce)) ||
        !json_copy_string(root, "issued_at", issued_at, sizeof(issued_at)) ||
        !json_copy_string(root, "expires_at", expires_at, sizeof(expires_at)) ||
        !json_copy_string(root, "purpose", purpose, sizeof(purpose))) {
        return;
    }
    if (strcmp(purpose, "activate-v1") != 0) {
        return;
    }
    (void)time_utils_sync_from_iso8601(issued_at);

    char canonical[320];
    char signature_b64[DEVICE_IDENTITY_SIGNATURE_B64_MAX_LEN];
    (void)snprintf(canonical, sizeof(canonical), "activate-v1|%s|%s|%s|%s",
                   s_bootstrap.registration_nonce, nonce, issued_at, expires_at);
    if (device_identity_sign_text(canonical, signature_b64, sizeof(signature_b64)) != ESP_OK) {
        ESP_LOGE(TAG, "activation signing failed");
        return;
    }

    char payload[320];
    const int payload_len = snprintf(payload, sizeof(payload),
                                     "{\"schema_version\":1,\"type\":\"response\",\"signature_b64\":\"%s\"}",
                                     signature_b64);
    (void)mqtt_publish(s_bootstrap.challenge_topic, payload, payload_len, 1, 0);
    ESP_LOGI(TAG, "Bootstrap challenge response sent");
}
```

- [ ] **Step 5: Simplify** `handle_bootstrap_result` **(no challenge_id match)**

Replace lines 293-307 head:

```c
static void handle_bootstrap_result(cJSON *root)
{
    char status[16];
    char public_id[DEVICE_CFG_PUBLIC_ID_MAX_LEN];
    char assigned_device_name[DEVICE_CFG_DEVICE_NAME_MAX_LEN];
    char store_id[DEVICE_CFG_STORE_ID_MAX_LEN];

    if (!json_copy_string(root, "status", status, sizeof(status)) || strcmp(status, "active") != 0) {
        return;
    }
    // ... unchanged: read public_id, assigned_device_name, store_id; commit activation ...
```

- [ ] **Step 6: Manual verification**

Flash a hub with a matching backend. Confirm via serial log: `Subscribed to bootstrap topic` (single nonce topic, no `+`), register published, `pending`, challenge response sent, activation confirmed. Confirm the backend accepts the `activate-v1|{registration_nonce}|…` signature (device reaches ACTIVE).

---



## Task 10: Firmware — `nonce_in_use` auto-retry

**Files:**

- Modify: `transmitter/main/network/mqtt.c` (`handle_bootstrap_rejected`, dispatch call site line 365)

- [ ] **Step 1: Add the retry cap constant**

Near the top of `mqtt.c` (with the other file-scope defines):

```c
#define BOOTSTRAP_MAX_IMMEDIATE_RETRIES 10U
```

- [ ] **Step 2: Rewrite** `handle_bootstrap_rejected` **to branch on** `reason`

```c
static void handle_bootstrap_rejected(cJSON *root)
{
    char reason[32] = "";
    (void)json_copy_string(root, "reason", reason, sizeof(reason));

    if (strcmp(reason, "nonce_in_use") == 0) {
        if (s_bootstrap.retry_count < BOOTSTRAP_MAX_IMMEDIATE_RETRIES) {
            s_bootstrap.retry_count++;
            (void)esp_mqtt_client_unsubscribe(s_client, s_bootstrap.challenge_topic);
            if (device_identity_generate_registration_nonce(s_bootstrap.registration_nonce,
                                                            sizeof(s_bootstrap.registration_nonce)) != ESP_OK) {
                ESP_LOGE(TAG, "nonce regeneration failed");
                return;
            }
            (void)snprintf(s_bootstrap.challenge_topic, sizeof(s_bootstrap.challenge_topic),
                           TOPIC_PREFIX "transmitter/bootstrap/%s", s_bootstrap.registration_nonce);
            s_bootstrap.register_published = false;
            (void)esp_mqtt_client_subscribe(s_client, s_bootstrap.challenge_topic, 1);
            (void)mqtt_publish_register();
            ESP_LOGW(TAG, "nonce_in_use: retrying with a fresh registration nonce (%u)", s_bootstrap.retry_count);
            return;
        }
        // Cap hit: reset session but KEEP the enroll token, and signal FAILED so the
        // higher-level provisioning loop re-arms bootstrap on its own backoff cadence.
        ESP_LOGE(TAG, "nonce_in_use retry cap reached; deferring to provisioning loop");
        mqtt_reset_bootstrap_session();
        xEventGroupSetBits(s_mqtt_events, MQTT_BIT_BOOTSTRAP_FAILED);
        return;
    }

    // Terminal rejections (invalid_token, hub_cap_reached, model_radio_mismatch, admin_rejected).
    ESP_LOGW(TAG, "Bootstrap rejected by backend: %s", reason[0] ? reason : "(none)");
    mqtt_reset_bootstrap_session();
    (void)device_config_clear_enroll_token();
    xEventGroupSetBits(s_mqtt_events, MQTT_BIT_BOOTSTRAP_FAILED);
}
```

- [ ] **Step 3: Pass** `root` **at the dispatch site**

In `bootstrap_listener_dispatch` (line 364-365) change:

```c
    } else if (strcmp(type, "rejected") == 0) {
        handle_bootstrap_rejected(root);
```

- [ ] **Step 4: Manual verification**

With the backend, force a collision by pre-seeding `device:activation:{nonce}` in Redis for a nonce the hub will use (or temporarily hard-code a repeated nonce in a scratch build). Confirm the serial log shows `nonce_in_use: retrying with a fresh registration nonce (1)` and the hub proceeds to `pending` on the new topic; confirm the enroll token is **not** cleared during retry. Confirm a `rejected` with reason `invalid_token` still clears the token and fails terminally.

---



## Task 11: `DeviceCanonical` — device→backend canonicals (`roster-update-v1`, `ack-v1`, `heartbeat-v1`)

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/core/device/DeviceCanonical.kt`
- Test: `backend/src/test/kotlin/com/thomas/notiguide/core/device/DeviceCanonicalTest.kt` (extend)

**Interfaces:**

- Produces:
  - `data class RosterCanonicalReceiver(val slot: Int, val band: String, val label: String)`
  - `rosterUpdate(hubPublicId: String, seq: Int, receivers: List<RosterCanonicalReceiver>): String`
  - `ack(hubPublicId: String, ackFor: String, id: String, status: String): String`
  - `heartbeat(hubPublicId: String, issuedAtRaw: String, heapPct: Int, rssi: String, uptimeMs: Long, dispD: Long, dispT: Long, ip: String): String`

- [ ] **Step 1: Write the failing golden tests**

```kotlin
    @Test
    fun `rosterUpdate sorts receivers by slot and inlines slot colon band colon label`() {
        val receivers = listOf(
            DeviceCanonical.RosterCanonicalReceiver(2, "2.4G", ""),
            DeviceCanonical.RosterCanonicalReceiver(1, "433M", "Table 1")
        )
        assertThat(DeviceCanonical.rosterUpdate("hub-9", 7, receivers))
            .isEqualTo("roster-update-v1|hub-9|7|1:433M:Table 1|2:2.4G:")
        assertThat(DeviceCanonical.rosterUpdate("hub-9", 0, emptyList()))
            .isEqualTo("roster-update-v1|hub-9|0")
    }

    @Test
    fun `ack and heartbeat canonicals`() {
        assertThat(DeviceCanonical.ack("hub-9", "transmit", "d-uuid", "applied"))
            .isEqualTo("ack-v1|hub-9|transmit|d-uuid|applied")
        assertThat(DeviceCanonical.heartbeat("hub-9", "2026-07-09T09:15:00Z", 42, "-58", 123456L, 12L, 340L, "192.168.1.50"))
            .isEqualTo("heartbeat-v1|hub-9|2026-07-09T09:15:00Z|42|-58|123456|12|340|192.168.1.50")
        assertThat(DeviceCanonical.heartbeat("hub-9", "2026-07-09T09:15:00Z", 0, "", 5L, 0L, 0L, ""))
            .isEqualTo("heartbeat-v1|hub-9|2026-07-09T09:15:00Z|0||5|0|0|")
    }
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.core.device.DeviceCanonicalTest"`
Expected: FAIL — new functions unresolved.

- [ ] **Step 3: Add the canonicals**

Append to the `DeviceCanonical` object:

```kotlin
    data class RosterCanonicalReceiver(val slot: Int, val band: String, val label: String)

    fun rosterUpdate(
        hubPublicId: String,
        seq: Int,
        receivers: List<RosterCanonicalReceiver>
    ): String {
        val head = "roster-update-v1|$hubPublicId|$seq"
        if (receivers.isEmpty()) return head
        val body = receivers.sortedBy { it.slot }
            .joinToString("|") { "${it.slot}:${it.band}:${it.label}" }
        return "$head|$body"
    }

    fun ack(
        hubPublicId: String,
        ackFor: String,
        id: String,
        status: String
    ): String = "ack-v1|$hubPublicId|$ackFor|$id|$status"

    fun heartbeat(
        hubPublicId: String,
        issuedAtRaw: String,
        heapPct: Int,
        rssi: String,
        uptimeMs: Long,
        dispD: Long,
        dispT: Long,
        ip: String
    ): String = "heartbeat-v1|$hubPublicId|$issuedAtRaw|$heapPct|$rssi|$uptimeMs|$dispD|$dispT|$ip"
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.core.device.DeviceCanonicalTest"`
Expected: PASS. These golden strings are the firmware byte-contract (Tasks 14–16).

---



## Task 12: `RosterSyncListener` — verify `roster-update-v1`

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/listener/RosterSyncListener.kt`
- Test: `backend/src/test/kotlin/com/thomas/notiguide/domain/device/listener/RosterSyncListenerTest.kt` (or a focused apply test; verify path unit-level)

**Interfaces:**

- Consumes: `DeviceSignatureVerifier.verify` (Task 1), `DeviceCanonical.rosterUpdate` (Task 11)

- [ ] **Step 1: Add** `public_key_der` **to the hub lookup + a signature field to the envelope**

Extend `lookupHub` SQL to `SELECT id, store_id, last_roster_seq, public_key_der`, add `publicKeyDer: ByteArray?` to `HubRecord`, and add `signatureB64` to `RosterUpdateEnvelope`:

```kotlin
private data class RosterUpdateEnvelope(
    @field:JsonProperty("schema_version")
    val schemaVersion: Int = 0,
    val seq: Int = 0,
    val receivers: List<RosterReceiverEntry> = emptyList(),
    @field:JsonProperty("signature_b64")
    val signatureB64: String = ""
)
```

`mapHubRecord` reads `row.get("public_key_der", ByteArray::class.java)`.

- [ ] **Step 2: Verify before applying**

In `handleRosterUpdate`, after `lookupHub` returns a non-null `hub`, insert:

```kotlin
        val publicKeyDer = hub.publicKeyDer
        if (publicKeyDer == null) {
            log.warn("Roster update for hub {} with no stored public key; dropping", publicId)
            return
        }
        val canonical = DeviceCanonical.rosterUpdate(
            hubPublicId = publicId,
            seq = roster.seq,
            receivers = roster.receivers.map {
                DeviceCanonical.RosterCanonicalReceiver(slot = it.slot, band = it.band, label = it.label ?: "")
            }
        )
        if (!DeviceSignatureVerifier.verify(publicKeyDer, canonical, roster.signatureB64)) {
            log.warn("Dropping roster update with invalid signature for hub {}", publicId)
            return
        }
```

- [ ] **Step 3: Verify coverage + compile**

The security-critical pieces are already unit-tested with real P-256 keys: signature verification (`DeviceSignatureVerifierTest`, Task 1) and the exact `roster-update-v1` bytes incl. slot-sort and null-label (`DeviceCanonicalTest`, Task 11). The listener change is thin glue (lookup key → build canonical → `verify` → drop-or-apply), so it is verified by compile + the Part B suite + manual integration (Step 4), not a brittle reflection test over the private handler.

Run: `cd backend && ./gradlew compileKotlin compileTestKotlin && ./gradlew test --tests "com.thomas.notiguide.core.device.*"`
Expected: SUCCESS + PASS (Part B is additive — it compiles independently of Part A once Part A is merged).

- [ ] **Step 4: Manual integration verification**

Pair a receiver so the hub publishes a `roster/update` (Task 15 firmware signs it); confirm the backend applies it (device appears, no `invalid signature` warning). Temporarily corrupt one byte of the firmware canonical in a scratch build → confirm the backend logs `Dropping roster update with invalid signature` and does not apply.

---



## Task 13: `TransmitterOperationalListener` — verify `ack` + `heartbeat`

**Files:**

- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/listener/TransmitterOperationalListener.kt`
- Test: `backend/src/test/kotlin/com/thomas/notiguide/domain/device/listener/TransmitterOperationalListenerTest.kt`

**Interfaces:**

- Consumes: `DeviceSignatureVerifier.verify` (Task 1), `DeviceCanonical.ack` / `DeviceCanonical.heartbeat` (Task 11)

- [ ] **Step 1: Add a** `public_key_der` **lookup + envelope signature fields**

Add a suspend `lookupHubKey(publicId): ByteArray?` (`SELECT public_key_der FROM device WHERE public_id = :publicId AND kind = 'TRANSMITTER_HUB'`). Add `commandId: UUID?` and `signatureB64: String` to `TransmitterAckEnvelope`; add a raw `issuedAt: String?` and `signatureB64: String` to `TransmitterHeartbeatEnvelope` (read `issued_at` as a **raw String** for the canonical; parse it separately for the freshness check).

- [ ] **Step 2: Verify** `ack` **before routing**

At the top of `handleAck`, after parsing:

```kotlin
        val keyDer = lookupHubKey(publicId) ?: run {
            log.warn("Ack from hub {} with no stored public key; dropping", publicId); return
        }
        val id = when (ack.ackFor) {
            "transmit" -> ack.dispatchId?.toString()
            "deact" -> ack.commandId?.toString()
            else -> null
        } ?: run { log.warn("Ack from {} missing id for ack_for={}", publicId, ack.ackFor); return }
        val canonical = DeviceCanonical.ack(publicId, ack.ackFor, id, ack.status)
        if (!DeviceSignatureVerifier.verify(keyDer, canonical, ack.signatureB64)) {
            log.warn("Dropping ack with invalid signature for hub {}", publicId); return
        }
```

- [ ] **Step 3: Verify** `heartbeat` **+ freshness window**

At the top of `handleHeartbeat`, after parsing and the `schemaVersion` check:

```kotlin
        val keyDer = lookupHubKey(publicId) ?: run {
            log.warn("Heartbeat from hub {} with no stored public key; dropping", publicId); return
        }
        val issuedAtRaw = heartbeat.issuedAt ?: run {
            log.warn("Heartbeat from {} missing issued_at; dropping", publicId); return
        }
        if (!heartbeatFresh(issuedAtRaw, OffsetDateTime.now(ZoneOffset.UTC).toInstant(), HEARTBEAT_FRESHNESS_SECONDS)) {
            log.warn("Dropping stale/future/unparseable heartbeat for hub {}", publicId); return
        }
        val diag = heartbeat.diag
        val canonical = DeviceCanonical.heartbeat(
            hubPublicId = publicId,
            issuedAtRaw = issuedAtRaw,
            heapPct = diag?.freeHeapPct ?: 0,
            rssi = diag?.rssi?.toString() ?: "",
            uptimeMs = diag?.uptimeMs ?: 0L,
            dispD = (diag?.dispatchDaily ?: 0).toLong(),
            dispT = (diag?.dispatchTotal ?: 0).toLong(),
            ip = diag?.ip ?: ""
        )
        if (!DeviceSignatureVerifier.verify(keyDer, canonical, heartbeat.signatureB64)) {
            log.warn("Dropping heartbeat with invalid signature for hub {}", publicId); return
        }
```

Add `const val HEARTBEAT_FRESHNESS_SECONDS = 120L` (top-level) and the freshness helper as a **top-level** `internal` **function** in the file (pure → directly unit-testable):

```kotlin
internal fun heartbeatFresh(issuedAtRaw: String, now: java.time.Instant, windowSeconds: Long): Boolean {
    val issued = runCatching { OffsetDateTime.parse(issuedAtRaw).toInstant() }.getOrNull() ?: return false
    return Duration.between(issued, now).abs().seconds <= windowSeconds
}
```

Change `TransmitterHeartbeatEnvelope.issuedAt` to `String?` (raw wire string; add `@field:JsonProperty("issued_at")`). Keep the `OffsetDateTime`/`Duration`/`ZoneOffset` imports (still used).

- [ ] **Step 4: Unit-test the freshness helper; cover the rest via Tasks 1/11 + manual**

The `ack`/`heartbeat` **signature** glue is covered by `DeviceSignatureVerifierTest` (Task 1) + the `ack-v1`/`heartbeat-v1` golden bytes in `DeviceCanonicalTest` (Task 11). The one genuinely new, error-prone piece — the freshness window — gets a direct test:

```kotlin
package com.thomas.notiguide.domain.device.listener

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import java.time.OffsetDateTime

class HeartbeatFreshnessTest {
    private val now = OffsetDateTime.parse("2026-07-09T09:00:00Z").toInstant()

    @Test
    fun `accepts within window, rejects stale future and unparseable`() {
        assertThat(heartbeatFresh("2026-07-09T09:01:00Z", now, 120L)).isTrue()   // +60s
        assertThat(heartbeatFresh("2026-07-09T08:59:00Z", now, 120L)).isTrue()   // -60s
        assertThat(heartbeatFresh("2026-07-09T09:10:00Z", now, 120L)).isFalse()  // +600s
        assertThat(heartbeatFresh("2026-07-09T08:50:00Z", now, 120L)).isFalse()  // -600s
        assertThat(heartbeatFresh("not-a-timestamp", now, 120L)).isFalse()
    }
}
```

Run: `cd backend && ./gradlew test --tests "com.thomas.notiguide.domain.device.listener.HeartbeatFreshnessTest"`
Expected: PASS.

- [ ] **Step 5: Manual integration verification**

Let a hub run: backend refreshes liveness/diagnostics with no `invalid signature`/`stale` drops. Trigger a dispatch + a deact: reconciliation and lifecycle acks complete (signed acks accepted). Corrupt one canonical byte in a scratch firmware build → confirm the matching `Dropping … invalid signature` warning.

---



## Task 14: Firmware — sign `ack` (`dispatch.c`)

**Files:**

- Modify: `transmitter/main/dispatch/dispatch.c` (`publish_transmit_ack` L233, `publish_deact_ack` L270)

- [ ] **Step 1: Sign transmit acks**

In `publish_transmit_ack`, after computing `status`, build `ack-v1|{pid}|transmit|{dispatch_id}|{status}`, sign, and add `signature_b64` to both payload forms:

```c
    char canonical[192];
    char signature_b64[DEVICE_IDENTITY_SIGNATURE_B64_MAX_LEN];
    const char *wire_status = (strcmp(status, "applied") == 0 || strcmp(status, "unchanged") == 0)
                                  ? status : "rejected";
    (void)snprintf(canonical, sizeof(canonical), "ack-v1|%s|transmit|%s|%s",
                   cfg->public_id, cmd->dispatch_id, wire_status);
    if (device_identity_sign_text(canonical, signature_b64, sizeof(signature_b64)) != ESP_OK) {
        ESP_LOGE(TAG, "transmit ack signing failed");
        return;
    }
```

Then extend each `snprintf` payload to include `"signature_b64":"%s"` (append the field, pass `signature_b64`). Note the canonical binds the **wire status** (`applied`/`unchanged`/`rejected`), matching what the JSON `status` carries.

- [ ] **Step 2: Sign deact acks**

In `publish_deact_ack`, build `ack-v1|{pid}|deact|{command_id}|{status}`, sign, and append `"signature_b64":"%s"` to the payload:

```c
    char canonical[192];
    char signature_b64[DEVICE_IDENTITY_SIGNATURE_B64_MAX_LEN];
    (void)snprintf(canonical, sizeof(canonical), "ack-v1|%s|deact|%s|%s",
                   cfg->public_id, cmd->command_id, status);
    if (device_identity_sign_text(canonical, signature_b64, sizeof(signature_b64)) != ESP_OK) {
        ESP_LOGE(TAG, "deact ack signing failed");
        return;
    }
```

Widen the `payload` buffers if needed (signatures are ≤ `DEVICE_IDENTITY_SIGNATURE_B64_MAX_LEN` = 192; bump `char payload[256]` → `char payload[512]`).

- [ ] **Step 3: Manual verification**

Trigger a dispatch and a deact from the backend; confirm the backend logs no "invalid signature" drop and dispatch reconciliation / lifecycle ack completes. Cross-check the canonical against `DeviceCanonical.ack` golden test (Task 11).

---



## Task 15: Firmware — sign `roster/update` (`roster_sync.c`)

**Files:**

- Modify: `transmitter/main/pair/roster_sync.c` (`roster_sync_publish` L108-166)

- [ ] **Step 1: Build the canonical from roster entries (sorted by slot) and sign**

Before serializing/publishing, build `roster-update-v1|{pid}|{seq}|{slot}:{band}:{label}|…` with entries **sorted by slot ascending** (match `DeviceCanonical.rosterUpdate`). Use `band_label(entry->band)` for band and `entry->name` for label (empty string when absent). Then `device_identity_sign_text(...)` and add `signature_b64` to the root object:

```c
    // after building `root` with schema_version, seq, receivers, before printing:
    char canonical[512];
    int off = snprintf(canonical, sizeof(canonical), "roster-update-v1|%s|%d", cfg->public_id, (int)s_roster->seq);
    // iterate entries in ascending slot order:
    for (/* each entry sorted by slot */) {
        off += snprintf(canonical + off, sizeof(canonical) - (size_t)off, "|%d:%s:%s",
                        entry->slot, band_label(entry->band), entry->name);
        if (off >= (int)sizeof(canonical)) break;
    }
    char signature_b64[DEVICE_IDENTITY_SIGNATURE_B64_MAX_LEN];
    if (device_identity_sign_text(canonical, signature_b64, sizeof(signature_b64)) != ESP_OK) {
        ESP_LOGE(TAG, "roster update signing failed");
        cJSON_Delete(root);
        return ESP_FAIL;
    }
    (void)cJSON_AddStringToObject(root, "signature_b64", signature_b64);
```

> Implementer note: the roster entries may not be stored slot-sorted. Build a temporary array of indices sorted by `slot` and iterate that for the canonical, so the byte order matches the backend's `sortedBy { it.slot }`. The JSON `receivers` array order does not matter (the backend sorts before verifying).

- [ ] **Step 2: Manual verification**

Pair a receiver so the hub publishes a roster; confirm the backend applies it (no "invalid signature" drop) and the device appears. Empty-roster publish (`seq` with no receivers) must sign `roster-update-v1|{pid}|{seq}` (no trailing `|`).

---



## Task 16: Firmware — sign `heartbeat` (`heartbeat.c`)

**Files:**

- Modify: `transmitter/main/network/heartbeat.c` (L86-124)

- [ ] **Step 1: Always emit** `heap_pct` **as an integer (0 fallback)**

Replace the `has_heap_pct ? AddNumber : AddNull` branch (lines 102-104) so `heap_pct` is always a number (matching the backend `Int` and the canonical):

```c
            const int heap_pct_out = has_heap_pct ? heap_pct_clamped : 0;
            cJSON *heap_item = cJSON_AddNumberToObject(diag, "heap_pct", heap_pct_out);
```

- [ ] **Step 2: Build the canonical and sign**

After assembling the diag values (and before `cJSON_PrintUnformatted`), build `heartbeat-v1|{pid}|{issued_at}|{heap_pct}|{rssi}|{uptime_ms}|{disp_d}|{disp_t}|{ip}` using the **same** `issued_at` **string** already emitted, empty strings for absent `rssi`/`ip`, and add `signature_b64` to `root`:

```c
            char rssi_str[12] = "";
            if (has_rssi) { (void)snprintf(rssi_str, sizeof(rssi_str), "%d", rssi); }
            const char *ip_field = has_ip ? ip_str : "";

            char canonical[256];
            (void)snprintf(canonical, sizeof(canonical),
                           "heartbeat-v1|%s|%s|%d|%s|%lld|%u|%u|%s",
                           cfg->public_id, issued_at, heap_pct_out, rssi_str,
                           (long long)uptime_ms, (unsigned)dispatch_daily, (unsigned)dispatch_total, ip_field);
            char signature_b64[DEVICE_IDENTITY_SIGNATURE_B64_MAX_LEN];
            if (device_identity_sign_text(canonical, signature_b64, sizeof(signature_b64)) == ESP_OK) {
                (void)cJSON_AddStringToObject(root, "signature_b64", signature_b64);
            } else {
                ESP_LOGE(TAG, "heartbeat signing failed");
                cJSON_Delete(root);
                goto heartbeat_wait;
            }
```

> Contract: `rssi`/`ip` are the **empty string** in the canonical when absent (they are also omitted from the JSON `diag`, and the backend maps missing → `""`). `heap_pct` is always present as an integer on both sides. `issued_at` is the raw wire string on both sides.

- [ ] **Step 3: Manual verification**

Let a hub run; confirm the backend refreshes liveness/diagnostics (no "invalid signature" or "stale heartbeat" drops in the log). Cross-check the canonical against `DeviceCanonical.heartbeat` golden test (Task 11).

---



## Self-Review

**1. Spec coverage**


| Spec section                                                         | Task(s)                                       |
| -------------------------------------------------------------------- | --------------------------------------------- |
| §4.1 topic scheme (`bootstrap/{registration_nonce}`)                 | 4, 9                                          |
| §4.2 nonce ≥16B floor                                                | 5 (backend); firmware already 16B             |
| §4.3 `challenge_id` removal (all files)                              | 2, 3, 4, 5, 6, 7, 8, 9                        |
| §4.4 `activate-v1` with `registration_nonce`                         | 2, 7, 9                                       |
| §4.5 `nonce_in_use` pre-consume guard                                | 5                                             |
| §4.6 envelope body trim (pending/challenge/result/rejected/response) | 4, 7, 9                                       |
| §4.7 Web Serial unaffected                                           | 6 (approval keyed by deviceId; no web change) |
| §4.9 firmware auto-retry                                             | 10                                            |
| §5.1 canonicals (roster/ack/heartbeat, inline, null rules)           | 11                                            |
| §5.2 envelope signature fields, `schema_version` stays 1             | 12, 13, 14, 15, 16                            |
| §5.3 `DeviceSignatureVerifier` + per-hub key lookup + freshness      | 1, 12, 13                                     |
| §5.4 firmware signing at 3 sites                                     | 14, 15, 16                                    |
| §5.5 failure = drop + warn                                           | 12, 13                                        |


**2. Placeholder scan:** the only intentional fill-in is `VALID_P256_B64` in Task 5 (a real key the implementer generates — instructions provided). No "TBD"/"handle errors"/"similar to" placeholders.

**3. Type consistency:** `DeviceSignatureVerifier.verify(ByteArray, String, String): Boolean` used identically in Tasks 1/7/12/13. `RedisKeyManager.deviceActivation(String)` used in 3/5/6/7. `DeviceActivationByDeviceRecord(registrationNonce)` used in 3/5/6. `publishChallenge(kind, registrationNonce, nonce, issuedAt, expiresAt)` defined in 4, used in 6. `DeviceCanonical.rosterUpdate/ack/heartbeat` signatures defined in 11, used in 12/13 and mirrored in firmware 14/15/16. `RosterCanonicalReceiver(slot, band, label)` consistent 11↔12.

**Non-goals confirmed excluded:** `cluster/roster` signing, per-device broker ACLs, backward compatibility.