# خطوط الاستيراد (Import Pipelines)

## خط استيراد الطلاب (Students import)

### المدخلات

- ملف Excel/CSV
- Tenant context
- File format schema مناسب (حسب البلد/السياق)

### المراحل

1) **Load schema**
- قراءة إعدادات `start_row`, `increments`, وخرائط الحقول.

2) **Parse**
- تحويل صفوف Excel إلى “Rows” منطقية موحدة.
- توحيد القيم (normalization) مثل:
  - trim
  - تحويل صيغ تاريخ مختلفة إلى ISO
  - توحيد تمثيل الهاتف

3) **Validate**
- قواعد عامة:
  - الاسم/الهوية مطلوبة
  - قيم الهوية ضمن طول/صيغة محددة
- قواعد Tenant:
  - الطالب غير مرتبط بجهة أخرى (حسب السياسة)
  - إلزام section/stage (Mandatory Grade Policy)

4) **Preview**
- إظهار الأخطاء لكل Row
- السماح بإزالة صفوف أو تعديل قيم قبل التنفيذ

5) **Execute (transactional)**
- Upsert Users
- Upsert Students
- Attach Student↔Tenant
- Attach Student↔Section + تحديث `active_section_id`
- ربط أولياء الأمور إن توفر

6) **Post-jobs**
- Set default passwords (إن كانت السياسة كذلك)
- Dedup pivots / Clean relations

## خط استيراد أولياء الأمور (Guardians import)

- parsing + upsert User/Guardian
- ربط guardian↔student يعتمد على:
  - هوية الطالب
  - أو mapping عبر أعمدة محددة

## خط استيراد الموظفين (Employees import)

- parsing + upsert User/Employee
- attach employee↔tenant
- تعيين الدور الافتراضي/الأساسي
- jobs لضبط كلمات المرور الافتراضية

