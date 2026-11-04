# 08 — User ID controlled by request parameter with password disclosure

| | |
|---|---|
| **المصدر / Source** | [PortSwigger Lab](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-password-disclosure) |
| **المستوى / Level** | Apprentice |
| **تاريخ التوثيق / Documented** | 2026-11-04 (توثيق بأثر رجعي) |
| **الحالة / Status** | ✅ Solved |

---

## 🇸🇦 النسخة العربية

### ١. وصف الثغرة
صفحة الحساب تعرض كلمة المرور الحالية **مسبقًا داخل حقل مخفي** (`type="password"`)، وصفحة `/my-account?id=` تسمح بقراءة حساب أي مستخدم. بالجمع بين IDOR وتسريب كلمة المرور، يمكن قراءة كلمة مرور المدير ثم تسجيل الدخول بحسابه.

- **النوع:** Horizontal → Vertical Privilege Escalation (IDOR + Password Disclosure).
- **بيانات الدخول:** `wiener:peter`.
- **الهدف:** كلمة مرور `administrator` ثم حذف `carlos`.

### ٢. كيف استغلّيتها
1. سجّلت الدخول وفتحت `/my-account?id=wiener`.
2. غيّرت `id` إلى `administrator`:
   ```http
   GET /my-account?id=administrator HTTP/1.1
   Host: <lab-host>
   Cookie: session=<wiener-session>
   ```
3. في HTML وجدت:
   ```html
   <input type="password" name="password" value="<admin-password>" readonly>
   ```
4. سجّلت الخروج، دخلت بحساب `administrator` وكلمة المرور المسرّبة.
5. حذفت `carlos` من لوحة الإدارة.

**النتيجة:** اختراق كامل لحساب المدير عبر قراءة كلمة المرور من صفحة مستخدم آخر.

### ٣. السبب الجذري (Root Cause)
- **IDOR** على صفحة الحساب (لا فحص ملكية).
- **تسريب بيانات اعتماد** إلى العميل — كلمة المرور الحالية لا يجب أن تُرسل أبدًا، حتى لو كانت مخفية أو `readonly`.
- الاعتماد على `input type=password` لإخفاء القيمة (الإخفاء البصري ليس أمانًا).

### ٤. كيف أمنعها في Java Spring Boot
**١) لا تُرجع كلمة المرور أبدًا في أي DTO:**

```java
public record AccountDto(String username, String email, String apiKey) {
    // لا يوجد password ولا passwordHash
}
```

```java
@Entity
public class AppUser {
    @JsonIgnore
    private String passwordHash; // hash فقط، وليس كلمة مرور قابلة للعكس
}
```

**٢) استخدم Endpoint منفصلًا لتغيير كلمة المرور، يطلب الكلمة الحالية:**

```java
@PostMapping("/my-account/password")
public void changePassword(@RequestBody @Valid ChangePasswordRequest req,
                           Authentication auth) {
    // تحقق من الكلمة الحالية، ثم خزّن hash جديدًا (BCrypt/Argon2)
}
```

**٣) افحص الملكية في كل قراءة:**

```java
@GetMapping("/my-account")
public AccountDto myAccount(Authentication auth) {
    return accountService.findByUsername(auth.getName());
}
```

- **قاعدة مراجعة:** ابحث عن `password` في أي Response DTO أو HTML Template أو `toString()`.
- **قاعدة تصميم:** كلمة المرور تُكتب فقط، ولا تُقرأ.

### ٥. درس مستفاد
كلمة المرور ليست "حقلًا" في الملف الشخصي. أي نظام يعرضها — حتى مقنّعة — يخالف مبدأ الحد الأدنى من البيانات. الخطورة تتضاعف عند وجود IDOR.

### ٦. أسئلة للتفكير
1. هل تخزين كلمة المرور مشفّرة (Encrypted) أفضل من Hash؟ ولماذا؟
2. كيف يؤثر هذا التسريب على إعادة استخدام كلمة المرور في أنظمة أخرى؟
3. ما الفرق بين `passwordHash` و`password` في DTO من ناحية secure code review؟

---

## 🇬🇧 English Version

### 1. Vulnerability
The account page pre-fills the current password in a masked input, and `/my-account?id=` allows reading any user's account. Chaining IDOR with password disclosure exposes the administrator's password.

### 2. Exploitation
Log in as `wiener`, request `/my-account?id=administrator`, read the password from the HTML input value, log in as `administrator`, and delete `carlos`.

### 3. Root Cause
- IDOR on the account page.
- Credentials are sent to the client, even though the field is masked and read-only.
- UI masking is treated as protection.

### 4. Prevention in Java Spring Boot
Never include passwords or hashes in response DTOs:

```java
public record AccountDto(String username, String email, String apiKey) {}

@Entity
public class AppUser {
    @JsonIgnore
    private String passwordHash;
}
```

Use a dedicated password-change endpoint that verifies the current password and stores a strong hash (BCrypt/Argon2). Enforce ownership on every read.

### 5. Lesson Learned
Passwords are write-only. Any system that returns them — masked or not — violates least data exposure, and IDOR turns it into full account takeover.

### 6. Reflection Questions
1. Is reversible encryption better than hashing for passwords? Why not?
2. How does this leak enable credential stuffing elsewhere?
3. What is the secure-code-review difference between `passwordHash` and `password` in a DTO?
