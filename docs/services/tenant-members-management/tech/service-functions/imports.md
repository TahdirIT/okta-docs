# وظائف الاستيراد (Service Functions — Imports)

## 1) PreviewImport

- **Goal**: Parsing + Validation + Preview بدون كتابة نهائية.
- **Auth**:
  - حسب النوع: `tenant_members_management.import.students` أو غيرها
- **Input**:
  - `import_type`: students | guardians | employees
  - `file`
  - (اختياري) `schema_variant`
- **Output**:
  - `rows[]` مع:
    - `data` (normalized)
    - `errors[]` (row-level)
  - `summary` (counts)

## 2) ExecuteImport

- **Goal**: تنفيذ الاستيراد النهائي.
- **Auth**: حسب النوع
- **Input**:
  - `import_session_id` أو `rows` بعد المعاينة
- **Writes**:
  - upsert entities
  - attach relations
  - dispatch post-jobs

## 3) DownloadEmptyTemplate (Okta template)

- **Goal**: تنزيل ملف Excel فارغ مُهيأ للجهة.
- **Auth**: `tenant_members_management.file_formats.download_templates`
- **Output**: ملف Excel

> يجب أن يحتوي القالب حقولاً تُحقق إلزامية الصف/الفصل.

