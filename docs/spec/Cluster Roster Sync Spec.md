# Cluster Roster Sync Spec

Enable redundant transmitter hubs to share their paired-receiver roster (including RF codes) via MQTT retained messages so that a standby hub can dispatch to all receivers after failover, and a returning hub can catch up with roster changes made while it was offline.

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| RF code sharing | Hub-to-hub via MQTT retained message | Codes stay off the backend DB. All hubs with the same PSK can decrypt. |
| Payload encryption | AES-256-GCM with key derived from build-time PSK | Keeps RF codes opaque to the MQTT broker. TLS protects the wire; AES protects at rest on the broker. |
| Sync trigger | On MQTT connect + on first dispatch receipt (implicit election) | Covers cold start and hot failover. No explicit election notification exists. |
| Conflict resolution | Slot-level union merge; higher `seq` determines overwrite direction for conflicting slots | Preserves entries from both hubs. Converges naturally after a few rounds. |
| Cluster identifier | `store_id` provisioned during activation | Semantically correct, debuggable topics, decoupled from PSK. |

## 0. Prerequisites

### 0.1 Backend: Include `store_id` in Activation Response

The backend's bootstrap activation response currently sends `public_id` and `device_name`. This spec requires it to also send `store_id` (the UUID of the store the hub is assigned to).

**Backend change:** In the bootstrap activation handler, include `store_id` in the activation ACK payload alongside `public_id` and `device_name`.

**Transmitter change:** Add `store_id` field to `device_config_t` (max 40 chars to accommodate UUID format `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`). Persist in NVS during `device_config_commit_activation()`. Add `has_store_id` flag.

### 0.2 Fix: `roster_set_name()` Must Bump Seq

Currently `roster_set_name()` in `roster.c` does not increment `seq` or set `pending = true`. This means label changes never trigger backend or cluster sync. This must be fixed as a prerequisite — `roster_set_name()` should bump `seq` and set `pending = true`, matching the pattern used by `roster_add()` and `roster_remove()`.

### 0.3 MQTT Buffer Sizing

The existing MQTT client uses a 2048-byte buffer and a 4096-byte fragment reassembly limit in `mqtt.c`. A full 32-receiver roster with typical names produces ~4.5 KB of JSON plus 29 bytes of encryption overhead. With max-length names (96 chars), it reaches ~7.4 KB.

**Required change:** Increase `.buffer.size` to 8192 and raise the fragment reassembly limit in `handle_mqtt_data_event` from 4096 to 8192 bytes. This adds ~6 KB of RAM usage on ESP32-C3 (which has ~400 KB total). This is acceptable given the feature's importance.

## 1. Cluster Topic

### 1.1 Topic Path

```
{prefix}/transmitter/cluster/{storeId}/roster
```

Where:
- `{prefix}` is `CONFIG_TRANSMITTER_MQTT_TOPIC_PREFIX` (default `"notiguide"`)
- `{storeId}` is the store UUID provisioned during activation (section 0.1)

Example: `notiguide/transmitter/cluster/550e8400-e29b-41d4-a716-446655440000/roster`

### 1.2 Why Store ID

The store ID is the natural cluster identifier — all hubs in the same store share the same receivers. Using it directly produces readable, debuggable MQTT topics that align with the backend's existing `store/{storeId}/queue` pattern. It also decouples the cluster identity from the PSK, so PSK rotation does not change the topic.

## 2. Retained Roster Message

### 2.1 Publish Trigger

The active hub publishes the retained cluster roster message after every successful roster mutation:
- After `roster_add()` (new pairing)
- After `roster_remove()` (local delete, serial delete, MQTT `cmd/unpair`)
- After `roster_set_name()` (MQTT `cmd/label`) — requires prerequisite fix 0.2

Additionally, `roster_cluster_publish()` is called on MQTT connect (alongside the existing `roster_sync_publish()` call in `mqtt_subscribe_current_phase`), **but only when `roster.seq > 0`**. This ensures:
- After broker restart, a hub with a populated roster re-publishes immediately on reconnect.
- A fresh hub (first boot, seq=0, empty roster) does **not** publish an empty roster that would overwrite a valid retained message on the broker. It subscribes, receives the retained message, merges, and only then has something to publish on its next mutation.

### 2.2 Payload Structure (Before Encryption)

