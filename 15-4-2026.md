# خطة تطوير: Multi-Tenant Management Platform + Reporting Dashboard

## الهدف
تحويل النظام الحالي إلى منصة SaaS متعددة المستأجرين (Multi-Tenant) مع لوحة إدارة احترافية، وإنشاء عملاء جدد تلقائياً بدون تدخل بشري، ولوحة تقارير أداء لكل عميل.

---

## البنية التحتية الحالية (ما وجدناه)

| المكون | التفاصيل |
|---|---|
| Chatwoot | Docker-based على `/home/chatwoot/` |
| PostgreSQL | `root-postgres-1` (pgvector:pg16) → port 5432 |
| DB الحالية | `chatwoot_production` — user: `postgres` — password: `Chatwoot@2026` |
| Redis | `root-redis-1` → port 6379 |
| تطبيقنا | Flask على `/root/aichatbot/AIChatBot/app7.py` |
| DB الحالية للتطبيق | SQLite (`clinic_data.db`) → سيتم الترحيل |

---

## القرارات المعمارية النهائية

| الموضوع | القرار |
|---|---|
| قاعدة البيانات | PostgreSQL الموجودة + إنشاء DB جديدة `aichatbot_platform` |
| نمط العزل | Schema-per-Tenant (`tenant_<id>` لكل عميل) |
| إنشاء بوت تيليجرام | Telethon → BotFather automation |
| دمج Chatwoot | Hybrid: Super-Admin Portal + Chatwoot per-client |
| Admin Portal | Flask + Jinja2 custom portal (يستبدل Flask-Admin البسيط) |
| Metrics Storage | PostgreSQL (يستبدل JSON files) |

---

## User Review Required

> [!IMPORTANT]
> **قبل البدء في التنفيذ:** نحتاج رقم هاتفك لتسجيل جلسة Telethon (لإنشاء بوتات تيليجرام تلقائياً). هذا يتم مرة واحدة فقط ثم يُحفظ في ملف `.session`. الرقم لن يُخزَّن في الكود.

> [!WARNING]
> **ترحيل قاعدة البيانات:** سيتم إنشاء `aichatbot_platform` DB داخل نفس حاوية Docker الخاصة بـ Chatwoot. بيانات Chatwoot لن تتأثر. سيتم إنشاء قاعدة بيانات **إضافية** فقط.

> [!NOTE]
> **Subscription Plans:** سنبني الهيكل لـ 3 خطط (`Basic / Pro / Enterprise`) مع إمكانية تخصيص كل خطة لاحقاً. المزايا والحدود ستُملأ بعد الموافقة على الخطة.

---

## نموذج البيانات (Data Model)

### Schema الرئيسي: `platform`

#### جدول `platform.tenants` (العملاء)
```sql
id                    UUID PRIMARY KEY
client_id             TEXT UNIQUE          -- "dentix_clinic_01"
display_name          TEXT                 -- "عيادة د. أحمد"
domain_type           TEXT                 -- clinic | restaurant | real_estate | ...
domain_name           TEXT                 -- "عيادة الشرباجي للأسنان"
country               TEXT                 -- SA | EG | AE | ...
phone_1               TEXT
phone_2               TEXT
email                 TEXT UNIQUE
-- Telegram (auto-provisioned)
telegram_bot_name     TEXT
telegram_bot_token    TEXT
telegram_bot_username TEXT
-- Chatwoot (auto-provisioned)  
chatwoot_account_id   INT
chatwoot_inbox_id     INT
chatwoot_agent_token  TEXT                 -- API token للعميل نفسه
-- Status
status                TEXT DEFAULT 'pending'  -- pending | active | suspended | cancelled
ai_usage_limit        INT DEFAULT 1000     -- عدد رسائل AI الشهرية
ai_usage_current      INT DEFAULT 0
-- Timestamps
created_at            TIMESTAMPTZ DEFAULT NOW()
activated_at          TIMESTAMPTZ
```

#### جدول `platform.subscription_plans` (خطط الاشتراك)
```sql
id                    SERIAL PRIMARY KEY
plan_name             TEXT UNIQUE          -- basic | pro | enterprise
price_monthly         DECIMAL(10,2)
price_yearly          DECIMAL(10,2)
ai_message_limit      INT                  -- حد رسائل AI شهرياً
features              JSONB                -- {"whatsapp": true, "telegram": true, ...}
is_active             BOOL DEFAULT TRUE
created_at            TIMESTAMPTZ DEFAULT NOW()
```

