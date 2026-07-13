# وظائف أولياء الأمور (Service Functions — Guardians)

## 1) ListGuardians

- **Goal**: إرجاع قائمة أولياء الأمور الذين لديهم أبناء ضمن الجهة.
- **Auth**: `tenant_members_management.guardians.view`
- **Input**:
  - `search` (optional)
  - `page`, `per_page`
- **Output**: `paginated list` مع `children_count`

## 2) LinkGuardianToStudent

- **Goal**: ربط ولي أمر بطالب ضمن الجهة.
- **Auth**: `tenant_members_management.guardians.link_student`
- **Input**:
  - `guardian_national_id` (أو `guardian_user_id`)
  - `student_id`
- **Validation**:
  - student ضمن الجهة
  - منع self-link
  - منع إعادة الربط إذا العلاقة فعّالة
- **Writes**:
  - upsert User/Guardian (إن لزم)
  - attach guardian↔student

## 3) UnlinkGuardianFromStudent

- **Goal**: فصل العلاقة بين ولي وطالب.
- **Auth**: `tenant_members_management.guardians.unlink_student`
- **Input**:
  - `guardian_id`
  - `student_id`
- **Writes**:
  - set `unlinked_at = now()` على pivot

