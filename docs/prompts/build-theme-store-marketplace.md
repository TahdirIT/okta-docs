# Prompt: بناء "متجر الثيمات ومنصة المطورين" (Theme Store & Developer Platform)

> **الغرض من هذا الملف:** تعليمات جلسة جديدة لـ Claude لبناء متجر ثيمات لصفحات الهبوط (على غرار متجر ثيمات سلة) + منصة مطورين متكاملة مربوطة بـ `okta-partners`، فوق ما أنجزناه سابقاً في **Landing Page Builder** داخل `okta-web`.

---

## 0) السياق السريع (اقرأ هذا قبل أي شيء)

- **الستاك الوحيد المعتمد:** Laravel 12 + Livewire 4 + Tailwind v4 + Vite 7.
- **ممنوع منعاً باتاً:** إدخال `Next.js` أو `React` أو أي تقنية أخرى خارج الستاك أعلاه.
- **لا تستخدم Flux في الكود الجديد** (الـ Landing Builder خالٍ منه، حافظ على ذلك هنا).
- **لا تستخدم `max-w-*`** في أي مكان. للعرض الكامل استخدم `w-full px-6 sm:px-10 lg:px-20`.
- **الأيقونات:** `afatmustafa/blade-hugeicons` فقط، وتحقق من وجود الاسم قبل استخدامه.
- **اللغة:** الواجهات عربية افتراضياً + دعم إنجليزي. كل المحتوى ثنائي اللغة `{ar, en}`.
- **التعددية (Tenancy):** `spatie/laravel-multitenancy` — السجلات التي تخص المنصّة (modules/themes) تعيش على **قاعدة اللاندلورد** (`landlord` connection)، بينما التثبيتات تعيش في قاعدة الجهة.

---

## 1) الأصول الموجودة التي يجب البناء عليها (لا تعد اختراع العجلة)

### 1.1 نظام الموديولز (موجود، استخدمه)
- جداول: `modules`, `module_versions`, `module_reviews`, `module_cross_access_grants`.
- نماذج: `App\Models\Module`, `ModuleVersion`, `ModuleReview`, `ModuleCrossAccessGrant`.
- Enum دورة الحياة: `App\Enums\ModuleStatus` — فيها State Machine كاملة (`Draft → Submitted → InReview → Approved → Beta → Published → Suspended/Deprecated`). **أعد استخدامه كما هو**.
- صفحات المتجر (Livewire):
  - `App\Livewire\Store\IndexPage` → `/store`
  - `App\Livewire\Store\ShowPage` → `/store/{slug}`
  - `App\Livewire\Store\InstallModal`
  - `App\Livewire\Store\InstalledPage`
  - `App\Livewire\Store\ModuleCrossAccessSection`
- جدول التثبيت: `tenant_module_installations` (استعلامات خام حالياً).

### 1.2 Landing Page Builder (أُنجز في الجلسة السابقة)
- `App\LandingBuilder\BlockRegistry` — سجل بلوكات مع `register/get/byCategory/sanitize`.
- `App\LandingBuilder\Contracts\Block` — العقد المجرد للبلوكات.
- البلوكات الحالية: `Header, Footer, Hero, Features, Faq, Cta, CustomHtml, ...` في `app/LandingBuilder/Blocks/`.
- نموذج الصفحة: `App\Models\TenantLandingPage` — فيه `draft_blocks, published_blocks, draft_theme, published_theme, seo, status`.
- محرر الصفحة: `App\Livewire\TenantLanding\Editor` + `resources/views/livewire/tenant-landing/editor.blade.php`.
- الإعدادات الجمالية: `App\Models\TenantBrandSettings` + `TenantBrandSettings::defaults()`.
- النشر العام: `PublicLandingController@__invoke` + `viaSlug` (سابعاً كـ fallback `/{tenantSlug}`).

### 1.3 منصة الشركاء (`okta-partners`)
- طلب الانضمام: `App\Livewire\Partners\ApplicationForm`.
- مرجع الوثائق: `docs/partner-platform.md` في `okta-tech-docs`.
- الفكرة: الشريك = tenant من نوع خاص (partner tenant) يملك الموديولز/الثيمات.
- حقل الارتباط الموجود: `modules.partner_tenant_id`.

