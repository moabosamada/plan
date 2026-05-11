# دليل Lago الشامل مع ChatzyAI — الجزء الأول

> المصدر: التوثيق الرسمي https://docs.getlago.com

---

## الفهرس
1. ما هو Lago؟
2. المفاهيم الأساسية
3. إعداد لوحة التحكم
4. إدارة العملاء (Customers)
5. إنشاء الخطط (Plans)
6. مقاييس الاستخدام (Billable Metrics)
7. الرسوم والشحن (Charges)

---

## 1. ما هو Lago؟

Lago هو محرك فوترة مفتوح المصدر (Open-Source Billing Engine) مصمم للتعامل مع:
- **الفوترة القائمة على الاستخدام** (Usage-Based Billing)
- **الاشتراكات الثابتة** (Fixed Subscriptions)
- **النماذج الهجينة** (Hybrid: ثابت + استخدام)

### لماذا Lago بدلاً من Stripe Billing؟
| الميزة | Lago | Stripe Billing |
|---|---|---|
| مفتوح المصدر | ✅ | ❌ |
| استضافة ذاتية | ✅ | ❌ |
| تحكم كامل بالبيانات | ✅ | ❌ |
| نماذج تسعير معقدة | ✅ متقدم جداً | محدود |
| التكلفة | مجاني (Self-hosted) | 0.7% من كل فاتورة |

### كيف يعمل مع ChatzyAI؟
```
المستخدم → ChatzyAI → Lago API → إنشاء فاتورة → Stripe/GoCardless/Adyen → تحصيل الدفع
                ↑                        ↓
        Webhook ← ← ← ← ← ← إشعار الدفع
```

**التدفق الكامل:**
1. المستخدم يسجل في ChatzyAI → يتم إنشاء **Customer** في Lago تلقائياً
2. المستخدم يختار خطة Pro → ChatzyAI يطلب **Payment Link** من Lago
3. المستخدم يدفع عبر Stripe → Lago يستلم الدفع ويُنشئ **فاتورة**
4. Lago يرسل **Webhook** لـ ChatzyAI → يتم تفعيل الاشتراك

---

## 2. المفاهيم الأساسية

### هيكل Lago
```
Organization (المؤسسة)
├── Customers (العملاء)
│   ├── Subscriptions (الاشتراكات)
│   ├── Wallets (المحافظ / الرصيد المسبق)
│   └── Invoices (الفواتير)
├── Plans (الخطط)
│   ├── Fixed Charges (رسوم ثابتة)
│   └── Usage Charges (رسوم الاستخدام)
├── Billable Metrics (مقاييس الاستخدام)
├── Add-ons (الإضافات)
├── Coupons (كوبونات الخصم)
└── Integrations (التكاملات - Stripe, Adyen, GoCardless)
```

### المصطلحات المهمة

| المصطلح | الشرح |
|---|---|
| **Customer** | العميل — يُعرَّف بـ `external_id` (في ChatzyAI: `account_123`) |
| **Plan** | خطة التسعير — تحتوي سعر ثابت + رسوم استخدام |
| **Subscription** | ربط عميل بخطة مع تاريخ بداية |
| **Billable Metric** | مقياس الاستخدام (مثل: عدد رسائل AI، حجم التخزين) |
| **Charge** | ربط مقياس استخدام بسعر داخل خطة |
| **Event** | حدث استخدام يُرسل من التطبيق (مثل: رسالة AI واحدة) |
| **Invoice** | فاتورة تُنشأ تلقائياً نهاية كل دورة |
| **Webhook** | إشعار يرسله Lago لتطبيقك عند حدوث شيء |
| **Coupon** | كوبون خصم يُطبّق على فواتير العميل |
| **Wallet** | رصيد مسبق الدفع (Prepaid Credits) |
| **Add-on** | إضافة لمرة واحدة (One-time charge) |

---

## 3. إعداد لوحة التحكم

### الوصول للوحة التحكم
```
URL: https://dent-ix.app:8443
```

### الإعدادات الأساسية
افتح **Settings** من القائمة الجانبية:

#### معلومات المؤسسة (Organization)
- **Name**: اسم شركتك (مثلاً: ChatzyAI)
- **Email**: بريد الفوترة
- **Address**: عنوان يظهر على الفواتير
- **Tax ID**: الرقم الضريبي (إن وُجد)
- **Logo**: شعار يظهر على الفواتير

