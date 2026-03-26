# Admin Web App Implementation Plan

## Current State of Repository
The repository (`/home/thomas/Coding/notiguide/web`) is a freshly initialized Next.js project using:
- Next.js 16.1 (App Router)
- React 19
- Tailwind CSS 4
- TypeScript
- Biome (Linter and Formatter)
- Yarn Berry (Package Manager)

---

## Design System

### Visual Style
Modern glass-based aesthetic inspired by Apple Liquid Glass, Samsung OneUI 8.5, and Google Material 3. Cards, panels, and components should use frosted glass effects (backdrop-blur, semi-transparent backgrounds), subtle shadows, and smooth transitions.

### Typography
- **Primary font**: Be Vietnam Pro — used for all body text, headings, and UI labels
- **Dashboard legends**: Inter — used for chart labels, stat counters, and data-dense displays

Both fonts should be loaded via `next/font/google` for optimal performance.

### Color Palette

#### Light Theme
| Token | Hex | Usage |
|---|---|---|
| Background | `#F8FAFC` | Main app canvas |
| Surface/Cards | `#FFFFFF` | Containers, table backgrounds, queue cards |
| Cool Primary | `#0D9488` | Headers, sidebars, primary navigation |
| Cool Primary Hover | `#0F766E` | Darker shade for hover on primary elements |
| Hot Action | `#F97316` | Call Next button, active queue numbers |
| Hot Action Hover | `#C2410C` | Darker burnt orange for action button clicks |
| Text (Main) | `#0C1324` | Standard reading text and headings |
| Text (Muted) | `#64748B` | Timestamps, secondary descriptions, table headers |
| Borders & Dividers | `#E2E8F0` | Lines separating table rows or app sections |
| Success | `#10B981` | "Successfully joined queue", serve confirmations |
| Warning | `#EAB308` | "Queue paused", "Delayed", pending verification badges |
| Error | `#E11D48` | Cancel ticket, system errors, destructive actions |
| Disabled Surface | `#F1F5F9` | Greyed-out buttons, inactive inputs |
| Disabled Text | `#94A3B8` | Text on disabled elements |
| Chart Accent | `#4F46E5` | Deep indigo for admin dashboard graphs (optional) |

#### Dark Theme
| Token | Hex | Usage |
|---|---|---|
| Background | `#0F172A` | Main app canvas |
| Surface/Cards | `#28354A` | Elevated containers and queue cards |
| Cool Primary | `#2DD4BE` | Bright teal for navigation and active states |
| Cool Primary Hover | `#5EEAD4` | Lighter glowing teal for hover states |
| Hot Action | `#FB923C` | Glowing orange for active queue numbers, primary alerts |
| Hot Action Hover | `#FDBA74` | Softer peach-orange for action button clicks |
| Text (Main) | `#F8FAFC` | High-contrast white for readability |
| Text (Muted) | `#94A3B8` | Slate grey for timestamps and secondary info |
| Borders & Dividers | `#3E5270` | Slightly lighter than surface for clean separations |
| Success | `#34D399` | Bright mint green for positive confirmations |
| Warning | `#FDE047` | Bright yellow for alerts and pauses |
| Error | `#FB7185` | Soft red that doesn't bleed against dark backgrounds |
| Disabled Surface | `#1E293B` | Dark receded grey for inactive buttons |
| Disabled Text | `#64748B` | Darkened text for inactive elements |
| Chart Accent | `#818CF8` | Soft indigo/blue for dashboard graphs (optional) |

### Color Application Guide
- **Sidebar & header**: Cool Primary background with white text
- **Call Next button**: Hot Action background — this is the most important action on the queue page
- **Serve button**: Success green — the expected happy path
- **Cancel button**: Error red, outline variant — destructive, requires confirmation
- **Verification badges**: Success green (verified), Warning amber (pending)
- **Active/Inactive store badges**: Success green (active), Error red (inactive)
- **Table headers**: Text (Muted) color
- **Glass effects**: Use `backdrop-blur-md` with semi-transparent Surface colors (e.g., `rgba(255, 255, 255, 0.7)` light / `rgba(40, 53, 74, 0.7)` dark) for card backgrounds

### CSS Architecture
- Define color tokens as CSS custom properties in `globals.css` (scoped under `:root` for light and `[data-theme="dark"]` or `@media (prefers-color-scheme: dark)` for dark)
- Complex or glass-effect styles should be extracted to separate CSS files (per project convention — avoid lengthy Tailwind inline)
- Use Tailwind for layout, spacing, and simple utility styles

