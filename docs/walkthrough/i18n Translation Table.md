# i18n Translation Table — English ↔ Vietnamese

This document maps every translatable string in the NotiGuide admin dashboard from English to Vietnamese. It serves as the source of truth for building `src/messages/en.json` and `src/messages/vi.json`.

Strings are organized by namespace (matching the message JSON structure from the i18n Implementation Plan). Parameterized values are shown in `{braces}`.

---

## `meta` — Page Metadata


| Key           | English                                           | Vietnamese                                     |
| ------------- | ------------------------------------------------- | ---------------------------------------------- |
| `title`       | NotiGuide Admin                                   | NotiGuide Admin (no translation)               |
| `description` | Queue management and notification admin dashboard | Trang điều khiển quản lý hàng đợi và thông báo |


---

## `common` — Shared UI Strings


| Key          | English                   | Vietnamese                |
| ------------ | ------------------------- | ------------------------- |
| `cancel`     | Cancel                    | Hủy                       |
| `delete`     | Delete                    | Xóa                       |
| `save`       | Save                      | Lưu                       |
| `create`     | Create                    | Tạo                       |
| `retry`      | Retry                     | Thử lại                   |
| `loading`    | Loading...                | Đang tải...               |
| `none`       | None                      | Không có                  |
| `unknown`    | Unknown                   | Không xác định            |
| `pagination` | Page {current} of {total} | Trang {current} / {total} |
| `appName`    | NotiGuide                 | NotiGuide                 |


---

## `navigation` — Sidebar & Layout


| Key              | English     | Vietnamese       |
| ---------------- | ----------- | ---------------- |
| `overview`       | Overview    | Tổng quan        |
| `queue`          | Queue       | Hàng đợi         |
| `stores`         | Stores      | Cửa hàng         |
| `admins`         | Admins      | Quản trị viên    |
| `settings`       | Settings    | Cài đặt          |
| `logout`         | Logout      | Đăng xuất        |
| `mobileNavTitle` | Navigation  | Điều hướng       |
| `roleSuperAdmin` | Super Admin | Quản trị cấp cao |
| `roleAdmin`      | Admin       | Quản trị viên    |


---

## `theme` — Theme Toggle


| Key      | English      | Vietnamese       |
| -------- | ------------ | ---------------- |
| `toggle` | Toggle theme | Chuyển giao diện |
| `light`  | Light        | Sáng             |
| `dark`   | Dark         | Tối              |
| `system` | System       | Hệ thống         |


---

## `auth` — Login Page


| Key                   | English                                                               | Vietnamese                                                               |
| --------------------- | --------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| `signInTitle`         | NotiGuide                                                             | NotiGuide                                                                |
| `signInSubtitle`      | Sign in to your admin account                                         | Đăng nhập vào tài khoản quản trị                                         |
| `usernameLabel`       | Username                                                              | Tên đăng nhập                                                            |
| `usernamePlaceholder` | Enter your username                                                   | Nhập tên đăng nhập                                                       |
| `passwordLabel`       | Password                                                              | Mật khẩu                                                                 |
| `passwordPlaceholder` | Enter your password                                                   | Nhập mật khẩu                                                            |
| `signInButton`        | Sign In                                                               | Đăng nhập                                                                |
| `invalidCredentials`  | Invalid username or password                                          | Tên đăng nhập hoặc mật khẩu không đúng                                   |
| `notVerified`         | Your account has not been verified yet. Please contact a Super Admin. | Tài khoản của bạn chưa được xác minh. Vui lòng liên hệ Quản trị cấp cao. |
| `connectionLost`      | Connection lost. Please check your internet.                          | Mất kết nối. Vui lòng kiểm tra kết nối mạng.                             |
| `usernameRequired`    | Username is required                                                  | Tên đăng nhập không được bỏ trống                                        |
| `passwordRequired`    | Password is required                                                  | Mật khẩu không được bỏ trống                                             |


---

## `settings` — Settings Pages

