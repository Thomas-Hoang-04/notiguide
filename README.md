# NotiGuide

NotiGuide is an end-to-end queue management and notification system for stores. Customers join a virtual queue from their phone and watch their position live. Staff run the floor from an admin dashboard — calling, serving, and redirecting tickets in real time. When a ticket is called, the notification reaches the customer wherever they are: as a web push in their browser, or as a buzz on a dedicated RF pager in their hand.

What makes it interesting is the full vertical slice: a reactive Kotlin backend with an atomic Redis queue engine, two bilingual Next.js apps, and custom ESP32/ESP8266 firmware speaking 2.4 GHz and 433 MHz radio — all designed, built, and wired together in this workspace.

## Techstack

<p>
  <a href="https://kotlinlang.org/"><img alt="kotlin" src="https://img.shields.io/badge/-Kotlin-7F52FF?logo=kotlin&logoColor=white"/></a>
  <a href="https://spring.io/projects/spring-boot"><img alt="spring-boot" src="https://img.shields.io/badge/-Spring%20Boot-6DB33F?logo=springboot&logoColor=white"/></a>
  <a href="https://www.timescale.com/"><img alt="timescaledb" src="https://img.shields.io/badge/-TimescaleDB-FDB515?logo=timescale&logoColor=black"/></a>
  <a href="https://redis.io/"><img alt="redis" src="https://img.shields.io/badge/-Redis-FF4438?logo=redis&logoColor=white"/></a>
  <a href="https://mqtt.org/"><img alt="mqtt" src="https://img.shields.io/badge/-MQTT%20v5-660066?logo=mqtt&logoColor=white"/></a>
  <a href="https://firebase.google.com/"><img alt="firebase" src="https://img.shields.io/badge/-Firebase%20FCM-DD2C00?logo=firebase&logoColor=white"/></a>
  <a href="https://nextjs.org/"><img alt="nextjs" src="https://img.shields.io/badge/-Next.js%2016-000000?logo=nextdotjs&logoColor=white"/></a>
  <a href="https://react.dev/"><img alt="react" src="https://img.shields.io/badge/-React%2019-61DAFB?logo=react&logoColor=black"/></a>
  <a href="https://www.typescriptlang.org/"><img alt="typescript" src="https://img.shields.io/badge/-TypeScript-3178C6?logo=typescript&logoColor=white"/></a>
  <a href="https://tailwindcss.com/"><img alt="tailwindcss" src="https://img.shields.io/badge/-Tailwind%20CSS%204-06B6D4?logo=tailwindcss&logoColor=white"/></a>
  <a href="https://ui.shadcn.com/"><img alt="shadcn-ui" src="https://img.shields.io/badge/-shadcn%2Fui-000000?logo=shadcnui&logoColor=white"/></a>
  <a href="https://docs.espressif.com/projects/esp-idf/en/stable/esp32c3/"><img alt="esp-idf" src="https://img.shields.io/badge/-ESP--IDF%20v6.0-E7352C?logo=espressif&logoColor=white"/></a>
  <a href="https://github.com/espressif/ESP8266_RTOS_SDK"><img alt="esp8266-rtos-sdk" src="https://img.shields.io/badge/-ESP8266%20RTOS%20SDK-E7352C?logo=espressif&logoColor=white"/></a>
  <a href="https://en.cppreference.com/w/c"><img alt="c" src="https://img.shields.io/badge/-C-A8B9CC?logo=c&logoColor=black"/></a>
  <a href="https://www.freertos.org/"><img alt="freertos" src="https://img.shields.io/badge/-FreeRTOS-6CB33E"/></a>
  <a href="https://lvgl.io/"><img alt="lvgl" src="https://img.shields.io/badge/-LVGL%208.3-343839?logo=lvgl&logoColor=white"/></a>
  <a href="https://www.docker.com/"><img alt="docker" src="https://img.shields.io/badge/-Docker-2496ED?logo=docker&logoColor=white"/></a>
</p>

## System Architecture

