# Cluster Roster Sync Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enable redundant transmitter hubs to share their paired-receiver roster (including RF codes) via AES-256-GCM encrypted MQTT retained messages, so a standby hub can dispatch to all receivers after failover.

**Architecture:** The active hub publishes an encrypted retained message to a cluster-scoped MQTT topic (`transmitter/cluster/{storeId}/roster`) on every roster mutation. Standby hubs subscribe and receive the retained message on connect. A slot-level union merge with seq-based conflict resolution ensures convergence. The backend only changes to include `store_id` in the activation response — it never sees the cluster roster topic. Existing activated hubs do not already have `store_id` in transmitter NVS; they must be reactivated/reprovisioned or covered by a separate backfill command before cluster sync can be considered enabled for a redundant hub pair.

**Tech Stack:** ESP-IDF v6.0 (C, mbedtls AES-GCM, cJSON, ESP-MQTT v5), Kotlin/Spring Boot (backend activation response)

**Spec:** `docs/spec/Cluster Roster Sync Spec.md`

---

### Task 1: Fix `roster_set_name()` to Bump Seq (Prerequisite)

**Files:**
- Modify: `transmitter/main/pair/roster.c:216-243`

Currently `roster_set_name()` updates the name in NVS but does not increment `seq` or set `pending = true`, meaning label changes never trigger sync.

- [ ] **Step 1: Add seq bump and pending flag to `roster_set_name()`**

Replace the current `roster_set_name` function body (lines 216-243 in `roster.c`). After the existing `nvs_set_str` + `nvs_commit` block, add seq increment and pending flag persistence — following the same pattern used by `roster_add()` and `roster_remove()`:

```c
esp_err_t roster_set_name(roster_t *r, uint8_t slot, const char *name)
{
    ESP_RETURN_ON_FALSE(r != NULL && name != NULL, ESP_ERR_INVALID_ARG, TAG, "args");
    if (slot < 1 || slot > CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS) {
        return ESP_ERR_INVALID_ARG;
    }

    roster_entry_t *e = &r->entries[slot - 1];
    if (!e->occupied) {
        return ESP_ERR_NOT_FOUND;
    }

    const uint32_t new_seq = r->seq + 1U;

    nvs_handle_t h;
    ESP_RETURN_ON_ERROR(nvs_open(ROSTER_NAMESPACE, NVS_READWRITE, &h), TAG, "open");

    char key[16];
    snprintf(key, sizeof(key), "rx_%u_name", slot);
    esp_err_t err = nvs_set_str(h, key, name);
    if (err == ESP_OK) {
        err = nvs_set_u32(h, "roster_seq", new_seq);
    }
    if (err == ESP_OK) {
        err = nvs_set_u8(h, "roster_pend", 1);
    }
    if (err == ESP_OK) {
        err = nvs_commit(h);
    }
    nvs_close(h);
    ESP_RETURN_ON_ERROR(err, TAG, "rename");

    strlcpy(e->name, name, sizeof(e->name));
    r->seq = new_seq;
    r->pending = true;
    ESP_LOGI(TAG, "Renamed receiver: slot=%u seq=%lu", slot, (unsigned long)new_seq);
    return ESP_OK;
}
```

---

### Task 2: Add `store_id` to Device Config (Transmitter)

**Files:**
- Modify: `transmitter/main/config/device_config.h`
- Modify: `transmitter/main/config/device_config.c`

- [ ] **Step 1: Add `store_id` field to `device_config_t`**

In `device_config.h`, add the max length constant after `DEVICE_CFG_DEVICE_NAME_MAX_LEN`:

```c
#define DEVICE_CFG_STORE_ID_MAX_LEN          40U
```

In the `device_config_t` struct, add after the `device_name` field:

```c
    char store_id[DEVICE_CFG_STORE_ID_MAX_LEN];
```

Add the `has_store_id` flag alongside the other `has_*` flags:

```c
    bool has_store_id;
```

- [ ] **Step 2: Update `device_config_commit_activation()` to accept and persist `store_id`**

Change the function signature in `device_config.h`:

```c
esp_err_t device_config_commit_activation(const char *public_id, const char *device_name, const char *store_id);
```

Add `KEY_STORE_ID` beside the existing NVS key constants:

```c
static const char *KEY_STORE_ID = "store_id";
```

Validate that the activation result includes a non-empty store ID before opening NVS and writing runtime activation state:

```c
    ESP_RETURN_ON_FALSE(store_id != NULL && store_id[0] != '\0',
                        ESP_ERR_INVALID_ARG, TAG, "store_id");
    ESP_RETURN_ON_FALSE(strlen(store_id) < DEVICE_CFG_STORE_ID_MAX_LEN,
                        ESP_ERR_INVALID_ARG, TAG, "store_id len");
```

After `write_string_or_erase(handle, KEY_DEVICE_NAME, device_name, &status);`, add:

```c
    write_string_or_erase(handle, KEY_STORE_ID, store_id, &status);
```

After `s_cfg.has_device_name = true;`, add:

```c
    strlcpy(s_cfg.store_id, store_id, sizeof(s_cfg.store_id));
    s_cfg.has_store_id = true;
```

- [ ] **Step 3: Load `store_id` from NVS in `device_config_load()`**

In the `device_config_load()` function, alongside the existing `nvs_get_str` calls for `public_id` and `device_name`, add:

```c
    load_string_if_present(handle, KEY_STORE_ID, s_cfg.store_id,
                           sizeof(s_cfg.store_id), &s_cfg.has_store_id);
```

In `device_config_save_provisioning()`, add `KEY_STORE_ID` to the `runtime_keys` array so a new provisioning flow clears stale activation state. Also clear the cached field with the other runtime fields:

```c
    s_cfg.store_id[0] = '\0';
    s_cfg.has_store_id = false;
```

