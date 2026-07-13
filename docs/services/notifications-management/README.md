# إدارة الإشعارات (Notifications Management)

توثيق خدمة **الإشعارات** في `okta-web`: مركز الإشعارات، القنوات (in_app /
واتساب / SMS / بريد / push)، القوالب، سياسات اللغة، وتفضيلات المستخدم. تقابلها
`App\Services\Notifications\*`.

> هي الطبقة التي تعتمدها التطبيقات المثبَّتة لإرسال إشعاراتها عبر المنصة
> (`PartnerApi\Notifications\*`) — راجع
> [`claude/web.md`](../../../claude/web.md).

## نظرة عامة

- [نظرة عامة على الخدمة](./overview.md)

## تجربة المستخدم (user-experience)

- **التدفقات**: [التدفقات](./user-experience/flows.md) · [الرحلات](./user-experience/journeys.md) · [الحالات الحدّية](./user-experience/edge-cases.md)
- **الشاشات**: [مركز الإشعارات](./user-experience/notifications-center.md) · [إعدادات المنصة](./user-experience/platform-settings.md) · [إعدادات الجهة](./user-experience/tenant-settings.md)

## الطبقة التقنية (tech)

- [المعمارية](./tech/architecture.md) · [تعدد المستأجرين](./tech/tenancy.md) · [الصلاحيات](./tech/permissions.md)
- **النماذج**: [الإشعار](./tech/models/notification.md) · [التعريف](./tech/models/notification-definition.md) · [القالب](./tech/models/notification-template.md) · [القناة](./tech/models/channel.md) · [سجل التسليم](./tech/models/delivery-log.md) · [التفضيل](./tech/models/preference.md) · [سياسة اللغة](./tech/models/locale-policy.md)
- **التعامل مع البيانات**: [النمذجة](./tech/data-handling/data_modeling.md) · [الترجمة](./tech/data-handling/localization.md) · [الطوابير وإعادة المحاولة](./tech/data-handling/queuing_and_retries.md) · [حدّ المعدّل](./tech/data-handling/rate_limiting_and_throttling.md)
- **وظائف الخدمة**: [نقاط النهاية](./tech/service-functions/api_endpoints.md) · [معالجات الأحداث](./tech/service-functions/event_handlers.md) · [خط العرض](./tech/service-functions/rendering_pipeline.md) · [المجدولات](./tech/service-functions/schedulers.md)
