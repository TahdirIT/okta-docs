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
| تسجيل الكيان | تسجيل جهة جديدة وإنشاء حساب المالك | [`tenant-registration/`](tenant-registration/) |
| إدارة منسوبي الجهة | الطلاب والموظفون وأولياء الأمور داخل الجهة | [`tenant-members-management/`](tenant-members-management/overview.md) |
| الأدوار والصلاحيات (RBAC) | الأدوار، الصلاحيات، النطاقات، تبديل السياق | [`role-based-access-control/`](role-based-access-control/) |
| إدارة الدول | الدول والمناطق والمراحل والتقويم والمواد والأعوام الدراسية | [`contries-management/`](contries-management/) |
| الإشعارات | مركز الإشعارات والقنوات والقوالب | [`notifications-management/`](notifications-management/overview.md) |
| خدمة OTP | إرسال والتحقق من رموز التحقق (مفاهيمي) | [`otp-service/`](otp-service/guide/README.md) |
| منشئ صفحة الهبوط | محرّر الموقع العام لكل جهة | [`landing-builder/`](landing-builder/README.md) |
| بوابة الدفع والمحاسبة | فوترة الجهات مقابل الاشتراكات + مزامنة وافق | [`payment-gateway/`](payment-gateway/overview.md) |

## قدرات تُقدَّم كتطبيقات مثبَّتة (لا خدمات جوهرية)

هذه القدرات لم تعد موديولات داخل `okta-web` — بل **تطبيقات شريكة مثبَّتة** لكل
مستودعها الخاص، تُثبَّت من المتجر:

| القدرة | التطبيق المثبَّت | التوثيق |
|---|---|---|
| إدارة الاختبارات النهائية | `okta-exams` | [`exams-management/`](exams-management/overview.md) · [`claude/installed-apps.md`](../../claude/installed-apps.md) |
| الجدول الدراسي الذكي | `okta-smart-timetable` | [`claude/installed-apps.md`](../../claude/installed-apps.md) |
| الحضور والانصراف | `okta-hdor` | [`claude/installed-apps.md`](../../claude/installed-apps.md) |

> قدرات أخرى وردت تاريخياً في قائمة «الخدمات الأساسية» (الدرجات، السلوك،
> الأعذار، النداء، الإجازات…) لم تُفرَد بعد بمجلد توثيق مستقل؛ تُضاف هنا عند
> توثيقها. لا تُنشئ قائمة قدرات ثابتة منفصلة عن هذا الفهرس.

---

## بنية توثيق الخدمة (المعتمدة)

كل خدمة جوهرية يُفضَّل أن تتبع نفس التخطيط لتسهيل التنقّل:

```
<service>/
├── README.md            # فهرس الخدمة (مدخل موحّد + جدول ملفات)
├── guide/               # دليل المنتج/التدفقات (مرقّم 01-, 02- عند الترتيب)
├── user-experience/     # تفاصيل تجربة المستخدم (شاشات/تدفقات)
└── tech/                # الطبقة التقنية
    ├── README.md
    ├── models/          # نماذج البيانات (+ README)
    ├── data-handling/   # التعامل مع البيانات
    ├── service-functions/  # الـ use-cases
    └── permissions.md
```

> التخطيط أعلاه هدف نتدرّج نحوه؛ لا تزال بعض الخدمات تستخدم فهارس/تسميات مختلفة
> (`overview.md` بدل `README.md`، `pages/` بدل `user-experience/`، شرطة سفلية بدل
> شرطة). راجع اقتراح توحيد البنية عند إعادة التنظيم.