- [ ] **Step 4: Update `handle_bootstrap_result()` in `mqtt.c` to extract `store_id`**

In `transmitter/main/network/mqtt.c`, in `handle_bootstrap_result()` (around line 268), add a `store_id` variable and require the field before committing activation:

```c
    char store_id[DEVICE_CFG_STORE_ID_MAX_LEN];
    store_id[0] = '\0';

    if (!json_copy_string(root, "store_id", store_id, sizeof(store_id))) {
        ESP_LOGE(TAG, "Bootstrap result missing store_id");
        cJSON_Delete(root);
        return;
    }

    if (device_config_commit_activation(public_id, assigned_device_name, store_id) != ESP_OK) {
```

- [ ] **Step 5: Roll out existing activated hubs**

Existing activated transmitters will not have `store_id` in NVS. For every redundant hub pair, either reactivate/reprovision the transmitter through the updated activation flow or implement a separate signed backfill command that writes `store_id` using the same NVS key and cache fields above. Do not mark this feature deployed for an existing store until every participating hub reports `has_store_id = true`; otherwise the cluster module will intentionally remain disabled.

---

### Task 3: Add `store_id` to Backend Activation Response

**Files:**
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/core/device/DeviceMqttPublisher.kt`
- Modify: `backend/src/main/kotlin/com/thomas/notiguide/domain/device/service/DeviceActivationService.kt`

- [ ] **Step 1: Add `storeId` to `ResultEnvelope`**

In `DeviceMqttPublisher.kt`, update the `ResultEnvelope` data class (line 257):

```kotlin
    private data class ResultEnvelope(
        @field:JsonProperty("schema_version")
        val schemaVersion: Int = 1,
        val type: String = "result",
        @field:JsonProperty("challenge_id")
        val challengeId: UUID,
        val status: String = "active",
        @field:JsonProperty("public_id")
        val publicId: String,
        @field:JsonProperty("assigned_device_name")
        val assignedDeviceName: String,
        @field:JsonProperty("store_id")
        val storeId: String? = null
    )
```

- [ ] **Step 2: Add `storeId` parameter to `publishResult()`**

Update the `publishResult()` function signature (line 84):

```kotlin
    suspend fun publishResult(
        kind: DeviceKind,
        challengeId: UUID,
        publicId: String,
        assignedDeviceName: String,
        storeId: String? = null
    ) {
        publishJson(
            topic = bootstrapTopic(kind, challengeId),
            payload = ResultEnvelope(
                challengeId = challengeId,
                publicId = publicId,
                assignedDeviceName = assignedDeviceName,
                storeId = storeId
            )
        )
    }
```

- [ ] **Step 3: Pass `storeId` from `DeviceActivationService`**

In `DeviceActivationService.kt`, at the `publishResult` call site (line 95), add `storeId`. Transmitter hubs must have a store ID before the activation result is published; receivers can continue to receive `null` if the existing activation contract allows it.

```kotlin
        val activationStoreId = if (saved.kind.isHub()) {
            requireNotNull(saved.storeId) {
                "Activated transmitter hubs must have a store_id for cluster roster sync"
            }.toString()
        } else {
            saved.storeId?.toString()
        }

        deviceMqttPublisher.publishResult(
            kind = saved.kind,
            challengeId = challengeId,
            publicId = newPublicId,
            assignedDeviceName = requireNotNull(saved.assignedName) {
                "Approved devices must have an assigned name before activation"
            },
            storeId = activationStoreId
        )
```

---

### Task 4: Increase MQTT Buffer Size

**Files:**
- Modify: `transmitter/main/network/mqtt.c`

- [ ] **Step 1: Increase buffer size and fragment limit**

In `mqtt.c`, find the MQTT client config (around line 515 where `.buffer.size` is set) and change to 8192:

```c
    .buffer.size = 8192,
```

In `handle_mqtt_data_event()` (line 415), change the fragment limit from 4096 to 8192:

```c
        if (event->total_data_len > 8192) {
```

---

### Task 5: Add Kconfig Entry for Cluster Roster Expiry

**Files:**
- Modify: `transmitter/main/Kconfig.projbuild`

- [ ] **Step 1: Add the Kconfig entry**

In `Kconfig.projbuild`, inside the pairing-enabled section (after `CONFIG_TRANSMITTER_PAIR_TIMEOUT_S` or similar), add:

```
config TRANSMITTER_CLUSTER_ROSTER_EXPIRY_S
    int "Cluster roster MQTT retain expiry (seconds)"
    default 604800
    depends on TRANSMITTER_PAIRING_ENABLED
    help
        Time in seconds before the retained cluster roster message
        expires on the MQTT broker. Default 604800 = 7 days. Set to
        0 to disable expiry (message persists indefinitely on broker).
```

---

### Task 6: Add `mqtt_publish_retained_v5()` to MQTT Module

**Files:**
- Modify: `transmitter/main/network/mqtt.h`
- Modify: `transmitter/main/network/mqtt.c`

The cluster module needs to set MQTT v5 properties (`messageExpiryInterval`, `contentType`) before publishing. The existing `mqtt_publish()` wrapper doesn't support v5 properties. This task adds a new function that does.

- [ ] **Step 1: Declare the function in `mqtt.h`**

Add after `mqtt_publish()`:

```c
/**
 * @brief Publish a binary retained message with MQTT v5 properties.
 *
 * Sets message expiry and content type before publishing with retain=1.
 *
 * @param topic Topic to publish to.
 * @param data Binary payload.
 * @param len Payload length in bytes.
 * @param expiry_seconds Message expiry interval (0 = no expiry).
 * @return MQTT message ID on success, or a negative ESP-MQTT error value.
 */
int mqtt_publish_retained_v5(const char *topic, const void *data, int len, uint32_t expiry_seconds);
```

- [ ] **Step 2: Implement in `mqtt.c`**

Add after the existing `mqtt_publish()` function. This requires the `mqtt5_client.h` include:

```c
#include "mqtt5_client.h"
```

Add at the top of `mqtt.c` alongside existing includes. Then implement:

```c
int mqtt_publish_retained_v5(const char *topic, const void *data, int len, uint32_t expiry_seconds)
{
    if (s_client == NULL) {
        return -1;
    }

    esp_mqtt5_publish_property_config_t prop = {
        .message_expiry_interval = expiry_seconds,
        .content_type = "application/octet-stream",
    };
    esp_err_t err = esp_mqtt5_client_set_publish_property(s_client, &prop);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "set_publish_property: %s", esp_err_to_name(err));
        return -1;
    }

    return esp_mqtt_client_publish(s_client, topic, (const char *)data, len, 1, 1);
}
```

---

### Task 7: Create `roster_cluster.h` Header

**Files:**
- Create: `transmitter/main/pair/roster_cluster.h`

- [ ] **Step 1: Write the header**

```c
/**
 * @file roster_cluster.h
 * @brief AES-256-GCM encrypted MQTT cluster roster sync between redundant hubs.
 */

