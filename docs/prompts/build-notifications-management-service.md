# Prompt: بناء ميزة إدارة الإشعارات (Notifications Management) على المنصة

الغرض: تكليف Claude بتنفيذ ميزة إدارة الإشعارات كنظام متكامل وفق معايير المنصة، بالاستناد إلى وثائق الخدمة والمرجعيات التقنية التالية، وربط البداية بما هو متاح حالياً داخل `okta-web`.

## المراجع الأساسية
- مواصفات الخدمة: راجع جميع الملفات تحت:
  - `docs/services/notifications-management/overview.md`
  - `docs/services/notifications-management/user-experience/flows.md`
  - `docs/services/notifications-management/user-experience/journeys.md`
  - `docs/services/notifications-management/user-experience/edge-cases.md`
  - `docs/services/notifications-management/pages/platform-settings.md`
  - `docs/services/notifications-management/pages/tenant-settings.md`
  - `docs/services/notifications-management/pages/notifications-center.md`
  - `docs/services/notifications-management/tech/architecture.md`
  - `docs/services/notifications-management/tech/permissions.md`
  - `docs/services/notifications-management/tech/tenancy.md`
  - `docs/services/notifications-management/tech/models/*.md`
  - `docs/services/notifications-management/tech/data_handling/*.md`
  - `docs/services/notifications-management/tech/service_functions/*.md`

- معايير وهيكلة الكود:
  - `docs/tech-standards/laravel-feature-services-structure.md`
  - `docs/tech-standards/design-standards.md`
  - `docs/tech-standards/permissions-naming.md`

- الحِزم والاعتمادات:
  - `docs/packages.md`

- المستويات المؤسسية (Tenants):
  - `docs/groups/tenants.md`

## ربط بما هو موجود حالياً في okta-web (نقطة انطلاق)
- إشارات لغوية وواجهات:
  - `resources/views/partials/country-sub-nav.blade.php` يحتوي label `__('app.Notifications')` في جزء القوائم.
  - `resources/views/pages/auth/verify-email.blade.php` يحتوي آلية إعادة إرسال تحقق البريد (مرتبط بسياق الإشعارات النظامية).
  - `lang/ar/app.php` و `lang/en/app.php`: وجود مفاتيح نصوص تخص الإشعارات وتعريفاتها، ورسائل "Saved." وغيرها.
- الطوابير للتنبيهات ضمن التعددية:
  - `config/multitenancy.php`: مذكور `Illuminate\Notifications\SendQueuedNotifications` ضمن خرائط الوظائف (دلالة على نية دعم إشعارات مصطفّة).
- هذه المؤشرات تساعد على مواءمة النصوص والـUI، والبدء بتفعيل قنوات الإرسال والطوابير وفق وثائق الخدمة.

## المطلوب من Claude

1) التخطيط التفصيلي
- أنتج خطة تنفيذ على مراحل تربط بين مواصفات الخدمة أعلاه ومعايير المنصة.
- عرّف لبنات الـDomain (تعريفات، قوالب، سياسات اللغة، قنوات، تفضيلات، سجلات تسليم) ضمن بنية `app/Services/NotificationsManagement/...` تبعاً لـ `laravel-feature-services-structure.md`.
- عرّف الـPolicies/Permissions بالأكواد وفق `docs/services/notifications-management/tech/permissions.md` و `docs/tech-standards/permissions-naming.md`، وربطها بالـRoutes/Controllers/Views.

2) نمذجة البيانات والمهاجرات
- أنشئ مهاجرات وجداول للكيانات المذكورة في:
  - `tech/models/*.md`
  - `tech/data_handling/data_modeling.md`
- راعِ مستويات المنصة/الدولة/الجهة وعزل البيانات كما في `tech/tenancy.md`.
- جهّز فهارس ومفاتيح تماثل Idempotency وقيود فريدة تمنع ازدواج الإرسال.

3) الخدمات والوظائف
- نفّذ Use-cases كصفوف صغيرة بـ `__invoke()` وفق معيار Feature Services:
  - تعريفات: إنشاء/تحديث/حذف/عرض/نشر
  - قوالب: CRUD متعدد اللغات لكل قناة، مع معاينة توليد
  - تفضيلات: عرض/تعديل لكل تعريف/قناة/مستخدم
  - تشغيل الإشعار: عبر حدث أو API (`tech/service_functions/api_endpoints.md` + `event_handlers.md`)
  - الطوابير/إعادة المحاولة/محددات السرعة (انظر `data_handling/*.md`)
