# 01 — Unprotected admin functionality

| | |
|---|---|
| **المصدر / Source** | [PortSwigger Lab](https://portswigger.net/web-security/access-control/lab-unprotected-admin-functionality) |
| **المستوى / Level** | Apprentice |
| **تاريخ التوثيق / Documented** | 2026-11-04 (توثيق بأثر رجعي) |
| **الحالة / Status** | ✅ Solved |

---

## 🇸🇦 النسخة العربية

### ١. وصف الثغرة
لوحة إدارة موجودة على المسار `/administrator-panel` دون أي تحقق من الصلاحية على السيرفر. الصفحة لا تظهر في واجهة المستخدم العادي، لكنها موجودة ومتاحة لأي زائر يعرف الرابط.

- **النوع:** Vertical Privilege Escalation.
- **الشرط:** معرفة المسار — وقد كشفه الملف `robots.txt`.

### ٢. كيف استغلّيتها
1. فتحت `/robots.txt` فوجدت:
   ```text
   User-agent: *
   Disallow: /administrator-panel
   ```
2. زرت `/administrator-panel` مباشرة دون تسجيل دخول.
3. حذفت المستخدم `carlos` من اللوحة.

```http
GET /administrator-panel HTTP/1.1
Host: <lab-host>
```

**النتيجة:** وصول كامل لوظيفة إدارية من دون أي حساب.

### ٣. السبب الجذري (Root Cause)
- الاعتماد على **إخفاء الرابط** (Security by Obscurity) بدل التحقق من الصلاحية.
- لا يوجد أي `authorization check` على السيرفر للوظائف الإدارية.
- `robots.txt` ليس سرًا؛ هو ملف عام يقرأه أي مهاجم.

### ٤. كيف أمنعها في Java Spring Boot
```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                // امنع افتراضيًا، ثم اسمح فقط بما هو عام
                .requestMatchers("/admin/**", "/administrator-panel").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults());
        return http.build();
    }
}
```

```java
@RestController
@RequestMapping("/admin")
@PreAuthorize("hasRole('ADMIN')") // دفاع إضافي حتى لو تغيّر الـFilterChain
public class AdminController {
    // ...
}
```

- **قاعدة تصميم:** Deny by default، وأي استثناء يجب أن يكون معلنًا صراحة.
- **قاعدة مراجعة:** كل Endpoint إداري يجب أن يحمل `@PreAuthorize` أو قاعدة صريحة في `SecurityFilterChain`.

### ٥. درس مستفاد
إذا كانت الوظيفة حساسة، فلا يهم إن كانت مخفية — يجب أن تكون محمية. أول شيء أفحصه في أي تطبيق: هل توجد مسارات إدارية معروفة مثل `/admin` أو `/administrator-panel`؟

### ٦. أسئلة للتفكير
1. كيف أكتشف لوحة إدارة غير مرتبطة بالواجهة؟ (wordlists، robots.txt، JS، sitemap)
2. هل حماية الواجهة فقط (إخفاء الزر) تُعد تحكمًا في الوصول؟ ولماذا لا؟
3. ماذا يحدث لو أضفت مسارًا جديدًا `/admin/v2` ونسيت تسجيله في قواعد الحماية؟

---

## 🇬🇧 English Version

### 1. Vulnerability
An admin panel at `/administrator-panel` is reachable without authentication or authorization. It is not linked from the UI, but the path is disclosed in `robots.txt`.

### 2. Exploitation
Read `/robots.txt`, find the `Disallow: /administrator-panel` entry, browse to the panel unauthenticated, and delete user `carlos`.

### 3. Root Cause
- Security by obscurity instead of server-side authorization.
- No access-control check on administrative functionality.
- `robots.txt` is public and should never be treated as a secret.

### 4. Prevention in Java Spring Boot
```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .anyRequest().authenticated());
```
Also annotate admin controllers with `@PreAuthorize("hasRole('ADMIN')")` as defense in depth, and deny by default.

### 5. Lesson Learned
Hidden ≠ protected. Enumerate admin paths and verify that every one enforces authorization server-side.

### 6. Reflection Questions
1. How would you discover an unlinked admin panel?
2. Why is hiding a UI button not an access control?
3. How do you prevent a newly added admin route from missing protection?