### Tabs & Layout


| Key           | English  | Vietnamese |
| ------------- | -------- | ---------- |
| `title`       | Settings | Cài đặt    |
| `accountTab`  | Account  | Tài khoản  |
| `passwordTab` | Password | Mật khẩu   |


### Account Page


| Key                  | English                                             | Vietnamese                                     |
| -------------------- | --------------------------------------------------- | ---------------------------------------------- |
| `accountTitle`       | Account Information                                 | Thông tin tài khoản                            |
| `accountDescription` | View your profile details and update your username. | Xem thông tin hồ sơ và cập nhật tên đăng nhập. |
| `usernameLabel`      | Username                                            | Tên đăng nhập                                  |
| `roleLabel`          | Role                                                | Vai trò                                        |
| `storeLabel`         | Store                                               | Cửa hàng                                       |
| `statusLabel`        | Status                                              | Trạng thái                                     |
| `createdLabel`       | Created                                             | Ngày tạo                                       |
| `roleSuperAdmin`     | Super Admin                                         | Quản trị cấp cao                               |
| `roleAdmin`          | Admin                                               | Quản trị viên                                  |
| `storeNone`          | None                                                | Không có                                       |
| `statusVerified`     | Verified                                            | Đã xác minh                                    |
| `statusPending`      | Pending                                             | Chờ xác minh                                   |
| `usernameUpdated`    | Username updated                                    | Đã cập nhật tên đăng nhập                      |


### Password Page


| Key                    | English                                                                                                                | Vietnamese                                                                                     |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `passwordTitle`        | Update your password                                                                                                   | Cập nhật mật khẩu                                                                              |
| `passwordDescription`  | Choose a strong password with at least 8 characters, including uppercase, lowercase, a digit, and a special character. | Chọn mật khẩu mạnh với ít nhất 8 ký tự, bao gồm chữ hoa, chữ thường, chữ số và ký tự đặc biệt. |
| `currentPassword`      | Current Password                                                                                                       | Mật khẩu hiện tại                                                                              |
| `newPassword`          | New Password                                                                                                           | Mật khẩu mới                                                                                   |
| `confirmPassword`      | Confirm New Password                                                                                                   | Xác nhận mật khẩu mới                                                                          |
| `updatePasswordButton` | Update Password                                                                                                        | Cập nhật mật khẩu                                                                              |
| `passwordUpdated`      | Password updated successfully                                                                                          | Cập nhật mật khẩu thành công                                                                   |
| `oldPasswordIncorrect` | Old password is incorrect                                                                                              | Mật khẩu hiện tại không đúng                                                                   |


---

## `stores` — Store Management

### Header & Table


| Key                | English                 | Vietnamese            |
| ------------------ | ----------------------- | --------------------- |
| `title`            | Store Management        | Quản lý cửa hàng      |
| `createButton`     | Create Store            | Tạo cửa hàng          |
| `columnName`       | Name                    | Tên                   |
| `columnAddress`    | Address                 | Địa chỉ               |
| `columnStatus`     | Status                  | Trạng thái            |
| `columnCreated`    | Created                 | Ngày tạo              |
| `columnActions`    | Actions                 | Thao tác              |
| `statusActive`     | Active                  | Hoạt động             |
| `statusInactive`   | Inactive                | Ngừng hoạt động       |
| `emptyState`       | No stores yet.          | Chưa có cửa hàng nào. |
| `emptyStateAction` | Create your first store | Tạo cửa hàng đầu tiên |


### Form Dialog (Create/Edit)


