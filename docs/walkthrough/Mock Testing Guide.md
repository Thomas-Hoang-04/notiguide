# Mock Testing Guide

This guide walks you through testing every major feature of both frontends (admin-web and client-web) using the built-in mock data layer. No backend required.

---

## How Mock Mode Works

Both `web/` and `client-web/` ship a `MockInit` component that patches `globalThis.fetch` at module-load time. Every `/api/*` request is intercepted and routed to in-memory handlers that simulate the backend.

**Currently active by default** — both `layout.tsx` files import `<MockInit />` with the comment `// MOCK: remove for production`.

When mock mode is active, you will see this in the browser console:

```
[MOCK] API mocking active — all /api/* calls are intercepted
```

### Starting the apps

```bash
# Admin dashboard
cd web && yarn dev    # → http://localhost:3000

# Client queue app
cd client-web && yarn dev    # → http://localhost:3001
```

---

## Part 1: Admin Dashboard (`web/`)

### 1.1 — Login

| Account        | Role             | Store              | Verified |
|----------------|------------------|--------------------|----------|
| `superadmin`   | SUPER_ADMIN      | None               | Yes      |
| `maria_admin`  | SUPER_ADMIN      | None               | Yes      |
| `john_doe`     | ADMIN            | Downtown Branch    | Yes      |
| `bob_wilson`   | ADMIN            | Airport Terminal 3 | Yes      |
| `alice_chen`   | ADMIN            | Airport Terminal 3 | Yes      |
| `jane_smith`   | ADMIN            | Downtown Branch    | No       |
| `carlos_reyes` | ADMIN            | Central Station Hub| No       |

**Steps:**

1. Open `http://localhost:3000` — you should be redirected to the login page.
2. Enter `superadmin` as the username. Any password works in mock mode.
3. Click **Login**. You should land on the dashboard.
4. Try `jane_smith` — login should be rejected with a "not verified" error (403).
5. Try a non-existent username — login should fail with "Invalid username or password" (401).

**What to verify:**
- Toast notification on success/failure
- Redirect to dashboard after successful login
- Unverified account gets a clear error message, not a generic one

---

### 1.2 — Admin Directory (SUPER_ADMIN only)

**Steps:**

1. Log in as `superadmin`.
2. Navigate to **Admin Directory** (sidebar).
3. You should see all 7 admins listed in a table with columns: Username, Role, Store, Verified status, Created date.
4. **Filter by store** — select "Downtown Branch" from the filter. Only `john_doe` and `jane_smith` should appear.
5. **Filter by role** — select "ROLE_ADMIN". Super admins should be excluded.
6. **Verify an admin** — click the verify action on `jane_smith`. Her badge should change from unverified to verified.
7. **Create admin** — click the create button:
   - Enter a username, select ROLE_ADMIN, pick a store.
   - The new admin should appear in the table with unverified status.
   - Try creating with an existing username — should get a conflict error.
8. **Assign store** — use the assign action on an admin to change their store. The store column should update.
9. **Delete admin** — delete a non-super admin. Confirm they disappear from the list.

**What to verify:**
- Store names display correctly (never raw IDs like `s-001`)
- Pagination works if more than the page size
- Role badges render with appropriate colors
- Actions are disabled/hidden based on permissions

---

### 1.3 — Store Management

**Pre-loaded stores:**

| Store                | Active | Jump Call | Queue State |
|----------------------|--------|-----------|-------------|
| Downtown Branch      | Yes    | Yes       | ACTIVE      |
| Airport Terminal 3   | Yes    | No        | ACTIVE      |
| Mall Kiosk           | No     | No        | ACTIVE      |
| Central Station Hub  | Yes    | Yes       | PAUSED      |

**Steps:**

1. Navigate to **Stores** from the sidebar.
2. Verify all 4 stores are listed with correct names, addresses, and active status.
3. **Create a new store** — fill in name and address. It should appear in the list.
4. **Edit a store** — change the name or toggle active/jump-call flags. Verify the change persists.
5. **Delete a store** — remove one and verify it disappears.

**What to verify:**
- Inactive stores (Mall Kiosk) visually distinct
- Address shows as empty/none for Mall Kiosk (null address)
- All text is localized — switch between English and Vietnamese

---

### 1.4 — Store Settings

**Steps:**

