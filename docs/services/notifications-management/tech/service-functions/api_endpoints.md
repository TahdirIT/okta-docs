# وظائف الخدمة: واجهات برمجية (APIs)

> صُحِّح هذا الملف ليطابق `okta-web` فعلياً. الصيغ السابقة (`/api/me/*` و
> `/api/admin/notification-definitions` و`/api/notifications/trigger`) **لم تكن
> موجودة في الكود** وأُزيلت.

## واجهة الجوال للإشعارات — `routes/api.php` (`/api/mobile/*`، `auth:sanctum`)

هذه الواجهة الوحيدة المكشوفة للإشعارات، يستهلكها تطبيق `okta-app`. راجع
[`claude/web.md`](../../../../../claude/web.md#2-mobile-client-api).

**تسجيل أجهزة الدفع (FCM):**
- `POST /api/mobile/notification-tokens` — body: `{ token, platform, app_version?, locale? }`
- `POST /api/mobile/notification-tokens/revoke` — إلغاء التسجيل عند تسجيل الخروج (قبل إتلاف توكن Sanctum). **لا يوجد** `DELETE .../{token}`.

**قراءة/تعليم الإشعارات:**
- `GET /api/mobile/notifications` — القائمة المُرقّمة (`filter=all|unread|read`، `page`، `per_page`).
- `GET /api/mobile/notifications/unread-count` — عدّاد غير المقروء.
- `POST /api/mobile/notifications/{id}/read` — تعليم إشعار مقروءاً. (**POST** لا `PUT`.)
- `POST /api/mobile/notifications/read-all` — تعليم الكل مقروءاً.

تُخدَم عبر `App\Services\Notifications\Center\*` — نفس الخدمات التي تستخدمها صفحة
`/notifications-center` في الويب.

## تعريفات الإشعارات ليست واجهة REST

تعريفات الإشعارات (definitions) وقوالبها **كتالوج في الكود**
(`App\Services\Notifications\NotificationEventCatalog`)، تُدار من صفحات إعدادات
Livewire (منصّة/جهة) — **لا** عبر `/api/admin/notification-definitions` أو
`/api/{scope}/notification-templates` (غير موجودة). التطبيقات الشريكة تُطلق
الإشعارات in-process عبر `App\Services\PartnerApi\Notifications\DispatchNotification`
(راجع [`claude/web.md`](../../../../../claude/web.md#partner-notifications))، لا عبر
`POST /api/notifications/trigger`.

## التفضيلات (مستخدم)

تُدار تفضيلات القنوات لكل تعريف عبر `App\Services\Notifications\Preferences\*`
وصفحات الإعدادات، لا عبر واجهة REST مستقلة.