| Key                  | English                                          | Vietnamese                                            |
| -------------------- | ------------------------------------------------ | ----------------------------------------------------- |
| `createTitle`        | Create Store                                     | Tạo cửa hàng                                          |
| `editTitle`          | Edit Store                                       | Chỉnh sửa cửa hàng                                    |
| `nameLabel`          | Name                                             | Tên                                                   |
| `namePlaceholder`    | Store name                                       | Tên cửa hàng                                          |
| `addressLabel`       | Address (optional)                               | Địa chỉ (không bắt buộc)                              |
| `addressPlaceholder` | Store address                                    | Địa chỉ cửa hàng                                      |
| `activeLabel`        | Active                                           | Hoạt động                                             |
| `activeHelper`       | Inactive stores cannot accept new queue tickets. | Cửa hàng ngừng hoạt động không thể nhận lượt chờ mới. |
| `createdToast`       | Store created                                    | Đã tạo cửa hàng                                       |
| `updatedToast`       | Store updated                                    | Đã cập nhật cửa hàng                                  |


### Delete Dialog


| Key                    | English                                                                         | Vietnamese                                                                           |
| ---------------------- | ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| `deleteTitle`          | Delete Store                                                                    | Xóa cửa hàng                                                                         |
| `deleteConfirmation`   | Are you sure you want to delete {storeName}? This action cannot be undone.      | Bạn có chắc chắn muốn xóa {storeName}? Hành động này không thể hoàn tác.             |
| `deletedToast`         | Store "{storeName}" deleted                                                     | Đã xóa cửa hàng "{storeName}"                                                        |
| `deleteConflictQueue`  | Store has active queue tickets. Drain or clear the queue before deleting.       | Cửa hàng có hàng đợi đang hoạt động. Hãy xử lý hết trước khi xóa.                    |
| `deleteConflictAdmins` | Store has assigned admins. Remove or reassign admins before deleting the store. | Cửa hàng còn quản trị viên. Hãy bỏ hoặc chuyển quản trị viên trước khi xóa cửa hàng. |


---

## `admins` — Admin Management

### Toolbar


| Key                | English         | Vietnamese              |
| ------------------ | --------------- | ----------------------- |
| `title`            | Admin Directory | Danh sách quản trị viên |
| `filterAllRoles`   | All Roles       | Tất cả vai trò          |
| `filterSuperAdmin` | Super Admin     | Quản trị cấp cao        |
| `filterAdmin`      | Admin           | Quản trị viên           |
| `filterAllStores`  | All Stores      | Tất cả cửa hàng         |
| `filterByStore`    | Filter by store | Lọc theo cửa hàng       |
| `createButton`     | Create Admin    | Tạo quản trị viên       |


### Table


| Key                | English                                   | Vietnamese                                              |
| ------------------ | ----------------------------------------- | ------------------------------------------------------- |
| `columnUsername`   | Username                                  | Tên đăng nhập                                           |
| `columnRole`       | Role                                      | Vai trò                                                 |
| `columnStore`      | Store                                     | Cửa hàng                                                |
| `columnStatus`     | Status                                    | Trạng thái                                              |
| `columnCreated`    | Created                                   | Ngày tạo                                                |
| `columnActions`    | Actions                                   | Thao tác                                                |
| `roleSuperAdmin`   | Super Admin                               | Quản trị cấp cao                                        |
| `roleAdmin`        | Admin                                     | Quản trị viên                                           |
| `statusVerified`   | Verified                                  | Đã xác minh                                             |
| `statusPending`    | Pending                                   | Chờ xác minh                                            |
| `emptyState`       | No admins found.                          | Không tìm thấy quản trị viên.                           |
| `emptyStateAction` | Create an admin                           | Tạo quản trị viên                                       |
| `cannotVerifySelf` | You cannot verify your own account.       | Bạn không thể tự xác minh tài khoản.                    |
| `cannotDeleteSelf` | You cannot delete your own account.       | Bạn không thể tự xóa tài khoản.                         |
| `noStoreAssigned`  | No store assigned. Contact a Super Admin. | Chưa được phân công cửa hàng. Liên hệ Quản trị cấp cao. |


### Create Admin Dialog


