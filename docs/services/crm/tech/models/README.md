# نماذج بيانات CRM (Models)

النماذج تحت `app/Models/Crm/*` وكلها تمتد `App\Models\Support\BaseModel`
(hashids في الروابط). الجداول بادئتها `crm_*`. المبالغ `decimal(15,2)`،
والعملة الافتراضية `SAR`. أغلب الجداول soft-deletes.

## الحسابات وجهات الاتصال

| النموذج → الجدول | أعمدة رئيسية |
|---|---|
| `Account` → `crm_accounts` | `name`/`name_ar`، `type` (company/individual/government/ngo)، `industry`، `website`، `phone`، `email`، `address`، `city`، `country` (char2=SA)، `logo`، `assigned_to`→users، `status` (active/inactive/suspended)، `created_by` |
| `Contact` → `crm_contacts` | `account_id`، `first_name`/`last_name`، `email`، `phone`/`mobile`، `job_title`، `department`، `is_primary`، `notes`، `created_by` |

## العملاء المحتملون والأنبوب

| النموذج → الجدول | أعمدة رئيسية |
|---|---|
| `Lead` → `crm_leads` | `title`، `account_id`/`contact_id`، `company_name`، `contact_name/email/phone`، `stage_id`→المرحلة، `source`، `priority` (low/medium/high/urgent)، `expected_value`، `currency`، `expected_close_date`، `assigned_to`، `lost_reason`، `converted_at`/`converted_by`، `last_activity_at` (مفهرس) |
| `PipelineStage` → `crm_pipeline_stages` | `name_ar`/`name_en`، `color` (char7)، `order`، `is_active`/`is_won`/`is_lost` |
| `Activity` → `crm_activities` | `lead_id`، `account_id`، `user_id`، `type` (call/email/meeting/note/stage_change/task/whatsapp/other)، `title`، `outcome`، `scheduled_at`، `completed_at`، `duration_minutes` |
| pivot → `crm_activity_contact` | `activity_id`، `contact_id`، unique(الاثنين) |
| `Task` → `crm_tasks` | `lead_id`/`account_id`، `assigned_to`/`created_by`، `title`، `due_date`، `priority` (low/medium/high)، `status` (pending/in_progress/completed/cancelled)، `reminder_at`، `completed_at` |

## العقود والفواتير

| النموذج → الجدول | أعمدة رئيسية |
|---|---|
| `Contract` → `crm_contracts` | `account_id`/`lead_id`، `contract_number` (unique)، `title`، `value`، `currency`، `start/end_date`، `status` (draft/sent/signed/active/expired/cancelled)، `signed_at`، `file_path`؛ **توقيع إلكتروني:** `signature_png` (longText)، `signed_by_name`، `signed_ip` (45)، `signed_user_agent` |
| `ContractTemplate` → `crm_contract_templates` | `name`، `body_html` (longText)، `variables` (json)، `is_active`، `created_by` |
| `Invoice` → `crm_invoices` | `account_id`/`contract_id`، `invoice_number` (unique)، `subtotal`، `tax_rate` (5,2=15)، `tax_amount`، `total`، `currency`، `issue_date`/`due_date`/`paid_at`، `status` (draft/sent/paid/overdue/cancelled) |
| `InvoiceItem` → `crm_invoice_items` | `invoice_id` (cascade)، `description`، `quantity` (10,2)، `unit_price`، `total`، `order` |

## التقويم والمستندات وسجلات الوقت

| النموذج → الجدول | أعمدة رئيسية |
|---|---|
| `CalendarEvent` → `crm_calendar_events` | `title`، `starts_at`/`ends_at`/`all_day`، `type` (meeting/call/visit/reminder/deadline/other)، `color`، `rrule` (varchar64)، `recurrence_until` (date)، `lead_id`/`account_id`/`assigned_to`، `reminder_at`؛ فهارس على `starts_at` و(`assigned_to`,`starts_at`) |
| `Document` → `crm_documents` | `documentable` (nullableMorphs)، `title`، `original_name`، `file_path`، `mime_type`، `size` (bigint)، `extension`، `category`، `folder_id`→المجلّد، `uploaded_by` |
| `DocumentFolder` → `crm_document_folders` | `name`، `parent_id`→ذاتي (cascade، شجرة)، `created_by` |
| `TimeEntry` → `crm_time_entries` | `task_id`/`lead_id`/`account_id`، `user_id`، `started_at`/`ended_at`، `duration_seconds` (uint)، `notes`، `billable`؛ فهرس (`user_id`,`started_at`) |

راجع [الأنابيب والعقود والتقويم والمستندات](../../guide/README.md).