### 1.4 الصلاحيات الموجودة (أُضيفت في Landing Builder)
- `landing.edit` (scope: tenant)
- `landing.publish` (scope: tenant)
- `landing.manage_domains` (scope: tenant)
- `landing.advanced_blocks` (scope: system)

---

## 2) نموذج النطاق (Domain Model)

### 2.1 ما هو "الثيم" عملياً؟
الثيم = **حزمة جاهزة من بلوكات Landing Builder + إعدادات الثيم (ألوان/خطوط) + صور وموارد**، يقوم الشريك ببنائها ثم نشرها في المتجر، فتقوم الجهة بتثبيتها وتخصيصها.

الثيم **ليس كود PHP جديد** — هو JSON + أصول (صور/خطوط) يستهلكها المحرّك الحالي. هذا يعني:
- آمن (لا تنفيذ كود طرف ثالث)
- قابل للإصدار (مجرد JSON مع نسخ)
- قابل للمعاينة والتخصيص مباشرة في المحرر الحالي.

### 2.2 توحيد المتجر مع الموديولز (قرار معماري)
بدل إنشاء نظام موازٍ، **وسّع جدول `modules` الموجود** بعمود `type` (`module | theme`)، وأضف حقل `payload` داخل الـ manifest خاص بالثيمات. هذا يضمن:
- إعادة استخدام State Machine (`ModuleStatus`)
- إعادة استخدام دورة المراجعة (`ModuleReview`)
- إعادة استخدام ربط الشريك (`partner_tenant_id`)
- إعادة استخدام صفحات المتجر (مع تبويب/فلتر `type=theme`).

### 2.3 بنية الـ Manifest للثيم (معيار)
كل ثيم يُخزَّن في `modules.manifest` (JSON) ويجب أن يطابق هذا المخطط:

```jsonc
{
  "type": "theme",
  "schemaVersion": 1,
  "slug": "modern-academy",
  "displayName":   "أكاديمية حديثة",
  "displayNameEn": "Modern Academy",
  "description":   "ثيم مؤسسي لمؤسسات التدريب",
  "descriptionEn": "Corporate theme for training institutions",
  "category": "education",            // education | corporate | minimal | creative
  "tags": ["rtl", "arabic-first", "dark"],
  "thumbnail": "https://cdn.../thumb.webp",
  "screenshots": [
    "https://cdn.../shot-1.webp",
    "https://cdn.../shot-2.webp"
  ],
  "author": {
    "partnerTenantId": 42,
    "name": "Acme Creative",
    "website": "https://acme.example"
  },
  "version": "1.0.3",
  "license": "commercial",            // free | commercial | subscription
  "price": { "amount": 99, "currency": "SAR", "model": "one_time" },
  "compatibility": {
    "platformMinVersion": "2026.04",
    "requiredBlocks": ["header", "hero", "features", "footer"]
  },
  "payload": {
    "theme":  { /* يتطابق مع TenantBrandSettings::defaults() */ },
    "seo":    { "ar": {...}, "en": {...} },
    "locales": ["ar", "en"],
    "defaultLocale": "ar",
    "blocks": [
      {
        "id": "uuid-placeholder-1",
        "type": "header",
        "order": 0,
        "content": { "ar": {...}, "en": {...} },
        "settings": { "padding": {"top": 24, "bottom": 24} },
        "visibility": { "mobile": true, "tablet": true, "desktop": true }
      }
      /* بقية البلوكات بالترتيب... */
    ]
  }
}
```

**ملاحظات حرجة على المخطط:**
- كل `block` داخل `payload.blocks` يجب أن يمرّ بـ `BlockRegistry::sanitize()` قبل القبول.
- `id` يُعاد توليده عند التثبيت لدى الجهة (لئلا تتعارض UUIDs).
- `price.amount = 0` أو `license = "free"` → ثيم مجاني.
- `schemaVersion` يُزاد عند كسر التوافق ويُكتب له Upgrader.

---

## 3) رحلة المطوّر (Developer Workflow) — عبر `okta-partners`

> **ملاحظة:** كل الأدوات التي تخص المطوّر تعيش داخل `okta-partners` (مشروع منفصل). `okta-web` يستقبل فقط الـ manifest النهائي عبر API معتمدة.

