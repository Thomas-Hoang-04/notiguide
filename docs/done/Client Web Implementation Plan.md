# Client Web App Implementation Plan

## Current State of Repository
The repository (`/home/thomas/Coding/notiguide/client-web`) is a freshly initialized Next.js project using:
- Next.js 16.1 (App Router)
- React 19
- Tailwind CSS 4
- TypeScript
- Biome (Linter and Formatter)
- Yarn Berry (Package Manager)

---

## Purpose

The client-web app is the **public-facing queue interface** for Notiguide. Customers access it on their phones (or tablets mounted at store kiosks) to:
1. Join a store's queue and receive a ticket
2. Track their ticket status and position in real time
3. Receive visual/audio cues when their ticket is called

This is **not** the admin dashboard — it has no authentication, no login, and no admin operations. It consumes only the **public queue API** endpoints.

---

## Design System

### Visual Style
Modern glass-based aesthetic inspired by Apple Liquid Glass, Samsung OneUI 8.5, and Google Material 3. Cards, panels, and components should use frosted glass effects (backdrop-blur, semi-transparent backgrounds), subtle shadows, and smooth transitions. Since this is a customer-facing mobile app, prioritize clarity, large touch targets, and minimal visual noise.

### Typography
- **Primary font**: Be Vietnam Pro — used for all body text, headings, and UI labels (Vietnamese-optimized)
- Both weights and styles loaded via `next/font/google` for optimal performance

### Color Palette

Uses the same design tokens defined in the [Web Styles Guide](../walkthrough/Web%20Styles.md). Key tokens for the client app:

#### Light Theme
| Token | Hex | Client Usage |
|---|---|---|
| Background | `#F8FAFC` | Main app canvas |
| Surface/Cards | `#FFFFFF` | Ticket card, status card, store info card |
| Primary | `#0D9488` | Join queue button, store branding, active states |
| Primary Hover | `#0F766E` | Hover/press on primary buttons |
| Action | `#F97316` | "Your number is being called" highlight, ticket number |
| Action Hover | `#C2410C` | Press state on action elements |
| Text (Main) | `#0C1324` | Headings, ticket numbers, status text |
| Text (Muted) | `#64748B` | Timestamps, secondary info, queue position label |
| Borders | `#E2E8F0` | Card borders, section dividers |
| Success | `#10B981` | "Successfully joined queue", "Being served" status |
| Warning | `#D97706` | "Queue is busy", estimated wait warnings |
| Destructive | `#E11D48` | "Cancel ticket" button, error states |
| Disabled Surface | `#F1F5F9` | Greyed-out join button (store inactive) |
| Disabled Text | `#94A3B8` | Text on disabled elements |

#### Dark Theme
| Token | Hex | Client Usage |
|---|---|---|
| Background | `#0F172A` | Main app canvas |
| Surface/Cards | `#28354A` | Ticket card, status card |
| Primary | `#2DD4BE` | Join queue button, active states |
| Primary Hover | `#5EEAD4` | Hover/press on primary buttons |
| Action | `#FB923C` | Called ticket highlight |
| Action Hover | `#FDBA74` | Press state on action elements |
| Text (Main) | `#F8FAFC` | All primary text |
| Text (Muted) | `#94A3B8` | Secondary info, timestamps |
| Borders | `#3E5270` | Card borders |
| Success | `#34D399` | Positive confirmations |
| Warning | `#FDE047` | Wait warnings |
| Destructive | `#FB7185` | Cancel, errors |
| Disabled Surface | `#1E293B` | Inactive elements |
| Disabled Text | `#64748B` | Disabled text |

### Color Application Guide (Client-Specific)
- **Join Queue button**: Primary background — the single most important action on the page
- **Ticket number display**: Action color when CALLED, Primary when WAITING
- **Cancel ticket button**: Destructive red, outline variant — requires confirmation
- **Status badges**: Success green (WAITING/being served), Action orange (CALLED), Muted (SERVED/CANCELLED)
- **Glass effects**: Use `backdrop-blur-md` with semi-transparent Surface colors for card backgrounds
- **Store inactive state**: Disabled surface + disabled text with explanatory message

### CSS Architecture
- Define color tokens as CSS custom properties in `globals.css` (scoped under `:root` for light and `.dark` for dark)
- Complex or glass-effect styles extracted to separate CSS files (per project convention)
- Use Tailwind for layout, spacing, and simple utility styles
- Map color tokens to Tailwind via `@theme inline` referencing CSS variables

---

## Responsive Breakpoint System (Mobile-First)

This app is **mobile-first**. The base (unprefixed) styles target the smallest phone screens. Breakpoints use `min-width` queries to progressively enhance for larger devices. Named with a clothing-size convention for intuitive usage.

### Breakpoint Definitions

| Name | Prefix | Min-Width | Target Devices | Typical Viewport |
|------|--------|-----------|----------------|------------------|
| Base | _(none)_ | 0px | Small phones (SE, Galaxy A series, older devices) | 320–359px |
| XS | `xs:` | 360px | Standard phones (Galaxy S series, Pixel, most Android) | 360–389px |
| S | `s:` | 390px | Modern phones (iPhone 14/15/16, Pixel 8+) | 390–429px |
| M | `m:` | 430px | Large phones (iPhone Pro Max, Galaxy Ultra, foldables inner) | 430–599px |
| L | `l:` | 600px | Small tablets, large foldables (Galaxy Fold open, iPad Mini) | 600–767px |
| XL | `xl:` | 768px | Standard tablets (iPad, Galaxy Tab, most Android tablets) | 768–1023px |
| 2XL | `2xl:` | 1024px | Large tablets landscape, small laptops (iPad Pro, Surface) | 1024px+ |

