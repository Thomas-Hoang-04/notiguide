# Glass UI Upgrade Plan

This document captures the glass design research, current implementation audit, proposed gradient system, and prioritized improvement plan for the Notiguide admin web frontend.

---

## Glass Design Reference

Our design direction draws from three modern glass-based design systems. This section documents their principles so we can apply them consistently.

### Apple Liquid Glass (iOS 26 / macOS Tahoe)

Announced at WWDC 2025, Liquid Glass is Apple's flagship material system. Key characteristics:

- **Three-layer composition**: Highlight (light casting/movement), Shadow (depth separation), Illumination (material flexibility)
- **Dynamic translucency**: Glass elements refract and reflect their surroundings, adapting to content and lighting
- **Saturation boost**: `saturate(180%)` applied alongside blur to keep colors vivid through the frosted layer
- **Inset glow**: Inner box-shadows (`inset 0 4px 20px rgba(255, 255, 255, 0.3)`) simulate light refracting through glass edges
- **Pseudo-element shine**: A `::after` layer with low opacity and `brightness(115%)` creates a luminous highlight band
- **Scope**: Navigation layer and system chrome — not meant for every surface. "Avoid glass on glass."
- **Rounded corners**: All glass elements use generous border-radius (typically 1.5-2rem)

### Samsung OneUI 8.5 Glass UI

Launching March 2026 on Galaxy S26. Samsung's approach emphasizes:

- **Layered floating**: Widgets and notifications appear to float on top of wallpaper using transparency + blur
- **Smooth gradients**: Background blur combined with gradient overlays to ensure text/icon sharpness on OLED
- **Adjustable intensity**: Users can tune blur and transparency levels — a signal that glass should not be binary (on/off) but gradual
- **Selective application**: Blur is "intelligently applied to highlight buttons and controls," avoiding overlaps that cause confusion
- **Color saturation tuning**: Carefully adjusted to maintain contrast and readability

### Google Material 3 / Material 4

Material Design uses glass-adjacent concepts through its elevation system:

- **Tonal elevation**: Rather than blur, M3 uses tonal surface color shifts to communicate depth — surfaces closer to the user get lighter tints
- **Material 4 blur**: Emerging M4 guidelines introduce backdrop blur as a first-class elevation treatment
- **Recommended blur ranges**: 5-8px (subtle frosting), 10-15px (balanced glass), 20-30px (dramatic, performance-costly)
- **Opacity sweet spot**: 10-40% background opacity for glass elements
- **Shadow hierarchy**: Elements further from background get higher blur values, creating natural depth stacking

### Glassmorphism Best Practices (Cross-System)


| Principle          | Guideline                                                                    |
| ------------------ | ---------------------------------------------------------------------------- |
| Background opacity | 10-40% for glass surfaces; 50-70% only for primary content cards             |
| Blur radius        | 8-16px for cards/panels; 2-4px for subtle nav chrome                         |
| Saturation         | Add `saturate(150-180%)` to keep blurred backgrounds vibrant                 |
| Borders            | 1px semi-transparent white borders (0.2-0.4 alpha) to define glass edges     |
| Shadows            | Combine outer depth shadow + inset glow for refraction illusion              |
| Contrast           | Maintain WCAG AA contrast ratios — test text over blurred backgrounds        |
| Performance        | `backdrop-filter` has ~95% browser support; use `-webkit-` prefix for Safari |
| Layering           | Avoid glass-on-glass; glass works best over solid or gradient backgrounds    |
| Dark mode          | Lower opacity (0.3-0.5), use inset glows, and tint shadows darker            |


---

## Glass Implementation Audit

### Current State (`web/src/styles/glass.css`)

We define three glass utility classes:


| Class               | Light                                                      | Dark                                                     |
| ------------------- | ---------------------------------------------------------- | -------------------------------------------------------- |
| `.glass-card`       | `rgba(255,255,255, 0.7)` + `blur(12px)` + white 0.3 border | `rgba(40,53,74, 0.7)` + `blur(12px)` + white 0.08 border |
| `.glass-card-hover` | Elevated shadow + `translateY(-1px)` on hover              | Darker shadow variant                                    |
| `.glass-panel`      | `rgba(255,255,255, 0.5)` + `blur(8px)` + white 0.2 border  | `rgba(40,53,74, 0.5)` + white 0.05 border                |