| Key                   | English                                                                                                                    | Vietnamese                                                                                                                         |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `createTitle`         | Create Admin                                                                                                               | Tạo quản trị viên                                                                                                                  |
| `usernameLabel`       | Username                                                                                                                   | Tên đăng nhập                                                                                                                      |
| `usernamePlaceholder` | Enter a username (e.g. john_admin)                                                                                         | Nhập tên đăng nhập (vd: john_admin)                                                                                                |
| `passwordLabel`       | Password                                                                                                                   | Mật khẩu                                                                                                                           |
| `passwordPlaceholder` | Enter a strong password                                                                                                    | Nhập mật khẩu mạnh                                                                                                                 |
| `roleLabel`           | Role                                                                                                                       | Vai trò                                                                                                                            |
| `storeLabel`          | Store                                                                                                                      | Cửa hàng                                                                                                                           |
| `storePlaceholder`    | Select a store                                                                                                             | Chọn cửa hàng                                                                                                                      |
| `storeRequired`       | Store is required for Admin role                                                                                           | Cửa hàng là bắt buộc cho vai trò Quản trị viên                                                                                     |
| `noteSuperAdmin`      | Super Admins have access to all stores. This account will need to be verified by another Super Admin before it can log in. | Quản trị cấp cao có quyền truy cập tất cả cửa hàng. Tài khoản này cần được xác minh bởi Quản trị cấp cao khác trước khi đăng nhập. |
| `noteAdmin`           | This admin will need to be manually verified before they can log in.                                                       | Quản trị viên này cần được xác minh thủ công trước khi đăng nhập.                                                                  |
| `createdToast`        | Admin created                                                                                                              | Đã tạo quản trị viên                                                                                                               |
| `verifiedToast`       | {username} verified                                                                                                        | Đã xác minh {username}                                                                                                             |
| `deletedToast`        | {username} deleted                                                                                                         | Đã xóa {username}                                                                                                                  |
| `usernameTaken`       | Username already taken                                                                                                     | Tên đăng nhập đã được sử dụng                                                                                                      |


### Delete Admin Dialog


| Key                  | English                                                                   | Vietnamese                                                              |
| -------------------- | ------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| `deleteTitle`        | Delete Admin                                                              | Xóa quản trị viên                                                       |
| `deleteConfirmation` | Are you sure you want to delete {username}? This action cannot be undone. | Bạn có chắc chắn muốn xóa {username}? Hành động này không thể hoàn tác. |


---

## `queue` — Queue Operations

### Page Header & Store Selector


| Key                    | English                                                         | Vietnamese                                                 |
| ---------------------- | --------------------------------------------------------------- | ---------------------------------------------------------- |
| `titleWithStore`       | Queue — {storeName}                                             | Hàng đợi — {storeName}                                     |
| `title`                | Queue                                                           | Hàng đợi                                                   |
| `selectStorePrompt`    | Select a store to manage the queue.                             | Chọn cửa hàng để quản lý hàng đợi.                         |
| `noStoreAssigned`      | No store assigned. Contact a Super Admin.                       | Chưa được phân công cửa hàng. Liên hệ Quản trị cấp cao.    |
| `selectStore`          | Select a store                                                  | Chọn cửa hàng                                              |
| `storeInactive`        | (Inactive)                                                      | (Ngừng hoạt động)                                          |
| `storeInactiveWarning` | This store is currently inactive. No new tickets can be issued. | Cửa hàng hiện đang ngừng hoạt động. Không thể phát vé mới. |
| `failedToLoadStores`   | Failed to load stores                                           | Không thể tải danh sách cửa hàng                           |


### Queue Controls


| Key                    | English               | Vietnamese               |
| ---------------------- | --------------------- | ------------------------ |
| `counterIdLabel`       | Counter ID (optional) | Mã quầy (không bắt buộc) |
| `counterIdPlaceholder` | e.g. Counter 1        | vd: Quầy 1               |
| `callNextButton`       | Call Next (N)         | Gọi tiếp (N)             |
| `queueEmpty`           | Queue is empty        | Hàng đợi trống           |


