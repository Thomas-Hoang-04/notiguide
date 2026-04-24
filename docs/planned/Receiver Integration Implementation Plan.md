# Receiver Integration Implementation Plan

Concrete backend + admin-web work required to integrate the ESP-01
receiver (`docs/planned/RECEIVER_DESIGN_GUIDE.md`) and the ESP32-C3
receiver (`docs/planned/RECEIVER_ESP32C3_DESIGN_GUIDE.md`) with the
existing notiguide stack.

**Scope boundaries.**

- Firmware work is **out of scope** — it lives in `receiver-esp32/` and
  `receiver-esp8266/` and is tracked by the two design guides.
- This plan covers the Spring Boot backend (`backend/`) and the admin
  dashboard (`web/`) only. The customer-facing `client-web/` is
  unaffected.
- One logical contract drives both device families (`H.1` of the C3
  guide). Do not fork topics, envelope types, or endpoints per family.

**Last updated**: 2026-04-24

---

## 1. Repo baseline (verified 2026-04-24)

Status at the start of this plan, so later steps don't fight stale
assumptions.

Backend (`backend/`):

- Kotlin 2.3, Spring Boot 3.5.11, Java 21, WebFlux + coroutines
  (`backend/build.gradle.kts:1-60`).
- MQTT v5 already wired: `core/mqtt/` contains `MqttConfig`,
  `MqttClientManager`, `MqttPublisher`, `MqttProperties`,
  `MqttMessageHandler`. `MqttClientManager.publish(...)` already
  supports `retained: Boolean`, and `subscribe(topicFilter)` +
  `registerHandler(MqttMessageHandler)` let new listeners plug in.
- `MqttPublisher` is queue-only and hard-codes the `notiguide/store/…`
  topic; it will **not** be reused for receiver topics. Receiver
  topics (`receiver/…`) will publish through `MqttClientManager`
  directly via a new `ReceiverMqttPublisher`.
- `MqttConfig` is `@ConditionalOnProperty(prefix = "mqtt", name =
  ["broker"])` — receiver features silently degrade if MQTT is not
  configured, matching the existing FCM/MQTT graceful-degradation
  pattern.
- Schema at `backend/src/main/resources/db/schema.sql` already contains
  a placeholder `notifier_device` table (FCM-token-shaped, referenced
  by `analytics_event.device_id`). **Keep it** — it is the legacy
  FCM-device slot. The receiver plan uses new tables under a clean
  `receiver_*` prefix so there is no collision.
- Redis keys are centralized in
  `core/redis/RedisKeyManager.kt` and serialized via
  `StringRedisSerializer` (see CLAUDE.md — do **not** introduce
  `GenericJackson2JsonRedisSerializer`).
- Security: `SecurityConfig` permits only `/api/auth/**` and
  `/api/queue/public/**`. New admin endpoints will live under
  `/api/admin/receivers/**` and require authenticated ADMIN /
  SUPER_ADMIN.
- Rate limiting: `RateLimitFilter` runs at `SecurityWebFiltersOrder.FIRST`;
  new endpoints inherit the `standard` tier automatically.
- `AnalyticsEvent.deviceId` already exists but is always written as
  `null` today — future work, not part of this plan.

Admin web (`web/`):

- Next.js 16.2.1, React 19.2.4, next-intl 4.8, Tailwind 4, shadcn/ui
  (`web/package.json`). Biome 2.2 for lint/format.
- Dynamic route params are **async promises** in Next.js 16
  (verified via Context7 against `/vercel/next.js/v16.2.2`). New
  pages must type `params: Promise<{ id: string }>` and unwrap via
  `await params` (RSC) or `use(params)` (client components).
- API client pattern lives at `web/src/lib/api.ts` (cookie-based JWT,
  silent refresh, typed `get/post/put/del`). `API_ROUTES` is the
  single source of URL truth in `web/src/lib/constants.ts`.
- Sidebar definition at `web/src/components/layout/sidebar.tsx`
  uses `navItems[]` + i18n `navigation.*` keys. New "Receivers" tab
  plugs in there.
- Bilingual messages: `web/src/messages/{en,vi}.json`. Existing
  domain namespaces: `stores`, `admins`, `queue`, `analytics`,
  `settings`. Add `receivers`.
- Only shadcn/ui is allowed for UI primitives (CLAUDE.md).

SDK / spec docs (firmware-side, referenced from this plan for payload
parity, not work items):

- `docs/planned/RECEIVER_DESIGN_GUIDE.md` — ESP-01 / ESP8266 reference.
- `docs/planned/RECEIVER_ESP32C3_DESIGN_GUIDE.md` — ESP32-C3 target.
- `docs/planned/Backend Remaining Plan.md` — P1 is "Implement Notifier
  Device Domain" and explicitly points to receiver work — this plan
  is how that P1 gets executed.