1. Navigate to store settings for **Downtown Branch**.
2. Verify the pre-loaded settings:
   - Max queue size: 50
   - Grace period: 120 seconds
   - No-show action: REQUEUE
   - Max requeues: 2
   - Requeue offset: 3
   - Alert threshold: 3
3. **Edit settings** — change the grace period to 90, save. Verify the update persists.
4. Check **Airport Terminal 3** settings — no-show action should be "SKIP" (max requeues = 0).
5. Check **Central Station Hub** — max queue size = 0 (unlimited).

**What to verify:**
- Form validation (min/max bounds on fields)
- Conditional fields (requeue options hidden when no-show action is SKIP)
- Success toast on save

---

### 1.5 — Service Types

**Pre-loaded service types:**

| Store               | Service Type       | Prefix | Active |
|---------------------|--------------------|--------|--------|
| Downtown Branch     | General Inquiry    | GEN    | Yes    |
| Downtown Branch     | Returns & Exchange | RET    | Yes    |
| Downtown Branch     | VIP Service        | VIP    | No     |
| Airport Terminal 3  | Check-in Assistance| CHK    | Yes    |
| Airport Terminal 3  | Baggage Claim      | BAG    | Yes    |
| Central Station Hub | Ticket Purchase    | TIX    | Yes    |
| Central Station Hub | Lost & Found       | LNF    | Yes    |

**Steps:**

1. Navigate to service types for **Downtown Branch**.
2. You should see 3 service types. VIP Service should be marked inactive.
3. **Create a service type** — add "Warranty Claims" with prefix "WRN". It should appear in the list.
4. **Duplicate prefix** — try adding another with prefix "GEN". Should get an error.
5. **Toggle active/inactive** — disable "General Inquiry", verify the badge updates.
6. **Delete a service type** — remove one, verify it disappears.
7. Check **Mall Kiosk** — should have no service types (empty state).

**What to verify:**
- Prefix is auto-uppercased
- Inactive types visually distinct
- Empty state shown for stores with no service types

---

### 1.6 — Queue Operations

**Steps:**

1. Log in as `superadmin` or `john_doe`.
2. Navigate to the **Queue** page. Select **Downtown Branch** (has 7 waiting tickets).
3. **View waiting list** — you should see 7 tickets numbered 90-96 with their positions.
4. **Call next** — click "Call Next". Ticket #90 should be removed from the waiting list and shown as CALLED. Queue size decrements to 6.
5. **Call next again** — ticket #91 is called. Queue size is now 5.
6. **Jump-call** (Downtown Branch has `allowJumpCall: true`):
   - Find ticket #95 in the waiting list.
   - Use the jump-call action to call it out of order.
   - It should be removed from the waiting list, and remaining tickets re-numbered.
7. **Serve a ticket** — mark a called ticket as served. It should be removed.
8. **Cancel a ticket** — cancel a waiting ticket. It should be removed and positions renumbered.
9. **Pause the queue** — click pause. The queue state should change to PAUSED. "Call Next" should be disabled or return an error.
10. **Resume the queue** — resume. Call Next should work again.
11. Switch to **Central Station Hub** — queue is pre-paused with 12 waiting tickets. Try to call next — should get "Queue is paused" error. Resume first, then call.

**What to verify:**
- Ticket numbers shown (not ticket IDs)
- Queue size counter updates in real-time after each action
- Pause/resume toggle works correctly
- Jump-call button only visible for stores with `allowJumpCall: true`
- Positions renumber correctly after removal

---

### 1.7 — Analytics Dashboard

**Steps:**

1. Navigate to **Analytics** from the sidebar.
2. **Overview panel** should show:
   - Active stores: 3
   - Total queue size: 22
   - Total serving: 6
   - Total issued today: 108
   - Avg wait: 8.6 min
3. **Period overview** — switch between TODAY / D7 / D30. Total issued: 755, completed: 679.
4. **Store ranking table** — stores sorted by issued count:
   1. Central Station Hub (380)
   2. Downtown Branch (245)
   3. Airport Terminal 3 (130)
   4. Mall Kiosk (0)
5. **Per-store analytics** — click into Downtown Branch:
   - Realtime: 7 in queue, 2 serving, 34 issued today, ~8.5 min avg wait
   - Summary: 245 issued, 218 completed, 6.1% cancel rate, peak hour: 11
