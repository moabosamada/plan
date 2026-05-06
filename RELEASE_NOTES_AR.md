# 📋 تقرير التحديثات الشامل — Chatzy AI
**التاريخ:** 6 مايو 2026  
**إعداد:** فريق التطوير  
**البيئة:** Chatwoot Custom + AI Bot System

---

## 🔧 قبل البدء في الاختبار

> **مهم:** يجب تنفيذ أمر البناء أولاً حتى تظهر جميع التغييرات:

```bash
cd /home/chatwoot && ./rebuild.sh
```

بعد انتهاء البناء (3-5 دقائق)، قم بعمل Hard Refresh للمتصفح:
- **Windows/Linux:** `Ctrl + F5`
- **Mac:** `Cmd + Shift + R`

---

## 📦 قائمة التحديثات المُنجزة

---

### 1️⃣ زر "Resume AI" — إعادة تفعيل البوت من المحادثة

**المشكلة السابقة:**  
لإعادة البوت للمحادثة، كان العميل يكتب `/unpause` يدوياً في ملاحظة خاصة (Private Note)، وهو أمر غير عملي ومربك.

**ما تم عمله:**  
إضافة زر **"Resume AI"** ملوّن بالأزرق مباشرةً في شريط الرد أسفل كل محادثة.

**كيفية التجربة:**
1. افتح أي محادثة فيها بوت مفعّل
2. قم بالرد كـ Agent بشري (سيتوقف البوت تلقائياً ويظهر نشاط "AI Bot paused")
3. انظر في الشريط السفلي — ستجد زر أزرق **"Resume AI"** بجانب زر **"Human"**
4. اضغط **"Resume AI"** — سيظهر نشاط "AI Bot was re-activated" وسيعود البوت للرد تلقائياً على الرسائل الجديدة

**الملفات المعدلة:**
- `app/javascript/dashboard/components/widgets/WootWriter/ReplyBottomPanel.vue`
- `app/javascript/dashboard/components/widgets/conversation/ReplyBox.vue`
- `app/javascript/dashboard/api/aichatbot.js`
- `app/controllers/api/v1/accounts/aichatbot_controller.rb`
- `config/routes.rb`

---

### 2️⃣ إعادة تصميم زر "Mark as Human" (Human Handled)

**المشكلة السابقة:**  
الزر كان صغيراً جداً وغير واضح — أيقونة خضراء بدون نص.

**ما تم عمله:**  
أصبح الزر:
- لون **أصفر/برتقالي (Amber) Solid** — مميز وواضح
- يحمل نص **"Human"** مع أيقونة `user-check`
- مطابق لنفس تصميم باقي الأزرار في النظام

**كيفية التجربة:**
1. افتح أي محادثة نشطة
2. انظر في الشريط السفلي — زر **"Human"** البرتقالي
3. اضغط عليه — ستظهر رسالة نجاح "Conversation marked as Human Resolved"
4. تُحسب هذه المحادثة في إحصائيات "Human Resolved" في لوحة التحكم

---

### 3️⃣ Pagination حقيقية لجدول العملاء (Clients)

**المشكلة السابقة:**  
جدول العملاء كان يحمّل جميع السجلات دفعة واحدة مما يبطّئ الصفحة مع كثرة العملاء.

**ما تم عمله:**  
- الجدول يعمل الآن بـ **Server-Side Pagination** (20 عميل لكل صفحة)
- البحث يعمل عبر السيرفر مباشرة (اسم العميل، البريد، Client ID)
- فلتر الحالة يعمل من السيرفر أيضاً

**كيفية التجربة:**
1. من لوحة السوبر أدمن → **Chatzy AI → Clients**
2. ستجد أزرار الصفحات في الأسفل
3. جرب البحث بالاسم أو البريد الإلكتروني — النتائج تأتي فورياً من السيرفر
4. فلتر الحالة (All Status / Active / Pending Approval / Suspended) يُرشّح من السيرفر

---

### 4️⃣ Core Knowledge مفتوح دائماً للسوبر أدمن

