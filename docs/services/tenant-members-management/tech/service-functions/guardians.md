# وظائف أولياء الأمور (Service Functions — Guardians)

## 1) ListGuardians

- **الهدف**: إرجاع قائمة أولياء الأمور الذين لديهم أبناء ضمن الجهة.
- **الصلاحية**: `tenant_members_management.guardians.view`
- **المدخلات**:
  - `search` (اختياري)
  - `page`, `per_page`
- **المخرجات**: `paginated list` مع `children_count`

## 2) LinkGuardianToStudent

- **الهدف**: ربط ولي أمر بطالب ضمن الجهة.
- **الصلاحية**: `tenant_members_management.guardians.link_student`
- **المدخلات**:
  - `guardian_national_id` (أو `guardian_user_id`)
  - `student_id`
- **التحقق**:
  - student ضمن الجهة
  - منع self-link
  - منع إعادة الربط إذا العلاقة فعّالة
- **الكتابة**:
  - upsert User/Guardian (إن لزم)
  - attach guardian↔student

## 3) UnlinkGuardianFromStudent

- **الهدف**: فصل العلاقة بين ولي وطالب.
- **الصلاحية**: `tenant_members_management.guardians.unlink_student`
- **المدخلات**:
  - `guardian_id`
  - `student_id`
- **الكتابة**:
  - set `unlinked_at = now()` على pivot