6. **Peak hours chart** — should show a heatmap or bar chart with spikes at 10-13 and 16-17.
7. **Daily throughput chart** — switch between D7, D30, D90. Verify data renders.

**What to verify:**
- Store names displayed (not IDs) in ranking table
- Null values handled gracefully (Mall Kiosk has null avgWait)
- Charts render without errors
- Period switcher works

---

### 1.8 — Security Settings

**Steps:**

1. Log in as `superadmin`.
2. Navigate to **Settings > Security**.
3. **Login history** — should show 7 entries with timestamps, IPs, and success/failure badges.
4. **Sessions** — should show 2 sessions:
   - Current session (desktop, 192.168.1.10) — marked as "This device"
   - Non-current session (iPhone, 172.16.0.1)
5. **Revoke a session** — revoke the non-current session. It should disappear.
6. **Load more** login history entries if pagination is available.
7. Log in as `john_doe` — should see 4 login history entries and 1 session.

**What to verify:**
- Browser/OS parsing from user agent strings
- "This device" badge on current session
- Success badge (green) vs failure badge (red) on login history
- Timestamps formatted correctly (not raw ISO strings)
- No session IDs displayed

---

### 1.9 — SSE Connection (Admin Queue)

**Steps:**

1. Open browser DevTools > Network tab, filter by "EventSource" or "events".
2. Navigate to the queue page for any store.
3. You should see a request to `/api/queue/admin/{storeId}/events` with `Content-Type: text/event-stream`.
4. The connection stays open. In mock mode, it sends heartbeat comments (`: heartbeat`) every 30 seconds.
5. Navigate away and back — verify the old connection is closed and a new one opens.

**What to verify:**
- SSE connection established without errors
- No console errors about failed EventSource connections
- Connection properly cleaned up on unmount

---

## Part 2: Client Queue App (`client-web/`)

### 2.1 — Store Page

**Pre-loaded stores:**

| Store ID | Store Name          | Queue State | Max Queue | Queue Size |
|----------|---------------------|-------------|-----------|------------|
| s-001    | Downtown Branch     | ACTIVE      | 50        | 7          |
| s-002    | Airport Terminal 3  | ACTIVE      | 100       | 3          |
| s-003    | Mall Kiosk          | ACTIVE      | 20        | 0          |
| s-004    | Central Station Hub | PAUSED      | 0         | 12         |
| s-005    | City Hall Office    | ACTIVE      | 5         | 5 (full)   |

**Steps:**

1. Open `http://localhost:3001/store/s-001`.
2. You should see "Downtown Branch" as the store name, with address "123 Main St, Suite 200".
3. Queue size should show 7 people waiting.
4. The **Join Queue** button should be enabled.

**What to verify:**
- Store name displayed (not `s-001`)
- Address shown when available
- Queue size is visible

---

### 2.2 — Join Queue

**Steps:**

1. On the Downtown Branch page (`/store/s-001`), click **Join Queue**.
2. A ticket should be issued. You should be redirected to the ticket page.
3. The ticket should show:
   - Ticket number (e.g., `43`) — not the internal ID (`t-s-001-43`)
   - Status: WAITING
   - Position in queue (e.g., 8)
   - Estimated wait time
4. Queue size on the store page should increment by 1.

**What to verify:**
- Ticket number is human-readable
- Position and wait time update on subsequent polls
- Ticket data persisted in localStorage (survives page refresh)

---

### 2.3 — Ticket Lifecycle Scenarios

The mock cycles through 5 scenarios based on the ticket counter. Each ticket you join will experience a different lifecycle. Poll frequency is ~5 seconds.

**Scenario A — Waiting (steady):**
- Status stays WAITING.
- Position slowly decrements every 4 polls.
- Wait time decreases proportionally.

**Scenario B — Called Soon:**
- After 3 polls (~15 seconds), status changes to CALLED.
- Position becomes null. A "You've been called!" alert should appear.

**Scenario C — Served Soon:**
- After 3 polls, status changes to CALLED.
- After 6 polls, status changes to SERVED.
- The ticket should show a "completed" state.

**Scenario D — Skipped:**
- After 4 polls, status changes to CALLED.
- After 6 polls, status changes to SKIPPED (no-show).
- UI should show appropriate "skipped" messaging.

