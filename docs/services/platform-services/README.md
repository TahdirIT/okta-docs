# خدمات المنصّة المتفرّقة (Platform Services)

خدمات `okta-web` الأصغر والمتقاطعة التي لا يفرد كلٌّ منها شجرة توثيق مستقلة.

## Tours — الجولات التعريفية

جولات onboarding داخل الواجهة. `app/Services/Tours/{TourCatalog,ResolveToursForUser,SyncBuiltinTours}`؛
حالة لكل مستخدم عبر `GET/POST /tours/{key}/state`؛ إدارة عبر `Livewire/Tours/TourManager`.
يعتمد `driver.js` (راجع [الحزم](../../reference/packages.md)).

## Impersonation — انتحال المستخدم (okta-web)

أداة platform-admin للدخول بهوية مستخدم. `app/Services/Impersonation/{StartImpersonation,StopImpersonation}`؛
`/impersonation/{start,stop}`؛ `Settings/TenantImpersonateModal`.
> مختلف عن انتحال okta-partners (`claude/partners.md`) — مستودع وميزة أخرى.

## Branding — هوية الجهة

ألوان/شعار/رموز ثيم لكل جهة. `app/Services/Branding/TenantBrand.php` عبر إعدادات
الجهة. (يختلف عن تنسيقات [منشئ صفحة الهبوط](../landing-builder/README.md).)

## Messaging — إرسال الرسائل متعدّد القنوات

خدمة إرسال عامّة (يستهلكها بثّ الجهات الحاوية وغيرها):
`app/Services/Messaging/{Messages,Channels,Recipients}` — حلّ القنوات والمستلمين
ثم الإرسال. منفصلة عن [الإشعارات](../notifications-management/README.md) (خط أنابيب
الإشعارات) وعن ميزة الباقة `messaging.broadcast`.

## Comms / BridgeSettings — جسر Connect (واتساب)

إعدادات جسر okta-whatsapp: `app/Services/Comms/BridgeSettings.php` (base URL،
`comms.connect.embed_sso_secret`، تفعيل Connect)، تُضبط من
`/settings/platform-delivery` (`PlatformDeliveryProviderModal`). يغذّي
[صندوق واتساب CRM](../crm/README.md) وسلسلة قنوات [OTP](../otp-service/README.md).

## Store — المراجعات والأوسمة + تجارة المتجر

- **المراجعات/الأوسمة**: `app/Services/Store/{Reviews,Badges}`؛ `Livewire/Store/{ModuleReviews,ReviewFormModal}` —
  تقييمات موديولات المتجر (submit/delete/summary) + أوسمة.
- **تجارة المتجر**: `app/Services/AppStore/{Checkout,RevenueSplit,Refunds,Subscriptions,Catalog,Webhooks}` —
  `InitiateAppCheckout`، `CalculateRevenueSplit`، `IssueAppRefund`، أهلية التجربة،
  `ExpireDueSubscriptions`. تكمّل آلية التثبيت الموثّقة في
  [`claude/web.md`](../../../claude/web.md) و[`claude/installed-apps.md`](../../../claude/installed-apps.md).

## Student Profile Panels — نقطة توسعة ملف الطالب

صفحة ملف الطالب لوحة بلوكات؛ التطبيقات المثبَّتة تسجّل panels عبر
`app/Support/StudentProfile/{StudentProfilePanels,StudentProfilePanel,BlockSource}`
والكروم الموحّد `<x-profile-block>`. التطبيق يسجّل من `boot()` الخاص به panel
باسم/عنوان/component/صلاحية. (تفصيل الآلية في `okta-web/CLAUDE.md` قسم «ملف الطالب».)
تجربة المستخدم للشاشة نفسها في
[`tenant-members-management/.../student-details`](../tenant-members-management/user-experience/student-details.md).
