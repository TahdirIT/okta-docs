# نظام الإحالات (Referral System)

برنامج إحالات على مستوى المنصّة (landlord): أكواد إحالة، مكافآت متعدّدة المستويات،
محافظ وعملات، مؤثّرون (influencers)، تلعيب (مستويات/أوسمة)، ومنظومة مكافحة احتيال.
الخدمات تحت `app/Services/ReferralSystem/*` والنماذج تحت `app/Models/Referrals/*`.

## المحتوى

- [الدليل — محرّك المكافآت ودورة الحياة ومكافحة الاحتيال](./guide/README.md)
- [الطبقة التقنية](./tech/README.md) — [نماذج البيانات](./tech/models/README.md) · [وظائف الخدمة](./tech/service-functions/README.md)

## المكوّنات بإيجاز

| المجال | المسؤولية |
|---|---|
| **Codes** | كود إحالة فريد لكل مستخدم؛ التقاط `?ref=` عبر middleware `referral.capture`. |
| **Rewards** | احتساب وصرف المكافآت متعدّدة المستويات (idempotent) بلقطة إعدادات. |
| **Wallets** | أرصدة/تجميد/تحويل عملات + حركات (`wallets` / `wallet_transactions`). |
| **Influencers** | حسابات المؤثّرين، نقراتهم/تحويلاتهم، محافظهم وعمولاتهم. |
| **Gamification** | مستويات (`referral_levels`) وأوسمة (`referral_badges`) وإنجازات المستخدم. |
| **Security** | درجة مخاطرة، كشف الإحالة الذاتية، حدود IP/يومية، OTP، وسم المشبوه. |
| **Lifecycle** | تأهيل زمني، إنهاء الإحالات الراكدة، تقرير يومي (أوامر مجدولة). |
| **Settings** | مخزن KV (`referral_settings`) يقرأ منه محرّك المكافآت كامل إعداداته. |

## المسارات (`routes/referrals.php`)

- **مستخدم** (`/referrals`): `Dashboard` · `WalletPage` · `HistoryPage` · `BadgesPage`.
- **إدارة** (`/admin/referrals`، كلٌّ بصلاحية `permission:*`): `Dashboard` ·
  `SettingsPage` · `ApprovalsPage` · `WalletsManagerPage` · `BadgesManagerPage` ·
  `CurrenciesPage` · `InfluencersIndex`/`InfluencerShow`.
