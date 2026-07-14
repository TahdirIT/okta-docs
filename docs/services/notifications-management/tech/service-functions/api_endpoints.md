# وظائف الخدمة: واجهات برمجية (APIs)

## واجهة الجوال للإشعارات — `routes/api.php` (`/api/mobile/*`، `auth:sanctum`)

الواجهة المكشوفة للإشعارات، يستهلكها تطبيق `okta-app`. راجع
[`claude/web.md`](../../../../../claude/web.md#2-mobile-client-api).

**تسجيل أجهزة الدفع (FCM):**
- `POST /api/mobile/notification-tokens` — body: `{ token, platform, app_version?, locale? }`
- `POST /api/mobile/notification-tokens/revoke` — إلغاء التسجيل عند تسجيل الخروج (قبل إتلاف توكن Sanctum).

**قراءة/تعليم الإشعارات:**
- `GET /api/mobile/notifications` — القائمة المُرقّمة (`filter=all|unread|read`، `page`، `per_page`).
- `GET /api/mobile/notifications/unread-count` — عدّاد غير المقروء.
- `POST /api/mobile/notifications/{id}/read` — تعليم إشعار مقروءاً. (**POST** لا `PUT`.)
- `POST /api/mobile/notifications/read-all` — تعليم الكل مقروءاً.

تُخدَم عبر `App\Services\Notifications\Center\*` — نفس الخدمات التي تستخدمها صفحة
`/notifications-center` في الويب.

## تعريفات الإشعارات

تعريفات الإشعارات (definitions) وقوالبها **كتالوج في الكود**
(`App\Services\Notifications\NotificationEventCatalog`)، تُدار من صفحات إعدادات
Livewire (منصّة/جهة). التطبيقات الشريكة تُطلق الإشعارات in-process عبر
`App\Services\PartnerApi\Notifications\DispatchNotification` (راجع
[`claude/web.md`](../../../../../claude/web.md#partner-notifications)).

## التفضيلات (مستخدم)

تُدار تفضيلات القنوات لكل تعريف عبر `App\Services\Notifications\Preferences\*`
وصفحات الإعدادات.