#### إعدادات الفوترة
- **Timezone**: المنطقة الزمنية لحساب الدورات
- **Document locale**: لغة الفواتير (en, ar, fr...)
- **Invoice grace period**: فترة السماح قبل إنهاء الفاتورة (بالأيام)

#### API Keys
```
API Key: lago_key-hooli-1234567890
```
> ⚠️ هذا المفتاح مطلوب لكل استدعاءات API. لا تشاركه أبداً.

---

## 4. إدارة العملاء (Customers)

### إنشاء عميل من لوحة Lago
1. اذهب إلى **Customers** من القائمة الجانبية
2. اضغط **Add a customer**
3. املأ:
   - **External ID**: معرّف فريد (في ChatzyAI: `account_{id}`)
   - **Name**: اسم العميل
   - **Email**: بريد العميل
   - **Currency**: العملة الافتراضية (USD)
   - **Timezone**: المنطقة الزمنية

### إنشاء عميل عبر API (تلقائياً من ChatzyAI)
```bash
curl -X POST http://127.0.0.1:3001/api/v1/customers \
  -H "Authorization: Bearer lago_key-hooli-1234567890" \
  -H "Content-Type: application/json" \
  -d '{
    "customer": {
      "external_id": "account_5",
      "name": "شركة النجوم",
      "email": "info@stars.com",
      "currency": "USD",
      "billing_configuration": {
        "payment_provider": "stripe",
        "sync": true,
        "sync_with_provider": true,
        "provider_payment_methods": ["card"]
      }
    }
  }'
```

### كيف يعمل تلقائياً في ChatzyAI؟
عند إنشاء حساب جديد في ChatzyAI، يتم استدعاء:
```ruby
# app/services/lago/customer_sync_service.rb
Lago::CustomerSyncService.new(account).sync!
```
هذا يُنشئ العميل في Lago تلقائياً مع ربطه بـ Stripe.

---

## 5. إنشاء الخطط (Plans)

### من لوحة التحكم
1. اذهب إلى **Plans** من القائمة الجانبية
2. اضغط **Add a plan**
3. املأ الحقول:

| الحقل | الوصف | مثال |
|---|---|---|
| **Name** | اسم الخطة | Pro Plan |
| **Code** | كود فريد (مهم جداً!) | `pro_plan` |
| **Interval** | دورة الفوترة | Monthly / Yearly / Weekly |
| **Amount** | السعر الثابت | $299 |
| **Currency** | العملة | USD |
| **Pay in advance** | الدفع مقدماً | ✅ (مُوصى به للاشتراكات) |
| **Trial period** | أيام التجربة المجانية | 14 |

### الأكواد المطلوبة لـ ChatzyAI
> ⚠️ **مهم جداً**: يجب أن تتطابق أكواد الخطط مع `PLAN_CODE_MAP` في ChatzyAI:

| كود الخطة في Lago | اسم الخطة | السعر |
|---|---|---|
| `free_plan` | Free Trial | $0 |
| `starter_plan` | Starter | $39/month |
| `pro_plan` | Pro | $299/month |
| `enterprise_plan` | Enterprise | $799/month |

### إنشاء خطة عبر API
```bash
curl -X POST http://127.0.0.1:3001/api/v1/plans \
  -H "Authorization: Bearer lago_key-hooli-1234567890" \
  -H "Content-Type: application/json" \
  -d '{
    "plan": {
      "name": "Pro Plan",
      "code": "pro_plan",
      "interval": "monthly",
      "amount_cents": 29900,
      "amount_currency": "USD",
      "pay_in_advance": true,
      "trial_period": 0,
      "description": "ChatzyAI Pro - 5000 AI Messages"
    }
  }'
```

### نموذج الخطة (Plan Model)
كل خطة تتكون من:
```
الخطة (Plan)
├── رسم ثابت (Fixed Fee): $299/شهر
├── رسوم استخدام (Usage Charges):
│   ├── رسائل AI: $0.02 لكل رسالة بعد 5000
│   ├── تخزين: $0.50 لكل GB بعد 10GB
│   └── قنوات: $5 لكل قناة بعد 5
└── إعدادات:
    ├── الدفع مقدماً: نعم
    ├── فترة التجربة: 14 يوم
    └── الدورة: شهرية
```

---

## 6. مقاييس الاستخدام (Billable Metrics)

مقاييس الاستخدام هي الأساس الذي يُبنى عليه نظام الفوترة حسب الاستهلاك.

