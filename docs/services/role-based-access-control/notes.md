# ملاحظات التنفيذ

## الحزم المستخدمة

- `spatie/laravel-multitenancy` — قاعدة بيانات واحدة (single database strategy).
- `spatie/laravel-permission` — مع تفعيل ميزة `teams = true` التي تمثّل المستأجرين.

## البنية الأساسية

- **Teams** في `spatie/laravel-permission` تمثّل المستأجرين في `spatie/laravel-multitenancy`، ويُدار الربط بينهما عبر Middleware.
- كل صلاحية أو دور يملك عنوان (`title_ar` / `title_en`, nullable) يُعرض للمستخدمين في الواجهة.
- **الأدوار القوالب** تُخزَّن بـ `tenant_id = null`؛ التعيين الفعلي للمستأجر يتم في `model_has_roles.tenant_id`.

## الأدوار المُنفَّذة

### System Scope
| الدور | الصلاحيات |
|---|---|
| `superadmin` | جميع صلاحيات `scope = system` |
| `platform-admin` | `rbac.*` + `tenants.*` |

### Tenant Scope
| الدور | الصلاحيات |
|---|---|
| `tenant-admin` | `users.*` + `roles.assign/revoke` + `tenants.members.*` |
| `reviewer` | `*.view` (tenant) |
| `member` | `users.view` |

## السياق (Context)

- المستخدم الذي لم يختر دوراً بعد يُوجَّه إلى `auth/context`.
- السياق يُحفظ في الجلسة: `scope`, `tenant_id`, `tenant_user_id`, `role_ids`, `active_role_name`.

## Middleware

```
EnsureContext          → يتحقق من وجود السياق في الجلسة
EnsureActiveTenantContext → يتحقق من المستأجر وربط المستخدم
EnsureActiveRoles      → يتحقق من أن الأدوار لم تُحذف
EnsureActiveUser       → يتحقق من أن المستخدم غير معلّق
EnsureTenantPermission → يتحقق من صلاحية محددة (tenant.permission:perm.name)
EnsureTenantRole       → يتحقق من دور محدد (tenant.role:role-name)
```

عند تفعيل سياق المستأجر، يُستدعى:
```php
app(PermissionRegistrar::class)->setPermissionsTeamId($tenantId);
```
وعند تغيير السياق:
```php
app(PermissionRegistrar::class)->forgetCachedPermissions();
```

## Tenant Scoping للموديلات

يتوفر trait مخصص `BelongsToTenant` في `app/Models/Concerns/BelongsToTenant.php` يضيف Global Scope تلقائي لأي موديل يحتوي على `tenant_id`:

```php
class Student extends Model {
    use BelongsToTenant; // يفلتر تلقائياً بـ Tenant::current()
}
```

للاستعلام بدون فلتر المستأجر: `Student::withoutTenantScope()->get()`

## القيود الفريدة

- `permissions`: `unique(['name', 'guard_name', 'scope'])`
- `roles`: `unique(['tenant_id', 'name', 'guard_name'])`
- `model_has_roles`: primary key يشمل `team_id` عند تفعيل Teams
