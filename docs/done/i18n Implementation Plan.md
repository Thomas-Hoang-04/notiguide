# i18n Implementation Plan — Bilingual Admin Dashboard (EN / VI)

## Library Choice: next-intl

**Selected**: [next-intl](https://next-intl.dev) v4.x
**Alternatives considered**: react-i18next, Lingui, next-i18next (Pages Router only — disqualified)

### Why next-intl

| Criterion | next-intl | react-i18next | Lingui |
|---|---|---|---|
| App Router / RSC | First-class — separate server (`getTranslations`) and client (`useTranslations`) APIs | Manual wiring, dual init files, per-request instances | Supported via SWC plugin + `cache()`, less seamless |
| Next.js 16 + React 19 | Explicit peer deps (`next ^16`, `react ^19`) | No Next.js peer dep — works but self-assembled | React 19 supported, SWC plugin version-sensitive |
| TypeScript | Best-in-class — `AppConfig` augmentation gives key autocomplete + param validation | Requires manual resource type augmentation | Good for ICU format, weaker key autocomplete |
| Setup complexity | 5 files | 6+ files + custom middleware + dual component patterns | Moderate-high (SWC plugin, extraction CLI, dual providers) |
| Bundle impact | ESM-only, server translations = zero client JS | ~60kB (core + react binding), harder to tree-shake | Excellent (~2kB core), but compile step adds build complexity |
| Community | 1.68M weekly downloads, actively maintained | 8M downloads (ecosystem-wide, not Next.js-specific) | 368K downloads, v6 in development |

**Decision**: next-intl is the only library purpose-built for the Next.js App Router with explicit Next.js 16 support. It has the simplest setup, best TypeScript integration, and zero client-side overhead for server-rendered translations. The admin dashboard is client-heavy (SPA-style), but next-intl handles `"use client"` components seamlessly via `useTranslations()`.

---

## Architecture Overview

```text
web/
├── src/
│   ├── i18n/
│   │   ├── request.ts          # Server-side locale resolution
│   │   └── routing.ts          # Locale routing config (pathnames, default locale)
│   ├── messages/
│   │   ├── en.json             # English translations (source of truth)
│   │   └── vi.json             # Vietnamese translations
│   ├── app/
│   │   └── [locale]/           # All existing routes move under dynamic [locale] segment
│   │       ├── layout.tsx      # Root layout with NextIntlClientProvider
│   │       ├── (auth)/
│   │       │   └── login/
│   │       ├── dashboard/
│   │       │   ├── layout.tsx
│   │       │   ├── queue/
│   │       │   ├── stores/
│   │       │   ├── admins/
│   │       │   └── settings/
│   │       │       ├── account/
│   │       │       └── password/
│   │       └── ...
│   ├── middleware.ts            # next-intl middleware for locale detection + redirect
│   └── ...
├── next.config.ts              # Wrap with createNextIntlPlugin
└── ...
```

### URL Structure

| Current | With i18n |
|---|---|
| `/dashboard/queue` | `/en/dashboard/queue` or `/vi/dashboard/queue` |
| `/login` | `/en/login` or `/vi/login` |

- **Default locale**: `en` (English)
- **Supported locales**: `en`, `vi`
- **Locale detection**: Browser `Accept-Language` header → cookie → default fallback
- **Locale prefix**: Always shown in URL for both locales (explicit, unambiguous)

---

## Phases

### Phase 1: Install & Configure next-intl

1. **Install the dependency**:
   ```bash
   cd web && yarn add next-intl
   ```

2. **Create locale routing config** (`src/i18n/routing.ts`):
   ```ts
   import { defineRouting } from "next-intl/routing";

   export const routing = defineRouting({
     locales: ["en", "vi"],
     defaultLocale: "en",
   });
   ```

3. **Create server-side request config** (`src/i18n/request.ts`):
   ```ts
   import { getRequestConfig } from "next-intl/server";
   import { routing } from "./routing";

   export default getRequestConfig(async ({ requestLocale }) => {
     let locale = await requestLocale;
     if (!locale || !routing.locales.includes(locale as any)) {
       locale = routing.defaultLocale;
     }

     return {
       locale,
       messages: (await import(`../messages/${locale}.json`)).default,
     };
   });
   ```

4. **Create middleware** (`src/middleware.ts`):
   ```ts
   import createMiddleware from "next-intl/middleware";
   import { routing } from "./i18n/routing";

   export default createMiddleware(routing);

   export const config = {
     matcher: ["/((?!api|_next|.*\\..*).*)"],
   };
   ```

5. **Wrap Next.js config** (`next.config.ts`):
   ```ts
   import createNextIntlPlugin from "next-intl/plugin";

   const withNextIntl = createNextIntlPlugin();

   const nextConfig = { /* existing config */ };
   export default withNextIntl(nextConfig);
   ```

6. **Create initial message files**:
   - `src/messages/en.json` — English translations (all current hardcoded strings)
   - `src/messages/vi.json` — Vietnamese translations (same keys, translated values)

### Phase 2: Restructure App Router for Locale Segment

1. **Move all routes under `[locale]` dynamic segment**:
   - `src/app/page.tsx` → `src/app/[locale]/page.tsx`
   - `src/app/layout.tsx` → `src/app/[locale]/layout.tsx`
   - `src/app/(auth)/` → `src/app/[locale]/(auth)/`
   - `src/app/dashboard/` → `src/app/[locale]/dashboard/`
   - `src/app/globals.css` stays at `src/app/globals.css` (imported by root layout)

2. **Update root layout** (`src/app/[locale]/layout.tsx`):
   - Accept `locale` param from the dynamic segment
   - Wrap children with `NextIntlClientProvider`
   - Set `<html lang={locale}>` dynamically

3. **Update all `redirect()` calls** to include locale prefix:
   - `redirect("/dashboard")` → `redirect("/${locale}/dashboard")`
   - Use `useRouter` from `next-intl/routing` for client-side navigation (provides locale-aware `push`/`replace`)

4. **Update auth guard and navigation links** in sidebar, topbar, and auth-guard to use locale-aware routing via `next-intl/navigation`:
   ```ts
   import { createNavigation } from "next-intl/navigation";
   import { routing } from "@/i18n/routing";

   export const { Link, redirect, usePathname, useRouter } = createNavigation(routing);
   ```

### Phase 3: Extract Hardcoded Strings

Replace all hardcoded English strings in components with translation function calls. The translation keys are organized by feature namespace.

**Message JSON structure** (top-level namespaces):

```json
{
  "meta": { ... },
  "common": { ... },
  "auth": { ... },
  "navigation": { ... },
  "theme": { ... },
  "settings": { ... },
  "stores": { ... },
  "admins": { ... },
  "queue": { ... },
  "validation": { ... },
  "errors": { ... }
}
```

**In components**, use the `useTranslations` hook with a namespace:

```tsx
// Client component example
"use client";
import { useTranslations } from "next-intl";

function LoginPage() {
  const t = useTranslations("auth");
  return <h1>{t("signInTitle")}</h1>;
}
```

**String extraction order** (by page/feature):
1. Root layout + metadata (`meta` namespace)
2. Navigation — sidebar, topbar (`navigation` namespace)
3. Theme toggle (`theme` namespace)
4. Login page (`auth` namespace)
5. Settings — account + password (`settings` namespace)
6. Store management — header, table, form, delete (`stores` namespace)
7. Admin management — toolbar, table, dialogs (`admins` namespace)
8. Queue operations — controls, stats, serving, lookup, cleanup (`queue` namespace)
9. Shared validation messages (`validation` namespace)
10. Shared error messages (`errors` namespace)

**Parameterized messages** use ICU MessageFormat syntax:
```json
{
  "stores": {
    "deleteConfirmation": "Are you sure you want to delete {storeName}? This action cannot be undone.",
    "deletedToast": "Store \"{storeName}\" deleted"
  },
  "queue": {
    "servedToast": "Ticket #{number} served",
    "pagination": "Page {current} of {total}"
  }
}
```

### Phase 4: Add Language Switcher Component

1. **Create `LanguageSwitcher` component** in `src/components/layout/`:
   - Dropdown or toggle button showing current locale (e.g., "EN" / "VI" or flag icons)
   - On switch: navigate to the same path but with the other locale prefix
   - Use `useRouter` and `usePathname` from `next-intl/navigation` for locale-aware switching:
     ```ts
     const router = useRouter();
     const pathname = usePathname();
     router.replace(pathname, { locale: newLocale });
     ```

2. **Placement**: Add to the topbar, next to the theme toggle

3. **Persist locale preference**: next-intl middleware stores the locale in a `NEXT_LOCALE` cookie automatically — this survives browser restarts

### Phase 5: Translate Remaining Edge Cases

1. **Toast messages**: All `toast.success()` and `toast.error()` calls must use translation keys
2. **Form validation messages**: Zod/manual validation error strings → translation keys
3. **Error response handling**: Map backend error messages to translated display strings (keep backend messages in English; translate on the frontend based on error code/pattern matching)
4. **Metadata**: Page titles and descriptions via `getTranslations` in `generateMetadata`
5. **Accessibility labels**: `sr-only` text, ARIA labels, tooltip content
6. **Date/time formatting**: Use `useFormatter` from next-intl for locale-aware date formatting instead of raw `toLocaleString()`
7. **Constants file**: Move display strings out of `src/lib/constants.ts` into message JSON (keep programmatic constants like API routes, regex patterns, and numeric limits in constants)

---

## Message File Organization

Each namespace maps to a feature area. Keys use camelCase. Nested objects group related strings.

```json
{
  "meta": {
    "title": "NotiGuide Admin",
    "description": "Queue management and notification admin dashboard"
  },
  "common": {
    "cancel": "Cancel",
    "delete": "Delete",
    "save": "Save",
    "create": "Create",
    "retry": "Retry",
    "loading": "Loading...",
    "none": "None",
    "unknown": "Unknown",
    "pagination": "Page {current} of {total}"
  },
  "navigation": { ... },
  "auth": { ... },
  "theme": { ... },
  "settings": { ... },
  "stores": { ... },
  "admins": { ... },
  "queue": { ... },
  "validation": { ... },
  "errors": { ... }
}
```

Full translation keys and EN↔VI mappings are documented in the separate **i18n Translation Table Walkthrough**.

---

## TypeScript Type Safety

Augment `next-intl` types so that translation keys are checked at compile time:

```ts
// src/types/next-intl.d.ts
import en from "../messages/en.json";

type Messages = typeof en;

declare module "next-intl" {
  interface AppConfig {
    Messages: Messages;
  }
}
```

This gives autocomplete on `t("key")` calls and compile errors for typos or missing keys.

---

## Migration Checklist

- [ ] Install `next-intl`
- [ ] Create `src/i18n/routing.ts` and `src/i18n/request.ts`
- [ ] Create `src/middleware.ts` with locale matcher
- [ ] Wrap `next.config.ts` with `createNextIntlPlugin`
- [ ] Move all routes under `src/app/[locale]/`
- [ ] Update root layout with `NextIntlClientProvider` and dynamic `lang`
- [ ] Create navigation utilities via `createNavigation(routing)`
- [ ] Update all `Link`, `redirect`, `useRouter`, `usePathname` imports to use next-intl versions
- [ ] Create `src/messages/en.json` with all extracted strings
- [ ] Create `src/messages/vi.json` with Vietnamese translations
- [ ] Replace hardcoded strings in all pages and components with `useTranslations()` / `getTranslations()`
- [ ] Add type declaration for compile-time key checking
- [ ] Add `LanguageSwitcher` component to topbar
- [ ] Convert date/time formatting to `useFormatter`
- [ ] Update `generateMetadata` functions for translated page titles
- [ ] Verify all toast messages, validation errors, and ARIA labels are translated
- [ ] Test both locales end-to-end (login → queue → settings)

---

## Verification Plan

1. **Phase 1**: Dev server starts without errors. Visiting `/` redirects to `/en/dashboard` (or `/en/login` if unauthenticated). Visiting `/vi/login` renders Vietnamese text.
2. **Phase 2**: All existing routes work under `/en/...`. Navigation between pages preserves locale. Auth guard redirects include locale prefix.
3. **Phase 3**: Every visible string on every page comes from the message JSON (no hardcoded English). Switching to Vietnamese shows all strings in Vietnamese.
4. **Phase 4**: Language switcher toggles between EN/VI. Locale persists across page refreshes (cookie). Switching locale on the queue page preserves the selected store context.
5. **Phase 5**: Toast messages, form validation errors, date formatting, and metadata all respect the active locale.