### Where Glass Is Used (9 usages across 8 files)


| File                                                     | Element              | Class Used   |
| -------------------------------------------------------- | -------------------- | ------------ |
| `[locale]/(auth)/login/page.tsx`                         | Login form card      | `glass-card` |
| `[locale]/dashboard/queue/page.tsx`                      | Stats + call-next bar| `glass-card` |
| `features/queue/serving-display.tsx`                     | Empty state card     | `glass-card` |
| `features/queue/serving-display.tsx`                     | Active ticket card   | `glass-card` |
| `features/queue/ticket-lookup.tsx`                       | Lookup card          | `glass-card` |
| `features/admin/admin-directory-table.tsx`               | Admin table wrapper  | `glass-card` |
| `features/store/store-management-table.tsx`              | Store table wrapper  | `glass-card` |
| `[locale]/dashboard/settings/account/page.tsx`           | Account info card    | `glass-card` |
| `[locale]/dashboard/settings/password/page.tsx`          | Password change card | `glass-card` |


### What Is NOT Using Glass


| Area                                                                    | Current Styling                    | Notes                                                           |
| ----------------------------------------------------------------------- | ---------------------------------- | --------------------------------------------------------------- |
| Dashboard layout (`bg-background`)                                      | Flat solid #F8FAFC / #0F172A       | No gradient, no texture — glass cards float on plain white/navy |
| Topbar                                                                  | Solid `bg-surface` + bottom border | No translucency or blur                                         |
| Sidebar                                                                 | Solid `var(--sidebar)`             | Opaque teal, no glass treatment                                 |
| Queue stat cards                                                        | Inherit from parent `.glass-card`  | Individual stat items are plain                                 |
| Modal overlays (dialog, alert-dialog, sheet)                            | `backdrop-blur-xs` (2px)           | Too subtle to register visually                                 |
| All shadcn UI components (buttons, inputs, badges, dropdowns, tooltips) | Solid theme colors                 | Expected — glass on interactive elements would hurt usability   |


### Diagnosis: Why It Looks "Normie"

1. **No background gradient**: The glass cards sit on flat `#F8FAFC` (light) or `#0F172A` (dark). Glass needs a colorful or textured background to show its translucency — without it, `backdrop-filter: blur()` has nothing to refract, so cards look like regular white/dark cards.
2. **No saturation boost**: Our `backdrop-filter` is `blur(12px)` only. Liquid Glass and OneUI both add `saturate(150-180%)` to keep blurred colors vivid.
3. **No inset glow**: We only have outer `box-shadow` for depth. Missing inner glow (`inset` shadow) that makes glass feel lit from within.
4. **No highlight/shine layer**: No `::before` or `::after` pseudo-element creating a light band or specular highlight across the glass surface.
5. `**.glass-panel` is never used**: Defined but zero references in the codebase.
6. `**.glass-card-hover` is never used**: Defined in CSS but zero references in any component.
7. **Opacity too high**: 0.7 in light mode is nearly opaque — closer to a frosted window than glass. Liquid Glass targets 0.15-0.4 for the actual glass layer.
8. **Topbar and sidebar are fully opaque**: Two of the most prominent chrome surfaces have zero glass treatment, contradicting the "Liquid Glass = navigation layer" principle.
9. **Modal overlay blur too weak**: `backdrop-blur-xs` = 2px is invisible. Should be 8-12px to create noticeable depth separation.

---

## Proposed Gradient System

Currently: **zero gradients anywhere in the codebase**. This section defines a gradient palette derived from our existing color tokens for use in backgrounds, accents, and decorative elements.

### Background Gradients (Page Canvas)

These replace the flat `bg-background` to give glass cards something to refract against.


