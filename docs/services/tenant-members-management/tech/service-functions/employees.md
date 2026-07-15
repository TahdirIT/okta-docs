# وظائف الموظفين (Service Functions — Employees)

## 1) ListEmployees

- **الهدف**: إرجاع قائمة موظفي الجهة مع بحث.
- **الصلاحية**: `tenant_members_management.employees.view`
- **المدخلات**:
  - `search` (اختياري)
  - `page`, `per_page`

## 2) AddEmployeeToTenant

- **الهدف**: إضافة موظف جديد أو ربط موظف موجود بالجهة.
- **الصلاحية**: `tenant_members_management.employees.create`
- **المدخلات**:
  - `national_id`
  - `name_ar` (+ اختياري)
  - `primary_role_id`
  - `extra_role_ids[]` (اختياري)
- **التحقق**:
  - منع وجود الموظف داخل الجهة مسبقاً
  - التأكد من أن الأدوار tenant-scoped ضمن نفس الجهة
- **الكتابة**:
  - upsert User
  - upsert Employee
  - attach employee↔tenant + assign roles

## 3) UpdateEmployeeRoles

- **الهدف**: تحديث أدوار موظف داخل الجهة.
- **الصلاحية**: `tenant_members_management.employees.update`
- **المدخلات**:
  - `employee_id`
  - `primary_role_id`
  - `extra_role_ids[]`
- **الكتابة**:
  - sync roles tenant-scoped

## 4) ChangeEmployeePassword

- **الهدف**: تغيير كلمة المرور لموظف (إجراء إداري).
- **الصلاحية**: `tenant_members_management.employees.change_password`
- **المدخلات**:
  - `employee_id`
  - `new_password`

## 5) UnlinkEmployeeFromTenant

- **الهدف**: فصل الموظف من الجهة.
- **الصلاحية**: `tenant_members_management.employees.unlink`
- **الكتابة**:
  - set `released_at`
  - (اختياري) revoke roles tenant-scoped

