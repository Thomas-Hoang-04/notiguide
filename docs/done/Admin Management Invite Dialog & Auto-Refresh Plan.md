# Admin Management Invite Dialog & Auto-Refresh Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** On the admin-management page, move the invite-link panel into a dialog triggered from the toolbar, add a manual whole-page refresh button, and auto-refresh the directory when a pending member is approved.

**Architecture:** Frontend-only. Keep `InviteLinkPanel` as the reusable card (still used by two Settings tabs) and add an `embedded` mode so a new `InviteLinkDialog` can render it chrome-less. Extract the role/store→invite selection into a pure, unit-tested `resolveInviteConfig` module. Lift `inviteOpen` / `refreshing` / `refreshSignal` state into `admins/page.tsx`; the toolbar gets the refresh + invite-trigger buttons, and `JoinRequestsPanel` gains a `refreshSignal` (refetch) and `onApproved` (parent refetch) contract.

**Tech Stack:** Next.js 16 / React 19 / TypeScript, Base UI dialog (`@/components/ui/dialog`), next-intl (type-safe against `en.json`), lucide-react, Biome, vitest (node env).

**Spec:** `docs/spec/Admin Management Invite Dialog & Auto-Refresh Spec.md`

---

## Project conventions for this plan (read first)

- **No git-commit steps.** By project convention the executor commits at their own cadence — there are deliberately no `git commit` steps in the tasks below.
- **Named imports only**, at the top of the file. Biome organizes imports alphabetically by path and named members; keep new imports in sorted position (commands below already reflect this).
- **Verification = lint → type-check → tests.** Run `yarn lint` (Biome), then `npx tsc --noEmit` (type-check), then `yarn test` (vitest). **Do not run `yarn build`** — production build is handled by the separate audit flow.
- **Transient type errors are expected mid-plan.** Tasks 5 and 6 make new props *required* on `AdminDirectoryToolbar` and `JoinRequestsPanel`; `page.tsx` only supplies them in Task 7. So `npx tsc --noEmit` will report missing-prop errors on `page.tsx` until Task 7 completes. Per-task verification for those tasks is `yarn lint` only; the **authoritative full type-check is Task 8**.
- **i18n is bilingual + type-safe.** `next-intl.d.ts` binds `Messages` to `en.json`, so referencing a missing key is a compile error. That is why Task 1 (add the key to **both** `en.json` and `vi.json`) comes first. `messages/parity.test.ts` enforces identical key sets across the two files.

## Pre-flight (GitNexus, per repo CLAUDE.md)

Before editing the existing components, run impact analysis and report the blast radius:

```
gitnexus_impact({target: "InviteLinkPanel", direction: "upstream"})
gitnexus_impact({target: "JoinRequestsPanel", direction: "upstream"})
gitnexus_impact({target: "AdminDirectoryToolbar", direction: "upstream"})
```

Expected: `InviteLinkPanel`'s consumers are exactly `dashboard/admins/page.tsx` and the two Settings tabs (`settings/organization`, `settings/store`); `JoinRequestsPanel` and `AdminDirectoryToolbar` are consumed only by `dashboard/admins/page.tsx`. If anything else shows up, stop and reconcile with the spec before proceeding. After Task 7, run `gitnexus_detect_changes()` to confirm scope.

## File structure

| File | Responsibility | Action |
|------|----------------|--------|
| `src/messages/en.json`, `src/messages/vi.json` | `admins.refresh` aria-label string | Modify |
| `src/features/organization/invite-config.ts` | Pure `resolveInviteConfig(...)` + `InviteConfig` type | Create |
| `src/features/organization/invite-config.test.ts` | Unit tests for the above | Create |
| `src/features/organization/invite-link-panel.tsx` | Add `embedded` mode (drop card chrome + heading) | Modify |
| `src/features/organization/invite-link-dialog.tsx` | Controlled dialog wrapping the embedded panel | Create |
| `src/features/admin/admin-directory-toolbar.tsx` | Refresh icon-button + `Link mời` trigger | Modify |
| `src/features/admin/join-requests-panel.tsx` | `refreshSignal` refetch + `onApproved` callback | Modify |
| `src/app/[locale]/dashboard/admins/page.tsx` | State, memoized config, refresh handler, wiring | Modify |
| `docs/CHANGELOGS.md` | Log the change | Modify |

