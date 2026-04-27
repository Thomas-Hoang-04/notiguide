# Receiver Integration Implementation Plan

Concrete backend and admin-web plan for integrating the ESP-01 and ESP32-C3 receiver families, plus passive PT2272 receivers, with the current NotiGuide stack.

The implementation creates a unified `device` domain because the later transmitter hub will share enrollment, approval, lifecycle, and admin UX. The receiver contract is the source of truth for v1; transmitter firmware, Web Serial, and transmitter MQTT topics are reserved here but implemented in [TRANSMITTER_ESP32C3_DESIGN_GUIDE.md](../planned/TRANSMITTER_ESP32C3_DESIGN_GUIDE.md) §H.

**Last updated:** 2026-04-27

---

## 1. Scope, Baseline, And Risk Map

### 1.1 Scope

In scope:

- Backend: Kotlin 2.3.20, Spring Boot 3.5.11, Java 21, WebFlux, coroutines, R2DBC, Redis, Paho MQTT v5.
- Admin web: Next.js 16.2.1, React 19.2.4, next-intl 4.8.3, Tailwind 4, shadcn/ui, Biome 2.2.0.
- Docs: this plan and `docs/CHANGELOGS.md`.

Out of scope:

- Firmware in `receiver-esp32/` and `receiver-esp8266/`.
- Customer-facing `client-web/`.
- Transmitter hub firmware, Web Serial bridge, transmitter MQTT topics, physical RF transmission.
- Analytics `DEVICE_TRIGGERED` emission.

### 1.2 Repo Invariants The Plan Must Preserve

- `SecurityConfig` permits only `/api/auth/**`, `/api/queue/public/**`, `/actuator/health`, `/actuator/info`. Device endpoints stay authenticated; no permit-all widening.
- `RateLimitFilter.resolveTier` currently matches strict only for `/api/queue/public/**`, auth for `/api/auth/**`, standard for everything else under `/api/**`. Method-specific strict routes require making this helper method-aware.
- `ReactiveRedisTemplate<String, String>` with `StringRedisSerializer`. Do not introduce JSON-wrapping serializers.
- Ticket hash key: `ticket:{storeId}:{ticketId}` via `RedisKeyManager.ticket(storeId, ticketId)`.
- `MqttConfig` is `@ConditionalOnProperty(prefix = "mqtt", name = ["broker"])`. New listeners and publishers must be `@ConditionalOnBean(MqttClientManager::class)`.
- `MqttPublisher` keeps the `${mqtt.topic-prefix}/store/{storeId}/queue` topic. Receiver topics use a fresh wrapper (`DeviceMqttPublisher`) that publishes to top-level `receiver/...` paths — separate from the prefixed queue topics.
- `web/src/components/ui` does **not** ship `table`, `tabs`, or `accordion`. Existing admin/store tables use native `<table>` markup inside `glass-card`.
- Sidebar labels are a TS union in `web/src/components/layout/sidebar.tsx`; adding `devices` requires updating both the union and `navigation.devices` in `web/src/messages/{en,vi}.json`.

### 1.3 Implementation Risk Map (GitNexus, upstream — verified 2026-04-25)

| Symbol | Risk | d=1 dependents | Action |
|---|---|---|---|
| `QueueService` | LOW | 4 | New methods only; no signature changes outside `issueTicket` (§D7a). |
| `RedisKeyManager` | MEDIUM | 9 | Add keys only; never rename existing ones. |
| `MqttClientManager` | LOW | 2 | Wrap; do not mutate the existing API. |
| `SecurityConfig` | LOW | 0 | No changes. |
| `RateLimitFilter` | LOW | 1 | Make tier resolution method-aware; only enrollment-token creation uses strict tier. |
| `TicketDto` | **HIGH** | 8 | `deviceId` / `deviceName` addition breaks 3 processes (`issueTicket`, `callNext`, `callSpecificTicket`); update every consumer in the same D7a slice. |

D0's first task re-runs `gitnexus_impact` per the repo rule; treat the numbers above as a snapshot.

---

## 2. Receiver Contract Summary

Authoritative sources: `docs/planned/RECEIVER_DESIGN_GUIDE.md`, `docs/planned/RECEIVER_ESP32C3_DESIGN_GUIDE.md`. If this summary disagrees with them, fix the guide first, then refresh this section.

### 2.1 MQTT Topics

| Topic | Direction | QoS | Retained | Use |
|---|---|---:|---|---|
| `receiver/bootstrap/register` | device → backend | 1 | no | Registration payload |
| `receiver/bootstrap/{challenge_id}` | both | 1 | no | `pending`, `challenge`, `rejected`, `result`, `response` envelopes |
| `receiver/device/{public_id}/cmd/rf_code` | backend → device | 1 | yes | Signed RF trigger code |
| `receiver/device/{public_id}/cmd/deact` | backend → device | 1 | yes | Signed suspend / resume / decommission |
| `receiver/device/{public_id}/ack` | device → backend | 1 | no | Single ack stream, `ack_for` discriminator |

Backend startup subscriptions: `receiver/bootstrap/register`, `receiver/bootstrap/+`, `receiver/device/+/ack`.

The bootstrap listener must drop backend-owned envelope types received on `receiver/bootstrap/+` because the ESP-01 path uses MQTT 3.1.1, which has no no-local flag.

Transmitter-family `transmitter/...` topics are owned by [TRANSMITTER_ESP32C3_DESIGN_GUIDE.md](../planned/TRANSMITTER_ESP32C3_DESIGN_GUIDE.md) §H.

### 2.2 On-Wire Names And Storage

The on-wire registration payload only ever appears for **MCU** receivers (ESP-01 / ESP32-C3). Passive PT2272-class receivers have no firmware and no return channel — the admin enrolls them manually (§7, §D2.1).

MCU registration payload uses:

- `hardware_model`: `ESP-01` or `ESP32-C3`
- `receiver_type`: `RECEIVER_433M` or `RECEIVER_2_4G`
- `firmware_version`: free-form string, persisted to `device.firmware_version`

Backend storage:

- `device.hardware_model`: identifier-safe enum (`ESP_01`, `ESP32_C3`, `PT2272`) mapped from the payload value
- `device.kind`: mapped from `receiver_type`, or `RECEIVER_433M_PASSIVE` for passive registrations

Legal pairs:

| Wire `hardware_model` | Stored enum | legal `kind` | Registered via |
|---|---|---|---|
| `ESP-01` | `ESP_01` | `RECEIVER_433M` | MQTT bootstrap (§D2) |
| `ESP32-C3` | `ESP32_C3` | `RECEIVER_433M`, `RECEIVER_2_4G` | MQTT bootstrap (§D2) |
| `PT2272` | `PT2272` | `RECEIVER_433M_PASSIVE` | Admin manual register (§D2.1) |

`PT2272` covers any pin-compatible non-MCU 433 MHz decoder family (HS1527, HX2272, etc.). A wholly different non-MCU chip family would get its own enum value and its own row here.

Do not persist dash-containing hardware labels directly: Kotlin enum constants and the repo's existing R2DBC `EnumCodec` pattern expect DB enum labels to match Kotlin enum names. DTO/request parsing owns the wire-label mapping.

### 2.3 Canonical Strings

UTF-8, no trailing newline, exact field order:

```text
activate-v1|<challenge_id>|<nonce>|<issued_at>|<expires_at>
rf-code-v1|<public_id>|<code_version>|<rf_code_hex>|<rf_code_bits>|<issued_at>
deact-v1|<public_id>|<command_id>|<action>|<issued_at>
```

Rules: `issued_at` / `expires_at` are ISO-8601 UTC ending in `Z`; `rf_code_hex` is uppercase hex; integers are base-10 without prefixes. Builders live in `core/device/DeviceCanonical.kt` and stay pure. The transmitter-family `transmit-v1` builder is defined by [TRANSMITTER_ESP32C3_DESIGN_GUIDE.md](../planned/TRANSMITTER_ESP32C3_DESIGN_GUIDE.md) §H.3.

### 2.4 RF-Code Validation

Validate before signing or persisting. The bit-width rules reflect what each receiver family physically uses on the air:

| `kind` | `rf_code_bits` | Constraints | Source of code | What the bytes mean on air |
|---|---|---|---|---|
| `RECEIVER_433M`         | 1..32 | `hex_len == 2 * ceil(bits/8)` | Backend `SecureRandom` (§D5.1) | RC-Switch payload value matched at the receiver MCU |
| `RECEIVER_433M_PASSIVE` | 1..24 | `hex_len == 2 * ceil(bits/8)`, **hardware cap** | Admin enters preset value (§D5.2) | PT2272 wire value matched in silicon |
| `RECEIVER_2_4G`         | **exactly 40** | `hex_len == 10` | Backend `SecureRandom` (§D5.1) | nRF24 5-byte `RX_ADDR_P1` — silicon-level identity, **not** an application-layer match value |

Defaults for backend-generated codes (auto-issued at activation) live in §6: `default-bits-433m = 32`, `default-bits-24g = 40`. PT2272 has no default — admins always supply the preset because it's hardware-fixed at the chip.

**PT2272 24-bit hardware guard.** The 24-bit cap on `RECEIVER_433M_PASSIVE` is a property of the chip itself: PT2272-class decoders encode 12 tri-state symbols, each represented by 2 bits, for a 24-bit total. The cap therefore must be enforced as an authoritative server-side checkpoint, not just a UI nicety: a request with `bits > 24` on a passive registration is rejected before any DB write with a dedicated `pt2272_hardware_cap_exceeded` error so the UI can show a hardware-specific message rather than the generic width error.

**RECEIVER_2_4G fixed at 40 bits.** nRF24L01 / nRF24L01+ silicon supports 3-, 4-, or 5-byte addresses via `SETUP_AW`, but 5 bytes is the chip default and the datasheet flags shorter widths as more noise-prone (PS v1.0 §7.3.2 and PS v2.0 §7.3.2). The plan locks `bits = 40` so the validator, the forbidden-set, and the `device_rf_code` schema CHECK can all stay simple. A request with any other `bits` for `RECEIVER_2_4G` is rejected before any DB write with `width_out_of_range`. The 40-bit value is written verbatim to the receiver's `RX_ADDR_P1` (LSByte first on the wire) at activation time and re-applied to the silicon on every rotation — see §D5.1.

**Toggle payload, not an address.** The transmitter hub sends a small fixed payload — `TOGGLE_MAGIC = 0xAA 0x55` — to the address. The receiver's `rf_trigger_on_packet` matches on the magic bytes (defense-in-depth against improbable CRC-pass-by-noise) before toggling the vibrator. The magic is firmware-defined and never appears on the wire of any backend/admin surface.

---

## 3. Backend Data Model

### 3.1 Final Schema Shape

Update `backend/src/main/resources/db/schema.sql` for fresh installs. Add a one-shot operator-run migration at:

```text
backend/src/main/resources/db/migrations/2026-04-25-device-domain.sql
```

The repo has no Flyway/Liquibase, and `bootJar` excludes `db/**`, so the migration ships separately and is not packaged in the prod JAR.

```sql
CREATE TYPE device_kind AS ENUM (
  'RECEIVER_433M',          -- MCU 433 MHz (ESP-01 / ESP32-C3); 1..32 bits
  'RECEIVER_433M_PASSIVE',  -- PT2272-class hardware decoder; 1..24 bits, admin-registered
  'RECEIVER_2_4G',          -- MCU 2.4 GHz (ESP32-C3); rf_code is the 5-byte nRF24 RX_ADDR_P1 (bits = 40)
  'TRANSMITTER_HUB'         -- reserved for transmitter family
);

CREATE TYPE device_hardware_model AS ENUM (
  'ESP_01',
  'ESP32_C3',
  'PT2272'                  -- pin-compatible non-MCU 433 MHz decoder family
);

CREATE TYPE device_status AS ENUM (
  'PENDING',
  'PENDING_RF_CODE',
  'ACTIVE',
  'SUSPENDED',
  'DECOMMISSIONED',
  'REJECTED'
);

CREATE TYPE device_rf_ack_status AS ENUM (
  'PENDING',
  'APPLIED',
  'UNCHANGED',
  'REJECTED'
);

CREATE TABLE device (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  public_id             VARCHAR(32),
  public_key_der        BYTEA,
  hardware_model        device_hardware_model NOT NULL,
  kind                  device_kind NOT NULL,
  status                device_status NOT NULL DEFAULT 'PENDING',
  assigned_name         VARCHAR(100),
  store_id              UUID REFERENCES store(id) ON DELETE SET NULL,
  firmware_version      VARCHAR(32),
  last_seen_at          TIMESTAMPTZ,
  activated_at          TIMESTAMPTZ,
  created_at            TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at            TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE UNIQUE INDEX idx_device_public_id
  ON device(public_id) WHERE public_id IS NOT NULL;

CREATE UNIQUE INDEX idx_device_public_key_der
  ON device(public_key_der) WHERE public_key_der IS NOT NULL;

CREATE INDEX idx_device_store ON device(store_id);
CREATE INDEX idx_device_status ON device(status);
CREATE INDEX idx_device_kind ON device(kind);

-- Plaintext payload encoding (firmware/backend MUST agree). Only
-- ciphertext is stored; plaintext meaning by kind:
--   RECEIVER_433M / RECEIVER_433M_PASSIVE: big-endian unsigned int,
--     left-padded to byte_len = ceil(bits/8); only the bottom `bits`
--     LSBs are significant.
--   RECEIVER_2_4G: literal nRF24L01 on-air payload of the receiver
--     pipe (no header, no CRC, no address).
-- `bits` is the significant width; `byte_len` is plaintext length.
-- Plaintext encoding by kind:
--   RECEIVER_433M / RECEIVER_433M_PASSIVE: big-endian unsigned int,
--     left-padded to byte_len = ceil(bits/8); only the bottom `bits`
--     LSBs are significant. Width: bits ∈ [1, 32] (433M) or [1, 24]
--     (433M_PASSIVE).
--   RECEIVER_2_4G: 5-byte nRF24 address (SETUP_AW = 11). bits = 40,
--     byte_len = 5 enforced by the kind-aware CHECK below. Bytes go on
--     the wire LSByte first (PS v1.0 §8.3.1).
CREATE TABLE device_rf_code (
  device_id      UUID PRIMARY KEY REFERENCES device(id) ON DELETE CASCADE,
  payload        BYTEA    NOT NULL,                       -- pgp_sym_encrypt_bytea(plaintext, device.rf-code.encryption-key)
  bits           SMALLINT NOT NULL,                       -- see kind-aware constraint below
  byte_len       SMALLINT NOT NULL,                       -- 1..32
  version        INT      NOT NULL,
  issued_at      TIMESTAMPTZ NOT NULL,
  ack            device_rf_ack_status NOT NULL DEFAULT 'PENDING',
  ack_at         TIMESTAMPTZ,

  CONSTRAINT chk_rf_bits      CHECK (bits     BETWEEN 1 AND 256),
  CONSTRAINT chk_rf_byte_len  CHECK (byte_len BETWEEN 1 AND 32),
  CONSTRAINT chk_rf_byte_math CHECK (byte_len = (bits + 7) / 8)
);

-- Width constraint scoped by device kind. RECEIVER_2_4G stores a 5-byte
-- nRF24 address (bits = 40) and nothing else; 433M variants keep their
-- existing 1..32 / 1..24 ranges. The trigger fires on insert and update;
-- the device_rf_code row is created in the same transaction as the
-- device row (D2.1) or after activation (D5.1), so the kind lookup is
-- always satisfiable.
CREATE OR REPLACE FUNCTION device_rf_code_kind_check() RETURNS TRIGGER AS $$
DECLARE
  k device_kind;
BEGIN
  SELECT kind INTO k FROM device WHERE id = NEW.device_id;
  IF k = 'RECEIVER_2_4G' AND (NEW.bits <> 40 OR NEW.byte_len <> 5) THEN
    RAISE EXCEPTION 'RECEIVER_2_4G requires bits = 40 and byte_len = 5';
  ELSIF k = 'RECEIVER_433M' AND NEW.bits NOT BETWEEN 1 AND 32 THEN
    RAISE EXCEPTION 'RECEIVER_433M requires bits BETWEEN 1 AND 32';
  ELSIF k = 'RECEIVER_433M_PASSIVE' AND NEW.bits NOT BETWEEN 1 AND 24 THEN
    RAISE EXCEPTION 'RECEIVER_433M_PASSIVE requires bits BETWEEN 1 AND 24';
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_device_rf_code_kind_check
  BEFORE INSERT OR UPDATE ON device_rf_code
  FOR EACH ROW EXECUTE FUNCTION device_rf_code_kind_check();
```