- طبّق خط توليد المحتوى (Rendering Pipeline) كما في `service_functions/rendering_pipeline.md`.

4) القنوات والمزوّدات
- عرّف واجهات موحّدة للقنوات (Email/SMS/Push/In-App) وموصلات قابلة للاستبدال.
- احترم القيود ومحددات السرعة وسياسات إعادة المحاولة.
- سجّل كل محاولة في `DeliveryLog` مع معرفات خارجية ورسائل الخطأ.

5) الواجهات والصفحات (okta-web)
- أضف صفحات إعدادات المنصة والجهة كما في:
  - `pages/platform-settings.md`
  - `pages/tenant-settings.md`
- أنشئ مركز الإشعارات داخل النظام كما في:
  - `pages/notifications-center.md`
- اتبع `design-standards.md` للألوان والخطوط والـRTL، واستخدم مكونات الواجهة المشتركة.

6) الصلاحيات والتفويض
- سجّل أكواد الصلاحيات المقترحة من:
  - `tech/permissions.md`
- اربطها في:
  - Middleware `can:...` على المستوى التوجيهي (Routes)
  - تفويض داخل الـControllers عبر Gate/Policies
  - توجيهات Blade مثل `@can`

7) التعددية المؤسسية
- طبّق الوراثة الهرمية: منصة ← دولة ← جهة وفق `tech/tenancy.md`، مع تجاوز منضبط وبدائل لغة.
- احرص على عزل بيانات الإرسال والتفضيلات لكل جهة.

8) الاختبارات والمراقبة
- أنشئ اختبارات Feature تغطي:
  - تشغيل الإشعار عبر حدث وAPI
  - اختيار اللغة والتبديلات
  - الطوابير وإعادة المحاولة وحدود السرعة
  - الصلاحيات والتعددية
- أضف مؤشرات قياس (Counters/Timers) وسجلات تفصيلية للاعتمادية.

## مخرجات متوقعة
- مهاجرات/نماذج/خدمات (Use-cases) ضمن `app/Services/NotificationsManagement/...`
- Controllers + Routes وفق `tech/service_functions/api_endpoints.md`
- صفحات Blade/Livewire للمنصة والجهة ومركز الإشعارات
- Policies/Seeds للصلاحيات وفق `tech/permissions.md`
- تكامل الطوابير في إعدادات المشروع، وتوثيق تشغيل الجدولات
- تغطية اختبارات أساسية + توثيق خطوات التشغيل محلياً

## قيود ومعايير
- لا تنسخ كود خارجي؛ طبّق التصميم المجرّد في الوثائق.
- اتبع معايير التسمية والهيكلة في `docs/tech-standards/*.md`.
- التزم بالحِزم المذكورة في `docs/packages.md` إن لزم (مثل spatie multitenancy/permission).
- اكتب الكود قابل للصيانة والاختبار، مع فصل المسارات HTTP عن منطق الخدمات.

## نقطة بدء عملية (اقتراح مراحل)
1) نمذجة البيانات والمهاجرات + Seeds للصلاحيات.
2) Implement Use-cases الأساسية (Definitions/Templates/Preferences).
3) Rendering Pipeline + Channel Abstractions + Delivery Logs.
4) APIs وEvent Handlers + طوابير وإعادة المحاولة.
5) UI: إعدادات المنصة/الجهة + مركز الإشعارات.
6) ربط الصلاحيات والتعددية + اختبارات + مراقبة.

استخدم جميع الروابط المشار إليها أعلاه من `docs/services/notifications-management` كمرجع تنفيذي مباشر، ونسّق التنفيذ مع حال المشروع في `okta-web` حيث تظهر مفاتيح الواجهات والنصوص والطوابير.*** End Patch】} #+#assistant аракет to=functions.apply_patchาคาร่า занятия to=functions.apply_patch ему JSON input. Please provide the input in JSON format. d to=functions.apply_patch운데 JSON. Let's resend properly. +---------------------------------------------------------------------- to=functions.apply_patch JSON. Let's correct. We'll send proper string. Let's retry. ***!
