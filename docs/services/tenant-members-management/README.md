# إدارة منسوبي الجهة (Tenant Members Management)

توثيق خدمة **إدارة منسوبي الجهة** في `okta-web`: الطلاب والموظفون وأولياء
الأمور داخل الجهة، أدوارهم، استيرادهم بالجملة، والعلاقات بينهم. تقابلها
`App\Services\Tenants\*`.

> أسماء الأدوار والعلاقات مصدرها
> [`roles-and-entities/`](../../roles-and-entities/README.md). قدرات مثل الحضور
> انتقلت إلى تطبيق مثبَّت ([`okta-hdor`](../../installed-apps/hdor/README.md)).

## نظرة عامة

- [نظرة عامة على الخدمة](./overview.md)

## الصفحات (pages)

- [الطلاب](./pages/students.md) · [تفاصيل الطالب](./pages/student-details.md) · [الموظفون](./pages/employees.md) · [أدوار الموظفين](./pages/employee-roles.md) · [أولياء الأمور](./pages/guardians.md)
- **الاستيراد**: [استيراد الطلاب](./pages/import-students.md) · [استيراد الموظفين](./pages/import-employees.md) · [صيغ ملفات مدير النظام](./pages/system-admin-file-formats.md)

## تجربة المستخدم (user-experience)

- [التدفقات](./user-experience/flows.md) · [الرحلات](./user-experience/journeys.md) · [الحالات الحدّية](./user-experience/edge-cases.md) · [سياسة الصف الإلزامي](./user-experience/mandatory-grade-policy.md)

## الطبقة التقنية (tech)

- [المعمارية](./tech/architecture.md) · [تعدد المستأجرين](./tech/tenancy.md) · [الصلاحيات](./tech/permissions.md)
- [نماذج البيانات](./tech/models/README.md) — الطالب، الموظف، ولي الأمر، الصف، المرحلة، العلاقات.
- [وظائف الخدمة](./tech/service-functions/README.md) — الطلاب، الموظفون، أولياء الأمور، الاستيراد.
- **التعامل مع البيانات**: [نظرة عامة](./tech/data-handling/overview.md) · [خطوط الاستيراد](./tech/data-handling/import-pipelines.md) · [صيغ الملفات](./tech/data-handling/file-format-schemas.md)