---

## 2. Shared contract (both receiver families)

Authoritative source: `RECEIVER_ESP32C3_DESIGN_GUIDE.md §H`. ESP-01
and ESP32-C3 devices share everything here.

### 2.1 MQTT topics

| Topic | Dir | QoS | Retained | Use |
|---|---|---|---|---|
| `receiver/bootstrap/register` | dev→be | 1 | no | registration payload |
| `receiver/bootstrap/{challenge_id}` | both | 1 | no | `pending`/`challenge`/`rejected`/`result`/`response` envelopes |
| `receiver/device/{public_id}/cmd/rf_code` | be→dev | 1 | **yes** | signed RF trigger code |
| `receiver/device/{public_id}/cmd/deact` | be→dev | 1 | **yes** | signed suspend/resume/decommission |
| `receiver/device/{public_id}/ack` | dev→be | 1 | no | ack stream (`ack_for` discriminator) |

Backend subscriptions at startup:

- `receiver/bootstrap/register` — registration intake
- `receiver/bootstrap/+` — to receive device `response` envelopes; the
  listener must drop any envelope whose `type` is backend-owned
  (`pending`/`challenge`/`rejected`/`result`) to handle self-echo
  (3.1.1 has no no-local).
- `receiver/device/+/ack` — single stream; dispatch on `ack_for`
  (`rf_code` | `deact`).

### 2.2 Canonical strings (signed)

Every canonical string is UTF-8, no trailing newline, `|`-joined, with
exact field spellings as on the wire:

- Device → backend (activation):
  `activate-v1|<challenge_id>|<nonce>|<issued_at>|<expires_at>`
- Backend → device (RF code):
  `rf-code-v1|<public_id>|<code_version>|<rf_code_hex>|<rf_code_bits>|<issued_at>`
- Backend → device (deactivation):
  `deact-v1|<public_id>|<command_id>|<action>|<issued_at>`

`rf_code_hex` is upper-case hex; `issued_at` / `expires_at` are
ISO-8601 `Z` form; integers are unprefixed base-10. Any drift here
breaks signature verification silently, so these formatters live in a
single helper (`ReceiverCanonical.kt`).

### 2.3 Cryptography

- Curve: EC P-256 (secp256r1)
- Sig alg: `SHA256withECDSA`
- Device keypair: DER in NVS on firmware; `devices.public_key_der`
  server-side
- Backend command-signing keypair: offline-generated; private half
  injected via Spring Application Properties; public half pinned in
  firmware via Kconfig (§3.7 / §E.4).

### 2.4 Registration validation

| hardware_model | legal receiver_type |
|---|---|
| `ESP-01` | `RECEIVER_433M` |
| `ESP32-C3` | `RECEIVER_433M`, `RECEIVER_2_4G` |

RF-code width rules (validated **before** signing):

- `RECEIVER_433M`: `rf_code_bits ∈ [1,32]`,
  `hex_len == 2 * ceil(bits/8)`
- `RECEIVER_2_4G`: `rf_code_bits ∈ [8,256]`, `bits % 8 == 0`,
  `hex_len == bits/4`

---

## 3. Backend — data model

New tables under `backend/src/main/resources/db/schema.sql` (append;
do not migrate the existing `notifier_device` table):