Rules:

- Root `device` table holds durable identity, assignment, status, timestamps. Nothing transient.
- The `(hardware_model, kind)` pairing is enforced in the application validator, not the DB — the DB enum constrains hardware families; per-pair legality varies with deployment intent.
- Schema-level `chk_rf_*` constraints enforce the universal envelope (1..256 bits, byte math). Per-kind bit-width rules from §2.4 live in `RfCodeValidator` because they need `device.kind`.
- Both 433 MHz and 2.4 GHz codes share the same row shape; `bits` / `byte_len` differ. Splitting into two tables would duplicate every read path and join for no behavioural gain.
- Update `R2DBCConfig` with `EnumCodec` entries and `EnumWriteSupport` converters for `device_kind`, `device_hardware_model`, `device_status`, and `device_rf_ack_status`, matching the existing `admin_role` pattern.
- Update `analytics_event.device_id` in `schema.sql` to reference `device(id) ON DELETE SET NULL`; it currently references the legacy `notifier_device(id)`.

### 3.2 Migration Rules

The deployed `notifier_device` table is an old FCM-token-shaped placeholder. Kotlin code does not use it today; `analytics_event.device_id` is its only known FK.

Migration steps (one BEGIN / COMMIT, idempotent — every step must be safe to rerun without data loss after passive devices exist):

1. Wrap in `BEGIN` / `COMMIT`.
2. Create enum types with duplicate-object guards (`DO $$ BEGIN ... EXCEPTION WHEN duplicate_object ...`).
3. Rename `notifier_device` → `device` only if `device` does not already exist.
4. Drop legacy FCM columns with `ALTER TABLE device DROP COLUMN IF EXISTS device_token, DROP COLUMN IF EXISTS battery_level, DROP COLUMN IF EXISTS last_ping, DROP COLUMN IF EXISTS is_active;` so a rerun after the columns are gone is a no-op.
5. Rename `name` → `assigned_name` only if `name` exists and `assigned_name` does not (guard with an `information_schema.columns` `DO` block).
6. Add new columns with `ADD COLUMN IF NOT EXISTS` (nullable / defaulted): `public_id`, `public_key_der`, `hardware_model`, `kind`, `status`, `firmware_version`, `last_seen_at`, `activated_at`.
7. Drop legacy rows that cannot be represented as receivers. Use the **new** required columns as the legacy discriminator — passive PT2272 rows legitimately have `public_key_der IS NULL`, so that column must not appear in this filter:
   ```sql
   DELETE FROM device
   WHERE kind IS NULL OR hardware_model IS NULL;
   ```
   On rerun, every surviving row already has both columns populated, so this DELETE matches nothing.
8. Replace the old `store_id` FK (`ON DELETE CASCADE`) with `ON DELETE SET NULL` — the old action would delete device audit history when a store is removed. Drop the old constraint with `IF EXISTS` before recreating.
9. Recreate `analytics_event.device_id` so it references `device(id) ON DELETE SET NULL` after the rename. Drop the old constraint with `IF EXISTS` first; this step runs unconditionally so a rerun against a `device`-renamed schema still repairs the FK.
10. Set `hardware_model` and `kind` to `NOT NULL`. Wrap in a `DO` block that checks `is_nullable` in `information_schema.columns` so the rerun is a no-op.
11. Create `device_rf_code` with `CREATE TABLE IF NOT EXISTS`.
12. Recreate indexes with `IF NOT EXISTS`.

Operator command:

```bash
psql "$DATABASE_URL" -f backend/src/main/resources/db/migrations/2026-04-25-device-domain.sql
```

Rollback is snapshot-based: the migration intentionally drops FCM-shaped columns that have no current consumers.

---

## 4. Redis Keyspace

Add only new keys to `RedisKeyManager`; do not rename existing ones.

```kotlin
fun enrollmentToken(sha256Hex: String) = "enroll:$sha256Hex"
fun enrollmentTokenPattern() = "enroll:*"
fun deviceActivation(challengeId: UUID) = "device:activation:$challengeId"
fun deviceActivationByDevice(deviceId: UUID) = "device:activation-by-device:$deviceId"
fun deviceLifecycleCommand(deviceId: UUID) = "device:lifecycle:$deviceId"
fun deviceBusy(deviceId: UUID) = "device:busy:$deviceId"
```

Values are JSON strings (Redis is `ReactiveRedisTemplate<String, String>`).

| Key | Value | TTL |
|---|---|---|
| `enroll:{sha256}` | `{storeId, issuedByAdminId, issuedAt, expiresAt}` | `device.enrollment.token-ttl-seconds`, default 3600 |
| `device:activation:{challengeId}` | `{deviceId, publicKeyFingerprint, registrationNonce, nonce, issuedAt, expiresAt, status}` | 15 minutes |
| `device:activation-by-device:{deviceId}` | `{challengeId}` | matches activation key |
| `device:lifecycle:{deviceId}` | `{commandId, action, issuedAt, ackStatus}` | 30 minutes |
| `device:busy:{deviceId}` | `{storeId, ticketId, boundAt}` | mirrors ticket lifecycle (§D7a) |

Token rules:

- Generate 128 bits with `SecureRandom`, encode as base64url without padding.
- Hash the exact trimmed token using SHA-256 hex (no case folding).
- Issue with `SET ... EX ... NX`.
- Consume with `GETDEL`.
- Plaintext tokens never enter Postgres; admin web shows them once.

Ticket binding rule: add `device_id` to the existing ticket hash at `RedisKeyManager.ticket(storeId, ticketId)` (§D7a). Do not introduce a parallel ticket key format.

---

## 5. Backend Package Layout

```text
backend/src/main/kotlin/com/thomas/notiguide/
├── core/device/
│   ├── DeviceCanonical.kt
│   ├── DeviceCommandSigner.kt
│   ├── DeviceCommandSigningConfig.kt
│   ├── DeviceCommandSigningProperties.kt
│   ├── DeviceMqttPublisher.kt
│   └── DevicePublicIdMinter.kt
└── domain/device/
    ├── controller/
    │   ├── DeviceAdminController.kt          # /api/devices/** except enrollment-tokens
    │   └── EnrollmentTokenController.kt      # /api/devices/enrollment-tokens/**
    ├── dto/
    ├── entity/
    │   ├── Device.kt
    │   └── DeviceRfCode.kt
    ├── listener/
    │   ├── DeviceBootstrapListener.kt        # receiver/bootstrap/{register,+}
    │   └── DeviceOperationalListener.kt      # receiver/device/+/ack
    ├── repository/
    │   ├── DeviceRepository.kt
    │   └── DeviceRfCodeRepository.kt
    ├── request/
    ├── service/
    │   ├── DeviceActivationService.kt
    │   ├── DeviceApprovalService.kt
    │   ├── DeviceDispatchService.kt          # producer for in-process dispatch events (§D7a)
    │   ├── DeviceLifecycleService.kt
    │   ├── DeviceRegistrationService.kt      # MCU bootstrap intake (transmitter plan generalises later)
    │   ├── PassiveDeviceRegistrationService.kt   # PT2272 manual register (§D2.1)
    │   ├── EnrollmentTokenService.kt
    │   ├── RfCodeForbiddenSet.kt             # §D5.0
    │   └── RfCodeService.kt
    └── types/
        ├── DeviceHardwareModel.kt
        ├── DeviceKind.kt
        ├── DeviceLifecycleAckStatus.kt
        ├── DeviceRfAckStatus.kt
        └── DeviceStatus.kt
```

