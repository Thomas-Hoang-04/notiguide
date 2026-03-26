# Browser Compatibility Report

This report evaluates the browser compatibility of both Notiguide web frontends — the **admin dashboard** (`web/`) and the **client queue app** (`client-web/`) — against the most common modern browsers as of March 2026.

## Target Browsers

| Browser | Minimum Version | Global Market Share (Desktop + Mobile) |
|---------|----------------|----------------------------------------|
| Google Chrome | 76+ | ~65% |
| Safari (macOS/iOS) | 14.1+ | ~19% |
| Mozilla Firefox | 104+ | ~3% |
| Microsoft Edge | 79+ | ~5% |
| Samsung Internet | 15+ | ~2.5% |
| Opera | 63+ | ~2% |

**Not supported:** Internet Explorer (any version), pre-Chromium Edge (<79), Safari <14.1.

---

## Shared Technology Stack (Both Apps)

Both apps share the same core stack and therefore the same baseline compatibility profile:

| Technology | Version | Minimum Browser Requirement |
|------------|---------|----------------------------|
| Next.js | 16.2.1 | ES2017+ browsers |
| React | 19.2.4 | ES2017+ browsers |
| TypeScript target | ES2017 | Chrome 55+, Safari 10.1+, Firefox 54+, Edge 15+ |
| TypeScript lib | `dom`, `dom.iterable`, `esnext` | N/A (compile-time only) |
| Tailwind CSS | v4 | No IE support; modern browsers only |
| PostCSS | `@tailwindcss/postcss` | Build-time only |
| React Compiler | babel-plugin-react-compiler 1.0.0 | Build-time only |
| shadcn/ui | via `@base-ui/react` ^1.3.0 | Modern browsers (no IE) |
| Zustand | v5 | Any ES2017+ environment |
| next-intl | v4.8.3 | Any ES2017+ environment |
| next-themes | v0.4.6 | Any ES2017+ environment |

**No browserslist config** is defined in either app. Both rely on Next.js defaults, which target modern browsers and exclude IE11.

**No polyfills** are included in either codebase. Both apps rely on feature detection and graceful degradation.

---

## CSS Feature Compatibility

### Features Used in Both Apps

| CSS Feature | Admin | Client | Chrome | Safari | Firefox | Edge | Notes |
|-------------|-------|--------|--------|--------|---------|------|-------|
| CSS Custom Properties | Yes | Yes | 49+ | 9.1+ | 31+ | 16+ | Full support across all modern browsers |
| CSS Grid | Yes | Yes | 57+ | 10.1+ | 52+ | 16+ | Full support |
| Flexbox | Yes | Yes | 29+ | 9+ | 22+ | 12+ | Full support |
| `calc()` | Yes | Yes | 26+ | 7+ | 16+ | 12+ | Full support |
| `backdrop-filter` | Yes | Yes | 76+ | 9+ | 103+ | 17+ | Safari needs `-webkit-` prefix (provided) |
| `-webkit-backdrop-filter` | Yes | Yes | N/A | 9+ | N/A | N/A | Vendor prefix for Safari (correctly applied) |
| `inset` shorthand | Yes | Yes | 87+ | 14.1+ | 66+ | 87+ | Shorthand for top/right/bottom/left |
| `@layer` | Yes | No | 99+ | 15.4+ | 97+ | 99+ | Used in admin for cascade organization |
| `:is()` pseudo-class | Yes | No | 88+ | 14+ | 78+ | 88+ | Used in dark theme selector |
| `:has()` selector | No | Yes | 105+ | 15.4+ | 121+ | 105+ | Used in card.tsx component |
| Container Queries | No | Yes | 105+ | 16+ | 110+ | 105+ | Used in card.tsx (`@container/card-header`) |
| `data-*` attribute selectors | Yes | Yes | 7+ | 3.1+ | 3+ | 12+ | Full support |
| `prefers-reduced-motion` | Yes | Yes | 74+ | 10.1+ | 63+ | 79+ | Accessibility media query |
| `prefers-contrast` | Yes | Yes | 96+ | 14.1+ | 101+ | 96+ | High-contrast fallback |
| CSS Animations (`@keyframes`) | Yes | Yes | 43+ | 9+ | 16+ | 12+ | Full support |
| `font-variant-numeric` | No | Yes | 52+ | 9.1+ | 34+ | 12+ | For tabular number rendering |
| `isolation: isolate` | Yes | Yes | 41+ | 8+ | 36+ | 79+ | Stacking context for glass effects |

### Compatibility Verdict: CSS

**Admin dashboard:** All CSS features are well-supported in Chrome 99+, Safari 15.4+, Firefox 97+, Edge 99+. The most restrictive feature is `@layer` (Chrome 99+).