### Rationale
- **Base (0px)**: Covers the long tail of 320px devices still in use globally. All core content must be usable here.
- **XS (360px)**: The single most common mobile viewport worldwide (Samsung Galaxy budget/mid-range). This is where ~40% of mobile traffic lives.
- **S (390px)**: The iPhone standard viewport since iPhone 12. Captures the Apple ecosystem and premium Android (Pixel 7/8/9).
- **M (430px)**: Pro Max / Ultra class devices. Enough room for side-by-side info on ticket cards.
- **L (600px)**: The tablet threshold. Layout can shift from single-column to more spacious arrangements. Google recommends 600px as the compact-to-medium transition.
- **XL (768px)**: Classic tablet breakpoint (iPad portrait). Two-column layouts become viable.
- **2XL (1024px)**: Tablet landscape / kiosk mode. Maximum layout for this app — anything larger would be the admin dashboard territory.

### Tailwind v4 Configuration

```css
@import "tailwindcss";

@theme {
  --breakpoint-xs: 22.5rem;   /* 360px */
  --breakpoint-s: 24.375rem;  /* 390px */
  --breakpoint-m: 26.875rem;  /* 430px */
  --breakpoint-l: 37.5rem;    /* 600px */
  --breakpoint-xl: 48rem;     /* 768px */
  --breakpoint-2xl: 64rem;    /* 1024px */
}
```

> **Note**: Tailwind v4's default breakpoints (`sm`, `md`, `lg`, `xl`, `2xl`) are overridden entirely using `--breakpoint-*: initial` before defining custom breakpoints. This avoids confusion between default desktop-oriented breakpoints and our mobile-first sizing system.

### Design Guidelines Per Breakpoint

| Breakpoint | Layout | Touch Targets | Typography |
|------------|--------|---------------|------------|
| Base–XS | Single column, full-width cards, stacked elements | Min 48px height, full-width buttons | 14px body, 28px ticket number |
| S | Single column, slightly more padding | Min 48px height | 15px body, 32px ticket number |
| M | Single column, card can show inline metadata | Min 44px height | 16px body, 36px ticket number |
| L | Centered content with max-width, more whitespace | Min 44px height | 16px body, 40px ticket number |
| XL | Centered card layout, optional two-column for info sections | Standard 40px+ | 16px body, 48px ticket number |
| 2XL | Kiosk-optimized layout, large centered ticket display | Large targets for touch kiosks | 18px body, 56px ticket number |

---

## Backend API Integration

### Existing Public Queue Endpoints (No Authentication Required)

| Endpoint | Method | Purpose | Response |
|----------|--------|---------|----------|
| `POST /api/queue/public/{storeId}/tickets` | POST | Issue a new queue ticket | `IssueTicketResponse` |
| `GET /api/queue/public/{storeId}/tickets/{ticketId}` | GET | Check ticket status & position | `TicketStatusResponse` |

### New Public Endpoints (To Be Added — see Phase B0)

| Endpoint | Method | Purpose | Response |
|----------|--------|---------|----------|
| `POST /api/queue/public/{storeId}/tickets/{ticketId}/cancel` | POST | Customer cancels their own ticket | 204 No Content |
| `GET /api/queue/public/{storeId}/info` | GET | Get public store info (name, address, active status) | `StorePublicInfoResponse` |
| `GET /api/queue/public/{storeId}/size` | GET | Get current queue size without authentication | `QueueSizeResponse` |

### Response DTOs

```typescript
// Ticket status enum
type TicketStatus = "WAITING" | "CALLED" | "SERVED" | "CANCELLED" | "UNKNOWN";

// POST /api/queue/public/{storeId}/tickets
interface IssueTicketResponse {
  storeId: string;          // UUID
  ticket: TicketDto;
}

// GET /api/queue/public/{storeId}/tickets/{ticketId}
interface TicketStatusResponse {
  status: TicketStatus;
  positionInQueue: number | null;      // null if not WAITING
  estimatedWaitTime: number | null;    // null until analytics domain is built
}

// Shared ticket structure
interface TicketDto {
  id: string;               // UUID
  number: string;           // Sequential display number (e.g., "042")
  status: TicketStatus;
  issuedAt: string | null;  // ISO-8601 Instant (UTC): "2026-03-15T10:30:00Z"
  calledAt: string | null;  // ISO-8601 Instant (UTC)
  position: number | null;  // 0-indexed queue position (null when not WAITING)
}

// GET /api/queue/public/{storeId}/info (new endpoint)
interface StorePublicInfoResponse {
  id: string;                // UUID
  name: string;              // Store display name
  address: string | null;    // Optional address
  isActive: boolean;         // Whether the store is accepting tickets
}

// GET /api/queue/public/{storeId}/size (new endpoint)
interface QueueSizeResponse {
  queueSize: number;
}

// Backend error response format
interface ErrorResponse {
  timestamp: string;       // LocalDateTime format: "2026-03-15T10:30:00" (no timezone)
  code: number;
  error: string;
  message: string;
  path: string;
  method: string;
  details?: Record<string, string>;
}
```

### Ticket Lifecycle (Client Perspective)

```
Customer joins queue → WAITING (position N, TTL: 12 hours)
    │
    ├─ Admin calls next → CALLED (position null, TTL: 30 minutes)
    │       │
    │       ├─ Admin serves → SERVED (ticket deleted immediately)
    │       └─ Admin cancels → CANCELLED (ticket deleted immediately)
    │
    ├─ Customer cancels → CANCELLED (ticket deleted immediately)
    │
    └─ TTL expires (12h) → Ticket auto-deleted from Redis
```

**Important client behaviors:**
- A `404` on ticket status check means the ticket has expired or been completed — show a "ticket no longer active" state, not an error
- `estimatedWaitTime` is always `null` until the analytics domain is built — the UI should gracefully hide this field when null
- `position` is 0-indexed — display as `position + 1` for human-friendly "You are #N in line"
- Ticket `number` is the customer-facing display number (daily sequential, e.g., "042") — this is what gets called out at the counter
- Customer cancel (`POST /api/queue/public/{storeId}/tickets/{ticketId}/cancel`) is idempotent — returns 204 whether the ticket exists or not. Safe to call multiple times.