### Queue Stats


| Key            | English  | Vietnamese |
| -------------- | -------- | ---------- |
| `inQueue`      | In Queue | Đang chờ   |
| `statFallback` | —        | —          |


### Serving Display


| Key                       | English                                                               | Vietnamese                                            |
| ------------------------- | --------------------------------------------------------------------- | ----------------------------------------------------- |
| `noTicketServing`         | No ticket currently serving                                           | Hiện không có vé đang phục vụ                         |
| `noTicketSubtext`         | Press Call Next to serve the next customer                            | Nhấn Gọi tiếp để phục vụ khách hàng kế tiếp           |
| `issuedLabel`             | Issued                                                                | Đã phát                                               |
| `calledLabel`             | Called                                                                | Đã gọi                                                |
| `serveButton`             | Serve (S)                                                             | Phục vụ (S)                                           |
| `cancelButton`            | Cancel (C)                                                            | Hủy (C)                                               |
| `servedToast`             | Ticket #{number} served                                               | Đã phục vụ vé #{number}                               |
| `cancelledToast`          | Ticket #{number} cancelled                                            | Đã hủy vé #{number}                                   |
| `cancelDialogTitle`       | Cancel Ticket                                                         | Hủy vé                                                |
| `cancelDialogDescription` | Cancel ticket #{number}? The customer will be removed from the queue. | Hủy vé #{number}? Khách hàng sẽ bị xóa khỏi hàng đợi. |
| `keepButton`              | Keep                                                                  | Giữ lại                                               |


### Ticket Lookup


| Key                 | English                                                  | Vietnamese                                                |
| ------------------- | -------------------------------------------------------- | --------------------------------------------------------- |
| `lookupTitle`       | Ticket Lookup                                            | Tra cứu vé                                                |
| `lookupPlaceholder` | Enter ticket ID                                          | Nhập mã vé                                                |
| `ticketNotFound`    | Ticket not found. It may have expired or been completed. | Không tìm thấy vé. Vé có thể đã hết hạn hoặc đã hoàn tất. |
| `statusLabel`       | Status:                                                  | Trạng thái:                                               |
| `positionLabel`     | Position in queue:                                       | Vị trí trong hàng đợi:                                    |


### Cleanup


| Key              | English                                                                                                                    | Vietnamese                                                                                                               |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `cleanupButton`  | Remove Stale Entries                                                                                                       | Xóa mục cũ                                                                                                               |
| `cleanupTooltip` | Removes tickets from the serving list that have expired or no longer exist. Use this if the serving count seems incorrect. | Xóa các vé đã hết hạn hoặc không còn tồn tại khỏi danh sách phục vụ. Sử dụng khi số lượng phục vụ có vẻ không chính xác. |
| `cleanupResult`  | Cleaned {count} stale entries                                                                                              | Đã xóa {count} mục cũ                                                                                                    |
| `cleanupClean`   | No stale entries found — serving list is clean.                                                                            | Không tìm thấy mục cũ — danh sách phục vụ trống.                                                                         |


---

## `validation` — Shared Validation Messages