| Token                  | Light                                                                         | Dark                                                                          | Usage                                         |
| ---------------------- | ----------------------------------------------------------------------------- | ----------------------------------------------------------------------------- | --------------------------------------------- |
| `--gradient-page`      | `linear-gradient(135deg, #F8FAFC 0%, #E0F2FE 40%, #F0FDFA 70%, #F8FAFC 100%)` | `linear-gradient(135deg, #0F172A 0%, #0C2D48 40%, #0A2F2A 70%, #0F172A 100%)` | Main dashboard canvas — subtle teal-sky sweep |
| `--gradient-page-warm` | `linear-gradient(160deg, #F8FAFC 0%, #FFF7ED 50%, #F0FDFA 100%)`              | `linear-gradient(160deg, #0F172A 0%, #2A1F0E 50%, #0A2F2A 100%)`              | Queue page — warm orange tint fading to teal  |
| `--gradient-auth`      | `radial-gradient(ellipse at top, #CCFBF1 0%, #E0F2FE 40%, #F8FAFC 80%)`       | `radial-gradient(ellipse at top, #0A2F2A 0%, #0C2D48 40%, #0F172A 80%)`       | Login/auth pages — centered teal glow         |


### Accent Gradients (Decorative Elements)

Follow the same `:root` / `.dark` override pattern used by all other tokens — same property name, different value per theme.

| Token                 | Light (`:root`)                                   | Dark (`.dark`)                                    | Usage                                         |
| --------------------- | ------------------------------------------------- | ------------------------------------------------- | --------------------------------------------- |
| `--gradient-primary`  | `linear-gradient(135deg, #0D9488, #0EA5E9)`       | `linear-gradient(135deg, #2DD4BE, #38BDF8)`       | Primary CTA shimmer, sidebar accent strip     |
| `--gradient-action`   | `linear-gradient(135deg, #F97316, #EAB308)`       | `linear-gradient(135deg, #FB923C, #FDE047)`       | Call Next button glow, active queue highlight |
| `--gradient-success`  | `linear-gradient(135deg, #10B981, #0D9488)`       | `linear-gradient(135deg, #34D399, #2DD4BE)`       | Success state accents (teal-green)            |
| `--gradient-danger`   | `linear-gradient(135deg, #E11D48, #F97316)`       | `linear-gradient(135deg, #FB7185, #FB923C)`       | Destructive state accents (red-orange)        |


### Glass-Enhancing Gradients (Overlay on Glass Surfaces)


| Token                  | Light                                                                        | Dark                                                                          | Usage                                                   |
| ---------------------- | ---------------------------------------------------------------------------- | ----------------------------------------------------------------------------- | ------------------------------------------------------- |
| `--glass-sheen`        | `linear-gradient(135deg, rgba(255,255,255,0.4) 0%, rgba(255,255,255,0) 50%)` | `linear-gradient(135deg, rgba(255,255,255,0.08) 0%, rgba(255,255,255,0) 50%)` | `::before` pseudo on glass cards — top-left light catch |
| `--glass-edge`         | `linear-gradient(180deg, rgba(255,255,255,0.6) 0%, rgba(255,255,255,0) 4px)` | `linear-gradient(180deg, rgba(255,255,255,0.12) 0%, rgba(255,255,255,0) 4px)` | Top edge highlight on glass cards                       |
| `--glass-glow-primary` | `radial-gradient(circle at 30% 20%, rgba(13,148,136,0.08), transparent 60%)` | `radial-gradient(circle at 30% 20%, rgba(45,212,190,0.12), transparent 60%)`  | Subtle teal tint on primary-context glass panels        |
| `--glass-glow-action`  | `radial-gradient(circle at 70% 30%, rgba(249,115,22,0.06), transparent 60%)` | `radial-gradient(circle at 70% 30%, rgba(251,146,60,0.1), transparent 60%)`   | Subtle orange tint on action-context glass panels       |


---

## Improvement Plan

Based on the audit and research above, here is the prioritized list of improvements to bring the UI closer to modern glass design. Each item targets a specific gap identified in the audit.

### P0 — Foundation (enables all other glass improvements)