### إنشاء مقياس
1. اذهب إلى **Billable Metrics**
2. اضغط **Add a billable metric**
3. الحقول:

| الحقل | الوصف | مثال |
|---|---|---|
| **Name** | اسم المقياس | AI Messages |
| **Code** | كود فريد | `ai_messages` |
| **Description** | وصف | عدد رسائل الذكاء الاصطناعي |
| **Aggregation type** | نوع التجميع | COUNT / SUM / MAX / UNIQUE COUNT |
| **Field name** | اسم الحقل في الحدث | `count` |
| **Recurring** | هل يتكرر كل دورة | No (يُصفَّر كل شهر) |

### أنواع التجميع (Aggregation Types)

| النوع | الشرح | مثال |
|---|---|---|
| **COUNT** | عدد الأحداث | عدد رسائل AI المرسلة |
| **SUM** | مجموع قيمة حقل معين | إجمالي حجم الملفات المرفوعة (MB) |
| **MAX** | أعلى قيمة | أقصى عدد مستخدمين متصلين |
| **UNIQUE COUNT** | عدد القيم الفريدة | عدد المحادثات النشطة |
| **LATEST** | آخر قيمة مُرسلة | عدد القنوات الحالي |
| **WEIGHTED SUM** | مجموع مرجّح بالزمن | استخدام CPU بالساعة |

### مقاييس ChatzyAI المطلوبة

```bash
# مقياس رسائل AI
curl -X POST http://127.0.0.1:3001/api/v1/billable_metrics \
  -H "Authorization: Bearer lago_key-hooli-1234567890" \
  -H "Content-Type: application/json" \
  -d '{
    "billable_metric": {
      "name": "AI Messages",
      "code": "ai_messages",
      "aggregation_type": "sum_agg",
      "field_name": "count"
    }
  }'

# مقياس التخزين
curl -X POST http://127.0.0.1:3001/api/v1/billable_metrics \
  -H "Authorization: Bearer lago_key-hooli-1234567890" \
  -H "Content-Type: application/json" \
  -d '{
    "billable_metric": {
      "name": "Storage (MB)",
      "code": "storage_mb",
      "aggregation_type": "sum_agg",
      "field_name": "size_mb"
    }
  }'
```

---

## 7. الرسوم (Charges)

الرسوم تربط **مقياس استخدام** بـ **سعر** داخل **خطة**.

### نماذج التسعير المتاحة

#### 1. Standard (قياسي)
سعر ثابت لكل وحدة:
```
كل رسالة AI = $0.02
```

#### 2. Graduated (متدرج)
السعر يتغير حسب الحجم:
```
أول 1000 رسالة  → $0.00 (مجاناً)
من 1001 إلى 5000 → $0.01 لكل رسالة
أكثر من 5000    → $0.005 لكل رسالة
```

#### 3. Package (حزمة)
تسعير بالحزم:
```
كل 100 رسالة = $1.50
(إذا استخدم 150 رسالة يدفع ثمن حزمتين = $3.00)
```

#### 4. Percentage (نسبة مئوية)
نسبة من قيمة المعاملة:
```
1.5% من قيمة كل معاملة مالية
+ رسم ثابت $0.30 لكل معاملة
```

#### 5. Volume (حجم)
السعر يعتمد على إجمالي الاستخدام:
```
إذا الإجمالي < 1000 → كل وحدة = $0.05
إذا الإجمالي < 10000 → كل وحدة = $0.03
إذا الإجمالي > 10000 → كل وحدة = $0.01
```

### إضافة رسوم لخطة (عبر API)
```bash
curl -X PUT http://127.0.0.1:3001/api/v1/plans/pro_plan \
  -H "Authorization: Bearer lago_key-hooli-1234567890" \
  -H "Content-Type: application/json" \
  -d '{
    "plan": {
      "charges": [
        {
          "billable_metric_code": "ai_messages",
          "charge_model": "graduated",
          "properties": {
            "graduated_ranges": [
              {"from_value": 0, "to_value": 5000, "per_unit_amount": "0", "flat_amount": "0"},
              {"from_value": 5001, "to_value": null, "per_unit_amount": "0.02", "flat_amount": "0"}
            ]
          }
        }
      ]
    }
  }'
```

---

> **تابع الجزء الثاني** في الملف `/home/lago-guide-part2.md`
> يتضمن: الاشتراكات، الفوترة، قنوات الدفع، Webhooks، الكوبونات، المحافظ، والإدارة.