`DeviceHardwareModel` is shared with the transmitter hub (which reuses `ESP32-C3`); no extra hardware-model enum value is needed for that family.
Its enum constants are `ESP_01`, `ESP32_C3`, and `PT2272`; request/response DTOs map these to the wire labels `ESP-01`, `ESP32-C3`, and `PT2272`.

---

## 6. Backend Configuration

Add to `application.yaml`. Five knobs, each with one job:

```yaml
device:
  command-signing:
    # PEM-encoded EC P-256 private key for SHA256withECDSA. Spring
    # resource location: `classpath:...` for dev or `file:/abs/path.pem`
    # for prod-mounted secrets. Prefix with `base64:` to inline base64-
    # encoded PEM contents (useful for env-only platforms).
    pk: ${DEVICE_CMD_SIGNING_PK:}
  rf-code:
    # Symmetric passphrase for `pgp_sym_encrypt_bytea` over plaintext RF
    # payload bytes. Required whenever any receiver is active.
    encryption-key: ${DEVICE_RF_CODE_ENCRYPTION_KEY:}
    # Auto-generated initial code widths.
    # 433M defaults at its 32-bit cap (max entropy on the band).
    # 2.4G is fixed at 40 bits — the nRF24 RX_ADDR_P1 width. The rf_code
    # for a 2.4G receiver IS its silicon-level address; 5 bytes is the
    # chip default (SETUP_AW = 11) and shorter widths are noise-prone
    # per the datasheet (§2.4). The width is intentionally not
    # config-tunable.
    default-bits-433m: 32
    default-bits-24g: 40
  enrollment:
    token-ttl-seconds: ${DEVICE_ENROLLMENT_TTL_SECONDS:3600}
```

Dev profile may use `pk: classpath:device/cmd_signing_priv.pem`.

Implementation rules:

- Bind `DeviceCommandSigningProperties` and RF-code properties unconditionally, but do not fail app startup just because `pk` or `encryption-key` is blank. APIs that need signing, MQTT publishing, or RF-code encryption return `503 Service Unavailable` with a structured error when the required dependency/config is absent; read-only list/detail endpoints still work.
- Create `DeviceCommandSigner` only when `pk` is non-blank, then fail fast if that provided key cannot be loaded as an EC P-256 private key. Do not stack multi-name `@ConditionalOnProperty` on the config — Spring Boot evaluates multiple `name` entries with AND semantics, not OR.
- Resolve `pk` via `ResourceLoader.getResource(pk).inputStream` then `PemContent.load(inputStream).getPrivateKey()`. Use explicit resource prefixes: `classpath:` for dev resources and `file:` for mounted files. Bare absolute paths are not part of the contract. If the value starts with `base64:`, decode the body to text and use `PemContent.of(text).getPrivateKey()`.
- Add `**/resources/device/**` to `backend/.gitignore` and `"device/**"` to the existing `bootJar.exclude(...)` list so a dev key never ships in the prod JAR.
- `DeviceMqttPublisher` and MQTT listeners are `@ConditionalOnBean(MqttClientManager::class)`. If MQTT or signing is missing, admin APIs that need them return `503 Service Unavailable` with a clear error code; token listing and device listing still work.

RF-code encryption boundary:

- `DeviceRfCodeRepository` encrypts binary payloads via `pgp_sym_encrypt_bytea(plaintext, <device.rf-code.encryption-key>)` on write and decrypts via `pgp_sym_decrypt_bytea(...)` only for operational publish/dispatch paths.
- Plaintext may exist only in short-lived service memory while generating/signing/publishing an RF code or while dispatching through a transmitter hub. It never appears in admin DTOs, logs, MQTT acks, or SSE events.
- Admin DTOs may surface `bits`, `byte_len`, `version`, `ack`, `issued_at` — never the ciphertext or plaintext.

---

## 7. Backend API

All routes require an authenticated admin. SUPER_ADMIN can act across stores; ADMIN is store-scoped via `StoreAccessUtil` and `principal.storeId`.

`DeviceAdminController` (`/api/devices/**` except enrollment tokens):

```text
GET    /api/devices
GET    /api/devices/{id}
POST   /api/devices/passive                # MCU-less PT2272 manual register (§D2.1)
POST   /api/devices/{id}/approve
POST   /api/devices/{id}/reject
POST   /api/devices/{id}/rf-code            # rotation only; 400 for RECEIVER_433M_PASSIVE (their initial code is set during D2.1 manual register and the chip is hardware-fixed)
POST   /api/devices/{id}/lifecycle          # body: {action: "suspend"|"resume"|"decommission"}
POST   /api/devices/{id}/reprovision        # 404 for RECEIVER_433M_PASSIVE
```

`EnrollmentTokenController` (`/api/devices/enrollment-tokens/**`):

```text
POST   /api/devices/enrollment-tokens
GET    /api/devices/enrollment-tokens
DELETE /api/devices/enrollment-tokens/{sha256}
```

`POST /api/devices/passive` body:

```json
{
  "hardwareModel": "PT2272",
  "kind": "RECEIVER_433M_PASSIVE",
  "assignedName": "Front gate fob #3",
  "storeId": "<UUID>",
  "rfCodeHex": "0A1B2C",
  "rfCodeBits": 24
}
```

Server flow (§D2.1 spells out the full sequence): validate the `(hardwareModel, kind)` pair, validate width per §2.4, parse hex to plaintext, run §D5.0 forbidden-set + uniqueness checks, transactionally INSERT `device` (`status = ACTIVE`, synthetic `pas-XXXXX` `public_id`) and `device_rf_code` (`version = 1`, `ack = APPLIED`). No MQTT, no signing, no enrollment token, no approval — admin authority *is* the approval. Returns the new `DeviceDto`.

Queue dispatch lives under existing queue admin routes (handled by `DeviceDispatchService` but exposed via `QueueAdminController`):

```text
POST /api/queue/admin/{storeId}/device-tickets
GET  /api/queue/admin/{storeId}/available-devices
```

Update `API_ROUTES` in admin web accordingly.

Rate limiting: change `RateLimitFilter.resolveTier` to accept both `method` and `path`, then route only `POST /api/devices/enrollment-tokens` to the strict tier. Token listing/revoke and all other device endpoints inherit the standard `/api/**` tier.

---

## 8. Backend Services

### Phase D0 — Foundation

1. Run GitNexus impact for every touched symbol before editing it.
2. Update schema and add the migration.
3. Add Redis keys (no renames).
4. Add device config properties, signing config, canonical helpers (`activate-v1`, `rf-code-v1`, `deact-v1`), `DevicePublicIdMinter`, and `DeviceMqttPublisher`.

`DevicePublicIdMinter` rules:

- MCU receivers: `rcv-XXXXX` (Crockford base32).
- Passive PT2272: `pas-XXXXX` (same alphabet, distinct prefix so the admin UI can tag "passive" from the ID alone).
- Reserve `hub-XXXXX` for the transmitter hub.
- Collision check is global over `device.public_id`, regardless of kind.

### Phase D1 — Enrollment Tokens

`EnrollmentTokenService`:

- Issue with `SET enroll:{hash} ... EX ... NX`.
- Return `{token, tokenHash, expiresAt}` once.
- List metadata only: `{tokenHash, storeId, issuedAt, expiresAt}`.
- Revoke by hash.

Authority: SUPER_ADMIN may issue for any store or none; ADMIN may only issue for `principal.storeId` and is rejected if absent.

### Phase D2 — Bootstrap Registration (MCU)

`DeviceBootstrapListener` routes:

- `receiver/bootstrap/register` → `DeviceRegistrationService.onRegister`.
- `receiver/bootstrap/{cid}` with `type == "response"` → `DeviceActivationService.onResponse`.
- Backend-owned envelope types arriving on `receiver/bootstrap/+` are ignored (self-echo guard).

`DeviceRegistrationService.onRegister`:

1. Validate `schema_version`, required fields, public-key DER/base64 shape, wire `hardware_model`, `receiver_type`, and `firmware_version`. Reject any payload claiming `RECEIVER_433M_PASSIVE` — passive devices never come in over MQTT.
2. Map wire `hardware_model` → `DeviceHardwareModel` and `receiver_type` → `DeviceKind`.
3. Consume the enrollment token via `GETDEL enroll:{hash}`.
4. If the token is missing, publish `rejected` with `invalid_token`.
5. Upsert by `public_key_der`. Persist `firmware_version` from the payload.
6. Set status to `PENDING`; on reprovision, clear operational fields and the old `device_rf_code` row.
7. Write `device:activation:{challengeId}` and `device:activation-by-device:{deviceId}`.
8. Publish `pending` on `receiver/bootstrap/{challengeId}`.

### Phase D2.1 — Manual Registration (Passive PT2272)

`PassiveDeviceRegistrationService.register(req, adminId)` for `POST /api/devices/passive`:

1. Authority check: SUPER_ADMIN may target any store; ADMIN must target `principal.storeId`.
2. Validate the `(hardwareModel, kind)` pair (§2.2). Reject anything that isn't `(PT2272, RECEIVER_433M_PASSIVE)`.
3. **PT2272 hardware-cap guard**: reject `bits > 24` with `400 Bad Request {"error":"pt2272_hardware_cap_exceeded","limit":24}` *before* any other validation — the cap is a hardware-physical property of the chip, not just a configuration. Then run the residual width checks: `bits >= 1` and `hex_len == 2 * ceil(bits/8)`. Both surface as `400 Bad Request {"error":"width_out_of_range"}`.
4. Parse `rfCodeHex` to plaintext bytes; run §D5.0 forbidden-set + uniqueness guards. Surface uniqueness collisions as `409 Conflict` with the colliding device's `public_id` (helps the admin diagnose without exposing the colliding code).
5. Mint a `pas-XXXXX` `public_id`.
6. Transactionally INSERT `device` (`status = ACTIVE`, `activated_at = now()`) and `device_rf_code` (`version = 1`, `ack = APPLIED`, `ack_at = now()` — no future ack is coming).
7. Return the new `DeviceDto`.

No MQTT, no enrollment token, no approval, no signing, no challenge. Skips D2 / D3 / D4 entirely.

### Phase D3 — Admin Approval

Approve:

- Validate admin store access.
- Set `store_id` and `assigned_name`.
- Load the activation challenge via `device:activation-by-device:{deviceId}`. Do not scan `device:activation:*`.
- Generate a ≥128-bit challenge nonce.
- Set challenge status `ISSUED`, `issuedAt`, `expiresAt`.
- Publish `challenge`.

Reject:

- Publish `rejected`, delete both activation keys, set `device.status = REJECTED`.
- Do not refund the consumed enrollment token.

### Phase D4 — Activation

`DeviceActivationService.onResponse`:

1. Load challenge and device. Capture `priorPublicId = device.public_id` (may be null on first activation; non-null on reprovision) — step 6 overwrites the column, so the value must be held in a local before the transaction.
2. Reject if missing, expired, or not `ISSUED`.
3. Rebuild `activate-v1|...` byte-for-byte.
4. Verify ECDSA P-256 signature against `device.public_key_der`.
5. Mint a fresh `public_id`.
6. Transactionally set `public_id` (the new one), `status = PENDING_RF_CODE`, `activated_at`.
7. Delete activation keys.
8. Publish `result` with the new `public_id` and assigned name.
9. If `priorPublicId != null`, clear retained `cmd/rf_code` and `cmd/deact` on `receiver/device/{priorPublicId}/...` so a stale broadcast can't replay against the new identity.
10. Issue the initial RF code via `RfCodeService.autoIssue(deviceId)`. When the receiver acks the current version as `APPLIED` or `UNCHANGED`, promote status to `ACTIVE`.

If the initial publish cannot happen because MQTT or signing is unavailable, leave status as `PENDING_RF_CODE` and expose a retry action in the RF-code panel.

### Phase D5 — RF Code

`RfCodeValidator` and the `device_rf_code` write path are shared across kinds; what differs is who picks the plaintext and whether MQTT is involved.

#### D5.0 — Forbidden-set + uniqueness guards (all kinds)

Run on every plaintext value, backend-generated or admin-supplied:

- **Forbidden, 433M family** (`RECEIVER_433M`, `RECEIVER_433M_PASSIVE`): reject low-entropy values where every nibble of the bottom `bits` is identical, including `0x00…0` and `0xFF…F`. Add firmware-documented sentinel patterns only when the firmware decoder actually reserves them.
- **Forbidden, 2.4G** (the rf_code is the 5-byte `RX_ADDR_P1`, so these guards protect against nRF24-specific address pathologies the datasheets call out, not generic payload entropy):
  1. Reject `0x0000000000` and `0xFFFFFFFFFF` — addresses with zero level transitions.
  2. Reject continuous-toggle patterns `0x5555555555` and `0xAAAAAAAAAA` — they look like the preamble and "raise the Packet-Error-Rate" per PS v1.0 §7.3.2 / PS v2.0 §7.3.2.
  3. Reject any address with **at most one level shift across the 5 bytes** (e.g., `0x00FFFFFFFF`, `0x000000FFFF`, `0xFFFF000000`). The datasheet warns these "can often be detected in noise and can give a false detection."
  4. Reject any address whose first byte (LSByte on the wire — the byte that gets matched against the preamble first) has fewer than two `0→1`/`1→0` transitions, on the same noise-detection rationale.
- **Uniqueness, kind-aware:**
  - **433M band** (groups `RECEIVER_433M` with `RECEIVER_433M_PASSIVE`): scoped to **same store**, among `device.status ∈ {PENDING_RF_CODE, ACTIVE, SUSPENDED}`. One hub broadcast hits every co-store 433M receiver, but stores are RF-isolated by deployment — and the PT2272 hardware-fixed code space is small enough that cross-store reuse is the realistic norm, not the exception.
  - **2.4G band**: scoped **globally** (across all stores), same status set. The address space (~10¹² values for 5 bytes minus the forbidden set above) is enormous, so global uniqueness costs nothing to enforce, and it eliminates the entire class of "two adjacent stores collided over the air" mystery bugs at mint time. The query is `WHERE kind = 'RECEIVER_2_4G' AND status IN (...)` — no `store_id` clause.
- On collision: backend-generated codes (D5.1) regenerate up to 8 times before failing with `RF code space exhausted` (only realistic at extreme density on 433M; effectively impossible on 2.4G). Admin-supplied codes (D5.2) fail immediately with `409 Conflict` and the colliding device's `public_id`.

#### D5.1 — Backend auto-issue and rotation (MCU receivers)

Applies to `RECEIVER_433M` and `RECEIVER_2_4G`. `RfCodeService.autoIssue(deviceId)` and `RfCodeService.rotate(deviceId, request, adminId)`:

- Pick `bits`:
  - `RECEIVER_433M`: rotation requests carry an explicit `bits` (admin choice within §2.4); auto-issue uses `default-bits-433m`.
  - `RECEIVER_2_4G`: width is **fixed at 40** (§2.4). Rotation requests for 2.4G must omit `bits` or pass `40`; any other value is a `400 Bad Request {"error":"width_out_of_range"}`. Auto-issue ignores `default-bits-24g` if it is ever set to anything other than 40.
- Generate plaintext via `SecureRandom`, sized to `byte_len = ceil(bits/8)`:
  - `RECEIVER_433M`: random integer; high `(8·byte_len − bits)` bits masked to 0 so the on-wire value matches `bits`.
  - `RECEIVER_2_4G`: 5 random bytes verbatim. The first generated byte goes on the wire as the LSByte (datasheet §8.3.1) and is therefore the byte the receiver's preamble matcher sees first; if it fails the §D5.0 first-byte-transition guard, regenerate.
- Run `RfCodeValidator.validate(device.kind, bits, plaintext)` (§2.4) and §D5.0 guards.
- `INSERT` on first issue, `UPDATE` with `version = version + 1` on rotation. Same `(payload, bits, byte_len)` cluster in one transaction.
- Encrypt plaintext via `pgp_sym_encrypt_bytea` inside `DeviceRfCodeRepository`. Plaintext is not returned to admin surfaces; it is available only to the operational publish/dispatch path.
- Sign `rf-code-v1|public_id|version|HEX(plaintext)|bits|issued_at` and publish a retained `cmd/rf_code`.
- Set `ack = PENDING`, `ack_at = NULL` on every write.
- Ack handling: normalise lower-case MQTT statuses to uppercase `DeviceRfAckStatus`. Update `device_rf_code` only when `ack.code_version == device_rf_code.version`. Update `device.last_seen_at`. If current status is `PENDING_RF_CODE` and the ack is `APPLIED` or `UNCHANGED`, promote to `ACTIVE`. Out-of-order old-version acks are ignored without user-facing noise.

