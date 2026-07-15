# وظائف الاستيراد (Service Functions — Imports)

## 1) PreviewImport

- **الهدف**: Parsing + Validation + Preview بدون كتابة نهائية.
- **الصلاحية**:
  - حسب النوع: `tenant_members_management.import.students` أو غيرها
- **المدخلات**:
  - `import_type`: students | guardians | employees
  - `file`
  - (اختياري) `schema_variant`
- **المخرجات**:
  - `rows[]` مع:
    - `data` (normalized)
    - `errors[]` (row-level)
  - `summary` (counts)

## 2) ExecuteImport

- **الهدف**: تنفيذ الاستيراد النهائي.
- **الصلاحية**: حسب النوع
- **المدخلات**:
  - `import_session_id` أو `rows` بعد المعاينة
- **الكتابة**:
  - upsert entities
  - attach relations
  - dispatch post-jobs

## 3) DownloadEmptyTemplate (Okta template)

- **الهدف**: تنزيل ملف Excel فارغ مُهيأ للجهة.
- **الصلاحية**: `tenant_members_management.file_formats.download_templates`
- **المخرجات**: ملف Excel

> يجب أن يحتوي القالب حقولاً تُحقق إلزامية الصف/الفصل.