```sql
CREATE TYPE receiver_status AS ENUM (
  'PENDING', 'ACTIVE', 'SUSPENDED', 'DECOMMISSIONED', 'REJECTED'
);

CREATE TYPE receiver_hardware_model AS ENUM ('ESP-01', 'ESP32-C3');
CREATE TYPE receiver_radio_type     AS ENUM ('RECEIVER_433M', 'RECEIVER_2_4G');

CREATE TYPE receiver_challenge_status AS ENUM (
  'PENDING_APPROVAL', 'ISSUED', 'RESPONDED', 'EXPIRED', 'REJECTED'
);

CREATE TYPE receiver_rf_code_ack AS ENUM ('applied', 'unchanged', 'rejected', 'pending');
CREATE TYPE receiver_deact_ack   AS ENUM ('ok', 'ignored', 'rejected', 'pending');
CREATE TYPE receiver_deact_action AS ENUM ('suspend', 'resume', 'decommission');

CREATE TABLE receiver_device (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  public_id           VARCHAR(32) UNIQUE,          -- null until activation
  public_key_der      BYTEA        NOT NULL UNIQUE,
  hardware_model      receiver_hardware_model NOT NULL,
  receiver_type       receiver_radio_type     NOT NULL,
  firmware_version    VARCHAR(32),
  status              receiver_status NOT NULL DEFAULT 'PENDING',
  assigned_name       VARCHAR(100),
  store_id            UUID REFERENCES store(id) ON DELETE SET NULL,
  registration_nonce  VARCHAR(32),
  pending_since       TIMESTAMP WITH TIME ZONE,
  created_at          TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
  activated_at        TIMESTAMP WITH TIME ZONE,
  updated_at          TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_receiver_device_store  ON receiver_device(store_id);
CREATE INDEX idx_receiver_device_status ON receiver_device(status);

CREATE TABLE receiver_activation_challenge (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  challenge_id        UUID UNIQUE NOT NULL,
  device_pk           UUID NOT NULL REFERENCES receiver_device(id) ON DELETE CASCADE,
  registration_nonce  VARCHAR(32) NOT NULL,
  nonce               VARCHAR(64),                 -- set on approve
  purpose             VARCHAR(32) NOT NULL DEFAULT 'activate-v1',
  issued_at           TIMESTAMP WITH TIME ZONE,
  expires_at          TIMESTAMP WITH TIME ZONE,
  responded_at        TIMESTAMP WITH TIME ZONE,
  status              receiver_challenge_status NOT NULL DEFAULT 'PENDING_APPROVAL'
);
CREATE INDEX idx_receiver_chal_device ON receiver_activation_challenge(device_pk);
CREATE INDEX idx_receiver_chal_status ON receiver_activation_challenge(status);

CREATE TABLE receiver_rf_code (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  device_pk           UUID NOT NULL REFERENCES receiver_device(id) ON DELETE CASCADE,
  public_id           VARCHAR(32) NOT NULL,        -- records the activation it was sent under
  rf_code_encrypted   BYTEA NOT NULL,              -- pgcrypto-encrypted bytes (1..32 B plaintext)
  rf_code_bits        SMALLINT NOT NULL,
  code_version        INT NOT NULL,
  issued_at           TIMESTAMP WITH TIME ZONE NOT NULL,
  last_ack_status     receiver_rf_code_ack NOT NULL DEFAULT 'pending',
  last_ack_at         TIMESTAMP WITH TIME ZONE,
  UNIQUE (device_pk, public_id, code_version)
);
CREATE INDEX idx_receiver_rfcode_device ON receiver_rf_code(device_pk, code_version DESC);

CREATE TABLE receiver_deactivation_command (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  device_pk           UUID NOT NULL REFERENCES receiver_device(id) ON DELETE CASCADE,
  public_id           VARCHAR(32) NOT NULL,
  command_id          UUID UNIQUE NOT NULL,
  action              receiver_deact_action NOT NULL,
  issued_by_admin_id  UUID REFERENCES admin(id) ON DELETE SET NULL,
  issued_at           TIMESTAMP WITH TIME ZONE NOT NULL,
  acknowledged_at     TIMESTAMP WITH TIME ZONE,
  ack_status          receiver_deact_ack NOT NULL DEFAULT 'pending'
);
CREATE INDEX idx_receiver_deact_device ON receiver_deactivation_command(device_pk, issued_at DESC);
```

Notes:

- `store_id` maps the guide's `tenant_id`/`site_id` onto this
  project's single-level store model. A SUPER_ADMIN can assign/detach
  receivers from a store during approval; ADMIN sees only their own
  store's receivers.
- `rf_code_encrypted`: encrypt at rest with `pgp_sym_encrypt` / a
  backend-held `app.secrets.rfCodeKey`. Plaintext widens to up to
  32 bytes on the 2.4 GHz path.
- `public_id` on child tables is denormalized (not an FK) because a
  new activation mints a new `public_id` — the child rows pin the
  activation they were issued under for audit, even after the parent
  `receiver_device.public_id` rotates.

### 3.1 Enrollment tokens (Redis)

Per `§12.1` / `§H.7`, tokens are Redis-only:

- Key: `enroll:<sha256-hex-of-token>`
- Value: JSON `{storeId, issuedByAdminId, issuedAt}` (string-serialized
  — matches the existing `StringRedisSerializer` policy)
- TTL: `enrollment.tokenTtlSeconds` (default 3600)
- Issue: `SET … EX … NX`
- Consume (atomic, single-use): `GETDEL`
- Revoke: `DEL`

Add to `RedisKeyManager`:

```kotlin
fun enrollmentToken(sha256Hex: String) = "enroll:$sha256Hex"
fun enrollmentTokenPattern()          = "enroll:*"
```

Plaintext **never** lands in Postgres or the `receiver_device` row.
The admin UI shows the plaintext exactly once when the token is
created.

---

## 4. Backend — package layout

