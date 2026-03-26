# i18n Translation Table — Client Queue App (English <-> Vietnamese)

This document maps every translatable string in the NotiGuide client queue app from English to Vietnamese. It serves as the source of truth for building `client-web/src/messages/en.json` and `client-web/src/messages/vi.json`.

Strings are organized by namespace (matching the message JSON structure from the [Client Web Implementation Plan](../planned/Client%20Web%20Implementation%20Plan.md)). Parameterized values are shown in `{braces}`.

---

## `meta` — Page Metadata


| Key           | English                                     | Vietnamese                                 |
| ------------- | ------------------------------------------- | ------------------------------------------ |
| `title`       | NotiGuide                                   | NotiGuide                                  |
| `description` | Join the queue and track your ticket status | Xếp hàng và theo dõi trạng thái vé của bạn |


---

## `common` — Shared UI Strings


| Key       | English              | Vietnamese    |
| --------- | -------------------- | ------------- |
| `loading` | Loading...           | Đang tải...   |
| `retry`   | Try again            | Thử lại       |
| `cancel`  | Cancel               | Hủy           |
| `confirm` | Confirm              | Xác nhận      |
| `close`   | Close                | Đóng          |
| `back`    | Back                 | Quay lại      |
| `error`   | Something went wrong | Đã xảy ra lỗi |
| `appName` | NotiGuide            | NotiGuide     |


---

## `store` — Store Landing Page


| Key                        | English                                           | Vietnamese                                       |
| -------------------------- | ------------------------------------------------- | ------------------------------------------------ |
| `storeInfo`                | Store Information                                 | Thông tin cửa hàng                               |
| `address`                  | Address                                           | Địa chỉ                                          |
| `queueStatus`              | Queue Status                                      | Trạng thái hàng đợi                              |
| `ticketsWaiting`           | {count} tickets waiting                           | {count} lượt đang chờ                            |
| `noOneWaiting`             | No one is in line — you'll be next!               | Hàng chờ đang trống — bạn sẽ là người tiếp theo! |
| `storeNotFound`            | Store not found                                   | Không tìm thấy cửa hàng                          |
| `storeNotFoundDescription` | This store does not exist or has been removed.    | Cửa hàng này không tồn tại hoặc đã bị xóa.       |
| `storeInactive`            | This store is not currently accepting new tickets | Cửa hàng hiện không nhận khách mới               |
| `storeClosed`              | Store Unavailable                                 | Cửa hàng không khả dụng                          |
| `poweredBy`                | Powered by NotiGuide                              | Vận hành bởi NotiGuide                           |


---

## `queue` — Queue Operations & Ticket Management

### Join Queue


