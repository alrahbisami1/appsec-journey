# 09 — Insecure direct object references

| | |
|---|---|
| **المصدر / Source** | [PortSwigger Lab](https://portswigger.net/web-security/access-control/lab-insecure-direct-object-references) |
| **المستوى / Level** | Apprentice |
| **تاريخ التوثيق / Documented** | 2026-11-04 (توثيق بأثر رجعي) |
| **الحالة / Status** | ✅ Solved |

---

## 🇸🇦 النسخة العربية

### ١. وصف الثغرة
التطبيق يخزّن سجلات المحادثة (Chat transcripts) كملفات ثابتة على السيرفر، ويقدّمها عبر روابط مباشرة بأسماء متسلسلة مثل `/download-transcript/2.txt`. لا يوجد تحقق من أن الملف يخص المستخدم الحالي.

- **النوع:** IDOR على ملفات ثابتة (Static File IDOR).
- **الهدف:** العثور على كلمة مرور `carlos` وتسجيل الدخول بحسابه.

### ٢. كيف استغلّيتها
1. فتحت **Live chat** وأرسلت رسالة، ثم استخدمت زر **View transcript**.
2. لاحظت رابط التنزيل:
   ```http
   GET /download-transcript/2.txt HTTP/1.1
   Host: <lab-host>
   Cookie: session=<wiener-session>
   ```
3. غيّرت الرقم يدويًا (`1.txt`, `2.txt`, `3.txt` ...) حتى وجدت محادثة مستخدم آخر.
4. في أحد الملفات وجدت محادثة `carlos` مع الدعم تحتوي كلمة مروره.
5. سجّلت الدخول بحساب `carlos` وكلمة المرور المستخرجة.

**النتيجة:** قراءة ملفات خاصة بمستخدمين آخرين عبر تخمين اسم الملف.

### ٣. السبب الجذري (Root Cause)
- ملفات تُقدّم **مباشرة من السيرفر** دون طبقة تحقق.
- أسماء الملفات **متسلسلة وقابلة للتخمين**.
- لا ربط بين الملف وصاحبه (Ownership غير موجود أصلًا في مسار التقديم).
- بيانات حساسة (كلمة مرور) مخزّنة داخل نص المحادثة.

### ٤. كيف أمنعها في Java Spring Boot
**١) لا تقدّم الملفات من مجلد عام. خزّنها خارج `static/` و`webroot`:**

```java
@GetMapping("/transcripts/{id}")
public ResponseEntity<Resource> download(@PathVariable Long id, Authentication auth) {
    Transcript t = transcripts.findById(id).orElseThrow(NotFoundException::new);

    boolean owner = t.getOwnerUsername().equals(auth.getName());
    boolean admin = auth.getAuthorities().stream()
        .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"));

    if (!owner && !admin) {
        throw new AccessDeniedException("Not your transcript");
    }

    Resource file = new FileSystemResource(transcriptStorage.resolve(t.getFileName()));
    return ResponseEntity.ok()
        .header(HttpHeaders.CONTENT_DISPOSITION, "attachment")
        .body(file);
}
```

**٢) استخدم مُعرّفًا عشوائيًا (UUID) كطبقة إضافية، لكن لا تعتمد عليه وحده.**

**٣) لا تخزّن كلمات المرور أو أسرار في نصوص محادثة.**

- **قاعدة تصميم:** كل ملف يُقدّم عبر Controller يفحص الصلاحية.
- **قاعدة مراجعة:** ابحث عن `ResourceHandler` أو `addResourceLocations` أو `static/` تحتوي ملفات مستخدمين.

### ٥. درس مستفاد
الملفات الثابتة ليست "بيانات"، بل كائنات لها مالك. أي مسار مباشر مثل `/files/123.pdf` يجب أن يمر بنفس منطق الـAuthorization مثل أي API.

### ٦. أسئلة للتفكير
1. هل يكفي تغيير اسم الملف إلى UUID لحل المشكلة؟ ولماذا؟
2. كيف أكتشف ملفات مستخدمين مكشوفة أثناء Recon؟
3. ما مخاطر تخزين الملفات داخل `src/main/resources/static`؟

---

## 🇬🇧 English Version

### 1. Vulnerability
Chat transcripts are stored as static files and served through predictable URLs such as `/download-transcript/2.txt`. There is no ownership check.

### 2. Exploitation
Open Live chat, view your transcript, then increment the file number until you find another user's transcript. Carlos's transcript contains his password; use it to log in as `carlos`.

### 3. Root Cause
- Files are served directly by the web server without authorization.
- File names are sequential and guessable.
- There is no ownership binding, and sensitive data is stored in chat logs.

### 4. Prevention in Java Spring Boot
Store user files outside the web root and serve them through a controller that checks ownership or admin role before returning a `Resource`.

```java
if (!owner && !admin) throw new AccessDeniedException("Not your transcript");
return ResponseEntity.ok(new FileSystemResource(storage.resolve(name)));
```
Use random file names as defense in depth, but never as the only control. Never store credentials in chat transcripts.

### 5. Lesson Learned
Static files are objects with owners. Any direct file path must go through the same authorization logic as an API.

### 6. Reflection Questions
1. Is renaming files to UUIDs sufficient? Why not?
2. How would you discover exposed user files during recon?
3. What are the risks of storing user files under `src/main/resources/static`?
