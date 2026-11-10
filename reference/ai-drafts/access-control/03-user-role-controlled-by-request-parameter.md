# 03 — User role controlled by request parameter

| | |
|---|---|
| **المصدر / Source** | [PortSwigger Lab](https://portswigger.net/web-security/access-control/lab-user-role-controlled-by-request-parameter) |
| **المستوى / Level** | Apprentice |
| **تاريخ التوثيق / Documented** | 2026-11-04 (توثيق بأثر رجعي) |
| **الحالة / Status** | ✅ Solved |

---

## 🇸🇦 النسخة العربية

### ١. وصف الثغرة
التطبيق يحدد صلاحية المستخدم من **Cookie** يحمل قيمة `Admin=false`، ويمكن للمستخدم تعديلها إلى `Admin=true`. أي قرار صلاحية يعتمد على قيمة يتحكم بها العميل هو ثغرة مباشرة.

- **النوع:** Vertical Privilege Escalation — Parameter/Cookie-based access control.
- **بيانات الدخول:** `wiener:peter`.

### ٢. كيف استغلّيتها
1. سجّلت الدخول بـ `wiener:peter`.
2. لاحظت الكوكي:
   ```http
   Cookie: session=...; Admin=false
   ```
3. غيّرتها إلى:
   ```http
   Cookie: session=...; Admin=true
   ```
4. طلبت `/admin` فظهرت اللوحة.
5. حذفت `carlos`.

```http
GET /admin HTTP/1.1
Host: <lab-host>
Cookie: session=<valid>; Admin=true
```

**النتيجة:** صلاحية إدارية كاملة عبر تعديل قيمة في المتصفح.

### ٣. السبب الجذري (Root Cause)
- الدور/الصلاحية مخزّنة في **Cookie غير موقّعة** يتحكم بها العميل.
- السيرفر يثق بالقيمة القادمة من الطلب بدل قراءة الدور من مصدر موثوق (DB/Session).
- لا يوجد ربط بين الجلسة والدور على السيرفر.

> ملاحظة: حتى لو كان الكوكي موقّعًا أو مشفّرًا، الاعتماد عليه لتحديد الدور ليس تصميمًا سليمًا إذا كان بإمكان المستخدم إعادة استخدام كوكي جلسة أخرى أو التلاعب في بنيتها.

### ٤. كيف أمنعها في Java Spring Boot
```java
// الدور يُحمّل من قاعدة البيانات عند المصادقة ويُخزّن في SecurityContext
@Service
public class AppUserDetailsService implements UserDetailsService {

    private final UserRepository users;

    @Override
    public UserDetails loadUserByUsername(String username) {
        AppUser user = users.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException(username));

        return User.withUsername(user.getUsername())
            .password(user.getPasswordHash())
            .authorities("ROLE_" + user.getRole().name()) // من DB
            .build();
    }
}
```

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .anyRequest().authenticated());
```

- لا تقرأ الدور من `Cookie` أو `request.getParameter("role")` أو `localStorage`.
- استخدم `SecurityContextHolder` / `Authentication` فقط.
- **قاعدة مراجعة:** أي `getRole()` أو `isAdmin` يأتي من الطلب = ثغرة محتملة.

### ٥. درس مستفاد
الصلاحية ليست "بيانًا" يرسله العميل، بل حالة يحتفظ بها السيرفر. أول ما أفحصه في الكوكيز: هل تحتوي على `role` أو `isAdmin` أو `permissions`؟

### ٦. أسئلة للتفكير
1. لو كان الكوكي موقّعًا بـ HMAC، هل تصبح الآلية آمنة؟ وما القيود المتبقية؟
2. كيف يختلف تخزين الدور في JWT عن تخزينه في Session من ناحية الإبطال (Revocation)؟
3. ماذا يحدث لو غيّر المستخدم قيمة الدور إلى قيمة غير موجودة؟ هل يفشل التطبيق بأمان؟

---

## 🇬🇧 English Version

### 1. Vulnerability
The application reads the user's role from a client-controlled cookie (`Admin=false`). Changing it to `Admin=true` grants admin access.

### 2. Exploitation
Log in as `wiener:peter`, intercept a request, change `Admin=false` to `Admin=true`, request `/admin`, and delete `carlos`.

### 3. Root Cause
- Authorization depends on a forgeable client-side value.
- The server trusts the cookie instead of loading the role from a trusted store.
- There is no server-side binding between the session and the user's role.

### 4. Prevention in Java Spring Boot
Load authorities from the database in `UserDetailsService`, store them in the `SecurityContext`, and never read roles from cookies or request parameters.

```java
return User.withUsername(user.getUsername())
    .password(user.getPasswordHash())
    .authorities("ROLE_" + user.getRole().name())
    .build();
```

### 5. Lesson Learned
Roles belong on the server. Cookies, hidden fields, and query parameters are user input, not authorization data.

### 6. Reflection Questions
1. Would signing the cookie make this design safe? What remains?
2. How does JWT role storage differ from server-side sessions regarding revocation?
3. What happens if the role value is invalid — fail open or fail closed?