| Key                | English                                       | Vietnamese               |
| ------------------ | --------------------------------------------- | ------------------------ |
| `joinQueue`        | Join Queue                                    | Xếp hàng                 |
| `joining`          | Joining...                                    | Đang xếp hàng...         |
| `joinSuccess`      | You've joined the queue!                      | Lấy vé thành công!       |
| `hasActiveTicket`  | You already have an active ticket (#{number}) | Bạn đã có vé (#{number}) |
| `viewActiveTicket` | View your ticket                              | Xem vé của bạn           |
| `joinAnyway`       | Join again                                    | Lấy vé lại               |


### Ticket Display


| Key               | English                      | Vietnamese                  |
| ----------------- | ---------------------------- | --------------------------- |
| `yourTicket`      | Your Ticket                  | Vé Của Bạn                  |
| `ticketNumber`    | Ticket Number                | Số vé                       |
| `position`        | Position in Queue            | Vị trí trong hàng đợi       |
| `positionDisplay` | #{position} in line          | Thứ #{position} trong hàng  |
| `peopleAhead`     | {count} people ahead of you! | Có {count} người trước bạn! |
| `noPeopleAhead`   | You're next!                 | Bạn là người tiếp theo!     |
| `estimatedWait`   | Estimated Wait               | Thời gian chờ dự kiến       |
| `waitUnavailable` | Estimate not available       | Không có ước tính           |
| `status`          | Status                       | Trạng thái                  |
| `issuedAt`        | Issued at                    | Phát vé lúc                 |
| `calledAt`        | Called at                    | Gọi lúc                     |
| `lastUpdated`     | Last updated {time} ago      | Đã cập nhật {time} trước    |


### Cancel Ticket


| Key                    | English                                   | Vietnamese                        |
| ---------------------- | ----------------------------------------- | --------------------------------- |
| `cancelTicket`         | Cancel Ticket                             | Hủy vé                            |
| `cancelConfirmTitle`   | Cancel your ticket?                       | Hủy vé của bạn?                   |
| `cancelConfirmMessage` | You will lose your position in the queue. | Bạn sẽ mất vị trí trong hàng đợi. |
| `keepTicket`           | Keep my ticket                            | Giữ vé                            |
| `yesCancel`            | Yes, cancel                               | Có, hủy                           |
| `cancelSuccess`        | Your ticket has been cancelled            | Vé của bạn đã được hủy            |
| `joinAgain`            | Join Queue Again                          | Lấy vé lại                        |


### Status Transitions


| Key                        | English                                                                                | Vietnamese                                                              |
| -------------------------- | -------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| `calledAlert`              | Your number is being called!                                                           | Lượt của bạn đã đến!                                                    |
| `calledAlertDescription`   | Please proceed to the counter                                                          | Vui lòng đến quầy phục vụ                                               |
| `calledDismiss`            | Got it                                                                                 | Đã hiểu                                                                 |
| `servedMessage`            | You have been served. Thank you!                                                       | Bạn đã được phục vụ. Cảm ơn!                                            |
| `cancelledMessage`         | Your ticket has been cancelled.                                                        | Vé của bạn đã bị hủy.                                                   |
| `ticketExpired`            | This ticket is no longer active                                                        | Vé này không còn hoạt động                                              |
| `ticketExpiredDescription` | Your ticket may have expired or been completed. Please join the queue again if needed. | Vé có thể đã hết hạn hoặc đã được phục vụ. Vui lòng lấy vé lại nếu cần. |


### Misc


| Key             | English        | Vietnamese |
| --------------- | -------------- | ---------- |
| `refreshStatus` | Refresh Status | Làm mới    |


---

## `status` — Ticket Status Values

These appear in badge components and status displays.


| Key         | English   | Vietnamese |
| ----------- | --------- | ---------- |
| `waiting`   | Waiting   | Đang chờ   |
| `called`    | Called    | Đã gọi     |
| `served`    | Served    | Đã phục vụ |
| `cancelled` | Cancelled | Đã hủy     |
| `unknown`   | Unknown   | Không rõ   |


---

## `errors` — Error Messages


| Key              | English                                           | Vietnamese                                      |
| ---------------- | ------------------------------------------------- | ----------------------------------------------- |
| `networkError`   | Connection lost. Please check your internet.      | Mất kết nối. Vui lòng kiểm tra kết nối mạng.    |
| `rateLimited`    | Too many requests. Please wait {seconds} seconds. | Quá nhiều yêu cầu. Vui lòng đợi {seconds} giây. |
| `serverError`    | Server error. Please try again later.             | Lỗi máy chủ. Vui lòng thử lại sau.              |
| `storeNotFound`  | Store not found                                   | Không tìm thấy cửa hàng                         |
| `ticketNotFound` | Ticket not found                                  | Không tìm thấy vé                               |
| `reconnecting`   | Reconnecting...                                   | Đang kết nối lại...                             |
| `offline`        | Offline — last updated {time} ago                 | Ngoại tuyến — cập nhật lần cuối {time} trước    |


---

## `theme` — Theme Toggle


| Key      | English | Vietnamese |
| -------- | ------- | ---------- |
| `light`  | Light   | Sáng       |
| `dark`   | Dark    | Tối        |
| `system` | System  | Hệ thống   |


---

## `language` — Language Switcher


| Key        | English         | Vietnamese      |
| ---------- | --------------- | --------------- |
| `switchTo` | Switch language | Chuyển ngôn ngữ |


---

## `time` — Relative Time Labels

Used in "Last updated X ago" and similar contexts. These are only fallback labels — primary time formatting uses `Intl.RelativeTimeFormat` via the active locale.


| Key       | English      | Vietnamese     |
| --------- | ------------ | -------------- |
| `justNow` | Just now     | Vừa xong       |
| `seconds` | {count}s ago | {count}s trước |
| `minutes` | {count}m ago | {count}m trước |


---

## Notes

- **Brand name "NotiGuide"** is kept as-is in both languages — it is a product name, not translatable.
- **Parameterized strings** use ICU MessageFormat `{variable}` syntax. next-intl parses these automatically.
- **Pluralization**: Vietnamese does not inflect for plurals, so no ICU plural rules are needed for these strings. The same `{count}` form works for singular and plural in Vietnamese.
- **Time formatting**: Ticket timestamps (`issuedAt`, `calledAt`) are formatted using fixed locale via `Intl.DateTimeFormat`, not through next-intl — per project convention (time display is language-agnostic).
- **Relative time** ("Last updated X ago") uses `Intl.RelativeTimeFormat` with the active locale for natural language output (e.g., "5 seconds ago" vs "5 giây trước").
- **Ticket number**: The `#{number}` format (e.g., "#042") is universal and not localized.
- **Status translations** are kept short for badge display. Both languages use concise forms that fit within badge UI components.

