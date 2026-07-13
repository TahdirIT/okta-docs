# مخططات صيغ الملفات (File Format Schemas)

## لماذا Schema؟

لأن ملفات الجهات تختلف حسب:

- البلد
- مصدر الملف
- نسخة القالب
- ترتيب الأعمدة أو وجود صفوف علوية (Headers)

لذلك تُدار قراءة الملف بواسطة Schema قابل للتعديل، بدلاً من تثبيت الأعمدة في الكود.

## تعريف Schema (مفاهيمياً)

- `start_row`: بداية البيانات
- `increments`: مقدار الانتقال بين سجلات منطقية
- `fields`: خريطة “اسم الحقل المنطقي” → “موقعه في الملف”

### Field locator

كل حقل يمكن تعريفه كمؤشر:

- `x`: رقم العمود
- `y_offset`: إزاحة صفية
- `relative`: هل `y_offset` محسوب من صف السجل الحالي؟

## أمثلة Schemas

### 1) Students file

```json
{
  "start_row": 2,
  "increments": 1,
  "fields": {
    "name_ar": { "x": 2, "y_offset": 0, "relative": true },
    "national_id": { "x": 5, "y_offset": 0, "relative": true },
    "stage_name": { "x": 8, "y_offset": 0, "relative": true },
    "section_name": { "x": 9, "y_offset": 0, "relative": true }
  }
}
```

### 2) Guardians file

```json
{
  "start_row": 2,
  "increments": 1,
  "fields": {
    "guardian_name": { "x": 2, "y_offset": 0, "relative": true },
    "guardian_national_id": { "x": 5, "y_offset": 0, "relative": true },
    "guardian_phone": { "x": 6, "y_offset": 0, "relative": true }
  }
}
```

## إسناد schema حسب السياق

عند الاستيراد، يتم اختيار schema بواسطة مفاتيح مثل:

- `country_id`
- `import_type` (students / guardians / employees)
- `template_variant` (okta / source_a / source_b)

## الربط الأكاديمي ضمن schema

لتحقيق إلزامية الصف/الفصل، يجب أن يوفر الملف واحداً من:

- `section_id` مباشرة
- أو `stage_name` + `section_name` (مع مطابقة على بيانات الجهة)

