# Access Control — فهرس اللابات

المصدر: [PortSwigger — Access Control](https://portswigger.net/web-security/access-control)

> **ملاحظة توثيق:** الـ12 ملاحظة كُتبت **بأثر رجعي** اعتمادًا على الحلول القياسية للابات. راجع كل ملاحظة وعدّل أسماء المستخدمين/الطلبات/التفاصيل لتطابق ما فعلته فعليًا — الهدف أن يكون المستودع سجلًا لعملك أنت، وليس نسخة من حلول PortSwigger.

| # | اللاب | المستوى | الملاحظة | الحالة |
|---|---|---|---|---|
| 01 | Unprotected admin functionality | Apprentice | [فتح](01-unprotected-admin-functionality.md) | ✅ |
| 02 | Unprotected admin functionality with unpredictable URL | Apprentice | [فتح](02-unprotected-admin-functionality-with-unpredictable-url.md) | ✅ |
| 03 | User role controlled by request parameter | Apprentice | [فتح](03-user-role-controlled-by-request-parameter.md) | ✅ |
| 04 | User role can be modified in user profile | Apprentice | [فتح](04-user-role-can-be-modified-in-user-profile.md) | ✅ |
| 05 | User ID controlled by request parameter | Apprentice | [فتح](05-user-id-controlled-by-request-parameter.md) | ✅ |
| 06 | User ID controlled by request parameter, with unpredictable user IDs | Apprentice | [فتح](06-user-id-controlled-by-request-parameter-with-unpredictable-user-ids.md) | ✅ |
| 07 | User ID controlled by request parameter with data leakage in redirect | Apprentice | [فتح](07-user-id-controlled-by-request-parameter-with-data-leakage-in-redirect.md) | ✅ |
| 08 | User ID controlled by request parameter with password disclosure | Apprentice | [فتح](08-user-id-controlled-by-request-parameter-with-password-disclosure.md) | ✅ |
| 09 | Insecure direct object references | Apprentice | [فتح](09-insecure-direct-object-references.md) | ✅ |
| 10 | URL-based access control can be circumvented | Practitioner | [فتح](10-url-based-access-control-can-be-circumvented.md) | ✅ |
| 11 | Method-based access control can be circumvented | Practitioner | [فتح](11-method-based-access-control-can-be-circumvented.md) | ✅ |
| 12 | Multi-step process with no access control on one step | Practitioner | [فتح](12-multi-step-process-with-no-access-control-on-one-step.md) | ✅ |
| 13 | Referer-based access control | Practitioner | — | ⏳ متبقٍ |

**الإجمالي:** 12 / 13

---

## خريطة الثغرات في هذا القسم

| النمط | اللابات | جوهر المشكلة |
|---|---|---|
| Security by obscurity | 01، 02 | الاعتماد على إخفاء المسار بدل التحقق من الصلاحية |
| بيانات صلاحية يتحكم بها العميل | 03، 04 | الدور/الصلاحية مخزّنة عند العميل أو قابلة للتعديل |
| IDOR أفقي | 05، 06، 07، 08، 09 | الوصول لكائن بمُعرّف يتحكم به المستخدم دون فحص الملكية |
| تجاوز على طبقة الـFront-end | 10، 11، 12، 13 | التحقق في طبقة/خطوة/طريقة HTTP واحدة فقط |

---

## أكبر درس في القسم

> **التحقق من الصلاحية يجب أن يحدث على السيرفر، لكل طلب، ولكل كائن، وفي كل خطوة — وليس على الواجهة أو على اسم المسار أو على طريقة HTTP.**

الأسئلة الثلاثة التي سأطرحها في أي مراجعة كود بعد هذا القسم:
1. من يملك هذا الكائن؟ وهل الكود يتحقق من الملكية أم يفترضها؟
2. هل هذه الصلاحية قادمة من مصدر يتحكم به المستخدم؟
3. هل يوجد مسار آخر (Method / Header / Step / Direct URL) يصل لنفس الوظيفة دون نفس الحماية؟
