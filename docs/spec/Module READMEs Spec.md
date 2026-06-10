# Module READMEs Spec

**Date:** 2026-06-10
**Status:** Approved design, pending implementation plan

## Goal

Replace the scaffold/missing READMEs across the NotiGuide workspace with a unified, portfolio-grade README set: one **master README** in the superproject plus one README per module (7 files total). The set must read as one coherent project on GitHub while each README accurately presents its module.

## Scope

| # | File | Repo | Current state |
|---|------|------|---------------|
| 1 | `README.md` (workspace root) | `Thomas-Hoang-04/notiguide` (superproject) | none |
| 2 | `backend/README.md` | `Thomas-Hoang-04/notiguide-be` | none |
| 3 | `web/README.md` | `Thomas-Hoang-04/notiguide-admin` | create-next-app scaffold |
| 4 | `client-web/README.md` | `Thomas-Hoang-04/notiguide-client` | create-next-app scaffold |
| 5 | `transmitter/README.md` | `Thomas-Hoang-04/notiguide-transmitter` | ESP-IDF hello-world scaffold |
| 6 | `receiver-esp32/README.md` | `Thomas-Hoang-04/notiguide-receiver` branch `esp32` | ESP-IDF hello-world scaffold |
| 7 | `receiver-esp8266/README.md` | `Thomas-Hoang-04/notiguide-receiver` branch `esp8266` | ESP8266 RTOS SDK scaffold |

Note: both receivers share one remote repo (`notiguide-receiver`) with per-target branches. Each branch keeps its own README.

## Approved decisions

| Decision | Choice |
|----------|--------|
| Audience | Portfolio / public showcase first; setup info condensed, not removed |
| Language | English only (no `README_VI.md`) |
| Cross-referencing | Every README carries an ecosystem section linking all sibling repos |
| Badge style | MATSim style: `img.shields.io` badges with brand hex colors, each wrapped in `<a>` linking to the tool homepage |
| Diagrams | Mermaid only (no ASCII art). Master gets the full-system diagram; each module gets one focused diagram |
| Structure approach | Shared spine + three family profiles (backend / web / firmware) |
| Version/status badges at top | Excluded |
| Photos/screenshots | Placeholder slots only; the author replaces them with real captures |
| Sign-off | Every README ends with `_**Created by Minh Hai Hoang. June 2026**_` (month may be adjusted per repo by the author) |

## Shared spine (all module READMEs)

Fixed section order. No section may be reordered; family slots are inserted at the marked point.

```markdown
# NotiGuide — <Module Name>

<1–2 paragraph intro: what the module is, its role in the system,
 one sentence on why it is technically interesting>

## Techstack
<clickable brand-color badges>

## The NotiGuide System
<2–3 sentence system summary, then ecosystem table:
 | Repository | Role | — all 7 entries; the current repo is bolded
 and unlinked, siblings link to their GitHub repos>

## Features
<bullet list of current capabilities — present tense, no roadmap items>

## Technical Highlights
<3–6 bullets on engineering-heavy parts>

## Architecture
<one focused Mermaid diagram>

<— FAMILY SLOT(S) — see family profiles —>

## Getting Started
<condensed prerequisites bullets + fenced run commands;
 no numbered IDE walkthroughs>

## Project Structure
<annotated directory tree, trimmed to meaningful directories>

---
_**Created by Minh Hai Hoang. June 2026**_
```

## Family profiles

### Backend (`notiguide-be`)

- **Family slot (after Architecture):** `## API Overview` — table of endpoint groups only (surface area, not full API docs): `| Group | Base path | Auth | Purpose |` covering auth, queue public, queue admin, store, admin, device/notification groups as they exist in code.
- **Getting Started:** `docker compose up -d`, `./gradlew bootRun`, `./gradlew test`; prerequisites mention Docker, JDK 21, and required env/credential files at a high level.
- **Technical Highlights candidates:** atomic Redis Lua queue operations, RSA-512 JWT + Argon2, sliding-window rate limiting (Redis Lua), MQTT v5 device sync, FCM web push, fully reactive WebFlux + coroutines + R2DBC, TimescaleDB analytics.

### Web apps (`notiguide-admin`, `notiguide-client`)

- **Family slot (after Architecture):** `## Screenshots` — 2–4 placeholder slots.
- **Features must mention:** bilingual EN/VI UI, and each app's headline flows (admin: queue dashboard, device dispatch, analytics; client: public queue join/track, web push notifications).
- **Getting Started:** `yarn dev`, `yarn build`, `yarn lint`; mention backend dependency.
- **Technical Highlights candidates:** Next.js 16 + React 19 + React Compiler, Tailwind 4, shadcn/ui, next-intl bilingual architecture, FCM web push + service worker (client), Storage Access API cross-site auth (client), Biome toolchain.