**Client queue app:** The `:has()` selector and Container Queries push the minimum to Chrome 105+, Safari 16+, Firefox 121+, Edge 105+. These are used in `card.tsx` for layout variants. Firefox 121 was released December 2023, so this is broadly available but worth noting.

---

## JavaScript / Web API Compatibility

### Features Used in Both Apps

| API / Feature | Admin | Client | Chrome | Safari | Firefox | Edge | Notes |
|---------------|-------|--------|--------|--------|---------|------|-------|
| `fetch()` | Yes | Yes | 42+ | 10.1+ | 39+ | 14+ | Full support |
| `async`/`await` | Yes | Yes | 55+ | 10.1+ | 52+ | 15+ | ES2017 baseline |
| Optional Chaining (`?.`) | Yes | Yes | 80+ | 13.1+ | 74+ | 80+ | Transpiled by TS but also natively supported |
| Nullish Coalescing (`??`) | Yes | Yes | 80+ | 13.1+ | 72+ | 80+ | Transpiled by TS but also natively supported |
| `AbortController` | Yes | Yes | 66+ | 12.1+ | 57+ | 16+ | Request cancellation |
| `localStorage` | Yes | Yes | 4+ | 4+ | 3.5+ | 12+ | Full support |
| `sessionStorage` | Yes | No | 5+ | 4+ | 2+ | 12+ | Admin only |
| `JSON.parse`/`stringify` | Yes | Yes | 3+ | 4+ | 3.5+ | 12+ | Full support |
| `Intl.DateTimeFormat` | Yes | Yes | 24+ | 10+ | 29+ | 12+ | Full support |
| Template Literals | Yes | Yes | 41+ | 9+ | 34+ | 12+ | Full support |
| Destructuring | Yes | Yes | 49+ | 8+ | 41+ | 14+ | Full support |
| Spread Operator | Yes | Yes | 46+ | 8+ | 16+ | 12+ | Full support |
| `Promise` | Yes | Yes | 32+ | 8+ | 29+ | 12+ | Full support |

### Features Unique to Admin Dashboard

| API / Feature | Chrome | Safari | Firefox | Edge | Notes |
|---------------|--------|--------|---------|------|-------|
| `EventSource` (SSE) | 6+ | 4+ | 6+ | 79+ | Real-time queue events. Pre-Chromium Edge unsupported. |
| Keyboard events (`keydown`) | All | All | All | All | Queue management shortcuts (N, S, C) |

### Features Unique to Client Queue App

| API / Feature | Chrome | Safari | Firefox | Edge | Notes |
|---------------|--------|--------|---------|------|-------|
| Service Worker API | 40+ | 11.1+ | 44+ | 17+ | FCM push notification registration |
| Notification API | 22+ | 16.4+ | 22+ | 14+ | Web Push. Safari required 16.4+ for Web Push. |
| `Notification.requestPermission()` | 46+ | 16.4+ | 47+ | 14+ | Promise-based. Safari 16.4+ for Web Push support. |
| Push API (via FCM) | 42+ | 16.4+ | 44+ | 17+ | Firebase Cloud Messaging |
| Vibration API | 32+ | No | 16+ | 79+ | **Not supported on Safari/iOS.** Feature-detected. |
| Page Visibility API | 33+ | 7+ | 18+ | 12+ | Pause polling when tab is hidden |
| `navigator.serviceWorker.register()` | 40+ | 11.1+ | 44+ | 17+ | SW lifecycle management |
| `ServiceWorker.postMessage()` | 40+ | 11.1+ | 44+ | 17+ | Config transfer to SW |
| `clients.matchAll()` (in SW) | 42+ | 11.1+ | 54+ | 17+ | Notification click handling |

### Compatibility Verdict: JavaScript

**Admin dashboard:** Full compatibility across all modern browsers. `EventSource` is the only notable API, and it's supported everywhere except pre-Chromium Edge (which is effectively dead).

**Client queue app:** The Web Push / Notification API stack requires Safari 16.4+ (released March 2023). The Vibration API is **not available on any Safari/iOS version** — this is correctly feature-detected and degrades gracefully. All other APIs have broad support.

---

## Detailed Browser Verdicts

### Google Chrome (Desktop & Android)

| Feature | Verdict |
|---------|---------|
| Admin Dashboard | Full support from Chrome 99+ (for `@layer`) |
| Client Queue App | Full support from Chrome 105+ (for `:has()` and Container Queries) |
| Push Notifications | Full support |
| Vibration | Full support (Android) |

**Overall: Fully compatible.** Chrome 105 was released August 2022; negligible user base below this version.

### Safari (macOS & iOS)

