# خدمات المنصة (Services)

فهرس توثيق خدمات منصة أوكتا. تنقسم القدرات إلى **خدمات جوهرية** تعيش داخل
`okta-web` (مصدر الحقيقة للبيانات)، و**قدرات تُقدَّم كتطبيقات مثبَّتة** (partner
apps) تُثبَّت لكل جهة من المتجر ولها مستودعاتها المستقلة.

> للصورة الشاملة عبر المستودعات راجع [`../../CLAUDE.md`](../../CLAUDE.md) وطبقة
> المراجع [`../../claude/`](../../claude/). لعقد التطبيقات المثبَّتة راجع
> [`claude/installed-apps.md`](../../claude/installed-apps.md).

---

## الخدمات الجوهرية (داخل okta-web)

كل خدمة موثَّقة في مجلدها. يقابل كل منها خدمة فعلية تحت
`okta-web/app/Services/*`.

| الخدمة | تغطّي | التوثيق |
|---|---|---|
| إدارة المستخدمين | حسابات المستخدمين، الهوية، منع التكرار، التحقق | [`user-management/`](user-management/README.md) |
| تسجيل الكيان | تسجيل جهة جديدة وإنشاء حساب المالك | [`tenant-registration/`](tenant-registration/README.md) |
| إدارة منسوبي الجهة | الطلاب والموظفون وأولياء الأمور داخل الجهة | [`tenant-members-management/`](tenant-members-management/README.md) |
| الأدوار والصلاحيات (RBAC) | الأدوار، الصلاحيات، النطاقات، تبديل السياق | [`role-based-access-control/`](role-based-access-control/README.md) |
| إدارة الدول | الدول والمناطق والمراحل والتقويم والمواد والأعوام الدراسية | [`countries-management/`](countries-management/README.md) |
| الإشعارات | مركز الإشعارات والقنوات والقوالب | [`notifications-management/`](notifications-management/README.md) |
| خدمة OTP | إرسال والتحقق من رموز التحقق | [`otp-service/`](otp-service/README.md) |
| منشئ صفحة الهبوط | محرّر الموقع العام لكل جهة | [`landing-builder/`](landing-builder/README.md) |
| بوابة الدفع والمحاسبة | فوترة الجهات مقابل الاشتراكات + مزامنة وافق | [`payment-gateway/`](payment-gateway/README.md) |
| النظام المالي (Finance) | محاسبة داخلية landlord: موردون، حوافظ صرف، اعتمادات، فواتير، عُهد، تقارير، وافق | [`finance/`](finance/README.md) |
| الاشتراكات ومميزات الباقات | الباقات ودورة الاشتراك + toggle القدرات لكل باقة (PlanGate) | [`subscriptions/`](subscriptions/README.md) |
| كونسول الجهات الحاوية | إدارة المجمعات لجهاتها التابعة (شجرة، دخول، تقارير، تعميم) | [`tenant-hierarchy/`](tenant-hierarchy/README.md) |
| نظام الإحالات | أكواد إحالة، مكافآت، محافظ، مؤثّرون، مكافحة احتيال | [`referral-system/`](referral-system/README.md) |
| نظام CRM | حسابات/عملاء/عقود/تقويم/مستندات/فواتير + صندوق واتساب مضمَّن | [`crm/`](crm/README.md) |
| خدمات المنصّة المتفرّقة | Tours، Impersonation، Branding، Messaging، Comms، مراجعات المتجر، لوحات ملف الطالب | [`platform-services/`](platform-services/README.md) |

## قدرات تُقدَّم كتطبيقات مثبَّتة

**الاختبارات** و**الجدول الدراسي** و**الحضور** تطبيقات شريكة مثبَّتة، لكلٍّ
مستودعه المستقل الذي يحوي توثيقه. نظرة المنصّة ودليل التطوير في
[`../apps/`](../apps/README.md).

---

## بنية توثيق الخدمة (المعتمدة)

كل خدمة جوهرية يُفضَّل أن تتبع نفس التخطيط لتسهيل التنقّل:

```
<service>/
├── README.md            # فهرس الخدمة (مدخل موحّد + جدول ملفات)
├── guide/               # دليل المنتج/التدفقات (مرقّم 01-, 02- عند الترتيب)
├── user-experience/     # تفاصيل تجربة المستخدم: التدفقات والرحلات ومواصفات الشاشات
└── tech/                # الطبقة التقنية
    ├── README.md
    ├── models/          # نماذج البيانات (+ README)
    ├── data-handling/   # التعامل مع البيانات
    ├── service-functions/  # الـ use-cases
    └── permissions.md
```

كل خدمة تملك فهرس `README.md` جذرياً، والمجلدات موحّدة بالشرطة
(`data-handling` / `service-functions`)، وكل محتوى تجربة المستخدم (بما فيه
مواصفات الشاشات) تحت `user-experience/`. تبايُن
مقبول باقٍ: بعض ملفات النماذج تحمل أسماء `snake_case` تعكس أسماء الجداول الفعلية.
