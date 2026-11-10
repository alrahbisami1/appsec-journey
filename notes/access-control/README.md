# Access Control — قيد الإعادة من الصفر

الترتيب حسب صفحة [Access Control](https://portswigger.net/web-security/access-control) على PortSwigger.

**القاعدة:** لا يُعلَّم ✅ إلا بعد استيفاء شروط [`notes/README.md`](../README.md) الأربعة.

| # | اللاب | تاريخ الحل | الوقت | طريقة الإنجاز | الحالة |
|---|---|---|---|---|---|
| 1 | Unprotected admin functionality | | | | ⬜ |
| 2 | Unprotected admin functionality with unpredictable URL | | | | ⬜ |
| 3 | User role controlled by request parameter | | | | ⬜ |
| 4 | User role can be modified in user profile | | | | ⬜ |
| 5 | URL-based access control can be circumvented | | | | ⬜ |
| 6 | Method-based access control can be circumvented | | | | ⬜ |
| 7 | User ID controlled by request parameter | | | | ⬜ |
| 8 | User ID controlled by request parameter, with unpredictable user IDs | | | | ⬜ |
| 9 | User ID controlled by request parameter with data leakage in redirect | | | | ⬜ |
| 10 | User ID controlled by request parameter with password disclosure | | | | ⬜ |
| 11 | Insecure direct object references | | | | ⬜ |
| 12 | Multi-step process with no access control on one step | | | | ⬜ |
| 13 | Referer-based access control | | | | ⬜ |

**التقدّم:** 0 / 13

---

## خريطة العائلات

| العائلة | اللابات | السؤال الكاشف |
|---|---|---|
| وظيفة بلا فحص | 1، 2، 5، 6، 12، 13 | هل كل مسار إداري يتحقق من الدور؟ وهل يوجد مسار بديل لنفس الوظيفة؟ |
| دور يتحكم به العميل | 3، 4 | هل الدور يُقرأ من Cookie أو Parameter أو جسم الطلب؟ |
| كائن بلا فحص مالك | 7، 8، 9، 10، 11 | هل الكائن يُجلب بمُعرّف من الطلب بلا شرط ملكية؟ |
