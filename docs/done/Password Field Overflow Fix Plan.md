# Password Field Overflow Fix — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix the password Show/Hide toggle button overlap in the USB provisioning dialog by replacing the absolute-positioned overlay pattern with a flex-sibling layout where the input text can never reach the toggle button.

**Architecture:** Two password fields (WiFi and MQTT) in `usb-provision-dialog.tsx` switch from `relative` + `absolute` button positioning to a flex wrapper that visually mimics the `Input` component. The wrapper carries border/focus/error styling; the `<input>` is borderless and takes `flex-1 min-w-0`. CSS classes are extracted to `device.css` to avoid lengthy inline Tailwind.

**Tech Stack:** Next.js 16, React 19, TypeScript 5, Tailwind CSS 4, shadcn/ui

**Spec:** `docs/spec/Password Field Overflow Fix Spec.md`

---

### Task 1: Add Password Field CSS Classes to device.css

**Files:**
- Modify: `web/src/styles/device.css`

- [ ] **Step 1: Add the wrapper and inner input classes**

Append to `web/src/styles/device.css`:

```css
.password-field-wrapper {
  @apply flex h-8 w-full items-center rounded-lg border border-input bg-transparent transition-colors focus-within:border-ring focus-within:ring-3 focus-within:ring-ring/50 dark:bg-input/30;
}

.password-field-wrapper[aria-invalid="true"] {
  @apply border-destructive ring-3 ring-destructive/20 dark:border-destructive/50 dark:ring-destructive/40;
}

.password-field-input {
  @apply h-full flex-1 min-w-0 bg-transparent px-2.5 py-1 text-base outline-none placeholder:text-muted-foreground md:text-sm;
}
```

Key design decisions:
- The wrapper replicates the exact styling from the `Input` component (`web/src/components/ui/input.tsx` line 12): same `h-8`, `rounded-lg`, `border border-input`, `bg-transparent`, focus ring, and dark mode background.
- `focus-within:` on the wrapper ensures the border/ring activates when the child `<input>` receives focus — verified as a standard Tailwind CSS v4 pseudo-class variant.
- `aria-invalid="true"` on the wrapper uses the same destructive border/ring values as the `Input` component's `aria-invalid:` variants.
- The inner input uses `min-w-0` to prevent flex overflow — without this, flex children default to `min-width: auto` and can push the sibling button off-screen.
- `outline-none` removes the native focus ring from the inner input (the wrapper handles focus indication).

---

### Task 2: Convert WiFi Password Field to Flex Layout

**Files:**
- Modify: `web/src/features/device/usb-provision-dialog.tsx:473-501`

- [ ] **Step 1: Add device.css import**

Add at the top of the file, after all named component imports (after line 37, the `StoreSelectField` import):

```tsx
import "@/styles/device.css";
```

- [ ] **Step 2: Replace the WiFi password field block**

Replace lines 473–501 (the `<div className="space-y-2">` containing the WiFi password Label, relative wrapper, Input, and absolute button):

Current code:
```tsx
                <div className="space-y-2">
                  <Label htmlFor="usb-wifi-pwd">{tUsb("wifi_password")}</Label>
                  <div className="relative">
                    <Input
                      id="usb-wifi-pwd"
                      type={showWifiPwd ? "text" : "password"}
                      value={wifiPwd}
                      onChange={(e) => setWifiPwd(e.target.value)}
                      maxLength={64}
                      className="pr-10"
                    />
                    <button
                      type="button"
                      onClick={() => setShowWifiPwd((v) => !v)}
                      className="absolute top-1/2 right-3 -translate-y-1/2 text-muted-foreground hover:text-foreground"
                      aria-label={
                        showWifiPwd
                          ? tUsb("hidePassword")
                          : tUsb("showPassword")
                      }
                    >
                      {showWifiPwd ? (
                        <EyeOff aria-hidden="true" className="size-4" />
                      ) : (
                        <Eye aria-hidden="true" className="size-4" />
                      )}
                    </button>
                  </div>
                </div>
```

New code:
```tsx
                <div className="space-y-2">
                  <Label htmlFor="usb-wifi-pwd">{tUsb("wifi_password")}</Label>
                  <div className="password-field-wrapper">
                    <input
                      id="usb-wifi-pwd"
                      type={showWifiPwd ? "text" : "password"}
                      value={wifiPwd}
                      onChange={(e) => setWifiPwd(e.target.value)}
                      maxLength={64}
                      className="password-field-input"
                    />
                    <button
                      type="button"
                      onClick={() => setShowWifiPwd((v) => !v)}
                      className="shrink-0 px-2.5 text-muted-foreground hover:text-foreground"
                      aria-label={
                        showWifiPwd
                          ? tUsb("hidePassword")
                          : tUsb("showPassword")
                      }
                    >
                      {showWifiPwd ? (
                        <EyeOff aria-hidden="true" className="size-4" />
                      ) : (
                        <Eye aria-hidden="true" className="size-4" />
                      )}
                    </button>
                  </div>
                </div>
```

Changes from the original:
- `<div className="relative">` → `<div className="password-field-wrapper">` (flex container with input styling)
- `<Input ... className="pr-10" />` → `<input ... className="password-field-input" />` (lowercase `input` — no longer using the shadcn `Input` component for these fields, since the wrapper carries the border/focus styling)
- Button: `absolute top-1/2 right-3 -translate-y-1/2` → `shrink-0 px-2.5` (flex sibling with horizontal padding, `shrink-0` prevents the button from shrinking)

---

### Task 3: Convert MQTT Password Field to Flex Layout