# دليل Lago الشامل مع ChatzyAI — الجزء الثاني

---

## 8. الاشتراكات (Subscriptions)

### تعيين خطة لعميل من لوحة التحكم
1. اذهب إلى **Customers** → اختر العميل
2. اضغط **Add a subscription**
3. اختر الخطة (مثل `pro_plan`)
4. حدد تاريخ البداية
5. اضغط **Add subscription**

### تعيين خطة عبر API
```bash
curl -X POST http://127.0.0.1:3001/api/v1/subscriptions \
  -H "Authorization: Bearer lago_key-hooli-1234567890" \
  -H "Content-Type: application/json" \
  -d '{
    "subscription": {
      "external_customer_id": "account_5",
      "plan_code": "pro_plan",
      "external_id": "sub_5",
      "billing_time": "anniversary"
    }
  }'
```

### أنواع توقيت الفوترة (Billing Time)
| النوع | الشرح |
|---|---|
| **Calendar** | الدورة تبدأ أول الشهر (1-30/31) |
| **Anniversary** | الدورة تبدأ من يوم الاشتراك |

### ترقية / تخفيض الخطة
عند تعيين خطة جديدة لعميل لديه اشتراك نشط:
- **الرسوم الثابتة**: تُحسب نسبياً (Prorated)
- **رسوم الاستخدام**: تُصدر فاتورة فورية للاستخدام الحالي
- الخطة الجديدة تبدأ فوراً

### إلغاء اشتراك
```bash
curl -X DELETE http://127.0.0.1:3001/api/v1/subscriptions/sub_5 \
  -H "Authorization: Bearer lago_key-hooli-1234567890"
```

### كيف يعمل تلقائياً في ChatzyAI؟
```ruby
# عند الترقية - app/services/lago/subscription_sync_service.rb
Lago::SubscriptionSyncService.new(account, subscription, plan).sync!

# عند الإلغاء - aichatbot/subscription_concern.rb
Lago::LagoClient.delete("subscriptions/#{sub.lago_subscription_id}")
```

---

## 9. إرسال أحداث الاستخدام (Events)

### كيف يعمل؟
```
ChatzyAI → رسالة AI جديدة → إرسال Event لـ Lago → Lago يجمّع الاستخدام → فاتورة نهاية الشهر
```

### إرسال حدث عبر API
```bash
curl -X POST http://127.0.0.1:3001/api/v1/events \
  -H "Authorization: Bearer lago_key-hooli-1234567890" \
  -H "Content-Type: application/json" \
  -d '{
    "event": {
      "transaction_id": "ai_5_conv123_1715443200_abc1",
      "external_customer_id": "account_5",
      "external_subscription_id": "sub_5",
      "code": "ai_messages",
      "timestamp": 1715443200,
      "properties": {
        "count": 1
      }
    }
  }'
```

### القواعد المهمة
- **transaction_id**: يجب أن يكون فريداً لكل حدث (Idempotency)
- **code**: يجب أن يتطابق مع كود الـ Billable Metric
- **external_subscription_id**: معرّف الاشتراك
- يمكن إرسال أحداث بأثر رجعي (حتى يوم واحد)

### كيف يعمل في ChatzyAI؟
```ruby
# app/services/lago/event_ingestion_service.rb

# بعد كل رسالة AI
Lago::EventIngestionService.new(account).track_ai_message!(
  conversation_id: conversation.id, count: 1
)

# بعد رفع ملف
Lago::EventIngestionService.new(account).track_storage!(
  size_mb: file_size, filename: file.name
)
```

---

## 10. الفوترة (Invoicing)

### كيف تُنشأ الفواتير؟
Lago يُنشئ الفواتير **تلقائياً** نهاية كل دورة فوترة:

```
بداية الشهر → العميل يستخدم الخدمة → نهاية الشهر → Lago يُنشئ فاتورة
                                                         ↓
                                              الرسم الثابت ($299)
                                            + رسائل AI إضافية ($50)
                                            + تخزين إضافي ($10)
                                            - كوبون خصم (-$30)
                                            ─────────────────────
                                              الإجمالي: $329
```

### أنواع الفواتير
| النوع | متى |
|---|---|
| **Subscription** | بداية كل دورة (للرسوم المدفوعة مقدماً) |
| **Usage** | نهاية كل دورة (لرسوم الاستخدام) |
| **Add-on** | فوري عند شراء إضافة |
| **Credit** | إشعار دائن (استرداد) |

