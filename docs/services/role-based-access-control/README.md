# الأدوار والصلاحيات (Role-Based Access Control)

توثيق خدمة **RBAC** في `okta-web`: الأدوار، الصلاحيات، النطاقات (`system` /
`tenant` / `general`)، تبديل السياق، والـ middleware. تقوم على
`spatie/laravel-permission` (مع `teams` تمثّل المستأجرين) فوق
`spatie/laravel-multitenancy`.

> ⚠️ **المصدر الكنسي لأسماء الأدوار:**
> [`roles-and-entities/role-scopes.md`](../../roles-and-entities/role-scopes.md)
> هو مرجع الحقيقة لأسماء الأدوار وترتيبها (الأدوار الثابتة في okta-web:
> `superadmin`, `tenant-admin`, `finance-admin`؛ وأدوار المستخدم النهائي
> `administrator` / `teacher` / `student` / `guardian` / `guardian-delegate`).
> **تنبيه اتساق:** بعض ملفات هذا المجلد تعرض مجموعتَي أدوار قديمتين متعارضتين
> (مثل `reviewer`/`member` مقابل `admin`/`teacher`/`student`) وصيغ صلاحيات
> تخالف [معيار تسمية الصلاحيات](../../tech-standards/permissions-naming.md)
> (wildcards و`manage`). عند التعارض قدّم `roles-and-entities` والمعيار.
> توحيد هذه الملفات مُدرَج كعمل لاحق.

## المحتوى

- [ملاحظات التنفيذ](./notes.md) — الحزم والبنية الأساسية.
- [الدليل](./guide/README.md) — نظرة عامة، الأدوار والصلاحيات، الإدارة، تبديل السياق، الـ middleware، الـ seeders، وعزل المستأجر.
- [الطبقة التقنية — النماذج](./tech/models/01-permissions.md) و[كتالوج الصلاحيات](./tech/permissions.md).
- [تجربة المستخدم](./user-experience/README.md) — إدارة الصلاحيات والأدوار واختيار السياق.