**Files:**
- Modify: `web/src/features/device/usb-provision-dialog.tsx:533-565`

- [ ] **Step 1: Replace the MQTT password field block**

Replace lines 533–565 (the `<div className="space-y-2">` containing the MQTT password Label, relative wrapper, Input, absolute button, and error display):

Current code:
```tsx
                <div className="space-y-2">
                  <Label htmlFor="usb-mqtt-pwd">{tUsb("mqtt_password")}</Label>
                  <div className="relative">
                    <Input
                      id="usb-mqtt-pwd"
                      type={showMqttPwd ? "text" : "password"}
                      value={mqttPwd}
                      onChange={(e) => setMqttPwd(e.target.value)}
                      maxLength={127}
                      aria-invalid={!!errors.mqttPwd}
                      className="pr-10"
                    />
                    <button
                      type="button"
                      onClick={() => setShowMqttPwd((v) => !v)}
                      className="absolute top-1/2 right-3 -translate-y-1/2 text-muted-foreground hover:text-foreground"
                      aria-label={
                        showMqttPwd
                          ? tUsb("hidePassword")
                          : tUsb("showPassword")
                      }
                    >
                      {showMqttPwd ? (
                        <EyeOff aria-hidden="true" className="size-4" />
                      ) : (
                        <Eye aria-hidden="true" className="size-4" />
                      )}
                    </button>
                  </div>
                  {errors.mqttPwd && (
                    <InlineError message={errors.mqttPwd} className="mt-1" />
                  )}
                </div>
```

New code:
```tsx
                <div className="space-y-2">
                  <Label htmlFor="usb-mqtt-pwd">{tUsb("mqtt_password")}</Label>
                  <div
                    className="password-field-wrapper"
                    aria-invalid={!!errors.mqttPwd || undefined}
                  >
                    <input
                      id="usb-mqtt-pwd"
                      type={showMqttPwd ? "text" : "password"}
                      value={mqttPwd}
                      onChange={(e) => setMqttPwd(e.target.value)}
                      maxLength={127}
                      className="password-field-input"
                    />
                    <button
                      type="button"
                      onClick={() => setShowMqttPwd((v) => !v)}
                      className="shrink-0 px-2.5 text-muted-foreground hover:text-foreground"
                      aria-label={
                        showMqttPwd
                          ? tUsb("hidePassword")
                          : tUsb("showPassword")
                      }
                    >
                      {showMqttPwd ? (
                        <EyeOff aria-hidden="true" className="size-4" />
                      ) : (
                        <Eye aria-hidden="true" className="size-4" />
                      )}
                    </button>
                  </div>
                  {errors.mqttPwd && (
                    <InlineError message={errors.mqttPwd} className="mt-1" />
                  )}
                </div>
```

Key difference from WiFi field:
- The wrapper gets `aria-invalid={!!errors.mqttPwd || undefined}` — this sets `aria-invalid="true"` when there's a validation error, triggering the destructive border/ring styles defined in `device.css`. The `|| undefined` ensures the attribute is removed entirely when there's no error (React omits attributes set to `undefined`).
- The `aria-invalid` moved from the `<Input>` to the wrapper `<div>`, since the wrapper now carries the border.
- The `InlineError` below the wrapper remains unchanged.

---

### Task 4: Verify

**Files:**
- None (verification only)

- [ ] **Step 1: Run lint**

```bash
cd web && yarn lint
```

Expected: No errors.

- [ ] **Step 2: Run build**

```bash
cd web && yarn build
```

Expected: Build succeeds with no type errors. Note: switching from shadcn `<Input>` to native `<input>` removes the `data-slot="input"` attribute and `InputPrimitive` wrapper — this is fine because there are no global selectors targeting `[data-slot="input"]` in the provisioning dialog context.

- [ ] **Step 3: Visual verification in browser**

```bash
cd web && yarn dev
```

Open the provisioning dialog and verify:

1. **Overflow fix:** Type a 64-character WiFi password, toggle to visible (eye icon) — text must stop before the eye button, never overlapping
2. **Overflow fix:** Type a 127-character MQTT password, toggle to visible — same check
3. **Focus ring:** Click into either password field — the wrapper div should show the teal focus ring (`border-ring` + `ring-ring/50`), matching the other `Input` fields in the form
4. **Error state:** Submit with an invalid MQTT password — the MQTT wrapper should show destructive red border/ring, matching the error styling on other fields
5. **Dark mode:** Toggle dark mode — wrapper background should show `bg-input/30`, matching other inputs
6. **Eye toggle:** Click Eye → EyeOff and back, confirm icon and aria-label switch correctly
7. **No regression:** WiFi SSID, MQTT URI, MQTT Username fields (still using `<Input>`) should be unaffected

---

## Verification Checklist

- [ ] `cd web && yarn lint` passes
- [ ] `cd web && yarn build` passes
- [ ] Long WiFi password does not overlap the eye button
- [ ] Long MQTT password does not overlap the eye button
- [ ] Focus ring appears on wrapper when input is focused
- [ ] MQTT error state shows destructive border/ring on wrapper
- [ ] Dark mode styling matches other input fields
- [ ] Eye/EyeOff toggle and aria-labels work correctly

## File Change Summary

| File | Action | Task |
|------|--------|------|
| `web/src/styles/device.css` | Add password field CSS classes | 1 |
| `web/src/features/device/usb-provision-dialog.tsx` | Add CSS import, convert WiFi password field | 2 |
| `web/src/features/device/usb-provision-dialog.tsx` | Convert MQTT password field | 3 |
