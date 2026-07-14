# تسجيل الكيان (Tenant Registration)

توثيق خدمة **تسجيل الجهة** في `okta-web`: إنشاء جهة تعليمية جديدة (Tenant)
وحساب المالك المرتبط بها، مع بوابات التحقق. تتكامل مع
[`user-management`](../user-management/README.md) (إنشاء المالك) و
[`countries-management`](../countries-management/README.md) (تخصيصات التسجيل).

## الدليل (guide)

- [نظرة عامة](./guide/01-overview.md) · [هدف الميزة](./guide/02-feature-goal.md)
- [تدفّق التسجيل](./guide/03-registration-flow.md) · [بوابات التحقق](./guide/04-verification-gating.md) · [الأحداث](./guide/05-events.md)

## تجربة المستخدم (user-experience)

- [الفهرس](./user-experience/README.md) — [معالج التسجيل](./user-experience/01-registration-wizard-ux.md) · [تجربة التحقق](./user-experience/02-verification-ux.md)

## الطبقة التقنية (tech)

- [الفهرس التقني](./tech/README.md) · [نماذج البيانات](./tech/models/README.md) · [التعامل مع البيانات](./tech/data-handling/overview.md)
- [الصلاحيات](./tech/permissions.md) · [وظائف الخدمة](./tech/service-functions.md)