### HTTP Status Codes

| Endpoint | Success Status | Notes |
|---|---|---|
| `POST .../tickets` | **201 Created** | Returns `IssueTicketResponse` body |
| `GET .../tickets/{ticketId}` | 200 OK | Returns `TicketStatusResponse` body |
| `POST .../tickets/{ticketId}/cancel` | **204 No Content** | No body — idempotent |
| `GET .../info` | 200 OK | Returns `StorePublicInfoResponse` body |
| `GET .../size` | 200 OK | Returns `QueueSizeResponse` body |

### Rate Limiting

The backend applies rate limiting to all public endpoints under the **strict** tier: **20 requests per 60 seconds** per client IP. The client must handle:
- **429 Too Many Requests**: Display a "Please wait" message. Parse `X-RateLimit-Reset` header (Unix epoch seconds) for countdown.
- **Rate limit headers** on every response: `X-RateLimit-Remaining`, `X-RateLimit-Limit`, `X-RateLimit-Reset`
- **IP extraction**: Backend uses `X-Forwarded-For` header (first IP) or falls back to remote address

---

## Bilingual Support (i18n)

### Architecture

Follow the same pattern established in the `web/` admin dashboard:

- **Library**: `next-intl` (v4.x)
- **Locales**: `["vi", "en"]` — Vietnamese as default
- **Route prefix**: `always` — all routes prefixed with locale (`/vi/store/...`, `/en/store/...`)
- **Locale detection**: `false` — explicit in URL, no browser sniffing
- **Translation files**: `src/messages/en.json`, `src/messages/vi.json`

### Translation Namespaces (Client-Specific)

The full English ↔ Vietnamese translation map is in the [Client i18n Translation Table](../walkthrough/Client%20i18n%20Translation%20Table.md). The JSON structure below shows the English keys for reference:

```json
{
  "meta": {
    "title": "NotiGuide",
    "description": "Join the queue and track your ticket status"
  },
  "common": {
    "loading": "Loading...",
    "retry": "Try again",
    "cancel": "Cancel",
    "confirm": "Confirm",
    "close": "Close",
    "back": "Back",
    "error": "Something went wrong",
    "appName": "NotiGuide"
  },
  "store": {
    "storeInfo": "Store Information",
    "address": "Address",
    "queueStatus": "Queue Status",
    "ticketsWaiting": "{count} tickets waiting",
    "noOneWaiting": "No one is in line — you'll be next!",
    "storeNotFound": "Store not found",
    "storeNotFoundDescription": "This store does not exist or has been removed.",
    "storeInactive": "This store is not currently accepting new tickets",
    "storeClosed": "Store Unavailable",
    "poweredBy": "Powered by NotiGuide"
  },
  "queue": {
    "joinQueue": "Join Queue",
    "joining": "Joining...",
    "joinSuccess": "You've joined the queue!",
    "hasActiveTicket": "You already have an active ticket (#{number})",
    "viewActiveTicket": "View your ticket",
    "joinAnyway": "Join again",
    "yourTicket": "Your Ticket",
    "ticketNumber": "Ticket Number",
    "position": "Position in Queue",
    "positionDisplay": "#{position} in line",
    "peopleAhead": "{count} people ahead of you!",
    "noPeopleAhead": "You're next!",
    "estimatedWait": "Estimated Wait",
    "waitUnavailable": "Estimate not available",
    "status": "Status",
    "issuedAt": "Issued at",
    "calledAt": "Called at",
    "lastUpdated": "Last updated {time} ago",
    "cancelTicket": "Cancel Ticket",
    "cancelConfirmTitle": "Cancel your ticket?",
    "cancelConfirmMessage": "You will lose your position in the queue.",
    "keepTicket": "Keep my ticket",
    "yesCancel": "Yes, cancel",
    "cancelSuccess": "Your ticket has been cancelled",
    "joinAgain": "Join Queue Again",
    "calledAlert": "Your number is being called!",
    "calledAlertDescription": "Please proceed to the counter",
    "calledDismiss": "Got it",
    "servedMessage": "You have been served. Thank you!",
    "cancelledMessage": "Your ticket has been cancelled.",
    "ticketExpired": "This ticket is no longer active",
    "ticketExpiredDescription": "Your ticket may have expired or been completed. Please join the queue again if needed.",
    "refreshStatus": "Refresh Status"
  },
  "status": {
    "waiting": "Waiting",
    "called": "Called",
    "served": "Served",
    "cancelled": "Cancelled",
    "unknown": "Unknown"
  },
  "errors": {
    "networkError": "Connection lost. Please check your internet.",
    "rateLimited": "Too many requests. Please wait {seconds} seconds.",
    "serverError": "Server error. Please try again later.",
    "storeNotFound": "Store not found",
    "ticketNotFound": "Ticket not found",
    "reconnecting": "Reconnecting...",
    "offline": "Offline — last updated {time} ago"
  },
  "theme": {
    "light": "Light",
    "dark": "Dark",
    "system": "System"
  },
  "language": {
    "switchTo": "Switch language"
  },
  "time": {
    "justNow": "Just now",
    "seconds": "{count}s ago",
    "minutes": "{count}m ago"
  }
}
```

### i18n Integration Points
- Language switcher in the header/footer (flag toggle: 🇻🇳 VI / 🇬🇧 EN)
- Server components use `getTranslations()`, client components use `useTranslations()`
- Locale-aware navigation via `createNavigation()` from `next-intl`
- Ticket timestamps displayed with fixed locale formatting (not next-intl dynamic — per project convention)

---

## Phases

### Phase B0: Backend — New Public Endpoints