#ifndef TRANSMITTER_ROSTER_CLUSTER_H
#define TRANSMITTER_ROSTER_CLUSTER_H

#include "esp_err.h"
#include "pair/roster.h"

#ifdef __cplusplus
extern "C" {
#endif

/**
 * @brief Initialize cluster roster sync.
 *
 * Derives the AES-256-GCM key from the pairing PSK via HKDF, builds the
 * cluster topic from the device store_id, and subscribes to the topic.
 *
 * @param roster Active roster snapshot.
 * @return ESP_OK on success.
 */
esp_err_t roster_cluster_init(roster_t *roster);

/**
 * @brief Publish the current roster as an encrypted retained message.
 *
 * Called after every roster mutation and on MQTT connect (when seq > 0).
 *
 * @return ESP_OK on success, ESP_FAIL on publish failure.
 */
esp_err_t roster_cluster_publish(void);

/**
 * @brief Handle an incoming cluster roster message from the broker.
 *
 * Decrypts, validates schema, and performs slot-level union merge into
 * the local NVS roster if the incoming data adds new information.
 *
 * @param data Raw MQTT payload (encrypted binary).
 * @param len Payload length.
 */
void roster_cluster_handle_message(const uint8_t *data, size_t len);

/**
 * @brief Stop cluster sync and clear module state.
 */
void roster_cluster_deinit(void);

#ifdef __cplusplus
}
#endif

#endif /* TRANSMITTER_ROSTER_CLUSTER_H */
```

---

### Task 8: Implement `roster_cluster.c` — Key Derivation and Init

**Files:**
- Modify: `transmitter/main/CMakeLists.txt`
- Create: `transmitter/main/pair/roster_cluster.c`

This is the core module. Split across Tasks 8-10 for manageability.

- [ ] **Step 0: Register the new source file**

In `transmitter/main/CMakeLists.txt`, add the new implementation file to the pairing-enabled source list that already includes `pair/roster.c` and `pair/roster_sync.c`:

```cmake
    "pair/roster_cluster.c"
```

The module is referenced from MQTT, pairing, and roster-sync call sites in later tasks; omitting the source registration will compile headers successfully but fail at link time.

- [ ] **Step 1: Write the file header, includes, and static state**

```c
#include "pair/roster_cluster.h"

#include <stdbool.h>
#include <stdlib.h>
#include <string.h>
#include <stdio.h>

#include "cJSON.h"
#include "config/device_config.h"
#include "network/mqtt.h"
#include "pair/roster.h"
#include "pair/roster_sync.h"
#include "events/transmitter_events.h"
#include "utils/time_utils.h"

#include "esp_event.h"
#include "esp_log.h"
#include "esp_random.h"
#include "mbedtls/gcm.h"
#include "mbedtls/md.h"
#include "nvs.h"
#include "sdkconfig.h"

#define TOPIC_PREFIX CONFIG_TRANSMITTER_MQTT_TOPIC_PREFIX "/"

static const char *TAG = "roster_cluster";

#define AES_KEY_LEN    32
#define GCM_IV_LEN     12
#define GCM_TAG_LEN    16
#define WIRE_OVERHEAD  (1 + GCM_IV_LEN + GCM_TAG_LEN) /* schema + iv + tag = 29 */
#define SCHEMA_V1      0x01

static roster_t *s_roster;
static uint8_t   s_aes_key[AES_KEY_LEN];
static char      s_cluster_topic[128];
static bool      s_initialized;
```

- [ ] **Step 2: Implement PSK hex decode helper**

```c
static bool hex_byte(char c, uint8_t *out)
{
    if (c >= '0' && c <= '9') { *out = (uint8_t)(c - '0'); return true; }
    if (c >= 'a' && c <= 'f') { *out = (uint8_t)(c - 'a' + 10); return true; }
    if (c >= 'A' && c <= 'F') { *out = (uint8_t)(c - 'A' + 10); return true; }
    return false;
}

