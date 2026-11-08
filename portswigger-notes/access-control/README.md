# Access Control — فهرس اللابات

المصدر: [PortSwigger — Access Control](https://portswigger.net/web-security/access-control)

📊 **[ملخص شامل للـ13 لابًا](00-summary-all-labs.md)** — جدول واحد: النوع · الاستغلال · النتيجة · السبب الجذري.

> **ملاحظة توثيق:** الـ13 ملاحظة كُتبت **بأثر رجعي** اعتمادًا على الحلول القياسية للابات. راجع كل ملاحظة وعدّل أسماء المستخدمين/الطلبات/التفاصيل لتطابق ما فعلته فعليًا — الهدف أن يكون المستودع سجلًا لعملك أنت، وليس نسخة من حلول PortSwigger.

> **ترقيمان مختلفان — لا تخلط بينهما:**
> - **# ريبو** = ترتيب أسماء الملفات، مبني على صفحة [All labs](https://portswigger.net/web-security/all-labs).
> - **# الصفحة** = ترتيب اللاب على صفحة [Access Control](https://portswigger.net/web-security/access-control) — وهي المرجع عند المتابعة.
> - العنوان الرسمي هو الفيصل دائمًا؛ الأرقام مجرد مؤشرات.

| # ريبو | # الصفحة | اللاب | المستوى | الملاحظة | الحالة |
|---|---|---|---|---|---|
| 01 | 1 | Unprotected admin functionality | Apprentice | [فتح](01-unprotected-admin-functionality.md) | ✅ |
| 02 | 2 | Unprotected admin functionality with unpredictable URL | Apprentice | [فتح](02-unprotected-admin-functionality-with-unpredictable-url.md) | ✅ |
| 03 | 3 | User role controlled by request parameter | Apprentice | [فتح](03-user-role-controlled-by-request-parameter.md) | ✅ |
| 04 | 4 | User role can be modified in user profile | Apprentice | [فتح](04-user-role-can-be-modified-in-user-profile.md) | ✅ |
| 05 | 7 | User ID controlled by request parameter | Apprentice | [فتح](05-user-id-controlled-by-request-parameter.md) | ✅ |
| 06 | 8 | User ID controlled by request parameter, with unpredictable user IDs | Apprentice | [فتح](06-user-id-controlled-by-request-parameter-with-unpredictable-user-ids.md) | ✅ |
| 07 | 9 | User ID controlled by request parameter with data leakage in redirect | Apprentice | [فتح](07-user-id-controlled-by-request-parameter-with-data-leakage-in-redirect.md) | ✅ |
| 08 | 10 | User ID controlled by request parameter with password disclosure | Apprentice | [فتح](08-user-id-controlled-by-request-parameter-with-password-disclosure.md) | ✅ |
| 09 | 11 | Insecure direct object references | Apprentice | [فتح](09-insecure-direct-object-references.md) | ✅ |
| 10 | 5 | URL-based access control can be circumvented | Practitioner | [فتح](10-url-based-access-control-can-be-circumvented.md) | ✅ |
| 11 | 6 | Method-based access control can be circumvented | Practitioner | [فتح](11-method-based-access-control-can-be-circumvented.md) | ✅ |
| 12 | 12 | Multi-step process with no access control on one step | Practitioner | [فتح](12-multi-step-process-with-no-access-control-on-one-step.md) | ✅ |
| 13 | 13 | Referer-based access control | Practitioner | [فتح](13-referer-based-access-control.md) | ✅ |

**الإجمالي:** 13 / 13 — مكتمل

**تسلسل صفحة Access Control:** 1–4 وظائف إدارية وصلاحية من العميل · 5–6 تجاوز على طبقة المنصّة (URL / Method) · 7–11 تصعيد أفقي و IDOR · 12–13 خطوات متعددة و Referer.

---

## خريطة الثغرات في هذا القسم

| النمط | اللابات (رقم الصفحة) | جوهر المشكلة |
|---|---|---|
| Security by obscurity | 1، 2 | الاعتماد على إخفاء المسار بدل التحقق من الصلاحية |
| بيانات صلاحية يتحكم بها العميل | 3، 4 | الدور/الصلاحية مخزّنة عند العميل أو قابلة للتعديل |
| IDOR أفقي | 7، 8، 9، 10، 11 | الوصول لكائن بمُعرّف يتحكم به المستخدم دون فحص الملكية |
| تجاوز على طبقة الـFront-end | 5، 6، 12 | التحقق في طبقة/خطوة/طريقة HTTP واحدة فقط |
| الثقة بسياق الطلب | 13 | الاعتماد على هيدر `Referer` بدل فحص دور المستخدم |

---

## أكبر درس في القسم

> **التحقق من الصلاحية يجب أن يحدث على السيرفر، لكل طلب، ولكل كائن، وفي كل خطوة — وليس على الواجهة أو على اسم المسار أو على طريقة HTTP.**

الأسئلة الثلاثة التي سأطرحها في أي مراجعة كود بعد هذا القسم:
1. من يملك هذا الكائن؟ وهل الكود يتحقق من الملكية أم يفترضها؟
2. هل هذه الصلاحية قادمة من مصدر يتحكم به المستخدم؟
3. هل يوجد مسار آخر (Method / Header / Step / Direct URL) يصل لنفس الوظيفة دون نفس الحماية؟
