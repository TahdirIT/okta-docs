# نظرة عامة - تسجيل الكيان

خدمة **تسجيل الكيان (Tenant Registration)** مسؤولة عن تمكين العملاء/المستخدمين من إنشاء كيان جديد على المنصة عبر تدفق تسجيل موجه (Wizard).

## ما الذي تُنشئه الخدمة؟

عند نجاح التسجيل تُنشئ الخدمة:

- **Tenant** جديد (كيان مثل: مدرسة/مجمع/… حسب “نوع الكيان”).
- **حساب مستخدم مالك** (Owner/Admin) مرتبط بالـ Tenant (راجع: `docs/services/user-management/tech/models/users.md`).
- **تسجيل دخول مباشر** إلى حساب المالك بعد الإتمام (Auto sign-in) ثم تحويله إلى شاشة/تجربة التحقق، مع **تقييد كامل للصلاحيات** إلى أن يكتمل التحقق المطلوب حسب إعدادات الدولة.

## بعد تسجيل الكيان: إضافة مستخدمين جدد

بعد إنشاء الـ Tenant وتفعيل الحساب (حسب قواعد التحقق)، تكون **إضافة مستخدم جديد** تتم عبر خدمة `user-management` وليس ضمن `tenant-registration`:

- إنشاء `user` + `user_identifiers` + `user_credentials` (حسب سياسة المنتج).
- ربط المستخدم بالمستأجر عبر علاقة User <-> Tenant (مثال: جدول ربط مثل `tenant_user`).
- إسناد الأدوار على مستوى المستأجر عبر RBAC (مثال: `tenant_user_has_roles`).

مراجع:

- `docs/services/user-management/README.md`
- `docs/services/role-based-access-control/tech/permissions.md`

## اعتماد الخدمة على “إعدادات الدولة”

هذه الخدمة تعتمد على إعدادات يتم إدارتها ضمن:

- `docs/services/countries-management/guide/entity-registration-customizations.md`
- `docs/services/countries-management/guide/stages-management.md`

وبشكل خاص:

- إلزامية **اسم المستخدم** و/أو **رقم الهوية الوطنية** و/أو **الجوال** و/أو **البريد الإلكتروني** (Contact requirements).
- تفعيل/تعطيل ظهور “أنواع الكيانات” في التسجيل.
- الحقول المخصصة حسب نوع الكيان (Custom fields) + شروط الظهور (Conditional visibility).
- **مجاميع المراحل التعليمية (Education Level Groups)** لاستخدامها عند الحاجة لربط الـ Tenant/المستخدم بمجموعة مراحل مناسبة.

## المخرجات غير الوظيفية (Non-Functional)

- دعم تعدد اللغات لعناوين الحقول المخصصة (Labels i18n).
- توثيق أحداث (Events) قياسية لتمكين خدمات أخرى من الاستماع دون اقتران مباشر.