### 3.1 التسجيل والاعتماد
1. الشريك يُسجّل طلباً عبر `App\Livewire\Partners\ApplicationForm`.
2. فريق المنصة يُراجع ويوافق → تُفعَّل بيئة المطور لهذا الشريك.
3. يحصل الشريك على `partner_tenant_id` (مستخدم أصلاً في `modules.partner_tenant_id`).
4. تُصدر له `OAuth2 Personal Access Token` عبر Passport/Sanctum مع scope: `themes:write themes:read`.

### 3.2 أدوات البناء (داخل okta-partners)
أضف في `okta-partners` ثلاث ميزات:

- **Theme Studio** — نفس `App\Livewire\TenantLanding\Editor` ولكن في وضع "Theme Authoring":
  - لا يحفظ إلى `TenantLandingPage`، بل إلى `partner_theme_drafts`.
  - يدعم **Preview as Tenant** (تحميل ثيم على بيانات وهمية).
  - يدعم تحميل الأصول لـ CDN خاص بالشركاء.
- **Theme Inspector** — يعرض الـ manifest مباشرة ويجري:
  - `BlockRegistry::sanitize()` على كل بلوك.
  - تحقّق من `compatibility.requiredBlocks`.
  - تحقّق من وجود الترجمات العربية والإنجليزية.
- **Submit for Review** — ينشر الـ manifest إلى `okta-web` عبر API.

### 3.3 الـ API التي يجب بناؤها في `okta-web`
كل هذه مسارات باسم `/api/partners/themes/*` ومحمية بـ `auth:sanctum` + scope `themes:write`:

| Method | Path | الغرض |
|---|---|---|
| `POST` | `/api/partners/themes` | إنشاء/تحديث مسودة ثيم (يستلم manifest كامل) |
| `POST` | `/api/partners/themes/{slug}/submit` | تحويل الحالة من `Draft → Submitted` |
| `POST` | `/api/partners/themes/{slug}/versions` | نشر نسخة جديدة (SemVer) |
| `GET`  | `/api/partners/themes/{slug}` | قراءة الثيم + آخر حالة مراجعة |
| `GET`  | `/api/partners/themes` | قائمة ثيمات الشريك |

جميعها يجب أن:
1. تُطبِّق `partner_tenant_id` تلقائياً من التوكن (لا يأتي من الجسم).
2. تمنع تعديل ثيم ليس مملوكاً للشريك (Policy: `ThemePolicy`).
3. ترمي Validation Exception واضحة إن فشل `BlockRegistry::sanitize()` لأي بلوك.

### 3.4 رحلة المراجعة والنشر
تُعاد نفس State Machine في `ModuleStatus`:
```
Draft → Submitted → InReview → (Approved | ChangesRequested | Rejected)
Approved → Beta → Published
Published → Suspended | Deprecated
```
واجهة الأدمن لمراجعة الثيمات يجب أن تعيد استخدام صفحات مراجعة الموديولز الموجودة، بفلتر `type=theme` فقط.

---

## 4) رحلة الجهة (Tenant Workflow) — داخل `okta-web`

### 4.1 تصفح المتجر
- المسار: `GET /store/themes` → `App\Livewire\Store\ThemesIndexPage`.
- فلاتر: الفئة (`category`)، السعر (مجاني/مدفوع)، اللغة (`rtl/ltr`)، النجوم.
- كل كرت ثيم يعرض: الصورة المصغّرة، الاسم، الشريك، التقييم، السعر، زر "عرض".

### 4.2 صفحة تفاصيل الثيم
- المسار: `GET /store/themes/{slug}` → `App\Livewire\Store\ThemeShowPage`.
- تعرض: Screenshots، وصف، Changelog للنسخ، مراجعات الجهات، ثيمات مشابهة.
- أزرار:
  - **معاينة حية** → يفتح الثيم في iframe sandboxed على بيانات وهمية.
  - **تثبيت** → يفتح `ThemeInstallModal`.

### 4.3 المعاينة الحية (Preview)
- المسار: `GET /store/themes/{slug}/preview?locale=ar` (علني، بدون سياق جهة).
- يستعمل نفس `PublicLandingController` لكن مع بيانات الـ manifest مباشرة (لا DB).
- يُعرض `X-Frame-Options: SAMEORIGIN` حتى يمكن تضمينه في iframe.