### فترة السماح (Grace Period)
يمكنك تحديد عدد أيام بعد إنشاء الفاتورة قبل إنهائها:
- خلال فترة السماح: الفاتورة **مسودة** (Draft) ويمكن تعديلها
- بعد انتهاء الفترة: الفاتورة **نهائية** (Finalized) ويتم تحصيلها

---

## 11. ربط قنوات الدفع (Payment Providers)

### القنوات المدعومة

| القناة | الوصف | طرق الدفع |
|---|---|---|
| **Stripe** | الأكثر شيوعاً | بطاقات، SEPA، ACH، Link، Crypto |
| **GoCardless** | خصم مباشر | SEPA، BACS، ACH |
| **Adyen** | عالمي | بطاقات، iDEAL، Bancontact، وأكثر |

---

### ربط Stripe (مُوصى به)

#### الخطوة 1: إعداد الربط
1. افتح لوحة Lago → **Settings** → **Integrations**
2. اضغط **Stripe** → **Add connection**
3. أدخل:
   - **Connection name**: `ChatzyAI Stripe`
   - **Connection code**: `chatzyai_stripe`
   - **API Key**: مفتاح Stripe السري (`sk_live_...` أو `sk_test_...`)
4. اضغط **Connect to Stripe**

#### الخطوة 2: تعيين Stripe للعميل
عند إنشاء/تحديث عميل، أضف:
```json
{
  "customer": {
    "external_id": "account_5",
    "billing_configuration": {
      "payment_provider": "stripe",
      "sync": true,
      "sync_with_provider": true,
      "provider_payment_methods": ["card"]
    }
  }
}
```

#### الخطوة 3: جمع بيانات الدفع (Checkout)
Lago يُنشئ رابط Stripe Checkout تلقائياً ويرسله عبر Webhook:
```json
{
  "webhook_type": "customer.checkout_url_generated",
  "payment_provider_customer_checkout_url": {
    "checkout_url": "https://checkout.stripe.com/c/pay/..."
  }
}
```

#### طرق الدفع عبر Stripe
| الطريقة | الكود | العملة |
|---|---|---|
| بطاقات ائتمان | `card` | أي عملة |
| Stripe Link | `link` | أي عملة |
| SEPA (أوروبا) | `sepa_debit` | EUR |
| ACH (أمريكا) | `us_bank_account` | USD |
| BACS (بريطانيا) | `bacs_debit` | GBP |
| Boleto (برازيل) | `boleto` | BRL |
| عملات رقمية | `crypto` | USD |
| تحويل بنكي | `customer_balance` | متعدد |

---

### ربط GoCardless

#### الإعداد
1. افتح **Settings** → **Integrations** → **GoCardless**
2. أدخل **Access Token** من لوحة GoCardless
3. اربط العملاء مع `"payment_provider": "gocardless"`

#### طرق الدفع
| الطريقة | المنطقة |
|---|---|
| SEPA Direct Debit | أوروبا |
| BACS Direct Debit | المملكة المتحدة |
| ACH Direct Debit | الولايات المتحدة |
| BECS Direct Debit | أستراليا |
| PAD | كندا |

---

### ربط Adyen

#### الإعداد
1. افتح **Settings** → **Integrations** → **Adyen**
2. أدخل:
   - **API Key**: من لوحة Adyen
   - **Merchant Account**: اسم حساب التاجر
   - **Live Prefix**: (للبيئة الحية فقط)
3. اربط العملاء مع `"payment_provider": "adyen"`

---

## 12. Webhooks (الإشعارات)

### ما هي؟
Lago يرسل إشعارات HTTP لتطبيقك عند حدوث أحداث مهمة.

### الإعداد في ChatzyAI
تم تسجيل Webhook تلقائياً:
```
URL: https://dent-ix.app/webhooks/lago
Secret: d8b37c7a-f668-4a52-9090-bb74c118e966
Algorithm: HMAC-SHA256
```

### أهم أنواع Webhooks