#### جدول `platform.subscriptions` (اشتراكات العملاء)
```sql
id                    UUID PRIMARY KEY
tenant_id             UUID REFERENCES platform.tenants(id)
plan_id               INT REFERENCES platform.subscription_plans(id)
status                TEXT    -- trial | active | past_due | cancelled
amount_due            DECIMAL(10,2)
amount_paid           DECIMAL(10,2)
currency              TEXT DEFAULT 'USD'
billing_cycle         TEXT    -- monthly | yearly
period_start          DATE
period_end            DATE
trial_ends_at         DATE
coupon_code           TEXT
discount_percent      INT DEFAULT 0
payment_method        TEXT
payment_reference     TEXT
created_at            TIMESTAMPTZ DEFAULT NOW()
updated_at            TIMESTAMPTZ DEFAULT NOW()
```

#### جدول `platform.metrics` (مقاييس الأداء)
```sql
id                    BIGSERIAL PRIMARY KEY
tenant_id             UUID REFERENCES platform.tenants(id)
date                  DATE
-- رسائل
user_messages_count   INT DEFAULT 0        -- رسائل المستخدمين الواردة
ai_messages_count     INT DEFAULT 0        -- رسائل الـ AI الصادرة
human_messages_count  INT DEFAULT 0        -- رسائل الموظفين البشريين الصادرة
-- محادثات
conversations_opened  INT DEFAULT 0        -- إجمالي المحادثات
conversations_ai_resolved   INT DEFAULT 0  -- حُلّت بواسطة AI
conversations_human_resolved INT DEFAULT 0 -- حُلّت بواسطة موظف
conversations_pending INT DEFAULT 0        -- قيد الانتظار
-- مصدر القناة
telegram_sessions     INT DEFAULT 0
whatsapp_sessions     INT DEFAULT 0
-- تحديث
updated_at            TIMESTAMPTZ DEFAULT NOW()
UNIQUE(tenant_id, date)
```

#### جدول `platform.audit_log` (سجل العمليات)
```sql
id                    BIGSERIAL PRIMARY KEY
tenant_id             UUID
action                TEXT    -- "tenant_created" | "bot_provisioned" | ...
actor                 TEXT    -- "system" | "admin"
details               JSONB
created_at            TIMESTAMPTZ DEFAULT NOW()
```

### Schema per-Tenant: `tenant_<client_id>`
كل عميل له schema خاص به يحتوي على:
```sql
tenant_alshurbaji.appointments     -- المواعيد
tenant_alshurbaji.inventory        -- المخزون
tenant_alshurbaji.conversations    -- سجل المحادثات التفصيلي
tenant_alshurbaji.custom_data      -- بيانات مخصصة حسب نوع النشاط
```

---

## مراحل التنفيذ

### المرحلة ١: Database Migration & Foundation
**الهدف:** ترحيل من SQLite إلى PostgreSQL وإنشاء نموذج البيانات

#### [NEW] `db_platform.py`
- الاتصال بـ PostgreSQL عبر `psycopg2`
- Connection pool مع `psycopg2.pool`
- دوال إنشاء الـ schemas تلقائياً
- `create_tenant_schema(tenant_id)` — ينشئ schema جديد لكل عميل

#### [MODIFY] `database.py`
- إزالة SQLite
- استخدام PostgreSQL عبر `db_platform.py`

#### [NEW] `migrations/001_initial_platform.sql`
- SQL لإنشاء كل الـ schemas والجداول

---

### المرحلة ٢: Tenant Provisioning System (نظام تجهيز العملاء تلقائياً)

هذا النظام هو قلب المنصة — عند إنشاء عميل جديد يحدث التالي تلقائياً:

```
[Admin يضغط "إنشاء عميل"]
        ↓
[١. حفظ بيانات العميل في DB بحالة "pending"]
        ↓
[٢. إنشاء Telegram Bot عبر Telethon → BotFather]
        ↓
[٣. تسجيل Webhook للبوت الجديد على سيرفرنا]
        ↓
[٤. إنشاء Chatwoot User + Inbox عبر API]
        ↓
[٥. إنشاء tenant schema في PostgreSQL]
        ↓
[٦. تحديث حالة العميل إلى "active"]
        ↓
[٧. إرسال بريد إلكتروني للعميل بالتفاصيل]
```

#### [NEW] `provisioning/telegram_provisioner.py`
```python
# يستخدم Telethon للتواصل مع BotFather
async def create_telegram_bot(client_name: str, client_id: str) -> dict:
    """
    يرسل لـ BotFather:
    /newbot
    → {bot_display_name}  (مثال: "Alshurbaji Clinic Bot")
    → {bot_username}      (مثال: "alshurbaji_clinic_bot")
    Returns: {"token": "...", "username": "...", "name": "..."}
    """
```