### 4.4 التثبيت (Install)
**سلوك حرج — اقرأ جيداً:**
1. الجهة تضغط "تثبيت" → `ThemeInstallModal` يظهر.
2. يحذّر المستخدم: "سيتم استبدال المسوّدة الحالية لصفحة الهبوط. المنشورة لن تتأثر حتى تضغط نشر".
3. يعطي خيار: **استبدال كامل** أو **دمج** (ثم محلّل تعارضات لاحقاً — خارج MVP).
4. عند التأكيد:
   - يُنشأ سجل في `tenant_theme_installations` (جديد، قاعدة الجهة).
   - تُنسخ `payload.blocks` إلى `TenantLandingPage.draft_blocks` بعد:
     - توليد UUIDs جديدة لكل بلوك.
     - تشغيل `BlockRegistry::sanitize()` مجدداً.
   - تُنسخ `payload.theme` إلى `TenantLandingPage.draft_theme`.
   - تُنسخ `payload.seo` إلى `TenantLandingPage.seo` (إن لم تكن مضبوطة).
   - يُسجّل `installed_version_id` ليتمكن محرك الترقيات من المقارنة لاحقاً.
5. تحويل المستخدم إلى `Editor` لتخصيص الثيم المثبّت.

### 4.5 التخصيص بعد التثبيت
- **لا حاجة لأي واجهة جديدة.** يستخدم الـ `Editor` الحالي بدون تغيير.
- الجهة يمكنها: تعديل النصوص، تغيير الألوان، إضافة/حذف بلوكات، تغيير الصور.
- التخصيصات كلها محلية على `TenantLandingPage` ولا تُعدّل الثيم الأصلي.

### 4.6 الترقية (Upgrade)
- عند نشر الشريك لنسخة جديدة، تظهر للجهة شارة "يتوفر تحديث" في صفحة الإعدادات.
- زر الترقية يفتح مقارنة (diff) بين البلوكات الأصلية والمستخدمة.
- الجهة يمكنها: "الإبقاء على تخصيصاتي" أو "استبدال كامل".
- **MVP:** الاكتفاء بـ "استبدال كامل مع تحذير"، ومحلّل الـ diff في مرحلة لاحقة.

### 4.7 إلغاء التثبيت
- إزالة الثيم = تنبيه ثم مسح `draft_blocks` + إعادة تعيين `draft_theme` إلى `TenantBrandSettings::defaults()`.
- الصفحة المنشورة تبقى حتى يُعاد النشر يدوياً.

---

## 5) العمارة التقنية — ما يجب إنشاؤه

### 5.1 مهاجرات اللاندلورد (`database/migrations/landlord/`)

**أ) إضافة `type` لجدول `modules`:**
```php
// xxxx_xx_xx_add_type_to_modules_table.php
Schema::table('modules', function (Blueprint $t) {
    $t->string('type', 20)->default('module')->index();   // module | theme
});
```

**ب) جدول قيم الترقيم لسكيمة الثيمات (اختياري لكنه مُستحسن):**
```php
// xxxx_xx_xx_add_schema_version_to_module_versions_table.php
Schema::table('module_versions', function (Blueprint $t) {
    $t->unsignedSmallInteger('schema_version')->default(1);
});
```

### 5.2 مهاجرات الجهة (`database/migrations/`)

**ج) جدول تثبيتات الثيمات:**
```php
// xxxx_xx_xx_create_tenant_theme_installations_table.php
Schema::create('tenant_theme_installations', function (Blueprint $t) {
    $t->id();
    $t->foreignId('tenant_id')->constrained('tenants')->cascadeOnDelete();
    $t->string('module_slug')->index();       // نسخة soft من الـ slug لعدم الربط الصلب
    $t->unsignedBigInteger('module_version_id')->nullable();
    $t->string('installed_version', 20);
    $t->json('installed_manifest');           // snapshot لحماية من اختفاء الإصدار
    $t->timestamp('installed_at');
    $t->timestamp('uninstalled_at')->nullable();
    $t->unique(['tenant_id', 'module_slug']); // ثيم واحد فعّال لكل جهة
});
```

