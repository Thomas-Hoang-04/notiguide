# Clangd Setup for ESP8266 Development in Cursor

This guide documents how to configure **clangd** as the C/C++ language server for
**ESP8266_RTOS_SDK** projects in **Cursor** (or VS Code) on Linux. It covers the
specific toolchain bundled with the `thomashoang04/thomas-esp:esp8266` devcontainer
image and the workarounds required for the packaged `esp-clangd` build.

---

## 1. What you get when it works

- Inline diagnostics, completion, go-to-definition, rename, and inlay hints on
  ESP8266 code.
- Clang-tidy checks on user code.
- IntelliSense disabled (no conflict with clangd).
- ESP-IDF SDK headers treated as system headers — their internal warnings are
  silenced so your inbox only contains warnings about **your** code.
- `UnusedIncludes` enforcement on user code, with SDK umbrella/transitive
  headers whitelisted so real unused includes still surface.

---

## 2. Toolchain layout (devcontainer image)

| Path | What's there |
|------|--------------|
| `/opt/esp/esp-clangd/bin/clangd` | Espressif clangd v21.1.3 (Xtensa-aware) |
| `/opt/esp/esp-clangd/lib/clang/21/include/` | **Missing** — see §5.2 |
| `/opt/esp/xtensa-lx106-elf/bin/xtensa-lx106-elf-*` | GCC 8.4.0 cross-compiler |
| `/opt/esp/xtensa-lx106-elf/lib/gcc/xtensa-lx106-elf/8.4.0/include/` | GCC freestanding headers (`stddef.h`, `stdbool.h`, `stdint.h`, …) |
| `/opt/esp/xtensa-lx106-elf/xtensa-lx106-elf/include/` | Newlib headers |
| `/opt/esp/ESP8266_RTOS_SDK/components/**` | SDK sources and headers |

`PATH` already includes `/opt/esp/esp-clangd/bin` and `/opt/esp/xtensa-lx106-elf/bin`,
so `which clangd` and `which xtensa-lx106-elf-gcc` both resolve.

---

## 3. Prerequisites

1. Cursor or VS Code with the **clangd** extension installed
   (`llvm-vs-code-extensions.vscode-clangd`).
2. The Microsoft **C/C++** extension's IntelliSense **disabled** — it will
   otherwise fight clangd for diagnostics. The settings in §4 handle this.
3. **`bear`** installed (`which bear` → `/usr/bin/bear` on this image). It's
   what generates `compile_commands.json` from a Make build — see §6.
4. A generated `compile_commands.json` at the project root (see §6).

---

## 4. IDE settings: `.vscode/settings.json`

```json
{
  "clangd.arguments": [
    "--query-driver=/opt/esp/xtensa-lx106-elf/bin/xtensa-lx106-elf-*",
    "--compile-commands-dir=${workspaceFolder}",
    "--background-index",
    "--clang-tidy",
    "--header-insertion=never"
  ],
  "clangd.path": "clangd",
  "[c]": {
    "editor.defaultFormatter": "llvm-vs-code-extensions.vscode-clangd"
  },
  "C_Cpp.intelliSenseEngine": "disabled"
}
```

Flag-by-flag:

- `--query-driver=...` — **required** allowlist. Without it, clangd refuses to
  invoke the Xtensa GCC to ask about sysroot/include paths for non-trusted
  drivers. The glob matches the cross-compiler binaries.
- `--compile-commands-dir=${workspaceFolder}` — look for
  `compile_commands.json` at the project root (not inside `build/`).
- `--background-index` — builds the symbol index under `.cache/clangd/index/`
  for fast cross-file navigation.
- `--clang-tidy` — enables clang-tidy diagnostics (noise is tuned via `.clangd`).
- `--header-insertion=never` — stop clangd from auto-adding `#include` lines;
  in embedded work they're usually wrong (wrong umbrella, wrong component).

---

## 5. Project-level config: `.clangd`

Place this file at the repository root. It applies to every file under it.