New domain package `domain/receiver/` mirroring the existing
`domain/admin/` / `domain/queue/` shape:

```
backend/src/main/kotlin/com/thomas/notiguide/
├── core/
│   └── receiver/                                    # cross-cutting infra for receivers
│       ├── ReceiverCommandSigner.kt                 # SHA256withECDSA sign/verify helper
│       ├── ReceiverCommandSigningProperties.kt      # @ConfigurationProperties
│       ├── ReceiverCommandSigningConfig.kt          # PEM → PrivateKey @Bean
│       ├── ReceiverCanonical.kt                     # canonical string builders (§2.2)
│       ├── ReceiverMqttPublisher.kt                 # publishes to receiver/* topics
│       └── ReceiverPublicIdMinter.kt                # collision-checked short-id gen
└── domain/receiver/
    ├── entity/
    │   ├── ReceiverDevice.kt
    │   ├── ActivationChallenge.kt
    │   ├── ReceiverRfCode.kt
    │   └── DeactivationCommand.kt
    ├── repository/
    │   ├── ReceiverDeviceRepository.kt
    │   ├── ActivationChallengeRepository.kt
    │   ├── ReceiverRfCodeRepository.kt
    │   └── DeactivationCommandRepository.kt
    ├── service/
    │   ├── EnrollmentTokenService.kt                # Redis GETDEL / SET NX EX
    │   ├── DeviceRegistrationService.kt             # register intake
    │   ├── DeviceApprovalService.kt                 # approve/reject
    │   ├── ActivationChallengeService.kt            # mint nonce, publish challenge
    │   ├── DeviceActivationService.kt               # verify sig, mint public_id
    │   ├── RfCodeService.kt                         # rotate + sign + publish
    │   └── DeactivationService.kt                   # suspend/resume/decommission
    ├── listener/
    │   ├── MqttBootstrapListener.kt                 # subs: bootstrap/register + bootstrap/+
    │   └── MqttOperationalListener.kt               # sub : device/+/ack
    ├── controller/
    │   ├── EnrollmentTokenController.kt             # admin token CRUD
    │   └── ReceiverAdminController.kt               # /api/admin/receivers/**
    ├── dto/ …
    ├── request/ …
    └── types/                                       # enums mirrored onto Kotlin
        ├── ReceiverHardwareModel.kt
        ├── ReceiverRadioType.kt
        ├── ReceiverStatus.kt
        ├── ActivationChallengeStatus.kt
        ├── RfCodeAckStatus.kt
        ├── DeactAckStatus.kt
        └── DeactAction.kt
```

---

## 5. Backend — implementation phases

Each phase ends on a working slice; audits (per project convention)
run between phases.

### Phase R0 — Foundation (infra)

1. **Schema** — append §3 tables + enums + indexes to `schema.sql`.
2. **RedisKeyManager** — add `enrollmentToken`/pattern (§3.1).
3. **Command-signing key plumbing** —
   `ReceiverCommandSigningProperties` binds:
   ```yaml
   receiver:
     command-signing:
       private-key: ${RECEIVER_CMD_SIGNING_PRIVATE_KEY_PEM:}          # inline PEM (prod)
       private-key-path: ${RECEIVER_CMD_SIGNING_PRIVATE_KEY_PATH:}    # classpath:/file path (dev)
   ```
   `ReceiverCommandSigningConfig` is
   `@ConditionalOnProperty(prefix = "receiver.command-signing",
   name = ["private-key", "private-key-path"],
   matchIfMissing = false)` and exposes a `PrivateKey` bean. Load via
   Spring's `PemContent.load(Path).getPrivateKey()`
   (Spring Boot 3.2+, verified via Context7 for 3.5) — avoids
   hand-rolling PEM parsing. Falls back to raw PEM text via
   `PemContent.of(text)` when the inline form is used.
   `ReceiverCommandSigner` wraps `Signature.getInstance("SHA256withECDSA")`.
4. **`ReceiverCanonical`** — pure functions returning the exact
   canonical strings of §2.2 (no Jackson; direct string concat with
   the documented formatters). Backed by unit tests (see §7).
5. **`ReceiverMqttPublisher`** — thin wrapper over `MqttClientManager`
   that knows the `receiver/…` topics, applies JSON serialization via
   Jackson, and defaults `qos=1`. Exposes:
   - `publishBootstrapEnvelope(challengeId: UUID, envelope: Envelope)`
   - `publishRfCode(publicId: String, payload: RfCodePayload)` (retained)
   - `publishDeactCommand(publicId: String, payload: DeactPayload)` (retained)
   - `clearRetained(publicId: String)` — publishes zero-length
     retained payload to both operational cmd topics (hygiene after
     re-activation / decommission).
