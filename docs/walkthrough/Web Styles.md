# Web Styles Guide

This guide provides a comprehensive overview of the styles, fonts, and colors used in the Notiguide web application (both admin dashboard and client queue management system). It is intended to be used as a reference to ensure consistency across the application.

## Component Library
- **shadcn/ui** is the sole UI component library for all Notiguide web frontends (admin dashboard and client queue app). No other UI framework (Material UI, Chakra, Ant Design, Headless UI, etc.) should be used.
- shadcn/ui components are customized to use the project's CSS variable color tokens (see Semantic Mappings below).
- Glass-effect customizations applied to shadcn components (Dialog, Sheet, AlertDialog, Card) should be preserved if components are regenerated via the shadcn CLI.

## Styles
- Modern glass-based styles for cards, panels and components, similar to that of Apple Liquid Glass (Can grab some inspiration from Samsung OneUI 8.5 design and Google Material 3)

## Fonts
- Be Vietnam Pro
- Inter (Dashboard legends)

## Colors

All colors are defined as CSS custom properties in `web/src/app/globals.css` and mapped to Tailwind via `@theme inline`.

### Light Theme (`:root`)

| Token                    | Value                        | Usage                                                |
|--------------------------|------------------------------|------------------------------------------------------|
| `--background`           | #F8FAFC                      | Main app canvas                                      |
| `--foreground`           | #0C1324                      | Standard reading text and headings                   |
| `--surface`              | #FFFFFF                      | Containers, table backgrounds, queue cards           |
| `--primary`              | #0D9488                      | Headers, sidebars, primary navigation                |
| `--primary-foreground`   | #FFFFFF                      | Text on primary-colored surfaces                     |
| `--primary-hover`        | #0F766E                      | Darker teal for hover states on primary elements     |
| `--action`               | #F97316                      | Call Next button, active queue numbers               |
| `--action-hover`         | #C2410C                      | Darker burnt orange for clicking action buttons      |
| `--action-foreground`    | #FFFFFF                      | Text on action-colored surfaces                      |
| `--muted`                | #F1F5F9                      | Muted backgrounds (same as disabled surface)         |
| `--muted-foreground`     | #64748B                      | Timestamps, secondary descriptions, table headers    |
| `--border`               | #E2E8F0                      | Lines separating table rows or app sections          |
| `--input`                | #E2E8F0                      | Input field borders (matches border)                 |
| `--ring`                 | #0D9488                      | Focus ring color (teal, matches primary)             |
| `--success`              | #10B981                      | Green for "Successfully joined queue"                |
| `--success-foreground`   | #FFFFFF                      | Text on success-colored surfaces                     |
| `--warning`              | #D97706                      | Amber for "Queue paused" or "Delayed"                |
| `--warning-foreground`   | #422006                      | Dark brown text on warning-colored surfaces          |
| `--destructive`          | #E11D48                      | Red for "Cancel ticket" or system errors             |
| `--destructive-foreground` | #FFFFFF                    | Text on destructive-colored surfaces                 |
| `--disabled-surface`     | #F1F5F9                      | Greyed-out buttons or inactive inputs                |
| `--disabled-text`        | #94A3B8                      | Text on disabled elements                            |
| `--chart-accent`         | #4F46E5                      | Deep indigo for admin dashboard graphs               |

#### shadcn Semantic Mappings (Light)

| Token                    | Value   | Usage                              |
|--------------------------|---------|------------------------------------|
| `--card`                 | #FFFFFF | Card component backgrounds         |
| `--card-foreground`      | #0C1324 | Card text                          |
| `--popover`              | #FFFFFF | Popover/dropdown backgrounds       |
| `--popover-foreground`   | #0C1324 | Popover text                       |
| `--secondary`            | #F1F5F9 | Secondary button backgrounds       |
| `--secondary-foreground` | #0C1324 | Secondary button text              |
| `--accent`               | #F1F5F9 | Accent highlights (hover states)   |
| `--accent-foreground`    | #0C1324 | Text on accent surfaces            |

#### Sidebar (Light)

| Token                           | Value                        | Usage                           |
|---------------------------------|------------------------------|---------------------------------|
| `--sidebar`                     | #0B7F74                      | Sidebar background (dark teal)  |
| `--sidebar-foreground`          | #FFFFFF                      | Sidebar text                    |
| `--sidebar-primary`             | #FFFFFF                      | Active nav item text            |
| `--sidebar-primary-foreground`  | #0D9488                      | Active nav item icon            |
| `--sidebar-accent`              | rgba(255, 255, 255, 0.15)    | Nav item hover background       |
| `--sidebar-accent-foreground`   | #FFFFFF                      | Nav item hover text             |
| `--sidebar-border`              | rgba(255, 255, 255, 0.2)     | Sidebar divider lines           |
| `--sidebar-ring`                | #0D9488                      | Sidebar focus ring              |

---

### Dark Theme (`.dark`)

