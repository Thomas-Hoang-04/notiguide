# Plan: Migrate JWT Auth from localStorage to HttpOnly Cookies

## Context

JWTs are currently stored in `localStorage`, which is vulnerable to XSS attacks — any injected script can steal the token. For an admin dashboard, this is a significant risk. The fix is to move JWT storage to `HttpOnly` + `Secure` + `SameSite=Strict` cookies set by the backend, making the token inaccessible to JavaScript entirely.

## Approach

### Backend Changes

**1. Add cookie config properties** — `AppProperties.kt`
- Add `cookie` section with `name` (default `"auth_token"`), `secure` (default `false` for dev, `true` for prod), `sameSite`, `domain`, `path` fields
- Bind prod values via env vars in `application-prod.yaml`

**2. AuthController — Set cookie on login** — `AuthController.kt`
- Change login to return `ResponseEntity<LoginResponse>` with a `Set-Cookie` response header
- Use `ResponseCookie.from(...)` with `httpOnly(true)`, `secure(...)`, `sameSite("Strict")`, `path("/api")`, `maxAge(jwtProperties.expirySeconds)`
- Still return the `AdminDto` in the body (frontend needs admin data), but **remove the token from the response body**

**3. Add logout endpoint** — `AuthController.kt`
- `POST /api/auth/logout` — returns the same cookie with `maxAge(0)` to clear it
- Add `/api/auth/logout` to the public permit list in `SecurityConfig` (or keep it authenticated — either works since clearing the cookie is harmless)

**4. JWTAuthFilter — Read token from cookie OR header** — `JWTAuthFilter.kt`
- Modify `extractToken()` to first check `Authorization: Bearer` header (backwards compat / API clients), then fall back to reading the `auth_token` cookie
- This is a small change to the `extractToken()` private function

**5. CORS config update** — `CorsConfig.kt`
- Already has `allowCredentials = true` — good
- No changes needed (cookies are sent automatically with `credentials: include`)

**6. SecurityConfig** — `SecurityConfig.kt`
- Add `/api/auth/logout` to `permitAll()` paths

### Frontend Changes

**7. API client — Add `credentials: "include"`** — `web/src/lib/api.ts`
- Add `credentials: "include"` to every `fetch()` call so the browser sends cookies cross-origin
- Remove the `Authorization` header injection (`getToken()` / Bearer logic)
- Remove the `getToken()` and `clearAuth()` functions (token no longer in localStorage)
- Keep the 401 auto-redirect logic but call the store's logout instead of `clearAuth()`

**8. Auth store — Remove token from state** — `web/src/store/auth.ts`
- Remove `token` from state (it's no longer accessible to JS)
- `isAuthenticated` is now derived from `admin !== null` (not `token !== null`)
- `login()` action: receives `LoginResponse` (which now only has `admin`), stores admin in state + localStorage
- `logout()` action: calls `POST /api/auth/logout` to clear cookie, clears localStorage admin, redirects
- `hydrate()`: skip the JWT expiry check (can't read the cookie), just call `/api/admins/me` — if 200, authenticated; if 401, not authenticated

**9. Auth feature API** — `web/src/features/auth/api.ts`
- Add `logout()` function that calls `POST /api/auth/logout`

**10. Login page** — `web/src/app/[locale]/(auth)/login/page.tsx`
- No changes needed (already calls `login()` API and `setAuth(response)`)

**11. LoginResponse type** — `web/src/types/admin.ts`
- Remove `token` field from `LoginResponse` interface

**12. Auth guard** — `web/src/components/layout/auth-guard.tsx`
- No changes needed (already depends on `isAuthenticated` and `isHydrated`)

### Files to Modify

| File | Change |
|------|--------|
| `backend/.../config/AppProperties.kt` | Add cookie properties |
| `backend/.../config/JWTProperties.kt` | No change (expirySeconds already available) |
| `backend/.../controller/AuthController.kt` | Set cookie on login, add logout endpoint |
| `backend/.../dto/LoginResponse.kt` | Remove `token` field |
| `backend/.../jwt/JWTAuthFilter.kt` | Read token from cookie fallback |
| `backend/.../security/SecurityConfig.kt` | Add logout to permitAll |
| `backend/.../resources/application.yaml` | Add cookie config defaults |
| `backend/.../resources/application-prod.yaml` | Add cookie prod config |
| `web/src/lib/api.ts` | Add `credentials: "include"`, remove Bearer logic |
| `web/src/store/auth.ts` | Remove token state, derive auth from admin |
| `web/src/features/auth/api.ts` | Add logout API call |
| `web/src/types/admin.ts` | Remove token from LoginResponse |

### What Stays the Same
- JWT issuance logic (`JWTManager`) — unchanged
- JWT verification logic — unchanged
- `JWTToPrincipal` — unchanged
- Rate limiting — unchanged
- All other controllers — unchanged

## Verification

1. Start backend with `docker compose up -d && ./gradlew bootRun`
2. Start frontend with `cd web && yarn dev`
3. Test login: cookie should appear in browser DevTools > Application > Cookies
4. Test authenticated requests: API calls should work without Authorization header
5. Test logout: cookie should be cleared
6. Test expired session: 401 should redirect to login
7. Verify `document.cookie` does NOT show the auth token (HttpOnly)