6. **`ReceiverPublicIdMinter`** — generates collision-checked
   `rcv-XXXXX` (Crockford base32, 5 chars) against
   `receiver_device.public_id`.

### Phase R1 — Enrollment tokens

1. `EnrollmentTokenService` — Redis issue / consume / revoke.
   Canonical token form: 128-bit urandom → 22-char base64url (per
   §12.1). Hash input is the upper-cased trimmed token.
2. `EnrollmentTokenController`:
   - `POST /api/admin/receivers/enrollment-tokens`
     (body: `{storeId?: UUID}`, SUPER_ADMIN or ADMIN for own store):
     returns `{token, expiresAt}` **once**.
   - `DELETE /api/admin/receivers/enrollment-tokens/{sha256}`: revoke.
   - `GET /api/admin/receivers/enrollment-tokens` (optional; MVP can
     skip since the plaintext is unrecoverable — the Redis hash key
     is the only handle).
3. Rate-limit tier: `strict` (token mint is a sensitive admin op).

### Phase R2 — Bootstrap intake

1. `MqttBootstrapListener` registers a single `MqttMessageHandler`
   and subscribes to:
   - `receiver/bootstrap/register`
   - `receiver/bootstrap/+`

   Topic prefix matching routes to:
   - exact `receiver/bootstrap/register` →
     `DeviceRegistrationService.onRegister(payload)`
   - `receiver/bootstrap/{cid}` with `type == "response"` →
     `DeviceActivationService.onResponse(cid, payload)`
   - any other `type` on `receiver/bootstrap/+` → drop (self-echo
     from backend-originated publishes on the same wildcard).
2. `DeviceRegistrationService.onRegister`:
   - Validate `schema_version`, required fields, `(hardware_model,
     receiver_type)` pair (§2.4).
   - `GETDEL enroll:<hash>` — miss ⇒ publish `rejected` with
     `reason: "invalid_token"`.
   - Upsert by `public_key_der`: if a prior row exists, re-scope it
     (new `registration_nonce`, `status = PENDING`,
     `receiver_type = incoming`). Else insert.
   - Insert `receiver_activation_challenge` with
     `challenge_id = UUID.randomUUID()`,
     `status = PENDING_APPROVAL`.
   - Publish `type: "pending"` on
     `receiver/bootstrap/{challenge_id}`.

### Phase R3 — Admin approve / reject

1. `DeviceApprovalService.approve(id: UUID, adminId: UUID)`:
   - Verify caller authority (SUPER_ADMIN, or ADMIN for this device's
     `store_id`).
   - Generate `nonce = base64url(≥16 random bytes)`,
     `expires_at = now + 5m`, flip challenge to `ISSUED`.
   - Publish `type: "challenge"` on
     `receiver/bootstrap/{challenge_id}`.
2. `DeviceApprovalService.reject(id, adminId, reason)`:
   - Publish `type: "rejected"` (reason defaulted to
     `admin_rejected`).
   - Wipe: delete `receiver_activation_challenge` row; set device
     `status = REJECTED` **and keep the row** (audit tombstone).
   - Enrollment token was already consumed in R2; it is **not**
     refunded.

### Phase R4 — Activation

`DeviceActivationService.onResponse(challengeId, payload)`:

1. Load challenge; reject if status ≠ `ISSUED`, or `expires_at` past,
   or `responded_at` already set.
2. Rebuild canonical `activate-v1|cid|nonce|issued_at|expires_at`
   byte-for-byte.
3. `Signature("SHA256withECDSA").initVerify(public_key_der).verify(sig)`.
4. Mint `public_id` via `ReceiverPublicIdMinter`.
5. `@Transactional` commit:
   - `receiver_device.public_id = …`, `status = ACTIVE`,
     `activated_at = now()`, `updated_at = now()`.
   - `receiver_activation_challenge.status = RESPONDED`,
     `responded_at = now()`.
6. Publish `type: "result"` with `{public_id, assigned_device_name}`.
7. **Hygiene** — call `ReceiverMqttPublisher.clearRetained(old_public_id)`
   if the device had a prior activation (re-provisioning case).

No MQTT credential minting in v1 (`§15` / `§H.9` — shared broker
creds; v2 path documented).

### Phase R5 — RF code rotation

`RfCodeService`:

- `issueInitial(deviceId: UUID)` — called from R4 step 6 after
  activation. Picks a code (store default, or a placeholder 24-bit
  preset for 433 MHz / 40-bit preset for 2.4 GHz — settable in R6),
  writes `receiver_rf_code` with `code_version = 1`, signs, publishes
  retained.
