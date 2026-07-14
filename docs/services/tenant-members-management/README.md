# إدارة منسوبي الجهة (Tenant Members Management)

توثيق خدمة **إدارة منسوبي الجهة** في `okta-web`: الطلاب والموظفون وأولياء
الأمور داخل الجهة، أدوارهم، استيرادهم بالجملة، والعلاقات بينهم. تقابلها
`App\Services\Tenants\*`.

> أسماء الأدوار والعلاقات مصدرها
> [`roles-and-entities/`](../../roles-and-entities/README.md). الحضور يُقدَّم عبر
> التطبيق المثبَّت `okta-hdor` (موثَّق في مستودعه الخاص).

## نظرة عامة

- [نظرة عامة على الخدمة](./overview.md)

## تجربة المستخدم (user-experience)

- **التدفقات**: [التدفقات](./user-experience/flows.md) · [الرحلات](./user-experience/journeys.md) · [الحالات الحدّية](./user-experience/edge-cases.md) · [سياسة الصف الإلزامي](./user-experience/mandatory-grade-policy.md)
- **الشاشات**: [الطلاب](./user-experience/students.md) · [تفاصيل الطالب](./user-experience/student-details.md) · [الموظفون](./user-experience/employees.md) · [أدوار الموظفين](./user-experience/employee-roles.md) · [أولياء الأمور](./user-experience/guardians.md)
- **الاستيراد**: [استيراد الطلاب](./user-experience/import-students.md) · [استيراد الموظفين](./user-experience/import-employees.md) · [صيغ ملفات مدير النظام](./user-experience/system-admin-file-formats.md)

## الطبقة التقنية (tech)

- [المعمارية](./tech/architecture.md) · [تعدد المستأجرين](./tech/tenancy.md) · [الصلاحيات](./tech/permissions.md)
- [نماذج البيانات](./tech/models/README.md) — الطالب، الموظف، ولي الأمر، الصف، المرحلة، العلاقات.
- [وظائف الخدمة](./tech/service-functions/README.md) — الطلاب، الموظفون، أولياء الأمور، الاستيراد.
- **التعامل مع البيانات**: [نظرة عامة](./tech/data-handling/overview.md) · [خطوط الاستيراد](./tech/data-handling/import-pipelines.md) · [صيغ الملفات](./tech/data-handling/file-format-schemas.md)