1. **Define all gradient tokens in `globals.css`**
  - In `:root` and `.dark`, define all gradient custom properties from the Proposed Gradient System above:
    - Page background gradients: `--gradient-page`, `--gradient-page-warm`, `--gradient-auth`
    - Accent gradients: `--gradient-primary`, `--gradient-action`, `--gradient-success`, `--gradient-danger`
    - Glass-enhancing gradients: `--glass-sheen`, `--glass-edge`, `--glass-glow-primary`, `--glass-glow-action`
  - **Note**: CSS custom properties containing gradient values cannot be used with Tailwind `bg-*` utilities (which expect color values). Apply gradients via utility CSS classes (e.g., `.bg-gradient-page { background: var(--gradient-page); }`) defined in `globals.css`
2. **Apply page background gradients**
  - Dashboard layout (`[locale]/dashboard/layout.tsx:11`): replace `bg-background` with the `.bg-gradient-page` utility class
  - Login page (`[locale]/(auth)/login/page.tsx:85`): replace `bg-background` with `.bg-gradient-auth` — the radial teal glow gives the glass card something to refract against
  - Queue page (`[locale]/dashboard/queue/page.tsx`): the queue page itself has no `bg-background` — its background comes from the dashboard layout. To apply the warm `--gradient-page-warm`, add a root wrapper `<div>` on the queue page with `.bg-gradient-page-warm` that overrides the layout gradient via `position: relative` + full size
  - This is the single most impactful change: glass has nothing to refract without a gradient behind it
3. **Upgrade `.glass-card` in `glass.css`**
  - Add `position: relative` (required for the `::before` pseudo-element)
  - Add `overflow: hidden` (clips the sheen pseudo-element within `border-radius` — many usages add `rounded-xl` via Tailwind)
  - Lower opacity: `rgba(255,255,255, 0.7)` → `rgba(255,255,255, 0.45)` (light), `rgba(40,53,74, 0.7)` → `rgba(40,53,74, 0.5)` (dark)
  - Add saturation: `backdrop-filter: blur(12px) saturate(160%)` — **update both prefixed and unprefixed**: `-webkit-backdrop-filter: blur(12px) saturate(160%)`
  - Add inset glow to existing `box-shadow`: `inset 0 1px 0 rgba(255,255,255, 0.4)` (light) / `inset 0 1px 0 rgba(255,255,255, 0.08)` (dark)
  - Add `::before` pseudo-element with `var(--glass-sheen)` gradient, `position: absolute`, `inset: 0`, `border-radius: inherit`, `pointer-events: none`, `z-index: 1`
  - **Specificity note**: 4 of 9 usages apply `glass-card` to shadcn `<Card>` components which have their own `background`, `border`, and `box-shadow`. Verify that `.glass-card` properties override shadcn defaults — add specificity or `!important` if needed
  - **WCAG check**: After lowering opacity from 0.7 to 0.45, validate text contrast over gradient backgrounds meets AA ratio (4.5:1 for normal text)

### P1 — Chrome Surfaces

1. **Topbar glass treatment** (`components/layout/topbar.tsx:24`)
  - Replace `bg-surface` with semi-transparent background: `rgba(255,255,255, 0.6)` (light) / `rgba(40,53,74, 0.6)` (dark)
  - Add `backdrop-filter: blur(16px) saturate(150%)` and `-webkit-backdrop-filter: blur(16px) saturate(150%)`
  - Add bottom border as `rgba(255,255,255, 0.2)` (light) / `rgba(255,255,255, 0.06)` (dark)
  - **Layout note**: The current flex layout (`dashboard/layout.tsx`) has `<main>` scrolling independently via `overflow-y-auto`, so content does not scroll behind the topbar. The `backdrop-filter` is primarily aesthetic (tinting the gradient background behind it). If true content-under-glass blur is desired later, the layout would need restructuring to `position: sticky` — but the semi-transparent tint alone is a meaningful improvement over the current opaque `bg-surface`
2. **Sidebar subtle glass overlay** (`styles/sidebar.css`)
  - Keep the teal `--sidebar` base color but add a subtle glass overlay via `::before` with `rgba(255,255,255, 0.05)` and `blur(4px)` + `-webkit-backdrop-filter`
  - Requires `position: relative` on `.sidebar`, and `::before` with `position: absolute`, `inset: 0`, `pointer-events: none`
  - Add a vertical gradient sheen on the right edge via `::after` (since `::before` is used for the overlay)
  - Light: teal base remains dominant. Dark: sidebar blends with background but gets an inner glow