```yaml
CompileFlags:
  Add:
    - --target=xtensa-lx106-elf
    - -ferror-limit=0
    # esp-clangd's resource-dir (/opt/esp/esp-clangd/lib/clang/21/include) ships
    # empty, so supply freestanding headers (stddef.h, stdbool.h, ...) from xtensa-gcc.
    - -isystem/opt/esp/xtensa-lx106-elf/lib/gcc/xtensa-lx106-elf/8.4.0/include
    - -isystem/opt/esp/xtensa-lx106-elf/lib/gcc/xtensa-lx106-elf/8.4.0/include-fixed
  Remove:
    # GCC flags clangd doesn't understand
    - -mlongcalls
    - -ffunction-sections
    - -fdata-sections
    - -fstrict-volatile-bitfields

Diagnostics:
  UnusedIncludes: Strict
  Includes:
    # Skip unused-include checks for SDK umbrella/transitive headers — they
    # trip clangd's IWYU heuristic constantly. User-code includes still get checked.
    IgnoreHeader:
      - "esp_.*\\.h"
      - "esp-.*\\.h"
      - "driver/.*"
      - "freertos/.*"
      - "soc/.*"
      - "hal/.*"
      - "rom/.*"
      - "lwip/.*"
      - "mbedtls/.*"
      - "sys/.*"
      - "sdkconfig\\.h"
      - "nvs.*\\.h"
      - "mqtt_client\\.h"
      - "tcpip_adapter\\.h"
      - "http_parser\\.h"
      - "cJSON\\.h"
  Suppress:
    - -Wno-gnu-zero-variadic-macro-arguments

InlayHints:
  Enabled: Yes
  ParameterNames: Yes
  DeducedTypes: Yes

---
# Treat ESP-IDF SDK headers as system headers (suppresses their warnings)
If:
  PathMatch: /opt/esp/.*
CompileFlags:
  Add: [-w]
```

### 5.1 Why `--target=xtensa-lx106-elf`

`compile_commands.json` calls the compiler as `xtensa-lx106-elf-gcc`. Clangd
uses clang internally, which doesn't infer the target triple from the binary
name. Setting the target explicitly makes clang pick the correct Xtensa
predefined macros and type widths (e.g. `sizeof(long)`, alignment rules).

### 5.2 Why the two `-isystem` lines

`esp-clangd` ships **only the binary** — its resource directory
`/opt/esp/esp-clangd/lib/clang/21/include/` is empty. That directory is where
a normal clang install puts freestanding headers (`stddef.h`, `stdbool.h`,
`stdint.h`, `stdarg.h`, `limits.h`, …). Without them, you get:

```
fatal error: 'stddef.h' file not found
```

The two `-isystem` entries supply them from the GCC 8.4.0 cross-toolchain.
(The `include-fixed` directory holds a few patched versions GCC uses for
broken system headers; it's harmless to include even if empty.)

> If your devcontainer image is rebuilt with a proper clangd install that
> populates `lib/clang/<ver>/include/`, these two lines become unnecessary and
> should be removed — otherwise you'd get GCC 8's `stddef.h` instead of
> clang 21's, which is subtly wrong.

### 5.3 Why the `Remove:` list

These are GCC-only flags that ESP8266_RTOS_SDK's build emits into every
compile command. Clangd (clang-based) errors out on each one:

- `-mlongcalls` — Xtensa-GCC long-call generation; clang Xtensa target uses
  a different mechanism.
- `-ffunction-sections`, `-fdata-sections` — linker-section splitting for
  `--gc-sections`; irrelevant for parsing.
- `-fstrict-volatile-bitfields` — GCC-specific memory-model knob.

Each unknown-arg error can short-circuit command parsing, so they **must** be
removed.

### 5.4 Why `UnusedIncludes: Strict` with an `IgnoreHeader` list

`Strict` flags headers whose symbols you don't use **directly** (only
transitively, via another included header). Without a whitelist this lights
up every `#include "esp_wifi.h"` because ESP-IDF relies heavily on umbrella
headers that forward-declare types used elsewhere.

The whitelist covers the ESP-family SDK paths (`esp_*.h`, `esp-*.h`,
`driver/*`, `freertos/*`, `soc/*`, `hal/*`, `rom/*`, `lwip/*`, `mbedtls/*`,
`sys/*`, plus specific standalone headers). **Your own project headers are
still checked** — so genuinely unused includes in `main.c`, `network/`,
`rf/`, `vibrator/`, etc. will be flagged.

### 5.5 Why the second `---` fragment

Clangd config fragments are separated by `---`. This one uses
`If: PathMatch: /opt/esp/.*` to apply `-w` (silence all warnings) **only**
when clangd is processing a file under `/opt/esp/`. Effect: if you jump
into an SDK header via go-to-definition, you don't see hundreds of SDK
warnings. Your own code keeps all its diagnostics.

---

## 6. Generating `compile_commands.json` with `bear`