---

## Phases

### Phase 0: Project Structure Setup
Establish a clean, scalable directory structure inside `src/` following Next.js App Router best practices.

```text
src/
├── app/                  # Next.js App Router (Pages, Layouts, API Routes)
│   ├── (auth)/           # Route group for authentication (not in URL)
│   │   └── login/        # /login page
│   ├── dashboard/        # Authenticated admin routes (/dashboard/...)
│   │   ├── layout.tsx    # Dashboard shell (Sidebar + Topbar + auth guard)
│   │   ├── page.tsx      # /dashboard landing (redirect to queue or overview)
│   │   ├── stores/       # /dashboard/stores (Super Admin only)
│   │   ├── admins/       # /dashboard/admins (all roles, scoped by authorization)
│   │   ├── queue/        # /dashboard/queue (all roles)
│   │   └── settings/     # /dashboard/settings
│   │       └── password/ # /dashboard/settings/password (change password)
│   ├── globals.css       # Global styles, CSS custom properties (color tokens, fonts)
│   └── layout.tsx        # Root layout (font loading via next/font/google)
├── components/           # Shared UI components
│   ├── ui/               # Generic UI primitives (Buttons, Inputs, Modals, Toasts)
│   └── layout/           # Layout components (Sidebar, Topbar, AuthGuard)
├── features/             # Feature-specific modules (Domain-driven)
│   ├── auth/             # Auth-specific components, hooks, api calls
│   ├── store/            # Store management components, hooks, api calls
│   ├── admin/            # Admin management components, hooks, api calls
│   └── queue/            # Queue operations components, hooks, api calls
├── lib/                  # Application utilities
│   ├── api.ts            # Fetch client config with interceptors
│   ├── utils.ts          # Generic helper functions (e.g., class merge)
│   └── constants.ts      # Global constants (Roles, API routes, error messages)
├── hooks/                # Shared custom React hooks (e.g., useDisclosure)
├── store/                # Global state management (Zustand)
│   └── auth.ts           # Auth state (token, admin profile, role helpers)
├── styles/               # Extracted CSS modules for complex components
│   ├── glass.css         # Glass/frosted card effects (backdrop-blur, transparency)
│   ├── sidebar.css       # Sidebar navigation styles
│   └── queue.css         # Queue dashboard component styles
└── types/                # Global TypeScript definitions & DTOs
    ├── admin.ts           # AdminDto, LoginResponse, CreateAdminRequest, etc.
    ├── store.ts           # StoreDto, CreateStoreRequest, UpdateStoreRequest
    ├── queue.ts           # TicketDto, NextTicketResponse, TicketStatusResponse
    └── api.ts             # ErrorResponse, API client types
```

### Phase 1: Foundation & Shared Infrastructure

- **HTTP Client (`lib/api.ts`)**: Set up a `fetch` wrapper configured with:
  - Base URL pointing to the backend API.
  - Request interceptor: Attach `Authorization: Bearer <jwt>` header from stored token.
  - Response interceptor: Parse structured `ErrorResponse` JSON (`{ timestamp, code, error, message, path, method, details? }`). On 401, clear auth state and redirect to `/login`. The response interceptor must NOT auto-redirect on 401 during the login request itself — only on authenticated routes.
  - On 429 (rate limited): Display *"Too many requests. Please wait [N] seconds."* using the `X-RateLimit-Reset` header or `ErrorResponse.message`.
  - On network error / timeout: Display *"Connection lost. Please check your internet."* with retry option.