### Firmware (`notiguide-transmitter`, `notiguide-receiver` × 2 branches)

- **Family slot (after Architecture):** `## Hardware` — short prose plus a small parts table covering MCU, radio module(s), display, and power. **No GPIO/pin tables** — pin mapping is configurable via Kconfig and the section must say so in one line.
  - Transmitter: ESP32-C3, nRF24L01+ (2.4 GHz link), FS1000A (433 MHz link), SSD1306 OLED, button + status LED.
  - Receiver (esp32 branch): ESP32-C3, nRF24L01+ (2.4 GHz).
  - Receiver (esp8266 branch): ESP8266, RXB-12 (433 MHz).
  - Each Hardware section includes a 📷 placeholder for a photo of the **fully assembled device** (transmitter hub / assembled receiver of that type).
- **Receivers only — extra section right after the intro:** `## Repository Branches` (ZipCracker style) explaining the shared repo with `esp32` and `esp8266` branches and what each targets.
- **Getting Started:** ESP-IDF modules use `idf.py set-target / build / flash / monitor`; the ESP8266 module uses its RTOS SDK `make` flow. Mention `idf.py menuconfig` for Kconfig options (pins, RF settings, provisioning).
- **Technical Highlights candidates:** custom RF dispatch protocol over nRF24/433 MHz, FreeRTOS task design, OLED UI with header/clock, local pairing & provisioning (UART), diagnostics heartbeat. Verify each against actual module code/docs during implementation.

## Master README (superproject root)

```markdown
# NotiGuide

<2–3 paragraph pitch: end-to-end queue management & notification
 system for stores — from admin dashboard to physical RF pagers>

## Techstack
<union of all module badges: Kotlin, Spring Boot, TimescaleDB,
 Redis, Next.js, React, TypeScript, Tailwind, MQTT, Firebase,
 ESP-IDF, C, FreeRTOS, Docker, …>

## System Architecture
<full-system Mermaid diagram: client-web / admin-web ↔ backend ↔
 MQTT broker ↔ transmitter hub → nRF24 (2.4 GHz) → esp32 receiver
 and → FS1000A (433 MHz) → esp8266 receiver; plus FCM web push path>

## Modules
<table: | Module | Repo | Stack | Role | — all 6 modules, linked>

## Key Capabilities
<cross-cutting bullets: real-time queueing, web push, physical
 pagers, analytics, bilingual UI>

## Gallery
<📷 placeholder grid: admin dashboard, client app,
 assembled transmitter hub, assembled receivers>

## Repository Layout
<tree: submodules + docs/>

---
_**Created by Minh Hai Hoang. June 2026**_
```

## Conventions

- **Badges:** `https://img.shields.io/badge/-<Name>-<brandhex>?logo=<logo>&logoColor=white` (or `black` for light badges), wrapped in `<a href="<tool homepage>">`. One `<p>` block. Reuse exact badge markup across READMEs for shared technologies so they render identically.
- **Photo placeholders:** a visible line `> 📷 *<description> — coming soon*` plus an HTML comment `<!-- PHOTO: <exact intended shot> -->` so slots are self-describing and easy to find.
- **Ecosystem table:** identical 7-row content in every README (master + 6 modules); only the bolded current-repo row changes. Receiver rows may share one repo link with branch noted.
- **Mermaid:** diagrams are detailed rather than minimal — use subgraphs for layers/boundaries (browser vs server, hub vs fleet, filter chain vs domains), name nodes after real packages/modules, and label edges with the protocol or action (`MQTT v5 / TLS`, `nRF24 · 2.4 GHz`, `SSE live updates`). Stay focused on the module's actual responsibilities; detail means real structure, not decorative nodes.
- **Tone:** factual present tense; describe only what exists in code today. No roadmap, no marketing superlatives.
- **Content sourcing:** module facts must be drawn from each module's code and `docs/` design guides (e.g., `TRANSMITTER_ESP32C3_DESIGN_GUIDE.md`, `RECEIVER_ESP32C3_DESIGN_GUIDE.md`, UART provisioning docs), not invented.

## Out of scope

- `README_VI.md` translations
- Version/release badges at the top of files
- Full API documentation (table of endpoint groups only)
- GPIO/pin mapping tables
- Taking the actual screenshots/photos (placeholders only)
- LICENSE files / license sections (not requested)

## Implementation notes

- Each README lands in a different git repo (and the two receiver READMEs on different branches of the same repo); the implementation plan must handle per-repo working trees. No commits are made unless explicitly requested.
- Log all file changes in `docs/CHANGELOGS.md`, including anything skipped.