### 5.3 كود PHP الذي يجب إنشاؤه

```
app/
├── LandingBuilder/
│   ├── Themes/
│   │   ├── ThemeManifestValidator.php    # يُشغّل BlockRegistry::sanitize() + schema checks
│   │   ├── ThemeInstaller.php            # نسخ payload → TenantLandingPage
│   │   ├── ThemeUninstaller.php
│   │   └── ThemePreviewRenderer.php      # رندر manifest مباشرة (بدون DB)
│   └── BlockRegistry.php                 # (موجود — لا تلمس)
├── Models/
│   ├── Module.php                        # (موجود — أضف scope theme/module)
│   └── TenantThemeInstallation.php       # جديد
├── Http/
│   ├── Controllers/
│   │   ├── Store/
│   │   │   └── ThemePreviewController.php
│   │   └── Api/
│   │       └── Partners/
│   │           └── ThemesController.php   # 5 endpoints (§3.3)
│   └── Resources/
│       └── ThemeResource.php
├── Livewire/
│   ├── Store/
│   │   ├── ThemesIndexPage.php           # GET /store/themes
│   │   ├── ThemeShowPage.php             # GET /store/themes/{slug}
│   │   ├── ThemeInstallModal.php
│   │   └── InstalledThemesPage.php       # GET /store/themes/installed
│   └── TenantLanding/
│       └── ThemeUpgradeBanner.php        # يعرض "يتوفر تحديث"
└── Policies/
    └── ThemePolicy.php                   # تمنع شريك من تعديل ثيم شريك آخر

routes/
├── api.php             # أضف group /api/partners/themes
├── landing.php         # (موجود)
└── store.php           # أضف مسارات /store/themes

resources/views/
├── livewire/store/
│   ├── themes-index-page.blade.php
│   ├── theme-show-page.blade.php
│   ├── theme-install-modal.blade.php
│   └── installed-themes-page.blade.php
└── landing-builder/
    └── preview/
        └── standalone.blade.php         # يُستعمل من ThemePreviewController

database/seeders/
└── ThemeStoreSeeder.php                # ثيم تجريبي "Starter Free"
```

### 5.4 المسارات (Routes)

**في `routes/store.php`:**
```php
Route::middleware(['web', 'auth'])->prefix('store')->group(function () {
    Route::get('themes',                 ThemesIndexPage::class)->name('store.themes.index');
    Route::get('themes/installed',       InstalledThemesPage::class)->name('store.themes.installed');
    Route::get('themes/{slug}',          ThemeShowPage::class)->name('store.themes.show');
});

// علني (بدون auth) — للمعاينة
Route::get('/store/themes/{slug}/preview',
    [ThemePreviewController::class, '__invoke']
)->name('store.themes.preview');
```

**في `routes/api.php`:**
```php
Route::middleware(['auth:sanctum', 'ability:themes:write'])
    ->prefix('api/partners/themes')
    ->group(function () {
        Route::get('/',                ThemesController::class.'@index');
        Route::post('/',               ThemesController::class.'@store');
        Route::get('{slug}',           ThemesController::class.'@show');
        Route::post('{slug}/submit',   ThemesController::class.'@submit');
        Route::post('{slug}/versions', ThemesController::class.'@publishVersion');
    });
```

### 5.5 قاعدة التشغيل (Execution order)
نفّذ بهذا الترتيب ولا تبدأ خطوة قبل نجاح سابقتها:
1. المهاجرات (§5.1 + §5.2) + `php artisan migrate --path=... --database=landlord`.
2. `ThemeManifestValidator` + اختبار وحدة عليه.
3. `ThemeInstaller` + اختبار إدماج يُثبّت ثيم تجريبي على جهة اختبار.
4. `ThemePreviewController` + الـ view.
5. صفحات المتجر الثلاث (Index / Show / InstallModal).
6. Controller الـ API للشركاء + Policies.
7. `InstalledThemesPage` + إلغاء التثبيت.
8. `ThemeUpgradeBanner`.
9. Seeder لثيم تجريبي مجاني.
10. صفحات مراجعة الأدمن (بإعادة استخدام صفحات الموديولز مع فلتر `type=theme`).

---

## 6) الإيرادات والترخيص

