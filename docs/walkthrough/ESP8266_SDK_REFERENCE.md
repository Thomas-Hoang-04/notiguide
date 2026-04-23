# ESP8266 RTOS SDK v3.x - Complete API Reference

> **SDK Location**: `/opt/esp/ESP8266_RTOS_SDK/`
> **Toolchain**: `/opt/esp/xtensa-lx106-elf/`
> **FreeRTOS Kernel**: v10.0.1
> **Generated**: 2026-04-16

---

## Table of Contents

1. [ESP8266 Core (esp8266)](#1-esp8266-core)
2. [FreeRTOS (freertos)](#2-freertos)
3. [WiFi (esp8266/esp_wifi)](#3-wifi)
4. [TCP/IP Adapter (tcpip_adapter)](#4-tcpip-adapter)
5. [Event Loop (esp_event)](#5-event-loop)
6. [GPIO Driver (driver/gpio)](#6-gpio-driver)
7. [UART Driver (driver/uart)](#7-uart-driver)
8. [SPI Driver (driver/spi)](#8-spi-driver)
9. [I2C Driver (driver/i2c)](#9-i2c-driver)
10. [I2S Driver (driver/i2s)](#10-i2s-driver)
11. [ADC Driver (driver/adc)](#11-adc-driver)
12. [PWM Driver (driver/pwm)](#12-pwm-driver)
13. [LEDC Driver (driver/ledc)](#13-ledc-driver)
14. [Timer APIs (esp_timer, hw_timer)](#14-timer-apis)
15. [Sleep & Power Management](#15-sleep--power-management)
16. [NVS Flash (nvs_flash)](#16-nvs-flash)
17. [SPI Flash (spi_flash)](#17-spi-flash)
18. [Partition Table](#18-partition-table)
19. [SPI RAM (spi_ram)](#19-spi-ram)
20. [SPIFFS Filesystem (spiffs)](#20-spiffs-filesystem)
21. [FAT Filesystem (fatfs)](#21-fat-filesystem)
22. [Wear Levelling](#22-wear-levelling)
23. [Virtual Filesystem (vfs)](#23-virtual-filesystem)
24. [lwIP Network Stack (lwip)](#24-lwip-network-stack)
25. [ESP-TLS (esp-tls)](#25-esp-tls)
26. [TCP Transport (tcp_transport)](#26-tcp-transport)
27. [HTTP Client (esp_http_client)](#27-http-client)
28. [HTTP Server (esp_http_server)](#28-http-server)
29. [HTTPS OTA (esp_https_ota)](#29-https-ota)
30. [MQTT Client (mqtt)](#30-mqtt-client)
31. [CoAP Protocol (coap)](#31-coap-protocol)
32. [mDNS (mdns)](#32-mdns)
33. [ESP-NOW (esp_now)](#33-esp-now)
34. [SmartConfig](#34-smartconfig)
35. [WPA Supplicant (wpa_supplicant)](#35-wpa-supplicant)
36. [mbedTLS (mbedtls)](#36-mbedtls)
37. [OpenSSL Compatibility (openssl)](#37-openssl-compatibility)
38. [libsodium](#38-libsodium)
39. [wolfSSL (esp-wolfssl)](#39-wolfssl)
40. [Logging (log)](#40-logging)
41. [Console (console)](#41-console)
42. [Heap Memory (heap)](#42-heap-memory)
43. [Ring Buffer (esp_ringbuf)](#43-ring-buffer)
44. [cJSON (json)](#44-cjson)
45. [JSMN (jsmn)](#45-jsmn)
46. [HTTP Parser (http_parser)](#46-http-parser)
47. [OTA / App Update (app_update)](#47-ota--app-update)
48. [Bootloader Support](#48-bootloader-support)
49. [Protocol Communication (protocomm)](#49-protocol-communication)
50. [WiFi Provisioning (wifi_provisioning)](#50-wifi-provisioning)
51. [Modbus (freemodbus)](#51-modbus)
52. [Common Types & Errors (esp_common)](#52-common-types--errors)
53. [POSIX Threads (pthread)](#53-posix-threads)
54. [CRC & SHA Utilities](#54-crc--sha-utilities)
55. [PHY & RF Calibration](#55-phy--rf-calibration)

---

## 1. ESP8266 Core

**Component**: `esp8266` | **Headers**: `esp_system.h`, `esp_clk.h`, `esp_attr.h`, `esp_interface.h`, `esp_task_wdt.h`

### System Functions (`esp_system.h`)

```c
// MAC Address
esp_err_t esp_base_mac_addr_set(uint8_t *mac);
esp_err_t esp_base_mac_addr_get(uint8_t *mac);
esp_err_t esp_efuse_mac_get_default(uint8_t *mac);
esp_err_t esp_read_mac(uint8_t *mac, esp_mac_type_t type);
esp_err_t esp_derive_local_mac(uint8_t *local_mac, const uint8_t *universal_mac);
esp_err_t esp_mac_init(void);

// System Control
void esp_set_cpu_freq(esp_cpu_freq_t cpu_freq);    // ESP_CPU_FREQ_80M, ESP_CPU_FREQ_160M
void system_restore(void);                          // Reset to defaults (noreturn)
void esp_restart(void);                             // Restart CPU (noreturn)
esp_reset_reason_t esp_reset_reason(void);          // Get last reset reason

// Memory
uint32_t esp_get_free_heap_size(void);
uint32_t esp_get_minimum_free_heap_size(void);

// Random
uint32_t esp_random(void);
void esp_fill_random(void *buf, size_t len);

// Chip Info
void esp_chip_info(esp_chip_info_t *out_info);
flash_size_map system_get_flash_size_map(void);

// Clock
int esp_clk_cpu_freq(void);                         // Returns CPU clock in Hz
```

### Enums

```c
typedef enum { ESP_MAC_WIFI_STA, ESP_MAC_WIFI_SOFTAP } esp_mac_type_t;
typedef enum { ESP_CPU_FREQ_80M, ESP_CPU_FREQ_160M } esp_cpu_freq_t;
typedef enum {
    ESP_RST_UNKNOWN, ESP_RST_POWERON, ESP_RST_EXT, ESP_RST_SW,
    ESP_RST_PANIC, ESP_RST_INT_WDT, ESP_RST_TASK_WDT, ESP_RST_WDT,
    ESP_RST_DEEPSLEEP, ESP_RST_BROWNOUT, ESP_RST_SDIO, ESP_RST_FAST_SW
} esp_reset_reason_t;
typedef enum { ESP_IF_WIFI_STA, ESP_IF_WIFI_AP } esp_interface_t;
```

### Structs

```c
typedef struct { esp_chip_model_t model; uint32_t features; uint8_t cores; uint8_t revision; } esp_chip_info_t;
```

### Memory Attributes (`esp_attr.h`)

```c
IRAM_ATTR           // Force function/data to IRAM
DRAM_ATTR           // Force data to DRAM
DRAM_STR(str)       // Force string literal to DRAM
RTC_DATA_ATTR       // RTC slow memory data
RTC_RODATA_ATTR     // RTC read-only data
RTC_NOINIT_ATTR     // RTC no-init section
__NOINIT_ATTR       // No-init section
WORD_ALIGNED_ATTR   // 4-byte alignment
```

### Task Watchdog (`esp_task_wdt.h`)

```c
esp_err_t esp_task_wdt_init(void);
void esp_task_wdt_reset(void);
```

### Kconfig Highlights

| Option | Default | Description |
|--------|---------|-------------|
| `ESP8266_DEFAULT_CPU_FREQ_MHZ` | 160 | CPU frequency (80/160) |
| `ESP8266_XTAL_FREQ_SEL` | 26MHz | Crystal frequency |
| `ESP_PANIC` | PRINT_REBOOT | Panic handler behavior |
| `SOC_FULL_ICACHE` | off | Full cache mode |
| `SCAN_AP_MAX` | 99 | Max APs to scan |
| `ESP8266_WIFI_NVS_ENABLED` | on | WiFi NVS storage |
| `ESP8266_WIFI_ENABLE_WPA3_SAE` | on | WPA3 support |
| `ESP8266_PHY_MAX_WIFI_TX_POWER` | 20 | Max TX power dBm |

---

## 2. FreeRTOS

**Component**: `freertos` | **Version**: v10.0.1 | **Headers**: `freertos/task.h`, `queue.h`, `semphr.h`, `timers.h`, `event_groups.h`, `stream_buffer.h`

### Core Types

```c
TaskHandle_t, QueueHandle_t, SemaphoreHandle_t, TimerHandle_t, EventGroupHandle_t, StreamBufferHandle_t
typedef void (*TaskFunction_t)(void *);
typedef void (*TimerCallbackFunction_t)(TimerHandle_t);
typedef enum { eRunning, eReady, eBlocked, eSuspended, eDeleted } eTaskState;
typedef enum { eNoAction, eSetBits, eIncrement, eSetValueWithOverwrite, eSetValueWithoutOverwrite } eNotifyAction;
```

### Task Management (`task.h`)

```c
// Creation & Deletion
BaseType_t xTaskCreate(TaskFunction_t pxTaskCode, const char *pcName, configSTACK_DEPTH_TYPE usStackDepth, void *pvParameters, UBaseType_t uxPriority, TaskHandle_t *pxCreatedTask);
TaskHandle_t xTaskCreateStatic(TaskFunction_t pxTaskCode, const char *pcName, uint32_t ulStackDepth, void *pvParameters, UBaseType_t uxPriority, StackType_t *puxStackBuffer, StaticTask_t *pxTaskBuffer);
void vTaskDelete(TaskHandle_t xTaskToDelete);

// Delays
void vTaskDelay(const TickType_t xTicksToDelay);
void vTaskDelayUntil(TickType_t *pxPreviousWakeTime, const TickType_t xTimeIncrement);
BaseType_t xTaskAbortDelay(TaskHandle_t xTask);

// Suspend / Resume
void vTaskSuspend(TaskHandle_t xTaskToSuspend);
void vTaskResume(TaskHandle_t xTaskToResume);
BaseType_t xTaskResumeFromISR(TaskHandle_t xTaskToResume);

// Priority
UBaseType_t uxTaskPriorityGet(TaskHandle_t xTask);
void vTaskPrioritySet(TaskHandle_t xTask, UBaseType_t uxNewPriority);

// Info
eTaskState eTaskGetState(TaskHandle_t xTask);
void vTaskGetInfo(TaskHandle_t xTask, TaskStatus_t *pxTaskStatus, BaseType_t xGetFreeStackSpace, eTaskState eState);
TaskHandle_t xTaskGetCurrentTaskHandle(void);
TaskHandle_t xTaskGetHandle(const char *pcNameToQuery);
TaskHandle_t xTaskGetIdleTaskHandle(void);
UBaseType_t uxTaskGetStackHighWaterMark(TaskHandle_t xTask);

// Notifications
BaseType_t xTaskNotifyWait(uint32_t ulBitsToClearOnEntry, uint32_t ulBitsToClearOnExit, uint32_t *pulNotificationValue, TickType_t xTicksToWait);
void vTaskNotifyGiveFromISR(TaskHandle_t xTaskToNotify, BaseType_t *pxHigherPriorityTaskWoken);
BaseType_t xTaskNotifyStateClear(TaskHandle_t xTask);
// Macros: xTaskNotify(), xTaskNotifyGive(), ulTaskNotifyTake()

// Scheduler
void vTaskStartScheduler(void);
void vTaskEndScheduler(void);
void vTaskSuspendAll(void);
BaseType_t xTaskResumeAll(void);
BaseType_t xTaskGetSchedulerState(void);    // taskSCHEDULER_SUSPENDED / NOT_STARTED / RUNNING
TickType_t xTaskGetTickCount(void);
TickType_t xTaskGetTickCountFromISR(void);

// Statistics (requires configUSE_TRACE_FACILITY=1)
void vTaskList(char *pcWriteBuffer);
void vTaskGetRunTimeStats(char *pcWriteBuffer);
UBaseType_t uxTaskGetSystemState(TaskStatus_t *pxTaskStatusArray, UBaseType_t uxArraySize, uint32_t *pulTotalRunTime);
```

### Critical Section Macros

```c
taskENTER_CRITICAL();  taskEXIT_CRITICAL();
taskENTER_CRITICAL_FROM_ISR();  taskEXIT_CRITICAL_FROM_ISR(x);
taskDISABLE_INTERRUPTS();  taskENABLE_INTERRUPTS();
taskYIELD();
portTICK_PERIOD_MS    // ms per tick
portMAX_DELAY         // max delay (~0)
```

### Queue Management (`queue.h`)

```c
// Create / Delete
QueueHandle_t xQueueCreate(UBaseType_t uxQueueLength, UBaseType_t uxItemSize);
QueueHandle_t xQueueCreateStatic(UBaseType_t uxQueueLength, UBaseType_t uxItemSize, uint8_t *pucQueueStorage, StaticQueue_t *pxQueueBuffer);
void vQueueDelete(QueueHandle_t xQueue);

// Task Context
BaseType_t xQueueSend(QueueHandle_t xQueue, const void *pvItemToQueue, TickType_t xTicksToWait);
BaseType_t xQueueSendToFront(QueueHandle_t xQueue, const void *pvItemToQueue, TickType_t xTicksToWait);
BaseType_t xQueueSendToBack(QueueHandle_t xQueue, const void *pvItemToQueue, TickType_t xTicksToWait);
BaseType_t xQueueOverwrite(QueueHandle_t xQueue, const void *pvItemToQueue);
BaseType_t xQueueReceive(QueueHandle_t xQueue, void *pvBuffer, TickType_t xTicksToWait);
BaseType_t xQueuePeek(QueueHandle_t xQueue, void *pvBuffer, TickType_t xTicksToWait);

// ISR Context
BaseType_t xQueueSendFromISR(QueueHandle_t xQueue, const void *pvItemToQueue, BaseType_t *pxHigherPriorityTaskWoken);
BaseType_t xQueueSendToFrontFromISR(QueueHandle_t xQueue, const void *pvItemToQueue, BaseType_t *pxHigherPriorityTaskWoken);
BaseType_t xQueueSendToBackFromISR(QueueHandle_t xQueue, const void *pvItemToQueue, BaseType_t *pxHigherPriorityTaskWoken);
BaseType_t xQueueReceiveFromISR(QueueHandle_t xQueue, void *pvBuffer, BaseType_t *pxHigherPriorityTaskWoken);

// Status
UBaseType_t uxQueueMessagesWaiting(const QueueHandle_t xQueue);
UBaseType_t uxQueueSpacesAvailable(const QueueHandle_t xQueue);
BaseType_t xQueueIsQueueEmptyFromISR(const QueueHandle_t xQueue);
BaseType_t xQueueIsQueueFullFromISR(const QueueHandle_t xQueue);
#define xQueueReset(xQueue)

// Queue Sets
QueueSetHandle_t xQueueCreateSet(const UBaseType_t uxEventQueueLength);
BaseType_t xQueueAddToSet(QueueSetMemberHandle_t xQueueOrSemaphore, QueueSetHandle_t xQueueSet);
BaseType_t xQueueRemoveFromSet(QueueSetMemberHandle_t xQueueOrSemaphore, QueueSetHandle_t xQueueSet);
QueueSetMemberHandle_t xQueueSelectFromSet(QueueSetHandle_t xQueueSet, const TickType_t xTicksToWait);
```

### Semaphores & Mutexes (`semphr.h`)

```c
// Binary Semaphore
SemaphoreHandle_t xSemaphoreCreateBinary(void);
SemaphoreHandle_t xSemaphoreCreateBinaryStatic(StaticSemaphore_t *pxSemaphoreBuffer);

// Counting Semaphore
SemaphoreHandle_t xSemaphoreCreateCounting(UBaseType_t uxMaxCount, UBaseType_t uxInitialCount);

// Mutex (with priority inheritance)
SemaphoreHandle_t xSemaphoreCreateMutex(void);
SemaphoreHandle_t xSemaphoreCreateRecursiveMutex(void);

// Operations
BaseType_t xSemaphoreTake(SemaphoreHandle_t xSemaphore, TickType_t xBlockTime);
BaseType_t xSemaphoreGive(SemaphoreHandle_t xSemaphore);
BaseType_t xSemaphoreTakeFromISR(SemaphoreHandle_t xSemaphore, BaseType_t *pxHigherPriorityTaskWoken);
BaseType_t xSemaphoreGiveFromISR(SemaphoreHandle_t xSemaphore, BaseType_t *pxHigherPriorityTaskWoken);
BaseType_t xSemaphoreTakeRecursive(SemaphoreHandle_t xMutex, TickType_t xBlockTime);
BaseType_t xSemaphoreGiveRecursive(SemaphoreHandle_t xMutex);

// Utilities
void vSemaphoreDelete(SemaphoreHandle_t xSemaphore);
UBaseType_t uxSemaphoreGetCount(SemaphoreHandle_t xSemaphore);
TaskHandle_t xSemaphoreGetMutexHolder(SemaphoreHandle_t xMutex);
```

### Software Timers (`timers.h`)

```c
TimerHandle_t xTimerCreate(const char *pcTimerName, const TickType_t xTimerPeriodInTicks, const UBaseType_t uxAutoReload, void *pvTimerID, TimerCallbackFunction_t pxCallbackFunction);
BaseType_t xTimerStart(TimerHandle_t xTimer, TickType_t xTicksToWait);
BaseType_t xTimerStop(TimerHandle_t xTimer, TickType_t xTicksToWait);
BaseType_t xTimerReset(TimerHandle_t xTimer, TickType_t xTicksToWait);
BaseType_t xTimerChangePeriod(TimerHandle_t xTimer, TickType_t xNewPeriod, TickType_t xTicksToWait);
BaseType_t xTimerDelete(TimerHandle_t xTimer, TickType_t xTicksToWait);

// ISR variants
BaseType_t xTimerStartFromISR(TimerHandle_t xTimer, BaseType_t *pxHigherPriorityTaskWoken);
BaseType_t xTimerStopFromISR(TimerHandle_t xTimer, BaseType_t *pxHigherPriorityTaskWoken);
BaseType_t xTimerResetFromISR(TimerHandle_t xTimer, BaseType_t *pxHigherPriorityTaskWoken);

// Properties
void *pvTimerGetTimerID(const TimerHandle_t xTimer);
void vTimerSetTimerID(TimerHandle_t xTimer, void *pvNewID);
BaseType_t xTimerIsTimerActive(TimerHandle_t xTimer);
const char *pcTimerGetName(TimerHandle_t xTimer);
TickType_t xTimerGetPeriod(TimerHandle_t xTimer);

// Deferred Function Calls
BaseType_t xTimerPendFunctionCall(PendedFunction_t xFunctionToPend, void *pvParameter1, uint32_t ulParameter2, TickType_t xTicksToWait);
BaseType_t xTimerPendFunctionCallFromISR(PendedFunction_t xFunctionToPend, void *pvParameter1, uint32_t ulParameter2, BaseType_t *pxHigherPriorityTaskWoken);
```

### Event Groups (`event_groups.h`)

```c
EventGroupHandle_t xEventGroupCreate(void);
EventBits_t xEventGroupWaitBits(EventGroupHandle_t xEventGroup, const EventBits_t uxBitsToWaitFor, const BaseType_t xClearOnExit, const BaseType_t xWaitForAllBits, TickType_t xTicksToWait);
EventBits_t xEventGroupSetBits(EventGroupHandle_t xEventGroup, const EventBits_t uxBitsToSet);
BaseType_t xEventGroupSetBitsFromISR(EventGroupHandle_t xEventGroup, const EventBits_t uxBitsToSet, BaseType_t *pxHigherPriorityTaskWoken);
EventBits_t xEventGroupClearBits(EventGroupHandle_t xEventGroup, const EventBits_t uxBitsToClear);
EventBits_t xEventGroupSync(EventGroupHandle_t xEventGroup, const EventBits_t uxBitsToSet, const EventBits_t uxBitsToWaitFor, TickType_t xTicksToWait);
EventBits_t xEventGroupGetBits(EventGroupHandle_t xEventGroup);
void vEventGroupDelete(EventGroupHandle_t xEventGroup);
```

### Stream & Message Buffers (`stream_buffer.h`)

```c
StreamBufferHandle_t xStreamBufferCreate(size_t xBufferSizeBytes, size_t xTriggerLevelBytes);
size_t xStreamBufferSend(StreamBufferHandle_t xStreamBuffer, const void *pvTxData, size_t xDataLengthBytes, TickType_t xTicksToWait);
size_t xStreamBufferReceive(StreamBufferHandle_t xStreamBuffer, void *pvRxData, size_t xBufferLengthBytes, TickType_t xTicksToWait);
size_t xStreamBufferSendFromISR(StreamBufferHandle_t xStreamBuffer, const void *pvTxData, size_t xDataLengthBytes, BaseType_t *pxHigherPriorityTaskWoken);
size_t xStreamBufferReceiveFromISR(StreamBufferHandle_t xStreamBuffer, void *pvRxData, size_t xBufferLengthBytes, BaseType_t *pxHigherPriorityTaskWoken);
BaseType_t xStreamBufferIsFull(StreamBufferHandle_t xStreamBuffer);
BaseType_t xStreamBufferIsEmpty(StreamBufferHandle_t xStreamBuffer);
size_t xStreamBufferSpacesAvailable(StreamBufferHandle_t xStreamBuffer);
size_t xStreamBufferBytesAvailable(StreamBufferHandle_t xStreamBuffer);
BaseType_t xStreamBufferReset(StreamBufferHandle_t xStreamBuffer);
void vStreamBufferDelete(StreamBufferHandle_t xStreamBuffer);

// Message Buffers (whole-message variant of stream buffers)
MessageBufferHandle_t xMessageBufferCreate(size_t xBufferSizeBytes);
// Same API pattern as stream buffers but for discrete messages
```

### Return Values

```c
pdPASS (1) / pdFAIL (0) / pdTRUE (1) / pdFALSE (0)
errCOULD_NOT_ALLOCATE_REQUIRED_MEMORY
```

---

## 3. WiFi

**Component**: `esp8266` | **Header**: `esp_wifi.h`, `esp_wifi_types.h`

### Initialization & Control

```c
esp_err_t esp_wifi_init(const wifi_init_config_t *config);    // Use WIFI_INIT_CONFIG_DEFAULT()
esp_err_t esp_wifi_deinit(void);
esp_err_t esp_wifi_set_mode(wifi_mode_t mode);                // WIFI_MODE_STA, WIFI_MODE_AP, WIFI_MODE_APSTA
esp_err_t esp_wifi_get_mode(wifi_mode_t *mode);
esp_err_t esp_wifi_start(void);
esp_err_t esp_wifi_stop(void);
esp_err_t esp_wifi_restore(void);
esp_err_t esp_wifi_connect(void);
esp_err_t esp_wifi_disconnect(void);
wifi_state_t esp_wifi_get_state(void);
```

### Scanning

```c
esp_err_t esp_wifi_scan_start(const wifi_scan_config_t *config, bool block);
esp_err_t esp_wifi_scan_stop(void);
esp_err_t esp_wifi_scan_get_ap_num(uint16_t *number);
esp_err_t esp_wifi_scan_get_ap_records(uint16_t *number, wifi_ap_record_t *ap_records);
esp_err_t esp_wifi_sta_get_ap_info(wifi_ap_record_t *ap_info);
```

### Configuration

```c
esp_err_t esp_wifi_set_config(wifi_interface_t interface, wifi_config_t *conf);
esp_err_t esp_wifi_get_config(wifi_interface_t interface, wifi_config_t *conf);
esp_err_t esp_wifi_set_storage(wifi_storage_t storage);       // WIFI_STORAGE_FLASH or WIFI_STORAGE_RAM
esp_err_t esp_wifi_set_protocol(wifi_interface_t ifx, uint8_t protocol_bitmap);
esp_err_t esp_wifi_get_protocol(wifi_interface_t ifx, uint8_t *protocol_bitmap);
esp_err_t esp_wifi_set_bandwidth(wifi_interface_t ifx, wifi_bandwidth_t bw);
esp_err_t esp_wifi_set_channel(uint8_t primary, wifi_second_chan_t second);
esp_err_t esp_wifi_get_channel(uint8_t *primary, wifi_second_chan_t *second);
esp_err_t esp_wifi_set_country(const wifi_country_t *country);
esp_err_t esp_wifi_set_mac(wifi_interface_t ifx, const uint8_t mac[6]);
esp_err_t esp_wifi_get_mac(wifi_interface_t ifx, uint8_t mac[6]);
```

### Power & TX

```c
esp_err_t esp_wifi_set_ps(wifi_ps_type_t type);               // WIFI_PS_NONE, WIFI_PS_MIN_MODEM, WIFI_PS_MAX_MODEM
esp_err_t esp_wifi_get_ps(wifi_ps_type_t *type);
esp_err_t esp_wifi_set_max_tx_power(int8_t power);
esp_err_t esp_wifi_get_max_tx_power(int8_t *power);
```

### Promiscuous Mode

```c
esp_err_t esp_wifi_set_promiscuous(bool en);
esp_err_t esp_wifi_get_promiscuous(bool *en);
esp_err_t esp_wifi_set_promiscuous_rx_cb(wifi_promiscuous_cb_t cb);
esp_err_t esp_wifi_set_promiscuous_filter(const wifi_promiscuous_filter_t *filter);
```

### AP Mode

```c
esp_err_t esp_wifi_ap_get_sta_list(wifi_sta_list_t *sta);
esp_err_t esp_wifi_deauth_sta(uint16_t aid);
esp_err_t esp_wifi_set_inactive_time(wifi_interface_t ifx, uint16_t sec);
```

### Raw TX & Advanced

```c
esp_err_t esp_wifi_80211_tx(wifi_interface_t ifx, const void *buffer, int len, bool en_sys_seq);
esp_err_t esp_wifi_set_vendor_ie(bool enable, wifi_vendor_ie_type_t type, wifi_vendor_ie_id_t idx, const void *vnd_ie);
esp_err_t esp_wifi_set_vendor_ie_cb(esp_vendor_ie_cb_t cb, void *ctx);
esp_err_t esp_wifi_set_rssi_threshold(int32_t rssi);
int64_t esp_wifi_get_tsf_time(wifi_interface_t interface);
```

---

## 4. TCP/IP Adapter

**Component**: `tcpip_adapter` | **Header**: `tcpip_adapter.h`

### Types

```c
typedef enum { TCPIP_ADAPTER_IF_STA, TCPIP_ADAPTER_IF_AP, TCPIP_ADAPTER_IF_ETH, TCPIP_ADAPTER_IF_MAX } tcpip_adapter_if_t;
typedef struct { ip4_addr_t ip, netmask, gw; } tcpip_adapter_ip_info_t;
typedef enum { TCPIP_ADAPTER_DHCP_INIT, TCPIP_ADAPTER_DHCP_STARTED, TCPIP_ADAPTER_DHCP_STOPPED } tcpip_adapter_dhcp_status_t;
typedef enum { IP_EVENT_STA_GOT_IP, IP_EVENT_STA_LOST_IP, IP_EVENT_AP_STAIPASSIGNED, IP_EVENT_GOT_IP6 } ip_event_t;
```

### Functions

```c
void tcpip_adapter_init(void);
esp_err_t tcpip_adapter_get_ip_info(tcpip_adapter_if_t tcpip_if, tcpip_adapter_ip_info_t *ip_info);
esp_err_t tcpip_adapter_set_ip_info(tcpip_adapter_if_t tcpip_if, tcpip_adapter_ip_info_t *ip_info);
esp_err_t tcpip_adapter_set_dns_info(tcpip_adapter_if_t tcpip_if, tcpip_adapter_dns_type_t type, tcpip_adapter_dns_info_t *dns);
esp_err_t tcpip_adapter_get_dns_info(tcpip_adapter_if_t tcpip_if, tcpip_adapter_dns_type_t type, tcpip_adapter_dns_info_t *dns);

// DHCP Server
esp_err_t tcpip_adapter_dhcps_start(tcpip_adapter_if_t tcpip_if);
esp_err_t tcpip_adapter_dhcps_stop(tcpip_adapter_if_t tcpip_if);
esp_err_t tcpip_adapter_dhcps_get_status(tcpip_adapter_if_t tcpip_if, tcpip_adapter_dhcp_status_t *status);
esp_err_t tcpip_adapter_dhcps_option(tcpip_adapter_option_mode_t opt_op, tcpip_adapter_option_id_t opt_id, void *opt_val, uint32_t opt_len);

// DHCP Client
esp_err_t tcpip_adapter_dhcpc_start(tcpip_adapter_if_t tcpip_if);
esp_err_t tcpip_adapter_dhcpc_stop(tcpip_adapter_if_t tcpip_if);
esp_err_t tcpip_adapter_dhcpc_get_status(tcpip_adapter_if_t tcpip_if, tcpip_adapter_dhcp_status_t *status);

// Hostname
esp_err_t tcpip_adapter_set_hostname(tcpip_adapter_if_t tcpip_if, const char *hostname);
esp_err_t tcpip_adapter_get_hostname(tcpip_adapter_if_t tcpip_if, const char **hostname);

// Misc
esp_err_t tcpip_adapter_get_sta_list(wifi_sta_list_t *wifi_sta_list, tcpip_adapter_sta_list_t *tcpip_sta_list);
esp_err_t tcpip_adapter_get_netif(tcpip_adapter_if_t tcpip_if, void **netif);
bool tcpip_adapter_is_netif_up(tcpip_adapter_if_t tcpip_if);
esp_err_t tcpip_adapter_set_default_wifi_handlers(void);
esp_err_t tcpip_adapter_create_ip6_linklocal(tcpip_adapter_if_t tcpip_if);
```

### Macros

```c
IP2STR(ipaddr)    // e.g., printf(IPSTR, IP2STR(&ip_info.ip));
IPSTR             // "%d.%d.%d.%d"
IPV62STR(ipaddr)
IPV6STR           // "%04x:%04x:%04x:%04x:%04x:%04x:%04x:%04x"
```

---

## 5. Event Loop

**Component**: `esp_event` | **Header**: `esp_event.h`

### Types

```c
typedef const char *esp_event_base_t;
typedef void *esp_event_loop_handle_t;
typedef void (*esp_event_handler_t)(void *event_handler_arg, esp_event_base_t event_base, int32_t event_id, void *event_data);
ESP_EVENT_ANY_BASE  // NULL - register for any event base
ESP_EVENT_ANY_ID    // -1   - register for any event ID
```

### Functions

```c
// Default loop (recommended)
esp_err_t esp_event_loop_create_default(void);
esp_err_t esp_event_loop_delete_default(void);
esp_err_t esp_event_handler_register(esp_event_base_t event_base, int32_t event_id, esp_event_handler_t event_handler, void *event_handler_arg);
esp_err_t esp_event_handler_unregister(esp_event_base_t event_base, int32_t event_id, esp_event_handler_t event_handler);
esp_err_t esp_event_post(esp_event_base_t event_base, int32_t event_id, void *event_data, size_t event_data_size, TickType_t ticks_to_wait);

// Custom loops
esp_err_t esp_event_loop_create(const esp_event_loop_args_t *event_loop_args, esp_event_loop_handle_t *event_loop);
esp_err_t esp_event_loop_delete(esp_event_loop_handle_t event_loop);
esp_err_t esp_event_handler_register_with(esp_event_loop_handle_t event_loop, esp_event_base_t event_base, int32_t event_id, esp_event_handler_t event_handler, void *event_handler_arg);
esp_err_t esp_event_post_to(esp_event_loop_handle_t event_loop, esp_event_base_t event_base, int32_t event_id, void *event_data, size_t event_data_size, TickType_t ticks_to_wait);

// ISR
esp_err_t esp_event_isr_post(esp_event_base_t event_base, int32_t event_id, void *event_data, size_t event_data_size, BaseType_t *task_unblocked);
```

### Common Event Bases

```c
WIFI_EVENT    // WiFi events (STA_START, STA_CONNECTED, STA_DISCONNECTED, SCAN_DONE, AP_START, AP_STACONNECTED, etc.)
IP_EVENT      // IP events (STA_GOT_IP, STA_LOST_IP, AP_STAIPASSIGNED, GOT_IP6)
```

---

## 6. GPIO Driver

**Component**: `esp8266` | **Header**: `driver/gpio.h`

### Types

```c
typedef enum { GPIO_NUM_0=0, ..., GPIO_NUM_16=16, GPIO_NUM_MAX } gpio_num_t;
typedef enum { GPIO_INTR_DISABLE, GPIO_INTR_POSEDGE, GPIO_INTR_NEGEDGE, GPIO_INTR_ANYEDGE, GPIO_INTR_LOW_LEVEL, GPIO_INTR_HIGH_LEVEL } gpio_int_type_t;
typedef enum { GPIO_MODE_DISABLE, GPIO_MODE_INPUT, GPIO_MODE_OUTPUT, GPIO_MODE_OUTPUT_OD } gpio_mode_t;
typedef enum { GPIO_PULLUP_ONLY, GPIO_PULLDOWN_ONLY, GPIO_FLOATING } gpio_pull_mode_t;
typedef struct { uint32_t pin_bit_mask; gpio_mode_t mode; gpio_pullup_t pull_up_en; gpio_pulldown_t pull_down_en; gpio_int_type_t intr_type; } gpio_config_t;
```

### Functions

```c
esp_err_t gpio_config(const gpio_config_t *gpio_cfg);
esp_err_t gpio_set_level(gpio_num_t gpio_num, uint32_t level);
int gpio_get_level(gpio_num_t gpio_num);
esp_err_t gpio_set_direction(gpio_num_t gpio_num, gpio_mode_t mode);
esp_err_t gpio_set_intr_type(gpio_num_t gpio_num, gpio_int_type_t intr_type);
esp_err_t gpio_set_pull_mode(gpio_num_t gpio_num, gpio_pull_mode_t pull);
esp_err_t gpio_pullup_en(gpio_num_t gpio_num);
esp_err_t gpio_pullup_dis(gpio_num_t gpio_num);
esp_err_t gpio_pulldown_en(gpio_num_t gpio_num);
esp_err_t gpio_pulldown_dis(gpio_num_t gpio_num);
esp_err_t gpio_wakeup_enable(gpio_num_t gpio_num, gpio_int_type_t intr_type);
esp_err_t gpio_wakeup_disable(gpio_num_t gpio_num);

// ISR
esp_err_t gpio_install_isr_service(int no_use);
void gpio_uninstall_isr_service(void);
esp_err_t gpio_isr_handler_add(gpio_num_t gpio_num, gpio_isr_t isr_handler, void *args);
esp_err_t gpio_isr_handler_remove(gpio_num_t gpio_num);
esp_err_t gpio_isr_register(void (*fn)(void *), void *arg, int no_use, gpio_isr_handle_t *handle_no_use);
```

---

## 7. UART Driver

**Component**: `esp8266` | **Header**: `driver/uart.h`

### Types

```c
typedef enum { UART_NUM_0, UART_NUM_1, UART_NUM_MAX } uart_port_t;
typedef enum { UART_DATA_5_BITS, UART_DATA_6_BITS, UART_DATA_7_BITS, UART_DATA_8_BITS } uart_word_length_t;
typedef enum { UART_STOP_BITS_1, UART_STOP_BITS_1_5, UART_STOP_BITS_2 } uart_stop_bits_t;
typedef enum { UART_PARITY_DISABLE, UART_PARITY_EVEN, UART_PARITY_ODD } uart_parity_t;
typedef struct { int baud_rate; uart_word_length_t data_bits; uart_parity_t parity; uart_stop_bits_t stop_bits; uart_hw_flowcontrol_t flow_ctrl; uint8_t rx_flow_ctrl_thresh; } uart_config_t;
```

### Functions

```c
esp_err_t uart_param_config(uart_port_t uart_num, uart_config_t *uart_conf);
esp_err_t uart_driver_install(uart_port_t uart_num, int rx_buffer_size, int tx_buffer_size, int queue_size, QueueHandle_t *uart_queue, int no_use);
esp_err_t uart_driver_delete(uart_port_t uart_num);
bool uart_is_driver_installed(uart_port_t uart_num);

// I/O
int uart_write_bytes(uart_port_t uart_num, const char *src, size_t size);
int uart_read_bytes(uart_port_t uart_num, uint8_t *buf, uint32_t length, TickType_t ticks_to_wait);
int uart_tx_chars(uart_port_t uart_num, const char *buffer, uint32_t len);
esp_err_t uart_wait_tx_done(uart_port_t uart_num, TickType_t ticks_to_wait);
esp_err_t uart_flush(uart_port_t uart_num);
esp_err_t uart_flush_input(uart_port_t uart_num);
esp_err_t uart_get_buffered_data_len(uart_port_t uart_num, size_t *size);

// Config
esp_err_t uart_set_baudrate(uart_port_t uart_num, uint32_t baudrate);
esp_err_t uart_get_baudrate(uart_port_t uart_num, uint32_t *baudrate);
esp_err_t uart_set_word_length(uart_port_t uart_num, uart_word_length_t data_bit);
esp_err_t uart_set_stop_bits(uart_port_t uart_num, uart_stop_bits_t stop_bits);
esp_err_t uart_set_parity(uart_port_t uart_num, uart_parity_t parity_mode);
esp_err_t uart_set_hw_flow_ctrl(uart_port_t uart_num, uart_hw_flowcontrol_t flow_ctrl, uint8_t rx_thresh);
esp_err_t uart_enable_swap(void);       // Swap UART0 to GPIO13/15
esp_err_t uart_disable_swap(void);
esp_err_t uart_set_rx_timeout(uart_port_t uart_num, const uint8_t tout_thresh);
```

---

## 8. SPI Driver

**Component**: `esp8266` | **Header**: `driver/spi.h`

### Types

```c
typedef enum { CSPI_HOST, HSPI_HOST } spi_host_t;
typedef enum { SPI_MASTER_MODE, SPI_SLAVE_MODE } spi_mode_t;
typedef enum { SPI_2MHz_DIV=40, SPI_4MHz_DIV=20, SPI_5MHz_DIV=16, SPI_8MHz_DIV=10, SPI_10MHz_DIV=8, SPI_16MHz_DIV=5, SPI_20MHz_DIV=4, SPI_40MHz_DIV=2, SPI_80MHz_DIV=1 } spi_clk_div_t;
```

### Functions

```c
esp_err_t spi_init(spi_host_t host, spi_config_t *config);
esp_err_t spi_deinit(spi_host_t host);
esp_err_t spi_trans(spi_host_t host, spi_trans_t *trans);
esp_err_t spi_set_clk_div(spi_host_t host, spi_clk_div_t *clk_div);
esp_err_t spi_get_clk_div(spi_host_t host, spi_clk_div_t *clk_div);
esp_err_t spi_set_mode(spi_host_t host, spi_mode_t *mode);
esp_err_t spi_set_interface(spi_host_t host, spi_interface_t *interface);
esp_err_t spi_set_event_callback(spi_host_t host, spi_event_callback_t *event_cb);
esp_err_t spi_set_dummy(spi_host_t host, uint16_t *bitlen);
esp_err_t spi_slave_get_status(spi_host_t host, uint32_t *status);
esp_err_t spi_slave_set_status(spi_host_t host, uint32_t *status);
```

---

## 9. I2C Driver

**Component**: `esp8266` | **Header**: `driver/i2c.h`

### Functions

```c
esp_err_t i2c_driver_install(i2c_port_t i2c_num, i2c_mode_t mode);
esp_err_t i2c_driver_delete(i2c_port_t i2c_num);
esp_err_t i2c_param_config(i2c_port_t i2c_num, const i2c_config_t *i2c_conf);
esp_err_t i2c_set_pin(i2c_port_t i2c_num, int sda_io_num, int scl_io_num, gpio_pullup_t sda_pullup_en, gpio_pullup_t scl_pullup_en, i2c_mode_t mode);

// Command Link API
i2c_cmd_handle_t i2c_cmd_link_create(void);
void i2c_cmd_link_delete(i2c_cmd_handle_t cmd_handle);
esp_err_t i2c_master_start(i2c_cmd_handle_t cmd_handle);
esp_err_t i2c_master_stop(i2c_cmd_handle_t cmd_handle);
esp_err_t i2c_master_write_byte(i2c_cmd_handle_t cmd_handle, uint8_t data, bool ack_en);
esp_err_t i2c_master_write(i2c_cmd_handle_t cmd_handle, uint8_t *data, size_t data_len, bool ack_en);
esp_err_t i2c_master_read_byte(i2c_cmd_handle_t cmd_handle, uint8_t *data, i2c_ack_type_t ack);
esp_err_t i2c_master_read(i2c_cmd_handle_t cmd_handle, uint8_t *data, size_t data_len, i2c_ack_type_t ack);
esp_err_t i2c_master_cmd_begin(i2c_port_t i2c_num, i2c_cmd_handle_t cmd_handle, TickType_t ticks_to_wait);
```

### Types

```c
typedef enum { I2C_MODE_MASTER } i2c_mode_t;     // Master only on ESP8266
typedef enum { I2C_NUM_0 } i2c_port_t;           // Port 0 only
typedef enum { I2C_MASTER_ACK, I2C_MASTER_NACK, I2C_MASTER_LAST_NACK } i2c_ack_type_t;
```

---

## 10. I2S Driver

**Component**: `esp8266` | **Header**: `driver/i2s.h`

```c
esp_err_t i2s_driver_install(i2s_port_t i2s_num, const i2s_config_t *i2s_config, int queue_size, void *i2s_queue);
esp_err_t i2s_driver_uninstall(i2s_port_t i2s_num);
esp_err_t i2s_set_pin(i2s_port_t i2s_num, const i2s_pin_config_t *pin);
esp_err_t i2s_write(i2s_port_t i2s_num, const void *src, size_t size, size_t *bytes_written, TickType_t ticks_to_wait);
// i2s_port_t: I2S_NUM_0 only
// i2s_bits_per_sample_t: I2S_BITS_PER_SAMPLE_8BIT, _16BIT, _24BIT
// i2s_channel_t: I2S_CHANNEL_MONO, I2S_CHANNEL_STEREO
```

---

## 11. ADC Driver

**Component**: `esp8266` | **Header**: `driver/adc.h`

```c
typedef enum { ADC_READ_TOUT_MODE, ADC_READ_VDD_MODE } adc_mode_t;
typedef struct { adc_mode_t mode; uint8_t clk_div; } adc_config_t;

esp_err_t adc_init(adc_config_t *config);
esp_err_t adc_deinit(void);
esp_err_t adc_read(uint16_t *data);              // Single measurement
esp_err_t adc_read_fast(uint16_t *data, uint16_t len);  // Fast burst read
```

---

## 12. PWM Driver

**Component**: `esp8266` | **Header**: `driver/pwm.h`

```c
esp_err_t pwm_init(uint32_t period, uint32_t *duties, uint8_t channel_num, const uint32_t *pin_num);
esp_err_t pwm_deinit(void);
esp_err_t pwm_set_duty(uint8_t channel_num, uint32_t duty);
esp_err_t pwm_get_duty(uint8_t channel_num, uint32_t *duty_p);
esp_err_t pwm_set_period(uint32_t period);
esp_err_t pwm_get_period(uint32_t *period_p);
esp_err_t pwm_start(void);
esp_err_t pwm_stop(uint32_t stop_level_mask);
esp_err_t pwm_set_duties(uint32_t *duties);
esp_err_t pwm_set_phase(uint8_t channel_num, float phase);
esp_err_t pwm_set_phases(float *phases);
esp_err_t pwm_set_period_duties(uint32_t period, uint32_t *duties);
esp_err_t pwm_set_channel_invert(uint16_t channel_mask);
esp_err_t pwm_clear_channel_invert(uint16_t channel_mask);
```

---

## 13. LEDC Driver

**Component**: `esp8266` | **Header**: `driver/ledc.h`

```c
esp_err_t ledc_timer_config(const ledc_timer_config_t *timer_conf);
esp_err_t ledc_channel_config(const ledc_channel_config_t *ledc_conf);
esp_err_t ledc_set_duty(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t duty);
esp_err_t ledc_update_duty(ledc_mode_t speed_mode, ledc_channel_t channel);
esp_err_t ledc_set_duty_and_update(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t duty, ledc_fade_mode_t fade_mode);
// ledc_timer_t: LEDC_TIMER_0..3   ledc_channel_t: LEDC_CHANNEL_0..7
// ledc_timer_bit_t: 1-20 bits resolution
```

---

## 14. Timer APIs

### ESP Timer (Software, us precision) - `esp_timer.h`

```c
esp_err_t esp_timer_init(void);
esp_err_t esp_timer_create(const esp_timer_create_args_t *create_args, esp_timer_handle_t *out_handle);
esp_err_t esp_timer_start_once(esp_timer_handle_t timer, uint64_t timeout_us);
esp_err_t esp_timer_start_periodic(esp_timer_handle_t timer, uint64_t period);
esp_err_t esp_timer_stop(esp_timer_handle_t timer);
esp_err_t esp_timer_delete(esp_timer_handle_t timer);
int64_t esp_timer_get_time(void);    // Current time in microseconds since boot
```

### Hardware Timer - `driver/hw_timer.h`

```c
esp_err_t hw_timer_init(hw_timer_callback_t callback, void *arg);
esp_err_t hw_timer_deinit(void);
esp_err_t hw_timer_alarm_us(uint32_t value, bool reload);
esp_err_t hw_timer_disarm(void);
esp_err_t hw_timer_set_clkdiv(hw_timer_clkdiv_t clkdiv);   // TIMER_CLKDIV_1, _16, _256
esp_err_t hw_timer_set_intr_type(hw_timer_intr_type_t intr_type);
esp_err_t hw_timer_set_reload(bool reload);
esp_err_t hw_timer_enable(bool en);
esp_err_t hw_timer_set_load_data(uint32_t load_data);
uint32_t hw_timer_get_count_data(void);
```

---

## 15. Sleep & Power Management

**Header**: `esp_sleep.h`, `driver/rtc.h`

```c
// Deep Sleep
void esp_deep_sleep(uint64_t time_in_us);          // Enter deep sleep (noreturn)
void esp_deep_sleep_set_rf_option(uint8_t option);

// Light Sleep
esp_err_t esp_sleep_enable_timer_wakeup(uint32_t time_in_us);
esp_err_t esp_sleep_enable_gpio_wakeup(void);
esp_err_t esp_sleep_disable_wakeup_source(esp_sleep_source_t source);
esp_err_t esp_light_sleep_start(void);

// WiFi Force Sleep (deprecated, use esp_sleep instead)
void esp_wifi_fpm_open(void);
void esp_wifi_fpm_close(void);
esp_err_t esp_wifi_fpm_do_sleep(uint32_t sleep_time_in_us);
void esp_wifi_fpm_set_sleep_type(wifi_sleep_type_t type);

// RTC
rtc_cpu_freq_t rtc_clk_cpu_freq_get(void);
void rtc_clk_cpu_freq_set(rtc_cpu_freq_t cpu_freq);
uint32_t rtc_time_get(void);
```

---

## 16. NVS Flash

**Component**: `nvs_flash` | **Headers**: `nvs_flash.h`, `nvs.h`

### Initialization

```c
esp_err_t nvs_flash_init(void);
esp_err_t nvs_flash_init_partition(const char *partition_label);
esp_err_t nvs_flash_deinit(void);
esp_err_t nvs_flash_erase(void);
esp_err_t nvs_flash_erase_partition(const char *part_name);
```

### Open / Close

```c
esp_err_t nvs_open(const char *name, nvs_open_mode_t open_mode, nvs_handle_t *out_handle);
esp_err_t nvs_open_from_partition(const char *part_name, const char *name, nvs_open_mode_t open_mode, nvs_handle_t *out_handle);
void nvs_close(nvs_handle_t handle);
// open_mode: NVS_READONLY, NVS_READWRITE
```

### Set Values

```c
esp_err_t nvs_set_i8 / nvs_set_u8 / nvs_set_i16 / nvs_set_u16(nvs_handle_t handle, const char *key, <type> value);
esp_err_t nvs_set_i32 / nvs_set_u32 / nvs_set_i64 / nvs_set_u64(nvs_handle_t handle, const char *key, <type> value);
esp_err_t nvs_set_str(nvs_handle_t handle, const char *key, const char *value);
esp_err_t nvs_set_blob(nvs_handle_t handle, const char *key, const void *value, size_t length);
esp_err_t nvs_commit(nvs_handle_t handle);   // MUST call after set operations
```

### Get Values

```c
esp_err_t nvs_get_i8 / nvs_get_u8 / nvs_get_i16 / nvs_get_u16(nvs_handle_t handle, const char *key, <type> *out_value);
esp_err_t nvs_get_i32 / nvs_get_u32 / nvs_get_i64 / nvs_get_u64(nvs_handle_t handle, const char *key, <type> *out_value);
esp_err_t nvs_get_str(nvs_handle_t handle, const char *key, char *out_value, size_t *length);   // Pass NULL to get required size
esp_err_t nvs_get_blob(nvs_handle_t handle, const char *key, void *out_value, size_t *length);
```

### Erase & Stats

```c
esp_err_t nvs_erase_key(nvs_handle_t handle, const char *key);
esp_err_t nvs_erase_all(nvs_handle_t handle);
esp_err_t nvs_get_stats(const char *part_name, nvs_stats_t *nvs_stats);
esp_err_t nvs_get_used_entry_count(nvs_handle_t handle, size_t *used_entries);
```

### Iterator

```c
nvs_iterator_t nvs_entry_find(const char *part_name, const char *namespace_name, nvs_type_t type);
nvs_iterator_t nvs_entry_next(nvs_iterator_t iterator);
void nvs_entry_info(nvs_iterator_t iterator, nvs_entry_info_t *out_info);
void nvs_release_iterator(nvs_iterator_t iterator);
```

---

## 17. SPI Flash

**Component**: `spi_flash` | **Header**: `spi_flash.h`

```c
size_t spi_flash_get_chip_size(void);
esp_err_t spi_flash_erase_sector(size_t sector);                        // Erase 4KB sector
esp_err_t spi_flash_erase_range(size_t start_address, size_t size);
esp_err_t spi_flash_write(size_t dest_addr, const void *src, size_t size);
esp_err_t spi_flash_read(size_t src_addr, void *dest, size_t size);

// Memory mapping (CONFIG_ENABLE_FLASH_MMAP)
esp_err_t spi_flash_mmap(size_t src_addr, size_t size, spi_flash_mmap_memory_t memory, const void **out_ptr, spi_flash_mmap_handle_t *out_handle);
void spi_flash_munmap(spi_flash_mmap_handle_t handle);

#define SPI_FLASH_SEC_SIZE 4096    // Sector size
```

---

## 18. Partition Table

**Header**: `esp_partition.h`

```c
typedef enum { ESP_PARTITION_TYPE_APP=0x00, ESP_PARTITION_TYPE_DATA=0x01 } esp_partition_type_t;
typedef struct { esp_partition_type_t type; esp_partition_subtype_t subtype; uint32_t address; uint32_t size; char label[17]; bool encrypted; } esp_partition_t;

// Find
const esp_partition_t *esp_partition_find_first(esp_partition_type_t type, esp_partition_subtype_t subtype, const char *label);
esp_partition_iterator_t esp_partition_find(esp_partition_type_t type, esp_partition_subtype_t subtype, const char *label);
const esp_partition_t *esp_partition_get(esp_partition_iterator_t iterator);
esp_partition_iterator_t esp_partition_next(esp_partition_iterator_t iterator);
void esp_partition_iterator_release(esp_partition_iterator_t iterator);

// I/O
esp_err_t esp_partition_read(const esp_partition_t *partition, size_t src_offset, void *dst, size_t size);
esp_err_t esp_partition_write(const esp_partition_t *partition, size_t dst_offset, const void *src, size_t size);
esp_err_t esp_partition_erase_range(const esp_partition_t *partition, uint32_t start_addr, uint32_t size);
```

---

## 19. SPI RAM

**Component**: `spi_ram` | **Header**: `spi_ram.h`, `spi_ram_fifo.h`

```c
esp_err_t spi_ram_init(spi_ram_num_t num, spi_ram_config_t *config);
esp_err_t spi_ram_deinit(spi_ram_num_t num);
esp_err_t spi_ram_read(spi_ram_num_t num, uint32_t addr, uint8_t *data, int len);
esp_err_t spi_ram_write(spi_ram_num_t num, uint32_t addr, uint8_t *data, int len);
esp_err_t spi_ram_check(spi_ram_num_t num);

// FIFO
spi_ram_fifo_handle_t spi_ram_fifo_create(spi_ram_fifo_config_t *config);
esp_err_t spi_ram_fifo_write(spi_ram_fifo_handle_t handle, uint8_t *data, int len, uint32_t timeout_ticks);
esp_err_t spi_ram_fifo_read(spi_ram_fifo_handle_t handle, uint8_t *data, int len, uint32_t timeout_ticks);
esp_err_t spi_ram_fifo_get_fill / _get_free / _get_total(spi_ram_fifo_handle_t handle, uint32_t *len);
```

---

## 20. SPIFFS Filesystem

**Component**: `spiffs` | **Header**: `esp_spiffs.h`

```c
typedef struct { const char *base_path; const char *partition_label; size_t max_files; bool format_if_mount_failed; } esp_vfs_spiffs_conf_t;

esp_err_t esp_vfs_spiffs_register(const esp_vfs_spiffs_conf_t *conf);
esp_err_t esp_vfs_spiffs_unregister(const char *partition_label);
bool esp_spiffs_mounted(const char *partition_label);
esp_err_t esp_spiffs_format(const char *partition_label);
esp_err_t esp_spiffs_info(const char *partition_label, size_t *total_bytes, size_t *used_bytes);
// After registration, use standard POSIX: fopen(), fread(), fwrite(), fclose(), etc.
```

---

## 21. FAT Filesystem

**Component**: `fatfs` | **Header**: `esp_vfs_fat.h`

```c
typedef struct { bool format_if_mount_failed; int max_files; size_t allocation_unit_size; } esp_vfs_fat_mount_config_t;

// SPI Flash (with wear levelling)
esp_err_t esp_vfs_fat_spiflash_mount(const char *base_path, const char *partition_label, const esp_vfs_fat_mount_config_t *mount_config, wl_handle_t *wl_handle);
esp_err_t esp_vfs_fat_spiflash_unmount(const char *base_path, wl_handle_t wl_handle);

// Raw Flash (read-only)
esp_err_t esp_vfs_fat_rawflash_mount(const char *base_path, const char *partition_label, const esp_vfs_fat_mount_config_t *mount_config);
esp_err_t esp_vfs_fat_rawflash_unmount(const char *base_path, const char *partition_label);
```

---

## 22. Wear Levelling

**Component**: `wear_levelling` | **Header**: `wear_levelling.h`

```c
#define WL_INVALID_HANDLE -1
esp_err_t wl_mount(const esp_partition_t *partition, wl_handle_t *out_handle);
esp_err_t wl_unmount(wl_handle_t handle);
esp_err_t wl_erase_range(wl_handle_t handle, size_t start_addr, size_t size);
esp_err_t wl_write(wl_handle_t handle, size_t dest_addr, const void *src, size_t size);
esp_err_t wl_read(wl_handle_t handle, size_t src_addr, void *dest, size_t size);
size_t wl_size(wl_handle_t handle);
size_t wl_sector_size(wl_handle_t handle);
```

---

## 23. Virtual Filesystem

**Component**: `vfs` | **Header**: `esp_vfs.h`, `esp_vfs_dev.h`

```c
esp_err_t esp_vfs_register(const char *base_path, const esp_vfs_t *vfs, void *ctx);
esp_err_t esp_vfs_unregister(const char *base_path);
int esp_vfs_select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *errorfds, struct timeval *timeout);

// UART VFS
void esp_vfs_dev_uart_register(void);
void esp_vfs_dev_uart_set_rx_line_endings(esp_line_endings_t mode);
void esp_vfs_dev_uart_set_tx_line_endings(esp_line_endings_t mode);
void esp_vfs_dev_uart_use_driver(int uart_num);
```

---

## 24. lwIP Network Stack

**Component**: `lwip` | **Key Headers**: `dhcpserver.h`, `esp_ping.h`, `sntp.h`

### DHCP Server

```c
void dhcps_start(struct netif *netif, ip4_addr_t ip);
void dhcps_stop(struct netif *netif);
void dhcps_set_option_info(u8_t op_id, void *opt_info, u32_t opt_len);
void dhcps_dns_setserver(const ip_addr_t *dnsserver);
void dhcps_set_new_lease_cb(dhcps_cb_t cb);
```

### Ping (Modern Socket API)

```c
esp_err_t esp_ping_new_session(const esp_ping_config_t *config, const esp_ping_callbacks_t *cbs, esp_ping_handle_t *hdl_out);
esp_err_t esp_ping_delete_session(esp_ping_handle_t hdl);
esp_err_t esp_ping_start(esp_ping_handle_t hdl);
esp_err_t esp_ping_stop(esp_ping_handle_t hdl);
esp_err_t esp_ping_get_profile(esp_ping_handle_t hdl, esp_ping_profile_t profile, void *data, uint32_t size);
```

### SNTP

```c
void sntp_set_sync_mode(sntp_sync_mode_t sync_mode);
sntp_sync_status_t sntp_get_sync_status(void);
void sntp_set_time_sync_notification_cb(sntp_sync_time_cb_t callback);
void sntp_set_sync_interval(uint32_t interval_ms);
bool sntp_restart(void);
// Also uses standard lwIP: sntp_setoperatingmode(), sntp_setservername(), sntp_init()
```

---

## 25. ESP-TLS

**Component**: `esp-tls` | **Header**: `esp_tls.h`

### Types

```c
typedef struct {
    const char **alpn_protos; const unsigned char *cacert_buf; unsigned int cacert_bytes;
    const unsigned char *clientcert_buf; unsigned int clientcert_bytes;
    const unsigned char *clientkey_buf; unsigned int clientkey_bytes;
    bool non_block; int timeout_ms; bool use_global_ca_store;
    const char *common_name; bool skip_common_name;
    const psk_hint_key_t *psk_hint_key;
} esp_tls_cfg_t;
typedef enum { ESP_TLS_INIT, ESP_TLS_CONNECTING, ESP_TLS_HANDSHAKE, ESP_TLS_FAIL, ESP_TLS_DONE } esp_tls_conn_state_t;
```

### Functions

```c
esp_tls_t *esp_tls_init(void);
esp_tls_t *esp_tls_conn_http_new(const char *url, const esp_tls_cfg_t *cfg);
int esp_tls_conn_new_sync(const char *hostname, int hostlen, int port, const esp_tls_cfg_t *cfg, esp_tls_t *tls);
int esp_tls_conn_new_async(const char *hostname, int hostlen, int port, const esp_tls_cfg_t *cfg, esp_tls_t *tls);
ssize_t esp_tls_conn_write(esp_tls_t *tls, const void *data, size_t datalen);
ssize_t esp_tls_conn_read(esp_tls_t *tls, void *data, size_t datalen);
void esp_tls_conn_delete(esp_tls_t *tls);
ssize_t esp_tls_get_bytes_avail(esp_tls_t *tls);
esp_err_t esp_tls_get_conn_sockfd(esp_tls_t *tls, int *sockfd);

// Global CA Store
esp_err_t esp_tls_init_global_ca_store(void);
esp_err_t esp_tls_set_global_ca_store(const unsigned char *cacert_pem_buf, const unsigned int cacert_pem_bytes);
void esp_tls_free_global_ca_store(void);

// Server (CONFIG_ESP_TLS_SERVER)
int esp_tls_server_session_create(esp_tls_cfg_server_t *cfg, int sockfd, esp_tls_t *tls);
void esp_tls_server_session_delete(esp_tls_t *tls);
```

---

## 26. TCP Transport

**Component**: `tcp_transport` | **Headers**: `esp_transport.h`, `esp_transport_tcp.h`, `esp_transport_ssl.h`, `esp_transport_ws.h`

```c
// Core
esp_transport_handle_t esp_transport_init(void);
int esp_transport_connect(esp_transport_handle_t t, const char *host, int port, int timeout_ms);
int esp_transport_read(esp_transport_handle_t t, char *buffer, int len, int timeout_ms);
int esp_transport_write(esp_transport_handle_t t, const char *buffer, int len, int timeout_ms);
int esp_transport_close(esp_transport_handle_t t);
esp_err_t esp_transport_destroy(esp_transport_handle_t t);

// TCP
esp_transport_handle_t esp_transport_tcp_init(void);

// SSL
esp_transport_handle_t esp_transport_ssl_init(void);
void esp_transport_ssl_set_cert_data(esp_transport_handle_t t, const char *data, int len);
void esp_transport_ssl_set_client_cert_data(esp_transport_handle_t t, const char *data, int len);
void esp_transport_ssl_set_client_key_data(esp_transport_handle_t t, const char *data, int len);
void esp_transport_ssl_enable_global_ca_store(esp_transport_handle_t t);
void esp_transport_ssl_skip_common_name_check(esp_transport_handle_t t);

// WebSocket
esp_transport_handle_t esp_transport_ws_init(esp_transport_handle_t parent_handle);
void esp_transport_ws_set_path(esp_transport_handle_t t, const char *path);
esp_err_t esp_transport_ws_set_subprotocol(esp_transport_handle_t t, const char *sub_protocol);
int esp_transport_ws_send_raw(esp_transport_handle_t t, ws_transport_opcodes_t opcode, const char *b, int len, int timeout_ms);
ws_transport_opcodes_t esp_transport_ws_get_read_opcode(esp_transport_handle_t t);
```

---

## 27. HTTP Client

**Component**: `esp_http_client` | **Header**: `esp_http_client.h`

### Types

```c
typedef enum { HTTP_EVENT_ERROR, HTTP_EVENT_ON_CONNECTED, HTTP_EVENT_HEADERS_SENT, HTTP_EVENT_ON_HEADER, HTTP_EVENT_ON_DATA, HTTP_EVENT_ON_FINISH, HTTP_EVENT_DISCONNECTED } esp_http_client_event_id_t;
typedef enum { HTTP_METHOD_GET, HTTP_METHOD_POST, HTTP_METHOD_PUT, HTTP_METHOD_PATCH, HTTP_METHOD_DELETE, HTTP_METHOD_HEAD, HTTP_METHOD_OPTIONS } esp_http_client_method_t;
```

### Functions

```c
esp_http_client_handle_t esp_http_client_init(const esp_http_client_config_t *config);
esp_err_t esp_http_client_perform(esp_http_client_handle_t client);
esp_err_t esp_http_client_cleanup(esp_http_client_handle_t client);

// Streaming API
esp_err_t esp_http_client_open(esp_http_client_handle_t client, int write_len);
int esp_http_client_write(esp_http_client_handle_t client, const char *buffer, int len);
int esp_http_client_fetch_headers(esp_http_client_handle_t client);
int esp_http_client_read(esp_http_client_handle_t client, char *buffer, int len);
esp_err_t esp_http_client_close(esp_http_client_handle_t client);

// Configuration
esp_err_t esp_http_client_set_url(esp_http_client_handle_t client, const char *url);
esp_err_t esp_http_client_set_method(esp_http_client_handle_t client, esp_http_client_method_t method);
esp_err_t esp_http_client_set_header(esp_http_client_handle_t client, const char *key, const char *value);
esp_err_t esp_http_client_set_post_field(esp_http_client_handle_t client, const char *data, int len);

// Response
int esp_http_client_get_status_code(esp_http_client_handle_t client);
int esp_http_client_get_content_length(esp_http_client_handle_t client);
bool esp_http_client_is_chunked_response(esp_http_client_handle_t client);
```

---

## 28. HTTP Server

**Component**: `esp_http_server` | **Header**: `esp_http_server.h`

```c
// Lifecycle
esp_err_t httpd_start(httpd_handle_t *handle, const httpd_config_t *config);  // Use HTTPD_DEFAULT_CONFIG()
esp_err_t httpd_stop(httpd_handle_t handle);

// URI Handlers
esp_err_t httpd_register_uri_handler(httpd_handle_t handle, const httpd_uri_t *uri_handler);
esp_err_t httpd_unregister_uri_handler(httpd_handle_t handle, const char *uri, httpd_method_t method);
esp_err_t httpd_unregister_uri(httpd_handle_t handle, const char *uri);

// Request
int httpd_req_recv(httpd_req_t *r, char *buf, size_t buf_len);
size_t httpd_req_get_hdr_value_len(httpd_req_t *r, const char *field);
esp_err_t httpd_req_get_hdr_value_str(httpd_req_t *r, const char *field, char *val, size_t val_size);
size_t httpd_req_get_url_query_len(httpd_req_t *r);
esp_err_t httpd_req_get_url_query_str(httpd_req_t *r, char *buf, size_t buf_len);
esp_err_t httpd_query_key_value(const char *qry, const char *key, char *val, size_t val_size);

// Response
esp_err_t httpd_resp_send(httpd_req_t *r, const char *buf, ssize_t buf_len);
esp_err_t httpd_resp_send_chunk(httpd_req_t *r, const char *buf, ssize_t buf_len);
esp_err_t httpd_resp_set_status(httpd_req_t *r, const char *status);
esp_err_t httpd_resp_set_type(httpd_req_t *r, const char *type);
esp_err_t httpd_resp_set_hdr(httpd_req_t *r, const char *field, const char *value);
esp_err_t httpd_resp_send_404 / _408 / _500(httpd_req_t *r);
```

---

## 29. HTTPS OTA

**Component**: `esp_https_ota` | **Header**: `esp_https_ota.h`

```c
esp_err_t esp_https_ota(const esp_http_client_config_t *config);
// One-call HTTPS OTA. Requires cert_pem in config for server verification.
// Returns ESP_OK on success; next reboot uses new partition.
```

---

## 30. MQTT Client

**Component**: `mqtt` | **Header**: `mqtt_client.h`

### Types

```c
typedef enum {
    MQTT_EVENT_ERROR, MQTT_EVENT_CONNECTED, MQTT_EVENT_DISCONNECTED,
    MQTT_EVENT_SUBSCRIBED, MQTT_EVENT_UNSUBSCRIBED, MQTT_EVENT_PUBLISHED,
    MQTT_EVENT_DATA, MQTT_EVENT_BEFORE_CONNECT, MQTT_EVENT_ANY=-1
} esp_mqtt_event_id_t;

typedef enum {
    MQTT_TRANSPORT_OVER_TCP, MQTT_TRANSPORT_OVER_SSL,
    MQTT_TRANSPORT_OVER_WS, MQTT_TRANSPORT_OVER_WSS
} esp_mqtt_transport_t;
```

### Functions

```c
esp_mqtt_client_handle_t esp_mqtt_client_init(const esp_mqtt_client_config_t *config);
esp_err_t esp_mqtt_client_start(esp_mqtt_client_handle_t client);
esp_err_t esp_mqtt_client_stop(esp_mqtt_client_handle_t client);
esp_err_t esp_mqtt_client_destroy(esp_mqtt_client_handle_t client);
esp_err_t esp_mqtt_client_reconnect(esp_mqtt_client_handle_t client);
esp_err_t esp_mqtt_client_disconnect(esp_mqtt_client_handle_t client);
esp_err_t esp_mqtt_client_set_uri(esp_mqtt_client_handle_t client, const char *uri);
esp_err_t esp_mqtt_set_config(esp_mqtt_client_handle_t client, const esp_mqtt_client_config_t *config);

// Pub/Sub (return msg_id on success, -1 on failure)
int esp_mqtt_client_subscribe(esp_mqtt_client_handle_t client, const char *topic, int qos);
int esp_mqtt_client_unsubscribe(esp_mqtt_client_handle_t client, const char *topic);
int esp_mqtt_client_publish(esp_mqtt_client_handle_t client, const char *topic, const char *data, int len, int qos, int retain);

// Event handler
esp_err_t esp_mqtt_client_register_event(esp_mqtt_client_handle_t client, esp_mqtt_event_id_t event, esp_event_handler_t event_handler, void *event_handler_arg);
```

### Config Struct Key Fields

```c
esp_mqtt_client_config_t: { uri, host, port, client_id, username, password,
    lwt_topic, lwt_msg, lwt_qos, lwt_retain, keepalive, disable_clean_session,
    disable_auto_reconnect, cert_pem, client_cert_pem, client_key_pem,
    transport, task_prio, task_stack, buffer_size, user_context }
```

### Default Ports

| Transport | Port |
|-----------|------|
| TCP | 1883 |
| SSL | 8883 |
| WS  | 80   |
| WSS | 443  |

---

## 31. CoAP Protocol

**Component**: `coap` | **Header**: `coap/coap.h` (aggregates libcoap)

### Key Functions

```c
// Lifecycle
void coap_startup(void);
void coap_cleanup(void);
coap_context_t *coap_new_context(const coap_address_t *listen_addr);

// Sessions
coap_session_t *coap_new_client_session(coap_context_t *ctx, const coap_address_t *local_if, const coap_address_t *server, coap_proto_t proto);
coap_session_t *coap_new_client_session_psk(coap_context_t *ctx, const coap_address_t *local_if, const coap_address_t *server, coap_proto_t proto, coap_dtls_cpsk_t *psk_data);

// PDU (Protocol Data Unit)
coap_pdu_t *coap_pdu_init(uint8_t type, uint8_t code, uint16_t tid, size_t size);
coap_pdu_t *coap_new_pdu(const coap_session_t *session);

// Resources
coap_resource_t *coap_resource_init(coap_str_const_t *uri_path, int flags);
void coap_register_handler(coap_resource_t *resource, unsigned char method, coap_method_handler_t handler);
void coap_add_resource(coap_context_t *context, coap_resource_t *resource);

// Send/Receive
coap_tid_t coap_send(coap_session_t *session, coap_pdu_t *pdu);
void coap_register_response_handler(coap_context_t *ctx, coap_response_handler_t handler);

// Message types: COAP_MESSAGE_CON(0), NON(1), ACK(2), RST(3)
// Methods: COAP_REQUEST_GET(1), POST(2), PUT(3), DELETE(4)
// Ports: COAP_DEFAULT_PORT(5683), COAPS_DEFAULT_PORT(5684)
```

---

## 32. mDNS

**Component**: `mdns` | **Header**: `mdns.h`

```c
esp_err_t mdns_init(void);
void mdns_free(void);
esp_err_t mdns_hostname_set(const char *hostname);
esp_err_t mdns_instance_name_set(const char *instance_name);

// Service Management
esp_err_t mdns_service_add(const char *instance_name, const char *service_type, const char *proto, uint16_t port, mdns_txt_item_t txt[], size_t num_items);
esp_err_t mdns_service_remove(const char *service_type, const char *proto);
esp_err_t mdns_service_txt_set(const char *service_type, const char *proto, mdns_txt_item_t txt[], uint8_t num_items);
esp_err_t mdns_service_txt_item_set(const char *service_type, const char *proto, const char *key, const char *value);

// Queries
esp_err_t mdns_query_ptr(const char *service_type, const char *proto, uint32_t timeout, size_t max_results, mdns_result_t **results);
esp_err_t mdns_query_srv(const char *instance, const char *service_type, const char *proto, uint32_t timeout, mdns_result_t **result);
esp_err_t mdns_query_a(const char *host_name, uint32_t timeout, ip4_addr_t *addr);
esp_err_t mdns_query_aaaa(const char *host_name, uint32_t timeout, ip6_addr_t *addr);
void mdns_query_results_free(mdns_result_t *results);
```

---

## 33. ESP-NOW

**Component**: `esp8266` | **Header**: `esp_now.h`

```c
esp_err_t esp_now_init(void);
esp_err_t esp_now_deinit(void);
esp_err_t esp_now_register_recv_cb(esp_now_recv_cb_t cb);   // void (*)(const uint8_t *mac, const uint8_t *data, int len)
esp_err_t esp_now_register_send_cb(esp_now_send_cb_t cb);
esp_err_t esp_now_send(const uint8_t *peer_addr, const uint8_t *data, size_t len);
esp_err_t esp_now_add_peer(const esp_now_peer_info_t *peer);
esp_err_t esp_now_del_peer(const uint8_t *peer_addr);
esp_err_t esp_now_get_peer(const uint8_t *peer_addr, esp_now_peer_info_t *peer);
// ESP_NOW_MAX_DATA_LEN=250, ESP_NOW_KEY_LEN=16, ESP_NOW_ETH_ALEN=6
```

---

## 34. SmartConfig

**Component**: `esp8266` | **Header**: `esp_smartconfig.h`

```c
esp_err_t esp_smartconfig_start(const smartconfig_start_config_t *config);
esp_err_t esp_smartconfig_stop(void);
esp_err_t esp_smartconfig_set_type(smartconfig_type_t type);
// Types: SC_TYPE_ESPTOUCH, SC_TYPE_AIRKISS, SC_TYPE_ESPTOUCH_AIRKISS, SC_TYPE_ESPTOUCH_V2
// Events via esp_event: SC_EVENT (SCAN_DONE, FOUND_CHANNEL, GOT_SSID_PSWD, SEND_ACK_DONE)
```

---

## 35. WPA Supplicant

**Component**: `wpa_supplicant` | **Headers**: `esp_wpa.h`, `esp_wpa2.h`, `esp_wps.h`

### WPA2 Enterprise

```c
esp_err_t esp_wifi_sta_wpa2_ent_enable(void);
esp_err_t esp_wifi_sta_wpa2_ent_disable(void);
esp_err_t esp_wifi_sta_wpa2_ent_set_identity(const unsigned char *identity, int len);
esp_err_t esp_wifi_sta_wpa2_ent_set_username(const unsigned char *username, int len);
esp_err_t esp_wifi_sta_wpa2_ent_set_password(const unsigned char *password, int len);
esp_err_t esp_wifi_sta_wpa2_ent_set_ca_cert(const unsigned char *ca_cert, int len);
esp_err_t esp_wifi_sta_wpa2_ent_set_cert_key(const unsigned char *client_cert, int cert_len, const unsigned char *private_key, int key_len, const unsigned char *passwd, int passwd_len);
```

### WPS

```c
esp_err_t esp_wifi_wps_enable(const esp_wps_config_t *config);
esp_err_t esp_wifi_wps_disable(void);
esp_err_t esp_wifi_wps_start(int timeout_ms);
// Types: WPS_TYPE_PBC, WPS_TYPE_PIN
```

---

## 36. mbedTLS

**Component**: `mbedtls` | **Key ESP Headers**: `esp_mem.h`, `esp8266/esp_aes.h`, `esp8266/esp_md5.h`, `esp8266/esp_base64.h`

### ESP-Specific Crypto Accelerations

```c
// AES (all modes: ECB, CBC, CFB128, CFB8, CTR, XTS, OFB)
int esp_aes_set_encrypt_key(esp_aes_t *aes, const void *key, size_t keybits);
int esp_aes_set_decrypt_key(esp_aes_t *aes, const void *key, size_t keybits);
int esp_aes_encrypt_cbc(esp_aes_t *aes, const void *src, size_t slen, void *dst, size_t dlen, void *iv);
int esp_aes_decrypt_cbc(esp_aes_t *aes, const void *src, size_t slen, void *dst, size_t dlen, void *iv);
int esp_aes_encrypt_ctr(esp_aes_t *aes, size_t *nc_off, void *nonce, void *stream_block, const void *src, size_t slen, void *dst, size_t dlen);
// + ECB, CFB128, CFB8, XTS, OFB variants

// MD5
int esp_md5_init(esp_md5_context_t *ctx);
int esp_md5_update(esp_md5_context_t *ctx, const uint8_t *input, size_t ilen);
int esp_md5_final(esp_md5_context_t *ctx, uint8_t output[16]);

// Base64
int esp_base64_encode(const void *src, uint32_t slen, void *dst, uint32_t dlen);
int esp_base64_decode(const void *src, uint32_t slen, void *dst, uint32_t dlen);

// Memory
void *esp_mbedtls_mem_calloc(size_t n, size_t size);
void esp_mbedtls_mem_free(void *ptr);

// Debug
void mbedtls_esp_enable_debug_log(mbedtls_ssl_config *conf, int threshold);  // 0-4
void mbedtls_esp_disable_debug_log(mbedtls_ssl_config *conf);
```

> Note: Full mbedTLS library APIs (ssl, x509, pk, etc.) are available via standard mbedTLS headers.

---

## 37. OpenSSL Compatibility

**Component**: `openssl` | **Header**: `openssl/ssl.h`

```c
// Context & Connection
SSL_CTX *SSL_CTX_new(const SSL_METHOD *method);
void SSL_CTX_free(SSL_CTX *ctx);
SSL *SSL_new(SSL_CTX *ctx);
void SSL_free(SSL *ssl);
int SSL_set_fd(SSL *ssl, int fd);

// Handshake & I/O
int SSL_connect(SSL *ssl);     // Client handshake (1=OK)
int SSL_accept(SSL *ssl);      // Server handshake (1=OK)
int SSL_read(SSL *ssl, void *buffer, int len);
int SSL_write(SSL *ssl, const void *buffer, int len);
int SSL_shutdown(SSL *ssl);
int SSL_do_handshake(SSL *ssl);

// Methods
const SSL_METHOD *TLSv1_2_client_method(void);
const SSL_METHOD *TLS_client_method(void);
const SSL_METHOD *TLSv1_2_server_method(void);
const SSL_METHOD *TLS_server_method(void);

// Certificates
int SSL_CTX_use_certificate(SSL_CTX *ctx, X509 *x);
int SSL_CTX_use_PrivateKey(SSL_CTX *ctx, EVP_PKEY *pkey);
X509 *d2i_X509(X509 **cert, const unsigned char *buffer, long len);
EVP_PKEY *d2i_PrivateKey(int type, EVP_PKEY **a, const unsigned char **pp, long length);

// ALPN
int SSL_CTX_set_alpn_protos(SSL_CTX *ctx, const unsigned char *protos, unsigned int len);

// Error
int SSL_get_error(const SSL *ssl, int ret_code);
// SSL_ERROR_NONE(0), SSL_ERROR_WANT_READ(2), SSL_ERROR_WANT_WRITE(3)
```

---

## 38. libsodium

**Component**: `libsodium` | **Header**: `sodium.h`

### Core

```c
int sodium_init(void);
```

### Random

```c
void randombytes_buf(void *buf, size_t size);
uint32_t randombytes_random(void);
uint32_t randombytes_uniform(uint32_t upper_bound);
```

### Authenticated Encryption (ChaCha20-Poly1305)

```c
int crypto_aead_chacha20poly1305_ietf_encrypt(unsigned char *c, unsigned long long *clen, const unsigned char *m, unsigned long long mlen, const unsigned char *ad, unsigned long long adlen, const unsigned char *nsec, const unsigned char *npub, const unsigned char *k);
int crypto_aead_chacha20poly1305_ietf_decrypt(unsigned char *m, unsigned long long *mlen, unsigned char *nsec, const unsigned char *c, unsigned long long clen, const unsigned char *ad, unsigned long long adlen, const unsigned char *npub, const unsigned char *k);
void crypto_aead_chacha20poly1305_ietf_keygen(unsigned char k[32]);
// KEYBYTES=32, NPUBBYTES=12, ABYTES=16
```

### Public-Key Encryption (Curve25519 + XSalsa20 + Poly1305)

```c
int crypto_box_keypair(unsigned char *pk, unsigned char *sk);
int crypto_box_easy(unsigned char *c, const unsigned char *m, unsigned long long mlen, const unsigned char *n, const unsigned char *pk, const unsigned char *sk);
int crypto_box_open_easy(unsigned char *m, const unsigned char *c, unsigned long long clen, const unsigned char *n, const unsigned char *pk, const unsigned char *sk);
```

### Secret-Key Encryption (XSalsa20 + Poly1305)

```c
int crypto_secretbox_easy(unsigned char *c, const unsigned char *m, unsigned long long mlen, const unsigned char *n, const unsigned char *k);
int crypto_secretbox_open_easy(unsigned char *m, const unsigned char *c, unsigned long long clen, const unsigned char *n, const unsigned char *k);
void crypto_secretbox_keygen(unsigned char k[32]);
```

### Digital Signatures (Ed25519)

```c
int crypto_sign_keypair(unsigned char *pk, unsigned char *sk);
int crypto_sign_detached(unsigned char *sig, unsigned long long *siglen, const unsigned char *m, unsigned long long mlen, const unsigned char *sk);
int crypto_sign_verify_detached(const unsigned char *sig, const unsigned char *m, unsigned long long mlen, const unsigned char *pk);
```

### Hashing (SHA-256, Blake2b)

```c
int crypto_hash_sha256(unsigned char *out, const unsigned char *in, unsigned long long inlen);
int crypto_generichash(unsigned char *out, size_t outlen, const unsigned char *in, unsigned long long inlen, const unsigned char *key, size_t keylen);
```

### Password Hashing (Argon2i)

```c
int crypto_pwhash(unsigned char *out, unsigned long long outlen, const char *passwd, unsigned long long passwdlen, const unsigned char *salt, unsigned long long opslimit, size_t memlimit, int alg);
int crypto_pwhash_str(char out[128], const char *passwd, unsigned long long passwdlen, unsigned long long opslimit, size_t memlimit);
int crypto_pwhash_str_verify(const char str[128], const char *passwd, unsigned long long passwdlen);
```

### Key Exchange (X25519 + Blake2b)

```c
int crypto_kx_keypair(unsigned char pk[32], unsigned char sk[32]);
int crypto_kx_client_session_keys(unsigned char rx[32], unsigned char tx[32], const unsigned char client_pk[32], const unsigned char client_sk[32], const unsigned char server_pk[32]);
int crypto_kx_server_session_keys(unsigned char rx[32], unsigned char tx[32], const unsigned char server_pk[32], const unsigned char server_sk[32], const unsigned char client_pk[32]);
```

### Utilities

```c
void sodium_memzero(void *pnt, size_t len);
int sodium_memcmp(const void *b1, const void *b2, size_t len);
char *sodium_bin2hex(char *hex, size_t hex_maxlen, const unsigned char *bin, size_t bin_len);
int sodium_hex2bin(unsigned char *bin, size_t bin_maxlen, const char *hex, size_t hex_len, const char *ignore, size_t *bin_len, const char **hex_end);
void *sodium_malloc(size_t size);
void sodium_free(void *ptr);
```

---

## 39. wolfSSL

**Component**: `esp-wolfssl` | **Header**: `user_settings.h`

ESP8266-specific configuration enables: ECC, AES-ECB, AES Direct, OpenSSL compatibility layer, FreeRTOS, small stack usage. Disables: DH, MD4, 3DES, DSA, RC4, RABBIT. Uses `os_get_random` for PRNG. Standard wolfSSL/OpenSSL-compat APIs available when linked.

---

## 40. Logging

**Component**: `log` | **Header**: `esp_log.h`

```c
typedef enum { ESP_LOG_NONE, ESP_LOG_ERROR, ESP_LOG_WARN, ESP_LOG_INFO, ESP_LOG_DEBUG, ESP_LOG_VERBOSE } esp_log_level_t;

void esp_log_level_set(const char *tag, esp_log_level_t level);
uint32_t esp_log_timestamp(void);

// Logging Macros (use these)
ESP_LOGE(tag, format, ...);   // Error (red)
ESP_LOGW(tag, format, ...);   // Warning (yellow)
ESP_LOGI(tag, format, ...);   // Info (green)
ESP_LOGD(tag, format, ...);   // Debug
ESP_LOGV(tag, format, ...);   // Verbose

// Buffer logging
ESP_LOG_BUFFER_HEX_LEVEL(tag, buffer, len, level);
ESP_LOG_BUFFER_HEXDUMP(tag, buffer, len, level);

// Early logging (before scheduler starts)
ESP_EARLY_LOGE / ESP_EARLY_LOGW / ESP_EARLY_LOGI(tag, format, ...);
```

---

## 41. Console

**Component**: `console` | **Header**: `esp_console.h`

Interactive CLI framework. Register commands, parse arguments, and run a REPL over UART.

---

## 42. Heap Memory

**Component**: `heap` | **Header**: `esp_heap_caps.h`

```c
#define MALLOC_CAP_EXEC     (1<<0)
#define MALLOC_CAP_32BIT    (1<<1)
#define MALLOC_CAP_8BIT     (1<<2)
#define MALLOC_CAP_DMA      (1<<3)
#define MALLOC_CAP_INTERNAL (1<<11)
#define MALLOC_CAP_SPIRAM   (1<<10)

void *heap_caps_malloc(size_t size, uint32_t caps);
void heap_caps_free(void *ptr);
void *heap_caps_calloc(size_t n, size_t size, uint32_t caps);
void *heap_caps_realloc(void *ptr, size_t size, uint32_t caps);
void *heap_caps_zalloc(size_t size, uint32_t caps);
size_t heap_caps_get_free_size(uint32_t caps);
size_t heap_caps_get_minimum_free_size(uint32_t caps);
```

---

## 43. Ring Buffer

**Component**: `esp_ringbuf` | **Header**: `freertos/ringbuf.h`

```c
typedef enum { RINGBUF_TYPE_NOSPLIT, RINGBUF_TYPE_ALLOWSPLIT, RINGBUF_TYPE_BYTEBUF } ringbuf_type_t;

RingbufHandle_t xRingbufferCreate(size_t buf_length, ringbuf_type_t type);
BaseType_t xRingbufferSend(RingbufHandle_t ringbuf, const void *data, size_t data_size, TickType_t ticks_to_wait);
void *xRingbufferReceive(RingbufHandle_t ringbuf, size_t *item_size, TickType_t ticks_to_wait);
void *xRingbufferReceiveUpTo(RingbufHandle_t ringbuf, size_t *item_size, TickType_t ticks_to_wait, size_t max_size);
void vRingbufferReturnItem(RingbufHandle_t ringbuf, void *item);
void vRingbufferDelete(RingbufHandle_t ringbuf);
size_t xRingbufferGetCurFreeSize(RingbufHandle_t ringbuf);
// ISR variants: xRingbufferSendFromISR, xRingbufferReceiveFromISR
```

---

## 44. cJSON

**Component**: `json` | **Header**: `cJSON.h`

```c
cJSON *cJSON_Parse(const char *value);
char *cJSON_Print(const cJSON *item);
char *cJSON_PrintUnformatted(const cJSON *item);
void cJSON_Delete(cJSON *c);

// Access
cJSON *cJSON_GetObjectItem(const cJSON *object, const char *string);
cJSON *cJSON_GetArrayItem(const cJSON *array, int index);
int cJSON_GetArraySize(const cJSON *array);
char *cJSON_GetStringValue(cJSON *item);
cJSON_bool cJSON_HasObjectItem(const cJSON *object, const char *string);

// Type Checking
cJSON_bool cJSON_IsString / IsNumber / IsObject / IsArray / IsBool / IsNull / IsTrue / IsFalse(const cJSON *item);

// Creation
cJSON *cJSON_CreateObject / CreateArray / CreateString(const char *) / CreateNumber(double) / CreateBool(cJSON_bool) / CreateNull(void);

// Manipulation
cJSON_bool cJSON_AddItemToObject(cJSON *object, const char *string, cJSON *item);
cJSON_bool cJSON_AddItemToArray(cJSON *array, cJSON *item);
cJSON *cJSON_AddStringToObject(cJSON *object, const char *name, const char *string);
cJSON *cJSON_AddNumberToObject(cJSON *object, const char *name, double number);
cJSON *cJSON_AddBoolToObject(cJSON *object, const char *name, cJSON_bool boolean);
cJSON *cJSON_Duplicate(const cJSON *item, cJSON_bool recurse);
void cJSON_Minify(char *json);
```

---

## 45. JSMN

**Component**: `jsmn` | **Header**: `jsmn.h`

```c
typedef enum { JSMN_UNDEFINED, JSMN_OBJECT, JSMN_ARRAY, JSMN_STRING, JSMN_PRIMITIVE } jsmntype_t;
typedef struct { jsmntype_t type; int start, end, size; int parent; } jsmntok_t;

void jsmn_init(jsmn_parser *parser);
int jsmn_parse(jsmn_parser *parser, const char *js, size_t len, jsmntok_t *tokens, unsigned int num_tokens);
```

---

## 46. HTTP Parser

**Component**: `http_parser` | **Header**: `http_parser.h`

Low-level HTTP/1.1 parser (callback-based). Supports all HTTP methods. Used internally by `esp_http_client` and `esp_http_server`.

---

## 47. OTA / App Update

**Component**: `app_update` | **Header**: `esp_ota_ops.h`

```c
esp_err_t esp_ota_begin(const esp_partition_t *partition, size_t image_size, esp_ota_handle_t *out_handle);
esp_err_t esp_ota_write(esp_ota_handle_t handle, const void *data, size_t size);
esp_err_t esp_ota_end(esp_ota_handle_t handle);
esp_err_t esp_ota_set_boot_partition(const esp_partition_t *partition);
const esp_partition_t *esp_ota_get_boot_partition(void);
const esp_partition_t *esp_ota_get_running_partition(void);
const esp_partition_t *esp_ota_get_next_update_partition(const esp_partition_t *start_from);
const esp_app_desc_t *esp_ota_get_app_description(void);
esp_err_t esp_ota_get_partition_description(const esp_partition_t *partition, esp_app_desc_t *app_desc);
```

---

## 48. Bootloader Support

**Component**: `bootloader_support` | **Headers**: `esp_app_format.h`, `esp_image_format.h`, `esp_flash_data_types.h`

```c
// App descriptor (256 bytes, embedded in firmware)
typedef struct {
    uint32_t magic_word;           // ESP_APP_DESC_MAGIC_WORD (0xABCD5432)
    uint32_t secure_version;
    char version[32], project_name[32], time[16], date[16], idf_ver[32];
    uint8_t app_elf_sha256[32];
} esp_app_desc_t;

// OTA selection
uint32_t bootloader_common_ota_select_crc(const esp_ota_select_entry_t *s);
bool bootloader_common_ota_select_valid(const esp_ota_select_entry_t *s);
```

---

## 49. Protocol Communication

**Component**: `protocomm` | **Header**: `protocomm.h`

```c
protocomm_t *protocomm_new(void);
void protocomm_delete(protocomm_t *pc);
esp_err_t protocomm_add_endpoint(protocomm_t *pc, const char *ep_name, protocomm_req_handler_t handler, void *priv_data);
esp_err_t protocomm_remove_endpoint(protocomm_t *pc, const char *ep_name);
esp_err_t protocomm_set_security(protocomm_t *pc, const char *ep_name, const protocomm_security_t *sec, const protocomm_security_pop_t *pop);
```

---

## 50. WiFi Provisioning

**Component**: `wifi_provisioning` | **Header**: `wifi_provisioning/manager.h`

Managed WiFi provisioning framework. Supports BLE and SoftAP transport. Events: `WIFI_PROV_INIT`, `WIFI_PROV_START`, `WIFI_PROV_CRED_RECV`, `WIFI_PROV_CRED_FAIL`, `WIFI_PROV_CRED_SUCCESS`, `WIFI_PROV_END`, `WIFI_PROV_DEINIT`.

---

## 51. Modbus

**Component**: `freemodbus` | **Header**: `esp_modbus_master.h`, `esp_modbus_common.h`

```c
esp_err_t mbc_master_init(mb_port_type_t port_type, void **handler);
esp_err_t mbc_master_init_tcp(void **handler);
esp_err_t mbc_master_destroy(void);
// Supports: RTU, ASCII, TCP modes
// Register types: HOLDING, INPUT, COIL, DISCRETE
```

---

## 52. Common Types & Errors

**Component**: `esp_common` | **Header**: `esp_err.h`

```c
typedef int32_t esp_err_t;

#define ESP_OK          0
#define ESP_FAIL        (-1)
#define ESP_ERR_NO_MEM          0x101
#define ESP_ERR_INVALID_ARG     0x102
#define ESP_ERR_INVALID_STATE   0x103
#define ESP_ERR_INVALID_SIZE    0x104
#define ESP_ERR_NOT_FOUND       0x105
#define ESP_ERR_NOT_SUPPORTED   0x106
#define ESP_ERR_TIMEOUT         0x107
#define ESP_ERR_INVALID_RESPONSE 0x108
#define ESP_ERR_INVALID_CRC     0x109
#define ESP_ERR_INVALID_MAC     0x10B
#define ESP_ERR_WIFI_BASE       0x3000
#define ESP_ERR_MESH_BASE       0x4000

const char *esp_err_to_name(esp_err_t code);
ESP_ERROR_CHECK(x)    // Abort on error
```

### Bit Definitions (`esp_bit_defs.h`)

```c
BIT0 through BIT31    // (1 << n)
BIT(nr)               // Runtime bit macro
```

---

## 53. POSIX Threads

**Component**: `pthread` | **Header**: `esp_pthread.h`

```c
typedef struct { size_t stack_size; size_t prio; bool inherit_cfg; const char *thread_name; int pin_to_core; } esp_pthread_cfg_t;
esp_pthread_cfg_t esp_pthread_get_default_config(void);
esp_err_t esp_pthread_set_cfg(const esp_pthread_cfg_t *cfg);
esp_err_t esp_pthread_get_cfg(esp_pthread_cfg_t *cfg);
// Standard POSIX: pthread_create, pthread_join, pthread_mutex_*, pthread_cond_*
```

---

## 54. CRC & SHA Utilities

**Component**: `esp8266` | **Headers**: `esp_crc.h`, `esp_sha.h`

### CRC

```c
uint32_t crc32_le(uint32_t crc, const uint8_t *buf, uint32_t len);
uint32_t crc32_be(uint32_t crc, const uint8_t *buf, uint32_t len);
uint16_t crc16_le / crc16_be(uint16_t crc, const uint8_t *buf, uint32_t len);
uint8_t crc8_le / crc8_be(uint8_t crc, const uint8_t *buf, uint32_t len);
```

### SHA

```c
// SHA-1, SHA-224, SHA-256, SHA-384, SHA-512
int esp_sha256_init(esp_sha256_t *ctx);
int esp_sha256_update(esp_sha256_t *ctx, const void *src, size_t size);
int esp_sha256_finish(esp_sha256_t *ctx, void *dest);
// Same pattern for sha1, sha224, sha384, sha512
```

---

## 55. PHY & RF Calibration

**Component**: `esp8266` | **Header**: `esp_phy_init.h`

```c
const esp_phy_init_data_t *esp_phy_get_init_data(void);
void esp_phy_release_init_data(const esp_phy_init_data_t *data);
esp_err_t esp_phy_rf_init(const esp_phy_init_data_t *init_data, esp_phy_calibration_mode_t mode, esp_phy_calibration_data_t *cal_data, phy_rf_module_t module);
esp_err_t esp_phy_rf_deinit(phy_rf_module_t module);
void esp_phy_load_cal_and_init(phy_rf_module_t module);
esp_err_t esp_phy_load_cal_data_from_nvs(esp_phy_calibration_data_t *out_cal_data);
esp_err_t esp_phy_store_cal_data_to_nvs(const esp_phy_calibration_data_t *cal_data);
// Calibration modes: PHY_RF_CAL_PARTIAL, PHY_RF_CAL_NONE, PHY_RF_CAL_FULL
```

---

## Quick Reference: Typical Initialization Sequence

```c
void app_main(void) {
    // 1. NVS
    nvs_flash_init();

    // 2. TCP/IP & Event Loop
    tcpip_adapter_init();
    esp_event_loop_create_default();

    // 3. WiFi
    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    esp_wifi_init(&cfg);
    esp_event_handler_register(WIFI_EVENT, ESP_EVENT_ANY_ID, &wifi_handler, NULL);
    esp_event_handler_register(IP_EVENT, IP_EVENT_STA_GOT_IP, &ip_handler, NULL);

    wifi_config_t wifi_config = { .sta = { .ssid = "...", .password = "..." } };
    esp_wifi_set_mode(WIFI_MODE_STA);
    esp_wifi_set_config(ESP_IF_WIFI_STA, &wifi_config);
    esp_wifi_start();
    esp_wifi_connect();
}
```

---

*End of ESP8266 RTOS SDK Reference*