- **TypeScript DTOs (`types/`)**: Define types matching the backend exactly:
  - `AdminDto` — `id: string` (UUID), `username: string`, `role: 'ROLE_ADMIN' | 'ROLE_SUPER_ADMIN'`, `storeId: string | null`, `isVerified: boolean`, `createdBy: string | null`, `verifiedBy: string | null`, `verifiedAt: string | null`, `createdAt: string | null`, `updatedAt: string | null`
  - `StoreDto` — `id: string`, `name: string`, `address: string | null`, `isActive: boolean`, `createdAt: string | null`, `updatedAt: string | null`
  - `LoginResponse` — `token: string`, `admin: AdminDto`
  - `AdminPageResponse` — `items: AdminDto[]`, `page: number`, `size: number`, `totalItems: number`, `totalPages: number`
  - `StorePageResponse` — `items: StoreDto[]`, `page: number`, `size: number`, `totalItems: number`, `totalPages: number`
  - `TicketDto` — `id: string`, `number: string`, `status: TicketStatus`, `issuedAt: string | null`, `calledAt: string | null`, `position: number | null`
  - `TicketStatus` — `'WAITING' | 'CALLED' | 'SERVED' | 'CANCELLED' | 'UNKNOWN'`
  - `NextTicketResponse` — `ticket: TicketDto | null`
  - `QueueSizeResponse` — `queueSize: number`
  - `TicketStatusResponse` — `status: TicketStatus`, `positionInQueue: number | null`, `estimatedWaitTime: number | null` (always `null` until the analytics domain is built; backend type is `Long?` representing seconds)
  - `ErrorResponse` — `timestamp: string` (LocalDateTime, no timezone — debugging only), `code: number`, `error: string`, `message: string`, `path: string`, `method: string`, `details?: Record<string, string>` (field-level validation errors on 400 responses)
  - `LoginRequest` — `username: string`, `password: string`
  - `CreateAdminRequest` — `username: string`, `password: string`, `role: 'ROLE_ADMIN' | 'ROLE_SUPER_ADMIN'`, `storeId: string | null`
  - `UpdatePasswordRequest` — `oldPassword: string`, `newPassword: string`
  - `CreateStoreRequest` — `name: string`, `address?: string`
  - `UpdateStoreRequest` — `name?: string`, `address?: string | null`, `isActive?: boolean`
  - **Timestamp note**: Admin/Store timestamps are `OffsetDateTime` (with timezone offset). Ticket timestamps (`issuedAt`, `calledAt`) are `Instant` (UTC, serialized as `"2026-03-15T10:30:00Z"`). Convert ticket timestamps to local timezone in the UI via `new Date(isoString).toLocaleString()`.
- **Authentication State (`store/auth.ts`)**: Zustand store holding:
  - `token: string | null` — the raw JWT string.
  - `admin: AdminDto | null` — the full admin profile from `LoginResponse`. The JWT only contains `sub` (admin UUID) and `auth` (role) — `storeId` and `isVerified` are not in the JWT, so the full `AdminDto` must be persisted.
  - `isAuthenticated` — derived from token presence.
  - `isSuperAdmin` — derived from `admin.role === 'ROLE_SUPER_ADMIN'`.
  - `storeId` — derived from `admin.storeId`.
  - `login(response: LoginResponse)` — persist token + admin.
  - `logout()` — clear all state and redirect.
  - `updateAdmin(admin: AdminDto)` — update the admin profile in state (used after password change or profile refresh).
  - **On app load / page refresh**: If a stored token exists, call `GET /api/admins/me` to re-hydrate the admin profile. If this returns 401 (token expired or admin deleted), clear stored token and redirect to `/login`.
- **Auth Persistence**: Store the JWT in `localStorage` for persistence across refreshes. The `admin` profile should be stored alongside it. JWT has 24h expiry — consider checking the `exp` claim before making requests (decode without verification) to preemptively redirect instead of waiting for a 401.
- **UI Architecture**:
  - **Sidebar**: Navigation links. Conditionally show/hide menu items based on `isSuperAdmin` (e.g., "Stores" only visible for Super Admin; "Admins" visible to all roles). Highlight the active route.
  - **Topbar**: Display logged-in admin's username, role badge, and a logout button. If `ROLE_ADMIN`, show the assigned store name.
  - **Page titles**: Each page should have a clear title bar (e.g., "Store Management", "Queue — Store Name", "Admin Directory").
- **Design System Setup**:
  - Load **Be Vietnam Pro** (primary) and **Inter** (dashboard legends) via `next/font/google` in root layout.
  - Define color tokens as CSS custom properties in `globals.css` — light theme under `:root`, dark theme under `[data-theme="dark"]` or `@media (prefers-color-scheme: dark)`.
  - Set up glass-effect base styles in `styles/glass.css` — frosted card backgrounds using `backdrop-blur-md` + semi-transparent surface colors.
  - Map color tokens to Tailwind config via `theme.extend.colors` referencing CSS variables (e.g., `primary: 'var(--color-primary)'`).