| الحدث | متى يُرسل | ماذا يفعل ChatzyAI |
|---|---|---|
| `invoice.created` | عند إنشاء فاتورة | يحفظ الفاتورة في `lago_invoices` |
| `invoice.payment_status_updated` | عند تغيّر حالة الدفع | يحدّث حالة الاشتراك |
| `subscription.started` | عند بدء اشتراك | يفعّل الاشتراك (`active`) |
| `subscription.terminated` | عند إلغاء اشتراك | يلغي الاشتراك (`cancelled`) |
| `customer.checkout_url_generated` | عند إنشاء رابط دفع | يوجّه المستخدم للدفع |
| `customer.payment_provider_created` | عند ربط عميل بـ Stripe | يحفظ `stripe_customer_id` |

### كيف يتعامل ChatzyAI مع Webhooks؟
```ruby
# app/controllers/api/v1/lago_webhooks_controller.rb
class Api::V1::LagoWebhooksController < ApplicationController
  def create
    # 1. التحقق من التوقيع (HMAC)
    # 2. تحديد نوع الحدث
    # 3. معالجة الحدث (تحديث الاشتراك/الفاتورة)
  end
end
```

---

## 13. الكوبونات (Coupons)

### إنشاء كوبون
1. اذهب إلى **Coupons** → **Add a coupon**
2. الحقول:

| الحقل | الوصف |
|---|---|
| **Name** | اسم الكوبون |
| **Code** | كود الكوبون (مثل: `WELCOME50`) |
| **Type** | نسبة مئوية أو مبلغ ثابت |
| **Value** | قيمة الخصم (50% أو $50) |
| **Duration** | مرة واحدة / عدة أشهر / للأبد |
| **Expiration** | تاريخ انتهاء الصلاحية |
| **Applies to** | كل الخطط أو خطط محددة |

### تطبيق كوبون على عميل
```bash
curl -X POST http://127.0.0.1:3001/api/v1/applied_coupons \
  -H "Authorization: Bearer lago_key-hooli-1234567890" \
  -H "Content-Type: application/json" \
  -d '{
    "applied_coupon": {
      "external_customer_id": "account_5",
      "coupon_code": "WELCOME50"
    }
  }'
```

---

## 14. المحافظ والرصيد المسبق (Wallets & Prepaid Credits)

### الفكرة
بدلاً من الدفع بعد الاستخدام، يدفع العميل مقدماً ويستهلك من رصيده.

### إنشاء محفظة
```bash
curl -X POST http://127.0.0.1:3001/api/v1/wallets \
  -H "Authorization: Bearer lago_key-hooli-1234567890" \
  -H "Content-Type: application/json" \
  -d '{
    "wallet": {
      "external_customer_id": "account_5",
      "name": "AI Credits",
      "rate_amount": "0.02",
      "paid_credits": "100.00",
      "granted_credits": "10.00",
      "currency": "USD",
      "expiration_at": "2027-01-01T00:00:00Z"
    }
  }'
```

| الحقل | الشرح |
|---|---|
| **rate_amount** | سعر كل كريديت ($0.02 = كل كريديت يساوي رسالة AI واحدة) |
| **paid_credits** | كريديتات مدفوعة ($100 = 5000 رسالة) |
| **granted_credits** | كريديتات مجانية هدية ($10 = 500 رسالة) |

### شحن المحفظة
```bash
curl -X POST http://127.0.0.1:3001/api/v1/wallet_transactions \
  -H "Authorization: Bearer lago_key-hooli-1234567890" \
  -H "Content-Type: application/json" \
  -d '{
    "wallet_transaction": {
      "wallet_id": "wallet_lago_id_here",
      "paid_credits": "50.00",
      "granted_credits": "5.00"
    }
  }'
```

---

## 15. الإضافات (Add-ons)

### ما هي؟
رسوم لمرة واحدة تُضاف للعميل خارج نطاق الاشتراك.

### إنشاء إضافة
```bash
curl -X POST http://127.0.0.1:3001/api/v1/add_ons \
  -H "Authorization: Bearer lago_key-hooli-1234567890" \
  -H "Content-Type: application/json" \
  -d '{
    "add_on": {
      "name": "Extra 1000 AI Messages",
      "code": "extra_ai_1000",
      "amount_cents": 1500,
      "amount_currency": "USD"
    }
  }'
```

### تطبيق إضافة على عميل
```bash
curl -X POST http://127.0.0.1:3001/api/v1/applied_add_ons \
  -H "Authorization: Bearer lago_key-hooli-1234567890" \
  -H "Content-Type: application/json" \
  -d '{
    "applied_add_on": {
      "external_customer_id": "account_5",
      "add_on_code": "extra_ai_1000"
    }
  }'
```

---