---

## Task 1: Add the `admins.refresh` i18n key (both locales)

**Files:**
- Modify: `src/messages/en.json:291`
- Modify: `src/messages/vi.json:291`
- Test: `src/messages/parity.test.ts` (existing — run, do not edit)

- [ ] **Step 1: Add the key to `en.json`**

In the `admins` object, immediately after the `"createButton": "Create Admin",` line (line 291 — note this is the **admins** one, distinct from the `stores` namespace's "Create Store" at line 187), add:

```json
    "createButton": "Create Admin",
    "refresh": "Refresh list",
```

- [ ] **Step 2: Add the matching key to `vi.json`**

In the `admins` object, immediately after `"createButton": "Tạo quản trị viên",` (line 291), add:

```json
    "createButton": "Tạo quản trị viên",
    "refresh": "Tải lại danh sách",
```

- [ ] **Step 3: Verify i18n parity passes**

Run: `yarn test parity`
Expected: PASS — `web i18n parity > en and vi share identical key sets` and the tag/`<p>` assertion both green (the new key is a plain string with no rich-text tags on either side).

- [ ] **Step 4: Lint**

Run: `yarn lint`
Expected: clean (run `yarn format` if Biome reports JSON formatting differences, then re-run `yarn lint`).

---

## Task 2: `resolveInviteConfig` pure module (TDD)

**Files:**
- Create: `src/features/organization/invite-config.ts`
- Test: `src/features/organization/invite-config.test.ts`

- [ ] **Step 1: Write the failing test**

Create `src/features/organization/invite-config.test.ts` (mock style mirrors `src/features/admin/api.test.ts`):

```ts
import { beforeEach, describe, expect, it, vi } from "vitest";

vi.mock("@/features/organization/api", () => ({
  getOrgInviteLink: vi.fn(),
  rotateOrgInviteLink: vi.fn(),
  getStoreInviteLink: vi.fn(),
  rotateStoreInviteLink: vi.fn(),
}));

import {
  getOrgInviteLink,
  getStoreInviteLink,
  rotateOrgInviteLink,
  rotateStoreInviteLink,
} from "@/features/organization/api";
import { resolveInviteConfig } from "@/features/organization/invite-config";

describe("resolveInviteConfig", () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it("returns the org invite pair for a super-admin", () => {
    const config = resolveInviteConfig(true, null, undefined);
    expect(config).not.toBeNull();
    expect(config?.fetchLink).toBe(getOrgInviteLink);
    expect(config?.generateLink).toBe(rotateOrgInviteLink);
  });

  it("returns store-scoped closures when a store manager's store has no org", () => {
    const config = resolveInviteConfig(false, "store-1", null);
    expect(config).not.toBeNull();
    config?.fetchLink();
    config?.generateLink();
    expect(getStoreInviteLink).toHaveBeenCalledWith("store-1");
    expect(rotateStoreInviteLink).toHaveBeenCalledWith("store-1");
  });

  it("returns null while the store's org is still loading (undefined)", () => {
    expect(resolveInviteConfig(false, "store-1", undefined)).toBeNull();
  });

  it("returns null when the store belongs to an org", () => {
    expect(resolveInviteConfig(false, "store-1", "org-1")).toBeNull();
  });

  it("returns null for a store manager with no store", () => {
    expect(resolveInviteConfig(false, null, null)).toBeNull();
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `yarn test invite-config`
Expected: FAIL — cannot resolve `@/features/organization/invite-config` (module does not exist yet).

- [ ] **Step 3: Write the module**

Create `src/features/organization/invite-config.ts`:

```ts
import {
  getOrgInviteLink,
  getStoreInviteLink,
  rotateOrgInviteLink,
  rotateStoreInviteLink,
} from "@/features/organization/api";
import type { InviteLinkState } from "@/types/organization";

export interface InviteConfig {
  fetchLink: () => Promise<InviteLinkState>;
  generateLink: () => Promise<InviteLinkState>;
}

// Returns the fetch/generate pair for the applicable invite link, or null when none applies.
export function resolveInviteConfig(
  isSuperAdmin: boolean,
  adminStoreId: string | null,
  storeOrgId: string | null | undefined,
): InviteConfig | null {
  if (isSuperAdmin) {
    return { fetchLink: getOrgInviteLink, generateLink: rotateOrgInviteLink };
  }
  if (adminStoreId && storeOrgId === null) {
    return {
      fetchLink: () => getStoreInviteLink(adminStoreId),
      generateLink: () => rotateStoreInviteLink(adminStoreId),
    };
  }
  return null;
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `yarn test invite-config`
Expected: PASS — all 5 cases green.

- [ ] **Step 5: Lint**

Run: `yarn lint`
Expected: clean.

---

## Task 3: Add `embedded` mode to `InviteLinkPanel`

**Files:**
- Modify: `src/features/organization/invite-link-panel.tsx:27-30, 51-54, 111-116`

This task keeps the default (card) rendering byte-for-byte for the two Settings tabs (they pass no `embedded` prop) and adds a chrome-less mode for the dialog.

- [ ] **Step 1: Add `embedded` to the props interface**

Replace:

```tsx
interface InviteLinkPanelProps {
  fetchLink: () => Promise<InviteLinkState>;
  generateLink: () => Promise<InviteLinkState>;
}
```

with:

```tsx
interface InviteLinkPanelProps {
  fetchLink: () => Promise<InviteLinkState>;
  generateLink: () => Promise<InviteLinkState>;
  embedded?: boolean;
}
```

- [ ] **Step 2: Destructure `embedded` with a default**

Replace:

```tsx
export function InviteLinkPanel({
  fetchLink,
  generateLink,
}: InviteLinkPanelProps) {
```

with:

```tsx
export function InviteLinkPanel({
  fetchLink,
  generateLink,
  embedded = false,
}: InviteLinkPanelProps) {
```

- [ ] **Step 3: Gate the card chrome and heading behind `embedded`**

Replace the start of the returned JSX:

```tsx
  return (
    <div className="rounded-xl border border-border bg-card p-4">
      <h2 className="text-sm font-semibold">{tAdmins("inviteLinkTitle")}</h2>
      <p className="mt-1 mb-3 text-sm text-muted-foreground">
        {tAdmins("inviteLinkDesc")}
      </p>
```

with:

```tsx
  return (
    <div className={embedded ? "" : "rounded-xl border border-border bg-card p-4"}>
      {!embedded && (
        <>
          <h2 className="text-sm font-semibold">
            {tAdmins("inviteLinkTitle")}
          </h2>
          <p className="mt-1 mb-3 text-sm text-muted-foreground">
            {tAdmins("inviteLinkDesc")}
          </p>
        </>
      )}
```

Leave everything else (loading/link/empty branches, expiry, warning box, recent-uses list, the regenerate `AlertDialog`, and the closing `</div>`) unchanged. No `cn` import is needed — the file does not use `cn` today; a bare ternary string is correct.

- [ ] **Step 4: Lint + type-check (no consumer break — `embedded` is optional)**

Run: `yarn lint`
Expected: clean.
Run: `npx tsc --noEmit`
Expected: no new errors (the Settings tabs and the still-present inline usage in `page.tsx` remain valid because `embedded` is optional).

---

## Task 4: Create `InviteLinkDialog`

**Files:**
- Create: `src/features/organization/invite-link-dialog.tsx`

- [ ] **Step 1: Write the component**

Create `src/features/organization/invite-link-dialog.tsx`:

```tsx
"use client";

import { useTranslations } from "next-intl";
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogHeader,
  DialogTitle,
} from "@/components/ui/dialog";
import { InviteLinkPanel } from "@/features/organization/invite-link-panel";
import type { InviteLinkState } from "@/types/organization";

interface InviteLinkDialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  fetchLink: () => Promise<InviteLinkState>;
  generateLink: () => Promise<InviteLinkState>;
}

export function InviteLinkDialog({
  open,
  onOpenChange,
  fetchLink,
  generateLink,
}: InviteLinkDialogProps) {
  const tAdmins = useTranslations("admins");
  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent className="xs:max-w-lg">
        <DialogHeader>
          <DialogTitle>{tAdmins("inviteLinkTitle")}</DialogTitle>
          <DialogDescription>{tAdmins("inviteLinkDesc")}</DialogDescription>
        </DialogHeader>
        <InviteLinkPanel embedded fetchLink={fetchLink} generateLink={generateLink} />
      </DialogContent>
    </Dialog>
  );
}
```

Notes (verified): `DialogContent`/`DialogDescription`/`DialogHeader`/`DialogTitle` are all exported from `@/components/ui/dialog`; `className="xs:max-w-lg"` overrides the default `xs:max-w-sm` because `cn` is `twMerge`-based; Base UI mounts dialog content lazily on open and unmounts on close, so the embedded panel refetches each time the dialog opens.

- [ ] **Step 2: Lint + type-check**

Run: `yarn lint`
Expected: clean.
Run: `npx tsc --noEmit`
Expected: no new errors (the component is self-contained and not yet consumed).

---

## Task 5: Add refresh + invite-trigger buttons to `AdminDirectoryToolbar`

**Files:**
- Modify: `src/features/admin/admin-directory-toolbar.tsx:3, 15-33, 38-49`

- [ ] **Step 1: Add the new lucide-react icon imports**

Replace:

```tsx
import { Plus } from "lucide-react";
```

with:

```tsx
import { Link2, Plus, RefreshCcw } from "lucide-react";
```

- [ ] **Step 2: Add the four new props to the type and destructure**

Replace the props type:

```tsx
type AdminDirectoryToolbarProps = {
  isSuperAdmin: boolean;
  onCreateAdmin: () => void;
  onRoleFilterChange: (value: string | null) => void;
  onStoreFilterChange: (value: string | null) => void;
  roleFilter: string;
  storeFilter: string;
  stores: StoreDto[];
};
```

with:

```tsx
type AdminDirectoryToolbarProps = {
  isSuperAdmin: boolean;
  onCreateAdmin: () => void;
  onOpenInvite: () => void;
  onRefresh: () => void;
  onRoleFilterChange: (value: string | null) => void;
  onStoreFilterChange: (value: string | null) => void;
  refreshing: boolean;
  roleFilter: string;
  showInviteButton: boolean;
  storeFilter: string;
  stores: StoreDto[];
};
```

And replace the destructure:

```tsx
export function AdminDirectoryToolbar({
  isSuperAdmin,
  onCreateAdmin,
  onRoleFilterChange,
  onStoreFilterChange,
  roleFilter,
  storeFilter,
  stores,
}: AdminDirectoryToolbarProps) {
```

with:

```tsx
export function AdminDirectoryToolbar({
  isSuperAdmin,
  onCreateAdmin,
  onOpenInvite,
  onRefresh,
  onRoleFilterChange,
  onStoreFilterChange,
  refreshing,
  roleFilter,
  showInviteButton,
  storeFilter,
  stores,
}: AdminDirectoryToolbarProps) {
```

- [ ] **Step 3: Render the button group in the header row**

Replace:

```tsx
      <div className="flex items-center justify-between gap-3">
        <h1 className="text-xl font-bold l:text-2xl">{tAdmins("title")}</h1>
        {isSuperAdmin && (
          <Button
            onClick={onCreateAdmin}
            className="bg-primary text-primary-foreground hover:bg-primary-hover"
          >
            <Plus className="mr-2 size-4" />
            {tAdmins("createButton")}
          </Button>
        )}
      </div>
```

with:

```tsx
      <div className="flex items-center justify-between gap-3">
        <h1 className="text-xl font-bold l:text-2xl">{tAdmins("title")}</h1>
        <div className="flex items-center gap-2">
          <Button
            variant="outline"
            size="icon"
            disabled={refreshing}
            onClick={onRefresh}
            aria-label={tAdmins("refresh")}
          >
            <RefreshCcw
              aria-hidden="true"
              className={`size-4 ${refreshing ? "animate-spin" : ""}`}
            />
          </Button>
          {showInviteButton && (
            <Button variant="outline" onClick={onOpenInvite}>
              <Link2 className="mr-2 size-4" />
              {tAdmins("inviteLinkTitle")}
            </Button>
          )}
          {isSuperAdmin && (
            <Button
              onClick={onCreateAdmin}
              className="bg-primary text-primary-foreground hover:bg-primary-hover"
            >
              <Plus className="mr-2 size-4" />
              {tAdmins("createButton")}
            </Button>
          )}
        </div>
      </div>
```

- [ ] **Step 4: Lint (full type-check deferred)**

Run: `yarn lint`
Expected: clean.
Note: `npx tsc --noEmit` will now report that `page.tsx`'s `<AdminDirectoryToolbar>` is missing the 4 new required props — **expected**, resolved in Task 7. Do not "fix" by making the props optional.

---

## Task 6: Wire `refreshSignal` + `onApproved` into `JoinRequestsPanel`

**Files:**
- Modify: `src/features/admin/join-requests-panel.tsx:23-31, 49-51, 53-60`

- [ ] **Step 1: Extend the props interface and destructure**

Replace:

```tsx
interface JoinRequestsPanelProps {
  isSuperAdmin: boolean;
  stores: StoreDto[];
}

export function JoinRequestsPanel({
  isSuperAdmin,
  stores,
}: JoinRequestsPanelProps) {
```

with:

```tsx
interface JoinRequestsPanelProps {
  isSuperAdmin: boolean;
  stores: StoreDto[];
  refreshSignal: number;
  onApproved: () => void;
}

export function JoinRequestsPanel({
  isSuperAdmin,
  stores,
  refreshSignal,
  onApproved,
}: JoinRequestsPanelProps) {
```

- [ ] **Step 2: Refetch when the refresh signal changes**

Replace:

```tsx
  useEffect(() => {
    void fetchRequests();
  }, [fetchRequests]);
```

with:

```tsx
  useEffect(() => {
    void fetchRequests();
  }, [fetchRequests, refreshSignal]);
```

This is a single combined effect: `fetchRequests` is a stable `useCallback([])`, so it fires once on mount (signal `0`) and again on each bump. **Do not add an `if (refreshSignal)` guard** — this effect owns the initial load too, so a guard would suppress the mount fetch.

- [ ] **Step 3: Notify the parent after a successful approval**

Replace:

```tsx
      await approveJoinRequest(approveTarget.requestId, role, storeId);
      toast.success(
        tAdmins("requestApprovedToast", { username: approveTarget.username }),
      );
      await fetchRequests();
    } catch (err) {
```

with:

```tsx
      await approveJoinRequest(approveTarget.requestId, role, storeId);
      toast.success(
        tAdmins("requestApprovedToast", { username: approveTarget.username }),
      );
      await fetchRequests();
      onApproved();
    } catch (err) {
```

Leave `doReject` unchanged — rejecting removes a pending request but adds no admin, so the directory is unaffected.

- [ ] **Step 4: Lint (full type-check deferred)**

Run: `yarn lint`
Expected: clean.
Note: `npx tsc --noEmit` will now also report that `page.tsx`'s `<JoinRequestsPanel>` is missing `refreshSignal` / `onApproved` — **expected**, resolved in Task 7.

---

## Task 7: Wire everything in `admins/page.tsx`

**Files:**
- Modify: `src/app/[locale]/dashboard/admins/page.tsx:4, 15-21, 46, 61-66, 156-194, 244-250`

- [ ] **Step 1: Update the React import to include `useMemo`**

Replace:

```tsx
import { useCallback, useEffect, useState } from "react";
```

with:

```tsx
import { useCallback, useEffect, useMemo, useState } from "react";
```

- [ ] **Step 2: Swap the organization imports**

Replace this block (the `@/features/organization/api` named import and the `InviteLinkPanel` import):

```tsx
import {
  getOrgInviteLink,
  getStoreInviteLink,
  rotateOrgInviteLink,
  rotateStoreInviteLink,
} from "@/features/organization/api";
import { InviteLinkPanel } from "@/features/organization/invite-link-panel";
```

with:

```tsx
import { resolveInviteConfig } from "@/features/organization/invite-config";
import { InviteLinkDialog } from "@/features/organization/invite-link-dialog";
```

(The four org-api functions now live behind `resolveInviteConfig`; `page.tsx` no longer references them directly.)

- [ ] **Step 3: Add the new state**

After the `actionLoading` state line:

```tsx
  const [actionLoading, setActionLoading] = useState<string | null>(null);
```

add:

```tsx
  const [inviteOpen, setInviteOpen] = useState(false);
  const [refreshing, setRefreshing] = useState(false);
  const [refreshSignal, setRefreshSignal] = useState(0);
```

- [ ] **Step 4: Derive the memoized invite config**

Immediately after the `storeOrgId` `useEffect` (the one that calls `getStore(adminStoreId)`), add:

```tsx
  const inviteConfig = useMemo(
    () => resolveInviteConfig(isSuperAdmin, adminStoreId, storeOrgId),
    [isSuperAdmin, adminStoreId, storeOrgId],
  );
```

- [ ] **Step 5: Add the refresh handler**

After the `handleVerify` function (and before `handleDelete`, or anywhere among the page's handlers), add:

```tsx
  async function handleRefresh() {
    setRefreshing(true);
    setRefreshSignal((s) => s + 1);
    try {
      await fetchAdmins(page, storeFilter, roleFilter);
    } finally {
      setRefreshing(false);
    }
  }
```

- [ ] **Step 6: Remove the two inline `<InviteLinkPanel>` blocks**

Delete this entire block (it sits at the top of the returned JSX, right after `<div>`):

```tsx
      {isSuperAdmin && (
        <div className="mb-6">
          <InviteLinkPanel
            fetchLink={getOrgInviteLink}
            generateLink={rotateOrgInviteLink}
          />
        </div>
      )}
      {!isSuperAdmin && adminStoreId && storeOrgId === null && (
        <div className="mb-6">
          <InviteLinkPanel
            fetchLink={() => getStoreInviteLink(adminStoreId)}
            generateLink={() => rotateStoreInviteLink(adminStoreId)}
          />
        </div>
      )}
```

(The invite UI now lives in the dialog rendered in Step 9.)

- [ ] **Step 7: Pass the new props to `JoinRequestsPanel`**

Replace:

```tsx
      <JoinRequestsPanel isSuperAdmin={isSuperAdmin} stores={stores} />
```

with:

```tsx
      <JoinRequestsPanel
        isSuperAdmin={isSuperAdmin}
        stores={stores}
        refreshSignal={refreshSignal}
        onApproved={() => void fetchAdmins(page, storeFilter, roleFilter)}
      />
```

- [ ] **Step 8: Pass the new props to `AdminDirectoryToolbar`**

Replace:

```tsx
      <AdminDirectoryToolbar
        isSuperAdmin={isSuperAdmin}
        roleFilter={roleFilter}
        storeFilter={storeFilter}
        stores={stores}
        onCreateAdmin={() => setCreateOpen(true)}
        onRoleFilterChange={(v) => {
          if (v) {
            setRoleFilter(v);
            setPage(0);
          }
        }}
        onStoreFilterChange={(v) => {
          if (v) {
            setStoreFilter(v);
            setPage(0);
          }
        }}
      />
```

with:

```tsx
      <AdminDirectoryToolbar
        isSuperAdmin={isSuperAdmin}
        refreshing={refreshing}
        roleFilter={roleFilter}
        showInviteButton={inviteConfig !== null}
        storeFilter={storeFilter}
        stores={stores}
        onCreateAdmin={() => setCreateOpen(true)}
        onOpenInvite={() => setInviteOpen(true)}
        onRefresh={() => void handleRefresh()}
        onRoleFilterChange={(v) => {
          if (v) {
            setRoleFilter(v);
            setPage(0);
          }
        }}
        onStoreFilterChange={(v) => {
          if (v) {
            setStoreFilter(v);
            setPage(0);
          }
        }}
      />
```

- [ ] **Step 9: Render the invite dialog**

After the `<DeleteAdminDialog ... />` element (the last dialog before the page's closing `</div>`), add:

```tsx
      {inviteConfig && (
        <InviteLinkDialog
          open={inviteOpen}
          onOpenChange={setInviteOpen}
          fetchLink={inviteConfig.fetchLink}
          generateLink={inviteConfig.generateLink}
        />
      )}
```

- [ ] **Step 10: Lint + full type-check (authoritative)**

Run: `yarn lint`
Expected: clean.
Run: `npx tsc --noEmit`
Expected: **no errors** — all required props are now supplied; the removed `InviteLinkPanel`/org-api imports are no longer referenced; `useMemo` is imported.

---

## Task 8: Changelog + final verification

**Files:**
- Modify: `docs/CHANGELOGS.md`

- [ ] **Step 1: Add a changelog entry**

Match the file's existing convention (verified against the top entries): a `## YYYY-MM-DD — title`
heading → prose summary → a "**Deliberately unchanged (recorded per convention):**" prose line → a
"Design docs:" line → a `### Files Changed` section with a `#### Web (`web/`)` Markdown table
(`| File | Action | Summary |`, Action ∈ CREATED/MODIFIED). This is a web-only change, so only the
Web table is needed. Insert the entry directly under the top `# Changelogs` heading, above the most
recent `## 2026-06-13 — Join Request Role Selection …` entry:

```markdown
## 2026-06-13 — Admin management: invite link moves to a dialog, manual refresh, auto-update on approval

On the admin-management page (`dashboard/admins`), the invite-link panel moves behind a `Link mời` button in the directory toolbar (opening a dialog), a manual refresh icon-button now reloads the whole page (admin directory + pending join requests) on demand, and approving a pending join request refetches the directory so the new verified admin appears without a page reload. The invite dialog refetches its state each time it opens, so recent-join activity is always current when viewed.

`InviteLinkPanel` gains an optional `embedded` prop (drops the card chrome + its own heading) so the new `InviteLinkDialog` can reuse it; with the prop omitted, its rendering is unchanged. The role/store → invite selection is extracted into a pure, unit-tested `resolveInviteConfig` module.

**Deliberately unchanged (recorded per convention):** the in-table shield-verify already updates optimistically and is untouched; both Settings tabs that embed `InviteLinkPanel` (`settings/organization`, `settings/store`) pass no `embedded` prop and render exactly as before; no backend/API/schema change.

Design docs: `docs/spec/Admin Management Invite Dialog & Auto-Refresh Spec.md`, `docs/planned/Admin Management Invite Dialog & Auto-Refresh Plan.md`.

### Files Changed

#### Web (`web/`)

| File | Action | Summary |
|------|--------|---------|
| `src/messages/en.json` / `src/messages/vi.json` | MODIFIED | Added mirrored `admins.refresh` aria-label ("Refresh list" / "Tải lại danh sách") |
| `src/features/organization/invite-config.ts` | CREATED | Pure `resolveInviteConfig(isSuperAdmin, adminStoreId, storeOrgId)` + `InviteConfig` type |
| `src/features/organization/invite-config.test.ts` | CREATED | Unit tests for `resolveInviteConfig` (mocks `@/features/organization/api`) |
| `src/features/organization/invite-link-panel.tsx` | MODIFIED | Added optional `embedded` prop — drops card chrome + own title/description; default rendering unchanged (Settings tabs unaffected) |
| `src/features/organization/invite-link-dialog.tsx` | CREATED | Controlled dialog wrapping the embedded panel; refetches on open |
| `src/features/admin/admin-directory-toolbar.tsx` | MODIFIED | Added refresh icon-button (`RefreshCcw`, spins while refreshing) + `Link mời` trigger (`Link2`); new props `refreshing`, `onRefresh`, `showInviteButton`, `onOpenInvite` |
| `src/features/admin/join-requests-panel.tsx` | MODIFIED | New `refreshSignal` prop (refetch on bump) + `onApproved` callback (parent refetches directory after approval) |
| `src/app/[locale]/dashboard/admins/page.tsx` | MODIFIED | Moved invite panel into `InviteLinkDialog` (toolbar-triggered); added `inviteOpen`/`refreshing`/`refreshSignal` state, memoized `inviteConfig`, whole-page `handleRefresh`; removed inline `InviteLinkPanel` blocks |
```

- [ ] **Step 2: Run the full lint**

Run: `yarn lint`
Expected: clean.

- [ ] **Step 3: Run the full type-check**

Run: `npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 4: Run the full test suite**

Run: `yarn test`
Expected: PASS — including the new `invite-config` tests and `messages/parity` (identical key sets across en/vi).

- [ ] **Step 5: Confirm scope with GitNexus**

Run: `gitnexus_detect_changes()`
Expected: only the files listed in this plan changed.

- [ ] **Step 6: Manual smoke check (browser)**

Do **not** run `yarn build` (audit flow handles production build). With `yarn dev`, sign in as a SUPER_ADMIN and verify on the admin-management page:
- The invite panel is gone from the page body; a `Link mời` button sits beside `Tạo quản trị viên`.
- Clicking `Link mời` opens a dialog with the URL, copy, regenerate (confirm dialog still works), expiry, warning, and recent uses; reopening it refetches.
- The refresh icon-button spins during refresh and the table rows stay visible (no skeleton flash) while data reloads; pending join requests also refresh.
- Approving a pending join request makes the new admin appear in the directory **without** a page reload.
- As a store manager whose store has no org, the toolbar shows the refresh button and the `Link mời` button (store invite) but no `Tạo quản trị viên`.

---

## Self-review checklist (for the author before handoff)

- [ ] Every spec section maps to a task: invite dialog (T3, T4, T7), Settings tabs untouched (T3 default + T7 verify), whole-page refresh (T1, T5, T7), auto-update on approval (T6, T7), `resolveInviteConfig` module + tests (T2), i18n (T1), changelog (T8).
- [ ] No placeholders — every code step shows complete code.
- [ ] Type/name consistency: `resolveInviteConfig` / `InviteConfig` (T2) match their use in T7; `InviteLinkDialog` props (T4) match T7 usage; toolbar props (T5) match T7 usage; `JoinRequestsPanel` props (T6) match T7 usage; `admins.refresh` (T1) matches `tAdmins("refresh")` (T5).