#### [NEW] `provisioning/chatwoot_provisioner.py`
```python
# يستخدم Chatwoot Super-Admin API
def create_chatwoot_account(tenant_data: dict) -> dict:
    """إنشاء User/Agent في Chatwoot باستخدام Super Admin API"""
    
def create_telegram_inbox(account_id: int, bot_token: str, bot_name: str) -> int:
    """إنشاء Inbox مرتبط بالبوت الجديد في Chatwoot"""
    
def setup_webhook(account_id: int, inbox_id: int, webhook_url: str) -> bool:
    """ضبط webhook لإنشاء المحادثات في Chatwoot"""
```

#### [NEW] `provisioning/orchestrator.py`
```python
# ينسق جميع خطوات التجهيز
def provision_new_tenant(tenant_data: dict) -> ProvisionResult:
    # يُنفّذ الخطوات ١-٧ بالترتيب مع rollback عند الفشل
```

---

### المرحلة ٣: Multi-Tenant Business Registry (ديناميكي من DB)

#### [MODIFY] `business_registry.py`
- **حالياً:** بيانات hard-coded في الكود
- **بعد التطوير:** يُحمَّل من PostgreSQL ديناميكياً
- Caching مع TTL (30 ثانية) لتجنب استعلامات مستمرة
- Hot-reload عند إضافة عميل جديد

```python
class DynamicBusinessRegistry:
    def __init__(self):
        self._cache = {}
        self._last_load = 0
        self._ttl = 30  # ثانية
    
    def _load_from_db(self):
        """يُحمّل جميع العملاء النشطين من PostgreSQL"""
    
    def resolve_by_telegram_token(self, token: str) -> Optional[BusinessProfile]:
        """يبحث في Cache أولاً ثم DB"""
```

---

### المرحلة ٤: Metrics System (بدلاً من JSON Files)

#### [MODIFY] `metrics.py`
- **حالياً:** JSON files في `metrics_store/`
- **بعد التطوير:** PostgreSQL `platform.metrics`
- تسجيل تلقائي لكل حدث:
  - رسالة مستخدم واردة → `user_messages_count++`
  - رد AI → `ai_messages_count++`
  - رد موظف بشري → `human_messages_count++`
  - انتهاء محادثة بـ AI → `conversations_ai_resolved++`
  - انتهاء محادثة بموظف → `conversations_human_resolved++`

```python
def record_event(tenant_id: str, event_type: str, channel: str = "telegram"):
    """يُسجّل حدث في جدول platform.metrics (upsert يومي)"""
```

---

### المرحلة ٥: Super-Admin Portal (لوحة الإدارة الكاملة)

#### [MODIFY] `admin_panel.py` (إعادة بناء كاملة)

**الصفحات:**

```
/admin/                    → Dashboard رئيسي (إحصائيات كل العملاء)
/admin/tenants             → قائمة كل العملاء مع حالتهم
/admin/tenants/new         → نموذج إنشاء عميل جديد (+ Provisioning)
/admin/tenants/<id>        → صفحة تفاصيل العميل (تعديل، تعطيل، حذف)
/admin/tenants/<id>/report → تقرير أداء عميل محدد
/admin/metrics             → تقارير الأداء لكل العملاء
/admin/subscriptions       → إدارة الاشتراكات والفواتير
/admin/plans               → إدارة خطط الاشتراك
/admin/audit               → سجل العمليات
```

**تصميم نموذج إنشاء عميل جديد:**
```
┌─ معلومات العميل ──────────────────────┐
│ الاسم الكامل  │ البريد الإلكتروني      │
│ الدولة        │ رقم الهاتف ١           │
│ رقم الهاتف ٢ │ نوع النشاط (dropdown) │
│ اسم النشاط   │ خطة الاشتراك (dropdown)│
├─ إعدادات AI ──────────────────────────┤
│ حد رسائل AI الشهري                   │
│ اللغة الافتراضية للبوت               │
├─ فترة التجربة والخصومات ──────────────┤
│ فترة تجربة مجانية (أيام)             │
│ كود خصم (إن وجد)                     │
└────────────────────────────────────────┘
[✓ إنشاء وتجهيز تلقائياً]
```

**مؤشرات Dashboard الرئيسي:**
```
┌──────────┬──────────┬──────────┬──────────┐
│ العملاء  │ نشطون    │ إجمالي   │ حُلّت   │
│ الكليين  │ اليوم    │ محادثات  │ بـ AI   │
│   47     │   12     │  1,234   │   89%   │
└──────────┴──────────┴──────────┴──────────┘
```