```json
{
  "schema_version": 1,
  "seq": 5,
  "publisher_id": "hub-abc123",
  "published_at_ms": 1717000000000,
  "receivers": [
    {
      "slot": 1,
      "mac": "AA:BB:CC:DD:EE:FF",
      "band": 0,
      "rf_code_hex": "A1B2C3D4",
      "rf_code_bits": 24,
      "name": "RX-Kitchen",
      "paired_at_ms": 1716900000000
    }
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `schema_version` | int | Always 1. Hubs accept messages with `schema_version` <= their own. Hubs silently discard messages with `schema_version` > their own. Firmware upgrades should be applied to all hubs simultaneously. |
| `seq` | uint32 | Roster sequence number. Higher wins in merge conflicts. |
| `publisher_id` | string | `public_id` of the publishing hub. Used for self-publish detection and as tiebreaker for equal seq. |
| `published_at_ms` | int64 | Epoch milliseconds when published. Informational. |
| `receivers` | array | All occupied roster entries with full RF code data. Empty array when roster is empty (seq still meaningful). |
| `receivers[].slot` | uint8 | 1-based slot number. |
| `receivers[].mac` | string | Receiver WiFi MAC (colon-separated uppercase hex). |
| `receivers[].band` | uint8 | 0 = 433MHz, 1 = 2.4GHz. |
| `receivers[].rf_code_hex` | string | RF code bytes as uppercase hex string. |
| `receivers[].rf_code_bits` | uint8 | Bit width of the RF code (433M: 1-32, 2.4G: 40). |
| `receivers[].name` | string | Human-readable label (max 96 chars). |
| `receivers[].paired_at_ms` | int64 | Pairing timestamp in epoch milliseconds. |

### 2.3 Encryption

The plaintext JSON is encrypted with **AES-256-GCM** before publishing.

**Key derivation** (HKDF-SHA256 pattern for proper domain separation):

```
ikm   = PSK (32 bytes, hex-decoded from CONFIG_TRANSMITTER_PAIR_PSK)
salt  = (none / zero-length)
info  = "notiguide-cluster-roster-v1" (27 bytes, UTF-8)

AES_KEY = HKDF-Expand-SHA256(ikm, info, 32)
```

Use PSA Crypto `psa_key_derivation` with `PSA_ALG_HKDF(PSA_ALG_SHA_256)`, which is already available in the project via mbedtls. This avoids bare concatenation and follows NIST SP 800-108 best practices.

**Encryption parameters:**
- Algorithm: AES-256-GCM (authenticated encryption)
- IV/Nonce: 12 bytes, randomly generated per publish via `esp_fill_random()`. On ESP32-C3, this is backed by the hardware RNG when WiFi is active (thermal noise source), making it cryptographically suitable. With 12-byte random nonces, the birthday bound is ~2^48 messages — well beyond the expected lifetime publish count.
- AAD (Additional Authenticated Data): the store ID string (defense-in-depth against topic confusion)
- Tag length: 16 bytes (128 bits)

**Wire format** (published as binary):

```
[1 byte: schema version = 0x01]
[12 bytes: IV/nonce]
[N bytes: ciphertext (encrypted JSON)]
[16 bytes: GCM authentication tag]
```

Total overhead: 29 bytes (1 + 12 + 16) on top of the JSON payload.

**MQTT publish parameters:**
- Topic: `{prefix}/transmitter/cluster/{storeId}/roster`
- QoS: 1 (at-least-once, ensures broker stores the retained message)
- Retain: `true`
- MQTT v5 properties (set via `esp_mqtt5_client_set_publish_property()` before each `esp_mqtt_client_publish()` call — the ESP-MQTT library does not store v5 properties between calls):
  - `messageExpiryInterval`: configurable via `CONFIG_TRANSMITTER_CLUSTER_ROSTER_EXPIRY_S` (default 604800 = 7 days). See section 6 for expiry considerations.
  - `contentType`: `"application/octet-stream"`

### 2.4 Decryption

Receiving hubs:
1. Check byte 0 for schema version. Accept if <= own version. Discard if > own version.
2. Extract the 12-byte IV from bytes 1-12.
3. Extract the ciphertext from bytes 13 to `(len - 16)`.
4. Extract the 16-byte GCM tag from the last 16 bytes.
5. Decrypt using AES-256-GCM with the HKDF-derived key, the extracted IV, and the store ID as AAD.
6. If decryption fails (GCM tag mismatch), log an error and discard. This means the message was published by a hub with a different PSK, is corrupted, or was tampered with.
7. Parse the decrypted JSON.

## 3. Sync Triggers

### 3.1 On MQTT Connect

When the hub connects to the MQTT broker, it subscribes to the cluster roster topic. The broker delivers the retained message (if one exists). The hub processes it via the merge logic in Section 4.

**MQTT v5 subscribe options** (set via `esp_mqtt5_client_set_subscribe_property()` before the subscribe call):
- `rh` (Retain Handling): `0` — send retained on every subscribe. This is also the MQTT v5 default, but set it explicitly to be safe across broker implementations.
- `nl` (No Local): `0` — per MQTT v5 spec section 3.8.3.1, No Local does not suppress delivery of retained messages. A hub's own retained publish will be delivered back on resubscribe. The `publisher_id` + `seq` comparison handles this (see 4.2).

After subscribing, the hub calls `roster_cluster_publish()` **only if `roster.seq > 0`** (i.e., the hub has a non-empty roster from NVS). This covers the broker-restart case (hub re-publishes its existing roster). A first-boot hub with an empty roster (seq=0) does **not** publish — it waits for the retained message to arrive and merge, preventing an empty publish from overwriting a valid retained roster on the broker.

### 3.2 On First Dispatch Receipt (Safety Net)

When a hub receives a `cmd/transmit` command for a slot it doesn't have in its local roster, it rejects with `slot_not_found`. This is the graceful degradation path — the admin is notified via the backend's dispatch failure event and can re-pair if necessary.

No additional sync action is needed here. The retained message delivery from 3.1 should have already populated the roster by the time dispatches arrive.

## 4. Merge Logic

### 4.1 Slot-Level Union Merge

When a hub receives a cluster roster message, it performs a **union merge by slot** — the result is the superset of both rosters, with per-slot conflict resolution based on `seq`.

```
incoming = decrypt_and_parse(message)