This project uses the ESP8266_RTOS_SDK **Make**-based build
(`$(IDF_PATH)/make/project.mk`), which does **not** emit a
`compile_commands.json` on its own. Use [`bear`](https://github.com/rizsotto/Bear)
to intercept every compiler invocation during a build and record it:

```bash
# From project root (perform a clean build so bear sees every TU):
make clean
bear -- make -j$(nproc)
```

This writes `compile_commands.json` at the repo root — exactly where
`--compile-commands-dir=${workspaceFolder}` (see §4) expects it.

> If you use the CMake variant instead (`idf.py` / `cmake -B build`), CMake
> emits `build/compile_commands.json` automatically when
> `CMAKE_EXPORT_COMPILE_COMMANDS=ON` — symlink it to the root with
> `ln -sf build/compile_commands.json compile_commands.json`. `bear` is only
> needed for the Make build.

Regenerate `compile_commands.json` any time you:
- Add or remove a source file.
- Change `component.mk`/`CMakeLists.txt` includes or `COMPONENT_ADD_INCLUDEDIRS`.
- Run `make menuconfig` and toggle components in/out of the build.

After regeneration, reload the clangd server (§7).

### 6.1 Why `bear` and not a plain `make` flag

GNU Make has no built-in "dump all compiler commands as JSON" mode. `bear`
solves this by `LD_PRELOAD`-ing a shim that records every `exec*` of a
compiler (as named in its config — `gcc`, `xtensa-lx106-elf-gcc`, etc.) and
serializing the captured argv/cwd to JSON when the build exits. That's why:

- `bear` must wrap the **entire** build from `clean`. If objects are already
  built and Make skips them, bear has nothing to intercept and those TUs
  will be missing from `compile_commands.json` — clangd will then lose
  navigation for those files.
- `make -j$(nproc)` is fine; bear handles parallel builds correctly.
- If you only rebuild one file, prefer a full `bear -- make` rebuild over a
  partial capture, or clangd's view of that file may be stale.

---

## 7. Reloading the clangd server

After editing `.clangd`, `.vscode/settings.json`, or `compile_commands.json`:

1. Open the Command Palette (`Ctrl+Shift+P`).
2. Run **"clangd: Restart language server"**.

Or simply reload the window: **"Developer: Reload Window"**.

The first parse of a file rebuilds the preamble (~1 s per file on this SDK);
the background index builds over the next few minutes and is then cached
under `.cache/clangd/index/`.

---

## 8. Verifying the setup from the terminal

Clangd has a `--check` mode that parses one file in isolation and reports
errors. Use it to validate the config without involving the IDE:

```bash
/opt/esp/esp-clangd/bin/clangd \
  --query-driver='/opt/esp/xtensa-lx106-elf/bin/xtensa-lx106-elf-*' \
  --enable-config \
  --check=main/main.c 2>&1 | grep -E '(unknown argument|file not found|All checks)'
```

A healthy output ends with **`All checks completed, 3 errors`**.

The three remaining errors are cosmetic: clangd's `SwapBinaryOperands` code
action can't generate a valid source rewrite when the `*` operator lives
inside a function-like macro expansion (`pdMS_TO_TICKS(x)` expands to a `*`
that maps back to the same 18-char source span as both operands, so the two
replacements overlap). This is upstream clangd behavior, visible only in
`--check`'s exhaustive tweak sweep; the IDE simply doesn't offer "Swap
operands" as a refactor on those macros and nothing else is affected.

---

## 9. Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `unknown argument: '-mlongcalls'` (or `-ffunction-sections`, `-fdata-sections`, `-fstrict-volatile-bitfields`, `-Wno-frame-address`) | GCC-only flag reached clangd | Add to `CompileFlags.Remove` in `.clangd` |
| `'stddef.h' file not found` (or `stdbool.h`, `stdint.h`) | `esp-clangd` resource-dir is empty | Keep the two `-isystem` lines in §5 |
| Every SDK include flagged as "unused" | `UnusedIncludes: Strict` without whitelist | Use the `Includes.IgnoreHeader` list in §5 |
| Hundreds of warnings inside `/opt/esp/...` headers | SDK code treated as user code | Keep the `PathMatch: /opt/esp/.*` fragment |
| Go-to-definition jumps to the wrong header | Stale `compile_commands.json` | Rebuild and re-symlink; restart clangd |
| Completion is slow or empty on a new file | Background index still building | Wait; check `.cache/clangd/index/` populates |
| C/C++ extension squiggles conflict with clangd | IntelliSense not disabled | Set `"C_Cpp.intelliSenseEngine": "disabled"` |
| Cursor shows "clangd not found" | `clangd.path` resolution | Set `"clangd.path": "/opt/esp/esp-clangd/bin/clangd"` absolute, or ensure `/opt/esp/esp-clangd/bin` is on `PATH` before launching Cursor |

---

## 10. Files summary

| File | Purpose | Checked in? |
|------|---------|-------------|
| `.vscode/settings.json` | IDE clangd invocation and IntelliSense toggle | Yes |
| `.clangd` | Project-wide clangd config (flags, diagnostics) | Yes |
| `compile_commands.json` | Produced by `bear -- make` at repo root | No (gitignored) |
| `.cache/clangd/` | Clangd's background index and preamble cache | No (gitignored) |

---

## 11. Related

- Clangd configuration reference: <https://clangd.llvm.org/config.html>
- ESP8266_RTOS_SDK: <https://github.com/espressif/ESP8266_RTOS_SDK>
- Project-local architecture and build notes: `CLAUDE.md`, `AGENTS.md`,
  `ESP8266_SDK_REFERENCE.md`.
