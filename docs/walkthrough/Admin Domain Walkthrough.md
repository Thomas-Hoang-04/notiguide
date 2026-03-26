# Phase 3: Admin/SuperAdmin Management Domain — Walkthrough

## What Was Implemented

22 Kotlin files + 2 config files implementing the full admin domain:

### Shared/Core Layer (11 files)
| File | Action | Purpose |
|---|---|---|
| [Validatable.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/shared/validation/Validatable.kt) | NEW | Entity validation interface |
| [AdminPrincipal.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/shared/principal/AdminPrincipal.kt) | NEW | `UserDetails` impl with `storeId` + `isVerified` |
| [AdminPrincipalAuthToken.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/shared/principal/AdminPrincipalAuthToken.kt) | NEW | Auth token wrapper |
| [JwtProperties.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/core/config/JwtProperties.kt) | NEW | Configurable JWT expiry (Finding #7) |
| [HttpException.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/core/exception/HttpException.kt) | NEW | Base + 4 concrete exception classes |
| [ExceptionHandler.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/core/exception/ExceptionHandler.kt) | MODIFY | 6 handlers added |
| [JWTToPrincipal.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/core/jwt/JWTToPrincipal.kt) | MODIFY | Fixed claim deserialization (Finding #8) |
| [JWTAuthFilter.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/core/jwt/JWTAuthFilter.kt) | MODIFY | Rewritten: [WebFilter](file:///home/thomas/Coding/Java%20%28Kotlin%29/notiguide/src/main/kotlin/com/thomas/notiguide/core/security/SecurityConfig.kt#30-45) + `contextWrite()` |
| [JWTManager.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/core/jwt/JWTManager.kt) | MODIFY | Injected [JwtProperties](file:///home/thomas/Coding/Java%20%28Kotlin%29/notiguide/src/main/kotlin/com/thomas/notiguide/core/config/JwtProperties.kt#5-9) |
| [SecurityConfig.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/core/security/SecurityConfig.kt) | MODIFY | Full security chain configured |
| [R2DBCConfig.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/core/database/R2DBCConfig.kt) | MODIFY | `EnumCodec` for `admin_role` |

### Admin Domain (11 files)
| File | Action | Purpose |
|---|---|---|
| [AdminRole.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/admin/entity/AdminRole.kt) | NEW | Enum matching Postgres type |
| [Admin.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/admin/entity/Admin.kt) | NEW | R2DBC entity with init validation |
| [AdminRepository.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/admin/repository/AdminRepository.kt) | NEW | `CoroutineCrudRepository` + derived queries |
| [AdminDto.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/admin/dto/AdminDto.kt) | NEW | Response DTO (no password) |
| [LoginResponse.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/admin/dto/LoginResponse.kt) | NEW | Token + profile |
| [LoginRequest.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/admin/request/LoginRequest.kt) | NEW | Login input |
| [CreateAdminRequest.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/admin/request/CreateAdminRequest.kt) | NEW | Admin creation input |
| [UpdatePasswordRequest.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/admin/request/UpdatePasswordRequest.kt) | NEW | Password change input |
| [AdminAuthService.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/admin/service/AdminAuthService.kt) | NEW | `ReactiveUserDetailsService` impl |
| [AdminService.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/admin/service/AdminService.kt) | NEW | Business logic + transactions |
| [AuthController.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/admin/controller/AuthController.kt) | NEW | `POST /api/auth/login` |
| [AdminController.kt](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/kotlin/com/thomas/notiguide/domain/admin/controller/AdminController.kt) | NEW | 6 admin endpoints |

### Config
| File | Change |
|---|---|
| [application.yaml](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/resources/application.yaml) | Added `jwt.expiry-seconds: 86400` |
| [application-dev.yaml](file:///home/thomas/Coding/Java%20(Kotlin)/notiguide/src/main/resources/application-dev.yaml) | Added `rate-limit.enabled: false` |

## Audit Fixes During Implementation

| Issue | Severity | Fix |
|---|---|---|
| [JWTAuthFilter](file:///home/thomas/Coding/Java%20%28Kotlin%29/ESP-Doorbell/src/main/kotlin/com/thomas/espdoorbell/doorbell/core/jwt/JWTAuthFilter.kt#14-47) used `CoWebFilter` with `block()` — deadlocks on Netty | Critical | Rewritten to standard [WebFilter](file:///home/thomas/Coding/Java%20%28Kotlin%29/notiguide/src/main/kotlin/com/thomas/notiguide/core/security/SecurityConfig.kt#30-45) with `contextWrite()` for proper Reactor context propagation |
| `AdminService.listAdminsByStore` called `findAll()` — full table scan | Medium | Added [findByStoreId()](file:///home/thomas/Coding/Java%20%28Kotlin%29/notiguide/src/main/kotlin/com/thomas/notiguide/domain/admin/repository/AdminRepository.kt#10-11) derived query to [AdminRepository](file:///home/thomas/Coding/Java%20%28Kotlin%29/notiguide/src/main/kotlin/com/thomas/notiguide/domain/admin/repository/AdminRepository.kt#7-12) |

## Security Checklist

- ✅ Login uses constant-time error message for both bad username and bad password (prevents enumeration)
- ✅ Unverified admins rejected before JWT issuance
- ✅ `AdminPrincipal.isEnabled()` maps to `isVerified` — Spring Security's [AuthenticationManager](file:///home/thomas/Coding/Java%20%28Kotlin%29/notiguide/src/main/kotlin/com/thomas/notiguide/core/security/SecurityConfig.kt#23-29) also rejects disabled accounts
- ✅ Password update requires old password verification
- ✅ Self-deletion prevented
- ✅ SuperAdmin-only operations guarded by [requireSuperAdmin()](file:///home/thomas/Coding/Java%20%28Kotlin%29/notiguide/src/main/kotlin/com/thomas/notiguide/domain/admin/controller/AdminController.kt#92-96)
- ✅ Password updates restricted to self via `principal.id != id` check
- ✅ Public paths: `/api/auth/**`, `/api/queue/public/**`, `/actuator/health`, `/actuator/info`
- ✅ All other paths require authentication
- ✅ CSRF disabled (JWT-only API, no cookies)
- ✅ `@Transactional` on all mutation methods
- ✅ JWTAuthFilter silently continues on bad tokens (`onErrorResume`) — doesn't crash the chain