**المشكلة السابقة:**  
حتى السوبر أدمن كان لا يستطيع رفع Core Knowledge إلا في اليوم الأول من الشهر.

**ما تم عمله:**  
السوبر أدمن الآن يستطيع رفع Core Knowledge في **أي وقت** دون قيود.

**كيفية التجربة:**
1. سجّل دخول بحساب السوبر أدمن
2. اذهب لأي عميل → **Knowledge Base → Core Knowledge**
3. جرب رفع ملف أو نص في أي يوم من الشهر (حتى لو ليس اليوم الأول)
4. يجب أن يعمل بدون رسالة خطأ "Core knowledge can only be updated on the 1st"
5. **للعملاء العاديين:** القيد لا يزال مفعّلاً (اليوم الأول فقط أو خلال 3 أيام من الإنشاء)

---

### 5️⃣ نظام الإغلاق التلقائي الذكي (Auto-Resolve)

**الميزة الجديدة:**  
البوت يمكنه إغلاق المحادثات تلقائياً بعد فترة من عدم رد العميل.

**آلية العمل:**
- إذا أرسل البوت رسالة وانتظر X دقيقة دون رد من العميل
- يرسل رسالة ختامية مخصصة
- يُغلق المحادثة ويصنّفها كـ `ai_resolved` في الإحصائيات
- يعمل كل 5 دقائق تلقائياً في الخلفية

**كيفية الإعداد والتجربة:**
1. اذهب إلى **Settings → Chatzy AI → Knowledge Base**
2. اضغط تبويب **Automation**
3. انزل لأسفل — ستجد قسم جديد **"Auto-Resolve Conversations"**
4. فعّل الخيار **"Enable Auto-Resolve"**
5. ضع مدة الانتظار (مثلاً: `5` دقائق للاختبار)
6. اكتب رسالة الإغلاق بالعربية مثلاً:
   > "شكراً لتواصلك معنا. سيتم إغلاق هذه المحادثة. نتطلع لخدمتك مجدداً!"
7. احفظ الإعدادات
8. افتح محادثة نشطة وارسل رسالة كعميل
9. دع البوت يرد ثم انتظر 5 دقائق بدون رد
10. يجب أن تُغلق المحادثة تلقائياً مع رسالة الإغلاق

**ملاحظة:** الـ Job يعمل كل 5 دقائق، لذا قد يستغرق حتى 10 دقائق في أسوأ الأحوال.

---

### 6️⃣ تتبع الاستخدام الزائد وجلسات الدعم البشري

**الميزة الجديدة:**  
عدادان جديدان يظهران في لوحة تحكم العميل ولوحة السوبر أدمن:

| العداد | الوصف |
|--------|-------|
| **AI Messages Over Limit** | عدد رسائل الذكاء الاصطناعي التي تجاوزت الحد الشهري |
| **Human Agent Sessions** | عدد المحادثات التي رد عليها موظف بشري (بالضغط على زر "Human") |

**كيفية التجربة:**
1. **لوحة العميل:** اذهب لـ **Chatzy AI → Dashboard** — ستجد 4 بطاقات إحصاء في السطر الثاني
2. **تفاصيل العميل (سوبر أدمن):** اذهب لـ **Clients → اختر عميل → Details**
   - في بطاقة **AI Limits** ستجد خانتان جديدتان:
     - "AI Messages Over Limit"
     - "Human Agent Sessions"
3. لاختبار العداد: اضغط زر **"Human"** في محادثة → سيزيد العداد بـ 1

**في لوحة السوبر أدمن (TenantDetail):**
- إذا كان العداد > 0 يظهر باللون الأحمر تنبيهاً

---

### 7️⃣ "Powered by Chatzy.AI" في Web Widget

**الميزة الجديدة:**  
عند ربط أي عميل بـ Web Widget، يظهر نص "Powered by Chatzy.AI" (رابط قابل للنقر) في نافذة الشات.

**كيفية التجربة:**
1. اذهب لـ **Settings → Chatzy AI → Channels**
2. اضغط **"Create Web Widget"**
3. انسخ الـ Script المُولَّد وضعه في صفحة HTML عادية:

```html
<!DOCTYPE html>
<html>
<body>
  <h1>Test Page</h1>
  <!-- الصق الـ Script هنا -->
</body>
</html>
```

4. افتح الصفحة في المتصفح
5. افتح نافذة الشات (الزر السفلي الأيمن)
6. ستجد نص **"Powered by Chatzy.AI"** بالقرب من الزر

---

### 8️⃣ نظام الموافقة على طلبات التسجيل

**كيف يعمل النظام الجديد:**

#### من جانب العميل:
1. يسجّل العميل عبر صفحة `/register`
2. يُدخل بريده الإلكتروني، اسمه، وكلمة المرور
3. يتحقق من بريده بكود التحقق
4. يصل لصفحة **"Registration Pending"** — تشرح له أن طلبه قيد المراجعة
5. ينتظر بريداً إلكترونياً بتأكيد التفعيل

#### من جانب السوبر أدمن:
1. اذهب لـ **Chatzy AI → Clients**
2. إذا كان هناك طلبات جديدة ستظهر **بانر أصفر** في الأعلى:
   > "2 registrations awaiting approval"
3. اضغط **"View Pending"** أو اختر من الفلتر **"Pending Approval"**
4. ستجد العملاء بحالة `pending_activation` مع زر **✓ أخضر** للموافقة
5. اضغط زر **✓** — سيطلب تأكيداً ثم:
   - يُفعّل الحساب فوراً
   - يُرسل بريد إلكتروني تلقائي للعميل يُخبره بأن حسابه مفعّل مع رابط تسجيل الدخول

**كيفية الاختبار:**
1. افتح `/register` في متصفح incognito
2. سجّل بحساب جديد (بريد إلكتروني غير مستخدم)
3. تحقق من البريد وادخل كود التحقق
4. ستنتقل لصفحة "Registration Pending"
5. ارجع للوحة السوبر أدمن → Clients → ستجد البانر الأصفر
6. وافق على الطلب → تحقق من البريد الإلكتروني للعميل

---

### 9️⃣ تحسين زر "Lift Knowledge Ban" في تفاصيل العميل

**المشكلة السابقة:**  
الزر كان يختفي بعد تفعيل الـ Override — مما يجعل السوبر أدمن غير قادر على تمديد الوقت.

**ما تم عمله:**
- إذا لم يكن هناك Override نشط → زر **"Lift Knowledge Ban"** (Teal outline)
- إذا كان هناك Override نشط → زر **"Override Active — Extend Ban Lift"** (Teal solid) لتمديد الوقت

**كيفية التجربة:**
1. افتح **Chatzy AI → Clients → اختر عميل**
2. في بطاقة **AI Limits** ستجد الزر في الأسفل
3. اضغطه → يطلب عدد الساعات (مثلاً: 24)
4. سيظهر زر جديد بلون مغاير يُشير أن الـ Override نشط
5. يمكن الضغط مرة أخرى لتمديد الوقت

---

### 🔟 التحقق من توازن مفاتيح AI (Load Balancing)

**ما كان موجوداً ومُتحقق منه:**  
النظام يستخدم منذ البداية خوارزمية Round-Robin عبر حقل `usage_count`:
- كل مرة يُستخدم مفتاح، يزيد `usage_count` بمقدار 1
- في كل طلب، يُختار المفتاح ذو أقل `usage_count` (الأقل استخداماً)
- إذا فشل مفتاح (خطأ مصادقة)، يُعطَّل تلقائياً وينتقل النظام للمفتاح التالي

**كيفية التجربة:**
1. اذهب لـ **Chatzy AI → AI Provider Keys**
2. أضف مفتاحين أو أكثر من نفس المزود أو مزودين مختلفين
3. أرسل عدة رسائل من محادثات مختلفة
4. عد للجدول وستجد أن `Usage Count` موزّع بالتساوي بين المفاتيح

---

## 📊 ملخص الملفات المعدلة