| Token                    | Value                        | Usage                                                       |
|--------------------------|------------------------------|-------------------------------------------------------------|
| `--background`           | #0F172A                      | Main app canvas                                             |
| `--foreground`           | #F8FAFC                      | High-contrast white for easy reading                        |
| `--surface`              | #28354A                      | Elevated containers and client queue cards                  |
| `--primary`              | #2DD4BE                      | Bright teal for navigation and active states                |
| `--primary-foreground`   | #0F172A                      | Text on primary-colored surfaces                            |
| `--primary-hover`        | #5EEAD4                      | Lighter, glowing teal for hover states                      |
| `--action`               | #FB923C                      | Glowing orange for active queue numbers and primary alerts  |
| `--action-hover`         | #FDBA74                      | Softer peach-orange for clicking action buttons             |
| `--action-foreground`    | #0F172A                      | Text on action-colored surfaces                             |
| `--muted`                | #1E293B                      | Muted backgrounds (same as disabled surface)                |
| `--muted-foreground`     | #94A3B8                      | Slate grey for timestamps and secondary info                |
| `--border`               | #3E5270                      | Slightly lighter than surface for clean separations         |
| `--input`                | #3E5270                      | Input field borders (matches border)                        |
| `--ring`                 | #2DD4BE                      | Focus ring color (bright teal, matches primary)             |
| `--success`              | #34D399                      | Bright mint green for positive confirmations                |
| `--success-foreground`   | #0F172A                      | Text on success-colored surfaces                            |
| `--warning`              | #FDE047                      | Bright yellow for alerts and pauses                         |
| `--warning-foreground`   | #422006                      | Dark brown text on warning-colored surfaces                 |
| `--destructive`          | #FB7185                      | Soft red that doesn't bleed against dark backgrounds        |
| `--destructive-foreground` | #0F172A                    | Text on destructive-colored surfaces                        |
| `--disabled-surface`     | #1E293B                      | Dark, receded grey for inactive buttons                     |
| `--disabled-text`        | #64748B                      | Darkened text for inactive elements                         |
| `--chart-accent`         | #818CF8                      | Soft indigo/blue for admin dashboard graphs                 |

#### shadcn Semantic Mappings (Dark)

| Token                    | Value   | Usage                              |
|--------------------------|---------|------------------------------------|
| `--card`                 | #28354A | Card component backgrounds         |
| `--card-foreground`      | #F8FAFC | Card text                          |
| `--popover`              | #28354A | Popover/dropdown backgrounds       |
| `--popover-foreground`   | #F8FAFC | Popover text                       |
| `--secondary`            | #1E293B | Secondary button backgrounds       |
| `--secondary-foreground` | #F8FAFC | Secondary button text              |
| `--accent`               | #1E293B | Accent highlights (hover states)   |
| `--accent-foreground`    | #F8FAFC | Text on accent surfaces            |

#### Sidebar (Dark)

| Token                           | Value                       | Usage                           |
|---------------------------------|-----------------------------|---------------------------------|
| `--sidebar`                     | #0F172A                     | Sidebar background (matches bg) |
| `--sidebar-foreground`          | #F8FAFC                     | Sidebar text                    |
| `--sidebar-primary`             | #2DD4BE                     | Active nav item (bright teal)   |
| `--sidebar-primary-foreground`  | #0F172A                     | Active nav item icon            |
| `--sidebar-accent`              | rgba(255, 255, 255, 0.1)    | Nav item hover background       |
| `--sidebar-accent-foreground`   | #F8FAFC                     | Nav item hover text             |
| `--sidebar-border`              | #3E5270                     | Sidebar divider lines           |
| `--sidebar-ring`                | #2DD4BE                     | Sidebar focus ring              |

---

### Border Radius

| Token          | Value                          |
|----------------|--------------------------------|
| `--radius`     | 0.625rem (base)                |
| `--radius-sm`  | calc(var(--radius) * 0.6)      |
| `--radius-md`  | calc(var(--radius) * 0.8)      |
| `--radius-lg`  | var(--radius)                  |
| `--radius-xl`  | calc(var(--radius) * 1.4)      |
| `--radius-2xl` | calc(var(--radius) * 1.8)      |
| `--radius-3xl` | calc(var(--radius) * 2.2)      |
| `--radius-4xl` | calc(var(--radius) * 2.6)      |

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

| Principle | Guideline |
|-----------|-----------|
| Background opacity | 10-40% for glass surfaces; 50-70% only for primary content cards |
| Blur radius | 8-16px for cards/panels; 2-4px for subtle nav chrome |
| Saturation | Add `saturate(150-180%)` to keep blurred backgrounds vibrant |
| Borders | 1px semi-transparent white borders (0.2-0.4 alpha) to define glass edges |
| Shadows | Combine outer depth shadow + inset glow for refraction illusion |
| Contrast | Maintain WCAG AA contrast ratios — test text over blurred backgrounds |
| Performance | `backdrop-filter` has ~95% browser support; use `-webkit-` prefix for Safari |
| Layering | Avoid glass-on-glass; glass works best over solid or gradient backgrounds |
| Dark mode | Lower opacity (0.3-0.5), use inset glows, and tint shadows darker |

