# Clangd + ESP-IDF in Cursor (Linux)

Setup for getting clangd to parse an ESP-IDF project cleanly in Cursor / VS Code on Linux. Covers both architecture families:

- **RISC-V targets** (ESP32-C3/C6/H2/P4) — works with stock distro `clangd`.
- **Xtensa targets** (ESP32 / S2 / S3) — requires Espressif's `esp-clang` fork (stock clang has no Xtensa backend).

Both paths share the same user-level config; only the clangd binary and a small VS Code setting differ.

The Microsoft C/C++ extension is not used. Its IntelliSense competes with clangd and gives inconsistent results on cross-compiled code. Clangd with `--query-driver` pointed at the Espressif GCC toolchain is the accurate option.

## Why the setup works

Three facts hold everything together, regardless of chip family:

1. **Clangd is a parser, not a compiler.** The build still runs `{riscv32,xtensa}-esp-elf-gcc`; clangd only re-parses the same commands for IDE features. Codegen flags like `-fno-tree-switch-conversion`, `-fstrict-volatile-bitfields`, or `-mlongcalls` only matter insofar as clang has to *parse* them — strip the few it refuses, silence the rest as warnings.
2. **`--query-driver` hands clang the GCC header world.** Clangd runs the Espressif GCC once, scrapes its include paths and target triple, and splices them into every parse. That's how picolibc, newlib, FreeRTOS, and the `esp_*` / `driver/*` component headers all resolve.
3. **The backend has to exist.** Stock LLVM has `riscv32` built in — so stock clangd parses RISC-V targets fine. Stock LLVM does **not** have `xtensa` — so Xtensa needs Espressif's `esp-clang` fork, which adds the backend (currently based on clang 20.1.1).

## 1. Prerequisites

Common to both chip families:

| Thing | Check | Notes |
|---|---|---|
| `vscode-clangd` extension | Install from extensions | Publisher `llvm-vs-code-extensions.vscode-clangd`. Works in Cursor. |
| ESP-IDF installed | `idf.py --version` | Assumed at `/opt/esp/idf`. |
| Espressif GCC toolchain | `ls /opt/esp/tools/{riscv32,xtensa}-esp-elf/*/` | Comes with ESP-IDF by default. |
| `build/compile_commands.json` exists | `ls build/compile_commands.json` | Run `idf.py build` (or at least `idf.py reconfigure`). |

Clangd binary depends on target family:

| Target family | Clangd to use | Install |
|---|---|---|
| RISC-V (C3/C6/H2/P4) | stock distro `clangd` ≥ 17 | `sudo apt install clangd`. Verified on clangd 18.1.3. |
| Xtensa (ESP32/S2/S3) | Espressif `esp-clang` (clang 20.1.1) | `python $IDF_PATH/tools/idf_tools.py install esp-clang` — see §11. |

If you change component `REQUIRES` or the target chip, re-run `idf.py reconfigure` so `compile_commands.json` regenerates. Clangd reads exact command lines from that file — stale DB, stale diagnostics.

## 2. Files to create

Three files, widening scope:

```
<project>/.clangd                   # project-level: points at build/
<project>/.vscode/settings.json     # editor-level: disable MS IntelliSense, pass flags to clangd
~/.config/clangd/config.yaml        # user-level: the real work
```

**Why the user-level file is load-bearing.** A project `.clangd` only applies to files under that directory. ESP-IDF component sources live at `/opt/esp/idf/components/...`, so when clangd indexes them in the background (compile_commands.json lists them), the project `.clangd` does **not** apply. Their parse fails on GCC-only flags, errors surface in the Problems pane, and the index stays broken. Put the flag-stripping rules at user level so they apply everywhere.

## 3. Project `.clangd`

`<project>/.clangd`:

```yaml
CompileFlags:
  CompilationDatabase: build
```

That's the whole thing. Everything else belongs in the user config.

## 4. `.vscode/settings.json`

```jsonc
{
  "C_Cpp.intelliSenseEngine": "disabled",
  "clangd.arguments": [
    "--background-index",
    "--clang-tidy",
    "--header-insertion=never",
    "--compile-commands-dir=${workspaceFolder}/build",
    "--query-driver=/opt/esp/tools/**/bin/*-elf-gcc*"
  ]
}
```

**For Xtensa targets**, add one more line telling vscode-clangd to use esp-clang's clangd instead of the system one (full path from §11):

```jsonc
"clangd.path": "/path/to/tools/esp-clang/esp-20.1.1_20250829/esp-clang/bin/clangd",
```

