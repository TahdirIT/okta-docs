# وظائف الطلاب (Service Functions — Students)

## 1) ListStudents

- **الهدف**: إرجاع قائمة طلاب الجهة مع بحث/تصفية.
- **الصلاحية**: `tenant_members_management.students.view`
- **المدخلات**:
  - `search` (اختياري)
  - `stage_id` (اختياري)
  - `section_id` (اختياري)
  - `status` (اختياري)
  - `page`, `per_page`
- **المخرجات**: `paginated list`

## 2) CreateStudentAndAttachToTenant

- **الهدف**: إنشاء/ربط طالب جديد داخل الجهة مع فصل فعّال.
- **الصلاحية**: `tenant_members_management.students.create`
- **المدخلات**:
  - `national_id`
  - `name_ar` (+ اختياري `name_en`)
  - `active_status` (regular/affiliate)
  - `section_id` (إلزامي)
- **التحقق**:
  - منع وجود الطالب داخل نفس الجهة.
  - منع ارتباطه بجهة أخرى (حسب السياسة).
  - `section_id` يجب أن ينتمي للجهة.
- **الكتابة**:
  - Upsert `User`
  - Upsert `Student`
  - Attach Student↔Tenant
  - Attach Student↔Section + set `active_section_id`

## 3) MoveStudentToSection

- **الهدف**: نقل طالب لفصل آخر داخل الجهة.
- **الصلاحية**: `tenant_members_management.students.move_section`
- **المدخلات**:
  - `student_id`
  - `to_section_id`
- **التحقق**:
  - student ضمن الجهة
  - to_section ضمن الجهة
- **الكتابة (atomic)**:
  - release pivot القديم
  - attach pivot جديد
  - update `active_section_id`

## 4) UnlinkStudentFromTenant

- **الهدف**: فصل الطالب من الجهة.
- **الصلاحية**: `tenant_members_management.students.unlink`
- **المدخلات**:
  - `student_id`
- **الكتابة**:
  - release Student↔Tenant pivot
  - release Student↔Section pivot (أو تحديث `active_section_id = null` حسب السياسة)
- **ملاحظات**:
  - يجب منع بقاء الطالب بحالة “نشط” دون فصل/جهة.

