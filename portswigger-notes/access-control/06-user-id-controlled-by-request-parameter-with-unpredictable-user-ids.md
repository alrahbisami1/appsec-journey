# 06 — User ID controlled by request parameter, with unpredictable user IDs

| | |
|---|---|
| **المصدر / Source** | [PortSwigger Lab](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-unpredictable-user-ids) |
| **المستوى / Level** | Apprentice |
| **تاريخ التوثيق / Documented** | 2026-11-04 (توثيق بأثر رجعي) |
| **الحالة / Status** | ✅ Solved |

---

## 🇸🇦 النسخة العربية

### ١. وصف الثغرة
نفس ثغرة IDOR في اللاب السابق، لكن التطبيق يستخدم **GUID** بدل اسم المستخدم. الـGUID غير قابل للتخمين، لكنه **مكشوف** في صفحات المدونة/التعليقات، لذلك يمكن جمعه ثم استخدامه للوصول لحساب الضحية.

- **النوع:** Horizontal Privilege Escalation / IDOR with unpredictable identifiers.
- **بيانات الدخول:** `wiener:peter`.
- **الهدف:** الحصول على API key الخاص بـ `carlos`.

### ٢. كيف استغلّيتها
1. سجّلت الدخول وفتحت `/my-account?id=wiener`، ولاحظت أن `id` قيمة GUID.
2. بحثت في صفحات المدونة عن أي إشارة إلى `carlos`.
3. وجدت رابط/تعليقًا يحتوي على `userId=<GUID>` الخاص به.
4. استخدمت الـGUID:
   ```http
   GET /my-account?id=<carlos-guid> HTTP/1.1
   Host: <lab-host>
   Cookie: session=<wiener-session>
   ```
5. استخرجت API key وأرسلته.

**النتيجة:** كسر العزل الأفقي رغم أن المُعرّف غير قابل للتخمين.

### ٣. السبب الجذري (Root Cause)
- **IDOR** نفسه: لا فحص ملكية على الكائن.
- الاعتماد على **صعوبة تخمين المُعرّف** كحماية وحيدة.
- تسريب المُعرّفات عبر محتوى عام (Blog authors, comments, API responses).

> القاعدة: `Unguessable ≠ Unauthorized`.

### ٤. كيف أمنعها في Java Spring Boot
```java
@GetMapping("/my-account")
public AccountDto myAccount(Authentication auth) {
    return accountService.findByUsername(auth.getName());
}
```

وعند الحاجة لكائنات عامة (مقالات، تعليقات):

```java
// لا تُرجع الكائن الداخلي كاملًا — استخدم PublicDto
public record AuthorDto(String displayName, String publicSlug) {}
```

- لا تعرض `userId` الداخلي في استجابات عامة إذا لم يكن لازمًا.
- حتى مع UUID، افحص الملكية:
  ```java
  @PreAuthorize("@accountSecurity.isOwner(#id, authentication)")
  ```
- **قاعدة مراجعة:** ابحث عن `UUID` في الـURLs، ثم تتبّع أين يُسرّب هذا الـUUID.

### ٥. درس مستفاد
المُعرّف العشوائي يمنع التخمين فقط، لا يمنع الاستغلال بعد التسريب. أي بيانات تُعرض لمستخدم آخر قد تكون جسرًا لكسر العزل.

### ٦. أسئلة للتفكير
1. أين تُسرّب الـUUIDs عادةً؟ (emails، invoices، public profiles، logs)
2. هل تحويل كل المُعرّفات إلى UUID يعني أن التطبيق آمن؟ ولماذا؟
3. كيف تصمم `PublicDto` يمنع تسريب الحقول الحساسة مستقبلًا؟

---

## 🇬🇧 English Version

### 1. Vulnerability
The same IDOR as lab 05, but user IDs are GUIDs. The GUID is not guessable, yet it leaks in public blog content, so it can be collected and used to access carlos's account.

### 2. Exploitation
Log in as `wiener`, find carlos's GUID in a blog post/comment author link, request `/my-account?id=<carlos-guid>`, and submit his API key.

### 3. Root Cause
- No ownership check on the account object.
- Unpredictability is treated as the only protection.
- Internal identifiers leak through public content.

### 4. Prevention in Java Spring Boot
```java
@GetMapping("/my-account")
public AccountDto myAccount(Authentication auth) {
    return accountService.findByUsername(auth.getName());
}
```
Never expose internal user IDs in public DTOs, and still enforce ownership checks even when using UUIDs.

### 5. Lesson Learned
Unguessable is not unauthorized. Random identifiers only prevent guessing; they do not survive disclosure.

### 6. Reflection Questions
1. Where do UUIDs typically leak?
2. Does switching to UUIDs make an application secure? Why not?
3. How would you design a `PublicDto` that resists future field leaks?