static bool hex_decode(const char *hex, uint8_t *out, size_t out_len)
{
    size_t hex_len = strlen(hex);
    if (hex_len != out_len * 2) return false;
    for (size_t i = 0; i < out_len; ++i) {
        uint8_t hi, lo;
        if (!hex_byte(hex[i * 2], &hi) || !hex_byte(hex[i * 2 + 1], &lo)) return false;
        out[i] = (uint8_t)((hi << 4) | lo);
    }
    return true;
}
```

- [ ] **Step 3: Implement HKDF-SHA256 key derivation**

Spec target: PSA Crypto HKDF-SHA256. Use PSA HKDF when the transmitter build enables the required PSA key-derivation support; otherwise document and implement the mbedtls HMAC-SHA256 fallback below so the derived key remains deterministic and compatible across hubs. Do not mix algorithms inside one deployed cluster; all hubs in a cluster must use the same derivation path.

```c
static esp_err_t derive_aes_key(const uint8_t *psk, size_t psk_len)
{
    static const char info[] = "notiguide-cluster-roster-v1";

    /* HKDF-Expand using HMAC-SHA256: PRK = HMAC(salt="", IKM=psk), then expand with info */
    /* Step 1: Extract — PRK = HMAC-SHA256(salt=zeros, IKM=psk) */
    uint8_t prk[32];
    const mbedtls_md_info_t *md = mbedtls_md_info_from_type(MBEDTLS_MD_SHA256);
    if (md == NULL) return ESP_FAIL;

    uint8_t salt[32] = { 0 };
    int ret = mbedtls_md_hmac(md, salt, sizeof(salt), psk, psk_len, prk);
    if (ret != 0) return ESP_FAIL;

    /* Step 2: Expand — T(1) = HMAC-SHA256(PRK, info || 0x01) */
    uint8_t expand_input[sizeof(info) - 1 + 1]; /* info + counter byte */
    memcpy(expand_input, info, sizeof(info) - 1);
    expand_input[sizeof(info) - 1] = 0x01;

    ret = mbedtls_md_hmac(md, prk, sizeof(prk), expand_input, sizeof(expand_input), s_aes_key);
    if (ret != 0) return ESP_FAIL;

    return ESP_OK;
}
```

- [ ] **Step 4: Implement `roster_cluster_init()`**

```c
esp_err_t roster_cluster_init(roster_t *roster)
{
    if (roster == NULL) return ESP_ERR_INVALID_ARG;

    const device_config_t *cfg = device_config_get();
    if (!cfg->has_store_id || cfg->store_id[0] == '\0') {
        ESP_LOGW(TAG, "No store_id — cluster sync disabled");
        return ESP_OK;
    }

    /* Decode PSK */
    uint8_t psk[32];
    if (!hex_decode(CONFIG_TRANSMITTER_PAIR_PSK, psk, sizeof(psk))) {
        ESP_LOGE(TAG, "Invalid PSK hex");
        return ESP_ERR_INVALID_ARG;
    }

    /* Derive AES key */
    esp_err_t err = derive_aes_key(psk, sizeof(psk));
    memset(psk, 0, sizeof(psk)); /* clear PSK from stack */
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "Key derivation failed");
        return err;
    }

    /* Build cluster topic */
    (void)snprintf(s_cluster_topic, sizeof(s_cluster_topic),
                   TOPIC_PREFIX "transmitter/cluster/%s/roster", cfg->store_id);

    s_roster = roster;
    s_initialized = true;

    ESP_LOGI(TAG, "Cluster roster sync initialized (topic: %s)", s_cluster_topic);
    return ESP_OK;
}