**Scenario E — Requeued:**
- After 3 polls, CALLED.
- After 5 polls, REQUEUED at position 2 with 6 min wait.
- After 8 polls, CALLED again.
- After 10 polls, SERVED.

**Steps:**

1. Join the queue 5 times (on different stores or after cancelling). Each ticket cycles through a different scenario.
2. For each ticket, stay on the ticket page and watch the status transitions.
3. Verify the UI updates correctly for each status change.

**What to verify:**
- Status badge changes color/text appropriately
- Position display disappears when CALLED
- "Called" alert/notification appears
- SERVED and SKIPPED are clearly terminal states
- REQUEUED shows the new position

---

### 2.4 — Cancel Ticket

**Steps:**

1. Join a queue and get a WAITING ticket.
2. Click **Cancel** on the ticket page.
3. The ticket should move to CANCELLED status.
4. Queue size for that store should decrement.
5. The "Join Queue" button on the store page should become available again.

**What to verify:**
- Cancel confirmation dialog appears
- After cancellation, localStorage is cleared for this ticket
- Re-joining the same store works

---

### 2.5 — Queue Full Rejection

**Steps:**

1. Navigate to `http://localhost:3001/store/s-005` (City Hall Office).
2. This store has `maxQueueSize: 5` and already has 5 people in queue.
3. Click **Join Queue** — should get a 400 error: "Queue is full".
4. The UI should show a clear error message, not a crash.

**What to verify:**
- Error message is user-friendly
- Join button state after rejection

---

### 2.6 — Queue Paused Rejection

**Steps:**

1. Navigate to `http://localhost:3001/store/s-004` (Central Station Hub).
2. This store's queue is PAUSED.
3. Click **Join Queue** — should get a 400 error: "Queue is currently paused".
4. The UI should indicate the queue is paused, ideally before the user even tries to join.

**What to verify:**
- Paused state communicated clearly
- Error message if user tries to join anyway

---

### 2.7 — Inactive Store

**Steps:**

1. Navigate to `http://localhost:3001/store/s-003` (Mall Kiosk).
2. This store has `isActive: false`.
3. The UI should indicate the store is not currently active.

**What to verify:**
- Clear messaging that the store is inactive
- Join button disabled or hidden

---

### 2.8 — Non-Existent Store

**Steps:**

1. Navigate to `http://localhost:3001/store/does-not-exist`.
2. Should get a 404 response from the mock.
3. The UI should show a "Store not found" error page, not a crash.

**What to verify:**
- Graceful error handling
- No blank/broken page

---

## Part 3: Cross-App Scenarios

### 3.1 — Admin Pauses Queue, Client Cannot Join

1. In admin-web, navigate to the queue page for **Downtown Branch**.
2. Click **Pause**. Queue state changes to PAUSED.
3. In client-web, navigate to `/store/s-001`.
4. Try to join — should be rejected with "Queue is currently paused".

> Note: In mock mode, both apps share in-memory state only within their own process. This cross-app scenario requires the admin-web mock to persist the pause state (it does via `QUEUE_STATES`), but the client-web has its own separate mock data. To test this cross-component flow end-to-end, use the real backend.

---

### 3.2 — Language Switching

1. In admin-web, switch language from English to Vietnamese using the locale switcher.
2. Verify all UI text, labels, button text, toast messages, and error messages are in Vietnamese.
3. Do the same in client-web.
4. Perform a full flow (join queue, get called, cancel) in Vietnamese mode.

**What to verify:**
- No untranslated strings (English text appearing in Vietnamese mode)
- Date/time formatting is consistent regardless of language
- Error messages from the mock API are shown alongside translated UI labels

---

## Troubleshooting

### Mock not intercepting requests
- Check browser console for the `[MOCK] API mocking active` message.
- If missing, verify `<MockInit />` is present in the locale layout file.
- Clear browser cache and hard-refresh.

### State resets on page refresh
- Admin-web mock state (except auth) resets on full page refresh because the data is in-memory JavaScript.
- Auth state persists via `sessionStorage` (admin-web) or `localStorage` (client-web tickets).

### SSE connection errors
- Some browsers limit concurrent connections. If SSE fails, check for connection limits.
- Mock SSE sends only heartbeat comments — no actual queue events are dispatched.
