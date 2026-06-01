# Password Field Overflow Fix — Spec

**Problem:** In the USB provisioning dialog, the Show/Hide password toggle button is absolutely positioned over the `<input>` element. When the password is long and `type="text"` (visible), the input scrolls text horizontally — the text physically scrolls under the button because the button sits in a separate layer above the input's scrollable content area. `pr-10` padding helps but does not prevent the overlap; `text-overflow: ellipsis` does not apply because the input scrolls rather than clips.

**Affected fields:**
- WiFi password (`usb-provision-dialog.tsx`, lines ~473–501)
- MQTT password (`usb-provision-dialog.tsx`, lines ~533–565)

**Root cause:** The `relative` wrapper + `absolute` button overlay pattern places the button in the same visual space as the input's scrollable text. These are independent stacking contexts — the input's scroll viewport extends under the button regardless of `padding-right`.

---

## Solution: Flex-Sibling Layout

Replace the `relative` + `absolute` overlay pattern with a flex wrapper that visually looks like a single input. The button becomes a flex sibling of the `<input>`, structurally outside the text flow.

### Current Structure

```tsx
<div className="relative">
  <Input type={show ? "text" : "password"} className="pr-10" />
  <button className="absolute top-1/2 right-3 -translate-y-1/2 ...">
    <Eye />
  </button>
</div>
```

### New Structure

```tsx
<div className="password-field-wrapper focus-within:border-ring focus-within:ring-3 focus-within:ring-ring/50">
  <input
    type={show ? "text" : "password"}
    className="password-field-input"
    ...
  />
  <button
    type="button"
    onClick={toggle}
    className="px-2.5 text-muted-foreground hover:text-foreground"
  >
    <Eye className="size-4" />
  </button>
</div>
```

The wrapper carries the border, border-radius, focus ring, and background — mimicking the `Input` component's appearance. The `<input>` is borderless and takes `flex-1 min-w-0`. The button is a flex sibling that the text can never reach.

### CSS Extraction

Because the wrapper needs to replicate the full Input component's border/focus/aria-invalid/dark-mode styling, this is best extracted into a small CSS block rather than a long inline `className`. Add the styles to an appropriate CSS file (e.g. the existing provisioning-related styles or a shared `password-field` utility class).

Wrapper styles to replicate from the `Input` component:
- `flex items-center` (layout)
- `h-8 w-full rounded-lg` (sizing)
- `border border-input bg-transparent` (border)
- `transition-colors` (transitions)
- `focus-within:border-ring focus-within:ring-3 focus-within:ring-ring/50` (focus)
- `dark:bg-input/30` (dark mode)
- `aria-invalid:` variants if the MQTT field uses them

Input-within-wrapper styles:
- `flex-1 min-w-0` (prevent flex overflow)
- `h-full bg-transparent px-2.5 py-1` (sizing)
- `text-base md:text-sm` (typography)
- `outline-none border-0 ring-0` (remove native input styling)
- `placeholder:text-muted-foreground`

### Scope

- **Only** the 2 password fields in `usb-provision-dialog.tsx`
- No changes to the shared `Input` component
- No new UI components — just inline structure change + CSS utility classes
- Existing `showWifiPwd`/`showMqttPwd` state, i18n keys, and aria-labels remain unchanged

### MQTT Error State

The MQTT password field has `aria-invalid={!!errors.mqttPwd}`. The wrapper `<div>` needs the `aria-invalid` data attribute to apply the destructive border/ring styles, since the inner `<input>` no longer carries the border.

### Verification

After implementation:
1. Type a 64-character WiFi password, toggle to visible — text must not overlap the eye icon
2. Type a 127-character MQTT password, toggle to visible — same check
3. Focus ring must appear on the wrapper when the input is focused
4. `aria-invalid` destructive styling must appear on the MQTT wrapper when validation fails
5. Eye/EyeOff icon and aria-labels toggle correctly
6. Dark mode appearance matches other input fields
