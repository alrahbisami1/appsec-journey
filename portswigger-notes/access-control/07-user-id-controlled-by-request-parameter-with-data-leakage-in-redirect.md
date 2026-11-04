# 07 — User ID controlled by request parameter with data leakage in redirect

| | |
|---|---|
| **المصدر / Source** | [PortSwigger Lab](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-data-leakage-in-redirect) |
| **المستوى / Level** | Apprentice |
| **تاريخ التوثيق / Documented** | 2026-11-04 (توثيق بأثر رجعي) |
| **الحالة / Status** | ✅ Solved |

---

## 🇸🇦 النسخة العربية

### ١. وصف الثغرة
عند طلب حساب مستخدم آخر، يعيد التطبيق **302 Redirect** إلى صفحة تسجيل الدخول — لكن جسم الاستجابة (Response Body) يحتوي على بيانات الحساب الحساسة قبل التحويل. المتصفح يتبع التحويل ولا يظهر البيانات، لكن Burp يعرضها.

- **النوع:** Horizontal Privilege Escalation / IDOR + Sensitive Data Exposure in Redirect.
- **بيانات الدخول:** `wiener:peter`.
- **الهدف:** API key الخاص بـ `carlos`.

### ٢. كيف استغلّيتها
1. سجّلت الدخول وفتحت `/my-account?id=wiener`.
2. غيّرت `id` إلى `carlos` مع إيقاف **Follow redirects** في Burp:
   ```http
   GET /my-account?id=carlos HTTP/1.1
   Host: <lab-host>
   Cookie: session=<wiener-session>
   ```
3. كانت الاستجابة:
   ```http
   HTTP/1.1 302 Found
   Location: /login
   ...
   ```
   لكن جسمها تضمّن بيانات حساب كارلوس ومفتاح API.
4. أرسلت المفتاح.

**النتيجة:** تسريب بيانات رغم أن التطبيق "يبدو" أنه منع الوصول عبر التحويل.

### ٣. السبب الجذري (Root Cause)
- **ترتيب خاطئ للتحقق:** جلب البيانات وتجهيز الاستجابة ثم التحقق من الصلاحية.
- الاعتماد على `redirect` كآلية منع، مع ترك البيانات في الجسم.
- عدم مسح الاستجابة عند رفض الوصول.

### ٤. كيف أمنعها في Java Spring Boot
التحقق يجب أن يحدث **قبل** جلب البيانات أو توليد الاستجابة:

```java
@GetMapping("/my-account")
public ResponseEntity<AccountDto> myAccount(
        @RequestParam(required = false) String id,
        Authentication auth) {

    AppUser me = users.findByUsername(auth.getName()).orElseThrow();

    // 1) تحقق أولًا
    if (id != null && !id.equals(me.getUsername())) {
        return ResponseEntity.status(HttpStatus.FORBIDDEN).build(); // بلا جسم
    }

    // 2) اجلب البيانات بعد التحقق فقط
    return ResponseEntity.ok(AccountMapper.toDto(me));
}
```

- استخدم `AccessDeniedException` بدل إعادة التوجيه لصفحة تسجيل الدخول في طلبات API.
- لا تضع بيانات حساسة في جسم استجابة `3xx`.
- **قاعدة مراجعة:** في أي `302` أو `401`، افحص الـBody — هل يحتوي بيانات قبل الرفض؟

### ٥. درس مستفاد
التحويل ليس حماية. إذا كانت البيانات قد وُلدت في الاستجابة، فقد تسرّبت بالفعل. الترتيب الصحيح: **Authorize → Fetch → Serialize**.

### ٦. أسئلة للتفكير
1. لماذا لا يظهر التسريب في المتصفح العادي؟
2. كيف يختلف هذا عن تسريب البيانات في رسائل الأخطاء؟
3. ما الأثر الأمني لو كان الجسم يحتوي بيانات إضافية غير معروضة في الواجهة؟

---

## 🇬🇧 English Version

### 1. Vulnerability
Requesting another user's account returns a 302 redirect to the login page, but the response body still contains the target user's sensitive data. Browsers follow the redirect and hide it; Burp reveals it.

### 2. Exploitation
Log in as `wiener`, request `/my-account?id=carlos` with redirects disabled, read carlos's API key from the 302 response body, and submit it.

### 3. Root Cause
- Authorization happens after the data is fetched and serialized.
- A redirect is used as an access-control mechanism while the body is left intact.
- No response sanitization on denial.

### 4. Prevention in Java Spring Boot
Authorize before fetching data, and return an empty `403` instead of a redirect for API requests.

```java
if (!id.equals(me.getUsername())) {
    return ResponseEntity.status(HttpStatus.FORBIDDEN).build();
}
return ResponseEntity.ok(AccountMapper.toDto(me));
```

### 5. Lesson Learned
A redirect is not protection. If the data was generated, it already leaked. The correct order is Authorize → Fetch → Serialize.

### 6. Reflection Questions
1. Why is the leak invisible in a normal browser?
2. How is this different from leaking data in error messages?
3. What is the impact of extra hidden fields in the response body?
