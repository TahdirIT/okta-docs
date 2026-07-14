# نظام الإحالات (Referral System)

برنامج إحالات كامل في `okta-web`: أكواد إحالة، مكافآت متعدّدة المستويات، محافظ
وعملات، مؤثّرون (influencers)، تلعيب (badges/gamification)، ومنظومة مكافحة
احتيال. المصدر: `app/Services/ReferralSystem/*` و`routes/referrals.php`.

> توثيق مبدئي مبنيّ على جرد الكود (`app/Services/ReferralSystem/*`)؛ يُعمّق لاحقاً
> بالتفاصيل الدقيقة عند الحاجة.

## المكوّنات (`app/Services/ReferralSystem/*`)

- `Codes/` — إنشاء/حلّ أكواد الإحالة وربطها بالمستخدم/الجهة.
- `Rewards/` — احتساب المكافآت متعدّدة المستويات وصرفها.
- `Wallets/` — محافظ الرصيد والعملات وحركاتها.
- `Influencers/` — حسابات المؤثّرين وروابطهم وتتبّع أدائهم.
- `Gamification/` — الأوسمة/النقاط ومعايير منحها.
- `Security/` — مكافحة الاحتيال: درجة مخاطرة، تحقق OTP، منع الإحالة الذاتية، حدود المعدّل.
- `Lifecycle/` — وظائف دورة الحياة (jobs) لنضج المكافآت والاعتمادات.
- `Reports/` · `Settings/` — تقارير البرنامج وإعداداته.

## المسارات (`routes/referrals.php`)

- **مستخدم**: `/referrals`، `/referrals/wallet`، `/referrals/history`، `/referrals/badges` (Livewire `Referrals/*`).
- **إدارة**: `/admin/referrals/{settings,approvals,wallets,badges,currencies,influencers}` (Livewire `Admin/Referrals/*`).

## ملاحظات للتوثيق العميق لاحقاً

- التقاط كود الإحالة يتم عبر middleware `referral.capture` (`CaptureReferralCode`).
- التحقق من الإحالة يستخدم OTP (`ReferralSystem\Security\{RequestReferralOtp,VerifyReferralOtp}`)
  الذي يختم `otp_verified_at` على الإحالة — راجع [otp-service](../otp-service/README.md).
- الأنماط المطلوب توثيقها: بنية المكافآت متعدّدة المستويات، نموذج المحفظة/العملة،
  معايير درجة المخاطرة، ودورة حياة نضج المكافأة.