Before building the client frontend, the backend must expose three new public endpoints. These reuse existing service methods — no new business logic is needed, only new controller mappings and a lightweight DTO.

#### B0.1: Public Store Info Endpoint

**Purpose**: Allow the client to display the store's name, address, and active status on the landing page before the customer joins the queue.

**Implementation**:
- **New DTO**: `StorePublicInfoResponse` in `domain/queue/dto/` (alongside other queue public DTOs, since it's served by `QueuePublicController`):
  ```kotlin
  data class StorePublicInfoResponse(
      val id: UUID,
      val name: String,
      val address: String?,
      val isActive: Boolean
  )
  ```
- **New endpoint** in `QueuePublicController`:
  ```kotlin
  @GetMapping("/info")
  suspend fun getStoreInfo(@PathVariable storeId: UUID): StorePublicInfoResponse
  ```
- **Service**: Calls `StoreService.getStore(storeId)` (already exists, read-only) and maps to `StorePublicInfoResponse` — intentionally omitting `createdAt`/`updatedAt` from the public response.
- **Security**: Already covered by the existing `permitAll()` rule on `/api/queue/public/**`.
- **Error**: Returns 404 if store does not exist (standard `NotFoundException` from `StoreService.getStore`).

#### B0.2: Public Queue Size Endpoint

**Purpose**: Allow the client to show how many people are currently waiting before the customer decides to join.

**Implementation**:
- **New endpoint** in `QueuePublicController`:
  ```kotlin
  @GetMapping("/size")
  suspend fun getQueueSize(@PathVariable storeId: UUID): QueueSizeResponse
  ```
- **Service**: Calls `QueueService.getQueueSize(storeId)` (already exists, used by the admin controller).
- **Security**: Already covered by `/api/queue/public/**` permit rule.
- **Note**: Does not verify store existence — returns `{ queueSize: 0 }` for non-existent stores (same behavior as admin endpoint). This is acceptable since the store info endpoint handles existence validation.

#### B0.3: Public Cancel Ticket Endpoint

**Purpose**: Allow customers to cancel their own ticket from the client app without needing admin intervention.

**Implementation**:
- **New endpoint** in `QueuePublicController`:
  ```kotlin
  @PostMapping("/tickets/{ticketId}/cancel")
  @ResponseStatus(HttpStatus.NO_CONTENT)
  suspend fun cancelTicket(
      @PathVariable storeId: UUID,
      @PathVariable ticketId: UUID
  )
  ```
- **Service**: Calls `QueueService.cancelTicket(storeId, ticketId)` (already exists). The method signature is `suspend fun cancelTicket(storeId: UUID, ticketId: UUID)` — it takes no `AdminPrincipal` or auth context. It is fully idempotent: if the ticket doesn't exist, it logs and returns silently (204).
- **Security**: Already covered by `/api/queue/public/**` permit rule.
- **Abuse consideration**: A malicious actor who knows both the `storeId` (UUID) and `ticketId` (UUID) could cancel someone else's ticket. This is an accepted trade-off because:
  - Both IDs are UUIDs — guessing a valid combination is computationally infeasible
  - The `ticketId` is only visible to the person who issued the ticket (returned in `IssueTicketResponse`)
  - Rate limiting applies to public endpoints, preventing brute-force enumeration
  - The alternative (token-based ticket ownership) adds significant complexity for a low-risk scenario

#### B0.4: Backend Verification

After implementing the three endpoints:
1. `GET /api/queue/public/{storeId}/info` — verify it returns store name/address/isActive for a valid store, 404 for invalid
2. `GET /api/queue/public/{storeId}/size` — verify it returns queue count
3. `POST /api/queue/public/{storeId}/tickets/{ticketId}/cancel` — verify 204 response, verify ticket is actually removed from queue/serving set, verify idempotency (second call also returns 204)

---

### Phase 0: Project Structure Setup

Establish a clean, scalable directory structure inside `src/` following Next.js App Router best practices.

```text
src/
├── app/                          # Next.js App Router
│   ├── [locale]/                 # Locale route group (vi, en)
│   │   ├── layout.tsx            # Root layout (fonts, NextIntlClientProvider, metadata)
│   │   ├── page.tsx              # Landing / redirect (not used — direct store links)
│   │   ├── store/                # Store-scoped routes
│   │   │   └── [storeId]/        # Dynamic store segment
│   │   │       ├── page.tsx      # Store queue landing — "Join Queue" page
│   │   │       └── ticket/
│   │   │           └── [ticketId]/
│   │   │               └── page.tsx  # Ticket status tracking page
│   │   └── not-found.tsx         # 404 page (store not found, invalid URL)
│   ├── globals.css               # Global styles, CSS custom properties, Tailwind theme
│   └── layout.tsx                # Root HTML layout (lang, viewport meta)
├── components/                   # Shared UI components
│   ├── ui/                       # shadcn/ui primitives (Button, Card, Badge, Dialog, etc.)
│   ├── layout/                   # Layout components (Header, Footer, LanguageSwitcher, ThemeToggle)
│   ├── queue/                    # Queue-specific components
│   │   ├── join-queue-button.tsx  # The primary "Join Queue" CTA
│   │   ├── ticket-card.tsx       # Ticket display card (number, status, position)
│   │   ├── ticket-status-badge.tsx   # Status badge with color coding
│   │   ├── position-display.tsx  # "You are #3 in line" with animation
│   │   ├── called-alert.tsx      # Full-screen alert when ticket is CALLED
│   │   └── cancel-ticket-dialog.tsx  # Confirmation dialog for cancellation
│   └── store/                    # Store-specific components
│       ├── store-header.tsx      # Store name, address, status display
│       └── store-closed-banner.tsx   # Banner when store is inactive
├── features/                     # Feature modules (domain-driven)
│   ├── queue/                    # Queue feature
│   │   ├── api.ts                # Queue API call functions (join, status, cancel)
│   │   ├── hooks.ts              # Custom hooks (useTicketPolling, useJoinQueue, etc.)
│   │   └── types.ts              # Re-exports from types/ with feature-specific helpers
│   └── store/                    # Store feature
│       ├── api.ts                # Store API call functions (getStoreInfo, getQueueSize)
│       └── hooks.ts              # Custom hooks (useStoreInfo, useQueueSize)
├── lib/                          # Application utilities
│   ├── api.ts                    # Fetch client config (base URL, error parsing, rate limit handling)
│   ├── utils.ts                  # Generic helpers (cn class merge, formatTime, etc.)
│   └── constants.ts              # Global constants (polling intervals, TTLs, API routes)
├── hooks/                        # Shared custom React hooks
│   ├── use-ticket-storage.ts     # localStorage persistence for active ticket
│   └── use-vibration.ts          # Vibration API hook for called alert
├── store/                        # Global state management (Zustand)
│   └── ticket.ts                 # Active ticket state (storeId, ticketId, ticket data)
├── styles/                       # Extracted CSS for complex components
│   ├── glass.css                 # Glass/frosted card effects
│   ├── ticket.css                # Ticket card animations and styles
│   └── called-alert.css          # Called alert pulse/glow animations
├── messages/                     # i18n translation files
│   ├── en.json                   # English translations
│   └── vi.json                   # Vietnamese translations
├── i18n/                         # i18n configuration
│   ├── routing.ts                # Locale list, default locale, prefix strategy
│   ├── request.ts                # Server-side getRequestConfig
│   ├── navigation.ts             # Locale-aware Link, useRouter, usePathname
│   └── locale.ts                 # Locale utility functions
└── types/                        # Global TypeScript definitions
    ├── queue.ts                  # TicketDto, IssueTicketResponse, TicketStatusResponse, etc.
    ├── store.ts                  # StorePublicInfoResponse, QueueSizeResponse
    └── api.ts                    # ErrorResponse, API client types
```

### Phase 1: Foundation & Shared Infrastructure

- **Tailwind & Theme Setup (`globals.css`)**:
  - Override default Tailwind breakpoints with the mobile-first system (XS through 2XL)
  - Define all color tokens as CSS custom properties under `:root` (light) and `.dark` (dark) — matching the Web Styles Guide exactly
  - Map tokens to Tailwind via `@theme inline`
  - Set up border-radius tokens (`--radius` through `--radius-4xl`)

- **Font Loading (Root Layout)**:
  - Load **Be Vietnam Pro** via `next/font/google` with Vietnamese subset
  - Apply as CSS variable `--font-be-vietnam-pro` on `<html>`
  - Set viewport meta for mobile: `width=device-width, initial-scale=1, maximum-scale=1, viewport-fit=cover`

- **HTTP Client (`lib/api.ts`)**:
  - `fetch` wrapper configured with base URL pointing to the backend API
  - **No auth interceptor** — this app has no authentication
  - Response interceptor: Parse structured `ErrorResponse` JSON on non-2xx responses
  - On **429** (rate limited): Extract `X-RateLimit-Reset` header, throw a typed error with countdown seconds
  - On **404**: Throw a typed `NotFoundError` (used to distinguish "ticket expired" from real errors)
  - On **network error / timeout**: Throw a typed `NetworkError` for UI handling
  - Request timeout: 10 seconds default

- **TypeScript DTOs (`types/`)**:
  - `TicketDto` — `id: string`, `number: string`, `status: TicketStatus`, `issuedAt: string | null`, `calledAt: string | null`, `position: number | null`
  - `TicketStatus` — `"WAITING" | "CALLED" | "SERVED" | "CANCELLED" | "UNKNOWN"`
  - `IssueTicketResponse` — `storeId: string`, `ticket: TicketDto`
  - `TicketStatusResponse` — `status: TicketStatus`, `positionInQueue: number | null`, `estimatedWaitTime: number | null`
  - `StorePublicInfoResponse` — `id: string`, `name: string`, `address: string | null`, `isActive: boolean`
  - `QueueSizeResponse` — `queueSize: number`
  - `ErrorResponse` — `timestamp: string`, `code: number`, `error: string`, `message: string`, `path: string`, `method: string`, `details?: Record<string, string>`
  - **Timestamp note**: Ticket timestamps (`issuedAt`, `calledAt`) are `Instant` (UTC, serialized as `"2026-03-15T10:30:00Z"`). Convert to local timezone for display using fixed locale formatting (not next-intl).

- **Queue API Functions (`features/queue/api.ts`)**:
  - `joinQueue(storeId: string): Promise<IssueTicketResponse>` — `POST /api/queue/public/{storeId}/tickets`
  - `getTicketStatus(storeId: string, ticketId: string): Promise<TicketStatusResponse>` — `GET /api/queue/public/{storeId}/tickets/{ticketId}`
  - `cancelTicket(storeId: string, ticketId: string): Promise<void>` — `POST /api/queue/public/{storeId}/tickets/{ticketId}/cancel` (204 No Content)
  - `getStoreInfo(storeId: string): Promise<StorePublicInfoResponse>` — `GET /api/queue/public/{storeId}/info`
  - `getQueueSize(storeId: string): Promise<QueueSizeResponse>` — `GET /api/queue/public/{storeId}/size`

- **Ticket State Management (`store/ticket.ts`)**: Zustand store with `localStorage` persistence:
  - `storeId: string | null` — the store this ticket belongs to
  - `ticketId: string | null` — the active ticket UUID
  - `ticket: TicketDto | null` — full ticket data from issue response
  - `status: TicketStatusResponse | null` — latest polled status
  - `setTicket(storeId, ticket)` — save after joining queue
  - `updateStatus(status)` — update from polling
  - `clearTicket()` — clear after served, cancelled, or expired
  - `hasActiveTicket(storeId)` — check if user already has a ticket for this store
  - **Persistence**: Ticket data persists in `localStorage` so refreshing the page doesn't lose the ticket. Key: `notiguide-ticket-{storeId}`. Cleared when ticket reaches a terminal state (SERVED, CANCELLED, or 404).

- **Bilingual Setup (i18n/)**:
  - Install `next-intl` and configure with `createNextIntlPlugin()` in `next.config.ts`
  - Set up `routing.ts` with `locales: ["vi", "en"]`, `defaultLocale: "vi"`, `localePrefix: "always"`
  - Set up `request.ts` with dynamic message loading per locale
  - Set up `navigation.ts` with `createNavigation()` for locale-aware routing
  - Create `messages/en.json` and `messages/vi.json` with all translation namespaces

- **shadcn/ui Setup** (sole UI component library — see [Web Styles Guide](../walkthrough/Web%20Styles.md#component-library)):
  - Install and initialize shadcn/ui with the project's color tokens
  - Install base components: Button, Card, Badge, Dialog, AlertDialog, Skeleton
  - Customize component theme to use CSS variable tokens
  - No other UI framework should be used alongside or instead of shadcn/ui

- **Glass Effect Styles (`styles/glass.css`)**:
  - `.glass-card` — frosted card: `backdrop-blur-md`, semi-transparent surface background, subtle border, inset glow
  - `.glass-card-elevated` — stronger blur and shadow for primary content cards
  - Light mode: `rgba(255, 255, 255, 0.7)` background, `saturate(150%)`
  - Dark mode: `rgba(40, 53, 74, 0.6)` background, lower opacity, inset glow
  - Maintain WCAG AA contrast for all text over glass surfaces

### Phase 2: Store Queue Landing Page

**Route**: `/[locale]/store/[storeId]` — the entry point customers reach (via QR code, NFC tap, or direct link at the store)

- **Page Layout**:
  - **Header bar**: Store name (from `getStoreInfo`), language switcher (🇻🇳/🇬🇧), theme toggle (light/dark/system)
  - **Store info section**: Fetched on page load via `GET /api/queue/public/{storeId}/info`:
    - Display the store's **name** prominently as the page title
    - Display the store's **address** below the name (if present)
    - If the endpoint returns **404**: Show a "Store not found" page — the store doesn't exist
    - If the store's **`isActive` is `false`**: Show a disabled state with full-width banner: *"This store is not currently accepting new tickets."* Hide or disable the Join Queue button.
  - **Queue status indicator**: Fetched on page load via `GET /api/queue/public/{storeId}/size`:
    - Display *"N tickets waiting"* or *"No one is in line — you'll be next!"* when queue is empty
    - Refresh on a 15-second interval to keep the count current while the customer decides
  - **Join Queue button**: Large, prominent, full-width on mobile. Primary color. Disabled with loading spinner during request. Disabled entirely when store is inactive.
  - **Footer**: Minimal — "Powered by Notiguide" text

- **Join Queue Flow**:
  1. Customer taps "Join Queue"
  2. Button shows spinner, disabled to prevent double-tap
  3. `POST /api/queue/public/{storeId}/tickets`
  4. On **success**: Store ticket in Zustand + localStorage → redirect to `/[locale]/store/[storeId]/ticket/[ticketId]`
  5. On **404** (store not found): Show "Store not found" error page with option to go back
  6. On **429** (rate limited): Show inline message with countdown timer
  7. On **network error**: Show "Connection lost" with retry button

- **Returning User Detection**:
  - On page load, check localStorage for an existing active ticket for this `storeId`
  - If found, show a banner: *"You already have an active ticket (#042)"* with a button to view it (navigates to ticket status page)
  - The user can still choose to dismiss and join again (the backend allows multiple tickets per client — deduplication is not enforced)

- **Store Inactive Handling**:
  - On page load, `GET /api/queue/public/{storeId}/info` returns `isActive: boolean`
  - If `isActive === false`: Disable the Join Queue button, show a full-width banner: *"This store is not currently accepting new tickets"*
  - Also handle the case where `POST /api/queue/public/{storeId}/tickets` returns an error indicating the store is inactive (race condition — store deactivated between page load and button press)

- **Responsive Behavior**:
  - Base–S: Full-bleed card, button fills width, large text
  - M–L: Card centered with max-width ~480px, more padding
  - XL–2XL: Card centered with max-width ~560px, kiosk-friendly large elements

### Phase 3: Ticket Status Tracking Page

**Route**: `/[locale]/store/[storeId]/ticket/[ticketId]` — the customer's ticket view after joining the queue

- **Page Layout**:
  - **Header bar**: Same as landing page (language switcher, theme toggle, back arrow to store page)
  - **Ticket Card** (primary content — the hero of the page):
    - **Ticket number**: Large, bold, centered (28px base → 56px at 2XL). This is the `ticket.number` field (e.g., "042")
    - **Status badge**: Color-coded — green "Waiting", orange "Called", muted "Served"/"Cancelled"
    - **Position display** (when WAITING): *"You are #3 in line"* (computed as `position + 1`). Animated counter for position changes.
    - **People ahead** (when WAITING): *"2 people ahead of you"* or *"You're next!"* when position is 0
    - **Estimated wait** (when WAITING): Show only if `estimatedWaitTime` is not null (future analytics feature). Otherwise display *"Estimate not available"*
    - **Timestamps**: Issued at (always shown), Called at (shown when CALLED)
  - **Cancel Ticket button**: Outline destructive style, at bottom of card. Shows confirmation dialog before proceeding.
  - **Refresh button**: Manual refresh for status (in addition to auto-polling)

- **Status Polling**:
  - Poll `GET /api/queue/public/{storeId}/tickets/{ticketId}` every **5 seconds** while status is `WAITING`
  - Increase to every **3 seconds** when position is ≤ 3 (customer is nearly called)
  - Poll every **10 seconds** when status is `CALLED` (to detect SERVED/CANCELLED transition)
  - **Stop polling** when status is `SERVED`, `CANCELLED`, or a `404` is received
  - Use `visibilitychange` event to pause polling when the tab is backgrounded and resume when foregrounded
  - Show a subtle "Last updated: X seconds ago" indicator

- **Status Transitions (Visual)**:
  - **WAITING → CALLED**: This is the critical moment. Trigger:
    - Full-screen overlay alert with large ticket number and pulsing animation
    - Screen background shifts to Action color (orange glow)
    - Text: *"Your number is being called! Please proceed to the counter."*
    - Device vibration (Vibration API: `navigator.vibrate([200, 100, 200, 100, 400])` — three-pulse pattern)
    - Audio chime (optional, with user opt-in due to autoplay restrictions)
    - The alert persists until the user taps "Got it" / dismisses — it does not auto-close
  - **WAITING → WAITING (position change)**: Animate the position number smoothly (count down animation)
  - **CALLED → SERVED**: Show success state: *"You have been served. Thank you!"* with a green checkmark. Clear ticket from localStorage after 5 seconds, then show option to join queue again.
  - **CALLED → CANCELLED**: Show *"Your ticket has been cancelled."* Clear ticket from localStorage. Show option to rejoin.
  - **Any → 404 (ticket not found)**: Show *"This ticket is no longer active. Your ticket may have expired or been completed."* Clear ticket from localStorage. Show option to rejoin.

- **Cancel Ticket Flow** (Customer-Initiated):
  - Customer taps "Cancel Ticket"
  - Confirmation dialog (AlertDialog): *"Cancel your ticket? You will lose your position in the queue."* with "Keep my ticket" (secondary) and "Yes, cancel" (destructive) buttons.
  - On confirm: `POST /api/queue/public/{storeId}/tickets/{ticketId}/cancel` (public endpoint added in Phase B0)
  - The cancel endpoint is **idempotent** — returns 204 whether the ticket exists or not. No error handling needed for "ticket already gone" cases.
  - On **success** (204): Clear ticket from Zustand + localStorage. Show brief toast *"Your ticket has been cancelled"*. Redirect back to the store landing page after 2 seconds, or show an immediate "Join Queue Again" button.
  - On **429** (rate limited): Show "Please wait" inline message.
  - On **network error**: Show "Connection lost" with retry button. Keep the dialog open.
  - The cancel button should be styled as outline/destructive (red border, red text, no filled background) to indicate it's a destructive but secondary action.
  - **Only show** the cancel button when ticket status is `WAITING` or `CALLED`. Hide for terminal states.

- **Responsive Behavior**:
  - Base–XS: Ticket number 28–32px, full-width card, stacked layout
  - S–M: Ticket number 36–40px, card with padding, position display more prominent
  - L–XL: Centered card (max-width ~500px), ticket number 48px, more whitespace
  - 2XL: Kiosk mode — ticket number 56px, card centered in viewport, large touch targets

### Phase 4: Layout Shell & Navigation Components

> **Ordering note**: Layout shell components (header, footer, language switcher, theme toggle) are prerequisites for Phases 2 and 3. In practice, build the header and language switcher early in Phase 2 and refine them here. This phase focuses on polishing and extracting reusable layout patterns.

- **Root Layout (`app/[locale]/layout.tsx`)**:
  - Load Be Vietnam Pro font with Vietnamese subset
  - `<NextIntlClientProvider>` wrapping all content
  - `<html lang={locale}>` for accessibility and SEO
  - Viewport meta for mobile optimization
  - Dark mode support via class-based toggling (`.dark` on `<html>`)

- **Header Component (`components/layout/header.tsx`)**:
  - Minimal, non-intrusive header for customer-facing pages
  - Left: Back arrow (when on ticket page) or Notiguide logo/text
  - Right: Language switcher + Theme toggle
  - Glass effect: subtle backdrop-blur, semi-transparent background
  - Fixed position on mobile (sticky header so it's always accessible)
  - Height: 48px on mobile, 56px on tablet

- **Language Switcher (`components/layout/language-switcher.tsx`)**:
  - Toggle between 🇻🇳 VI and 🇬🇧 EN
  - Uses `useRouter()` from `@/i18n/navigation` to swap locale in URL
  - Compact pill-shaped button, fits in header

- **Theme Toggle (`components/layout/theme-toggle.tsx`)**:
  - Three states: Light ☀️, Dark 🌙, System 💻
  - Cycle on tap (no dropdown on mobile — too fiddly)
  - Persists choice in `localStorage`
  - Uses `next-themes` or manual class toggling on `<html>`

- **Footer Component (`components/layout/footer.tsx`)**:
  - Minimal: "Powered by Notiguide" text
  - Only shown on the store landing page, not on the ticket tracking page (to maximize screen real estate)

### Phase 5: Polish, Animations & Real-Time Feel

- **Polling Optimization**:
  - Adaptive polling intervals based on position (5s default → 3s when near front → 10s when called)
  - Pause on tab background via `visibilitychange` API
  - Exponential backoff on consecutive errors (5s → 10s → 20s → 30s max)
  - Show "Reconnecting..." indicator after 3 consecutive failures
  - Resume normal interval after successful response

- **Animations**:
  - **Position countdown**: When position decreases, animate the number with a slide-down + fade transition (CSS `@keyframes` or Framer Motion)
  - **Called alert pulse**: Pulsing glow ring around ticket number using CSS animation (`box-shadow` pulse with Action color)
  - **Join queue success**: Brief confetti or checkmark animation on the ticket card appearing
  - **Status badge transitions**: Smooth color transitions when status changes
  - **Skeleton loading**: Shimmer skeletons for ticket card and status while data loads

- **Haptic Feedback (Vibration API)**:
  - On ticket CALLED: Strong pattern `[200, 100, 200, 100, 400]`
  - On join success: Short pulse `[100]`
  - Feature-detect before use: `if ('vibrate' in navigator)`

- **Error Handling & User Messaging**:
  - 404 on ticket status → "Ticket no longer active" (not a scary error)
  - 404 on store → "Store not found" dedicated page
  - 429 rate limit → Inline countdown message (not a toast — it should persist)
  - Network failure → "Connection lost" banner at top with retry. Auto-retry on reconnect via `navigator.onLine` event
  - Generic server error → "Something went wrong. Please try again."

- **Accessibility**:
  - All interactive elements have proper ARIA labels and roles
  - Focus management: auto-focus ticket number on status page load
  - Screen reader announcements for status changes (`aria-live="polite"` region)
  - Called alert uses `aria-live="assertive"` for immediate announcement
  - Color is never the only indicator — all badges have text labels
  - Touch targets minimum 48×48px on mobile (WCAG 2.5.8)
  - Reduced motion: respect `prefers-reduced-motion` — disable pulse/glow animations

- **Offline Resilience**:
  - Ticket data persists in localStorage — page refresh doesn't lose ticket
  - Show last-known status with "Offline — last updated X seconds ago" when connection lost
  - Auto-resume polling when connection restored
  - Service Worker (optional, deferred): Cache the app shell for offline access, serve stale ticket data

### Phase 6: QR Code & Store Entry Integration

- **QR Code Support**:
  - Stores generate QR codes pointing to `/vi/store/{storeId}` (default Vietnamese locale)
  - The URL structure is designed to be scannable: short, no auth tokens, no query params
  - QR codes can be printed and displayed at store entrances, counters, and table cards
  - Consider adding UTM-like params for tracking source: `/vi/store/{storeId}?ref=qr-counter`

- **Deep Link Handling**:
  - If a customer scans a QR code while already having an active ticket for that store, the returning-user detection (Phase 2) handles it gracefully
  - If the locale in the URL doesn't match the user's preference, the language switcher is prominently available

- **Kiosk Mode Considerations (2XL breakpoint)**:
  - At tablet/kiosk sizes, the UI shifts to a large centered layout
  - Ticket numbers are extra large (56px+) for readability from a distance
  - Touch targets are oversized for public touchscreen use
  - Consider auto-clearing the page after 30 seconds of inactivity on the "served" screen (reset for next customer)

---

## Deferred / Blocked Items

| Item | Blocked By | Impact | Workaround |
|---|---|---|---|
| Estimated wait time display | Analytics domain not built | `estimatedWaitTime` always null | Show "Estimate not available" or hide the field entirely |

### Previously Deferred, Now Resolved in This Plan

| Item | Resolution |
|---|---|
| Customer-initiated ticket cancellation | Phase B0.3 adds `POST /api/queue/public/{storeId}/tickets/{ticketId}/cancel`. Reuses existing `QueueService.cancelTicket()` which requires no auth. |
| Store name/status display on landing | Phase B0.1 adds `GET /api/queue/public/{storeId}/info`. Returns `StorePublicInfoResponse` with name, address, isActive. |
| Queue size on landing page | Phase B0.2 adds `GET /api/queue/public/{storeId}/size`. Reuses existing `QueueService.getQueueSize()`. |
| Push notifications (FCM) | Addressed in Phase 5 via polling + Vibration API + optional audio chime. FCM integration is a future enhancement but not needed for MVP. |
| Service Worker / offline caching | Addressed in Phase 5 via localStorage persistence for ticket state + connection monitoring. Full SW is a future enhancement. |

---

## Verification Plan

- **Phase B0**: Start backend with Docker Compose. Test all three new public endpoints:
  - `GET /api/queue/public/{storeId}/info` — verify it returns store name/address/isActive, verify 404 for non-existent store
  - `GET /api/queue/public/{storeId}/size` — verify it returns queue count (0 for empty, N after issuing tickets)
  - `POST /api/queue/public/{storeId}/tickets/{ticketId}/cancel` — issue a ticket, cancel it, verify 204 and ticket is removed from queue. Call cancel again, verify idempotent 204. Verify from admin side that the ticket is gone.
- **Phase 1**: Start the Next.js dev server. Confirm the API client can reach the backend. Confirm bilingual routing works (`/vi/...` and `/en/...`). Confirm dark/light theme switching. Verify all custom breakpoints render correctly.
- **Phase 2**: Navigate to `/vi/store/{validStoreId}`. Verify store name and queue size display on page load. Tap "Join Queue". Verify ticket is issued and persisted. Verify redirect to ticket page. Test with invalid store ID (expect "Store not found" page). Test with inactive store (expect disabled join button + banner). Test returning-user detection (refresh page with existing ticket).
- **Phase 3**: On ticket status page, verify polling updates position in real time. Test customer-initiated cancel: tap "Cancel Ticket", confirm dialog, verify 204 response and redirect to store page. Have an admin call the ticket from the admin dashboard — verify the CALLED alert triggers with vibration. Have the admin serve the ticket — verify the "served" state displays. Test with an expired ticket (wait for TTL or delete from Redis) — verify 404 handling.
- **Phase 4**: Test language switching on both pages. Test theme toggle persistence. Test header back navigation. Verify responsive layouts at all breakpoints (use Chrome DevTools device emulator for 320px, 360px, 390px, 430px, 600px, 768px, 1024px).
- **Phase 5**: Test polling pause/resume on tab switch. Test offline banner when network is disconnected. Test rate limit handling (rapid-fire join attempts). Test reduced-motion preference.
- **Phase 6**: Generate a QR code for a store URL and scan it on a real phone. Test deep linking with existing ticket. Test kiosk-size layout on a tablet.