The `--query-driver` glob is the hinge. It must match the Espressif GCC binary path; clangd invokes it once per driver to extract system include paths and target triple. The glob `*-elf-gcc*` deliberately covers both `riscv32-esp-elf-gcc` and the chip-specific Xtensa names (`xtensa-esp32-elf-gcc`, `xtensa-esp32s3-elf-gcc`, etc.) — the narrower `*-esp-elf-*` would miss Xtensa. The first time clangd runs, the vscode-clangd extension prompts once per driver to confirm executing it — accept.

`--header-insertion=never` is taste. Drop it if you want auto-insertion on completion.

## 5. User-level `~/.config/clangd/config.yaml`

Same config works for both RISC-V (stock clang) and Xtensa (esp-clang), verified with `clangd --check`.

```yaml
CompileFlags:
  Add:
    - -Wno-unknown-warning-option
    - -Wno-unknown-attributes
    - -Wno-single-bit-bitfield-constant-conversion
  Remove:
    # GCC-only optimization / codegen flags clang can't parse
    - -fno-tree-switch-conversion
    - -fstrict-volatile-bitfields
    - -fno-shrink-wrap
    - -fno-inline-functions-called-once
    - -fno-inline-small-functions
    - -fno-malloc-dce
    - -fzero-init-padding-bits*
    # GCC-only -W flags
    - -Wlogical-op
    - -Wformat-signedness
    - -Wformat-overflow*
    - -Wformat-truncation*
    - -Wno-old-style-declaration
    - -Wno-error=unused-but-set-variable
    # Espressif GCC fork extension — clang rejects the CPU name
    - -mtune=esp*

Diagnostics:
  Suppress:
    - drv_unknown_argument
    - drv_unknown_argument_with_suggestion
  UnusedIncludes: Strict
  MissingIncludes: Strict
  Includes:
    IgnoreHeader:
      - "driver/.*"
      - "freertos/.*"
      - "esp_.*\\.h"
      - "sdkconfig\\.h"
      - "soc/.*"
      - "hal/.*"
```

### Why each Remove entry exists

- **`-mtune=esp*`** — this is the load-bearing one on RISC-V. Espressif's GCC fork bakes `-mtune=esp-base` into its specs. `--query-driver` picks it up and injects it into the command clang sees. `esp-base` is not a CPU stock clang knows → `CreateTargetInfo()` returns null → no preamble → all indexing fails. Strip it and clang falls back to `generic-rv32`, which is fine. Harmless to keep the rule on Xtensa (esp-clang knows the CPU either way).
- **`-fno-tree-switch-conversion`, `-fstrict-volatile-bitfields`, `-fno-shrink-wrap`, `-fno-inline-functions-{called-once,small-functions}`, `-fno-malloc-dce`, `-fzero-init-padding-bits=all`** — GCC-only optimization tuning. Clang would emit "unknown argument" warnings for each, which `-Werror` turns into fatal errors (or at least noise).
- **`-Wlogical-op`, `-Wformat-signedness`, `-Wformat-overflow*`, `-Wformat-truncation*`, `-Wno-old-style-declaration`, `-Wno-error=unused-but-set-variable`** — GCC-only warning names. `-Wno-unknown-warning-option` (in Add) would normally silence these, but stripping them is cleaner and avoids any chance of `-Werror` interactions.

**Do NOT remove `-specs=picolibc.specs`.** Clang treats it as an unknown argument and ignores it (warning suppressed via `drv_unknown_argument`), but GCC needs it to be present when clangd runs the query-driver step — that's how picolibc include paths get emitted. Remove it and you lose `<stdio.h>`, `<stdlib.h>`, and half of libc.

**Xtensa-specific GCC flags.** Verified on ESP32-S3 + esp-clang 20.1.1: `-mlongcalls`, `-fno-builtin-memcpy`, `-fno-builtin-memset`, `-fno-builtin-bzero` — all parsed without error. No extra Remove rules needed. Same goes for the `@build/toolchain/cflags` response file pattern that ESP-IDF uses on Xtensa (clang expands `@file` the same way GCC does).

### Why each Add entry exists

- **`-Wno-unknown-warning-option`** — backstop for any GCC-only `-W` flag we haven't added to Remove.
- **`-Wno-unknown-attributes`** — ESP-IDF uses GCC attributes (`__attribute__((section(...)))`, various custom ones) that clang sometimes doesn't recognize.
- **`-Wno-single-bit-bitfield-constant-conversion`** — clang-only diagnostic that fires on `struct { int bit:1; } x; x.bit = 1;`. GCC doesn't complain. Some ESP-IDF headers do this; with `-Werror` in the command, it's fatal.