void roster_cluster_deinit(void)
{
    memset(s_aes_key, 0, sizeof(s_aes_key));
    s_roster = NULL;
    s_initialized = false;
}
```

`TOPIC_PREFIX` was defined locally in Task 8 Step 1 (same pattern as `roster_sync.c` line 24).

---

### Task 9: Implement `roster_cluster.c` — Encrypt and Publish

**Files:**
- Modify: `transmitter/main/pair/roster_cluster.c` (continue from Task 8)

- [ ] **Step 1: Implement JSON serialization of the roster**

```c
static char *build_roster_json(void)
{
    const device_config_t *cfg = device_config_get();

    cJSON *root = cJSON_CreateObject();
    if (root == NULL) return NULL;

    cJSON_AddNumberToObject(root, "schema_version", 1);
    cJSON_AddNumberToObject(root, "seq", (double)s_roster->seq);
    cJSON_AddStringToObject(root, "publisher_id", cfg->has_public_id ? cfg->public_id : "");

    int64_t now_ms = time_utils_now_epoch_ms();
    cJSON_AddNumberToObject(root, "published_at_ms", (double)now_ms);

    cJSON *receivers = cJSON_AddArrayToObject(root, "receivers");
    if (receivers == NULL) { cJSON_Delete(root); return NULL; }

    for (uint8_t i = 0; i < CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS; ++i) {
        const roster_entry_t *e = &s_roster->entries[i];
        if (!e->occupied) continue;

        cJSON *rx = cJSON_CreateObject();
        cJSON_AddNumberToObject(rx, "slot", e->slot);

        char mac_str[18];
        snprintf(mac_str, sizeof(mac_str), "%02X:%02X:%02X:%02X:%02X:%02X",
                 e->mac[0], e->mac[1], e->mac[2],
                 e->mac[3], e->mac[4], e->mac[5]);
        cJSON_AddStringToObject(rx, "mac", mac_str);
        cJSON_AddNumberToObject(rx, "band", e->band);

        char rf_hex[ROSTER_MAX_CODE_LEN * 2 + 1];
        for (uint8_t j = 0; j < e->rf_code_len; ++j) {
            snprintf(&rf_hex[j * 2], 3, "%02X", e->rf_code[j]);
        }
        rf_hex[e->rf_code_len * 2] = '\0';
        cJSON_AddStringToObject(rx, "rf_code_hex", rf_hex);
        cJSON_AddNumberToObject(rx, "rf_code_bits", e->rf_bits);
        cJSON_AddStringToObject(rx, "name", e->name);
        cJSON_AddNumberToObject(rx, "paired_at_ms", (double)e->paired_at_ms);
        cJSON_AddItemToArray(receivers, rx);
    }

    char *json = cJSON_PrintUnformatted(root);
    cJSON_Delete(root);
    return json;
}
```

- [ ] **Step 2: Implement AES-256-GCM encryption**

```c
static uint8_t *encrypt_payload(const char *json, size_t json_len, size_t *out_len)
{
    const device_config_t *cfg = device_config_get();
    const char *aad = cfg->store_id;
    size_t aad_len = strlen(aad);

    size_t wire_len = 1 + GCM_IV_LEN + json_len + GCM_TAG_LEN;
    uint8_t *wire = malloc(wire_len);
    if (wire == NULL) return NULL;

    /* schema version */
    wire[0] = SCHEMA_V1;

    /* random IV */
    esp_fill_random(&wire[1], GCM_IV_LEN);

    /* encrypt */
    mbedtls_gcm_context gcm;
    mbedtls_gcm_init(&gcm);

    int ret = mbedtls_gcm_setkey(&gcm, MBEDTLS_CIPHER_ID_AES, s_aes_key, AES_KEY_LEN * 8);
    if (ret != 0) {
        ESP_LOGE(TAG, "gcm_setkey: -0x%04x", (unsigned)-ret);
        mbedtls_gcm_free(&gcm);
        free(wire);
        return NULL;
    }

    uint8_t *ciphertext = &wire[1 + GCM_IV_LEN];
    uint8_t *tag = &wire[1 + GCM_IV_LEN + json_len];

    ret = mbedtls_gcm_crypt_and_tag(&gcm, MBEDTLS_GCM_ENCRYPT, json_len,
                                     &wire[1], GCM_IV_LEN,
                                     (const uint8_t *)aad, aad_len,
                                     (const uint8_t *)json, ciphertext,
                                     GCM_TAG_LEN, tag);
    mbedtls_gcm_free(&gcm);

    if (ret != 0) {
        ESP_LOGE(TAG, "gcm_encrypt: -0x%04x", (unsigned)-ret);
        free(wire);
        return NULL;
    }

    *out_len = wire_len;
    return wire;
}
```

- [ ] **Step 3: Implement `roster_cluster_publish()`**

```c
esp_err_t roster_cluster_publish(void)
{
    if (!s_initialized || s_roster == NULL) return ESP_OK;
    if (!mqtt_is_connected()) return ESP_OK;

    char *json = build_roster_json();
    if (json == NULL) {
        ESP_LOGE(TAG, "Failed to build roster JSON");
        return ESP_ERR_NO_MEM;
    }

    size_t json_len = strlen(json);
    size_t wire_len = 0;
    uint8_t *wire = encrypt_payload(json, json_len, &wire_len);
    free(json);

    if (wire == NULL) {
        return ESP_FAIL;
    }

    uint32_t expiry = CONFIG_TRANSMITTER_CLUSTER_ROSTER_EXPIRY_S;
    int msg_id = mqtt_publish_retained_v5(s_cluster_topic, wire, (int)wire_len, expiry);
    free(wire);

    if (msg_id < 0) {
        ESP_LOGE(TAG, "Cluster roster publish failed");
        return ESP_FAIL;
    }

    ESP_LOGI(TAG, "Cluster roster published (seq=%lu, %zu bytes)",
             (unsigned long)s_roster->seq, wire_len);
    return ESP_OK;
}
```

---

### Task 10: Implement `roster_cluster.c` — Decrypt and Union Merge

**Files:**
- Modify: `transmitter/main/pair/roster_cluster.c` (continue)

- [ ] **Step 1: Implement AES-256-GCM decryption**

```c
static char *decrypt_payload(const uint8_t *data, size_t len, size_t *json_len)
{
    if (len < WIRE_OVERHEAD + 1) {
        ESP_LOGW(TAG, "Payload too short (%zu bytes)", len);
        return NULL;
    }

    if (data[0] > SCHEMA_V1) {
        ESP_LOGW(TAG, "Unknown schema version %u, skipping", data[0]);
        return NULL;
    }

    const uint8_t *iv = &data[1];
    size_t ct_len = len - WIRE_OVERHEAD;
    const uint8_t *ciphertext = &data[1 + GCM_IV_LEN];
    const uint8_t *tag = &data[len - GCM_TAG_LEN];

    const device_config_t *cfg = device_config_get();
    const char *aad = cfg->store_id;
    size_t aad_len = strlen(aad);

    char *plaintext = malloc(ct_len + 1);
    if (plaintext == NULL) return NULL;

    mbedtls_gcm_context gcm;
    mbedtls_gcm_init(&gcm);

    int ret = mbedtls_gcm_setkey(&gcm, MBEDTLS_CIPHER_ID_AES, s_aes_key, AES_KEY_LEN * 8);
    if (ret != 0) {
        mbedtls_gcm_free(&gcm);
        free(plaintext);
        return NULL;
    }

    ret = mbedtls_gcm_auth_decrypt(&gcm, ct_len,
                                    iv, GCM_IV_LEN,
                                    (const uint8_t *)aad, aad_len,
                                    tag, GCM_TAG_LEN,
                                    ciphertext, (uint8_t *)plaintext);
    mbedtls_gcm_free(&gcm);

    if (ret != 0) {
        ESP_LOGW(TAG, "GCM auth failed (different PSK or corrupted): -0x%04x", (unsigned)-ret);
        free(plaintext);
        return NULL;
    }

    plaintext[ct_len] = '\0';
    *json_len = ct_len;
    return plaintext;
}
```

- [ ] **Step 2: Implement JSON parsing of incoming roster**

```c
typedef struct {
    uint32_t seq;
    char publisher_id[DEVICE_CFG_PUBLIC_ID_MAX_LEN];
    int64_t published_at_ms;
    roster_entry_t entries[CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS];
    uint8_t count;
} parsed_cluster_roster_t;