- **Component Library**: **shadcn/ui is the sole UI component library** — no other UI framework should be used (see [Web Styles Guide](../walkthrough/Web%20Styles.md#component-library)). Install and initialize shadcn/ui. Customize the component theme to use the project's color tokens (Cool Primary for primary actions, Hot Action for queue actions).

### Phase 2: Authentication Flow
- **Login Page (`/login`)**:
  - Form accepting `username` and `password`.
  - Submit to `POST /api/auth/login` with `LoginRequest` body.
  - On **success** (200): Extract `LoginResponse` → store JWT and `AdminDto` in auth state → redirect to `/dashboard`.
  - On **403 with message containing "not been verified"**: Show an inline alert/banner (not a toast — it should persist): *"Your account has not been verified yet. Please contact a Super Admin."* Do not redirect.
  - On **401 (BadCredentials)**: Show *"Invalid username or password"* inline under the form.
  - Disable the submit button while the login request is in-flight to prevent double-submission.
  - If user is already authenticated (token exists in storage), redirect away from `/login` to `/dashboard` immediately.
- **Protected Routes**: Wrap all `dashboard/` routes in an authentication guard via the dashboard `layout.tsx`. If no valid token exists, redirect to `/login`. If `GET /api/admins/me` returns 401 on rehydration, clear stored token and redirect. Show a full-page loading spinner during rehydration (not a flash of the dashboard then redirect).
- **Password Change Page (`/dashboard/settings/password`)**:
  - Available to **all** authenticated admins as a voluntary self-service feature.
  - Form with "Current Password", "New Password", and "Confirm New Password" fields.
  - Submit to `PATCH /api/admins/{id}/password` with `UpdatePasswordRequest` body. `{id}` is taken from `admin.id` in auth state.
  - **Client-side validation**:
    - Min 8, max 128 characters (matches backend `@Size(min = 8, max = 128)`).
    - Must contain at least one uppercase letter, one lowercase letter, and one digit (matches backend `@Pattern`).
    - New password and confirm password must match.
    - New password must differ from old password.
  - On **success**: Update the `admin` object in auth state with the returned `AdminDto` (to sync `updatedAt`). Show a success toast. Clear the form fields.
  - On **400 "Old password is incorrect"**: Show inline error on the "Current Password" field.
  - **Known limitation**: Super Admins cannot reset other admins' passwords. Recovery path for forgotten passwords: delete and recreate the admin account.

### Phase 3: Store Management (Super Admin)

- **Store List (`/dashboard/stores`)**: Fetch and display a paginated table of all stores.
  - Calls `GET /api/stores?page=0&size=20` (`ROLE_SUPER_ADMIN` only).
  - Response is `StorePageResponse`.
  - Display columns: Name, Address, Active Status, Created At.
  - Active Status: colored badge — green "Active" / red "Inactive".
  - Action buttons per row: Edit, Delete.
  - **Pagination controls**: Previous/Next buttons with page indicator. Respect `totalPages` to disable navigation at boundaries.
  - **Loading state**: Table skeleton with shimmer rows.
  - **Empty state**: *"No stores yet."* with a prominent "Create Store" button.
  - **Error state**: Inline error banner with retry button.
- **Create Store**: Modal or page with a form.
  - Fields: `name` (required, non-blank, max 255 chars), `address` (optional, max 1000 chars).
  - Submit to `POST /api/stores` with `CreateStoreRequest`.
  - On success: close the modal/navigate back, show success toast, refresh the store list.
- **Edit Store**: Form pre-filled with current store data.
  - Fields: `name`, `address`, `isActive` toggle.
  - The `isActive` toggle should include helper text: *"Inactive stores cannot accept new queue tickets."*
  - Submit to `PUT /api/stores/{id}` with `UpdateStoreRequest` (all fields optional — only send changed fields).
  - To clear the address, send `"address": null` in the request body. To leave it unchanged, omit the `address` key entirely. The backend distinguishes these via an internal `addressProvided` flag.
- **Delete Store**: Confirmation dialog with warning: *"Are you sure you want to delete [Store Name]? This action cannot be undone."*
  - On delete attempt → `DELETE /api/stores/{id}`.
  - On **409 (ConflictException)**: Display the backend message verbatim inside the confirmation dialog (not as a toast). Possible messages:
    - *"Store has active queue tickets. Drain or clear the queue before deleting."*
    - *"Store has assigned admins. Remove or reassign admins before deleting the store."*
- **Authorization**: Hide these routes from the sidebar for non-Super-Admin users. If a `ROLE_ADMIN` navigates directly to `/dashboard/stores`, redirect to `/dashboard` or show a "forbidden" state.

### Phase 4: Admin Management

- **Admin Directory (`/dashboard/admins`)** — visible to **all roles** in the sidebar:
  - For `ROLE_SUPER_ADMIN`: Calls `GET /api/admins?page=0&size=20`. Shows all admins. Provide a filter dropdown to narrow by store (`storeId` query param). Optionally add a role filter (All / Super Admin / Admin).
  - For `ROLE_ADMIN`: Calls `GET /api/admins?storeId={admin.storeId}&page=0&size=20`. Auto-filters to their own store. Hide the store filter dropdown, the "Create Admin" button, and the Verify/Delete action buttons.
  - Table columns: Username, Role, Store (name, resolved from store data), Verified Status, Created At.
  - **Verified Status**: Colored badge — green checkmark "Verified" / amber "Pending Verification". All roles (including Super Admin) require manual verification.
  - Action buttons per row (Super Admin only):
    - **Verify** (if `isVerified === false`) → `PATCH /api/admins/{id}/verify`. Disable for already-verified admins (greyed out). Disable on the logged-in admin's own row with tooltip: *"You cannot verify your own account."* On success, update the row in-place with the returned `AdminDto` to reflect the new badge immediately. On 409 ("already verified"), show a toast.
    - **Delete** → `DELETE /api/admins/{id}`. Show a confirmation dialog. Disable for the currently logged-in admin's own row with tooltip: *"You cannot delete your own account."*
  - **Loading state**: Table skeleton with shimmer rows.
  - **Empty state**: *"No admins found."* with "Create Admin" CTA (Super Admin only).
- **Create Admin** (Super Admin only): Form with:
  - `username` — required, min 3 chars, max 100 chars, alphanumeric + underscores only (`[a-zA-Z0-9_]`). Usernames are normalized to lowercase on the backend, so display them lowercase in the UI.
  - `password` — required, min 8 chars, max 128 chars, must contain uppercase + lowercase + digit. Add a show/hide toggle.
  - `role` selector — `ROLE_ADMIN` or `ROLE_SUPER_ADMIN`.
  - `storeId` selector — **shown only when `role === ROLE_ADMIN`**. When `ROLE_SUPER_ADMIN` is selected, hide the store selector and send `storeId: null`. Populate from `GET /api/stores?size=100` (backend caps page size at 100; acceptable for initial deployment).
  - Role-specific notes:
    - `ROLE_SUPER_ADMIN`: *"Super Admins have access to all stores. This account will need to be verified by another Super Admin before it can log in."*
    - `ROLE_ADMIN`: *"This admin will need to be manually verified before they can log in."*
  - On **409 ("Username already taken")**: Show inline error on the username field, not just a toast.
  - All roles are created with `isVerified = false` and require manual verification.

### Phase 5: Queue Operations

- **Store Context for Queue**:
  - `ROLE_SUPER_ADMIN`: Show a store selector dropdown at the top of the queue page, populated from `GET /api/stores?size=100`. Persist the selected store in session storage so it survives navigation. With no store selected, show a prompt to select a store before displaying queue controls.
  - `ROLE_ADMIN`: Auto-select and **lock** to `admin.storeId` from auth state. No selector displayed.
  - All queue API calls use the selected `storeId` as a path parameter.
  - If the selected store is inactive, show a warning banner: *"This store is currently inactive. No new tickets can be issued."*
- **Queue Dashboard (`/dashboard/queue`)**:
  - **Page Title**: Show the selected store's name in the header (e.g., *"Queue — MegaMart"*).
  - **Queue Statistics**: Display current queue size via `GET /api/queue/admin/{storeId}/size` → `QueueSizeResponse`.
  - **Call Next Button**: `POST /api/queue/admin/{storeId}/next`. Optional text input for `counterId` query parameter (max 100 chars).
    - Disable the button immediately on click and show a spinner inside it. Re-enable only after the response. This prevents double-clicks calling two tickets.
    - Response `{ ticket: TicketDto }` → display the called ticket info (number, status, calledAt).
    - Response `{ ticket: null }` → show "Queue is empty" indicator (subtle status, not an error).
    - Ghost ticket skipping is handled entirely by the backend (`callNextUntilSuccess`). The spinner communicates any delay from the loop.
  - **Currently Serving Display**: Show the most recently called ticket info from the `callNext` response. Persist in Zustand with `sessionStorage` middleware so it survives navigation within the session. Display prominently with large ticket number, status badge, and timestamps (issuedAt, calledAt).
  - **Ticket Actions** (on the currently serving or looked-up ticket):
    - **Serve**: `POST /api/queue/admin/{storeId}/tickets/{ticketId}/serve` → 204 No Content.
      - Style as primary green button. No confirmation dialog needed (happy path).
      - On success: clear the serving display, show toast *"Ticket #42 served"*.
      - Serve is fully idempotent — it never returns 404. If the ticket was already handled or expired, the backend silently cleans up and returns 204.
    - **Cancel**: `POST /api/queue/admin/{storeId}/tickets/{ticketId}/cancel` → 204 No Content.
      - Style as destructive red/outline button. Show a confirmation dialog: *"Cancel ticket #42? The customer will be removed from the queue."*
      - On success: clear the serving display, show toast *"Ticket #42 cancelled"*.
      - Cancel is also fully idempotent — same 204 behavior as serve.
    - Both buttons must be disabled during the request with a loading spinner.
  - **Ticket Lookup**: Input field to look up a specific ticket by ID → `GET /api/queue/admin/{storeId}/tickets/{ticketId}`.
    - Returns `TicketStatusResponse` (`status`, `positionInQueue`, `estimatedWaitTime`).
    - Display result in a card showing status, position (if WAITING), and timestamps.
    - On 404: show *"Ticket not found. It may have expired or been completed."*
  - **Stale Entry Cleanup**: Button labeled *"Remove Stale Entries"* with tooltip: *"Removes tickets from the serving list that have expired or no longer exist. Use this if the serving count seems incorrect."*
    - Trigger `POST /api/queue/admin/{storeId}/cleanup`.
    - Returns `{ cleanedEntries: number }` → display: *"Cleaned 3 stale entries"* or *"No stale entries found — serving list is clean."*
  - **Active Tickets List**: Deferred until the backend exposes a serving/waiting ticket listing endpoint. For now, the dashboard operates via Call Next + individual ticket lookups only.

### Phase 6: Polish and Real-time Capabilities

- **Polling**: Implement periodic polling (React Query with `refetchInterval`) to keep the Queue Dashboard live (queue size, ticket statuses) without manual refreshes.
  - Recommended polling interval: 5–10 seconds for queue stats.
  - Use `staleTime` / `refetchOnWindowFocus` to avoid redundant fetches.
- **Error Handling & Toasts**: Ensure all HTTP errors translate into user-friendly toast notifications. Parse the `message` field from `ErrorResponse`. For 400 validation errors, check the `details` field for field-level messages. Error-to-UI mapping:
  - 401 → Auto-logout + redirect to login.
  - 403 "not been verified" → "Account not yet verified" banner.
  - 403 "Only elevated admins" → "You don't have permission" toast.
  - 409 → Conflict-specific message (username taken, store has active queue, already verified, etc.).
  - 404 → "Resource not found" toast.
  - 429 → Rate limit message with retry timer.
  - Network errors / timeouts → "Connection lost. Please check your internet." with retry.
- **Responsive Design**: Ensure the dashboard works on tablet-sized screens (primary use case for store counters). The queue page is the highest priority for tablet responsiveness — the Call Next / Serve / Cancel buttons should be large and touch-friendly.
- **Keyboard Shortcuts (Queue Page)**:
  - `N` or `Space` → Call Next
  - `S` → Serve current ticket
  - `C` → Cancel current ticket (opens confirmation)
  - `Esc` → Dismiss modals/confirmations
  - Show shortcut hints next to each button (e.g., "Call Next (N)"). Only activate when no input field is focused.
- **Accessibility**: Ensure all interactive elements have proper ARIA labels, focus states, and keyboard navigation. Color badges must not rely on color alone — include text labels.

---

## Deferred Backend Work

| Endpoint | Blocked By | Impact |
|---|---|---|
| `GET /api/queue/admin/{storeId}/serving` — list serving tickets | Analytics domain deployment | Queue dashboard cannot show a live list of all serving tickets. Workaround: individual ticket lookups. |

---

## Verification Plan

- **Phase 1**: Start the Next.js dev server. Confirm the API client can reach the backend. Confirm a request to a public endpoint succeeds.
- **Phase 2**: Test login with valid credentials, invalid credentials, and an unverified admin (expect 403). Verify token persistence across page refresh.
- **Phase 3**: Create, edit, toggle active status, and delete stores. Attempt deleting a store with active tickets.
- **Phase 4**: Create admins with both roles. Verify storeId conditional logic. Verify, then attempt to re-verify an admin.
- **Phase 5**: Test Call Next, Serve, Cancel flows. Test with empty queue. Test cleanup.
