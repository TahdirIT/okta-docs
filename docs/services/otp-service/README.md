# خدمة OTP (رموز التحقق)

توثيق خدمة **رموز التحقق لمرة واحدة (OTP)** في `okta-web`: إنشاء رمز رقمي
قصير العمر، إرساله عبر سلسلة قنوات مع fallback، والتحقق منه. تقابلها
`App\Services\Otp\*`.

> **بنية هذا المجلد:** الملف الحالي هو **المرجع المطابق للتنفيذ (as-built)**.
> ملفات [`guide/`](./guide/README.md) **مفاهيمية** (تصف نموذجاً مثالياً) وتختلف
> عن الواقع في نقاط مذكورة أدناه — تُقرأ كخلفية تصميمية لا كواقع.

## المكوّنات الفعلية

- `App\Services\Otp\OtpService` — النواة: `create()` (يولّد الرمز ويكتب صفّاً في
  جدول `otps`) و`verify()` (مطابقة + عدّ المحاولات) و`search()` (سرد إداري).
- `App\Services\Otp\OtpDispatcher` — مدخل الإرسال: ينادي `OtpService::create()`
  ثم يُدرِج `App\Jobs\SendOtpJob` بسلسلة القنوات المحلولة.
- `App\Services\Otp\TenantOtpNotificationProvider` — يحاول توجيه OTP عبر تطبيق
  الإشعارات المثبَّت للجهة قبل قنوات المنصة الافتراضية.
- `App\Jobs\SendOtpJob` — يتكرّر على القنوات فعلياً مع الـ fallback.
- النموذج `App\Models\Otp` (جدول `otps`, SoftDeletes) — الحقول: `purpose`,
  `delivery_method`, `recipient`, `provider`, `code_hash`, `code_length`,
  `attempts`, `max_attempts`, `expires_at`, `verified_at`, `last_sent_at`, `meta`.

## القنوات والـ fallback

الترتيب الافتراضي: **WhatsApp → SMS → Email** (مُعرَّف في
`Comms\BridgeSettings::DEFAULT_OTP_CHANNELS`، قابل للتخصيص عبر PlatformSetting
`comms.otp.channels`). التنفيذ في `SendOtpJob`:

- **WhatsApp** عبر Okta Connect (`POST {base}/api/v1/admin/messages/otp`) — يُسقَط إن كان Connect مُعطّلاً.
- **SMS** عبر **Taqnyat** (`api.taqnyat.sa/v1/messages`) — يُسقَط إن غابت الاعتمادات.
- **Email** عبر SMTP (من `PlatformSetting`) — يُضاف دائماً كخيار أخير.

لا يوجد `config/otp.php`؛ الإعدادات في قاعدة البيانات عبر `PlatformSetting`
(`comms.*`, `notifications.taqnyat.*`, `notifications.smtp.*`).

## التوليد والتخزين والأمان

- رمز **رقمي** بطول افتراضي **6** (محصور 4–10)، يُولَّد بـ `random_int`.
- يُخزَّن **مُجزَّأً (bcrypt)** في `code_hash` — لا يُحفظ الرمز الصريح أبداً.
- **TTL** افتراضي **300 ثانية** (`expires_at`)، والـ route يحدّ `ttl_seconds` بـ 1800.
- **أقصى محاولات** = **5** لكل صفّ (`max_attempts`)، يُفرَض في `verify()`.
- **throttle** على المسار `throttle:10,1` (10/دقيقة على `/otp/*`).
- **رسائل خطأ عامة** لا تكشف وجود الوجهة (expired/invalid تُدمَج).

## أين يُستخدَم

- **تسجيل الجهة** — تحقق جوال/بريد المالك (`RegisterTenant`)؛ ملاحظة: التحقق
  حالياً **غير مفروض** لإكمال التسجيل.
- **نظام الإحالات** — `RequestReferralOtp` / `VerifyReferralOtp`.
- **لا يوجد** OTP لتسجيل الدخول ولا لحذف الحساب.

## فروق جوهرية عن الدليل المفاهيمي

- **لا أحداث (Events):** لا توجد `OtpSent`/`OtpVerified` — يُكتفى بـ `Log` (خلافاً لـ [`guide/05-events.md`](./guide/05-events.md)).
- **لا خطوة Consume منفصلة:** `verify()` يختم `verified_at` مباشرةً (خلافاً لـ [`guide/03-otp-flow.md`](./guide/03-otp-flow.md)).
- **قيد fallback:** `comms.otp.throttle_seconds` (60ث) مُعرَّف لكنه **غير مُطبَّق** في مسار الإرسال حالياً (فجوة).

## الدليل المفاهيمي (خلفية)

- [نظرة عامة](./guide/01-overview.md) · [هدف الميزة](./guide/02-feature-goal.md)
- [تدفّق OTP](./guide/03-otp-flow.md) · [الأمان والحدود](./guide/04-security-and-limits.md) · [الأحداث](./guide/05-events.md)