static bool parse_mac(const char *str, uint8_t mac[6])
{
    return sscanf(str, "%02hhX:%02hhX:%02hhX:%02hhX:%02hhX:%02hhX",
                  &mac[0], &mac[1], &mac[2], &mac[3], &mac[4], &mac[5]) == 6;
}

static bool parse_cluster_roster(const char *json, parsed_cluster_roster_t *out)
{
    cJSON *root = cJSON_Parse(json);
    if (root == NULL) return false;

    memset(out, 0, sizeof(*out));

    cJSON *sv = cJSON_GetObjectItem(root, "schema_version");
    if (!cJSON_IsNumber(sv) || (int)sv->valuedouble > 1) {
        cJSON_Delete(root);
        return false;
    }

    cJSON *seq = cJSON_GetObjectItem(root, "seq");
    if (cJSON_IsNumber(seq)) out->seq = (uint32_t)seq->valuedouble;

    cJSON *pub_id = cJSON_GetObjectItem(root, "publisher_id");
    if (cJSON_IsString(pub_id)) {
        strlcpy(out->publisher_id, cJSON_GetStringValue(pub_id), sizeof(out->publisher_id));
    }

    cJSON *pub_at = cJSON_GetObjectItem(root, "published_at_ms");
    if (cJSON_IsNumber(pub_at)) out->published_at_ms = (int64_t)pub_at->valuedouble;

    cJSON *receivers = cJSON_GetObjectItem(root, "receivers");
    if (!cJSON_IsArray(receivers)) {
        cJSON_Delete(root);
        return false;
    }

    cJSON *rx = NULL;
    cJSON_ArrayForEach(rx, receivers) {
        if (out->count >= CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS) break;

        roster_entry_t *e = &out->entries[out->count];
        cJSON *slot_item = cJSON_GetObjectItem(rx, "slot");
        if (!cJSON_IsNumber(slot_item)) continue;
        e->slot = (uint8_t)slot_item->valuedouble;
        if (e->slot < 1 || e->slot > CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS) continue;

        cJSON *mac_item = cJSON_GetObjectItem(rx, "mac");
        if (!cJSON_IsString(mac_item) || !parse_mac(cJSON_GetStringValue(mac_item), e->mac)) continue;

        cJSON *band_item = cJSON_GetObjectItem(rx, "band");
        if (!cJSON_IsNumber(band_item)) continue;
        e->band = (uint8_t)band_item->valuedouble;
        if (e->band > 1) continue;

        cJSON *rf_hex = cJSON_GetObjectItem(rx, "rf_code_hex");
        if (!cJSON_IsString(rf_hex)) continue;
        const char *hex_str = cJSON_GetStringValue(rf_hex);
        size_t hex_len = strlen(hex_str);
        if (hex_len == 0 || (hex_len % 2U) != 0 || hex_len > (ROSTER_MAX_CODE_LEN * 2U)) continue;
        e->rf_code_len = (uint8_t)(hex_len / 2U);
        if (!hex_decode(hex_str, e->rf_code, e->rf_code_len)) continue;

        cJSON *bits_item = cJSON_GetObjectItem(rx, "rf_code_bits");
        if (!cJSON_IsNumber(bits_item)) continue;
        e->rf_bits = (uint8_t)bits_item->valuedouble;
        if (e->rf_bits == 0 || e->rf_bits > (e->rf_code_len * 8U)) continue;

        cJSON *name_item = cJSON_GetObjectItem(rx, "name");
        if (cJSON_IsString(name_item)) {
            strlcpy(e->name, cJSON_GetStringValue(name_item), sizeof(e->name));
        }

        cJSON *paired_at = cJSON_GetObjectItem(rx, "paired_at_ms");
        if (cJSON_IsNumber(paired_at)) e->paired_at_ms = (int64_t)paired_at->valuedouble;

        e->occupied = true;
        out->count++;
    }

    cJSON_Delete(root);
    return true;
}
```

- [ ] **Step 3: Implement the slot-level union merge**

```c
static const roster_entry_t *find_in_parsed(const parsed_cluster_roster_t *pr, uint8_t slot)
{
    for (uint8_t i = 0; i < pr->count; ++i) {
        if (pr->entries[i].slot == slot) return &pr->entries[i];
    }
    return NULL;
}