3. **Increase modal overlay blur**
  - `backdrop-blur-xs` (2px) → `backdrop-blur-lg` (12px) on dialog/alert-dialog/sheet overlays
  - Files: `components/ui/dialog.tsx:33`, `components/ui/sheet.tsx:30`, `components/ui/alert-dialog.tsx:32`
  - **Note**: These are shadcn/ui-managed components. Mark the blur customization with a comment so it is not lost if shadcn components are regenerated
  - This creates proper depth separation when modals open

### P2 — Accents & Polish

1. **Gradient accent on Call Next button** (`[locale]/dashboard/queue/page.tsx`)
  - Apply `var(--gradient-action)` as background on the primary CTA
  - Add a subtle glow shadow: `0 0 20px rgba(249,115,22, 0.3)` to make it "pop"
2. **Glass-enhancing glow on contextual cards**
  - Queue page cards (serving-display, ticket-lookup): apply `--glass-glow-action` via `::after` for a warm orange tint
  - Settings/admin cards: apply `--glass-glow-primary` for a cool teal tint
  - **Note**: `.glass-card` already uses `::before` for sheen (P0.3). Use `::after` for contextual glow — both pseudo-elements are now occupied on glass cards
  - This adds color personality to glass cards based on their context
3. **Active sidebar link glass highlight** (`styles/sidebar.css`)
  - Replace opaque `rgba(255,255,255, 0.15)` on `.sidebar-link-active` with a frosted pill: `backdrop-filter: blur(4px)` + `-webkit-backdrop-filter: blur(4px)` + `rgba(255,255,255, 0.12)` + inset border glow
  - Makes active nav items feel like illuminated glass pills
4. **Upgrade `.glass-panel` and apply to stat cards**
  - Upgrade `.glass-panel` in `glass.css`: same saturation and inset glow treatment as `.glass-card` but at lower intensity. Blur: `blur(8px) saturate(140%)` + `-webkit-backdrop-filter`. Add `position: relative` + `overflow: hidden`
  - Apply `.glass-panel` to queue stat items (`features/queue/queue-stats.tsx`, styled via `styles/queue.css` `.queue-stat-card`) with contextual tint:
    - Waiting: `--glass-glow-primary` (teal). Serving: `--gradient-success` tint (green). Called: `--glass-glow-action` (orange)

### P3 — Refinement

1. **Login page decorative blur circles** (`[locale]/(auth)/login/page.tsx`)
  - Add floating decorative blur orbs (pure CSS `div` with large `border-radius` + colored gradient + `filter: blur(80px)`) positioned behind the login card
  - Orbs should be siblings of the `<Card>` element inside the outer wrapper, with `position: absolute`, `pointer-events: none`, and `z-index` below the card
  - Light: soft teal orb top-left, soft sky-blue orb bottom-right. Dark: deeper teal + navy orbs at lower opacity
  - On small screens where the card fills most of the viewport, orbs may be partially clipped — ensure the wrapper has `overflow: hidden`
  - Creates atmospheric depth behind the glass card (the gradient background from P0.2 is the prerequisite)
2. **Glass card hover state upgrade** (`styles/glass.css`)
  - On hover: increase `backdrop-filter` blur by 2px, brighten inset glow, and add a faint `box-shadow` glow matching the card's contextual color
  - Apply `.glass-card-hover` to all interactive glass cards: admin table (`admin-directory-table.tsx`), store table (`store-management-table.tsx`), ticket lookup (`ticket-lookup.tsx`)
3. **Reduce-motion / accessibility** (`styles/glass.css`)
  - Wrap all transitions and hover transforms in `@media (prefers-reduced-motion: no-preference)`
  - Provide a `prefers-contrast: high` media query that disables transparency and uses solid backgrounds
  - **Note**: `ThemeProvider` in `[locale]/layout.tsx` already sets `disableTransitionOnChange`, so light/dark theme toggles snap instantly — no flash-of-wrong-glass concern