## 16. إعدادات ChatzyAI الحالية

### المتغيرات في قاعدة البيانات (InstallationConfig)

| الاسم | القيمة | الوظيفة |
|---|---|---|
| `LAGO_ENABLED` | `true` | تفعيل/تعطيل Lago |
| `LAGO_API_KEY` | `lago_key-hooli-1234567890` | مفتاح API |
| `LAGO_API_URL` | `http://127.0.0.1:3001` | عنوان Lago API |
| `LAGO_WEBHOOK_SECRET` | `d8b37c7a-...` | مفتاح التوقيع |

### الخدمات البرمجية (Services)

| الخدمة | الملف | الوظيفة |
|---|---|---|
| **LagoClient** | `lago/lago_client.rb` | HTTP Client لـ Lago API |
| **CustomerSyncService** | `lago/customer_sync_service.rb` | مزامنة العملاء |
| **SubscriptionSyncService** | `lago/subscription_sync_service.rb` | مزامنة الاشتراكات |
| **PaymentLinkService** | `lago/payment_link_service.rb` | إنشاء روابط الدفع |
| **EventIngestionService** | `lago/event_ingestion_service.rb` | إرسال أحداث الاستخدام |

### جداول قاعدة البيانات

| الجدول | أعمدة Lago |
|---|---|
| `account_subscriptions` | `lago_customer_id`, `lago_subscription_id`, `lago_plan_code` |
| `lago_invoices` | `lago_invoice_id`, `amount_cents`, `payment_status`, `invoice_url` |

---

## 17. دليل المشاكل الشائعة

### المشكلة: خطأ 401 Unauthorized
**السبب**: مفتاح API خاطئ
**الحل**: تأكد من المفتاح:
```bash
curl -s http://127.0.0.1:3001/api/v1/plans \
  -H "Authorization: Bearer lago_key-hooli-1234567890" | head -20
```

### المشكلة: Webhooks لا تصل
**تأكد من:**
1. تسجيل Webhook: `Settings → Webhooks` في لوحة Lago
2. URL صحيح: `https://dent-ix.app/webhooks/lago`
3. شهادة SSL صالحة
4. Nginx يمرر الطلبات بشكل صحيح

### المشكلة: الخطة غير موجودة
**السبب**: كود الخطة في Lago لا يتطابق مع `PLAN_CODE_MAP`
**الحل**: تأكد أن أكواد الخطط: `free_plan`, `starter_plan`, `pro_plan`, `enterprise_plan`

### المشكلة: الفواتير لا تُنشأ
**تأكد من:**
1. العميل لديه اشتراك نشط
2. الدورة انتهت (أو استخدم الفوترة المقدمة)
3. خادم Lago Worker يعمل:
```bash
docker logs chatwoot-lago-worker-1 --tail 50
```

---

## 18. أوامر مفيدة

```bash
# عرض كل الخطط
curl -s http://127.0.0.1:3001/api/v1/plans \
  -H "Authorization: Bearer lago_key-hooli-1234567890" | python3 -m json.tool

# عرض كل العملاء
curl -s http://127.0.0.1:3001/api/v1/customers \
  -H "Authorization: Bearer lago_key-hooli-1234567890" | python3 -m json.tool

# عرض فواتير عميل
curl -s "http://127.0.0.1:3001/api/v1/invoices?external_customer_id=account_5" \
  -H "Authorization: Bearer lago_key-hooli-1234567890" | python3 -m json.tool

# عرض اشتراكات عميل
curl -s "http://127.0.0.1:3001/api/v1/subscriptions?external_customer_id=account_5" \
  -H "Authorization: Bearer lago_key-hooli-1234567890" | python3 -m json.tool

# فحص سجلات Lago
docker logs chatwoot-lago-api-1 --tail 100
docker logs chatwoot-lago-worker-1 --tail 100

# إعادة تشغيل Lago
docker restart chatwoot-lago-api-1 chatwoot-lago-worker-1
```

---

## المراجع الرسمية
- [التوثيق الرسمي](https://docs.getlago.com)
- [API Reference](https://docs.getlago.com/api-reference/intro)
- [GitHub](https://github.com/getlago/lago)
- [Stripe Integration](https://docs.getlago.com/integrations/payments/stripe-integration)
- [GoCardless Integration](https://docs.getlago.com/integrations/payments/gocardless-integration)
- [Adyen Integration](https://docs.getlago.com/integrations/payments/adyen-integration)