void roster_cluster_handle_message(const uint8_t *data, size_t len)
{
    if (!s_initialized || s_roster == NULL) return;

    size_t json_len = 0;
    char *json = decrypt_payload(data, len, &json_len);
    if (json == NULL) return;

    parsed_cluster_roster_t incoming;
    if (!parse_cluster_roster(json, &incoming)) {
        ESP_LOGW(TAG, "Failed to parse cluster roster JSON");
        free(json);
        return;
    }
    free(json);

    const device_config_t *cfg = device_config_get();

    /* Self-publish detection: same publisher with equal or older seq → no new data.
     * Case: incoming.seq == local.seq → echo of own publish.
     * Case: incoming.seq < local.seq → stale self-publish (hub made changes since).
     * Case: incoming.seq > local.seq → crash recovery, fall through to merge. */
    if (cfg->has_public_id &&
        strcmp(incoming.publisher_id, cfg->public_id) == 0 &&
        incoming.seq <= s_roster->seq) {
        return;
    }

    bool changed = false;
    const uint32_t local_seq_before = s_roster->seq;

    /* Phase 1: Adopt incoming entries into local roster */
    for (uint8_t i = 0; i < incoming.count; ++i) {
        const roster_entry_t *inc = &incoming.entries[i];
        const roster_entry_t *local = roster_find_slot(s_roster, inc->slot);

        if (local == NULL) {
            /* Slot empty locally — adopt incoming entry */
            roster_entry_t entry = *inc;
            if (roster_add(s_roster, &entry) == ESP_OK) {
                changed = true;
            }
        } else if (incoming.seq > local_seq_before) {
            /* Incoming is newer than the pre-merge local snapshot — overwrite conflicting slot */
            roster_remove(s_roster, inc->slot);
            roster_entry_t entry = *inc;
            if (roster_add(s_roster, &entry) == ESP_OK) {
                changed = true;
            }
        }
        /* Equal or lower seq: keep local entry for this slot */
    }

    /* Phase 2: Handle deletions — entries in local but not in incoming */
    if (incoming.seq > local_seq_before) {
        for (uint8_t i = 0; i < CONFIG_TRANSMITTER_PAIR_MAX_RECEIVERS; ++i) {
            const roster_entry_t *local = &s_roster->entries[i];
            if (!local->occupied) continue;
            if (find_in_parsed(&incoming, local->slot) == NULL) {
                roster_remove(s_roster, local->slot);
                changed = true;
            }
        }
    }

    if (changed) {
        ESP_LOGI(TAG, "Cluster merge: adopted entries (incoming seq=%lu, local seq=%lu)",
                 (unsigned long)incoming.seq, (unsigned long)s_roster->seq);

        /* Update local seq to max(original_local, incoming) + 1.
         * Use local_seq_before (snapshot before merge) because roster_add/roster_remove
         * each bump seq internally — we override that inflation here. */
        uint32_t new_seq = (incoming.seq > local_seq_before ? incoming.seq : local_seq_before) + 1;
        s_roster->seq = new_seq;
        s_roster->pending = true;

        /* Persist updated seq */
        nvs_handle_t h;
        if (nvs_open(ROSTER_NAMESPACE, NVS_READWRITE, &h) == ESP_OK) {
            nvs_set_u32(h, "roster_seq", new_seq);
            nvs_set_u8(h, "roster_pend", 1);
            nvs_commit(h);
            nvs_close(h);
        }

        /* Re-publish merged roster and sync with backend */
        (void)roster_cluster_publish();
        (void)roster_sync_publish();
    } else if (incoming.seq > local_seq_before) {
        /* No entries changed but seq is higher — update local seq to stay current */
        s_roster->seq = incoming.seq;
        nvs_handle_t h;
        if (nvs_open(ROSTER_NAMESPACE, NVS_READWRITE, &h) == ESP_OK) {
            nvs_set_u32(h, "roster_seq", incoming.seq);
            nvs_commit(h);
            nvs_close(h);
        }
    }
}
```

The `nvs.h` include was already added in Task 8 Step 1.

---

### Task 11: Wire Cluster Sync into MQTT Module

**Files:**
- Modify: `transmitter/main/network/mqtt.c`

- [ ] **Step 1: Add include and cluster topic subscription**

Add include at the top of `mqtt.c` (inside `#if CONFIG_TRANSMITTER_PAIRING_ENABLED`):

```c
#if CONFIG_TRANSMITTER_PAIRING_ENABLED
#include "pair/roster_cluster.h"
#endif
```

In `mqtt_subscribe_current_phase()`, after the existing pairing topic subscriptions (after `cmd/unpair` subscribe at line 130), add the cluster topic subscription:

```c
        /* cluster roster sync — subscribe with rh=0 for retained message delivery */
        {
            const device_config_t *sub_cfg = device_config_get();
            if (sub_cfg->has_store_id && sub_cfg->store_id[0] != '\0') {
                esp_mqtt5_subscribe_property_config_t sub_prop = {
                    .retain_handle = 0,
                    .no_local_flag = false,
                };
                (void)esp_mqtt5_client_set_subscribe_property(s_client, &sub_prop);

                char cluster_topic[128];
                (void)snprintf(cluster_topic, sizeof(cluster_topic),
                               TOPIC_PREFIX "transmitter/cluster/%s/roster",
                               sub_cfg->store_id);
                (void)esp_mqtt_client_subscribe(s_client, cluster_topic, 1);
            }
        }
```

- [ ] **Step 2: Add cluster publish on connect (guarded by seq > 0)**

In `mqtt_subscribe_current_phase()`, after the existing `(void)roster_sync_publish();` call (line 140), add:

```c
        if (roster_get_active() != NULL && roster_get_active()->seq > 0) {
            (void)roster_cluster_publish();
        }
```

- [ ] **Step 3: Add cluster topic routing in `route_message()`**

In `route_message()`, add the cluster topic match **before** the `if (!cfg->has_public_id)` guard (line 351). Insert after the bootstrap topic check:

```c
#if CONFIG_TRANSMITTER_PAIRING_ENABLED
    if (cfg->has_store_id && cfg->store_id[0] != '\0') {
        char cluster_topic[128];
        (void)snprintf(cluster_topic, sizeof(cluster_topic),
                       TOPIC_PREFIX "transmitter/cluster/%s/roster", cfg->store_id);
        if (strcmp(topic, cluster_topic) == 0) {
            roster_cluster_handle_message((const uint8_t *)payload, payload_len);
            return;
        }
    }
#endif
```

---

### Task 12: Add `roster_cluster_publish()` Call Sites

**Files:**
- Modify: `transmitter/main/pair/roster_sync.c` (unpair and label handlers)
- Modify: `transmitter/main/display/display.c` (pair complete callback)
- Modify: `transmitter/main/serial/serial_protocol.c` (when the Transmitter Roster Delete Plan is implemented)

- [ ] **Step 1: Add include to `roster_sync.c`**

```c
#include "pair/roster_cluster.h"
```

- [ ] **Step 2: Add cluster publish after `roster_sync_handle_unpair()`**

In `roster_sync_handle_unpair()`, after the existing `roster_sync_publish()` call (or the `roster_remove` call), add:

```c
    (void)roster_cluster_publish();
```

- [ ] **Step 3: Add cluster publish after `roster_sync_handle_label()`**

In `roster_sync_handle_label()`, use one successful mutation block. For cluster-only implementation:

```c
    esp_err_t err = roster_set_name(s_roster, slot, label);
    if (err == ESP_OK) {
        (void)roster_sync_publish();
        (void)roster_cluster_publish();
    }
```

If `docs/planned/Transmitter Roster Delete Plan.md` Task 6A is also implemented, keep its `post_roster_changed("renamed", slot, label, "mqtt");` call in the same success block before the two publish calls. Do not create a second `roster_sync_publish()` call.

Note: `roster_sync_handle_label()` currently does NOT call `roster_sync_publish()` — but now that Task 1 fixed `roster_set_name()` to bump seq and set pending, the sync publish is needed here too.

- [ ] **Step 4: Add cluster publish in `display.c` pair complete callback**

The actual pairing complete callback is `pair_complete_cb()` in `display.c` (not in `espnow_pair_host.c`). That callback already calls `roster_sync_publish()` on success (line 1025).

Add include at the top of `display.c` (inside the `#if CONFIG_TRANSMITTER_PAIRING_ENABLED` include block):

```c
#include "pair/roster_cluster.h"
```

In `pair_complete_cb()`, after the existing `(void)roster_sync_publish();` call, add:

```c
        (void)roster_cluster_publish();
```

- [ ] **Step 5: Add cluster publish after OLED roster delete**

If `docs/planned/Transmitter Roster Delete Plan.md` is implemented in the same branch, add the same include to `display.c` and call `roster_cluster_publish()` in `display_delete_mode_confirm()` immediately after the existing `(void)roster_sync_publish();` call:

```c
    (void)roster_sync_publish();
    (void)roster_cluster_publish();
```

- [ ] **Step 6: Add cluster publish after serial roster delete**

If `docs/planned/Transmitter Roster Delete Plan.md` is implemented in the same branch, add `#include "pair/roster_cluster.h"` to `serial_protocol.c` inside the pairing-enabled include block and call `roster_cluster_publish()` in `handle_roster_unpair()` immediately after the existing `(void)roster_sync_publish();` call:

```c
    (void)roster_sync_publish();
    (void)roster_cluster_publish();
```

---

### Task 13: Initialize Cluster Sync at Startup

**Files:**
- Modify: `transmitter/main/main.c`

- [ ] **Step 1: Find and add `roster_cluster_init()` call**

In `main.c`, add the include beside the other pairing includes:

```c
#if CONFIG_TRANSMITTER_PAIRING_ENABLED
#include "pair/roster_cluster.h"
#endif
```

Then find the startup block that initializes the active roster and calls `roster_sync_init(&g_roster)`. Add `roster_cluster_init()` immediately after it with the same pointer:

```c
#if CONFIG_TRANSMITTER_PAIRING_ENABLED
    ESP_ERROR_CHECK(roster_sync_init(&g_roster));
    ESP_ERROR_CHECK(roster_cluster_init(&g_roster));
#endif
```

If the local variable name differs at implementation time, use the exact same `roster_t *` or `roster_t` address already passed to `roster_sync_init()`; do not introduce a new undefined `roster` symbol.

---

### Task 14: Update `docs/CHANGELOGS.md`

**Files:**
- Modify: `docs/CHANGELOGS.md`

- [ ] **Step 1: Add changelog entry**

```markdown
## Cluster Roster Sync

**Spec:** `docs/spec/Cluster Roster Sync Spec.md`

### Prerequisites
- Fixed `roster_set_name()` to bump roster `seq` and set `pending = true` (was missing, label changes never triggered sync)
- Increased MQTT buffer from 2048 to 8192 bytes and fragment limit from 4096 to 8192 (required for 32-receiver encrypted roster payloads)

### Backend
- Added `store_id` field to bootstrap activation result envelope (`ResultEnvelope`, `DeviceMqttPublisher.publishResult()`)
- Added `storeId` parameter to `DeviceActivationService` → `publishResult()` call chain
- Required `store_id` for transmitter hub activation results and documented the reactivation/backfill rollout for already-activated hubs

### Transmitter Firmware
- Added `store_id` / `has_store_id` fields to `device_config_t`, persisted in NVS during activation
- Updated `device_config_commit_activation()` to accept and store `store_id`
- Updated `handle_bootstrap_result()` in `mqtt.c` to extract `store_id` from activation response
- Added `CONFIG_TRANSMITTER_CLUSTER_ROSTER_EXPIRY_S` Kconfig entry (default 7 days)
- Added `mqtt_publish_retained_v5()` function for MQTT v5 property-aware retained publishes
- Registered `pair/roster_cluster.c` in `transmitter/main/CMakeLists.txt`
- Created `roster_cluster.c` / `roster_cluster.h` module:
  - AES-256-GCM encryption with HKDF-SHA256 key derivation from pairing PSK
  - Encrypted retained MQTT publish to `{prefix}/transmitter/cluster/{storeId}/roster`
  - Slot-level union merge with seq-based conflict resolution
  - Strict inbound roster JSON validation before marking parsed entries occupied
  - Self-publish detection (publisher_id + seq comparison)
  - Handles concurrent multi-hub pairing without data loss
- Added cluster topic subscription with `rh=0` (retain handling) in `mqtt_subscribe_current_phase()`
- Added cluster topic routing in `route_message()` (binary payload, before hub-specific topics)
- Added `roster_cluster_publish()` calls at all roster mutation sites (pairing, unpair, label, MQTT commands, OLED delete, serial delete, MQTT connect)
- Guarded connect-time publish with `seq > 0` to prevent fresh hubs from overwriting valid retained messages
```
