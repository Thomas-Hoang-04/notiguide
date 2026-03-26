# Web Architecture Walkthrough

This guide explains the key architectural decisions and concepts in the Notiguide web frontend. It covers font loading, client vs server rendering, navigation patterns, responsive design, and the hydration model.

---

## 1. Font Loading (Custom Properties)

The app uses two Google Fonts loaded via `next/font/google` in `web/src/app/layout.tsx`:

```tsx
const beVietnamPro = Be_Vietnam_Pro({ variable: "--font-be-vietnam-pro", ... });
const inter = Inter({ variable: "--font-inter", ... });
```

These are applied to `<body>` as CSS class names. Next.js generates classes that define `--font-be-vietnam-pro` and `--font-inter` as CSS custom properties at runtime.

In `globals.css`, they are mapped to Tailwind tokens:

```css
@theme inline {
  --font-sans: var(--font-be-vietnam-pro);
  --font-mono: var(--font-inter);
}
```

> **IDE Note:** WebStorm (and similar IDEs) may flag `--font-be-vietnam-pro` / `--font-inter` as "unresolved" because they are injected at runtime by Next.js, not statically defined in CSS. This is a false positive and can be suppressed in IDE settings (Settings > Editor > Inspections > CSS > Invalid CSS property value).

---

## 2. Client vs Server Rendering (`"use client"`)

### Why most files are client components

This is an **SPA-style admin dashboard**. Almost every component interacts with browser-only APIs:

- **Zustand stores** (`useAuthStore`, `useQueueStore`) — require React hooks
- **localStorage** — JWT token and admin data persistence
- **Event handlers** — form submissions, button clicks, dialog toggles
- **Polling** — `setInterval` for queue stats
- **shadcn/ui components** — built on base-ui primitives with DOM interaction

Server components excel for content-heavy pages with data fetching. A dashboard that is fundamentally interactive with client-side auth does not benefit from server rendering.

### What legitimately needs `"use client"`

| Category | Examples | Reason |
|---|---|---|
| Pages with hooks | login, stores, admins, queue, password | `useState`, `useEffect`, `useRouter` |
| Stores | `auth.ts`, `queue.ts` | Zustand `create()` requires client context |
| Layout components | sidebar, topbar, auth-guard | Navigation hooks, Sheet/dialog state |
| Feature components | store-form-dialog, create-admin-dialog, etc. | Form state, event handlers |
| `api.ts` | HTTP client | Accesses `localStorage`, `window` |
| shadcn/ui | select, sheet, dialog, tooltip, etc. | DOM interaction via base-ui primitives |
| Custom hooks | `use-disclosure.ts` | Uses `useState`/`useCallback` (technically redundant since hooks inherit client context from consumers, but harmless) |

### What should NOT have `"use client"`

- **Redirect-only pages** (e.g. `settings/page.tsx`, `app/page.tsx`) — these call `redirect()` from `next/navigation`, which works as a server component. No hooks, no browser APIs.

### About `suppressHydrationWarning`

This attribute appears only on the `<html>` tag in `layout.tsx`. Browser extensions (password managers, ad blockers, dark mode tools) often modify `<html>` attributes before React hydrates. Without it, you get console warnings about server/client HTML mismatch for things outside your control. It does **not** suppress warnings on child elements.

---

## 3. Navigation & Redirect Patterns

Two distinct redirect patterns exist, each for a different reason:

### Server-side redirect — `redirect()` from `next/navigation`

Used when the decision requires **no browser state**:

```tsx
// app/page.tsx — always redirects, no conditions
export default function Home() {
  redirect("/dashboard");
}
```

Runs on the server before any HTML is sent. Fast, no flash.

### Client-side redirect — `router.replace()` + `return null`

Used when the decision depends on **browser-only data** (localStorage JWT):

```tsx
// dashboard/page.tsx — needs auth state from Zustand/localStorage
useEffect(() => {
  if (isHydrated && isAuthenticated) {
    router.replace("/dashboard/queue");
  }
}, [isHydrated, isAuthenticated, router]);

return null;
```

This pattern appears in:

| File | Condition | Redirect target |
|---|---|---|
| `dashboard/page.tsx` | Authenticated | `/dashboard/queue` |
| `stores/page.tsx` | Not super admin | `/dashboard` |
| `auth-guard.tsx` | Not authenticated | `/login` |

### Why `return null`?

While `router.replace()` is in-flight (async), the component must render *something*. Returning `null` renders nothing, preventing a brief flash of unauthorized content. The alternative (showing a spinner) is heavier than necessary for a redirect that takes milliseconds.

### Why not use server-side redirect for auth?

Auth tokens live in `localStorage`, which is browser-only. The server has no access to it. With HTTP-only cookies, server-side auth checks and redirects would be possible, but this app uses bearer tokens for API compatibility.

---

## 4. Responsive Specification

The app uses a **mobile-first, two-tier** responsive approach with Tailwind's `md:` breakpoint (768px) as the single divider.

### Breakpoint behavior

| Element | Mobile (<768px) | Desktop (>=768px) |
|---|---|---|
| **Sidebar** | Hidden. Mobile menu via Sheet in Topbar | Visible, fixed left column |
| **Topbar** | Hamburger menu + admin info | Admin info only (no hamburger) |
| **Table columns** | Address + Created hidden (`hidden md:table-cell`) | All columns visible |
| **Content padding** | `p-4` | `p-6` (`md:p-6`) |
| **Queue action buttons** | `min-height: 3rem` | `min-height: 3.5rem` (via `queue.css` media query) |
| **Header rows** | Wrap via `flex-wrap gap-4` | Single row |

### Key patterns used

- `hidden md:block` — hide on mobile, show on desktop (sidebar)
- `hidden md:table-cell` — hide table columns on mobile
- `flex-wrap gap-4` — allow toolbar items to wrap naturally on small screens
- CSS `@media (min-width: 768px)` in `queue.css` — larger touch targets on desktop

No tablet-specific breakpoints (`lg:`, `xl:`) are used. The dashboard is functional on phones but optimized for desktop use.

---

## 5. Hydration

### What is hydration?

When you visit a Next.js page, the server sends pre-built HTML so you see content instantly. But that HTML is "dead" — buttons don't work, forms don't submit. **Hydration** is the process where React "wakes up" the dead HTML by attaching event handlers and connecting it to JavaScript state.

Think of it like receiving a printed menu at a restaurant (server HTML) vs. the waiter actually arriving to take your order (hydration complete).

### Two layers of hydration in this app

**Layer 1 — React hydration:** React attaches event handlers to the server-rendered HTML. Automatic, handled by Next.js.

**Layer 2 — Auth hydration:** The app reads `localStorage` for a saved JWT and admin data, then verifies it against the backend. This is the `hydrate()` function in `store/auth.ts`.

### Auth hydration timeline

```
Page loads
  → React hydrates (buttons work)
    → AuthGuard mounts, calls hydrate()
      → Reads localStorage for JWT + admin JSON
        → Checks JWT expiry client-side
          → Calls GET /api/admins/me to verify token
            → isHydrated = true
              → Render dashboard OR redirect to /login
```

Before `isHydrated` becomes `true`, the AuthGuard shows a loading spinner. Without this gate, you'd see a flash: the login page appears for a split second, then jumps to the dashboard (or vice versa).

### The `isHydrating` guard

```tsx
let isHydrating = false; // module-level flag

hydrate: async () => {
  if (isHydrating) return;
  isHydrating = true;
  // ... rest of hydration logic
}
```

React 18+ Strict Mode runs effects twice in development. Without this flag, `hydrate()` would fire two parallel API calls to `/api/admins/me`, potentially racing each other and causing inconsistent state. The module-level flag (not React state) ensures only the first call proceeds, because it persists across re-renders.

### Why not use cookies instead?

With HTTP-only cookies, the server could check auth and redirect before sending any HTML — no client-side hydration dance needed. This app uses bearer tokens in `localStorage` for API compatibility with the mobile client (which also consumes the same Spring Boot backend). The tradeoff is the extra client-side hydration step.
