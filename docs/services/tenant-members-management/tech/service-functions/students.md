# وظائف الطلاب (Service Functions — Students)

## 1) ListStudents

- **Goal**: إرجاع قائمة طلاب الجهة مع بحث/تصفية.
- **Auth**: `tenant_members_management.students.view`
- **Input**:
  - `search` (optional)
  - `stage_id` (optional)
  - `section_id` (optional)
  - `status` (optional)
  - `page`, `per_page`
- **Output**: `paginated list`

## 2) CreateStudentAndAttachToTenant

- **Goal**: إنشاء/ربط طالب جديد داخل الجهة مع فصل فعّال.
- **Auth**: `tenant_members_management.students.create`
- **Input**:
  - `national_id`
  - `name_ar` (+ optional `name_en`)
  - `active_status` (regular/affiliate)
  - `section_id` (mandatory)
- **Validation**:
  - منع وجود الطالب داخل نفس الجهة.
  - منع ارتباطه بجهة أخرى (حسب السياسة).
  - `section_id` يجب أن ينتمي للجهة.
- **Writes**:
  - Upsert `User`
  - Upsert `Student`
  - Attach Student↔Tenant
  - Attach Student↔Section + set `active_section_id`

## 3) MoveStudentToSection

- **Goal**: نقل طالب لفصل آخر داخل الجهة.
- **Auth**: `tenant_members_management.students.move_section`
- **Input**:
  - `student_id`
  - `to_section_id`
- **Validation**:
  - student ضمن الجهة
  - to_section ضمن الجهة
- **Writes (atomic)**:
  - release pivot القديم
  - attach pivot جديد
  - update `active_section_id`

## 4) UnlinkStudentFromTenant

- **Goal**: فصل الطالب من الجهة.
- **Auth**: `tenant_members_management.students.unlink`
- **Input**:
  - `student_id`
- **Writes**:
  - release Student↔Tenant pivot
  - release Student↔Section pivot (أو تحديث `active_section_id = null` حسب السياسة)
- **Notes**:
  - يجب منع بقاء الطالب بحالة “نشط” دون فصل/جهة.