if incoming.publisher_id == local.public_id
   && incoming.seq == local.seq:
    skip (self-publish echo, no new information)
    return

changed = false

// Phase 1: Adopt incoming entries into local roster
for each incoming_entry in incoming.receivers:
    local_entry = roster_find_slot(local, incoming_entry.slot)

    if local_entry is NULL (slot empty):
        // New entry from another hub — adopt it
        roster_add(local, incoming_entry)
        changed = true

    elif incoming.seq > local.seq:
        // Incoming roster is newer overall — overwrite conflicting slot
        roster_remove(local, incoming_entry.slot)
        roster_add(local, incoming_entry)
        changed = true

    else:
        // Equal or lower seq — keep local entry for this slot
        // (local is same age or newer)

// Phase 2: Handle deletions (entries in local but not in incoming)
if incoming.seq > local.seq:
    for each local_entry in local.entries:
        if local_entry.occupied
           && not in incoming.receivers (by slot):
            // Entry was deleted on the publishing hub
            roster_remove(local, local_entry.slot)
            changed = true

// Phase 3: Update seq and re-publish if changed
if changed:
    local.seq = max(local.seq, incoming.seq) + 1
    persist seq to NVS
    roster_cluster_publish()   // propagate merged state
```

**Why this converges:** After Hub A merges Hub B's entries and re-publishes, Hub B receives the merged roster. Hub B runs the same logic — incoming entries that B already has are skipped (slot occupied, seq not higher). Entries B is missing (originally from A) are adopted. After at most 2 rounds, both hubs have the same superset.

### 4.2 Self-Publish Detection

When `publisher_id` matches the hub's own `public_id`:

- `incoming.seq == local.seq`: skip entirely (echo of own publish, no new data).
- `incoming.seq > local.seq`: legitimate crash recovery — the hub published and crashed before persisting the new seq to NVS. Run the normal merge logic.
- `incoming.seq < local.seq`: the hub has made changes since its last publish. Skip the incoming message. The hub's next mutation will trigger a fresh publish.

### 4.3 Concurrent Pairing Example

Hub A has `{slot1: R1, slot3: R3}` at seq=5. Hub B has `{slot2: R4, slot4: R5}` at seq=5.

1. Hub A publishes its roster (seq=5, publisher="A").
2. Hub B publishes its roster (seq=5, publisher="B").
3. Hub A receives Hub B's message (seq=5 == local seq=5):
   - slot2 (R4): empty locally → adopt. `changed = true`.
   - slot4 (R5): empty locally → adopt. `changed = true`.
   - Phase 2: `incoming.seq == local.seq` → skip deletions.
   - Hub A now has `{R1, R4, R3, R5}`, publishes with seq=6.
4. Hub B receives Hub A's original message (seq=5 == local seq=5):
   - slot1 (R1): empty locally → adopt. `changed = true`.
   - slot3 (R3): empty locally → adopt. `changed = true`.
   - Hub B now has `{R1, R4, R3, R5}`, publishes with seq=6.
5. Hub A receives Hub B's seq=6 message: all slots occupied with same data → no changes. Done.

Both hubs converged to the same roster in 2 rounds.

### 4.4 Slot Conflict Example

Hub A paired receiver X to slot 3 (seq=6). Hub B independently paired receiver Y to slot 3 (seq=6).

1. Hub A receives Hub B's roster (seq=6 == local seq=6):
   - slot3: occupied locally → `incoming.seq == local.seq` → keep local entry (no overwrite for equal seq).
   - Other slots from B that are empty locally → adopted.
   - Hub A re-publishes with seq=7.
2. Hub B receives Hub A's roster (seq=6):
   - Same logic — B keeps its slot3, adopts A's other slots, publishes seq=7.
3. Hub A receives B's seq=7 (> local seq=7? No, A also published seq=7).
   - seq=7 == seq=7 → no overwrite for slot3.
   - At this point, both hubs have the same entries except for slot3, where each keeps their own version.

**This is the expected outcome for a true slot conflict.** Both receivers are physically paired to the same slot number on different hubs, which means they received different RF codes. Neither hub can dispatch to the other's receiver at that slot. The admin must resolve this by re-pairing one of the receivers to a different slot. The system does not silently lose either entry.

### 4.5 Empty Roster / Full Deletion

If all receivers on the active hub are deleted, the hub publishes a retained message with an empty `receivers` array and the current `seq`. Standby hubs process this:

- Phase 1: no incoming entries to adopt (empty array).
- Phase 2: `incoming.seq > local.seq` → remove local entries not in incoming (all of them).
- Result: standby roster is cleared.

The retained message is NOT cleared with an empty-payload publish. The seq and empty-receivers array are preserved so future merges work correctly.

## 5. Implementation Scope

### 5.1 New Transmitter Module: `roster_cluster.c` / `roster_cluster.h`

A new module in `transmitter/main/pair/` responsible for:
- Deriving the AES-256-GCM key from the PSK via HKDF
- Encrypting and publishing the retained roster message
- Subscribing to the cluster topic
- Decrypting and merging incoming roster messages
- Integrating with the existing `roster_sync` module (both publish independently)

The module decodes `CONFIG_TRANSMITTER_PAIR_PSK` independently using the same `hex_to_bytes` pattern as `espnow_pair_host.c`. The PSK is not shared via module state — each module decodes it from the compile-time constant. Consider extracting `hex_to_bytes` into a shared utility if it doesn't already exist as one.

### 5.2 Public API

```c
esp_err_t roster_cluster_init(roster_t *roster);
esp_err_t roster_cluster_publish(void);
void roster_cluster_handle_message(const uint8_t *data, size_t len);
void roster_cluster_deinit(void);
```

All functions gated behind `#if CONFIG_TRANSMITTER_PAIRING_ENABLED`.