**Receiver-side rotation, 2.4G specific.** On `RECEIVER_2_4G`, applying a new rf_code is more than an in-RAM matcher update — the receiver must rewrite `RX_ADDR_P1` in silicon. The live `receiver-esp32` path delegates this to `rf_sup_apply_rx_address(...)`, which in turn calls `nrf24_recv_set_rx_address(...)`: if the receiver is active and not already suspended, the driver suspends it, rewrites `RX_ADDR_P1`, then resumes; if it is already suspended, it rewrites in place; if it has not been started yet, it just stashes the address for the next bringup. Resume waits `Tstby2a` (130 µs max per the nRF24L01 / nRF24L01+ timing tables) after CE returns high, and the active-path resume also clears STATUS / flushes RX before re-enabling IRQ delivery. During this window, plus the network round-trip from `cmd/rf_code` publish to the receiver applying the change, a dispatch sent to the new address can be missed until the receiver acks `applied`. Acceptable because rotation is admin-driven and rare; the admin UI surfaces `ack = PENDING` so the operator knows not to dispatch through this device until it lands.

#### D5.2 — Admin-set code (passive PT2272)

Applies to `RECEIVER_433M_PASSIVE`, used at registration (D2.1) and **only** at registration — rotation is rejected because the chip is hardware-fixed.

- `POST /api/devices/{id}/rf-code` returns `400 Bad Request` for `RECEIVER_433M_PASSIVE` with body `{"error":"hardware_fixed_code","message":"PT2272 codes are set at the chip and cannot rotate. Replace the device or update the chip's DIP switches and re-register."}`.
- The registration path supplies plaintext directly: `RfCodeValidator.validate` plus §D5.0 guards run, then `INSERT` with `version = 1`, `ack = APPLIED`, `ack_at = now()` — recording the truthful state immediately because no acknowledgement is ever coming.
- Plaintext is encrypted on write with the same `encryption-key`. No MQTT publish, no signing. The transmitter reads the row at dispatch time exactly like an MCU code.
- Passive devices have no acks; `ack` stays `APPLIED` from the moment of registration.

### Phase D6 — Lifecycle

`DeviceLifecycleService.issue(deviceId, action, adminId)`:

- Allowed actions: `suspend`, `resume`, `decommission`.
- Reject any command for a `DECOMMISSIONED` device.
- **Passive devices (`RECEIVER_433M_PASSIVE`)**: skip the MQTT path entirely. Flip `device.status` directly (`suspend → SUSPENDED`, `resume → ACTIVE`, `decommission → DECOMMISSIONED`) and return synchronously. Dispatch (§D7a) consults `device.status` before binding, so a SUSPENDED passive device is unreachable at the queue level even though the chip itself never stopped listening — that's the trade-off for owning a hardware decoder.
- **MCU devices**: sign `deact-v1|public_id|command_id|action|issued_at`, publish retained `cmd/deact`, write `device:lifecycle:{deviceId}` with `{commandId, action, issuedAt, ackStatus: "PENDING"}` and a 30-minute TTL. No lifecycle-command columns on the root `device` table.

Ack handling (MCU only):

- Normalise lower-case MQTT statuses to uppercase `DeviceLifecycleAckStatus`.
- Load `device:lifecycle:{deviceId}` by public-id lookup. If missing, update `last_seen_at`, log, and stop.
- On `OK`: `suspend → SUSPENDED`, `resume → ACTIVE`, `decommission → DECOMMISSIONED`, then delete the lifecycle key.
- On `IGNORED` or `REJECTED`: update the lifecycle key with the ack status until TTL expiry so the admin detail page can show the recent result without making it durable schema.
- On decommission `OK`, clear retained `cmd/rf_code` and `cmd/deact`.

### Phase D7 — Queue Dispatch

D7 has a hard prerequisite that this plan does **not** fulfil: physical RF transmission requires a registered, live transmitter hub from [TRANSMITTER_ESP32C3_DESIGN_GUIDE.md](../planned/TRANSMITTER_ESP32C3_DESIGN_GUIDE.md) §H. Because of that, D7 splits into two parts:

- **D7a — backend plumbing (lands in this plan).** `DeviceDispatchService`, `DeviceDispatchEventBroadcaster`, `TicketDto` widening, `QueueService.issueTicket` extra-fields path, no-show / serve / cancel hooks, `device:busy:{deviceId}` lifecycle, and the admin endpoints (`POST /device-tickets`, `GET /available-devices`). Until the transmitter consumer is wired and a hub is `ACTIVE` for the store, dispatch fails closed at the preflight (step 1 below) with `409 Conflict {"error":"no_active_transmitter"}` *before* touching Redis or the queue.
- **D7b — user-facing dispatch UI (deferred to the transmitter plan, see §11).** The queue dispatch button, the device-selection dialog, and the in-row `Radio` ticket badge ship only when at least one transmitter hub can be `ACTIVE` for a store. Rendering them earlier would surface a button that always errors. The receiver detail page does land in this plan: it shows device status, RF code metadata, lifecycle controls, and a "queue dispatch awaiting transmitter" notice on the dispatched-ticket panel until D7b lands.

GitNexus warning: `TicketDto` is HIGH risk (8 d=1 dependents). Land D7a only after D0-D6 are stable, and update all consumers in one slice: `QueueService.issueTicket`, `callNext`, `callSpecificTicket`, `listWaitingTickets`, `getTicketDto`, `CallNextResult`, `QueueAdminController`, and the admin-web queue types/components.

`TicketDto` gains:

```kotlin
val deviceId: UUID? = null
val deviceName: String? = null
```

Update the matching `web/src/types/queue.ts`.

`QueueService.issueTicket` gains an optional internal parameter:

```kotlin
suspend fun issueTicket(
  storeId: UUID,
  serviceTypeId: UUID? = null,
  extraFields: Map<String, String> = emptyMap()
): TicketDto
```

`extraFields` is written to the ticket hash *after* the `ISSUE_TICKET_SCRIPT` Lua call returns, using the same `redis.opsForHash().put(...)` pattern that already writes `service_type_id`. The Lua script itself is **not** modified — the post-script HSET preserves the existing TTL set by the script.

`DeviceDispatchService.issueDeviceTicket(storeId, deviceId, adminId, serviceTypeId)`:

1. Preflight dispatch availability. Until the transmitter dispatch consumer is implemented and a live hub exists for the store, fail before writing Redis with `409 Conflict {"error":"no_active_transmitter"}`.
2. Load device; require `status = ACTIVE`, same store, receiver kind, no `device:busy:{deviceId}` key, and a `device_rf_code` row.
3. Set `device:busy:{deviceId}` with `NX` and `RedisTTLPolicy.TICKET_WAITING`.
4. Call `QueueService.issueTicket(..., extraFields = mapOf("device_id" to deviceId.toString()))`.
5. If ticket issuance or extra-field persistence fails, delete the busy key before rethrowing. If the failure happens after the Lua issue script wrote a ticket, remove that ticket from both global/service queues and delete its ticket hash before returning an error.
6. Return the resulting `TicketDto` with device fields populated.

`GET /available-devices` returns only receiver devices that are `ACTIVE`, store-owned by the requested store, have a current `device_rf_code` row, and have no `device:busy:{deviceId}` key.

Call hooks (in `callNext` and `callSpecificTicket`):

- Inspect `ticketData["device_id"]`.
- If present, skip `FcmNotificationService.sendTicketCalledNotification`.
- Publish an in-process `DEVICE_CALL_REQUESTED` event through `DeviceDispatchEventBroadcaster` (mirrors the existing `QueueEventBroadcaster` pattern in `core/sse`).
- Refresh `device:busy:{deviceId}` to `RedisTTLPolicy.TICKET_CALLED`.

