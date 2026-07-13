# الأدوار والصلاحيات (Role-Based Access Control)

توثيق خدمة **RBAC** في `okta-web`: الأدوار، الصلاحيات، النطاقات (`system` /
`tenant` / `general`)، تبديل السياق، والـ middleware. تقوم على
`spatie/laravel-permission` (مع `teams` تمثّل المستأجرين) فوق
`spatie/laravel-multitenancy`.

> **المصدر الكنسي لأسماء الأدوار:**
> [`roles-and-entities/role-scopes.md`](../../roles-and-entities/role-scopes.md).
> الأدوار الثابتة المبذورة في okta-web ثلاثة: `superadmin` و`finance-admin`
> (نظام) و`tenant-admin` (مستأجر) — بالإضافة إلى أدوار CRM النظامية (`crm_*`).
> أدوار المستخدم النهائي (`administrator` كمجموعات لكل جهة، `teacher`،
> `student`/`guardian` بنطاق `general`، `guardian-delegate`) تُسنَد ديناميكياً
> لكل جهة، وليست أدواراً ثابتة.
>
> تمّت **مواءمة هذا المجلد مع كود okta-web** (يوليو 2026): أُزيلت الأدوار
> الوهمية (`platform-admin` مُزال بترحيل، و`reviewer`/`member` مُسقَطان)، وصُحِّح
> القيد الفريد للصلاحيات إلى `unique(name, guard_name, scope)`، وأُكملت قائمة
> الـ middleware الفعلية، وأُزيلت صلاحيات wildcard الوهمية (`rbac.*`) بما يوافق
> [معيار تسمية الصلاحيات](../../tech-standards/permissions-naming.md).

## المحتوى

- [ملاحظات التنفيذ](./notes.md) — الحزم والبنية الأساسية.
- [الدليل](./guide/README.md) — نظرة عامة، الأدوار والصلاحيات، الإدارة، تبديل السياق، الـ middleware، الـ seeders، وعزل المستأجر.
- [الطبقة التقنية — النماذج](./tech/models/01-permissions.md) و[كتالوج الصلاحيات](./tech/permissions.md).
- [تجربة المستخدم](./user-experience/README.md) — إدارة الصلاحيات والأدوار واختيار السياق.
