# الحزم الأساسية في okta-web

مرجع سريع لأهم الحزم التي يقوم عليها `okta-web`. المصدر الفعلي دائماً هو
`okta-web/composer.json` و `okta-web/package.json` — حدّث هذا الملف عند تغيّرهما.

## حزم PHP (composer)

- `spatie/laravel-multitenancy` — تعدد المستأجرين (Tenants): تحديد المستأجر الحالي، والاستعلامات/الطوابير/الأوامر الواعية بالمستأجر.
- `spatie/laravel-permission` — الأدوار والصلاحيات (RBAC) على مستوى قاعدة البيانات مع دعم الـ guards وBlade directives.
- `nwidart/laravel-modules` — تنظيم التطبيق كوحدات (Modular Monolith)؛ وهي الآلية التي تُستضاف بها التطبيقات المثبَّتة (embedded) ككود داخل `Modules/`.
- `mhmiton/laravel-modules-livewire` — ربط مكونات Livewire بالوحدات.
- `livewire/livewire` + `wire-elements/modal` — الواجهات التفاعلية والنوافذ المنبثقة.
- `laravel/fortify` — المصادقة والتحقق بخطوتين.
- `laravel/sanctum` — التوكنات (تسجيل دخول تطبيق الجوال وواجهات `/api/mobile/*`).
- `spatie/laravel-activitylog` — سجل النشاط.
- `laravel/pulse` — المراقبة (مع بطاقات مخصصة لنشاط الشركاء).
- `maatwebsite/excel`، `barryvdh/laravel-dompdf`، `mpdf/mpdf` — التصدير والاستيراد وطباعة PDF.
- `firebase/php-jwt` — يُستخدم في `okta-partners` لتوقيع JWT (وليس في okta-web نفسه؛ التوقيعات في okta-web هاند-رولد HS256).

## حزم JavaScript (`package.json`)

### Dependencies

- `@tailwindcss/vite` — تكامل Tailwind CSS v4 مع Vite.
- `autoprefixer` — إضافة بادئات المتصفحات لقواعد CSS تلقائياً.
- `axios` — عميل HTTP للمتصفح.
- `concurrently` — تشغيل عدة أوامر تطوير معاً (`composer run dev`).
- `driver.js` — الجولات التعريفية داخل الواجهة (product tours).
- `flatpickr` — منتقي التاريخ/الوقت.
- `laravel-vite-plugin` — تكامل Vite الرسمي مع Laravel.
- `sortablejs` — السحب والإفلات (إعادة ترتيب القوائم).
- `vite` — أداة البناء وخادم التطوير.

### Optional dependencies

- `@rollup/rollup-linux-x64-gnu`، `@tailwindcss/oxide-linux-x64-gnu`، `lightningcss-linux-x64-gnu` — ملفات ثنائية أصلية لتسريع البناء على Linux x64.

### Dev dependencies

- `tailwindcss` — إطار CSS الأساسي للواجهة.