```mermaid
flowchart TB
    subgraph Web["Web clients"]
        Client["notiguide-client<br/>customer app"]
        Admin["notiguide-admin<br/>staff dashboard"]
    end

    subgraph Cloud["notiguide-be — reactive API"]
        API["Spring WebFlux<br/>Kotlin coroutines"]
        Queue["Queue engine<br/>Redis Lua scripts"]
        Notif["Notification fan-out"]
        RD[("Redis<br/>queue state · rate limits")]
        PG[("TimescaleDB<br/>entities · analytics")]
    end

    subgraph Fleet["Pager fleet"]
        TX["notiguide-transmitter<br/>ESP32-C3 dual-radio hub"]
        RX1["receiver (esp32)<br/>dual-radio pager"]
        RX2["receiver (esp8266)<br/>433 MHz pager"]
    end

    Client -->|"REST · join & poll status"| API
    Admin -->|"REST · call, serve, dispatch"| API
    API -.->|"SSE live updates"| Admin
    API --> Queue --> RD
    API --> PG
    Queue --> Notif
    Notif -->|"FCM web push"| Client
    Notif <-->|"MQTT v5 / TLS"| TX
    TX -->|"nRF24 · 2.4 GHz"| RX1
    TX -->|"433 MHz OOK"| RX2
    TX -.->|"433 MHz OOK · 433 MHz build"| RX1
```

## Modules

| Module | Stack | Role |
|--------|-------|------|
| [notiguide-be](https://github.com/Thomas-Hoang-04/notiguide-be) | Kotlin · Spring Boot 3.5 · WebFlux · R2DBC | Reactive API: queue engine, auth, analytics, device orchestration |
| [notiguide-admin](https://github.com/Thomas-Hoang-04/notiguide-admin) | Next.js 16 · React 19 · Tailwind 4 | Staff dashboard: live queue control, pager dispatch, analytics |
| [notiguide-client](https://github.com/Thomas-Hoang-04/notiguide-client) | Next.js 16 · React 19 · Firebase | Customer app: join queues, track position, web push |
| [notiguide-transmitter](https://github.com/Thomas-Hoang-04/notiguide-transmitter) | ESP-IDF v6.0 · C · LVGL | Dual-radio hub: MQTT in, nRF24 + 433 MHz out, OLED status UI |
| [notiguide-receiver (`esp32`)](https://github.com/Thomas-Hoang-04/notiguide-receiver/tree/esp32) | ESP-IDF v6.0 · C | Dual-radio pager (2.4 GHz nRF24 or 433 MHz OOK) with vibration alert |
| [notiguide-receiver (`esp8266`)](https://github.com/Thomas-Hoang-04/notiguide-receiver/tree/esp8266) | ESP8266 RTOS SDK · C | 433 MHz pager with vibration alert |

## Key Capabilities

- **Real-time queueing** — atomic ticket lifecycle (join → call → serve) backed by Redis Lua scripts, with TTL-driven cleanup and SSE live updates to the staff dashboard.
- **Two notification paths** — Firebase Cloud Messaging for browsers, MQTT-to-RF for physical pagers; both fire from the same dispatch action.
- **Physical pager fleet** — enrollment tokens, local PSK pairing, roster sync, and a transmitter hub with an OLED status screen.
- **Analytics** — queue KPIs on TimescaleDB, charted in the admin dashboard.
- **Bilingual by design** — every UI ships in English and Vietnamese, with copy written natively rather than translated.

## Gallery

> 📷 *Admin dashboard — live queue management — coming soon*
<!-- PHOTO: admin dashboard, queue page with active tickets and dispatch controls -->

> 📷 *Customer app — queue tracking on mobile — coming soon*
<!-- PHOTO: client-web ticket view showing live position -->

> 📷 *Assembled transmitter hub — coming soon*
<!-- PHOTO: transmitter hub with OLED, button, both radio modules visible -->

> 📷 *Assembled pager receivers (ESP32-C3 and ESP8266) — coming soon*
<!-- PHOTO: both receiver builds side by side -->

## Repository Layout

This repository is a superproject: each module is a git submodule with its own history, and `docs/` carries the cross-cutting design documents.

```
├── backend/            — notiguide-be          (Kotlin/Spring Boot API)
├── web/                — notiguide-admin       (staff dashboard)
├── client-web/         — notiguide-client      (customer app)
├── transmitter/        — notiguide-transmitter (ESP32-C3 hub firmware)
├── receiver-esp32/     — notiguide-receiver    (branch: esp32)
├── receiver-esp8266/   — notiguide-receiver    (branch: esp8266)
└── docs/               — specs, plans, changelogs, style guides
```

```bash
git clone --recurse-submodules https://github.com/Thomas-Hoang-04/notiguide.git
```

---

_**Created by Minh Hai Hoang. June 2026**_
