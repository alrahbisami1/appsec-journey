# 13 — Referer-based access control

| | |
|---|---|
| **المصدر / Source** | [PortSwigger Lab](https://portswigger.net/web-security/access-control/lab-referer-based-access-control) |
| **المستوى / Level** | Practitioner |
| **تاريخ التوثيق / Documented** | 2026-11-04 |
| **الحالة / Status** | ✅ Solved |

---

## 🇸🇦 النسخة العربية

### ١. وصف الثغرة
التطبيق يتحكم في الوصول إلى وظيفة إدارية — ترقية المستخدمين — بناءً على هيدر `Referer`. إذا كان `Referer` يشير إلى `/admin`، يُسمح بالطلب؛ وإلا يُرفض. لا يوجد فحص لدور المستخدم المصادَق عليه.

- **النوع:** Vertical Privilege Escalation — Referer-based access control / Broken Access Control.
- **بيانات الاستكشاف:** `administrator:admin`.
- **بيانات المهاجم:** `wiener:peter`.
- **الهدف:** ترقية `wiener` إلى مدير.

### ٢. كيف استغلّيتها
1. سجّلت الدخول كمدير (`administrator:admin`) وفتحت لوحة الإدارة.
2. رقّيت مستخدمًا (مثل `carlos`)، واعترضت الطلب:
   ```http
   GET /admin-roles?username=carlos&action=upgrade HTTP/1.1
   Host: <lab-host>
   Cookie: session=<administrator-session>
   Referer: https://<lab-host>/admin
   ```
3. أرسلت الطلب إلى **Burp Repeater**.
4. سجّلت الخروج ودخلت بـ`wiener:peter`، ونسخت كوكي جلسته.
5. في Repeater، استبدلت الجلسة بجلسة `wiener` وغيّرت `username` إلى `wiener`، مع **الإبقاء على هيدر `Referer`**:
   ```http
   GET /admin-roles?username=wiener&action=upgrade HTTP/1.1
   Host: <lab-host>
   Cookie: session=<wiener-session>
   Referer: https://<lab-host>/admin
   ```
6. نُفّذت الترقية رغم أن الجلسة لمستخدم عادي.

**إثبات السبب الجذري:**
- فتح `/admin-roles?username=wiener&action=upgrade` مباشرةً في المتصفح → الطلب يُرفض لغياب `Referer`.
- إرسال نفس الطلب بجلسة `wiener` مع `Referer: /admin` → ينجح.
- إرسال الطلب بجلسة المدير بدون `Referer` → يُرفض أيضًا.

> **مدير حقيقي بدون `Referer` يُرفض، ومستخدم عادي مع `Referer` يُقبل** — هذا دليل أن القرار على الهيدر، لا على الصلاحية.

### ٣. السبب الجذري (Root Cause)
- **الخلط بين «مصدر الطلب» و«صلاحية مُرسِل الطلب».**
- السيرفر يثق بهيدر `Referer` — وهو قيمة يتحكم بها العميل بالكامل — كدليل صلاحية.
- لا يوجد فحص لدور المستخدم (`hasRole('ADMIN')`) على الوظيفة الإدارية.
- `Referer` يمكن تزييفه أو حذفه، وقد لا يُرسل أصلًا من بعض المتصفحات/إعدادات الخصوصية.
- حتى لو كان `Referer` صحيحًا 100%، فهو يثبت الصفحة التي بدأ منها الطلب فقط، وليس هوية من نفّذه.

**التصنيف:**
- OWASP A01:2021 — Broken Access Control
- CWE-807 — Reliance on Untrusted Inputs in a Security Decision
- CWE-285 / CWE-863 — Improper / Incorrect Authorization

### ٤. كيف أمنعها في Java Spring Boot
**١) افحص الدور على الوظيفة، وليس على الهيدر:**

```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(auth -> auth
            // لاحظ: /admin-roles لا يبدأ بـ /admin/
            .requestMatchers("/admin/**", "/admin-roles").hasRole("ADMIN")
            .anyRequest().authenticated());
        return http.build();
    }
}
```

