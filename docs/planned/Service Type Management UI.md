# Service Type Management UI

## Context

The backend fully supports ServiceType CRUD at `GET/POST /api/stores/{storeId}/service-types` and `PUT/DELETE /api/stores/{storeId}/service-types/{id}`. The frontend already has the API layer (`features/store/api.ts`), types (`types/store.ts`), route constants (`lib/constants.ts`), and i18n keys (`en.json`, `vi.json`) — but **no UI components or pages** exist yet.

This plan adds a new **"Service Types"** tab in Settings (separate from the existing Store tab) where non-SuperAdmin admins can manage their store's service types.

---

## What Already Exists (Do Not Recreate)

| Layer | File | Notes |
|-------|------|-------|
| API functions | `web/src/features/store/api.ts` | `listServiceTypes`, `createServiceType`, `updateServiceType`, `deleteServiceType` |
| Types | `web/src/types/store.ts` | `ServiceTypeDto`, `CreateServiceTypeRequest`, `UpdateServiceTypeRequest` |
| Route constants | `web/src/lib/constants.ts` | `API_ROUTES.STORES.SERVICE_TYPES(id)`, `API_ROUTES.STORES.SERVICE_TYPE(storeId, id)` |
| i18n keys | `web/src/messages/en.json`, `vi.json` | Under `"stores"` namespace: `serviceTypes`, `serviceTypeName`, `serviceTypePrefix`, `serviceTypeStatus`, `addServiceType`, `editServiceType`, `deleteServiceType`, `deleteServiceTypeConfirm`, `serviceTypeCreated`, `serviceTypeUpdated`, `serviceTypeDeleted`, `serviceTypeNamePlaceholder`, `serviceTypePrefixPlaceholder`, `serviceTypeActive`, `noServiceTypes` |

---

## Files to Create

### 1. `web/src/app/[locale]/dashboard/settings/service-types/page.tsx`

Settings tab page (container component).

- `"use client"` component
- State: `serviceTypes: ServiceTypeDto[]`, `loading`, `error`, dialog open/close states
- Fetch `listServiceTypes(storeId)` on mount via `useAuthStore().storeId`
- Guard: if `!storeId` return null (SuperAdmins have no store)
- Render: `ServiceTypesTable` + `ServiceTypeFormDialog` + `DeleteServiceTypeDialog`
- Callbacks: `openCreate()`, `openEdit(st)`, `openDelete(st)`, `handleSuccess()` (refetch list)
- Layout: `<div className="mx-auto max-w-2xl">` (matches store settings page width)

### 2. `web/src/features/store/service-types-table.tsx`

Table listing all service types for the store.

- Follow `store-management-table.tsx` pattern exactly
- Columns: **Name**, **Prefix**, **Status** (Badge), **Actions** (edit, delete)
- No pagination needed (stores rarely have >10 service types)
- Skeleton loading: 3 placeholder rows (fewer than stores since list is smaller)
- Empty state: "No service types configured" + "Add service type" link button
- Status badge styling:
  - Active: `border-success/30 bg-success/10 text-success`
  - Inactive: `border-destructive/30 bg-destructive/10 text-destructive`
- Action buttons: edit (Pencil icon), delete (Trash2 icon) — same `variant="ghost" size="icon-sm"` pattern
- Active/inactive toggle: Switch component inline in the status column, calls `updateServiceType` with `{ isActive: !current }` — provides instant toggle without opening edit dialog

Props:
```typescript
type ServiceTypesTableProps = {
  items: ServiceTypeDto[];
  loading: boolean;
  onCreate: () => void;
  onEdit: (st: ServiceTypeDto) => void;
  onDelete: (st: ServiceTypeDto) => void;
  onToggleActive: (st: ServiceTypeDto) => void;
};
```

### 3. `web/src/features/store/service-type-form-dialog.tsx`

Create/Edit form dialog.

- Follow `delete-store-dialog.tsx` simplicity (this is a small form, not the 600-line store form)
- Dialog with two fields:
  - **Name**: text input, required, max 100 chars
  - **Prefix**: text input, required, max 5 chars, uppercase hint
- Edit mode pre-fills from `serviceType` prop
- Client-side validation:
  - Name: required, max 100
  - Prefix: required, max 5
- Error handling:
  - 409 on prefix → show inline "Prefix already in use" error
  - Field-level `err.details` mapping
  - Generic API errors → toast
