# ملخص شامل — 13 لابًا

مرجع سريع للمراجعة قبل المقابلة: **النوع · الاستغلال · النتيجة · السبب الجذري**.
الترتيب حسب صفحة [Access Control](https://portswigger.net/web-security/access-control). أرقام ملفات الريبو في [الفهرس](README.md).

| # | اللاب | النوع | الاستغلال | النتيجة | السبب الجذري |
|---|---|---|---|---|---|
| 1 | Unprotected admin functionality | Vertical — Broken Function Level Authorization | `robots.txt` يكشف `/administrator-panel` → افتحه بلا تسجيل دخول | وصول كامل للوحة الإدارة وحذف `carlos` | لا يوجد فحص صلاحية على السيرفر؛ الإخفاء استُخدم بديلًا عنه |
| 2 | Unprotected admin functionality with unpredictable URL | Vertical + Information Disclosure | View source → مسار اللوحة داخل JavaScript → افتحه مباشرة | نفس النتيجة | مسار عشوائي ليس تحكم وصول، والكود يُرسل لكل الزوار |
| 3 | User role controlled by request parameter | Vertical — Trusting client-controlled data | `Cookie: Admin=false` → `Admin=true` → `/admin` | صلاحية مدير كاملة | السيرفر يقرأ الدور من كوكي يتحكم به العميل بدل قاعدة البيانات |
| 4 | User role can be modified in user profile | Mass Assignment → Vertical | `POST /my-account/change-email` بجسم `{"email":"…","roleid":2}` | ترقية ذاتية إلى مدير | ربط جسم الطلب بكيان `User` مباشرة بدون قائمة حقول مسموحة |
| 5 | URL-based access control can be circumvented | Path Confusion (platform bypass) | `GET /` مع `X-Original-URL: /admin`، ثم `/admin/delete` | تنفيذ وظيفة إدارية بلا صلاحية | الـfront-end يفحص المسار الظاهر، والـback-end يعطي الأولوية لهيدر يغيّر المسار |
| 6 | Method-based access control can be circumvented | Method-based bypass | `POST /admin-roles` محمي → `GET /admin-roles?username=wiener&action=upgrade` | ترقية ذاتية إلى مدير | الفحص مربوط بـMethod + URL بدل الوظيفة نفسها |
| 7 | User ID controlled by request parameter | Horizontal IDOR | `/my-account?id=wiener` → `?id=carlos` | API key الخاص بـ`carlos` | جلب الكائن بمُعرّف من الطلب بدون فحص ملكية |
| 8 | User ID controlled by request parameter, with unpredictable user IDs | Horizontal IDOR | جمع GUID الخاص بـ`carlos` من المدونة/التعليقات → `?id=<guid>` | API key الخاص بـ`carlos` | نفس غياب فحص الملكية + تسريب المُعرّفات في محتوى عام |
| 9 | User ID controlled by request parameter with data leakage in redirect | IDOR + Sensitive Data Exposure | `?id=carlos` مع إيقاف تتبع الـredirect → اقرأ جسم الاستجابة | API key من داخل جسم `302` | الترتيب خاطئ: Fetch → Authorize → Redirect؛ البيانات تُسلسل قبل الرفض |
| 10 | User ID controlled by request parameter with password disclosure | Horizontal → Vertical | `?id=administrator` → `<input type=password value="…">` → دخول كمدير | اختطاف حساب المدير | IDOR + إرسال كلمة المرور إلى العميل؛ الإخفاء البصري ليس أمانًا |
| 11 | Insecure direct object references | IDOR على ملفات ثابتة | `/download-transcript/2.txt` → غيّر الرقم تسلسليًا | كلمة مرور `carlos` | ملفات تُقدَّم مباشرة بلا فحص ملكية، بأسماء متسلسلة قابلة للتخمين |
| 12 | Multi-step process with no access control on one step | Context-dependent / Workflow | أرسل خطوة التأكيد فقط: `POST /admin-roles` بـ`username=wiener&action=upgrade&confirmed=true` | ترقية ذاتية إلى مدير | الفحص على خطوة واحدة فقط؛ `confirmed` حقل يتحكم به العميل وليس دليل إتمام الخطوة الأولى |
| 13 | Referer-based access control | Context-dependent / Header trust | `GET /admin-roles?username=wiener&action=upgrade` مع `Referer: https://<lab-host>/admin` | ترقية ذاتية إلى مدير | الاعتماد على هيدر `Referer` القابل للتزوير بدل فحص دور المستخدم |

---

## السبب الرئيسي الموحّد

> **قرار الصلاحية إما غير موجود، أو مبني على شيء يتحكم به العميل، أو مفروض في طبقة لا تملك القرار النهائي.**

كل الـ13 لابًا تقع في ثلاث عائلات:

| العائلة | اللابات | السؤال الذي يكشفها |
|---|---|---|
| **١. فحص الوظيفة غائب** (Function-level) | 1، 2، 5، 6، 12، 13 | هل كل مسار إداري يتحقق من الدور بنفسه؟ وهل يوجد مسار آخر (Header / Method / Step / Direct URL) يصل لنفس الوظيفة؟ |
| **٢. الثقة ببيانات يتحكم بها العميل** | 3، 4 | هل الدور أو الصلاحية يُقرأ من Cookie / Parameter / Header / جسم الطلب؟ |
| **٣. فحص الكائن غائب** (Object-level / IDOR) | 7، 8، 9، 10، 11 | هل الكود يتحقق أن هذا الكائن يملكه صاحب الجلسة، أم يفترض ذلك؟ |

لاحظ الفرق الجوهري:

- **1–6 و12–13** = مشكلة في **الوظيفة**: هل يحق لك استدعاء هذا الـEndpoint أصلًا؟
- **7–11** = مشكلة في **الكائن**: يحق لك استدعاء الـEndpoint، لكن **ليس هذا الكائن**.

---

## المنع المختصر حسب العائلة

**العائلة ١ — الوظيفة**
- `deny by default` + `requestMatchers("/admin/**").hasRole("ADMIN")`.
- `@PreAuthorize` على الـController **والـService**.
- افحص كل مسار بديل: كل HTTP Method، الهيدرات التي تغيّر المسار، وكل خطوة في التدفق.

**العائلة ٢ — مصدر القرار**
- الدور يُحمَّل من قاعدة البيانات في `UserDetailsService` ويُخزَّن في `SecurityContext`.
- لا تقرأ الدور من `Cookie` / `getParameter` / hidden field.
- DTO بقائمة حقول مسموحة؛ لا تربط الطلب بكيان فيه `role` أو `isAdmin`.

**العائلة ٣ — الكائن**
- الهوية من `Authentication` فقط: `findByUsername(auth.getName())`.
- أو فحص صريح: `@PreAuthorize("hasRole('ADMIN') or #id == authentication.principal.id")`.
- أو على مستوى الاستعلام: `findByIdAndOwnerId(id, ownerId)`.
- الملفات تمر عبر Controller يفحص الملكية؛ وتُخزَّن خارج `static/`.

---

## قواعد سريعة لمراجعة الكود

| ابحث عن | الاحتمال |
|---|---|
| `@RequestBody User` / `copyProperties` / `ModelMapper` إلى Entity | Mass Assignment |
| `getParameter("id")` ثم `findById(...)` ثم `return` | IDOR |
| `getRole()` / `isAdmin` من الطلب | دور يتحكم به العميل |
| `requestMatchers` تغطي `POST` فقط | تجاوز بالـMethod |
| `X-Original-URL` / `X-Rewrite-URL` مقبولة | Path Confusion |
| `Referer` في شرط صلاحية | Header trust |
| `confirmed` / `step` / `stage` / `next` كحقول عميل | تجاوز تدفق متعدد الخطوات |
| `addResourceLocations` / ملفات مستخدمين داخل `static/` | IDOR على ملفات |

---

**المرجع التفصيلي:** كل لاب له ملف مستقل بالعربية والإنجليزية في هذا المجلد، يشرح الاستغلال خطوة بخطوة وكود المنع في Spring Boot.