### 5.3 MQTT Integration

In `mqtt.c`:

**Subscription:** Add the cluster roster topic to the operational subscribe phase, alongside the existing `roster/ack`, `cmd/label`, `cmd/unpair` subscriptions. Set v5 subscribe properties (`rh = 0`) via `esp_mqtt5_client_set_subscribe_property()` before the subscribe call.

**Routing:** In `route_message()`, add a cluster topic match **before** the hub-specific topic matches (the cluster topic pattern `transmitter/cluster/{storeId}/roster` does not contain a `public_id` and must not be inside the `cfg->has_public_id` guard). Route to `roster_cluster_handle_message((const uint8_t *)payload, payload_len)` — note this handler receives binary data, not JSON. The existing `payload_len` parameter in `route_message` must be used; the null-terminated `char *` representation is not sufficient for binary.

**Publish:** The cluster module needs to set v5 publish properties (`messageExpiryInterval`, `contentType`) via `esp_mqtt5_client_set_publish_property()` before each publish. Since `s_client` is currently `static` in `mqtt.c`, expose a new function:

```c
int mqtt_publish_retained_v5(const char *topic, const uint8_t *data,
                             size_t len, uint32_t expiry_seconds);
```

This wraps `esp_mqtt5_client_set_publish_property()` + `esp_mqtt_client_publish()` with `retain = 1`.

**Connect publish:** Call `roster_cluster_publish()` alongside `roster_sync_publish()` in `mqtt_subscribe_current_phase()`, guarded by `roster.seq > 0`. A fresh hub (seq=0) must not publish an empty roster that would overwrite a valid retained message on the broker. It subscribes first, merges the retained message, and publishes only after its first mutation or after adopting entries from the cluster.

### 5.4 Publish Call Sites

