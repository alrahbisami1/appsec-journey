# 05 — User ID controlled by request parameter

| | |
|---|---|
| **المصدر / Source** | [PortSwigger Lab](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter) |
| **المستوى / Level** | Apprentice |
| **تاريخ التوثيق / Documented** | 2026-11-04 (توثيق بأثر رجعي) |
| **الحالة / Status** | ✅ Solved |

---

## 🇸🇦 النسخة العربية

### ١. وصف الثغرة
صفحة الحساب تقرأ اسم المستخدم من المُعامل `id` وتعرض بياناته دون التحقق من أن الحساب يخص المستخدم الحالي. تغيير `id=wiener` إلى `id=carlos` يعرض حساب كارلوس ومفتاح API الخاص به.

- **النوع:** Horizontal Privilege Escalation / IDOR.
- **بيانات الدخول:** `wiener:peter`.

### ٢. كيف استغلّيتها
1. سجّلت الدخول وفتحت `/my-account?id=wiener`.
2. غيّرت القيمة إلى `carlos`:
   ```http
   GET /my-account?id=carlos HTTP/1.1
   Host: <lab-host>
   Cookie: session=<wiener-session>
   ```
3. ظهرت صفحة حساب كارلوس وفيها `API Key`.
4. أرسلت المفتاح كحل للاب.

**النتيجة:** قراءة بيانات مستخدم آخر باستخدام جلسة صحيحة خاصة بي.

### ٣. السبب الجذري (Root Cause)
- **IDOR:** الكائن يُجلب بمُعرّف يتحكم به المستخدم.
- لا يوجد فحص ملكية (Ownership check) قبل إرجاع البيانات.
- السيرفر يفترض أن المستخدم سيعدّل `id` الخاص به فقط.

### ٤. كيف أمنعها في Java Spring Boot
**القاعدة الأولى:** لا تأخذ هوية المستخدم الحالي من الطلب، خذها من `Authentication`.

```java
@GetMapping("/my-account")
public AccountDto myAccount(Authentication auth) {
    AppUser me = users.findByUsername(auth.getName())
        .orElseThrow(() -> new UsernameNotFoundException(auth.getName()));
    return AccountMapper.toDto(me);
}
```

إذا كان الـEndpoint يجب أن يعمل على مستخدم آخر (مثلًا للمدير)، فافحص الصلاحية والملكية صراحةً:

```java
@GetMapping("/users/{id}/account")
@PreAuthorize("hasRole('ADMIN') or #id == authentication.principal.id")
public AccountDto account(@PathVariable Long id) { ... }
```

أو على مستوى الاستعلام:

```java
// لا تُرجع الكائن إلا إذا كان يخص المستخدم الحالي
Optional<Account> findByIdAndOwnerId(Long id, Long ownerId);
```

- **قاعدة مراجعة:** أي `findById(request.getParameter(...))` ثم `return` مباشرة = IDOR محتمل.
- **قاعدة تصميم:** استخدم مُعرّفات غير قابلة للتخمين كطبقة إضافية، لكن لا تعتبرها بديلًا عن فحص الملكية.

### ٥. درس مستفاد
المستخدم لا "يملك" مُعرّف الحساب في الطلب؛ الجلسة هي التي تحدد من هو. أي معامل يشير إلى كائن يجب أن يُقارن بمالك الجلسة أو بصلاحية إدارية.

### ٦. أسئلة للتفكير
1. كيف أفرّق بين IDOR و Broken Function Level Authorization؟
2. لو كان المُعرّف UUID، هل تختفي الثغرة؟ ولماذا؟
3. كيف أختبر هذا النوع بسرعة على API فيه 50 Endpoint؟

---

## 🇬🇧 English Version

### 1. Vulnerability
The account page takes the username from the `id` parameter and returns that user's data without checking ownership. Changing `id=wiener` to `id=carlos` discloses carlos's API key.

### 2. Exploitation
Log in as `wiener`, request `/my-account?id=carlos`, extract carlos's API key, and submit it.

### 3. Root Cause
- IDOR: an object is fetched using a user-controlled identifier.
- No ownership check before returning the object.
- The server assumes users only modify their own `id`.

### 4. Prevention in Java Spring Boot
Derive the current user from `Authentication`, never from a request parameter.

```java
@GetMapping("/my-account")
public AccountDto myAccount(Authentication auth) {
    return users.findByUsername(auth.getName())
        .map(AccountMapper::toDto)
        .orElseThrow();
}
```

For endpoints that legitimately target other users, enforce `@PreAuthorize` and/or query by owner: `findByIdAndOwnerId(id, ownerId)`.

### 5. Lesson Learned
The session defines identity, not a parameter. Every object reference must be checked against the session owner or an admin role.

### 6. Reflection Questions
1. How does IDOR differ from broken function-level authorization?
2. Would a UUID identifier fix this? Why not?
3. How would you test this pattern quickly across a large API?