| Key                       | English                                                                                                | Vietnamese                                                                      |
| ------------------------- | ------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------- |
| `nameRequired`            | Name is required                                                                                       | Tên là bắt buộc                                                                 |
| `nameMax`                 | Max 255 characters                                                                                     | Tối đa 255 ký tự                                                                |
| `addressMax`              | Max 1000 characters                                                                                    | Tối đa 1000 ký tự                                                               |
| `usernameMin`             | Min {min} characters                                                                                   | Tối thiểu {min} ký tự                                                           |
| `usernameMax`             | Max {max} characters                                                                                   | Tối đa {max} ký tự                                                              |
| `usernamePattern`         | Only letters, numbers, and underscores allowed                                                         | Chỉ cho phép chữ cái, chữ số và dấu gạch dưới                                   |
| `passwordMin`             | Min {min} characters                                                                                   | Tối thiểu {min} ký tự                                                           |
| `passwordMax`             | Max {max} characters                                                                                   | Tối đa {max} ký tự                                                              |
| `currentPasswordRequired` | Current password is required                                                                           | Mật khẩu hiện tại là bắt buộc                                                   |
| `newPasswordRequired`     | New password is required                                                                               | Mật khẩu mới là bắt buộc                                                        |
| `confirmPasswordRequired` | Please confirm your new password                                                                       | Vui lòng xác nhận mật khẩu mới                                                  |
| `passwordsDoNotMatch`     | Passwords do not match                                                                                 | Mật khẩu không khớp                                                             |
| `passwordMustDiffer`      | New password must differ from old password                                                             | Mật khẩu mới phải khác mật khẩu cũ                                              |
| `missingRequirement`      | Missing requirement: {requirement}                                                                     | Thiếu yêu cầu: {requirement}                                                    |
| `missingRequirements`     | Missing requirements: {requirements}                                                                   | Thiếu các yêu cầu: {requirements}                                               |
| `passwordMustContain`     | Must contain at least one uppercase letter, one lowercase letter, one digit, and one special character | Phải chứa ít nhất một chữ hoa, một chữ thường, một chữ số và một ký tự đặc biệt |


### Password Requirement Checklist


| Key            | English                                        | Vietnamese                                |
| -------------- | ---------------------------------------------- | ----------------------------------------- |
| `reqLength`    | {min}-{max} characters                         | {min}-{max} ký tự                         |
| `reqUppercase` | At least one uppercase letter                  | Ít nhất một chữ hoa                       |
| `reqLowercase` | At least one lowercase letter                  | Ít nhất một chữ thường                    |
| `reqDigit`     | At least one digit                             | Ít nhất một chữ số                        |
| `reqSpecial`   | At least one special character (e.g. !@#$%^&*) | Ít nhất một ký tự đặc biệt (vd: !@#$%^&*) |


---

## `errors` — Error Messages


| Key               | English                                           | Vietnamese                                      |
| ----------------- | ------------------------------------------------- | ----------------------------------------------- |
| `connectionLost`  | Connection lost. Please check your internet.      | Mất kết nối. Vui lòng kiểm tra kết nối mạng.    |
| `tooManyRequests` | Too many requests. Please wait {seconds} seconds. | Quá nhiều yêu cầu. Vui lòng đợi {seconds} giây. |
| `forbidden`       | You don't have permission to perform this action. | Bạn không có quyền thực hiện hành động này.     |
| `notFound`        | Resource not found.                               | Không tìm thấy tài nguyên.                      |
| `serverError`     | Something went wrong. Please try again.           | Đã xảy ra lỗi. Vui lòng thử lại.                |


---

## Ticket Status Values

These appear in badge components and ticket lookup results.


| Key               | English   | Vietnamese |
| ----------------- | --------- | ---------- |
| `statusWaiting`   | Waiting   | Đang chờ   |
| `statusCalled`    | Called    | Đã gọi     |
| `statusServed`    | Served    | Đã phục vụ |
| `statusCancelled` | Cancelled | Đã hủy     |
| `statusUnknown`   | Unknown   | Không rõ   |


---

## Notes

- **Brand name "NotiGuide"** is kept as-is in both languages — it is a product name, not translatable.
- **Keyboard shortcut hints** (N, S, C) are kept as-is in both languages — they reference physical keys.
- **Parameterized strings** use ICU MessageFormat `{variable}` syntax. next-intl parses these automatically.
- **Pluralization**: Vietnamese does not inflect for plurals, so no ICU plural rules are needed for these strings.
- **Date formatting**: Handled by `useFormatter` from next-intl, which uses the browser's `Intl.DateTimeFormat` with the active locale — no manual date strings needed.
- **Abbreviations**: Vietnamese role names use natural terms ("Quản trị cấp cao" for Super Admin, "Quản trị viên" for Admin) rather than literal translations.

