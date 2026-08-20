# عقود الحدود — ما الذي يعبر، بأي شكل، وتحت سلطة مَن

> التطبيقات المصغّرة الأصيلة (Dart على الجهاز). `okta-miniapp@50e394e`،
> `okta-app@e49423b`، `okta-web@52e48ee`، `okta-partners@608abc3`. قُرِئت 2026-08-20.
> ملفات مرافقة: [`README.md`](./README.md) · [`flow.md`](./flow.md) ·
> [`data.md`](./data.md) · [`edges.md`](./edges.md) · [`working-notes.md`](./working-notes.md)
> · 🇬🇧 [النسخة الإنجليزية](../miniapp-native/contracts.md)

خمسة حدود تهمّ على المسار الأصيل. ثلاثة منها قفزات شبكة، واثنان سِيَمٌ داخل
العملية تتصرّف كقفزات شبكة لأن اختلاف الإصدارات عبرهما يفشل **على جهاز** لا في CI.

---

## H1 · okta-app ← okta-web · الإطلاق (launch)

| | |
|---|---|
| **المسار** | `POST /api/mobile/app-catalog/{slug}/launch` — `okta-web/routes/api.php:119` |
| **توأم البوابة** | `POST /api/mobile/app-catalog/portal/{slug}/launch` — `:126` |
| **المصادقة** | Sanctum bearer + وسيط `mobile.min-version` — `:117` |
| **الطلب** | `{tenant_id:int, role_id:int}`؛ البوابة: `{tenant_id:int, portal:'student'\|'guardian'}` — `MobileAppCatalogController.php:152-155`، `:106-109` |
| **الاستجابة (native)** | `{mode:'native', url:<signed>, payload_version:<int>}` — `IssueWebviewLaunch.php:139-143`، `:196-200` |
| **متزامن/لا** | متزامن |
| **الفشل** | 404 حين يكون المودول غير نشط، أو `mobile.supported !== true`، أو لا جمهور يطابق الدور — `:167-188`. و426 من `mobile.min-version` لعميل قديم — `routes/api.php:113-115` |

**`payload_version` مشتقّ من طابع `updated_at` لصفّ المودول**
(`IssueWebviewLaunch.php:143`، `:200`)، لا من نصّ إصدار يكتبه الشريك. وهو مُدخَل
مفتاح الكاش على الجهاز. `[مؤكَّد]`

## H2 · okta-app ← okta-web · حزمة المصدر

| | |
|---|---|
| **المسار** | `GET /app/{slug}/miniapp-payload` — `okta-web/routes/app.php:45` |
| **المصادقة** | **توقيع الرابط هو التفويض**. `['web','signed','app.webview','throttle:30,1']` — `routes/app.php:33`. و`app.webview` يعيد ترطيب المستخدم ويؤكّد أن الدور يحمل نطاق البطاقة المطلوب، فلا يفعل الـ controller سوى التجميع — `MiniappPayloadController.php:12-22` |
| **الاستجابة** | JSON، بترويسة `Cache-Control: no-store, private` — `:44-45` |
| **الفشل** | 404 ما لم يكن الوضع المحلول `native` (`:29-31`)؛ و404 من `BundleMiniappSource` عند مدخل خاطئ، أو `lib/` مفقود، أو ملف دخول مفقود، أو تجاوز أي حدّ |

### شكل الحمولة، وانحرافُ اسمٍ يستحقّ المعرفة

الخادم يُصدِر — `BundleMiniappSource.php:114-121`:

```json
{ "package": "...", "entry_file": "main.dart", "entry_function": "main",
  "min_contract": 13, "capabilities": [{"key":"...","reason":"..."}],
  "files": { "main.dart": "<source>", "widgets/card.dart": "<source>" } }
```

والعميل يفكّ — `okta-miniapp/lib/src/bundle/okta_mini_app_bundle.dart:176-191`.
المفاتيح نفسها بصيغة snake_case، مربوطةً على حقول Dart بصيغة camelCase
(`entry_file` ← `entryFile`، `min_contract` ← `minContract`).

**الحقلان اللذان لا يرسلهما الخادم.** يُكتَب `slug` و`payload_version` في الخريطة
**من جهة العميل، من استجابة الإطلاق**، قبل الفكّ مباشرةً —
`okta-app/lib/features/miniapps/bridge/okta_dart_miniapp_host.dart:160-162`:

```dart
map['slug'] = slug;
map['payload_version'] = '${launch.payloadVersion}';
```

وهذا مقصود: كلاهما مُدخَل لمفتاح الكاش، وأخذهما من الإطلاق الموقَّع بدل جسم
الحمولة يجعل المفتاح **مرجعياً بصرف النظر عمّا يقوله الجسم**. `[مؤكَّد]`
`okta_dart_miniapp_host.dart:97-100`