### 6.1 نماذج التسعير المدعومة
- **مجاني (`free`)**: لا عملية شراء، تثبيت مباشر.
- **شراء لمرة واحدة (`one_time`)**: ترخيص دائم للجهة الحالية.
- **اشتراك (`subscription`)**: شهري/سنوي، يُجدَّد تلقائياً، يُلغى التثبيت عند عدم الدفع.

### 6.2 توزيع الإيرادات
- المنصة: **30%** افتراضياً (قابل للتعديل لكل شريك في `partners.revenue_share`).
- الشريك: **70%** افتراضياً.
- العملة الأساسية: `SAR`.
- الضرائب (VAT 15%) تُضاف على المستخدم النهائي وتُفصل محاسبياً.

### 6.3 جدول الإيرادات المطلوب (landlord)
```php
Schema::create('theme_purchases', function (Blueprint $t) {
    $t->id();
    $t->foreignId('tenant_id')->constrained();        // الجهة المشترية
    $t->foreignId('module_id')->constrained();        // الثيم
    $t->foreignId('module_version_id')->nullable()->constrained();
    $t->unsignedBigInteger('partner_tenant_id');
    $t->unsignedInteger('amount_halalas');            // بالهللة لتجنب عائمة النقطة
    $t->unsignedInteger('platform_cut_halalas');
    $t->unsignedInteger('partner_cut_halalas');
    $t->string('currency', 3)->default('SAR');
    $t->string('payment_provider')->nullable();       // stripe | moyasar | ...
    $t->string('payment_reference')->nullable();
    $t->string('status', 20);                         // pending | paid | refunded | failed
    $t->timestamps();
});
```

### 6.4 التحقّق من الترخيص عند التثبيت
قبل تنفيذ `ThemeInstaller`:
1. إن كان `license = free` → اسمح بالتثبيت.
2. إن كان `license = commercial` → تحقّق من وجود `theme_purchases.status = paid` لنفس `tenant_id + module_id`.
3. إن كان `license = subscription` → تحقّق من `subscriptions.status = active` وتاريخ الانتهاء.
4. عند الفشل → وجّه إلى صفحة الدفع (خارج نطاق هذا البرومبت، اتركها TODO).

### 6.5 بوابة الدفع
- **MVP**: لا تربط بوابة دفع فعلية. سجّل العملية كـ `pending` واسمح لأدمن المنصة بقبولها يدوياً.
- **بعد MVP**: تكامل مع `Moyasar` أو `Stripe` عبر Adapter يحترم `App\Contracts\PaymentGateway`.

---

## 7) الصلاحيات (Spatie Permissions)

أضف في `database/seeders/ThemeStoreSeeder.php`:

| Permission | Scope | يُعطى لـ |
|---|---|---|
| `themes.browse` | tenant | أدوار الجهة الإدارية |
| `themes.install` | tenant | `Tenant Admin` فقط |
| `themes.uninstall` | tenant | `Tenant Admin` فقط |
| `themes.purchase` | tenant | `Tenant Admin` + أدوار الفوترة |
| `themes.author` | system | شركاء معتمدون فقط |
| `themes.review` | system | فريق المراجعة في المنصة |
| `themes.publish` | system | Super Admin |

> **ملاحظة:** اتبع اتفاقية التسمية في `docs/tech-standards/permissions-naming.md`. الصلاحية التي بـ scope `system` لا تُعطى عبر أدوار الجهة.

---

## 8) واجهات جانبية يجب إضافتها

### 8.1 سايدبار أدمن الجهة
في `config/sidebar.php`، تحت قسم "الموقع العام" الذي أُنشئ في الجلسة السابقة:
```php
[
    'title'       => __('sidebar.theme_store'),
    'icon'        => 'hugeicons-store-04',    // تحقق من وجود الاسم
    'route'       => 'store.themes.index',
    'permissions' => ['themes.browse'],
],
```

### 8.2 تبويب في صفحة إعدادات صفحة الهبوط
أضف زر "تصفح الثيمات" في شريط الـ Editor العلوي، يفتح `/store/themes` في تبويب جديد.

