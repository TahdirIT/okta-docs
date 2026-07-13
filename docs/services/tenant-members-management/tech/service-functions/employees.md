# وظائف الموظفين (Service Functions — Employees)

## 1) ListEmployees

- **Goal**: إرجاع قائمة موظفي الجهة مع بحث.
- **Auth**: `tenant_members_management.employees.view`
- **Input**:
  - `search` (optional)
  - `page`, `per_page`

## 2) AddEmployeeToTenant

- **Goal**: إضافة موظف جديد أو ربط موظف موجود بالجهة.
- **Auth**: `tenant_members_management.employees.create`
- **Input**:
  - `national_id`
  - `name_ar` (+ optional)
  - `primary_role_id`
  - `extra_role_ids[]` (optional)
- **Validation**:
  - منع وجود الموظف داخل الجهة مسبقاً
  - التأكد من أن الأدوار tenant-scoped ضمن نفس الجهة
- **Writes**:
  - upsert User
  - upsert Employee
  - attach employee↔tenant + assign roles

## 3) UpdateEmployeeRoles

- **Goal**: تحديث أدوار موظف داخل الجهة.
- **Auth**: `tenant_members_management.employees.update`
- **Input**:
  - `employee_id`
  - `primary_role_id`
  - `extra_role_ids[]`
- **Writes**:
  - sync roles tenant-scoped

## 4) ChangeEmployeePassword

- **Goal**: تغيير كلمة المرور لموظف (إجراء إداري).
- **Auth**: `tenant_members_management.employees.change_password`
- **Input**:
  - `employee_id`
  - `new_password`

## 5) UnlinkEmployeeFromTenant

- **Goal**: فصل الموظف من الجهة.
- **Auth**: `tenant_members_management.employees.unlink`
- **Writes**:
  - set `released_at`
  - (اختياري) revoke roles tenant-scoped