و`payload_version` **عدد صحيح** على السلك من H1، و**نصّ** على الحزمة
(`'${json['payload_version']}'`، `:189`). بالاستيفاء (interpolation) لا بالـ cast
— فتغيّر نوعه على الخادم لن يرمي هنا. `[مؤكَّد]`

## H3 · بوّابة نشر okta-web ← manifest من okta-partners

| | |
|---|---|
| **الشكل** | كتلة `mobile` من manifest المودول: `supported`، `mode`، `runtime`، `minContract`، `entry` (لكل جمهور)، `capabilities[]` |
| **المُنتِج** | `okta-partners` — `VersionEditor.php`، `PersistImportedSurfaces.php`، أداة MCP `set_mobile_surface` (`ControlTools.php:114`) |
| **المستهلِك** | `okta-web/app/Modules/Core/ManifestValidator.php:446-490`، `:817-821`، `:912-958` |
| **مرجع شكل المدخل** | `okta-partners/app/Services/Partners/Manifest/MobileEntryRules.php:36` — ومُعاد تنفيذه كـ regex داخلي في `ManifestValidator.php:818` ومرّة ثالثة في `BundleMiniappSource.php:49` |

توجد ثلاث نسخ من regex مدخل native، في مستودعين:

```
okta-partners  MobileEntryRules.php:36        ^okta_app/native/[A-Za-z0-9_-]+/lib/[A-Za-z0-9._/-]+\.dart$
okta-web       ManifestValidator.php:818      ^okta_app/native/[A-Za-z0-9_-]+/lib/
okta-web       BundleMiniappSource.php:50     ^okta_app/native/([A-Za-z0-9_-]+)/lib/
```

نسختا okta-web فحصُ **بادئة** إضافةً إلى `str_ends_with('.dart')` و
`str_contains('..')` منفصلَين، فهي مكافئة في الأثر لا مطابقة في النصّ. `[مؤكَّد]`
`ManifestValidator.php:817-821`، `BundleMiniappSource.php:49-55`. والثالثة ليست
تكراراً لذاته — فـ `BundleMiniappSource` يخدم أيضاً مودولات مخزَّنة قبل وجود
بوّابة النشر.

## H4 · سِيم الصندوق الرملي · التطبيق المصغّر ↔ المضيف

الحدّ الوحيد الذي يراه كود الشريك فعلاً.

| | |
|---|---|
| **الواجهة** | `package:okta_host/okta_host.dart` ← الصنف الساكن `Okta` — `okta-miniapp/lib/src/host/okta_host_source.dart:74` |
| **الناقل** | ارتباطات `registerBridgeFunc`، كلّها ببادئة `$host*` — `okta_host_plugin.dart:79-141` |
| **صيغة السلك** | **نصوص JSON في الاتجاهين.** `_context` يعيد `$String(jsonEncode(...))` (`:152-154`)؛ و`_apiCall` يفكّ `args[0]` كـ JSON ويعيد ترميز الاستجابة (`:156-172`) |
| **اللاتزامن** | `$Future.wrap(...)` — `:171`، `:215` |
| **الاتجاه** | **من الصندوق الرملي إلى المضيف فقط.** لا يوجد مسار مسجَّل ليُنادي المضيفُ الداخلَ. |

ثلاث نتائج تُشكّل كل قدرة:

1. **القارئ المستمرّ لا يستطيع الدفع.** `Okta.toolKeyReads()` هو **سحب (drain)**:
   المضيف يخزّن مؤقتاً، والتطبيق المصغّر يأتي ويأخذ كل ما تجمّع منذ النداء
   السابق. هذا **مفروض** بالارتباط أحادي الاتجاه، لا مُختار. `[مؤكَّد]`
   `okta-miniapp/CLAUDE.md` §العقد 12، وغياب أي تسجيل من المضيف إلى الصندوق في
   `okta_host_plugin.dart:79-144`.
2. **لا شيء مُنمَّط يعبر.** الكائن المبنيّ داخل تطبيق مصغّر قيمةٌ مفسَّرة لا
   نسخة أصيلة. كل ما يعبر هو JSON.
3. **الرفض قيمة عادية تُعاد، لا استثناء يُرمى.** انظر الجدول في
   `edges.md` §مفردات الرفض.

## H5 · سِيم الإصدار · ملف okta-app الثنائي ↔ حزمة الشريك

ليس قفزة شبكة، وهو أغلى حدّ يُخطَأ فيه.

`okta_kit` و`okta_host` و`okta_glass` و`okta_motion` **مترجَمة داخل ملف okta-app
الثنائي**، لا داخل حزمة الشريك
(`okta-miniapp/lib/src/injected_sources.dart:41-46`). ولذلك:

- إصلاحٌ في مكتبة محقونة لا يصل المستخدم إلا بتثبيت **بناء okta-app جديد**.
  إعادة نشر التطبيق المصغّر لا تفعل شيئاً. `[مؤكَّد]`
  `okta-miniapp/CLAUDE.md` §"A fix in an INJECTED library ships with okta-app"