Serve / cancel hooks:

- If `device_id` is present, publish `DEVICE_STOP_REQUESTED` and delete `device:busy:{deviceId}`.

No-show hooks (cover both branches of `handleNoShow`):

- Requeue branch: publish `DEVICE_STOP_REQUESTED`, keep the `device_id` field on the ticket hash, reset busy TTL to `TICKET_WAITING`.
- Skip branch: publish `DEVICE_STOP_REQUESTED`, delete busy.

`DeviceDispatchEventBroadcaster` is the producer-side hand-off. The consumer that turns these events into RF transmissions lives in [TRANSMITTER_ESP32C3_DESIGN_GUIDE.md](../planned/TRANSMITTER_ESP32C3_DESIGN_GUIDE.md) under `H.7.5 Phase T5 — Dispatch consumer`. Until that consumer is wired and a hub is `ACTIVE` for the store, the preflight surfaces `no_active_transmitter` before a ticket is issued, and the queue UI dispatch surface (D7b) is not rendered.

---

## 9. Admin Web Plan

### 9.1 Routes

```text
web/src/app/[locale]/dashboard/devices/page.tsx
web/src/app/[locale]/dashboard/devices/pending/page.tsx
web/src/app/[locale]/dashboard/devices/tokens/page.tsx
web/src/app/[locale]/dashboard/devices/[id]/page.tsx
```

For `[id]`, prefer an async server wrapper:

```tsx
export default async function DeviceDetailPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;
  return <DeviceDetailContent deviceId={id} />;
}
```

Use React `use(params)` only when the page itself must be a client component; `useParams` from `next/navigation` is also fine inside nested client components.

### 9.2 Sidebar

Update the `NavItem.label` union in `web/src/components/layout/sidebar.tsx`:

```ts
label:
  | "overview"
  | "queue"
  | "analytics"
  | "stores"
  | "admins"
  | "settings"
  | "devices";
```

Append a `devices` item using the lucide `Radio` icon. Add `navigation.devices` to both message files. Do not hide the nav item by role — backend scoping controls data visibility.

### 9.3 Feature Folder

Add `web/src/features/device/`:

```text
api.ts
types.ts
device-list-table.tsx
device-status-badge.tsx
device-filter-bar.tsx
pending-review-card.tsx
approve-dialog.tsx
reject-dialog.tsx
rf-code-editor.tsx                 # rotation editor (MCU); read-only render for passive devices (§9.8)
lifecycle-panel.tsx
enrollment-token-dialog.tsx
enrollment-token-table.tsx
passive-device-form-dialog.tsx     # PT2272 manual register dialog (§9.8)
dispatched-ticket-panel.tsx
use-device-ack-poll.ts
```

Use the existing repo pattern:

- API calls via `get`, `post`, `put`, `del` from `@/lib/api`; routes from `API_ROUTES`.
- Tables: native `<table>` markup inside `glass-card`, matching the existing admin/store pattern.
- Reuse already-present shadcn primitives: `Button`, `Badge`, `Dialog`, `AlertDialog`, `Input`, `Label`, `Select`, `Switch`, `Skeleton`, `Tooltip`, `InlineError`, `Sonner`.
- If a missing primitive is genuinely needed, add it via the shadcn CLI/source pattern and log it in `CHANGELOGS.md`. Do not add another UI framework.

### 9.4 UI Rules

Follow `docs/walkthrough/Web Styles.md` and current admin-web conventions:

- Use `glass-card` / `glass-panel` surfaces consistently with queue, store, and admin pages. Never nest cards (the design system explicitly forbids glass-on-glass); a detail page uses sibling panels.
- Use theme tokens (`primary`, `action`, `success`, `warning`, `destructive`, `muted`, `border`) — no arbitrary new colors.
- Move long Tailwind class lists to `web/src/styles/device.css`.
- Action icons (`Radio`, `Copy`, `RotateCcw`, `Power`, `Ban`, `Trash2`, `Check`, `X`, `Loader2`) need accessible labels and tooltips when icon-only.
- `AlertDialog` for reject, decommission, reprovision, and token-revoke confirmations. `Decommission` uses destructive styling.
- One-time token display lives in a dialog with a copy action and a clear warning. The plaintext token never returns once the dialog closes.
- Native tables wrap in `overflow-x-auto` with stable min widths.
- No marketing hero, no explanatory feature panels, no duplicate edit affordances.
- All visible strings flow through `en.json` and `vi.json`. Both are valid JSON — no comments.

### 9.5 API Constants

Append to `web/src/lib/constants.ts`:

```ts
DEVICES: {
  BASE: "/api/devices",
  BY_ID: (id: string) => `/api/devices/${id}`,
  PASSIVE: "/api/devices/passive",
  APPROVE: (id: string) => `/api/devices/${id}/approve`,
  REJECT: (id: string) => `/api/devices/${id}/reject`,
  RF_CODE: (id: string) => `/api/devices/${id}/rf-code`,
  LIFECYCLE: (id: string) => `/api/devices/${id}/lifecycle`,
  REPROVISION: (id: string) => `/api/devices/${id}/reprovision`,
  TOKENS: "/api/devices/enrollment-tokens",
  TOKEN_BY_HASH: (hash: string) => `/api/devices/enrollment-tokens/${hash}`,
},
```

Leave `QUEUE` unchanged in this plan. The queue-dispatch route helpers for `POST /api/queue/admin/{storeId}/device-tickets` and `GET /api/queue/admin/{storeId}/available-devices` belong to D7b in [TRANSMITTER_ESP32C3_DESIGN_GUIDE.md](../planned/TRANSMITTER_ESP32C3_DESIGN_GUIDE.md).

### 9.6 i18n Keys

Add valid JSON entries to both `en.json` and `vi.json`. Vietnamese copy follows the user's natural / concise style — no English fallbacks left in `vi.json`.

