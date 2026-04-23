# ESP-IDF v6.0 SDK — Full Indexed Reference

Comprehensive per-component reference for ESP-IDF at `/opt/esp/idf/` (v6.0.0), focused on ESP32-C3 but chip-agnostic where APIs are generic. Intended as an indexed lookup for future sessions — scan the table of contents, jump to the relevant section.

- **SDK root:** `/opt/esp/idf/`
- **Components dir:** `/opt/esp/idf/components/` (~110 components)
- **Examples dir:** `/opt/esp/idf/examples/`
- **Target chip (this project):** ESP32-C3 (single-core RISC-V RV32IMC, BLE 5.0, 400 KB SRAM, no PSRAM / camera / LCD / touch / DAC / 802.15.4)
- **Build tool:** `idf.py` (wraps CMake + Ninja + esptool)

## Table of contents

- [0. Orientation](#0-orientation)
- [1. Core system & runtime](#1-core-system--runtime)
- [2. Digital peripheral drivers](#2-digital-peripheral-drivers)
- [3. Networking stack](#3-networking-stack)
- [4. Storage & filesystems](#4-storage--filesystems)
- [5. Bluetooth & wireless](#5-bluetooth--wireless)
- [6. Analog / display / imaging peripherals](#6-analog--display--imaging-peripherals)
- [7. Security, boot & OTA](#7-security-boot--ota)
- [8. Utilities & dev-tools](#8-utilities--dev-tools)
- [9. Examples directory map](#9-examples-directory-map)
- [10. Tools directory map](#10-tools-directory-map)

---

## 0. Orientation

### 0.1 Component map (by domain)

**Core system:** `esp_system` · `freertos` · `heap` · `log` · `esp_timer` · `esp_event` · `esp_pm` · `pthread` · `esp_common` · `esp_rom` · `esp_libc` · `esp_stdio` · `esp_hw_support` · `esp_mm` · `esp_psram` · `riscv` · `xtensa` · `soc` · `hal` · `rt` · `perfmon`.

**Digital peripheral drivers (v6 `esp_driver_*`):** `esp_driver_gpio` · `esp_driver_spi` · `esp_driver_i2c` · `esp_driver_i2s` · `esp_driver_i3c` · `esp_driver_uart` · `esp_driver_ledc` · `esp_driver_rmt` · `esp_driver_gptimer` · `esp_driver_pcnt` · `esp_driver_mcpwm` · `esp_driver_parlio` · `esp_driver_twai` · `esp_driver_sdm` · `esp_driver_ana_cmpr` · `esp_driver_dma` · `esp_driver_bitscrambler` · `esp_driver_usb_serial_jtag` · `driver` (legacy umbrella).

**Analog / display / imaging:** `esp_adc` · `esp_driver_dac` · `esp_driver_touch_sens` · `esp_driver_tsens` · `esp_lcd` · `esp_driver_cam` · `esp_driver_isp` · `esp_driver_jpeg` · `esp_driver_ppa`.

**Networking:** `esp_wifi` · `esp_netif` · `esp_netif_stack` · `lwip` · `esp_eth` · `esp_http_client` · `esp_http_server` · `esp_https_server` · `esp_https_ota` · `mqtt` · `esp-tls` · `mbedtls` · `tcp_transport` · `http_parser` · `protobuf-c` · `protocomm` · `esp_local_ctrl` · `wpa_supplicant`.

**Storage:** `spi_flash` · `esp_partition` · `partition_table` · `nvs_flash` · `nvs_sec_provider` · `fatfs` · `spiffs` · `vfs` · `wear_levelling` · `sdmmc` · `esp_driver_sdmmc` · `esp_driver_sdspi` · `esp_driver_sdio` · `esp_driver_sd_intf` · `esp_blockdev`.

**Bluetooth / wireless:** `bt` (Bluedroid + NimBLE hosts) · `esp_hid` · `ieee802154` · `openthread` · `esp_coex` · `esp_phy`.

**Security / boot / OTA:** `efuse` · `esp_security` · `esp_tee` · `bootloader` · `bootloader_support` · `esp_app_format` · `esp_bootloader_format` · `app_update` · `esp_https_ota`.

**Utilities / dev-tools:** `console` · `esp_ringbuf` · `unity` · `cmock` · `cxx` · `ulp` · `app_trace` · `esp_gdbstub` · `espcoredump` · `esp_trace` · `esp_usb_cdc_rom_console` · `linux` · `esptool_py` · `idf_test`.

**HAL layer:** `hal` + `esp_hal_*` per-peripheral — low-level inline functions consumed by drivers; rarely called by app code.

### 0.2 ESP32-C3 capability quick-reference

| Peripheral | Count / note |
|---|---|
| CPU | 1× RISC-V RV32IMC @ up to 160 MHz |
| SRAM | 400 KB (16 KB cache) |
| Flash | external, up to 16 MB |
| PSRAM | **not supported** |
| GPIO | 22 pins (GPIO0–GPIO21) |
| ADC | 2 units, 12-bit SAR; ADC1: GPIO0–4, ADC2: GPIO5 (avoid with Wi-Fi) |
| SPI | SPI2/FSPI general-purpose (SPI0/1 are for flash) |
| I2C | 1 controller |
| UART | 2 |
| LEDC | 6 channels, 1 speed mode, up to 14-bit resolution |
| RMT | 4 channels (2 TX + 2 RX) |
| GPTimer | 2 |
| PCNT | — |
| MCPWM | — |
| I2S | 1 |
| TWAI (CAN) | 1 |
| USB | USB-Serial-JTAG only |
| Wi-Fi | 2.4 GHz 802.11 b/g/n |
| Bluetooth | BLE 5.0 (no Classic) |
| 802.15.4 | — |
| Touch sensor | — |
| LCD/camera/JPEG/ISP/PPA | — |
| Secure boot | V2 (RSA-PSS / ECDSA) |
| Flash encryption | AES-XTS-128 |

Full list: `/opt/esp/idf/components/soc/esp32c3/include/soc/soc_caps.h`.

### 0.3 "Where do I look?" quick table

| I need to… | Component | Header |
|---|---|---|
| Spawn a task | `freertos` | `freertos/FreeRTOS.h`, `freertos/task.h` |
| Allocate from a specific memory region | `heap` | `esp_heap_caps.h` |
| Log | `log` | `esp_log.h` |
| One-shot / periodic SW timer | `esp_timer` | `esp_timer.h` |
| Broadcast / subscribe to events | `esp_event` | `esp_event.h` |
| Drive a GPIO or attach an ISR | `esp_driver_gpio` | `driver/gpio.h` |
| SPI master to a radio/display | `esp_driver_spi` | `driver/spi_master.h` |
| I2C master (new API) | `esp_driver_i2c` | `driver/i2c_master.h` |
| UART | `esp_driver_uart` | `driver/uart.h` |
| PWM via LEDC | `esp_driver_ledc` | `driver/ledc.h` |
| Precise waveforms (IR, WS2812) | `esp_driver_rmt` | `driver/rmt_tx.h`, `driver/rmt_rx.h` |
| Hardware timer w/ alarm | `esp_driver_gptimer` | `driver/gptimer.h` |
| CAN / TWAI | `esp_driver_twai` | `esp_twai.h`, `esp_twai_onchip.h` |
| Read ADC | `esp_adc` | `esp_adc/adc_oneshot.h`, `esp_adc/adc_continuous.h` |
| Die-temperature sensor | `esp_driver_tsens` | `driver/temperature_sensor.h` |
| Persist key/value | `nvs_flash` | `nvs.h`, `nvs_flash.h` |
| Read/write a partition | `esp_partition` | `esp_partition.h` |
| Mount FAT on SPI flash | `fatfs` + `wear_levelling` | `esp_vfs_fat.h` |
| Mount SPIFFS | `spiffs` | `esp_spiffs.h` |
| Wi-Fi STA connect | `esp_wifi` + `esp_netif` + `nvs_flash` | `esp_wifi.h`, `esp_netif.h` |
| HTTP client | `esp_http_client` | `esp_http_client.h` |
| HTTP server | `esp_http_server` | `esp_http_server.h` |
| MQTT client | `mqtt` | `mqtt_client.h` |
| TLS sockets | `esp-tls` | `esp_tls.h` |
| OTA | `app_update` / `esp_https_ota` | `esp_ota_ops.h`, `esp_https_ota.h` |
| BLE (recommended on C3) | `bt` (NimBLE) | `host/ble_hs.h`, `host/ble_gap.h`, `host/ble_gatt.h` |
| Sleep | `esp_hw_support` | `esp_sleep.h`, `esp_pm.h` |
| eFuse / MAC | `efuse` / `esp_hw_support` | `esp_efuse.h`, `esp_mac.h` |
| Crash dump to flash | `espcoredump` | `esp_core_dump.h` |

### 0.4 Build & flash cheat-sheet

```bash
. /opt/esp/idf/export.sh           # activate environment
idf.py set-target esp32c3
idf.py menuconfig                  # edit Kconfig
idf.py build
idf.py -p /dev/ttyUSB0 flash
idf.py -p /dev/ttyUSB0 monitor     # Ctrl-] to exit
idf.py size / size-components / size-files
idf.py erase-flash
```

Build outputs: `build/bootloader/bootloader.bin`, `build/partition_table/partition-table.bin`, `build/<project>.bin`, `build/<project>.elf`.

### 0.5 Components actually pulled in by this project

From [main/CMakeLists.txt](main/CMakeLists.txt), [main/idf_component.yml](main/idf_component.yml), [CMakeLists.txt](CMakeLists.txt):

- `spi_flash` (direct `PRIV_REQUIRES`)
- `jgromes/radiolib ^7.6.0` (managed component — C++ SX1276/SX1262/CC1101 family driver)
- Transitive via RadioLib: `driver` / `esp_driver_spi` / `esp_driver_gpio` / `freertos` / `log`
- Top-level enables `MINIMAL_BUILD` — only the above plus their transitive closure are compiled.

---

## 1. Core system & runtime

### 1.1 esp_system — init, reset, shutdown, watchdog

**Headers:** `esp_system.h`, `esp_task_wdt.h`, `esp_task.h`, `esp_cpu.h`, `esp_chip_info.h`, `esp_mac.h`, `esp_random.h`, `esp_int_wdt.h`.

| Function | Purpose |
|---|---|
| `esp_restart(void)` | Software restart; does not return |
| `esp_reset_reason(void)` | Last reset reason (`ESP_RST_POWERON`, `ESP_RST_SW`, `ESP_RST_PANIC`, `ESP_RST_WDT`, `ESP_RST_BROWNOUT`, `ESP_RST_DEEPSLEEP`, …) |
| `esp_register_shutdown_handler(handle)` | Run callback before restart |
| `esp_unregister_shutdown_handler(handle)` |  |
| `esp_get_free_heap_size()` | Total free heap bytes |
| `esp_get_free_internal_heap_size()` | Internal (non-PSRAM) free heap |
| `esp_get_minimum_free_heap_size()` | Historical minimum |
| `esp_system_abort(details)` | Panic with `details` string |
| `esp_chip_info(&info)` | Fill `{model, revision, cores, features}` |
| `esp_read_mac(mac, type)` | `ESP_MAC_WIFI_STA/AP/BT/ETH/EFUSE_FACTORY/EFUSE_CUSTOM` |
| `esp_base_mac_addr_set(mac)` / `_get(mac)` | Override base MAC |
| `esp_random()` / `esp_fill_random(buf, len)` | HW TRNG (full entropy only with Wi-Fi or BT enabled) |

**Task Watchdog (TWDT):**
`esp_task_wdt_config_t { timeout_ms, idle_core_mask, trigger_panic }` →
`esp_task_wdt_init/reconfigure/deinit`,
`esp_task_wdt_add(task | NULL)`, `esp_task_wdt_add_user(name, &uh)`,
`esp_task_wdt_reset()`, `esp_task_wdt_reset_user(uh)`,
`esp_task_wdt_delete(task)`, `esp_task_wdt_delete_user(uh)`,
`esp_task_wdt_status(task)`, `esp_task_wdt_print_triggered_tasks(print_fn, ctx, …)`.

**Kconfig highlights:**
`ESP_SYSTEM_PANIC_{PRINT_REBOOT|PRINT_HALT|SILENT_REBOOT|GDBSTUB}`,
`ESP_INT_WDT`, `ESP_INT_WDT_TIMEOUT_MS`,
`ESP_TASK_WDT_EN`, `ESP_TASK_WDT_TIMEOUT_S`, `ESP_TASK_WDT_CHECK_IDLE_TASK_CPU0`,
`ESP_MAIN_TASK_STACK_SIZE`, `ESP_CONSOLE_{UART_DEFAULT|USB_SERIAL_JTAG|NONE}`.

### 1.2 freertos — tasks, IPC, timers

**Headers:** `freertos/FreeRTOS.h`, `freertos/task.h`, `freertos/queue.h`, `freertos/semphr.h`, `freertos/event_groups.h`, `freertos/stream_buffer.h`, `freertos/message_buffer.h`, `freertos/timers.h`, `freertos/portmacro.h`, `freertos/idf_additions.h`.

**Task management:**
`xTaskCreate(fn, name, stack_words, arg, prio, &h)`,
`xTaskCreatePinnedToCore(fn, name, stack_words, arg, prio, &h, core)`,
`xTaskCreateStatic(…)`, `vTaskDelete(h|NULL)`,
`vTaskDelay(ticks)`, `xTaskDelayUntil(&lastWake, period)`,
`vTaskSuspend(h)`, `vTaskResume(h)`,
`uxTaskGetStackHighWaterMark(h|NULL)`, `eTaskGetState(h)`,
`vTaskPrioritySet(h, prio)` / `uxTaskPriorityGet(h)`,
`xTaskGetTickCount()` / `xTaskGetTickCountFromISR()`,
`vTaskGetRunTimeStats(buf)` (requires `FREERTOS_GENERATE_RUN_TIME_STATS`).

**Task notifications (fast IPC, preferred over queues for single-int signalling):**
`xTaskNotifyGive(h)` / `vTaskNotifyGiveFromISR(h, &hpw)` (counting semaphore style),
`ulTaskNotifyTake(clearOnExit, timeout)`,
`xTaskNotify(h, value, action)`, `xTaskNotifyWait(clearEntry, clearExit, &val, timeout)`.

**Queues:**
`xQueueCreate(len, itemSz)`, `xQueueSend(q, &item, timeout)`, `xQueueReceive(q, &buf, timeout)`,
`xQueueSendFromISR(…, &hpw)`, `xQueueReceiveFromISR(…, &hpw)`,
`uxQueueMessagesWaiting(q)`, `vQueueDelete(q)`.

**Semaphores/mutexes:**
`xSemaphoreCreateBinary/Counting/Mutex/RecursiveMutex`,
`xSemaphoreTake(s, timeout)` / `xSemaphoreGive(s)`,
`xSemaphoreTakeFromISR/GiveFromISR(s, &hpw)`,
`xSemaphoreTakeRecursive/GiveRecursive` for recursive mutexes,
`vSemaphoreDelete`.

**Event groups:**
`xEventGroupCreate()`, `xEventGroupSetBits/ClearBits`,
`xEventGroupWaitBits(eg, bits, clearOnExit, waitAll, timeout)`,
`xEventGroupSetBitsFromISR/ClearBitsFromISR`, `vEventGroupDelete`.

**Stream / message buffers:**
`xStreamBufferCreate(bytes, trigger)` / `Send/Receive` (byte stream),
`xMessageBufferCreate(bytes)` / `Send/Receive` (preserves message boundaries).

**Software timers:**
`xTimerCreate(name, period, autoReload, id, cb)`, `xTimerStart/Stop/Reset/ChangePeriod`, `pvTimerGetTimerID`, `xTimerDelete`.

**Key macros:** `pdMS_TO_TICKS(ms)`, `portTICK_PERIOD_MS`, `portMAX_DELAY`, `pdTRUE/pdFALSE`, `portYIELD_FROM_ISR(hpw)`.

**IDF additions:** `xTaskCreatePinnedToCoreWithCaps(…, caps)` lets you place stack in specific heap region (e.g. `MALLOC_CAP_SPIRAM`).

**Kconfig:** `FREERTOS_HZ` (default 100), `FREERTOS_UNICORE` (yes on C3), `FREERTOS_IDLE_TASK_STACKSIZE`, `FREERTOS_THREAD_LOCAL_STORAGE_POINTERS`, `FREERTOS_CHECK_STACKOVERFLOW_{NONE|CANARY|PTRVAL}`, `FREERTOS_USE_TRACE_FACILITY`, `FREERTOS_GENERATE_RUN_TIME_STATS`.

**Gotchas:** stack size in `xTaskCreate` is WORDS (4 bytes). Priority 0 = idle, higher wins. Never call blocking FreeRTOS APIs from ISR — use `*FromISR` variants and propagate `HigherPriorityTaskWoken`.

### 1.3 heap — capability-aware allocator

**Headers:** `esp_heap_caps.h`, `esp_heap_caps_init.h`, `esp_heap_trace.h`, `multi_heap.h`.

Capability flags: `MALLOC_CAP_DEFAULT`, `_EXEC`, `_32BIT`, `_8BIT`, `_DMA`, `_DMA_DESC_AHB/AXI`, `_SPIRAM`, `_INTERNAL`, `_IRAM_8BIT`, `_RTCRAM`, `_RETENTION`, `_CACHE_ALIGNED`, `_SIMD`.

APIs:
`heap_caps_malloc(sz, caps)`, `heap_caps_calloc(n, sz, caps)`, `heap_caps_realloc(p, sz, caps)`,
`heap_caps_aligned_alloc(align, sz, caps)`, `heap_caps_free(p)`,
`heap_caps_get_free_size(caps)`, `heap_caps_get_largest_free_block(caps)`, `heap_caps_get_minimum_free_size(caps)`,
`heap_caps_get_info(&info, caps)`, `heap_caps_print_heap_info(caps)`, `heap_caps_check_integrity_all(print)`,
`heap_caps_register_failed_alloc_callback(cb)`.

Tracing: `heap_trace_init_standalone(buf, n)`, `heap_trace_start(HEAP_TRACE_LEAKS|HEAP_TRACE_ALL)`, `heap_trace_stop()`, `heap_trace_dump()`, `heap_trace_summary(&s)`.

Kconfig: `HEAP_POISONING_{DISABLED|LIGHT|COMPREHENSIVE}`, `HEAP_TRACING_{OFF|STANDALONE|TOHOST}`, `HEAP_USE_HOOKS`, `HEAP_CORRUPTION_DETECTION`.

C3 has no PSRAM → `MALLOC_CAP_SPIRAM` always fails.

### 1.4 log — tagged logging

**Headers:** `esp_log.h`, `esp_log_level.h`, `esp_log_buffer.h`, `esp_log_timestamp.h`, `esp_log_color.h`, `esp_log_write.h`.

Macros: `ESP_LOGE/W/I/D/V(tag, fmt, ...)`, `ESP_EARLY_LOG*` (pre-heap / ISR-ish), `ESP_DRAM_LOG*` (cache-disabled; needs `DRAM_STR("…")`). Level enum: `ESP_LOG_{NONE,ERROR,WARN,INFO,DEBUG,VERBOSE}`.

Runtime control: `esp_log_level_set(tag, level)` (wildcards with `"*"`), `esp_log_level_get(tag)`, `esp_log_set_vprintf(fn)`, `esp_log_system_timestamp()`, `esp_log_timestamp()`, `esp_log_buffer_hex/hexdump/char(tag, buf, len, level)`.

Kconfig: `LOG_DEFAULT_LEVEL`, `LOG_MAXIMUM_LEVEL` (compile-time ceiling), `LOG_COLORS`, `LOG_TIMESTAMP_SOURCE_{NONE|MILLISECONDS|SYSTEM}`, `LOG_VERSION_2` (runtime formatter, smaller binary).

### 1.5 esp_timer — µs-resolution timers

**Headers:** `esp_timer.h`.

Types: `esp_timer_handle_t`, `esp_timer_cb_t = void(*)(void*)`, `esp_timer_dispatch_t { ESP_TIMER_TASK, ESP_TIMER_ISR }`, `esp_timer_create_args_t { callback, arg, dispatch_method, name, skip_unhandled_events }`.

APIs: `esp_timer_early_init`, `esp_timer_init`, `esp_timer_create`, `esp_timer_start_once(us)`, `esp_timer_start_periodic(us)`, `esp_timer_restart(new_us)`, `esp_timer_stop`, `esp_timer_delete`, `esp_timer_get_time()` (int64 µs since boot), `esp_timer_get_next_alarm()`, `esp_timer_dump(stream)`, `esp_timer_is_active`.

Kconfig: `ESP_TIMER_TASK_STACK_SIZE`, `ESP_TIMER_INTERRUPT_LEVEL`, `ESP_TIMER_ISR_AFFINITY`, `ESP_TIMER_SUPPORTS_ISR_DISPATCH_METHOD`.

### 1.6 esp_event — event loop

**Headers:** `esp_event.h`, `esp_event_base.h`.

Macros: `ESP_EVENT_DEFINE_BASE(NAME)`, `ESP_EVENT_DECLARE_BASE(NAME)`, `ESP_EVENT_ANY_BASE`, `ESP_EVENT_ANY_ID`.

Handler sig: `void (*)(void *arg, esp_event_base_t base, int32_t id, void *event_data)`.

APIs:
`esp_event_loop_create_default/delete_default`,
`esp_event_loop_create(&args, &loop)` / `esp_event_loop_delete(loop)`,
`esp_event_loop_run(loop, ticks)` (manual dispatch mode),
`esp_event_handler_register[_with](base, id, h, arg)`,
`esp_event_handler_unregister[_with](base, id, h)`,
`esp_event_handler_instance_register[_with](base, id, h, arg, &instance)` and `_unregister`,
`esp_event_post[_to](loop, base, id, data, size, ticks)`,
`esp_event_post_from_isr[_to](loop, base, id, data, size, &hpw)`,
`esp_event_dump(FILE*)`.

Config: `esp_event_loop_args_t { queue_size, task_name, task_priority, task_stack_size, task_core_id }`.

### 1.7 esp_pm — power management

**Headers:** `esp_pm.h`, `esp_private/pm_impl.h` (internal).

`esp_pm_config_t { max_freq_mhz, min_freq_mhz, light_sleep_enable }` → `esp_pm_configure(&cfg)`.

Locks (hold = prevent the change): types `ESP_PM_CPU_FREQ_MAX`, `ESP_PM_APB_FREQ_MAX`, `ESP_PM_NO_LIGHT_SLEEP`.
`esp_pm_lock_create(type, arg, name, &h)` / `_acquire` / `_release` / `_delete` / `esp_pm_dump_locks(FILE*)`.

Kconfig: `PM_ENABLE`, `PM_DFS_INIT_AUTO`, `PM_PROFILING`, `PM_SLP_IRAM_OPT`, `PM_RTOS_IDLE_OPT`, `PM_POWER_DOWN_CPU_IN_LIGHT_SLEEP`.

### 1.8 esp_hw_support — sleep, clocks, interrupts, MAC, RNG, spinlocks

**Headers:** `esp_sleep.h`, `esp_clk_tree.h`, `esp_intr_alloc.h`, `esp_cpu.h`, `esp_mac.h`, `esp_random.h`, `esp_chip_info.h`, `spinlock.h`, `esp_crc.h`, `esp_ldo_regulator.h`, `esp_sleep_cpu_retention.h`.

**Sleep:** `esp_sleep_enable_timer_wakeup(us)`, `_enable_gpio_wakeup()`, `_enable_ext0_wakeup(gpio, level)` / `_ext1_wakeup(mask, mode)`, `_enable_uart_wakeup(port)`, `_enable_touchpad_wakeup()`, `_enable_ulp_wakeup()`, `_enable_wifi_wakeup()`, `_enable_bt_wakeup()`.
Enter: `esp_light_sleep_start()` (returns on wake), `esp_deep_sleep_start()` (no return), helper `esp_deep_sleep(us)`.
Query wakeup cause: `esp_sleep_get_wakeup_cause()`, `_get_ext1_wakeup_status()`, `_get_gpio_wakeup_status()`.
Retention control: `esp_sleep_pd_config(ESP_PD_DOMAIN_*, ESP_PD_OPTION_ON/OFF/AUTO)`.

**Interrupts:** `esp_intr_alloc(source, flags, handler, arg, &h)`, `_alloc_intrstatus(…)`, `_free(h)`, `_enable/disable(h)`.
Flags: `ESP_INTR_FLAG_LEVEL{1..6}`, `_EDGE`, `_SHARED`, `_IRAM`, `_INTRDISABLED`, `_NMI`.

**Clock tree:** `esp_clk_tree_src_get_freq_hz(src, precision, &hz)`, `esp_clk_cpu_freq()`, `esp_clk_apb_freq()`, `esp_clk_xtal_freq()`.

**Spinlock:** `spinlock_initialize(&l)`, `spinlock_acquire(&l, timeout)`, `spinlock_release(&l)`. Use for multi-core safety even on C3 (future-proof).

**CPU helpers:** `esp_cpu_get_cycle_count()`, `esp_cpu_set_watchpoint(…)`, `esp_cpu_stall/unstall`, `esp_cpu_reset`.

### 1.9 esp_common — error codes & attributes

**Headers:** `esp_err.h`, `esp_attr.h`, `esp_check.h`, `esp_bit_defs.h`, `esp_assert.h`, `esp_macros.h`.

Codes: `ESP_OK=0`, `ESP_FAIL=-1`, then hex ranges. Key ones: `ESP_ERR_NO_MEM=0x101`, `_INVALID_ARG=0x102`, `_INVALID_STATE=0x103`, `_INVALID_SIZE=0x104`, `_NOT_FOUND=0x105`, `_NOT_SUPPORTED=0x106`, `_TIMEOUT=0x107`, `_INVALID_RESPONSE=0x108`, `_INVALID_CRC=0x109`, `_INVALID_VERSION=0x10A`, `_INVALID_MAC=0x10B`, `_NOT_FINISHED=0x10C`, `_NOT_ALLOWED=0x10D`. Wi-Fi base `0x3000`, Mesh `0x4000`, Flash `0x6000`, eFuse `0x1600`, NVS `0x1100`, HTTPD `0x8000`, etc.

Macros: `ESP_ERROR_CHECK(x)` (abort on fail), `ESP_ERROR_CHECK_WITHOUT_ABORT(x)`, `ESP_RETURN_ON_ERROR(x, tag, fmt, …)`, `ESP_GOTO_ON_ERROR(x, goto_tag, …)`, `ESP_RETURN_ON_FALSE(c, err, tag, …)`.

Placement: `IRAM_ATTR`, `DRAM_ATTR`, `FLASH_ATTR`, `RTC_IRAM_ATTR`, `RTC_DATA_ATTR`, `RTC_NOINIT_ATTR`, `COREDUMP_IRAM_ATTR`, `NOINLINE_ATTR`, `ALWAYS_INLINE_ATTR`.

Function: `esp_err_to_name(code)`.

### 1.10 esp_rom, esp_libc, esp_stdio, esp_mm, esp_psram

- **esp_rom:** `esp_rom_printf`, `esp_rom_delay_us`, `esp_rom_crc32_{le,be}`, `esp_rom_md5_*`, `esp_rom_software_reset_{system,cpu}`. Avoid ROM printf in fast paths — use `ESP_LOGx`.
- **esp_libc:** Newlib-ish C library; `malloc`→heap_caps; `time()`→esp_timer; locale stubs.
- **esp_stdio:** Routes `stdin/stdout/stderr` per `CONFIG_ESP_CONSOLE_*`.
- **esp_mm:** `esp_mmu_map(paddr, size, target, caps, flags, &out_ptr)`, `esp_mmu_unmap(ptr)`, `esp_cache_msync(addr, size, flags)` (DMA/cache coherency).
- **esp_psram:** `esp_psram_init`, `esp_psram_get_size`, `esp_psram_is_initialized`, `esp_psram_get_address(void**)`. **C3: unused.**

### 1.11 soc, hal, riscv, xtensa, rt, perfmon

- **soc:** per-chip register and capability headers under `components/soc/<chip>/include/`. Consult `soc_caps.h` for `SOC_*_SUPPORTED`, channel counts, GPIO lists.
- **hal:** low-level inline helpers (`*_hal_*`, `*_ll_*`) used by drivers. Not an app-facing API.
- **riscv:** CPU utilities for RISC-V chips (C3/C6/H2). Provides vector table, `rv_utils.h` (CSR read/write, interrupt masking), boot entry.
- **xtensa:** equivalent for ESP32/S2/S3.
- **rt:** small runtime glue (abort handler, stack-chk-fail hooks).
- **perfmon:** perf counters — xtensa-centric; limited on RISC-V.

### 1.12 pthread — POSIX threads

**Headers:** `esp_pthread.h`, `pthread.h`, `semaphore.h`, `sched.h`.

Config: `esp_pthread_cfg_t { stack_size, prio, inherit_cfg, thread_name, pin_to_core, stack_alloc_caps }`. APIs: `esp_pthread_init`, `esp_pthread_get_default_config`, `esp_pthread_set_cfg(&cfg)`, `esp_pthread_get_cfg(&cfg)`.

Supports `pthread_create/join/exit/detach/self`, `pthread_mutex_*`, `pthread_cond_*`, `pthread_key_*` (TLS), `pthread_once`, `sem_*`. No `pthread_cancel`.

Kconfig: `PTHREAD_STACK_MIN` (768), `PTHREAD_TASK_STACK_SIZE_DEFAULT` (3072), `PTHREAD_TASK_PRIO_DEFAULT` (5), `PTHREAD_TASK_CORE_DEFAULT`.

### 1.13 Canonical init pattern

```c
#include "esp_system.h"
#include "esp_log.h"
#include "esp_event.h"
#include "nvs_flash.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

static const char *TAG = "app";

void app_main(void) {
    esp_err_t err = nvs_flash_init();
    if (err == ESP_ERR_NVS_NO_FREE_PAGES || err == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ESP_ERROR_CHECK(nvs_flash_init());
    }
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    ESP_LOGI(TAG, "free heap: %" PRIu32, esp_get_free_heap_size());
    xTaskCreate(worker, "worker", 4096, NULL, 5, NULL);
}
```

---

## 2. Digital peripheral drivers

All modern drivers live under `esp_driver_*` and use a handle-based lifecycle: **new → configure → enable → start/transmit → stop/disable → delete**. The legacy `driver/` umbrella still compiles but forwards to the new drivers; many legacy APIs are deprecated in v6.

### 2.1 esp_driver_gpio — general-purpose I/O

**Headers:** `driver/gpio.h`, `driver/gpio_etm.h`, `driver/gpio_filter.h`, `driver/dedic_gpio.h`, `driver/rtc_io.h`, `driver/lp_io.h`.

```c
gpio_config_t io = {
    .pin_bit_mask = 1ULL << PIN,
    .mode = GPIO_MODE_OUTPUT,           // INPUT / OUTPUT / OUTPUT_OD / INPUT_OUTPUT / …
    .pull_up_en = GPIO_PULLUP_DISABLE,
    .pull_down_en = GPIO_PULLDOWN_DISABLE,
    .intr_type = GPIO_INTR_DISABLE,     // POSEDGE / NEGEDGE / ANYEDGE / LOW_LEVEL / HIGH_LEVEL
};
gpio_config(&io);
gpio_set_level(PIN, 1);
int v = gpio_get_level(PIN);
```

**Key APIs:** `gpio_config`, `gpio_reset_pin`, `gpio_set_direction`, `gpio_set_pull_mode`, `gpio_pullup_en/dis`, `gpio_pulldown_en/dis`, `gpio_set_level`, `gpio_get_level`, `gpio_set_intr_type`, `gpio_intr_enable/disable`, `gpio_install_isr_service(flags)`, `gpio_uninstall_isr_service`, `gpio_isr_handler_add(pin, fn, arg)`, `gpio_isr_handler_remove(pin)`, `gpio_wakeup_enable/disable`, `gpio_set_drive_capability(pin, cap)`, `gpio_hold_en/dis`, `gpio_deep_sleep_hold_en/dis`.

**Glitch filter:** `gpio_new_pin_glitch_filter(&cfg, &h)` / `gpio_glitch_filter_enable/disable/delete`.

**Dedicated GPIO (parallel, low-latency):** `dedic_gpio_new_bundle(&cfg, &bundle)`, `dedic_gpio_bundle_write(bundle, mask, val)` — useful for bit-banging at CPU speed.

**ETM (Event Task Matrix):** `gpio_new_etm_event/task(&cfg, &event)` — connect GPIO edges to peripheral actions without CPU.

ESP32-C3 has 22 GPIOs (0–21). Pins 18–19 are USB D-/D+ (used by USB-Serial-JTAG).

### 2.2 esp_driver_spi — SPI master/slave

**Headers:** `driver/spi_master.h`, `driver/spi_slave.h`, `driver/spi_slave_hd.h`, `driver/spi_common.h`.

**Master flow:**
```c
spi_bus_config_t buscfg = {
    .mosi_io_num = MOSI, .miso_io_num = MISO, .sclk_io_num = SCLK,
    .quadwp_io_num = -1, .quadhd_io_num = -1, .max_transfer_sz = 4096,
};
spi_bus_initialize(SPI2_HOST, &buscfg, SPI_DMA_CH_AUTO);

spi_device_interface_config_t devcfg = {
    .clock_speed_hz = 10 * 1000 * 1000,
    .mode = 0, .spics_io_num = CS,
    .queue_size = 7,
    .flags = 0,
    .pre_cb = NULL, .post_cb = NULL,
};
spi_device_handle_t dev;
spi_bus_add_device(SPI2_HOST, &devcfg, &dev);

spi_transaction_t t = { .length = 8 * 3, .tx_buffer = buf, .rx_buffer = buf };
spi_device_transmit(dev, &t);            // blocking
spi_device_polling_transmit(dev, &t);    // non-blocking busy wait (lower overhead small xfers)
spi_device_queue_trans(dev, &t, portMAX_DELAY);
spi_device_get_trans_result(dev, &out_t, portMAX_DELAY);

spi_bus_remove_device(dev);
spi_bus_free(SPI2_HOST);
```

**Key APIs/types:** `spi_host_device_t` (`SPI1_HOST`—flash only, `SPI2_HOST`/`SPI3_HOST`), `spi_bus_initialize/free`, `spi_bus_add_device/remove_device`, `spi_device_acquire_bus/release_bus` (multi-device locking), `spi_device_get_actual_freq`, `spi_device_polling_start/end`.

**Slave:** `spi_slave_interface_config_t`, `spi_slave_initialize/deinitialize`, `spi_slave_transmit/queue_trans/get_trans_result`.

On C3, `SPI2_HOST` is the general-purpose "FSPI" controller.

### 2.3 esp_driver_i2c — new-style I²C

**Headers:** `driver/i2c_master.h`, `driver/i2c_slave.h`, `driver/i2c_types.h`.

```c
i2c_master_bus_config_t mcfg = {
    .i2c_port = I2C_NUM_0,
    .scl_io_num = SCL, .sda_io_num = SDA,
    .clk_source = I2C_CLK_SRC_DEFAULT,
    .glitch_ignore_cnt = 7,
    .flags.enable_internal_pullup = true,
};
i2c_master_bus_handle_t bus;
i2c_new_master_bus(&mcfg, &bus);

i2c_device_config_t dcfg = { .dev_addr_length = I2C_ADDR_BIT_LEN_7, .device_address = 0x50, .scl_speed_hz = 400000 };
i2c_master_dev_handle_t dev;
i2c_master_bus_add_device(bus, &dcfg, &dev);

uint8_t tx[3] = {…}, rx[8];
i2c_master_transmit(dev, tx, sizeof(tx), -1);
i2c_master_receive(dev, rx, sizeof(rx), -1);
i2c_master_transmit_receive(dev, tx, sizeof(tx), rx, sizeof(rx), -1);
i2c_master_probe(bus, addr, -1);

i2c_master_bus_rm_device(dev);
i2c_del_master_bus(bus);
```

**Slave:** `i2c_new_slave_device`, `i2c_slave_transmit`, `i2c_slave_receive`, `i2c_slave_register_event_callbacks`.

**Note:** the old `driver/i2c.h` (`i2c_driver_install`, `i2c_cmd_link_*`) still exists but is **deprecated** — use `i2c_master.h` for new code.

### 2.4 esp_driver_uart — UART

**Headers:** `driver/uart.h`, `driver/uart_select.h`, `driver/uart_vfs.h`, `driver/uart_wakeup.h`, `driver/uhci.h` (DMA-backed UART).

```c
uart_config_t cfg = {
    .baud_rate = 115200, .data_bits = UART_DATA_8_BITS,
    .parity = UART_PARITY_DISABLE, .stop_bits = UART_STOP_BITS_1,
    .flow_ctrl = UART_HW_FLOWCTRL_DISABLE,
    .source_clk = UART_SCLK_DEFAULT,
};
uart_param_config(UART_NUM_1, &cfg);
uart_set_pin(UART_NUM_1, TX, RX, UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE);

QueueHandle_t evtq;
uart_driver_install(UART_NUM_1, rx_buf_sz, tx_buf_sz, 20, &evtq, 0);

uart_write_bytes(UART_NUM_1, data, len);
int n = uart_read_bytes(UART_NUM_1, buf, 64, pdMS_TO_TICKS(20));
uart_flush(UART_NUM_1);
uart_driver_delete(UART_NUM_1);
```

Key APIs: `uart_param_config`, `uart_set_pin`, `uart_driver_install/delete`, `uart_write_bytes[_with_break]`, `uart_read_bytes`, `uart_flush[_input]`, `uart_wait_tx_done`, `uart_set_mode` (RS485 half-duplex, IrDA), `uart_set_rx_timeout`, `uart_set_rx_full_threshold`, `uart_pattern_queue_reset`, `uart_enable_pattern_det_baud_intr`, `uart_enable_rx_intr`, `uart_set_wakeup_threshold`.

Event queue delivers `uart_event_t { type: UART_DATA, UART_BREAK, UART_BUFFER_FULL, UART_FIFO_OVF, UART_FRAME_ERR, UART_PARITY_ERR, UART_PATTERN_DET, size }`.

C3: `UART_NUM_0` (console default), `UART_NUM_1`.

### 2.5 esp_driver_ledc — LED/PWM controller

**Headers:** `driver/ledc.h`, `driver/ledc_etm.h`.

```c
ledc_timer_config_t t = { .speed_mode=LEDC_LOW_SPEED_MODE, .timer_num=LEDC_TIMER_0,
    .duty_resolution=LEDC_TIMER_13_BIT, .freq_hz=5000, .clk_cfg=LEDC_AUTO_CLK };
ledc_timer_config(&t);

ledc_channel_config_t c = { .gpio_num=PIN, .speed_mode=LEDC_LOW_SPEED_MODE,
    .channel=LEDC_CHANNEL_0, .intr_type=LEDC_INTR_DISABLE, .timer_sel=LEDC_TIMER_0, .duty=0, .hpoint=0 };
ledc_channel_config(&c);

ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 4096);
ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);

ledc_fade_func_install(0);
ledc_set_fade_with_time(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 0, 1000);
ledc_fade_start(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, LEDC_FADE_NO_WAIT);
```

Key APIs: `ledc_timer_config`, `ledc_channel_config`, `ledc_set_duty[_and_update][_with_hpoint]`, `ledc_update_duty`, `ledc_get_duty`, `ledc_set_freq`, `ledc_get_freq`, `ledc_timer_pause/resume/rst`, `ledc_bind_channel_timer`, `ledc_stop`, `ledc_set_fade_with_time/step`, `ledc_fade_start/stop`, `ledc_cb_register`, `ledc_isr_register`.

C3: `LEDC_LOW_SPEED_MODE` only; 6 channels (0–5), 4 timers (0–3). Up to `LEDC_TIMER_14_BIT` resolution. Clock sources: `LEDC_USE_APB_CLK`, `_RTC8M_CLK`, `_XTAL_CLK`.

### 2.6 esp_driver_rmt — precise waveforms / IR / WS2812

**Headers:** `driver/rmt_tx.h`, `driver/rmt_rx.h`, `driver/rmt_encoder.h`, `driver/rmt_common.h`, `driver/rmt_types.h`.

```c
rmt_tx_channel_config_t cfg = {
    .gpio_num=PIN, .clk_src=RMT_CLK_SRC_DEFAULT,
    .resolution_hz=10*1000*1000, .mem_block_symbols=64,
    .trans_queue_depth=4, .flags.invert_out=false, .flags.with_dma=false,
};
rmt_channel_handle_t tx;
rmt_new_tx_channel(&cfg, &tx);

rmt_copy_encoder_config_t ecfg = {};
rmt_encoder_handle_t enc;
rmt_new_copy_encoder(&ecfg, &enc);

rmt_enable(tx);
rmt_transmit(tx, enc, symbols, size_bytes, &(rmt_transmit_config_t){ .loop_count = 0 });
rmt_tx_wait_all_done(tx, -1);
rmt_disable(tx);
rmt_del_encoder(enc);
rmt_del_channel(tx);
```

Encoders: `rmt_new_copy_encoder` (raw symbols), `rmt_new_bytes_encoder(&{bit0, bit1, flags})` (per-byte encoding — WS2812 style), `rmt_new_simple_encoder` (custom callback), plus custom encoder via `rmt_encoder_t` vtable.

RX: `rmt_new_rx_channel`, `rmt_rx_register_event_callbacks(ch, &{on_recv_done}, ctx)`, `rmt_receive(ch, buf, sz, &{signal_range_min_ns, signal_range_max_ns})`.

Sync manager: `rmt_new_sync_manager(&{array_size, tx_channel_array}, &mgr)` for synchronous multi-channel TX.

Carrier modulation: `rmt_apply_carrier(ch, &{freq_hz, duty_cycle, flags})` — for IR.

C3: 4 channels, direct channel→direction assignment at `rmt_new_*_channel` time, mem_block_symbols must be multiple of 48.

### 2.7 esp_driver_gptimer — general-purpose hardware timer

**Headers:** `driver/gptimer.h`, `driver/gptimer_etm.h`, `driver/gptimer_types.h`.

```c
gptimer_config_t cfg = { .clk_src=GPTIMER_CLK_SRC_DEFAULT, .direction=GPTIMER_COUNT_UP, .resolution_hz=1000000 };
gptimer_handle_t t;
gptimer_new_timer(&cfg, &t);

gptimer_event_callbacks_t cbs = { .on_alarm = on_alarm_cb };
gptimer_register_event_callbacks(t, &cbs, NULL);

gptimer_alarm_config_t acfg = { .alarm_count = 1000000, .reload_count = 0, .flags.auto_reload_on_alarm = true };
gptimer_set_alarm_action(t, &acfg);

gptimer_enable(t);
gptimer_start(t);
/* … */
gptimer_stop(t);
gptimer_disable(t);
gptimer_del_timer(t);
```

APIs: `gptimer_new_timer/del_timer`, `gptimer_enable/disable`, `gptimer_start/stop`, `gptimer_set_raw_count/get_raw_count`, `gptimer_get_captured_count`, `gptimer_set_alarm_action`, `gptimer_register_event_callbacks`, `gptimer_get_resolution`.

Callback `gptimer_alarm_cb_t(handle, &event_data, user_ctx)` runs in ISR — return bool to indicate whether `HigherPriorityTaskWoken` should trigger yield.

C3: 2 timers.

### 2.8 esp_driver_twai — TWAI / CAN

**Headers:** `esp_twai.h`, `esp_twai_onchip.h`, `esp_twai_types.h`.

v6 introduces a handle-based API:
```c
twai_onchip_node_config_t cfg = {
    .io_cfg = { .tx = TX, .rx = RX }, .bit_timing.bitrate = 500000,
    .data_timing.bitrate = 500000, .tx_queue_depth = 5,
    .fail_retry_cnt = 1, .flags.enable_self_test = false,
};
twai_node_handle_t node;
twai_new_node_onchip(&cfg, &node);

twai_event_callbacks_t cbs = { .on_rx_done = rx_cb, .on_state_change = state_cb };
twai_node_register_event_callbacks(node, &cbs, NULL);

twai_node_enable(node);
twai_node_transmit(node, &(twai_frame_t){ .header.id=0x123, .buffer=data, .buffer_len=8 }, -1);
twai_node_receive_from_isr(node, &frame);   // inside on_rx_done
twai_node_disable(node);
twai_node_delete(node);
```

Bus filtering: `twai_node_config_mask_filter` (ID + mask). Legacy umbrella `driver/twai.h` still works (`twai_driver_install/start/transmit/receive/reconfigure_alerts`) but is deprecated.

C3: 1 TWAI controller.

### 2.9 esp_driver_i2s — I²S audio

**Headers:** `driver/i2s_std.h` (standard Philips/MSB/LSB), `driver/i2s_pdm.h` (PDM mic / speaker), `driver/i2s_tdm.h` (time-division multiplex), `driver/i2s_common.h`, `driver/i2s_etm.h`, `driver/lp_i2s*.h` (low-power, non-C3).

Handle lifecycle (same pattern as other v6 drivers): `i2s_new_channel(&chan_cfg, &tx_h, &rx_h)` → per-mode init `i2s_channel_init_std_mode(h, &std_cfg)` / `_init_pdm_tx_mode` / `_init_pdm_rx_mode` / `_init_tdm_mode` → `i2s_channel_enable(h)` → `i2s_channel_write/read(h, buf, sz, &bytes, tmo)` → `i2s_channel_disable(h)` → `i2s_del_channel(h)`.

Config structs: `i2s_chan_config_t { id, role=MASTER|SLAVE, dma_desc_num, dma_frame_num, auto_clear }`, `i2s_std_config_t { clk_cfg(sample_rate_hz, clk_src, mclk_multiple), slot_cfg(data_bit_width, slot_bit_width, slot_mode=MONO|STEREO, …), gpio_cfg(mclk, bclk, ws, dout, din, invert_flags) }`. Reconfigure on the fly via `i2s_channel_reconfig_std_clock/slot/gpio`.

Event callbacks: `i2s_channel_register_event_callback(h, &{on_recv, on_recv_q_ovf, on_sent, on_send_q_ovf}, ctx)`. DMA-backed; can run with async callback-driven I/O for line-rate audio.

C3: 1 I²S controller (no LP-I2S, no TDM practical limits beyond the controller's channels). Typical use: external I²S DAC / PCM5102 / microphone (INMP441).

### 2.10 esp_driver_pcnt, esp_driver_mcpwm, esp_driver_parlio (not on C3)

- **PCNT** (`driver/pulse_cnt.h`): `pcnt_new_unit/channel`, watch points (`pcnt_unit_add_watch_point`), glitch filter, enable/start/stop. **Not on C3.**
- **MCPWM** (`driver/mcpwm_prelude.h` which pulls in `mcpwm_timer/oper/cmpr/gen/fault/sync/cap.h`): timer → operator → comparator → generator topology; plus capture timer, fault detection, sync bus. **Not on C3.**
- **Parallel IO** (`driver/parlio_tx.h`, `parlio_rx.h`): parallel 1/2/4/8/16-bit TX or RX units; used for high-speed non-standard parallel interfaces. **Not on C3.**

### 2.11 esp_driver_sdm, esp_driver_ana_cmpr

- **Sigma-Delta Modulator** (`driver/sdm.h`): 1-bit DAC-like output; `sdm_new_channel`, `sdm_channel_set_pulse_density`, `sdm_channel_enable/disable`.
- **Analog comparator** (`driver/ana_cmpr.h`): compare analog input to internal ref or external pin; **C3 has one unit on GPIO1**.

### 2.12 esp_driver_dma — GDMA (usually internal to drivers)

**Headers:** `esp_private/gdma.h` (private, but used by drivers that need shared DMA channels: SPI, I2S, RMT, parlio, LCD, crypto).

Key calls: `gdma_new_channel`, `gdma_connect`, `gdma_apply_strategy`, `gdma_set_transfer_ability`, `gdma_register_tx/rx_event_callbacks`. App code rarely touches this directly.

### 2.13 esp_driver_bitscrambler

Hardware "bit scrambler" engine on newer chips — programmable bit-level data transform fed from DMA. **Not on C3.**

### 2.14 esp_driver_usb_serial_jtag — C3 USB console

**Headers:** `driver/usb_serial_jtag.h`, `driver/usb_serial_jtag_vfs.h`.

```c
usb_serial_jtag_driver_config_t cfg = USB_SERIAL_JTAG_DRIVER_CONFIG_DEFAULT();
usb_serial_jtag_driver_install(&cfg);

int n = usb_serial_jtag_write_bytes(data, len, pdMS_TO_TICKS(100));
int r = usb_serial_jtag_read_bytes(buf, sz, pdMS_TO_TICKS(100));

usb_serial_jtag_vfs_use_driver();   // route stdin/stdout through driver
```

On C3, USB-Serial-JTAG is the only native USB interface. Pins GPIO18 (D-) / GPIO19 (D+).

### 2.15 Legacy `driver/` umbrella

Historically housed all peripherals. In v6, many headers forward into the `esp_driver_*` modules. Anything that includes `driver/i2c.h` (legacy cmd-link API), `driver/adc.h`, `driver/dac.h`, `driver/touch_sensor.h`, `driver/can.h` (old TWAI name) is using a deprecated path — prefer the new handle-based drivers.

---

## 3. Networking stack

Stack diagram:

```
          Application
 ┌────────────┬────────────┬──────────┐
 │ http_client│ http_server│   mqtt   │
 └─────┬──────┴─────┬──────┴────┬─────┘
       │            │    tcp_transport (TCP/SSL/WS)
       ▼            ▼            ▼
              esp-tls (TLS abstraction)
                    │
                 mbedtls
                    │
     lwIP sockets / lwIP netif / DNS / DHCP
                    │
                esp_netif (abstraction)
                    │
 ┌──────────────────┼──────────────────┐
 │                  │                  │
esp_wifi         esp_eth           (PPP/SLIP)
 │                  │
 wpa_supplicant   MAC+PHY (internal/SPI modules)
 │
esp_phy / esp_coex
```

### 3.1 esp_wifi — Wi-Fi

**Headers:** `esp_wifi.h`, `esp_wifi_types.h`, `esp_wifi_default.h`, `esp_wifi_netif.h`, `esp_wifi_he.h` (802.11ax), `esp_wifi_he_types.h`, `esp_wifi_ap_get_sta_list.h`, `esp_wifi_crypto_types.h`, `esp_now.h`, `esp_smartconfig.h`, `smartconfig_ack.h`, `esp_mesh.h`, `esp_mesh_internal.h`.

Lifecycle:
```c
wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
esp_wifi_init(&cfg);
esp_wifi_set_storage(WIFI_STORAGE_FLASH);   // or WIFI_STORAGE_RAM
esp_wifi_set_mode(WIFI_MODE_STA);           // STA / AP / APSTA / NAN / NULL

wifi_config_t sta_cfg = {
    .sta = {
        .ssid = "…", .password = "…",
        .threshold.authmode = WIFI_AUTH_WPA2_PSK,
        .pmf_cfg = { .required = false },
    },
};
esp_wifi_set_config(WIFI_IF_STA, &sta_cfg);
esp_wifi_start();
esp_wifi_connect();
```

Key APIs: `esp_wifi_init/deinit`, `esp_wifi_set_mode/get_mode`, `esp_wifi_start/stop`, `esp_wifi_connect/disconnect`, `esp_wifi_set_config/get_config`, `esp_wifi_scan_start/stop`, `esp_wifi_scan_get_ap_num`, `esp_wifi_scan_get_ap_records`, `esp_wifi_sta_get_ap_info`, `esp_wifi_set_ps(WIFI_PS_NONE|MIN_MODEM|MAX_MODEM)`, `esp_wifi_set_country(&cc)`, `esp_wifi_set_channel`, `esp_wifi_set_protocol` (11b/g/n/ax, LR), `esp_wifi_set_bandwidth`, `esp_wifi_set_max_tx_power/get_max_tx_power`, `esp_wifi_set_event_mask`, `esp_wifi_set_vendor_ie`, `esp_wifi_set_promiscuous/rx_cb/filter`, `esp_wifi_ap_get_sta_list`, `esp_wifi_sta_get_rssi`, `esp_wifi_sta_wpa2_ent_*` (enterprise), `esp_wifi_80211_tx` (raw).

Event base: `WIFI_EVENT` with IDs `WIFI_EVENT_STA_START/STOP/CONNECTED/DISCONNECTED/AUTHMODE_CHANGE/GOT_IP` (GOT_IP is on `IP_EVENT`), `WIFI_EVENT_SCAN_DONE`, `WIFI_EVENT_AP_STACONNECTED/STADISCONNECTED/PROBEREQRECVED`, `WIFI_EVENT_HOME_CHANNEL_CHANGE`, `WIFI_EVENT_ROC_DONE`. IP: `IP_EVENT_STA_GOT_IP`, `IP_EVENT_STA_LOST_IP`, `IP_EVENT_AP_STAIPASSIGNED`, `IP_EVENT_GOT_IP6`.

ESP-NOW: `esp_now_init`, `esp_now_register_send_cb/recv_cb`, `esp_now_add_peer`, `esp_now_send(mac, data, len)`, `esp_now_set_pmk`, `esp_now_get_peer`. Max payload 250 B.

Kconfig highlights: `ESP_WIFI_STATIC_RX_BUFFER_NUM`, `_DYNAMIC_RX_BUFFER_NUM`, `_DYNAMIC_TX_BUFFER_NUM`, `_TX_BA_WIN`, `_AMSDU_TX_ENABLED`, `_NVS_ENABLED`, `_SOFTAP_SUPPORT`, `_ENABLE_WPA3_SAE`, `_IRAM_OPT`, `_RX_IRAM_OPT`.

**Gotchas:** NVS must be initialized before `esp_wifi_init()`. Event loop must exist before start. ADC2 unusable while Wi-Fi is on.

### 3.2 esp_netif — netif abstraction

**Headers:** `esp_netif.h`, `esp_netif_defaults.h`, `esp_netif_types.h`, `esp_netif_ip_addr.h`, `esp_netif_ppp.h`, `esp_netif_sntp.h`, `esp_netif_net_stack.h`.

Quickstart:
```c
esp_netif_init();
esp_netif_t *sta = esp_netif_create_default_wifi_sta();
esp_netif_t *ap  = esp_netif_create_default_wifi_ap();
```

Key APIs: `esp_netif_init/deinit`, `esp_netif_new(&cfg)`, `esp_netif_destroy`, `esp_netif_attach`, `esp_netif_set_mac`, `esp_netif_set_hostname/get_hostname`, `esp_netif_set_ip_info/get_ip_info`, `esp_netif_set_dns_info/get_dns_info`, `esp_netif_dhcpc_start/stop/get_status`, `esp_netif_dhcps_start/stop/option`, `esp_netif_get_netif_impl_name`, `esp_netif_get_handle_from_ifkey`, `esp_netif_action_{start,stop,connected,disconnected,got_ip}`, `esp_netif_up/down`, `esp_netif_napt_enable`, `esp_netif_get_flags`.

SNTP (`esp_netif_sntp.h`): `esp_netif_sntp_init(&cfg)`, `esp_netif_sntp_start/stop`, `esp_netif_sntp_sync_wait(timeout)`, config helpers `ESP_NETIF_SNTP_DEFAULT_CONFIG(server)`.

PPP: `esp_netif_ppp_set_auth`, events `NETIF_PPP_STATUS`.

### 3.3 lwip — TCP/IP stack

lwIP provides BSD sockets (`#include "lwip/sockets.h"`) plus: `lwip/netdb.h`, `lwip/dns.h`, `lwip/ip_addr.h`, `lwip/api.h` (netconn API), `ping/ping_sock.h` (esp-extra), `lwip/err.h`.

Common: `socket/bind/listen/accept/connect/send/recv/sendto/recvfrom/close/shutdown`, `getaddrinfo/freeaddrinfo`, `setsockopt(SO_REUSEADDR, SO_RCVTIMEO, SO_KEEPALIVE, TCP_NODELAY, IP_MULTICAST_*)`.

Ping: `esp_ping_new_session(&cfg, &cbs, &handle)`, `esp_ping_start/stop`, `esp_ping_get_profile`.

Kconfig critical: `LWIP_MAX_SOCKETS` (default 10), `LWIP_SO_RCVBUF`, `LWIP_SO_LINGER`, `LWIP_IPV6`, `LWIP_IPV4`, `LWIP_TCP_MSL`, `LWIP_TCP_MSS`, `LWIP_TCP_SND_BUF_DEFAULT`, `LWIP_TCP_WND_DEFAULT`, `LWIP_DHCP_MAX_NTP_SERVERS`, `LWIP_NETIF_API`, `LWIP_TCPIP_CORE_LOCKING`.

### 3.4 esp_eth — Ethernet (SPI modules like W5500 on C3)

**Headers:** `esp_eth.h`, `esp_eth_driver.h`, `esp_eth_mac.h`, `esp_eth_phy.h`, `esp_eth_netif_glue.h`, `esp_eth_mac_spi.h`.

```c
eth_mac_config_t mac_cfg = ETH_MAC_DEFAULT_CONFIG();
eth_phy_config_t phy_cfg = ETH_PHY_DEFAULT_CONFIG();
/* For SPI ethernet (e.g. W5500) on C3: */
eth_w5500_config_t w5500_cfg = ETH_W5500_DEFAULT_CONFIG(SPI2_HOST, &spi_devcfg);
esp_eth_mac_t *mac = esp_eth_mac_new_w5500(&w5500_cfg, &mac_cfg);
esp_eth_phy_t *phy = esp_eth_phy_new_w5500(&phy_cfg);
esp_eth_config_t cfg = ETH_DEFAULT_CONFIG(mac, phy);
esp_eth_handle_t eth;
esp_eth_driver_install(&cfg, &eth);

esp_netif_t *netif = esp_netif_new(&netif_cfg);
esp_netif_attach(netif, esp_eth_new_netif_glue(eth));
esp_eth_start(eth);
```

Supported SPI PHY/MACs: W5500, DM9051, KSZ8851SNL, LAN8720 (via RMII, not on C3), IP101, RTL8201. ESP32-C3 has no internal EMAC → SPI only.

Event base: `ETH_EVENT` with `_START/_STOP/_CONNECTED/_DISCONNECTED`.

### 3.5 esp_http_client

**Headers:** `esp_http_client.h`.

```c
esp_http_client_config_t cfg = {
    .url = "https://example.com/api", .event_handler = http_event,
    .crt_bundle_attach = esp_crt_bundle_attach,
    .method = HTTP_METHOD_GET, .buffer_size = 2048, .timeout_ms = 5000,
};
esp_http_client_handle_t c = esp_http_client_init(&cfg);
esp_http_client_set_header(c, "Authorization", "Bearer …");
esp_http_client_perform(c);
int status = esp_http_client_get_status_code(c);
int64_t len = esp_http_client_get_content_length(c);
esp_http_client_cleanup(c);
```

Key APIs: `esp_http_client_init/cleanup`, `esp_http_client_perform` (blocking) / `_open`+`_write`+`_fetch_headers`+`_read`+`_close` (streaming), `esp_http_client_set_url/method/header/post_field/username/password/authtype/redirection`, `esp_http_client_get_status_code/content_length/header/user_data`, `esp_http_client_is_chunked_response`, `esp_http_client_delete_header`.

Events (via `event_handler`): `HTTP_EVENT_ERROR`, `_ON_CONNECTED`, `_HEADERS_SENT`, `_ON_HEADER`, `_ON_DATA`, `_ON_FINISH`, `_DISCONNECTED`, `_REDIRECT`.

### 3.6 esp_http_server / esp_https_server

**Headers:** `esp_http_server.h`, `esp_https_server.h`.

```c
httpd_handle_t server = NULL;
httpd_config_t cfg = HTTPD_DEFAULT_CONFIG();
httpd_start(&server, &cfg);

httpd_uri_t hello = { .uri="/hello", .method=HTTP_GET, .handler=hello_handler, .user_ctx=NULL };
httpd_register_uri_handler(server, &hello);
```

Handler API: `httpd_req_get_hdr_value_str`, `httpd_req_get_url_query_str`, `httpd_query_key_value`, `httpd_req_recv(req, buf, len)`, `httpd_resp_set_status`, `httpd_resp_set_type`, `httpd_resp_set_hdr`, `httpd_resp_send(req, buf, len)`, `httpd_resp_send_chunk`, `httpd_resp_sendstr_chunk(NULL)` terminator, `httpd_resp_send_err`. Websocket: `httpd_ws_recv_frame`, `httpd_ws_send_frame[_async]`, register handler with `.is_websocket=true, .handle_ws_control_frames=true`. Async: `httpd_req_async_handler_begin/complete`. Sessions: `httpd_sess_get/set_ctx`, `httpd_sess_trigger_close`, LRU session purge.

HTTPS server: `httpd_ssl_config_t` with cert/key arrays, `httpd_ssl_start/stop`.

Kconfig: `HTTPD_MAX_REQ_HDR_LEN`, `HTTPD_MAX_URI_LEN`, `HTTPD_WS_SUPPORT`, `HTTPD_QUEUE_WORK_BLOCKING`, `HTTPD_SERVER_EVENT_POST_TIMEOUT`.

### 3.7 mqtt (esp-mqtt)

**Headers:** `mqtt_client.h` (pulled in via managed component in examples, or component in some builds).

```c
esp_mqtt_client_config_t cfg = {
    .broker.address.uri = "mqtts://broker.example:8883",
    .broker.verification.crt_bundle_attach = esp_crt_bundle_attach,
    .session.keepalive = 60, .credentials.username = "…", .credentials.authentication.password = "…",
};
esp_mqtt_client_handle_t c = esp_mqtt_client_init(&cfg);
esp_mqtt_client_register_event(c, ESP_EVENT_ANY_ID, mqtt_event_handler, NULL);
esp_mqtt_client_start(c);
esp_mqtt_client_publish(c, "topic", "payload", 0, /*qos*/1, /*retain*/0);
esp_mqtt_client_subscribe(c, "other/#", 1);
```

Events: `MQTT_EVENT_CONNECTED`, `_DISCONNECTED`, `_SUBSCRIBED`, `_UNSUBSCRIBED`, `_PUBLISHED`, `_DATA`, `_ERROR`, `_BEFORE_CONNECT`, `_DELETED` (outgoing queue overflow). Supports MQTT 3.1.1 and MQTT 5 (`cfg.session.protocol_ver = MQTT_PROTOCOL_V_5`). Transports: `mqtt://`, `mqtts://`, `ws://`, `wss://`.

### 3.8 esp-tls

**Headers:** `esp_tls.h`, `esp_tls_errors.h`.

```c
esp_tls_cfg_t tls = {
    .crt_bundle_attach = esp_crt_bundle_attach,   // global CA bundle
    .alpn_protos = (const char*[]){ "h2", NULL },
    .skip_common_name = false, .timeout_ms = 5000,
};
esp_tls_t *t = esp_tls_init();
esp_tls_conn_new_sync("example.com", 11, 443, &tls, t);
esp_tls_conn_write(t, buf, len);
esp_tls_conn_read(t, buf, sizeof(buf));
esp_tls_conn_destroy(t);
```

APIs: `esp_tls_init/conn_new_sync/conn_new_async`, `esp_tls_conn_write/read`, `esp_tls_conn_destroy`, `esp_tls_get_and_clear_last_error`, `esp_tls_get_bytes_avail`, `esp_tls_set_global_ca_store(cert_pem, len)`, `esp_tls_get_global_ca_store`, `esp_tls_free_global_ca_store`, `esp_tls_set_conn_sockfd` (wrap existing socket), `esp_tls_server_session_create/delete`.

Config fields worth knowing: `cacert_buf`/`cacert_bytes`, `clientcert_buf`/`clientkey_buf`, `psk_hint_key`, `use_global_ca_store`, `use_secure_element`, `is_plain_tcp`, `non_block`, `ds_data` (digital signature peripheral).

### 3.9 mbedtls

Default crypto backend for esp-tls. Exposes standard `mbedtls/*.h` headers (ssl.h, x509.h, rsa.h, ecp.h, ecdh.h, etc.). ESP-IDF provides:
- Hardware accelerators wired into `mbedtls_sha*`, `mbedtls_aes_*`, `mbedtls_mpi_*`, `mbedtls_ecp_*` when enabled.
- Global CA cert bundle: `esp_crt_bundle.h` → `esp_crt_bundle_attach(&conf)` and CMake helper `espressif__esp_crt_bundle` for custom bundles via `idf.py add-dependency espressif/esp_crt_bundle` or build-time `COMPONENT_EMBED_TXTFILES`.

Kconfig: `MBEDTLS_HARDWARE_{AES,MPI,SHA}`, `MBEDTLS_ASYMMETRIC_CONTENT_LEN`, `MBEDTLS_SSL_IN_CONTENT_LEN`, `MBEDTLS_SSL_OUT_CONTENT_LEN`, `MBEDTLS_HAVE_TIME_DATE`, `MBEDTLS_TLS_ENABLED`, `MBEDTLS_{ECP,RSA,ECDSA,ECDH}_C`, `MBEDTLS_CERTIFICATE_BUNDLE` + variant (full / CMN / custom).

### 3.10 tcp_transport

Transport abstraction used by mqtt, http_client and custom users:
`esp_transport_tcp_init`, `esp_transport_ssl_init`, `esp_transport_ws_init(parent)`, `esp_transport_list_init/add/destroy`, `esp_transport_set_default_port`, `esp_transport_connect/read/write/close/destroy`, options via `esp_transport_ssl_set_*` (cert bundle, alpn, skip_common_name, psk, client cert).

### 3.11 http_parser, protobuf-c, protocomm, esp_local_ctrl, wpa_supplicant

- **http_parser:** llhttp wrapper (`http_parser.h`) — used internally by http_client/server.
- **protobuf-c:** runtime for generated .pb-c protobuf files; consumed by protocomm.
- **protocomm:** provisioning framework — `protocomm_new`, `protocomm_delete`, `protocomm_add_endpoint(pc, name, handler, priv)`, transports `protocomm_httpd_start/stop`, `protocomm_ble_start/stop`, `protocomm_console_start/stop`; security `protocomm_set_security(pc, ep, &sec1/&sec2, password|key)`.
- **esp_local_ctrl:** device control over any protocomm transport; APIs in `esp_local_ctrl.h` (`esp_local_ctrl_start/stop`, `esp_local_ctrl_add_property`, `esp_local_ctrl_set_handler`).
- **wpa_supplicant:** mostly internal; exposed bits: WPS (`esp_wifi_wps_enable/start/disable`), EAP (`esp_wifi_sta_enterprise_enable/disable`, enterprise cert/key setters in `esp_eap_client.h`).

---

## 4. Storage & filesystems

```
POSIX I/O  →  VFS  →  [FATFS | SPIFFS | NVS]  →  [wear_levelling | sec_provider]  →  esp_partition  →  [spi_flash | sdmmc | esp_blockdev]
```

### 4.1 spi_flash

**Headers:** `esp_flash.h`, `spi_flash_mmap.h`, `esp_partition.h`, `esp_spi_flash_counters.h`.

`esp_flash_t *esp_flash_default_chip`.

APIs: `esp_flash_init(chip)`, `esp_flash_read(chip, buf, addr, len)`, `esp_flash_write(chip, buf, addr, len)`, `esp_flash_erase_region(chip, start, size)`, `esp_flash_get_size`, `esp_flash_chip_driver_initialized`. External SPI flash via `esp_flash_add_host` + `esp_flash_init_ext` (uses SPI2/SPI3 host).

Memory-mapped: `spi_flash_mmap(src, size, SPI_FLASH_MMAP_DATA|INST, &ptr, &handle)`, `spi_flash_munmap(handle)`. `spi_flash_cache2phys(ptr)` / `_phys2cache(offset, memory)`.

Sector = 4 KB, block = 64 KB. Writes need pre-erased region.

### 4.2 esp_partition

Iterate: `esp_partition_find(type, subtype, label)` → `esp_partition_next(it)` → `esp_partition_get(it)` → `esp_partition_iterator_release(it)`. Shortcut: `esp_partition_find_first`. Read-only fields on `esp_partition_t`: `address, size, erase_size, type, subtype, label[17], encrypted, readonly, flash_chip`.

Ops: `esp_partition_read/write/erase_range`, `esp_partition_read_raw/write_raw` (bypass flash encryption), `esp_partition_verify(part)` (re-check against partition table), `esp_partition_mmap(part, offset, size, MMAP_DATA|INST, &ptr, &h)`, `esp_partition_stats(&s)` (hit counters).

Types: `ESP_PARTITION_TYPE_APP/DATA/BOOTLOADER/PARTITION_TABLE/ANY`. Subtypes: factory / ota_0..15 / ota / test / nvs / phy / nvs_keys / coredump / efuse_em / undefined / fat / spiffs / littlefs / tee_ota_0..1 / tee_sec_storage.

### 4.3 nvs_flash

**Headers:** `nvs_flash.h`, `nvs.h`, `nvs_handle.hpp` (C++).

Init: `nvs_flash_init()`, `nvs_flash_init_partition(label)`, `nvs_flash_erase()`, `nvs_flash_erase_partition(label)`, `nvs_flash_deinit[_partition(label)]`, `nvs_flash_read_security_cfg`, `nvs_flash_register_security_scheme`, `nvs_flash_secure_init`.

Handle: `nvs_open(namespace, NVS_READWRITE|NVS_READONLY, &h)`, `nvs_open_from_partition(label, ns, mode, &h)`, `nvs_close(h)`.

Typed get/set for `i8/u8/i16/u16/i32/u32/i64/u64/str/blob`:
`nvs_set_u32(h, "key", v)`, `nvs_get_u32(h, "key", &v)`, `nvs_set_str(h, "key", s)`, `nvs_get_str(h, "key", buf, &len)` (pass `NULL` first to query length), same for blob.

Misc: `nvs_erase_key(h, key)`, `nvs_erase_all(h)`, `nvs_commit(h)` (required to persist), `nvs_get_stats(label, &stats)`, `nvs_get_used_entry_count(h, &count)`, `nvs_entry_find(label, ns, type, &it)` / `nvs_entry_next(&it)` / `nvs_entry_info(it, &info)` / `nvs_release_iterator(it)`.

Key size limit: 15 chars. Namespace limit: 15 chars. Blob limit: ~4000 B chunk, multi-chunk up to ~508 KB per entry on 2-sector NVS. On `ESP_ERR_NVS_NO_FREE_PAGES` / `_NEW_VERSION_FOUND`, erase and reinit.

### 4.4 nvs_sec_provider

Register HMAC-based XTS key derivation:
`nvs_sec_provider_register_hmac(&scheme_cfg, &scheme)`, `nvs_flash_register_security_scheme(scheme)` before `nvs_flash_secure_init`.

`nvs_sec_cfg_t { uint8_t eky[32]; uint8_t tky[32]; }` — XTS keys. Config stored in `nvs_keys` partition (subtype 0x04).

### 4.5 fatfs (+ wear_levelling)

`esp_vfs_fat_mount_config_t { format_if_mount_failed, max_files, allocation_unit_size, disk_status_check_enable, use_one_fat }`.
Mount on SPI flash (wear-levelled): `esp_vfs_fat_spiflash_mount_rw_wl(path, label, &cfg, &wl_handle)` / `esp_vfs_fat_spiflash_unmount_rw_wl`. Read-only: `esp_vfs_fat_spiflash_mount_ro`.
Mount on SD: `esp_vfs_fat_sdmmc_mount(path, &host, &slot, &cfg, &card)` / `esp_vfs_fat_sdspi_mount(path, &host, &dev, &cfg, &card)` / `esp_vfs_fat_sdcard_format(path, card)` / `esp_vfs_fat_sdcard_unmount(path, card)`.
Low-level FatFS under `ff.h` still available (`f_open`, `f_read`, `f_write`, `f_sync`, `f_close`, `f_mkdir`, `f_unlink`, `f_stat`, `f_getfree`).

Kconfig: `FATFS_LONG_FILENAMES_{NONE|HEAP|STACK}`, `FATFS_MAX_LFN` (12–255), `FATFS_SECTOR_{512|4096}`, `FATFS_CODEPAGE`, `FATFS_PER_FILE_CACHE`, `FATFS_VOLUME_COUNT`.

### 4.6 spiffs

`esp_vfs_spiffs_conf_t { base_path, partition_label, max_files, format_if_mount_failed }`.
`esp_vfs_spiffs_register(&cfg)`, `esp_vfs_spiffs_unregister(label)`, `esp_spiffs_info(label, &total, &used)`, `esp_spiffs_format(label)`, `esp_spiffs_gc(label, size)`, `esp_spiffs_check(label)`, `esp_spiffs_mounted(label)`.

No directories. Good for <100 flat small files; degrades above ~90% full.

### 4.7 vfs

Register a filesystem at a base path: `esp_vfs_register(base_path, &vfs, ctx)` / `_unregister(base_path)` / `esp_vfs_register_fd_range(&vfs, ctx, min_fd, max_fd)`. Eventfd: `esp_vfs_eventfd_register(&{max_fds})`.

POSIX `open/close/read/write/lseek/fstat/stat/unlink/rename/fsync/ftruncate/opendir/readdir/closedir/mkdir/rmdir` are routed per path prefix. `select` is supported by UART, socket, eventfd, some other drivers.

Kconfig: `VFS_SUPPORT_TERMIOS`, `VFS_SUPPORT_SELECT`, `VFS_SUPPORT_EVENTFD`, `VFS_SUPPORT_IO`, `VFS_SUPPORT_DIR`.

### 4.8 wear_levelling

`wl_mount(partition, &h)` / `wl_unmount(h)`, `wl_read/write/erase_range`, `wl_size(h)`, `wl_sector_size(h)`. Handle is `WL_INVALID_HANDLE = -1` on failure. Kconfig: `WL_SECTOR_SIZE_512|4096`, `WL_SECTOR_MODE_PERF|SAFE`.

### 4.9 sdmmc, esp_driver_sdmmc, esp_driver_sdspi, esp_driver_sdio, esp_driver_sd_intf

Protocol layer `sdmmc_cmd.h` exposes `sdmmc_card_init(host, &card)`, `sdmmc_read_sectors/write_sectors` (512-byte blocks), `sdmmc_get_status`, `sdmmc_erase_sectors`.

Host variants:
- `driver/sdmmc_host.h` — SDMMC peripheral host (not on C3).
- `driver/sdspi_host.h` — SPI mode: `sdspi_host_init`, `sdspi_host_init_device(&{host_id, gpio_cs}, &handle)`, uses any SPI bus. **Use this on C3.**
- SDIO master/slave: `driver/sdio_slave.h` (device side), via `sdmmc_host`.

Macros: `SDMMC_HOST_DEFAULT()`, `SDSPI_HOST_DEFAULT()`, `SDSPI_DEVICE_CONFIG_DEFAULT()`.

### 4.10 esp_blockdev

v6 generic block device layer. `esp_blockdev_handle_t`, `esp_blockdev_geometry_t { disk_size, read_size, write_size, erase_size }`, `esp_blockdev_ops_t` — usable by NVS/FAT alternative backends. Typically internal.

---

## 5. Bluetooth & wireless

### 5.1 bt — controller + host(s)

Controller APIs (common regardless of host): `esp_bt.h` / `esp_bt_defs.h`.
- `esp_bt_controller_config_t cfg = BT_CONTROLLER_INIT_CONFIG_DEFAULT();`
- `esp_bt_controller_init(&cfg)`, `_enable(ESP_BT_MODE_{IDLE|BLE|CLASSIC_BT|BTDM})`, `_disable()`, `_deinit()`, `_get_status()`.
- `esp_bt_controller_mem_release(mode)` — release RAM for modes you won't use (non-reclaimable until reset).
- Baseband controls: `esp_ble_tx_power_set/get`, `esp_ble_scan_dupilcate_list_flush`.

**NimBLE host** (preferred for BLE-only — smaller footprint; recommended on C3):

Headers under `components/bt/host/nimble/nimble/nimble/host/include/host/`:
`ble_hs.h`, `ble_hs_id.h`, `ble_gap.h`, `ble_gatt.h`, `ble_att.h`, `ble_uuid.h`, `ble_store.h`, `ble_sm.h`, plus services: `services/gap/ble_svc_gap.h`, `services/gatt/ble_svc_gatt.h`. And porting: `esp_nimble_hci.h`, `nimble/nimble_port.h`, `nimble/nimble_port_freertos.h`.

Boilerplate:
```c
#include "esp_nimble_hci.h"
#include "nimble/nimble_port.h"
#include "nimble/nimble_port_freertos.h"
#include "host/ble_hs.h"
#include "services/gap/ble_svc_gap.h"
#include "services/gatt/ble_svc_gatt.h"

nimble_port_init();
ble_hs_cfg.sync_cb = on_sync;
ble_hs_cfg.reset_cb = on_reset;
ble_svc_gap_init();
ble_svc_gatt_init();
ble_svc_gap_device_name_set("esp32c3");
ble_gatts_count_cfg(gatt_svcs);
ble_gatts_add_svcs(gatt_svcs);
nimble_port_freertos_init(host_task);
```

GAP adv: `ble_gap_adv_set_fields`, `ble_gap_adv_start(own_addr, duration, &adv_params, gap_event_cb, ctx)`, `ble_gap_adv_stop`, `ble_gap_conn_find`, `ble_gap_terminate`.
Scan: `ble_gap_disc(own_addr, duration_ms, &params, cb, ctx)`, `ble_gap_disc_cancel`.
GATT server: `ble_gatts_add_svcs(&svcs[])`, `ble_gatts_notify/indicate(conn, attr_handle)`, `ble_gatts_chr_updated`.
GATT client: `ble_gattc_disc_all_svcs`, `_disc_all_chrs`, `_read`, `_write`, `_exchange_mtu`, `_subscribe`.
Security: `ble_sm_pair`, `ble_sm_inject_io`, security callbacks via `ble_hs_cfg.sm_*` fields.

**Bluedroid host** (Classic + BLE; heavier). Not usable on C3 Classic-free silicon, but BLE half works.
Headers: `components/bt/host/bluedroid/api/include/api/`:
`esp_bt_main.h`, `esp_bt_device.h`, `esp_gap_ble_api.h`, `esp_gatts_api.h`, `esp_gattc_api.h`, `esp_gap_bt_api.h` (classic), `esp_a2dp_api.h`, `esp_avrc_api.h`, `esp_spp_api.h`, `esp_hidd_api.h`, `esp_hidh_api.h`.
Skeleton: `esp_bluedroid_init(&cfg)`, `esp_bluedroid_enable()`, `esp_ble_gap_register_callback(cb)`, `esp_ble_gatts_register_callback(cb)`, `esp_ble_gatts_app_register(app_id)`, `esp_ble_gap_set_device_name`, `esp_ble_gap_config_adv_data(&adv_data)`, `esp_ble_gap_start_advertising(&adv_params)`. Many event enums: `ESP_GAP_BLE_ADV_START_COMPLETE_EVT`, `ESP_GATTS_CONNECT_EVT`, `ESP_GATTS_WRITE_EVT`, `ESP_GATTS_READ_EVT`, etc.

**Kconfig:** `BT_ENABLED`, `BT_NIMBLE_ENABLED` / `BT_BLUEDROID_ENABLED` / `BT_CONTROLLER_ONLY`, `BT_NIMBLE_MAX_CONNECTIONS`, `BT_NIMBLE_ROLE_{CENTRAL|PERIPHERAL|BROADCASTER|OBSERVER}`, `BT_NIMBLE_SM_SC` (secure connections), `BT_BLE_50_FEATURES_SUPPORTED`, `BT_BLE_CCA_MODE_*`. On C3, `BT_CTRL_MODE_BLE_ONLY` is default.

### 5.2 esp_hid

`esp_hidd.h` (device), `esp_hidh.h` (host), `esp_hid_common.h`. Supports BLE-HID and BT-HID (Classic). Provides `esp_hidd_dev_init`, `esp_hidd_dev_connected`, `esp_hidd_dev_input_set/get`; client `esp_hidh_init`, `esp_hidh_dev_open`, `esp_hidh_dev_input_set_cb`.

### 5.3 ieee802154 — not on C3

`esp_ieee802154.h`. APIs: `esp_ieee802154_enable/disable`, `_set_channel/_coordinator/_pan_id/_short_address/_extended_address`, `_transmit(frame, cca)`, `_receive()`, `_sleep()`. Callbacks: `esp_ieee802154_receive_done`, `_transmit_done`, `_energy_detect_done`. Available on C6/H2.

### 5.4 openthread — not on C3

`esp_openthread.h`, `esp_openthread_types.h`, `esp_openthread_cli.h`, `esp_openthread_netif_glue.h`, `esp_openthread_border_router.h`, `esp_openthread_lock.h`. Provides `esp_openthread_init(&cfg)`, `esp_openthread_launch_mainloop`, `esp_openthread_deinit`, plus CLI and netif glue. Available on C6/H2; others can run as Thread-over-Host via external RCP.

### 5.5 esp_coex

`esp_coexist.h`. `esp_coex_preference_set(ESP_COEX_PREFER_BALANCE|WIFI|BT)`, `esp_coex_status_bit_set/clear`, external coex GPIO via `esp_coex_external_params`. Used automatically when both Wi-Fi and BT are on.

### 5.6 esp_phy

`esp_phy_init.h`, `esp_phy_cert_test.h`. `esp_phy_load_cal_data_from_nvs`, `esp_phy_store_cal_data_to_nvs`, `esp_phy_enable(ESP_PHY_MODEM_{WIFI|BT|IEEE802154})` (internal; apps rarely call directly). Kconfig: `ESP_PHY_CALIBRATION_AND_DATA_STORAGE`, `ESP_PHY_ENABLE_USB`, `ESP_PHY_ENABLE_CERT_TEST`.

### 5.7 wpa_supplicant (public bits)

WPS: `esp_wifi_wps_enable(&cfg)`, `esp_wifi_wps_start(timeout)`, `_disable()`. Enterprise (WPA2-EAP, WPA3-Enterprise): via `esp_eap_client.h` — `esp_eap_client_set_{identity,username,password,ca_cert,certificate_and_key,disable_time_check}`, `esp_wifi_sta_enterprise_enable/disable`.

---

## 6. Analog / display / imaging peripherals

### 6.1 esp_adc

On C3 use ADC1 (GPIO0–4). Driver is handle-based with oneshot and continuous modes and separate calibration schemes.

Oneshot:
```c
adc_oneshot_unit_init_cfg_t ucfg = { .unit_id=ADC_UNIT_1, .ulp_mode=ADC_ULP_MODE_DISABLE };
adc_oneshot_unit_handle_t u;
adc_oneshot_new_unit(&ucfg, &u);
adc_oneshot_chan_cfg_t ccfg = { .bitwidth=ADC_BITWIDTH_DEFAULT, .atten=ADC_ATTEN_DB_12 };
adc_oneshot_config_channel(u, ADC_CHANNEL_0, &ccfg);
int raw; adc_oneshot_read(u, ADC_CHANNEL_0, &raw);
```

Calibration (C3 = curve fitting): `adc_cali_create_scheme_curve_fitting(&{unit, chan, atten, bitwidth}, &cali)`, then `adc_cali_raw_to_voltage(cali, raw, &mv)`.

Continuous (DMA):
`adc_continuous_new_handle(&{max_store_buf_size, conv_frame_size}, &h)` → `adc_continuous_config(h, &{pattern_num, adc_pattern, sample_freq_hz, conv_mode, format})` → `adc_continuous_register_event_callbacks(h, &{on_conv_done, on_pool_ovf}, arg)` → `adc_continuous_start/stop` → `adc_continuous_read(h, buf, sz, &out_len, timeout)` → `adc_continuous_deinit(h)`.

Filter: `adc_new_continuous_iir_filter(h, &{unit, channel, coeff}, &fh)`, enable/disable/delete.
Monitor: `adc_new_continuous_monitor(h, &{h_threshold, l_threshold}, &mh)`, enable/disable, callbacks `on_over_high_thresh` / `on_below_low_thresh`.

ATTEN options: `ADC_ATTEN_DB_0/2_5/6/12`. 12 dB is the old 11 dB re-branded.

Kconfig: `ADC_ONESHOT_CTRL_FUNC_IN_IRAM`, `ADC_CONTINUOUS_ISR_IRAM_SAFE`, `ADC_DISABLE_DAC_OUTPUT`, `ADC_ONESHOT_FORCE_USE_ADC2_ON_C3`.

### 6.2 esp_driver_tsens

`driver/temperature_sensor.h`:
```c
temperature_sensor_config_t cfg = TEMPERATURE_SENSOR_CONFIG_DEFAULT(-10, 80);
temperature_sensor_handle_t h;
temperature_sensor_install(&cfg, &h);
temperature_sensor_enable(h);
float c; temperature_sensor_get_celsius(h, &c);
```

Threshold interrupt: `temperature_sensor_set_absolute_threshold(h, &{high_threshold, low_threshold})`, `temperature_sensor_register_callbacks(h, &{on_threshold}, ctx)`. C3 supported.

### 6.3 esp_lcd

Headers (see peripheral index): panel interface, panel IO (SPI / I2C / I80 / RGB / DSI), vendor panels. On C3 only SPI/I2C make sense.

SPI snippet in §2.2-style already shown above. Panel vtable methods: `reset/init/del/draw_bitmap/mirror/swap_xy/set_gap/invert_color/disp_on_off/disp_sleep`. Panel IO vtable: `rx_param/tx_param/tx_color/del/register_event_callbacks`.

Vendor panels: `esp_lcd_new_panel_st7789`, `_ssd1306`, `_nt35510`, `_gc9a01` (community), etc.

### 6.4 esp_driver_dac, _touch_sens, _cam, _isp, _jpeg, _ppa — not on C3

See section index. DAC: oneshot / cosine / continuous DMA. Touch: controller+channel, benchmark, waterproof, sleep-wake. Camera: DVP (ESP32, S3) + MIPI CSI (S3). ISP: Bayer pipeline (S3). JPEG: HW codec (S3). PPA: SRM / blend / fill (S3).

---

## 7. Security, boot & OTA

### 7.1 efuse

Header: `esp_efuse.h` (+ per-chip `esp_efuse_table.h` for field definitions, generated from CSV).

Core APIs:
- `esp_efuse_read_field_blob(field, dst, size_bits)`
- `esp_efuse_write_field_blob(field, src, size_bits)`
- `esp_efuse_read_field_cnt(field, &cnt)` — for count fields (e.g. anti-rollback secure_version)
- `esp_efuse_write_field_cnt(field, cnt)`
- `esp_efuse_read_block(blk, dst, offset_bits, size_bits)` / `_write_block`
- `esp_efuse_batch_write_begin/commit/cancel` — batch writes (single operation, atomic).
- `esp_efuse_set_read_protect(blk)` / `_set_write_protect(blk)` / `_read_write_protect_*`.
- `esp_efuse_check_secure_version(sec_ver)`, `esp_efuse_update_secure_version(sec_ver)`.
- `esp_efuse_get_pkg_ver`, `esp_efuse_get_chip_ver`.
- Virtual mode: `esp_efuse_utility_reset`, enabled by `EFUSE_VIRTUAL` Kconfig — writes are kept in RAM, useful for simulation.

Key purposes (block 0..N each assigned one purpose): `ESP_EFUSE_KEY_PURPOSE_{USER|RESERVED|XTS_AES_128_KEY|SECURE_BOOT_DIGEST0..2|HMAC_DOWN_ALL|HMAC_DOWN_JTAG|HMAC_DOWN_DIGITAL_SIGNATURE|HMAC_UP|ECDSA_KEY}`. Chip-specific.

### 7.2 esp_security

Header: `esp_hmac.h`, `esp_ds.h`, `esp_random.h`, `esp_sha.h`.
- `esp_hmac_calculate(key_id, msg, len, hmac_out)` — HMAC using eFuse key blocks (DOWN_ALL purpose).
- `esp_hmac_jtag_enable(key_id, token)` / `_disable()` — re-enable JTAG after soft disable.
- `esp_ds_start_sign(&data, &signature)` / `_finish_sign` / `_encrypt_params` — Digital Signature peripheral, signs with hardware-protected private key.
- `bootloader_random_enable/disable`.

### 7.3 esp_tee

Trusted Execution Environment integration (limited on C3). Public bits in `components/esp_tee/include/` — mostly concerns partition subtypes `tee_*` and bootloader glue.

### 7.4 bootloader & bootloader_support

`bootloader_support` exposes application-side utilities:
- `esp_app_get_description()` → `const esp_app_desc_t*` `{ magic_word, secure_version, version[32], project_name[32], time[16], date[16], idf_ver[32], app_elf_sha256[32] }`.
- `esp_app_get_elf_sha256(buf, len)` — hex string of app ELF SHA-256.
- `esp_image_verify(mode, &part_pos, &image_data)`.
- `bootloader_common_get_sha256_of_partition`, `bootloader_common_get_partition_description`.
- `esp_flash_encrypt_check_and_update`, `esp_secure_boot_enabled`, `esp_secure_boot_verify_signature_block`, `esp_flash_encryption_enabled`.

### 7.5 app_update — OTA

Header: `esp_ota_ops.h`.

State machine per OTA app partition: `ESP_OTA_IMG_NEW` → `PENDING_VERIFY` (first boot of new image) → `VALID` (after `esp_ota_mark_app_valid_cancelled_rollback()`) or `INVALID` (after `_mark_app_invalid_rollback_and_reboot()`) / `ABORTED`.

APIs:
- Queries: `esp_ota_get_running_partition`, `_get_boot_partition`, `_get_next_update_partition(from)`, `_get_partition_description(part, &desc)`, `_get_app_description()` (running image), `_get_app_elf_sha256`.
- Write flow: `esp_ota_begin(part, size|OTA_SIZE_UNKNOWN|OTA_WITH_SEQUENTIAL_WRITES, &h)` → `esp_ota_write(h, data, len)` (`_with_offset` variant for non-sequential) → `esp_ota_end(h)` → `esp_ota_set_boot_partition(part)`.
- Abort: `esp_ota_abort(h)`.
- Rollback: `esp_ota_mark_app_valid_cancelled_rollback()` (confirm running image is good), `esp_ota_mark_app_invalid_rollback_and_reboot()` (explicit revert). `esp_ota_check_rollback_is_possible()`.
- Anti-rollback (requires Secure Boot v2 or enabled via Kconfig): image's `secure_version` must be ≥ eFuse count; `esp_ota_mark_app_valid_cancelled_rollback` updates eFuse.
- State machine introspection: `esp_ota_get_state_partition(part, &state)`, `esp_ota_erase_last_boot_app_partition()`.

### 7.6 esp_https_ota

Header: `esp_https_ota.h`.

Simple API (single call):
```c
esp_http_client_config_t hcfg = { .url = "https://…/firmware.bin", .crt_bundle_attach = esp_crt_bundle_attach };
esp_https_ota_config_t ocfg = { .http_config = &hcfg };
esp_err_t err = esp_https_ota(&ocfg);
if (err == ESP_OK) esp_restart();
```

Advanced (progress + callbacks):
`esp_https_ota_begin(&ocfg, &h)` → loop `esp_https_ota_perform(h)` while `ESP_ERR_HTTPS_OTA_IN_PROGRESS` → `esp_https_ota_finish(h)`. Inspection: `esp_https_ota_get_image_len_read`, `_get_image_size`, `_get_img_desc(h, &desc)`, `_abort(h)`. Event base: `ESP_HTTPS_OTA_EVENT` with `ESP_HTTPS_OTA_START/CONNECTED/GET_IMG_DESC/VERIFY_CHUNK/DECRYPT_CB/WRITE_FLASH/UPDATE_BOOT_PARTITION/FINISH/ABORT`.

Kconfig: `ESP_HTTPS_OTA_DECRYPT_CB`, `ESP_HTTPS_OTA_ALLOW_HTTP`, `ESP_HTTPS_OTA_EVENT_POST`.

### 7.7 esp-tls (already in §3.8)

### 7.8 app_trace, esp_gdbstub, espcoredump

- `app_trace.h`: `esp_apptrace_write(channel, data, len, tmo_us)`, SystemView integration (`SYSVIEW_FreeRTOS.h`).
- `esp_gdbstub.h`: `esp_gdbstub_init`, invoked by panic handler if `ESP_SYSTEM_PANIC_GDBSTUB` selected.
- `esp_core_dump.h`: `esp_core_dump_image_check/get/erase`, `esp_core_dump_to_flash` (invoked by panic), output format ELF (v6 default) or legacy binary. Decode offline with `espcoredump.py dbg_corefile`.

### 7.9 Secure Boot v2 / Flash Encryption

Mostly build-time + eFuse programming; runtime detect via `esp_secure_boot_enabled()` / `esp_flash_encryption_enabled()`. Kconfig gates: `SECURE_BOOT_V2_ENABLED`, `SECURE_SIGNED_APPS_RSA_SCHEME|ECDSA_V2_SCHEME`, `SECURE_FLASH_ENC_ENABLED`, `SECURE_FLASH_ENCRYPTION_MODE_{DEVELOPMENT|RELEASE}`, `SECURE_BOOT_ALLOW_JTAG`, `SECURE_BOOT_INSECURE`. Release mode burns keys and revokes JTAG — irreversible.

---

## 8. Utilities & dev-tools

### 8.1 console

Header: `esp_console.h`. Provides REPL over UART / USB-Serial-JTAG / USB-CDC, argtable3 argument parsing, linenoise line editing and history.

REPL:
```c
esp_console_repl_config_t repl_cfg = ESP_CONSOLE_REPL_CONFIG_DEFAULT();
esp_console_dev_uart_config_t uart_cfg = ESP_CONSOLE_DEV_UART_CONFIG_DEFAULT();
esp_console_repl_t *repl;
esp_console_new_repl_uart(&uart_cfg, &repl_cfg, &repl);
/* or esp_console_new_repl_usb_serial_jtag(&usj_cfg, &repl_cfg, &repl); */
esp_console_register_help_command();
esp_console_cmd_t cmd = { .command="hello", .help="Say hi", .func=&hello_fn };
esp_console_cmd_register(&cmd);
esp_console_start_repl(repl);
```

Lower-level: `esp_console_init(&cfg)` / `esp_console_deinit`, `esp_console_run(cmdline, &ret)`, `esp_console_get_completion`, `esp_console_get_hint`. argtable3 via `arg_int0/str1/lit0/end(...)`.

Kconfig: `CONSOLE_MAX_COMMAND_LINE_LENGTH`, `CONSOLE_SORTED_HELP`, linenoise history size.

### 8.2 esp_ringbuf

Header: `freertos/ringbuf.h` (shipped as separate component but consumed like a FreeRTOS primitive).

Types: `RingbufHandle_t`, modes `RINGBUF_TYPE_NOSPLIT` (atomic items, may waste space), `RINGBUF_TYPE_ALLOWSPLIT` (item can be split across wrap; receive gives two pointers), `RINGBUF_TYPE_BYTEBUF` (stream bytes).

APIs: `xRingbufferCreate(size, type)`, `xRingbufferCreateStatic/WithCaps`, `xRingbufferSend(rb, data, len, timeout)`, `xRingbufferSendFromISR(rb, data, len, &hpw)`, `xRingbufferReceive(rb, &size, timeout)` (returns pointer; call `vRingbufferReturnItem` after), `xRingbufferReceiveSplit`, `xRingbufferReceiveUpTo` (bytebuf), `xRingbufferGetCurFreeSize`, `xRingbufferGetMaxItemSize`, `vRingbufferDelete`, `vRingbufferReturnItem[FromISR]`. Add task-notify on space available via `xRingbufferAddToQueueSetWrite`.

### 8.3 unity

Header: `unity.h`. ESP-IDF extensions via `unity_test_runner.h` / `unity_test_utils.h`.

Register via macro `TEST_CASE("name", "[tags]") { … }`. Run suite from console or via `UNITY_BEGIN(); unity_run_menu(); UNITY_END();`. Asserts: `TEST_ASSERT_EQUAL_*`, `_TRUE/_FALSE`, `_NULL/_NOT_NULL`, `_EQUAL_STRING`, `_EQUAL_MEMORY`, `_FLOAT_WITHIN`, `_MESSAGE` variants. Used both in host `pytest` flow and on-device tests.

### 8.4 cmock

Generates mocks from headers at build time (ruby script). Runtime API after mock init is `<Fn>_Expect(...)`, `_ExpectAndReturn(...)`, `_StubWithCallback(...)`, `_ReturnThruPtr_*`, `_IgnoreArg_*`. Build integration via `REQUIRES cmock` and providing a `.cmocka` file.

### 8.5 cxx

C++ runtime glue. Supported: static constructors/destructors, exceptions (if `COMPILER_CXX_EXCEPTIONS` enabled), RTTI (if `COMPILER_CXX_RTTI`), thread_local. No iostream by default (pulls large newlib). `std::thread` maps to FreeRTOS via pthread.

### 8.6 ulp — Ultra-Low-Power co-processor

Variants:
- **ULP-RISC-V** (ESP32-S2/S3): `ulp_riscv.h`, build separately, `ulp_riscv_load_binary(bin, size)`, `ulp_riscv_run()`, `ulp_riscv_halt`, shared variables via symbols in generated `ulp_main.h`. Wake-up: `ulp_riscv_timer_resume`. Peripheral APIs: `ulp_riscv_i2c.h`, `ulp_riscv_adc.h`.
- **ULP-FSM** (ESP32 classic): assembly programs, `ulp.h`, `ulp_insn_t` ops, `ulp_load_binary`, `ulp_run`.
- **LP-Core** (C6/P4): `ulp_lp_core.h`, runs alongside HP core; shares mailbox (`lp_core_mailbox.h`) and peripherals `lp_core_i2c/spi/uart.h`.

**C3 has no ULP coprocessor.**

### 8.7 app_trace, esp_trace, esp_usb_cdc_rom_console

- `app_trace.h` + SystemView integration for high-bandwidth trace over JTAG.
- `esp_trace` — low-level trace buffer.
- `esp_usb_cdc_rom_console` — ROM-provided USB CDC console (select via `ESP_CONSOLE_SECONDARY_USB_SERIAL_JTAG` or enable CDC ROM).

### 8.8 linux, esptool_py, idf_test

- **linux:** host-target shim (ESP-IDF builds on Linux for unit testing w/o hardware).
- **esptool_py:** pulls the flashing tool; `idf.py flash` / `monitor` integrate via CMake targets.
- **idf_test:** internal performance baselines; not app-facing.

---

## 9. Examples directory map

Top of `/opt/esp/idf/examples/` (one-liner per dir):

| Dir | Purpose |
|---|---|
| `get-started/` | hello_world, blink (first-run examples) |
| `system/` | OTA, console, deep-sleep, sysview, watchdog, pm (power mgmt), heap_task_tracking, gdbstub, esp_event, app_trace, efuse, console of subsystems |
| `peripherals/` | One subdir per peripheral: gpio/, spi_master/, spi_slave/, i2c/, uart/, ledc/, rmt/, gptimer/, pcnt/, mcpwm/, adc/, dac/, touch_sensor/, temperature_sensor/, twai/, lcd/, usb/, parlio/, sdio/, sdm/, ana_cmpr/, ethernet/ — reference drivers for every driver component |
| `storage/` | fatfs, spiffs, sd_card (SDMMC & SDSPI), nvs_rw_*, nvsgen, partition_*, wear_levelling, virtual_fs |
| `protocols/` | http_server, http_client, esp_http_client, https_server, https_request, mqtt/tcp/ssl/ws, sntp, mdns, asio, icmp_echo, static_ip, sockets |
| `wifi/` | getting_started (STA/AP), scan, wps, fast_scan, smart_config, iperf, espnow, power_save, wifi_eap_fast, wifi_easy_connect (DPP), roaming |
| `bluetooth/` | `nimble/` (BLE peripheral, central, HID, mesh, iBeacon, proximity, hci), `bluedroid/` (classic+BLE, SPP, A2DP, AVRCP, HFP), `esp_hid_host`, `esp_hid_device`, `ble_get_started/` |
| `ethernet/` | basic, iperf (SPI ethernet module examples for C3) |
| `ieee802154/` | sniffer, receive, transmit — **needs C6/H2** |
| `openthread/` | ot_cli, ot_rcp, ot_br (border router) — **needs C6/H2** |
| `provisioning/` | wifi_prov_mgr (BLE/SoftAP), legacy, mfg |
| `security/` | flash_encryption, secure_boot, ds (Digital Signature), hmac_soft_jtag, nvs_encrypted_hmac, efuse |
| `lowpower/` | deep sleep helpers, light sleep + wifi, ulp, lp_core on supported chips |
| `mesh/` | ESP-MESH (tree topology on Wi-Fi) |
| `network/` | simple_sniffer, vlan, eth2ap bridge |
| `build_system/` | component manager, wrappers, cmake basics |
| `custom_bootloader/` | adding hooks / custom flow |
| `cxx/` | C++ samples |
| `phy/` | cert test |
| `zigbee/` | Zigbee HA examples (needs H2/C6) |

Look here first when implementing a new peripheral or protocol — examples are the closest to idiomatic ESP-IDF v6 code.

---

## 10. Tools directory map

Key entry points under `/opt/esp/idf/tools/`:

| Path | Purpose |
|---|---|
| `idf.py` | Main build/flash/monitor CLI (wraps CMake + Ninja + esptool + size + menuconfig) |
| `idf_monitor.py` | Serial monitor with exception decoding and panic handling (invoked by `idf.py monitor`) |
| `idf_size.py` | Size reports per component/file (`idf.py size`, `size-components`, `size-files`) |
| `idf_tools.py` | Toolchain installer/manager (installs xtensa/riscv gcc, openocd, python deps) |
| `gdb_panic_server.py` | Post-mortem GDB from panic UART output |
| `mkdfu.py` / `mkuf2.py` | Build DFU / UF2 images |
| `esp_app_trace/` | SystemView / apptrace helpers |
| `ldgen/` | Linker-script generator from `linker.lf` fragments |
| `kconfig_new/` | Kconfig frontend |
| `cmake/`, `cmakev2/` | CMake build rules |
| `mass_mfg/` | Manufacturing partition generation (mass MFG tool for NVS provisioning) |
| `templates/` | Project skeletons used by `idf.py create-project` |
| `detect_python.{sh,fish}` | Pre-activation python detection |
| `activate.py` | `export.sh` helper (sets `IDF_PATH`, PATH, Python venv) |

Also at top level: `export.sh`, `install.sh`, `add_path.sh`.

Separately, `esptool.py` lives in a managed Python package (installed by `idf_tools.py install`); not in `tools/` directly. It is the low-level flasher invoked under the hood by `idf.py flash`.

---

## Full index files (on-disk memory mirror)

The same content is split per-domain under the memory folder so future sessions can load a single domain cheaply:

- `~/.claude/projects/-workspaces-receiver-esp32/memory/esp_idf_sdk/01_core_system.md`
- `~/.claude/projects/-workspaces-receiver-esp32/memory/esp_idf_sdk/04_storage.md`
- `~/.claude/projects/-workspaces-receiver-esp32/memory/esp_idf_sdk/06_peripherals_advanced.md`

(Sections 2, 3, 5, 7, 8 of this file are the authoritative index for those domains.)