- `rotate(deviceId: UUID, req: RotateRequest, adminId: UUID)` —
  validates width rules (§2.4) **before signing**, increments
  `code_version`, writes row (`last_ack_status = pending`), signs,
  publishes retained.
- On ack (`MqttOperationalListener` → dispatch `ack_for = "rf_code"`):
  update `last_ack_status` / `last_ack_at`; never trust/echo the
  code.

### Phase R6 — Deactivation

`DeactivationService.issue(deviceId, action, adminId)`:

- Validate transition: if `status = DECOMMISSIONED`, reject all
  further commands (firmware treats decommission as one-way).
- Insert `receiver_deactivation_command(command_id = UUID, action,
  ack_status = pending)`.
- Sign `deact-v1|public_id|command_id|action|issued_at`.
- Publish retained on `receiver/device/{public_id}/cmd/deact`.
- On ack (`ack_for = "deact"`): update `acknowledged_at` /
  `ack_status`; if ack is `ok` and action ≠ ignored, mirror the
  action onto `receiver_device.status`
  (`suspend→SUSPENDED`, `resume→ACTIVE`,
  `decommission→DECOMMISSIONED`).

### Phase R7 — Admin API

All endpoints require `ROLE_SUPER_ADMIN` or `ROLE_ADMIN`; ADMIN is
scoped to own store via `StoreAccessUtil`. Paths are stable on the
internal UUID from `receiver_device.id`, **not** the rotating
`public_id` (`§H.8`).

```
POST   /api/admin/receivers/enrollment-tokens                  (R1)
DELETE /api/admin/receivers/enrollment-tokens/{sha256}         (R1)

GET    /api/admin/receivers                                    list; filters: status, receiver_type, hardware_model, store_id
GET    /api/admin/receivers/{id}                               detail (device + latest rf_code meta + latest deact cmd)
POST   /api/admin/receivers/{id}/approve                       (R3) body: {assignedName?: string, storeId?: UUID}
POST   /api/admin/receivers/{id}/reject                        (R3) body: {reason?: string}
POST   /api/admin/receivers/{id}/rf-code                       (R5) body: {rfCodeHex, rfCodeBits}
POST   /api/admin/receivers/{id}/deactivation                  (R6) body: {action: "suspend"|"resume"|"decommission"}
POST   /api/admin/receivers/{id}/reprovision                   hygiene: clearRetained + status = PENDING (see §7)
GET    /api/admin/receivers/{id}/rf-codes                      last N rf_code rows (audit)
GET    /api/admin/receivers/{id}/deactivations                 last N deact cmd rows (audit)
```

Register paths in `API_ROUTES.RECEIVERS` on the admin-web side (§6).

Add to `API_ROUTES` source of truth — **no** widening of
`SecurityConfig.permitAll`. All receiver admin paths fall under the
default `.anyExchange().authenticated()` rule.

### Phase R8 — Wiring + config

1. `application.yaml`:
   ```yaml
   receiver:
     command-signing:
       private-key-path: ${RECEIVER_CMD_SIGNING_PRIVATE_KEY_PATH:}
     enrollment:
       token-ttl-seconds: ${RECEIVER_ENROLLMENT_TTL_SECONDS:3600}
     default-rf-code:
       bits-433m: 24
       bits-24g:  40
   ```
2. `application-dev.yaml`: point `private-key-path:
   classpath:receiver/cmd_signing_priv.pem` and ship a dev-only key
   under `backend/src/main/resources/receiver/` (gitignored like the
   RSA / Firebase material already is).
3. `BootJar` exclusion: add `"receiver/**"` to the existing
   `tasks.named<BootJar>("bootJar") { exclude(...) }` block
   (`build.gradle.kts`) so the dev key doesn't ship in prod JARs.
4. Prod reads the key path from an env-var-supplied host path per
   `§3.7`.

---

## 6. Admin-web — implementation

### 6.1 Routing

Add under `web/src/app/[locale]/dashboard/receivers/`:

```
receivers/
├── page.tsx                   list + filters (status / type / model / store)
├── pending/
│   └── page.tsx               review queue (status=PENDING only)
├── tokens/
│   └── page.tsx               enrollment token issuance
└── [id]/
    └── page.tsx               detail: header, rf-code editor, deact controls, audit tabs
```

Dynamic `[id]` pages must type `params: Promise<{ id: string }>`
(Next.js 16 async params — verified via Context7).

### 6.2 Sidebar nav

Append to `navItems[]` in `web/src/components/layout/sidebar.tsx`:

```ts
{
  href: "/dashboard/receivers",
  label: "receivers",                 // i18n key
  icon: <Radio aria-hidden="true" className="size-5" />,
},
```