### 8.3 ترجمات
أضف في `resources/lang/ar/app.php` + `en/app.php`:
- `sidebar.theme_store`
- `themes.browse_all`, `themes.installed`, `themes.install`, `themes.uninstall`
- `themes.confirm_replace_draft`
- `themes.upgrade_available`
- `themes.free`, `themes.commercial`, `themes.subscription`

---

## 9) نطاق الـ MVP (Phase 1) مقابل ما يُؤجَّل

### ✅ ضمن MVP
- توحيد الجدول `modules` بإضافة `type`.
- Manifest Validator + Installer + Uninstaller.
- API الشركاء (5 endpoints).
- صفحات المتجر الثلاث + المعاينة.
- تثبيت الثيمات المجانية (بدون دفع).
- ثيم تجريبي واحد مجاني في الـ Seeder.
- صلاحيات + سايدبار + ترجمات.

### ⏸ يؤجَّل بعد MVP
- تكامل بوابة الدفع.
- محلّل الـ diff عند الترقية.
- نظام المراجعات والنجوم من الجهات (`ModuleReview` موجود — يُفعَّل لاحقاً للثيمات).
- Theme Studio الكامل داخل okta-partners (يمكن في البداية قبول manifest عبر API فقط).
- دعم البلوكات المخصصة من الشركاء (Advanced).

---

## 10) معايير القبول (Acceptance Criteria)

على Claude أن يُنجز الميزة مقبولة **فقط** عندما:

1. تثبيت Laravel migrations بدون أخطاء على قاعدة landlord والجهة.
2. تشغيل `ThemeStoreSeeder` ينتج ثيم `starter-free` مرئي في `/store/themes`.
3. الضغط على "تثبيت" يملأ `TenantLandingPage.draft_blocks` ويحوّل إلى المحرر.
4. المحرر يعرض بلوكات الثيم بشكل قابل للتعديل تماماً كما لو أنشأها المستخدم يدوياً.
5. النشر يعمل والصفحة العامة (`/{tenantSlug}`) تعرض الثيم الجديد.
6. إلغاء التثبيت يرجع المسوّدة إلى فارغة ويحتفظ بالمنشور.
7. صفحة المعاينة `/store/themes/{slug}/preview` تعمل بدون تسجيل دخول.
8. استدعاء `POST /api/partners/themes` بتوكن صالح ينشئ ثيماً بحالة `Draft`.
9. محاولة شريك تعديل ثيم شريك آخر ترجع 403.
10. كل النصوص مترجمة للعربية والإنجليزية.
11. لا وجود لأي `max-w-*` في الكود الجديد.
12. لا وجود لأي مكون `<flux-*>` في الكود الجديد.

---

## 11) تعليمات سلوكية للجلسة الجديدة

- **اسأل قبل التنفيذ** إن كان أي قرار أعلاه غير واضح.
- **لا تبدأ بكتابة كود** قبل مراجعة الأصول المذكورة في §1 والتأكد من مطابقة أسماء الملفات.
- **التزم بالخطوات المذكورة في §5.5** ولا تقفز بينها.
- **اكتب اختبارات وحدة** لـ `ThemeManifestValidator` و`ThemeInstaller` قبل الواجهات.
- **لا تعدّل `BlockRegistry`** — فقط استهلكه.
- **لا تعدّل `ModuleStatus`** — فقط أعد استخدامه.
- **عند كل ميلستون صغير، Commit + Push** إلى الفرع المخصص للجلسة.

---

## 12) مراجع سريعة

- ملف هذا البرومبت: `docs/prompts/build-theme-store-marketplace.md`
- نتاج الجلسة السابقة (Landing Builder) — تحقق من:
  - `app/LandingBuilder/`
  - `app/Livewire/TenantLanding/Editor.php`
  - `resources/views/livewire/tenant-landing/editor.blade.php`
  - `routes/landing.php`
- وثيقة الشركاء: `docs/partner-platform.md`
- معايير: `docs/tech-standards/permissions-naming.md`، `docs/tech-standards/design-standards.md`
- تعددية: `docs/roles-and-entities/tenants.md`

> **تذكير أخير:** هذا الملف هو "عقد" لجلسة جديدة. أي انحراف عنه — خصوصاً إدخال تقنيات خارج Laravel+Livewire، أو استخدام Flux، أو استخدام `max-w-*` — مرفوض.