### Why the Diagnostics block looks the way it does

- **`Suppress: drv_unknown_argument*`** — backstop for any stray GCC-only flag. Without this you still see yellow squiggles in the Problems pane even though the parse succeeds.
- **`UnusedIncludes: Strict` + `MissingIncludes: Strict`** — clangd's IWYU-style "included header not used directly" diagnostics. Keeping them on is useful. The `IgnoreHeader` regexes opt out the ESP-IDF umbrella / config headers where the check misfires (e.g., `driver/gpio.h` declares types used only transitively, `sdkconfig.h` is the big generated config).
- To **disable** the unused-includes check entirely, set `UnusedIncludes: None`. For **per-case** overrides while keeping it on, annotate the include: `#include "foo.h"  // IWYU pragma: keep`.

## 6. Verify

Run clangd in check mode against one project source and one ESP-IDF component source. Both should report 0 errors.

```bash
clangd --check=main/main.c \
       --compile-commands-dir=build \
       --query-driver='/opt/esp/tools/**/bin/*-elf-gcc*' 2>&1 \
  | grep -E '^E\[|All checks'

clangd --check=/opt/esp/idf/components/esp_hw_support/intr_alloc.c \
       --compile-commands-dir=build \
       --query-driver='/opt/esp/tools/**/bin/*-elf-gcc*' 2>&1 \
  | grep -E '^E\[|All checks'
```

Expected: `All checks completed, 0 errors` on both. Reload Cursor (`Developer: Reload Window`) so the running clangd picks up config changes.

## 7. Per-case suppression

For unused-include false positives outside the IgnoreHeader regexes:

```c
#include "driver/gpio.h"   // IWYU pragma: keep
```

For a header you own that should count as used wherever it's included:

```c
// at top of the header
// IWYU pragma: always_keep
```

For an umbrella header re-exporting a private one:

```c
// IWYU pragma: export
#include "internal_impl.h"
```

## 8. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `unknown argument: '-fno-shrink-wrap'` (and friends) | GCC-only flag reached clang | Add to `CompileFlags.Remove` |
| `unknown target CPU 'esp-base'` / `CreateTargetInfo() return null` | Espressif GCC default mtune in specs | `-mtune=esp*` must be in Remove |
| `Couldn't build compiler instance` | Preamble-crashing flag (usually the one above) | Run `clangd --check=<file> ...` and read the actual error |
| `driver/gpio.h` (or any `driver/*`, `esp_*`, etc.) not found | Corresponding ESP-IDF component isn't in `main/CMakeLists.txt` `REQUIRES` | Add the REQUIRES; `idf.py reconfigure` |
| Project files parse, but ESP-IDF component files still show errors | Project `.clangd` doesn't apply outside the project tree | Put Remove/Suppress rules in `~/.config/clangd/config.yaml` (not project `.clangd`) |
| Missing picolibc types (`__FILE`, `_REENT`, `FILE`) | Removed `-specs=picolibc.specs` | Don't strip it — clang ignores it, GCC needs it for query-driver extraction |
| Endless "included header not used directly" on ESP-IDF umbrellas | `UnusedIncludes: Strict` with no IgnoreHeader regexes | Add regexes or set `UnusedIncludes: None` |
| Config edits don't take effect | Running clangd holds old config | `Ctrl+Shift+P` → `clangd: Restart language server`, or reload window |
| `file is not in compile database` | Source not in `compile_commands.json` | Confirm it's in `SRCS`; `idf.py reconfigure` |

## 9. Regenerating the index

Any time you change `idf.py set-target`, a `CMakeLists.txt` `REQUIRES`/`SRCS`, or Kconfig/sdkconfig:

```bash
idf.py reconfigure
```

`build/compile_commands.json` rewrites in place. Clangd picks up changes on next file open, or via `clangd: Restart language server`.

## 10. Cheat sheet (RISC-V)

```bash
# 1. Generate compile DB
idf.py build        # or at least: idf.py reconfigure

# 2. Project marker
cat > .clangd <<'EOF'
CompileFlags:
  CompilationDatabase: build
EOF

# 3. Editor settings
mkdir -p .vscode
cat > .vscode/settings.json <<'EOF'
{
  "C_Cpp.intelliSenseEngine": "disabled",
  "clangd.arguments": [
    "--background-index",
    "--clang-tidy",
    "--header-insertion=never",
    "--compile-commands-dir=${workspaceFolder}/build",
    "--query-driver=/opt/esp/tools/**/bin/*-elf-gcc*"
  ]
}
EOF

# 4. User config — paste section 5 into this file
mkdir -p ~/.config/clangd
${EDITOR:-nano} ~/.config/clangd/config.yaml

# 5. Reload Cursor, then verify
clangd --check=main/main.c \
       --compile-commands-dir=build \
       --query-driver='/opt/esp/tools/**/bin/*-elf-gcc*' 2>&1 \
  | grep -E '^E\[|All checks'
```