(`Radio` from `lucide-react`.) No role gate — both SUPER_ADMIN and
ADMIN see receivers; the backend scopes ADMIN to own store.

### 6.3 Feature folder

`web/src/features/receiver/`:

```
api.ts                       typed client (listReceivers, getReceiver,
                             approveReceiver, rejectReceiver, rotateRfCode,
                             issueDeactivation, issueEnrollmentToken,
                             revokeEnrollmentToken, listEnrollmentTokens)
types.ts                     ReceiverDto, ReceiverListResponse, PendingReceiverDto,
                             RfCodeDto, DeactivationCmdDto, EnrollmentTokenDto,
                             ReceiverStatus, ReceiverType, HardwareModel enums
receiver-list-table.tsx      shadcn-styled table, status badges, filter bar
receiver-status-badge.tsx    badge variants per status (match Web Styles.md palette)
receiver-filter-bar.tsx      status / type / model / store select
pending-review-card.tsx      approve / reject buttons + public-key fingerprint
approve-dialog.tsx           fields: assignedName, storeId (super-admin only)
reject-dialog.tsx            reason field
rf-code-editor.tsx           width-aware hex input:
                               - RECEIVER_433M: maxLength 8, common preset 24 bits
                               - RECEIVER_2_4G: maxLength 64, default 40 bits
deactivation-panel.tsx       Suspend / Resume / Decommission buttons with confirm
enrollment-token-dialog.tsx  one-time plaintext display + copy-to-clipboard
rf-code-history-table.tsx    last 20 rf_code rows (code_version, ack status, timestamps — NO hex)
deactivation-history-table.tsx
use-receiver-ack-poll.ts     optional: poll /api/admin/receivers/{id} every 3s
                             while a pending ack is outstanding
```

Conventions already in repo that this domain inherits:

- `import { api, get, post, put, del } from "@/lib/api"` — cookie
  auth + silent refresh already handled.
- shadcn/ui only (CLAUDE.md). Buttons, dialogs, tables, inputs,
  badges, switches — all exist under `web/src/components/ui/`.
- No inline Tailwind soup; complex composed styles go to a sibling
  `.css` file per `web/src/styles/*.css`.
- Bilingual (`navigation.receivers`, `receivers.*` namespace).

### 6.4 Constants

Append to `web/src/lib/constants.ts`:

```ts
RECEIVERS: {
  BASE: "/api/admin/receivers",
  BY_ID: (id: string) => `/api/admin/receivers/${id}`,
  APPROVE: (id: string) => `/api/admin/receivers/${id}/approve`,
  REJECT: (id: string) => `/api/admin/receivers/${id}/reject`,
  RF_CODE: (id: string) => `/api/admin/receivers/${id}/rf-code`,
  DEACTIVATION: (id: string) => `/api/admin/receivers/${id}/deactivation`,
  REPROVISION: (id: string) => `/api/admin/receivers/${id}/reprovision`,
  RF_CODE_HISTORY: (id: string) => `/api/admin/receivers/${id}/rf-codes`,
  DEACT_HISTORY:  (id: string) => `/api/admin/receivers/${id}/deactivations`,
  TOKENS: "/api/admin/receivers/enrollment-tokens",
  TOKEN_BY_HASH: (hash: string) => `/api/admin/receivers/enrollment-tokens/${hash}`,
},
```

### 6.5 i18n keys

Append to `web/src/messages/en.json` and `vi.json` (pilot keys; more
will emerge during build):