| Feature | Verdict |
|---------|---------|
| Admin Dashboard | Full support from Safari 15.4+ (for `@layer`, `:is()`) |
| Client Queue App | Full support from Safari 16.4+ (for Web Push / Notification API) |
| `backdrop-filter` | Supported via `-webkit-` prefix (correctly implemented) |
| Push Notifications | Safari 16.4+ only (March 2023). Older Safari cannot receive push. |
| Vibration | **Not supported on any Safari/iOS version.** Degrades gracefully via feature detection. |

**Overall: Compatible with graceful degradation.** Safari 16.4 is widely adopted. The Vibration API absence on iOS is an inherent platform limitation, not a code issue. The `-webkit-backdrop-filter` prefix is correctly applied for glass effects.

### Mozilla Firefox

| Feature | Verdict |
|---------|---------|
| Admin Dashboard | Full support from Firefox 104+ (for `backdrop-filter` without prefix) |
| Client Queue App | Full support from Firefox 121+ (for `:has()` selector) |
| Push Notifications | Full support |
| Vibration | Full support (Android) |
| `backdrop-filter` | Firefox 103+ without prefix |

**Overall: Fully compatible.** Firefox 121 was released December 2023. The `:has()` selector is the limiting factor — older Firefox versions will not render card.tsx layout variants correctly. Given Firefox's auto-update policy, the vast majority of users are on 121+.

### Microsoft Edge (Chromium)

| Feature | Verdict |
|---------|---------|
| Admin Dashboard | Full support from Edge 99+ |
| Client Queue App | Full support from Edge 105+ |
| Push Notifications | Full support |
| Vibration | Full support (Android) |

**Overall: Fully compatible.** Edge tracks Chromium releases closely. Pre-Chromium Edge (Legacy) is not supported, which is expected.

### Samsung Internet (Android)

| Feature | Verdict |
|---------|---------|
| Admin Dashboard | Full support from Samsung Internet 15+ |
| Client Queue App | Full support from Samsung Internet 19+ (`:has()` support) |
| Push Notifications | Full support via FCM |
| Vibration | Full support |

**Overall: Fully compatible.** Samsung Internet tracks Chromium with a slight delay. Version 19+ (Chromium 105 base) covers `:has()` and Container Queries.

### Opera

| Feature | Verdict |
|---------|---------|
| Both Apps | Follows Chromium compatibility. Opera 85+ (Chromium 99 base) for admin, Opera 91+ (Chromium 105 base) for client. |

**Overall: Fully compatible.**

---

## Known Limitations & Graceful Degradation

| Limitation | Affected Browsers | Impact | Mitigation |
|------------|-------------------|--------|------------|
| Vibration API unavailable | All Safari / iOS | No haptic feedback when ticket is called | Feature-detected: `if ("vibrate" in navigator)`. Silent fallback. |
| Web Push requires Safari 16.4+ | Safari <16.4 | Cannot receive push notifications | Polling-based fallback for queue status updates. |
| `:has()` selector | Firefox <121 | Card layout variants may not apply | Affects `card.tsx` styling only. Content remains accessible, layout degrades to default. |
| Container Queries | Firefox <110, Safari <16 | Card header sizing may not respond to container | Content remains functional, only sizing refinement is lost. |
| `prefers-contrast: high` | Partial support | High-contrast mode fallback for glass effects | Graceful: glass effects become flat solid backgrounds. |
| `@layer` cascade | Chrome <99, Safari <15.4 | CSS specificity ordering may differ | Minimal visual impact; Tailwind utilities still apply correctly. |

---

## Firebase SDK Version Note

The client queue app uses two different Firebase SDK versions:
- **npm package:** `firebase@12.11.0` (main app)
- **Service Worker CDN:** `firebase-compat@11.8.1` (push notification worker)

These are independent version tracks. The compat SDK in the Service Worker uses the legacy Firebase API surface, which is stable and does not need to match the main app version. No compatibility issues arise from this version difference.

---

## Summary

| App | Effective Minimum Browsers | Coverage |
|-----|---------------------------|----------|
| Admin Dashboard (`web/`) | Chrome 99+, Safari 15.4+, Firefox 104+, Edge 99+ | ~97% of global browser traffic |
| Client Queue App (`client-web/`) | Chrome 105+, Safari 16.4+, Firefox 121+, Edge 105+ | ~95% of global browser traffic |

Both applications are built with modern standards and have **excellent compatibility** across all major browsers released in the past 2 years. The codebase correctly applies vendor prefixes where needed (`-webkit-backdrop-filter`), uses feature detection for optional APIs (Vibration, Notifications), and provides graceful degradation through `@media` queries (`prefers-reduced-motion`, `prefers-contrast`).

No polyfills are needed, and no critical functionality is gated behind unsupported APIs — the most impactful limitation (Web Push on older Safari) is mitigated by polling-based fallback.