---

## 11. Xtensa setup (ESP32, ESP32-S2, ESP32-S3)

Stock distro clang has no Xtensa backend, so it can't parse `xtensa-esp-elf` even with `--query-driver`. Use Espressif's `esp-clang` fork instead.

### Install esp-clang

```bash
# Writes under $IDF_TOOLS_PATH (default /opt/esp if writable, else set your own)
IDF_TOOLS_PATH=$HOME/.espressif \
  python $IDF_PATH/tools/idf_tools.py install esp-clang
```

Verified layout after install (clang 20.1.1, ~340 MB download):

```
$IDF_TOOLS_PATH/tools/esp-clang/esp-20.1.1_20250829/esp-clang/
├── bin/
│   ├── clangd              ← point vscode-clangd at this
│   ├── clang, clang++, clang-tidy, clang-format, ...
│   └── esp32.cfg, esp32s2.cfg, esp32s3.cfg, ...
└── lib/clang-runtimes/
    ├── xtensa-esp-unknown-elf/
    └── riscv32-esp-unknown-elf/
```

`clangd --version` reports: `Espressif clangd version 20.1.1 ... target=riscv32-esp-elf` (default triple; it auto-selects the right one per compile command).

### Tell vscode-clangd to use it

Add to `.vscode/settings.json`:

```jsonc
{
  "C_Cpp.intelliSenseEngine": "disabled",
  "clangd.path": "/your/IDF_TOOLS_PATH/tools/esp-clang/esp-20.1.1_20250829/esp-clang/bin/clangd",
  "clangd.arguments": [
    "--background-index",
    "--clang-tidy",
    "--header-insertion=never",
    "--compile-commands-dir=${workspaceFolder}/build",
    "--query-driver=/opt/esp/tools/**/bin/*-elf-gcc*"
  ]
}
```

The rest — project `.clangd`, user-level `~/.config/clangd/config.yaml`, the `--query-driver` glob — is **identical to the RISC-V setup**. The same Remove/Add/Suppress rules from §5 handle Xtensa's flags (`-mlongcalls`, `-fno-builtin-memcpy`, `-fno-builtin-memset`, `-fno-builtin-bzero`, the `@build/toolchain/cflags` response file) without modification.

### Verify

```bash
ESPCLANGD=$IDF_TOOLS_PATH/tools/esp-clang/esp-20.1.1_20250829/esp-clang/bin/clangd

# Project source
$ESPCLANGD --check=main/main.c \
  --compile-commands-dir=build \
  --query-driver='/opt/esp/tools/**/bin/*-elf-gcc*' 2>&1 \
  | grep -E '^E\[|All checks'

# ESP-IDF component
$ESPCLANGD --check=/opt/esp/idf/components/esp_hw_support/intr_alloc.c \
  --compile-commands-dir=build \
  --query-driver='/opt/esp/tools/**/bin/*-elf-gcc*' 2>&1 \
  | grep -E '^E\[|All checks'
```

Both should report `All checks completed, 0 errors`. The `--log=verbose` mode prints `got target: "xtensa-esp-elf"` on success — if you see `not allowed driver /opt/esp/tools/xtensa-esp-elf/.../xtensa-esp32s3-elf-gcc`, your `--query-driver` glob is too narrow. Use `*-elf-gcc*` as shown — not `*-esp-elf-*`, which misses the chip-specific Xtensa names (`xtensa-esp32s3-elf-gcc`, `xtensa-esp32-elf-gcc`).

### Xtensa-specific gotchas

- **Chip-specific GCC binaries.** Xtensa toolchain ships `xtensa-esp32-elf-gcc`, `xtensa-esp32s2-elf-gcc`, `xtensa-esp32s3-elf-gcc` — chip-specific because each has different ISA extensions. The glob must be wide enough to match.
- **Response file `@build/toolchain/cflags`.** On Xtensa, ESP-IDF puts `-mlongcalls -fno-builtin-* -specs=picolibc.specs` into a response file referenced by `@...` in the compile command, rather than inlining them. Clang follows `@file` natively, so no special handling needed.
- **Mixing chip families in one workspace.** If you work on both Xtensa and RISC-V projects from the same Cursor install, the simplest approach is to always use esp-clang (it handles both). Stock distro clangd only works for RISC-V.