### Backend (Ruby on Rails)
| الملف | التغييرات |
|-------|-----------|
| `app/controllers/api/v1/accounts/aichatbot_controller.rb` | resume_ai، تحسين pagination، super admin bypass، auto-resolve، overage counters، widget branding |
| `app/jobs/ai_bot/router_job.rb` | تتبع الاستخدام الزائد (ai_overage_count) |
| `app/jobs/ai_bot/smart_auto_resolve_job.rb` | **ملف جديد** — job الإغلاق التلقائي الذكي |
| `config/routes.rb` | إضافة route لـ resume_ai |
| `config/schedule.yml` | إضافة smart_auto_resolve_job كل 5 دقائق |
| `db/migrate/20260505130000_add_auto_resolve_to_ai_tenants.rb` | **ملف جديد** — migration للأعمدة الجديدة |
| `rebuild.sh` | إضافة الـ jobs الجديدة في Docker image |

### Frontend (Vue.js)
| الملف | التغييرات |
|-------|-----------|
| `app/javascript/dashboard/api/aichatbot.js` | إضافة `resumeAI()` method |
| `app/javascript/dashboard/components/widgets/WootWriter/ReplyBottomPanel.vue` | زر "Human" الجديد + زر "Resume AI" |
| `app/javascript/dashboard/components/widgets/conversation/ReplyBox.vue` | handler لـ resumeAI |
| `app/javascript/dashboard/routes/dashboard/settings/aichatbot/Dashboard.vue` | بطاقات العدادات الجديدة |
| `app/javascript/dashboard/routes/dashboard/settings/aichatbot/TenantDetail.vue` | عداد overage + human sessions + إصلاح زر Knowledge Ban |
| `app/javascript/dashboard/routes/dashboard/settings/aichatbot/TenantsList.vue` | بانر الموافقة + زر الموافقة السريع + فلتر Pending Approval |
| `app/javascript/dashboard/routes/dashboard/settings/aichatbot/KnowledgeBase.vue` | إعدادات Auto-Resolve في تبويب Automation |

### قاعدة البيانات — الأعمدة الجديدة على جدول `ai_tenants`
| العمود | النوع | القيمة الافتراضية | الوصف |
|--------|-------|-------------------|-------|
| `auto_resolve_enabled` | boolean | false | تفعيل الإغلاق التلقائي |
| `auto_resolve_timeout_minutes` | integer | 30 | مدة الانتظار قبل الإغلاق |
| `auto_resolve_message` | text | NULL | رسالة الإغلاق المخصصة |
| `ai_overage_count` | integer | 0 | عدد الرسائل الزائدة عن الحد |
| `human_agent_session_count` | integer | 0 | عدد جلسات الدعم البشري |

---

## ⚠️ تنبيهات مهمة

1. **أمر البناء إلزامي:** كل التغييرات في `/home/chatwoot` لن تنعكس في النظام حتى يتم تنفيذ `./rebuild.sh`

2. **الـ migrations:** تم تنفيذ migration الأعمدة الجديدة مباشرة على قاعدة البيانات وتسجيله في جدول `schema_migrations`، لذا عند تشغيل `./rebuild.sh` لن يُنفَّذ مرة أخرى.

3. **Auto-Resolve Job:** يعمل كل 5 دقائق عبر Sidekiq. بعد البناء وإعادة تشغيل Sidekiq، سيبدأ العمل تلقائياً.

4. **Widget Branding:** يعمل فقط على الـ widgets الجديدة. الـ widgets القديمة تحتاج حذف وإعادة إنشاء للحصول على الـ script الجديد.

---

## 🚀 خطوات النشر الكاملة

```bash
# 1. بناء كل شيء
cd /home/chatwoot && ./rebuild.sh

# 2. بعد انتهاء البناء، تحقق من الـ containers
docker ps

# 3. Hard refresh المتصفح
# Windows: Ctrl+F5  |  Mac: Cmd+Shift+R
```

**وقت البناء المتوقع:** 3-5 دقائق

---

*تم إعداد هذا التقرير بتاريخ 6 مايو 2026*