`roster_cluster_publish()` is called alongside `roster_sync_publish()` at every roster mutation site:
- After successful pairing in `espnow_pair_host.c` (`pair_complete_cb`)
- After MQTT `cmd/unpair` in `roster_sync.c` (`roster_sync_handle_unpair`)
- After MQTT `cmd/label` in `roster_sync.c` (`roster_sync_handle_label`)
- After OLED delete in `display.c` (`display_delete_mode_confirm`) — from the Transmitter Roster Delete spec, not yet implemented
- After serial unpair in `serial_protocol.c` (`handle_roster_unpair`) — from the Transmitter Roster Delete spec, not yet implemented
- On MQTT connect in `mqtt.c` (`mqtt_subscribe_current_phase`)

### 5.5 Kconfig

New entry:

```
config TRANSMITTER_CLUSTER_ROSTER_EXPIRY_S
    int "Cluster roster MQTT retain expiry (seconds)"
    default 604800
    depends on TRANSMITTER_PAIRING_ENABLED
    help
        Time in seconds before the retained cluster roster message
        expires on the broker. Default 604800 = 7 days. Set to 0
        to disable expiry (message persists indefinitely).
```

### 5.6 Backend Changes

**Activation response:** Add `store_id` field to the bootstrap activation ACK (see section 0.1). This is the only backend change. The backend does not subscribe to or process the cluster roster topic.

## 6. Security Considerations

| Concern | Mitigation |
|---------|------------|
| RF codes visible to MQTT broker | AES-256-GCM encryption. Broker stores only ciphertext. |
| RF codes visible on the wire | MQTTS (TLS) encrypts the transport. AES encrypts the payload. Double protection. |
| Key compromise via PSK reuse | AES key derived via HKDF with domain-separated info string, distinct from HMAC key, PMK, and LMK. |
| Nonce reuse (AES-GCM) | 12-byte random IV per publish via `esp_fill_random()` (HW RNG with WiFi active). Birthday bound ~2^48 messages. |
| Replay of stale roster | Seq comparison + tiebreaker ensures only newer rosters are adopted. |
| Tampered ciphertext | GCM authentication tag detects tampering. Decryption fails and message is discarded. |
| Cross-store injection | Different PSKs → different AES keys. Decryption fails. Store ID as AAD provides defense-in-depth. |
| Retained message expiry | Configurable via Kconfig (default 7 days). Hubs re-publish on every connect and every mutation. NVS is the fallback if the broker has no retained message. |

**Known limitation:** If all hubs in a cluster are offline for longer than the expiry interval, the retained message expires on the broker. A surviving hub uses its NVS roster as fallback but may be missing changes from other hubs that were online during its downtime. This is an accepted limitation — the admin can re-pair if necessary. Operators can increase the expiry via Kconfig.

## 7. MQTT v5 Behavior Reference

| MQTT v5 Feature | Usage in This Spec |
|-----------------|--------------------|
| Retained messages | Core mechanism. Broker stores one message per topic; delivered to new subscribers immediately. A new retained publish replaces the previous one. |
| `messageExpiryInterval` | Configurable (default 604800s / 7 days). Broker discards the retained message after expiry. |
| `contentType` | Set to `"application/octet-stream"` (binary payload). |
| Retain Handling (`rh`) | Subscriber uses `rh = 0` (send retained on every subscribe). Set explicitly via `esp_mqtt5_client_set_subscribe_property()`. |
| No Local (`nl`) | Not used. Per MQTT v5 spec section 3.8.3.1, No Local does not suppress retained messages. Self-publish detection uses `publisher_id` + `seq` instead. |
| QoS 1 | Retained publish uses QoS 1 for at-least-once delivery guarantee. |

## 8. Scope Boundaries

**In scope:**
- `roster_cluster.c` / `roster_cluster.h` module (encrypt, publish, subscribe, decrypt, merge)
- AES-256-GCM encryption with HKDF-derived key
- MQTT retained message publish with v5 properties
- Merge logic with seq-based conflict resolution + tiebreaker + local-entry protection
- MQTT buffer sizing increase
- `mqtt_publish_retained_v5()` function in `mqtt.c`
- Cluster topic subscription and routing in `mqtt.c`
- `store_id` in backend activation response and `device_config_t`
- `roster_set_name()` seq bump fix
- Kconfig for expiry interval

**Out of scope:**
- Backend subscribing to the cluster roster topic (hub-to-hub only)
- Frontend changes (cluster sync is transparent to the web dashboard)
- Hub-to-hub direct communication (ESP-NOW between hubs)
- Explicit election notification (election remains implicit via dispatch receipt)
- Roster backup/restore via serial USB