- `navigation.devices`
- `devices.title`, `devices.pendingTab`, `devices.tokensTab`, `devices.listTitle`, `devices.emptyState`
- `devices.groupByStore`, `devices.filterStatus`, `devices.filterKind`, `devices.filterHardware`, `devices.filterStore`
- `devices.status.{PENDING, PENDING_RF_CODE, ACTIVE, SUSPENDED, DECOMMISSIONED, REJECTED}`
- `devices.kind.{RECEIVER_433M, RECEIVER_433M_PASSIVE, RECEIVER_2_4G, TRANSMITTER_HUB}`
- `devices.pending.*`, `devices.rfCode.*`, `devices.lifecycle.*`, `devices.dispatch.*` (including `devices.dispatch.awaitingTransmitter` for the dispatched-ticket panel's empty state until D7b lands), `devices.tokens.*`
- `devices.passive.*` — full list in §9.8

Queue-page dispatch copy stays deferred with D7b. Do not add `queue.dispatch.*` in this plan.

### 9.7 Queue UI (deferred — D7b)

The queue page **does not** gain a dispatch button or per-row device badge in this plan. Rendering them now would expose a control that always errors with `no_active_transmitter` until the transmitter plan ships a registered, live hub (see Phase D7 above). The `web/src/app/[locale]/dashboard/queue/page.tsx` change set is therefore:

- **In this plan**: no edits to `queue/page.tsx`. `web/src/types/queue.ts` does add the optional `deviceId` / `deviceName` fields on `TicketDto` so the type stays in lockstep with the backend D7a slice — they simply never carry data until D7b.
- **Deferred to the transmitter plan**: header action button, device-selection `Dialog`, queue-specific i18n, queue route helpers for `/device-tickets` and `/available-devices`, and the in-row `Radio` badge linking to `/dashboard/devices/{deviceId}`. The transmitter plan owns rendering these only when at least one hub can be `ACTIVE` for the store.

Existing Call Next, serve, cancel, no-show, transfer, and queue-state controls are unchanged.

### 9.8 Passive PT2272 Manual Registration UI

Passive PT2272 receivers have no firmware and cannot self-enroll. The admin reads the chip's preset code (often from DIP switches or a label) and registers manually. The UI uses only primitives already shipped in `web/src/components/ui` (`alert-dialog`, `badge`, `button`, `card`, `dialog`, `dropdown-menu`, `inline-error`, `input`, `label`, `popover`, `select`, `separator`, `skeleton`, `sonner`, `switch`, `tooltip`).

#### 9.8.1 Entry point and dialog

In `web/src/app/[locale]/dashboard/devices/page.tsx`, next to the **Issue enrollment token** button:

- Lucide `Plus` icon (existing add-action convention in the repo).
- Label key: `devices.passive.registerAction`.
- Disabled when ADMIN has no `principal.storeId`; tooltip uses `devices.passive.disabledNoStore`.
- Click opens `<PassiveDeviceFormDialog />`. Dialog-only flow — no separate route — matching the precedent of `store-form-dialog.tsx` and `service-type-form-dialog.tsx`.

The button is **not** role-gated in the UI; backend `StoreAccessUtil` enforces authority. The disabled state is purely a UX hint.

`passive-device-form-dialog.tsx` submits to `DEVICES.PASSIVE` via `post` from `@/lib/api`. Body shape per §7. Form fields:

| Field | Component | Notes |
|---|---|---|
| Kind | `Badge` | Read-only, `devices.kind.RECEIVER_433M_PASSIVE`. Hardcoded in payload. |
| Hardware model | `Badge` | Read-only, `PT2272`. Hardcoded in payload. |
| Store | `Select` | SUPER_ADMIN: reuse `listStores` from the store feature and load enough pages for a selector; do not couple this dialog to the store-management table state. ADMIN: single read-only entry forced to `principal.storeId`. Required. |
| Assigned name | `Input` | Required, `maxLength={100}`. |
| Bit width | `Select` | Options 1..24; default 24. Selecting a width re-derives the hex field's `maxLength`. Helper text via `devices.passive.bitsHelp` explains the 24-bit cap is a PT2272 hardware limit. |
| RF code hex | `Input` with `font-mono` | `[0-9A-Fa-f]+`, uppercased on blur. `maxLength = 2 * Math.ceil(bits / 8)`. |

Field errors render via `InlineError`; field labels via `Label`; helper text uses the existing muted-text pattern from `store-general-fields.tsx`. Footer: primary `Button` (`devices.passive.submit`) with a `Loader2` spinner during submit (matching `create-admin-dialog.tsx`); secondary `Button variant="outline"` (`common.cancel`, existing key). `Dialog` already provides its surface — do not nest it inside a `glass-card`.

On success: close, `toast.success` with `devices.passive.successToast` (interpolated with the new device's `public_id`), and re-fetch the device list (parent owns the query state, same pattern as `delete-admin-dialog.tsx`).

#### 9.8.2 Validation

Client-side (blocks submit; mirrors §2.4 + §D5.0):

- `assignedName.trim().length > 0`.
- `storeId` present.
- `rfCodeBits ∈ [1, 24]`.
- `rfCodeHex` matches `/^[0-9A-F]+$/i` and `rfCodeHex.length === 2 * Math.ceil(rfCodeBits / 8)`.

Server-side (surface via `toast` from `sonner` plus `InlineError` next to the relevant field):

- `409 Conflict` on uniqueness clash → `devices.passive.errorConflict`. The response body includes the colliding `public_id`; render it as plain text in the toast (no link). Never render the colliding code itself.
- `400 Bad Request` with structured `error` field (`pt2272_hardware_cap_exceeded`, `forbidden_pattern`, `width_out_of_range`) → keys under `devices.passive.errors.*`. The hardware-cap variant is the authoritative server-side echo of the `Select` cap, in case a client bypasses the form.
- `403 Forbidden` from `StoreAccessUtil` → `devices.passive.errorForbidden`.

#### 9.8.3 Detail page consistency for passive devices

`/dashboard/devices/{id}` renders panels conditionally on `device.kind`:

- `rf-code-editor.tsx` (`kind === "RECEIVER_433M_PASSIVE"`): render read-only. Show `bits` and a `Badge` reading `••  ••  ••` via key `devices.rfCode.maskedValue`. Hide the **Rotate** button. Replace the panel description with `devices.rfCode.lockedNote` — a frontend-i18n string that conveys the same meaning as the server's `hardware_fixed_code` 400 (§D5.2). Do not parrot the server's English message; the server returns the structured `error` code, the UI provides the user-facing copy.
- `lifecycle-panel.tsx`: behaviour is unchanged (backend handles passive lifecycle as a status flip per §D6). Add `devices.lifecycle.passiveSubtitle` so admins understand the chip itself does not stop listening.
- The **Reprovision** action is hidden for passive devices (the server returns 404 anyway per §7).

#### 9.8.4 i18n keys (`devices.passive.*`)

Add to both `en.json` and `vi.json`:

- `devices.passive.registerAction`
- `devices.passive.disabledNoStore`
- `devices.passive.dialogTitle`, `devices.passive.dialogDescription`
- `devices.passive.kindLabel`, `devices.passive.hardwareLabel`, `devices.passive.storeLabel`, `devices.passive.nameLabel`, `devices.passive.bitsLabel`, `devices.passive.bitsHelp`, `devices.passive.hexLabel`, `devices.passive.hexHelp`
- `devices.passive.hexErrorFormat`, `devices.passive.hexErrorLength`
- `devices.passive.submit`, `devices.passive.successToast`
- `devices.passive.errorConflict`, `devices.passive.errorForbidden`
- `devices.passive.errors.pt2272_hardware_cap_exceeded`, `devices.passive.errors.forbidden_pattern`, `devices.passive.errors.width_out_of_range`
- `devices.rfCode.maskedValue`, `devices.rfCode.lockedNote`
- `devices.lifecycle.passiveSubtitle`

---

## 10. Execution Order

1. **D0** — Impact checks, schema/migration, config, signing, canonical helpers, MQTT wrapper, Redis keys.
2. **D1** — Enrollment token backend + admin-web token screen.
3. **D2 + D2.1** — MCU bootstrap registration and PT2272 manual register; admin-web pending-review and manual-register UI.
4. **D3-D4** — Approval, activation.
5. **D5** — Forbidden-set + uniqueness, auto-issue / rotation / admin-set, ack handling, RF-code editor.
6. **D6** — Lifecycle commands and lifecycle panel.
7. **D7a** — Queue-dispatch backend plumbing: `DeviceDispatchService`, `DeviceDispatchEventBroadcaster`, `TicketDto` widening, `QueueService.issueTicket` extra-fields path, serve / cancel / no-show hooks, `device:busy:*` lifecycle, and admin endpoints. Preflight returns `no_active_transmitter` until the transmitter plan lands. **D7b — queue UI dispatch button, device-selection dialog, and per-row badge — is owned by [TRANSMITTER_ESP32C3_DESIGN_GUIDE.md](../planned/TRANSMITTER_ESP32C3_DESIGN_GUIDE.md)** because they cannot be exercised end-to-end without a live hub.

Each phase updates `docs/CHANGELOGS.md` with every file touched and any skipped item. After all phases ship, mark the notifier/device-domain row complete in `docs/planned/Backend Remaining Plan.md` with a link back here.

---

## 11. Deferred

- Transmitter hub backend + firmware plan — owned by [TRANSMITTER_ESP32C3_DESIGN_GUIDE.md](../planned/TRANSMITTER_ESP32C3_DESIGN_GUIDE.md). The receiver schema reserves `TRANSMITTER_HUB`, the `hub-XXXXX` `public_id` prefix, and `device.last_seen_at` for heartbeats, so no schema retrofit is needed when that plan lands.
- D7b queue dispatch UI — the queue page's dispatch button, the device-selection `Dialog`, and the per-row `Radio` ticket badge. Owned by the transmitter plan because the dispatch event has no consumer until a hub is registered and live; rendering D7b here would surface a button that always returns `no_active_transmitter`.
- Transmitter hub firmware (SoftAP provisioning, NVS layout, on-device key gen, RF driver).
- Web Serial bridge.
- Per-device MQTT credentials and broker ACLs.
- SSE for device acks; v1 polls detail pages while ack is pending.
- RF-code history beyond the current version.
- `DEVICE_TRIGGERED` analytics emission.
- Backend command-signing key rotation in the field.
- A dedicated `device_event` audit table.
