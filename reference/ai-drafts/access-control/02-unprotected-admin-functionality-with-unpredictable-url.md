# 02 — Unprotected admin functionality with unpredictable URL

| | |
|---|---|
| **المصدر / Source** | [PortSwigger Lab](https://portswigger.net/web-security/access-control/lab-unprotected-admin-functionality-with-unpredictable-url) |
| **المستوى / Level** | Apprentice |
| **تاريخ التوثيق / Documented** | 2026-11-04 (توثيق بأثر رجعي) |
| **الحالة / Status** | ✅ Solved |

---

## 🇸🇦 النسخة العربية

### ١. وصف الثغرة
لوحة الإدارة موجودة على مسار عشوائي غير قابل للتخمين (مثل `/admin-3f9x2`)، لكن الرابط مكشوف داخل **JavaScript** الخاص بالواجهة. المسار المخفي لا يوفّر أي حماية لأن الكود يُرسل لكل المستخدمين.

- **النوع:** Vertical Privilege Escalation + Information Disclosure.
- **الشرط:** قراءة كود الصفحة (View Source / Burp / DevTools).

### ٢. كيف استغلّيتها
1. فتحت الصفحة الرئيسية واخترت **View source**.
2. وجدت سكربتًا يشبه:
   ```javascript
   var isAdmin = false;
   if (isAdmin) {
       var adminPanelTag = document.createElement('a');
       adminPanelTag.setAttribute('href', '/admin-8m3kq1');
       adminPanelTag.innerText = 'Admin panel';
       document.body.appendChild(adminPanelTag);
   }
   ```
3. زرت `/admin-8m3kq1` مباشرة دون تسجيل دخول.
4. حذفت المستخدم `carlos`.

```http
GET /admin-8m3kq1 HTTP/1.1
Host: <lab-host>
```

**النتيجة:** وصول للوحة الإدارة رغم أن الرابط "غير متوقع".

### ٣. السبب الجذري (Root Cause)
- نفس مشكلة اللاب السابق: **لا يوجد تحقق صلاحية على السيرفر**.
- الاعتماد على **URL غير قابل للتخمين** كآلية أمنية.
- تسريب المسار عبر JavaScript/HTML الذي يصل لكل المستخدمين بغض النظر عن دورهم.

### ٤. كيف أمنعها في Java Spring Boot
```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .anyRequest().authenticated());
```

- لا تُرسل بيانات خاصة بالدور إلى الواجهة قبل التحقق من الدور:
  ```java
  // خطأ: توليد الرابط في الـController وإرساله للجميع
  model.addAttribute("adminUrl", "/admin-8m3kq1");
  ```
- الأفضل: الواجهة تطلب الوظيفة من API محمي، والسيرفر يرفض الطلب غير المصرح به.
- **قاعدة مراجعة:** ابحث عن أي `href` أو `url` أو `endpoint` يُبنى في JS بناءً على `isAdmin` — هذه إشارة تسريب.

### ٥. درس مستفاد
"Unpredictable" ليست كلمة مرور. أي شيء يُرسل إلى المتصفح يجب اعتباره معلومًا للمهاجم. الحماية الحقيقية هي أن السيرفر يرفض الطلب بغض النظر عن معرفة الرابط.

### ٦. أسئلة للتفكير
1. أين أبحث عن مسارات مخفية في تطبيق Angular؟ (main.js، source maps، environment files)
2. هل Source Maps المنشورة في الإنتاج قد تكشف مسارات إدارية؟
3. كيف يختلف هذا اللاب عن اللاب الأول من ناحية الاكتشاف فقط، لا من ناحية الحماية؟

---

## 🇬🇧 English Version

### 1. Vulnerability
The admin panel lives at an unpredictable path, but the path is disclosed in client-side JavaScript. Any user can read it and browse to it directly.

### 2. Exploitation
View the page source, find the generated admin link (e.g. `/admin-8m3kq1`), browse to it unauthenticated, and delete `carlos`.

### 3. Root Cause
- No server-side authorization on the admin route.
- An unpredictable URL is used as the only protection.
- The route leaks to every user through JavaScript.

### 4. Prevention in Java Spring Boot
```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .anyRequest().authenticated());
```
Do not send admin URLs or role-specific UI data to unauthenticated users. Enforce authorization on the API, not on the client.

### 5. Lesson Learned
Unpredictability is not a security control. Anything shipped to the browser is public; the server must authorize every request.

### 6. Reflection Questions
1. Where would you look for hidden routes in an Angular app?
2. Can published source maps leak admin endpoints?
3. What differs between this lab and lab 01 — discovery or protection?
