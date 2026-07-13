# إدارة الدول (Countries Management)

توثيق خدمة **إدارة الدول** في `okta-web`: البنية المرجعية للدول والمناطق
والمراحل التعليمية والتقويم والمواد والأعوام الدراسية التي تُبنى عليها بيانات
الجهات. تقابلها `App\Services\CountriesManagement\*`.

> أسماء الأنواع/الأدوار المرتبطة مصدرها
> [`roles-and-entities/`](../../roles-and-entities/README.md).

## الدليل (guide)

- [نظرة عامة](./guide/overview.md) · [هدف الميزة](./guide/feature-goal.md) · [بنية قاعدة البيانات](./guide/database-structure.md)
- [بيانات الدولة](./guide/country-data.md) · [المناطق](./guide/regions-management.md) · [المراحل](./guide/stages-management.md)
- [المواد](./guide/subjects-management.md) · [الأعوام الدراسية](./guide/academic-years.md) · [التقويم](./guide/calendar.md)
- [أيام الأسبوع](./guide/weekdays-settings.md) · [تخصيصات تسجيل الجهة](./guide/entity-registration-customizations.md)

## تجربة المستخدم (user-experience)

- [الفهرس](./user-experience/README.md) — بيانات الدولة، المناطق، المراحل، المواد، الأعوام، التقويم، أيام الأسبوع، تخصيصات التسجيل.

## الطبقة التقنية (tech)

- [نماذج البيانات](./tech/models/README.md) — الدول، المدن، المناطق، المراحل ومجموعاتها، المواد، الأعوام، التقويم، إعدادات النموذج.
- [التعامل مع البيانات](./tech/data-handling/README.md) · [الصلاحيات](./tech/permissions.md) · [وظائف الخدمة](./tech/service-functions.md)
