# إدارة المستخدمين (User Management)

هذا المجلد يحتوي على توثيق خدمة **إدارة المستخدمين** في المنصة.

تشمل الخدمة:

- تعريف نموذج المستخدم (User) وبياناته الأساسية.
- قواعد منع التكرار (Uniqueness) لحقول الهوية/اسم المستخدم/الجوال/البريد.
- حالات التحقق (Mobile/Email verification) وعلاقتها بتفعيل الحساب.
- نقاط التكامل مع خدمات أخرى مثل:
  - `tenant-registration` لإنشاء حساب مالك (Owner) أثناء تسجيل كيان جديد.
  - `role-based-access-control` لإدارة الأدوار والصلاحيات للمستخدمين.

## الملفات

- [نظرة عامة](./guide/01-overview.md)
- [تسجيل مستخدم (Registration)](./guide/02-user-registration.md)
- [تسجيل الدخول (Login)](./guide/03-login.md)
- [تحديث بيانات المستخدم (User Update)](./guide/04-user-update.md)
- [Tech](./tech/README.md)
- [نماذج البيانات (Models)](./tech/models/README.md)
- [وظائف/Use-cases الخدمة](./tech/service-functions.md)