- والشريك يترجم مقابل **commit مثبَّت** من `okta_miniapp`
  (`pubspec.yaml:18-22`)، ويجب أن يطابق زمن التشغيل الذي يحمله الملف الثنائي
  المشحون — وإلا بطل ضمان CI في الاتجاه الوحيد الذي يهمّ.
- **و okta-app يثبّته أيضاً** (`okta-app/pubspec.yaml`، `ref:`). فإضافة رمز إلى
  زمن التشغيل ثم الإشارة إليه من okta-app في النَفَس نفسه **لا تُبنى** حتى
  يتحرّك ذلك الـ ref.

**وهكذا يبدو حين يعضّ**، لأن الرسالة تسمّي مساراً لا سبباً:

```
lib/features/miniapps/bridge/okta_dart_miniapp_host.dart:83:3:
  Error: Undefined name 'OktaMiniAppLog'.
 - 'OktaMiniAppException' is from 'package:okta_miniapp/src/errors.dart'
   ('…/Pub/Cache/git/okta-miniapp-50e394e…/lib/src/errors.dart')
```

الـ SHA داخل مسار الـ pub-cache هو الجواب: إنه الـ ref الذي يثبّت عليه okta-app،
والرمز المضاف بعده **لا يوجد** لهذا البناء. اقرأ ذلك المسار قبل أن تقرأ الكود —
ثم ارفع `ref:` في `okta-app/pubspec.yaml` **وزوج `ref`/`resolved-ref` المقابل في
`pubspec.lock`**، ثم `flutter pub get`.

**رقمان، وظيفتان مختلفتان، ويُخلَط بينهما باستمرار:**

| | `oktaHostContractVersion` | `oktaMiniAppRuntimeSignature` |
|---|---|---|
| معرَّف في | `okta-miniapp/lib/src/host/okta_host_delegate.dart:228` (= **17**) | `okta-miniapp/lib/src/runtime_info.dart:28` |
| هو | عدد صحيح يُرفَع يدوياً | نصّ سلسلة الأدوات + بصمة كل مصدر محقون |
| يبوّب | **القبول.** `minContract <= hostContract` وإلا رفض التشغيل — `okta_mini_app_bundle.dart:154-163` | **لا شيء.** يفتح مفتاح كاش الترجمة فقط |
| يتحرّك حين | يرفعه أحد | تتغيّر أي بايت في أي مصدر محقون |

`oktaMiniAppRuntimeSignature` **لا يبوّب القبول**، والاعتماد عليه لذلك هو الطريق
الموثّق الذي أخطأ من قبل: نزلت `OktaTabs.bottomBar` بعد أربعة أيام من شحن العقد 7
بلا رفع، وأعلن okta-hdor `minContract: 7`، فمرّ كل جهاز على العقد 7 من البوّابة
ثم مات بـ `CompileError: Cannot find static method OktaTabs.bottomBar`. `[مؤكَّد]`
`okta-miniapp/lib/src/host/okta_host_delegate.dart:50-57`

ويعيد okta-app تصدير الرقمين معاً لتقارير الأخطاء، لأن العقد وحده لا يعرّف
البناء — فمضيفان قد يقولان 17 وأحدهما وحده يحمل الإصلاح. `[مؤكَّد]`
`okta_dart_miniapp_host.dart:47`، `:60`

---

## أين تنحرف الأسماء عبر السِيَم

| المفهوم | okta-partners | okta-web | السلك | okta-miniapp / okta-app |
|---|---|---|---|---|
| حدّ العقد | `min_contract` (في `mobile_config`) | `mobile.minContract` (manifest)، `min_contract` (الحمولة) | `min_contract` | `minContract` |
| مسار الدخول | `mobile_entry` (لكل جمهور) | `entry` (الكتلة المحلولة) | — (يُحلّ في الخادم) | `entryFile` (نسبةً إلى `lib/`) |
| الحزمة | — | `package` (يُقرأ من `pubspec.yaml` للشريك) | `package` | `package` |
| الإصدار | — | `payload_version` (من `updated_at`) | `payload_version` (عدد) | `payloadVersion` (**نصّ**) |
| الصلاحية | `capabilities[].{key,reason}` | نفسه | نفسه | `OktaDeclaredCapability{key,reason}` ← النوع البوّابي `MiniAppDeclaredCapability` |

الصفّ الأخير يعبر سِيماً إضافياً **داخل** okta-app: لا يجوز ذكر
`OktaDeclaredCapability` خارج `lib/features/miniapps/bridge/`، فيُترجَم مرّةً
واحدة إلى نوع بوّابي بسيط تتكلّمه ورقة الموافقة وشاشة الصلاحيات وبطاقة الكتالوج
جميعاً. `[مؤكَّد]` `okta_dart_miniapp_host.dart:168-176`. وهذا الحصر **مفروض
آلياً** — انظر `edges.md` §حدّ الاستيراد.