```java
@RestController
public class AdminRolesController {

    @GetMapping("/admin-roles") // الأفضل: POST بدل GET لتغيير الحالة
    @PreAuthorize("hasRole('ADMIN')")
    public void changeRole(@RequestParam String username,
                           @RequestParam String action,
                           Authentication auth) {
        audit.log(auth.getName(), username, action);
        roleService.change(username, action);
    }
}
```

**٢) الفاعل من الجلسة، وليس من الطلب:**

```java
String actor = auth.getName(); // من SecurityContext
```

**٣) لا تستخدم `Referer` لقرار الصلاحية:**

```java
// ❌ لا تفعل هذا
String referer = request.getHeader("Referer");
if (referer != null && referer.contains("/admin")) {
    roleService.change(username, action);
}
```

**٤) إن أردت `Referer`/`Origin`، فكطبقة إضافية ضد CSRF فقط — مع CSRF Token، وليس كبديل عن فحص الدور.**

**٥) غيّر العملية إلى `POST` مع CSRF Token:** تغيير الدور عبر `GET` يجعلها عرضة أيضًا لـCSRF وأخطاء الـMethod-based bypass.

### ٥. درس مستفاد
الصلاحية تُبنى على **هوية ودور الفاعل** (`Authentication` / `GrantedAuthority`)، لا على **مصدر الطلب** (`Referer`, `Origin`, `X-Requested-With`). أي قرار أمني يعتمد على قيمة يرسلها العميل هو ثغرة محتملة.

### ٦. أسئلة للتفكير
1. لماذا لا يكفي `Referer` حتى لو كان المتصفح يرسله دائمًا؟
2. ما الفرق بين استخدام `Referer` للحماية من CSRF واستخدامه لتحديد الصلاحية؟
3. لماذا يُعد تغيير الدور عبر `GET` مشكلة إضافية بغض النظر عن `Referer`؟
4. كيف تكتشف هذه الفئة في كود Spring؟ (ابحث عن `getHeader("Referer")` و`contains("/admin")`)

---

## 🇬🇧 English Version

### 1. Vulnerability
Access to the admin role-change function is controlled by the `Referer` header. If the header points to `/admin`, the request is allowed; otherwise it is denied. The user's actual role is never checked.

### 2. Exploitation
Log in as `administrator`, promote a user, and capture:
```http
GET /admin-roles?username=carlos&action=upgrade HTTP/1.1
Referer: https://<lab-host>/admin
```
Send it to Burp Repeater. Log in as `wiener`, replace the session cookie with wiener's, change `username` to `wiener`, keep the `Referer` header, and replay:
```http
GET /admin-roles?username=wiener&action=upgrade HTTP/1.1
Cookie: session=<wiener-session>
Referer: https://<lab-host>/admin
```
The promotion succeeds.

**Proof:** Browsing directly to `/admin-roles?...` fails because no `Referer` is sent. A real admin without `Referer` is also denied, while a normal user with `Referer` is accepted.

### 3. Root Cause
- The server confuses **request provenance** with **the requester's authorization**.
- It trusts `Referer`, a fully client-controlled header, as an authorization signal.
- No role check (`hasRole('ADMIN')`) exists on the admin function.
- Even a genuine `Referer` only proves the originating page, not who sent the request.

**Classification:** OWASP A01 Broken Access Control; CWE-807, CWE-285/CWE-863.

### 4. Prevention in Java Spring Boot
Enforce the role on the function:

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/admin/**", "/admin-roles").hasRole("ADMIN")
    .anyRequest().authenticated());
```

```java
@GetMapping("/admin-roles")
@PreAuthorize("hasRole('ADMIN')")
public void changeRole(@RequestParam String username,
                       @RequestParam String action,
                       Authentication auth) { ... }
```

Never use `Referer`/`Origin` for authorization. Use them only as CSRF defense-in-depth alongside a CSRF token. Prefer `POST` with CSRF protection for state-changing operations.

### 5. Lesson Learned
Authorization must be based on the authenticated principal and its authorities, never on request provenance headers.

### 6. Reflection Questions
1. Why is `Referer` insufficient even when browsers send it?
2. How does using `Referer` for CSRF defense differ from using it for authorization?
3. Why is a state-changing `GET` an additional problem?
4. How would you spot this pattern in a Spring codebase?