- On success: toast `serviceTypeCreated` / `serviceTypeUpdated`, close dialog, call `onSuccess`

Props:
```typescript
interface ServiceTypeFormDialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  serviceType?: ServiceTypeDto | null; // null = create mode
  storeId: string;
  onSuccess: () => void;
}
```

### 4. `web/src/features/store/delete-service-type-dialog.tsx`

Delete confirmation dialog.

- Follow `delete-store-dialog.tsx` pattern exactly
- AlertDialog with destructive action
- Conflict handling: 409 → "Cannot delete the last active service type" (inline error box with AlertCircle icon)
- On success: toast `serviceTypeDeleted`, close, call `onSuccess`

Props:
```typescript
interface DeleteServiceTypeDialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  serviceType: ServiceTypeDto | null;
  storeId: string;
  onSuccess: () => void;
}
```

---

## Files to Modify

### 5. `web/src/app/[locale]/dashboard/settings/layout.tsx`

Add the "Service Types" tab to the settings tab navigation.

- Add new tab entry after the existing `storeTab` (both are non-SuperAdmin only):
  ```typescript
  {
    href: "/dashboard/settings/service-types",
    label: "serviceTypesTab" as const,
    icon: ListOrdered, // from lucide-react
  }
  ```
- Place inside the same `!isSuperAdmin` conditional spread so it's hidden for SuperAdmins
- Import `ListOrdered` from `lucide-react`

### 6. `web/src/messages/en.json`

Add missing i18n keys under `"settings"` namespace:
```json
"serviceTypesTab": "Service Types"
```

Add missing i18n keys under `"stores"` namespace:
```json
"serviceTypeInactive": "Inactive",
"deleteServiceTypeConflict": "Cannot delete the last active service type",
"serviceTypePrefixTaken": "This prefix is already in use",
"serviceTypeActions": "Actions"
```

### 7. `web/src/messages/vi.json`

Add corresponding Vietnamese translations:
```json
"serviceTypesTab": "Loại dịch vụ"
```

Under `"stores"`:
```json
"serviceTypeInactive": "Tạm ngưng",
"deleteServiceTypeConflict": "Không thể xóa loại dịch vụ hoạt động cuối cùng",
"serviceTypePrefixTaken": "Tiền tố này đã được sử dụng",
"serviceTypeActions": "Thao tác"
```

### 8. `docs/CHANGELOGS.md`

Document all changes.

---

## Implementation Order

1. Add i18n keys to `en.json` and `vi.json`
2. Create `delete-service-type-dialog.tsx` (simplest component, no form state)
3. Create `service-type-form-dialog.tsx` (create/edit dialog)
4. Create `service-types-table.tsx` (table with inline toggle)
5. Create `settings/service-types/page.tsx` (container wiring everything together)
6. Modify `settings/layout.tsx` (add tab)
7. Update `CHANGELOGS.md`

---

## Validation Rules

| Field | Rule | Error Key |
|-------|------|-----------|
| Name | Required, non-blank | `validation.nameRequired` (existing) |
| Name | Max 100 chars | `validation.nameMax` (existing, parameterized) |
| Prefix | Required, non-blank | `validation.nameRequired` (reuse) |
| Prefix | Max 5 chars | custom inline check |

---

## Error Handling Matrix

| HTTP Code | Scenario | UI Behavior |
|-----------|----------|-------------|
| 409 (create/update) | Duplicate prefix | Inline error on prefix field |
| 409 (delete) | Last active service type | Inline conflict message in AlertDialog |
| 400 | Validation errors with `details` | Map to field-level InlineError |
| 400 | Generic | Toast via `translateCommonApiError` |
| Network error | Fetch/submit failure | Toast via `translateNetworkError` |

---

## Verification

1. Log in as a regular admin (not SuperAdmin) → Settings should show "Service Types" tab
2. Log in as SuperAdmin → "Service Types" tab should NOT appear
3. Navigate to Service Types tab → table loads with existing types (at least "General" / "A")
4. Click "Add service type" → dialog opens, fill name + prefix → creates successfully
5. Try duplicate prefix → 409 → inline error shown
6. Click edit → dialog pre-fills → update name/prefix → saves
7. Toggle active switch → instant toggle with toast
8. Delete a service type → confirmation dialog → deletes
9. Try deleting last active → 409 → conflict message shown
10. Test both English and Vietnamese locales
