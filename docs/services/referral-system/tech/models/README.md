# نماذج بيانات نظام الإحالات (Models)

الخدمات تحت `app/Services/ReferralSystem/*`، والنماذج تحت `app/Models/Referrals/*`،
وكل الجداول **landlord** (مستوى المنصّة). المعرّفات UUID.

## الإحالة والأكواد

| النموذج → الجدول | أعمدة رئيسية |
|---|---|
| `ReferralCode` → `referral_codes` | `user_id` (unique FK)، `code` (varchar10 unique)، `is_active`، `clicks_count`، `conversions_count` |
| `Referral` → `referrals` | `referrer_user_id`، `referred_user_id` (**unique** — إحالة واردة واحدة لكل مستخدم)، `referral_code_id`، `parent_referral_id` (self-FK → سلسلة المستويات)، `level` (tinyint، 1)، `status`، `qualification_trigger`، `qualified_at`، `rewarded_at`، وحقول الاحتيال: `ip_address`, `user_agent`, `device_fingerprint`, `otp_verified_at`, `risk_score` (0–100), `flagged_reason`, `metadata` (jsonb)، soft-deletes |

- **status**: `pending / qualified / rewarded / rejected / clawed_back / flagged`.
- **qualification_trigger**: `registration / first_subscription / time_based`.

## المكافآت والمحافظ

| النموذج → الجدول | أعمدة رئيسية |
|---|---|
| `ReferralReward` → `referral_rewards` | `referral_id`، `beneficiary_user_id`، `beneficiary_type` (`referrer`/`referred`)، `reward_type` (`fixed`/`percentage`/`discount`)، `amount` (15,2)، `currency` (def SAR)، `status` (`pending`/`credited`/`clawed_back`)، `credited_at`، `settings_snapshot` (jsonb) |
| `Wallet` → `wallets` | `user_id`، `balance`، `frozen_balance`، `total_earned`، `total_spent`، `currency`، `is_locked`؛ unique(`user_id`,`currency`) |
| `WalletTransaction` → `wallet_transactions` | `wallet_id`، `type`، `source`، `amount`، `balance_before/after`، `reference_type/id` (polymorphic)، `performed_by` |
| `currency_exchange_rates` | `from/to_currency`، `rate` (15,8)، `source`، `valid_from/until` |

## المستويات والأوسمة (Gamification)

| النموذج → الجدول | أعمدة رئيسية |
|---|---|
| `ReferralLevel` → `referral_levels` | `slug` (bronze/silver/gold)، `name_ar/en`، `min_successful_referrals`، `bonus_multiplier` (4,2)، `perks` (jsonb)، `order` |
| `ReferralBadge` → `referral_badges` | `slug`، `name_ar/en`، `criteria` (jsonb)، `tier`، `order` |
| `UserReferralAchievement` → `user_referral_achievements` | `user_id`، `badge_id`، `level_id`، `earned_at` |
| `ReferralSetting` → `referral_settings` | `key` (unique)، `value` (jsonb)، `group` — مخزن KV يقرأ منه محرّك المكافآت كامل إعداداته |

## المؤثّرون (Influencers)

| النموذج → الجدول | أعمدة رئيسية |
|---|---|
| `Influencer` → `influencers` | `code` (12 unique)، معلومات + `commission_type` (fixed/percentage)، `commission_amount/percentage`، `currency`، `clicks/conversions_count`، `total_earned`، حقول بنكية |
| `InfluencerWallet` / `InfluencerWalletTransaction` | يعكسان محفظة/حركة المستخدم، مفتاحهما `influencer_id` |
| `InfluencerConversion` → `influencer_conversions` | `influencer_id`، `referred_user_id`، `status`، `reward_amount`، `ip_address`، `source_url`، `qualified_at`، `rewarded_at` |

راجع [محرّك المكافآت ودورة الحياة ومكافحة الاحتيال](../../guide/README.md).