---

### المرحلة ٦: Client Reporting Dashboard

#### لوحة التقارير للعميل (داخل Chatwoot عبر Custom Dashboard)
- Chatwoot يدعم **Custom Tabs** في الـ sidebar عبر iFrame
- سنبني صفحة `/client/<tenant_id>/dashboard` في تطبيقنا
- العميل يراها مدمجة داخل Chatwoot الخاص به

**مقاييس تظهر للعميل:**
```
📊 تقرير الأداء — هذا الشهر
┌─────────────────────────────────────────┐
│ 🤖 محادثات حُلّت بـ AI        │  142  │
│ 👨 محادثات حُلّت بموظف        │   23  │
│ 💬 رسائل أرسلها AI            │  890  │
│ 📨 رسائل أرسلها الموظفون      │  156  │
│ 📥 رسائل واردة من العملاء     │ 1,046 │
│ ⚡ نسبة الأتمتة               │  86%  │
└─────────────────────────────────────────┘
```

#### لوحة Super-Admin: مراقبة كل العملاء
- جدول مقارنة بين جميع العملاء
- رسوم بيانية للأداء الشهري
- تنبيهات عند تجاوز الحدود

---

### المرحلة ٧: App Integration (ربط كل شيء بـ app7.py)

#### [MODIFY] `app7.py`
- استخدام `DynamicBusinessRegistry` بدلاً من القديم
- تسجيل الـ Metrics تلقائياً في كل event handler
- دعم webhook ديناميكي: `/telegram/<bot_token>` يعمل لأي عميل مُسجَّل في DB
- إضافة middleware لفحص حدود الاشتراك (AI usage limit)

#### [MODIFY] `chatwoot_integration.py`
- دعم Multi-tenant: كل عميل له Chatwoot account_id و inbox_id منفصل
- تمرير `(account_id, inbox_id, api_token)` بدلاً من المتغيرات الثابتة

---

## ملخص الملفات الجديدة والمعدّلة

### ملفات جديدة
| الملف | الوظيفة |
|---|---|
| `db_platform.py` | PostgreSQL connection + schema management |
| `migrations/001_initial.sql` | SQL لإنشاء كل الجداول |
| `provisioning/telegram_provisioner.py` | Telethon + BotFather automation |
| `provisioning/chatwoot_provisioner.py` | Chatwoot API: create account + inbox |
| `provisioning/orchestrator.py` | تنسيق خطوات تجهيز العميل الجديد |
| `portal/admin_portal.py` | Super-Admin Portal الكامل (يستبدل admin_panel.py) |
| `portal/templates/` | HTML templates للـ portal |
| `portal/static/` | CSS/JS للـ portal |

### ملفات معدّلة
| الملف | التغيير |
|---|---|
| `admin_panel.py` | إعادة بناء كاملة → يستدعي `portal/` |
| `business_registry.py` | Dynamic loading من PostgreSQL + cache |
| `metrics.py` | PostgreSQL بدلاً من JSON files |
| `chatwoot_integration.py` | Multi-tenant support |
| `app7.py` | Dynamic registry + metrics integration |
| `database.py` | PostgreSQL بدلاً من SQLite |

---

## خطة التحقق (Verification Plan)

### اختبارات تلقائية
1. إنشاء عميل تجريبي → التحقق من إنشاء البوت والـ inbox
2. إرسال رسالة → التحقق من تسجيل الـ metrics
3. Handoff بشري → التحقق من عدّ `human_messages_count`

### تحقق يدوي
1. فتح Admin Portal والتأكد من ظهور كل المقاييس
2. فتح Chatwoot والتأكد من ظهور لوحة التقارير كـ Custom Tab
3. اختبار إنشاء عميل جديد من البداية للنهاية

---

## ترتيب التنفيذ المقترح

```
١. إنشاء DB + Migration SQL       ← أساس كل شيء
٢. db_platform.py                 ← الاتصال بـ PostgreSQL  
٣. Dynamic BusinessRegistry       ← ترحيل تدريجي
٤. Metrics → PostgreSQL           ← يحسن الأداء فوراً
٥. Telegram Provisioner           ← Telethon setup
٦. Chatwoot Provisioner           ← Chatwoot API
٧. Orchestrator                   ← ربط ٥+٦
٨. Super-Admin Portal             ← الواجهة الكاملة
٩. Client Dashboard               ← Reporting
١٠. app7.py integration           ← ربط كل شيء
```