```jsonc
"navigation": { "receivers": "Receivers" /* VI: "Máy thu" */ },
"receivers": {
  "title": "Receivers",
  "pendingTab": "Pending review",
  "tokensTab": "Enrollment tokens",
  "listTitle": "All receivers",
  "emptyState": "No receivers yet.",
  "filterStatus": "Status",
  "filterType": "Radio",
  "filterModel": "Hardware",
  "filterStore": "Store",
  "status": {
    "PENDING": "Pending",
    "ACTIVE": "Active",
    "SUSPENDED": "Suspended",
    "DECOMMISSIONED": "Decommissioned",
    "REJECTED": "Rejected"
  },
  "radioType": { "RECEIVER_433M": "433 MHz", "RECEIVER_2_4G": "2.4 GHz" },
  "hardwareModel": { "ESP-01": "ESP-01", "ESP32-C3": "ESP32-C3" },
  "pending": {
    "emptyState": "No receivers awaiting review.",
    "fingerprintLabel": "Public key fingerprint",
    "approveAction": "Approve",
    "rejectAction": "Reject",
    "approveDialogTitle": "Approve receiver",
    "assignedNameLabel": "Device name",
    "storeLabel": "Assign to store",
    "rejectDialogTitle": "Reject receiver",
    "reasonLabel": "Reason (optional)"
  },
  "rfCode": {
    "sectionTitle": "RF trigger code",
    "hexLabel": "Code (hex)",
    "bitsLabel": "Width (bits)",
    "rotateAction": "Rotate code",
    "rotatedToast": "Code rotated — waiting for device ack",
    "ackPending": "Waiting for device ack",
    "ackApplied": "Applied v{version}",
    "ackUnchanged": "Unchanged v{version}",
    "ackRejected": "Device rejected code v{version}",
    "currentVersion": "Current version: {version}",
    "historyTitle": "Rotation history"
  },
  "deact": {
    "sectionTitle": "Lifecycle",
    "suspendAction": "Suspend",
    "resumeAction": "Resume",
    "decommissionAction": "Decommission",
    "decommissionWarning": "Decommission is permanent — the device can only come back via factory reset + re-provisioning.",
    "historyTitle": "Command history"
  },
  "tokens": {
    "issueAction": "Issue enrollment token",
    "issueDialogTitle": "New enrollment token",
    "plaintextWarning": "Copy this token now — it will not be shown again.",
    "copied": "Copied",
    "ttlHint": "Expires in 1 hour",
    "revokeAction": "Revoke",
    "revokedToast": "Token revoked"
  }
}
```

Vietnamese translations follow the user's established natural/concise
style (see memory `feedback_translation_review.md`).

### 6.6 Live-update strategy (ack status)

Receiver acks arrive on the backend seconds after a command is
published. For v1, the detail page **polls**
`GET /api/admin/receivers/{id}` every ~3 s while `last_ack_status ==
"pending"` — implemented via `use-receiver-ack-poll.ts`. SSE bridging
(similar to `use-queue-events.ts`) is deferred to v2; there is no
per-ticket fanout cost here so polling is fine.

---

## 7. Testing & verification

Backend (placeholders exist but no real tests yet — plan adds the
first meaningful ones):

- **Unit**: `ReceiverCanonical` — golden canonical strings for
  `activate-v1`, `rf-code-v1`, `deact-v1` matching the exact byte
  layout from `§3.3` / `§E.3`.
- **Unit**: `ReceiverCommandSigner` — round-trip sign/verify; reject
  malformed sig.
- **Unit**: RF-code width validator — all §2.4 boundary cases.
- **Integration** (opt-in via Testcontainers Redis): Enrollment
  token issue/consume/revoke atomicity, `GETDEL` semantics.

Admin-web:

- Lint pass (`yarn lint` → biome) for the new feature folder.
- Manual smoke: pending review → approve → see status flip, rotate
  code → see pending → (mock ack) → see applied.

Per CLAUDE.md: **do not build after implementation**; audits run in
their own flow.

---

## 8. Documentation & changelog

Append to `docs/CHANGELOGS.md` under the landing date, listing every
file touched — including skipped items per the project rule. Link
back to this plan from the entry.

Update `docs/planned/Backend Remaining Plan.md` once Phase R0–R8
ship: move the P1 row from "Priority Backlog" to "Completed".

---

## 9. Suggested execution order

1. Phase R0 — schema + signing infra + canonical helpers (no
   behavior visible yet). Audit.
2. Phase R1 + R7 partial — enrollment tokens end-to-end (backend
   service + admin-web token dialog). Audit.
3. Phase R2 + R3 + R4 — bootstrap intake → admin review → activation.
   Admin-web: pending review card. Audit.
4. Phase R5 — RF code rotation + ack handling. Admin-web: RF code
   editor + history. Audit.
5. Phase R6 — deactivation + ack handling. Admin-web: lifecycle
   panel. Audit.
6. Phase R7 remaining — list, filters, audit tabs, reprovision
   hygiene. Final audit.

Each phase should land as its own small slice: easier to audit and
to bisect if a firmware regression surfaces.

---

## 10. Deferred (not in v1)

- Per-device MQTT broker credentials / ACLs (`§15` / `§H.9`) — the
  managed broker lacks the capability; shared creds + `public_id`
  scoping + signed commands is the v1 protection model.
- SSE push for ack events — v1 polls the detail page.
- Migrating `notifier_device` off FCM-specific columns — tracked in
  memory as a separate P2; receiver domain uses `receiver_device` so
  the rename can happen independently.
- In-field rotation of the backend command-signing key — requires a
  coordinated firmware + backend release (Kconfig pin changes). v1
  treats it as long-lived.
- Analytics wiring for `DEVICE_TRIGGERED` — the `analytics_event.device_id`
  column already exists; populating it requires a bridge from
  receiver acks to ticket events, which is a separate feature.
