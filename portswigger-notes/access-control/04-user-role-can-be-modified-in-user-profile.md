# 04 — User role can be modified in user profile

| | |
|---|---|
| **المصدر / Source** | [PortSwigger Lab](https://portswigger.net/web-security/access-control/lab-user-role-can-be-modified-in-user-profile) |
| **المستوى / Level** | Apprentice |
| **تاريخ التوثيق / Documented** | 2026-11-04 (توثيق بأثر رجعي) |
| **الحالة / Status** | ✅ Solved |

---

## 🇸🇦 النسخة العربية

### ١. وصف الثغرة
نموذج تحديث الملف الشخصي يقبل الحقول المرسلة كما هي ويطبّقها على كائن المستخدم. بإضافة الحقل `"roleid": 2` إلى طلب تحديث البريد، يمكن للمستخدم ترقية نفسه إلى مدير.

- **النوع:** Mass Assignment / Parameter Tampering → Vertical Privilege Escalation.
- **بيانات الدخول:** `wiener:peter`.
- **الشرط:** لوحة الإدارة متاحة فقط لمن لديه `roleid = 2`.

### ٢. كيف استغلّيتها
1. سجّلت الدخول وذهبت إلى **My account**.
2. فعّلت اعتراض Burp، وغيّرت البريد الإلكتروني.
3. كان الطلب:
   ```http
   POST /my-account/change-email HTTP/1.1
   Content-Type: application/json

   {"email":"wiener@example.com"}
   ```
4. أضفت الحقل:
   ```json
   {"email":"wiener@example.com","roleid":2}
   ```
5. أرسلت الطلب، ثم فتحت `/admin` وحذفت `carlos`.

**النتيجة:** ترقية ذاتية إلى مدير عبر حقل غير مقصود في الـAPI.

### ٣. السبب الجذري (Root Cause)
- **Mass Assignment:** ربط الطلب مباشرة بكيان المستخدم (`User`) دون قائمة حقول مسموحة.
- الحقل `roleid` موجود في الكيان وقابل للكتابة من نفس الـEndpoint.
- لا يوجد فصل بين "تحديث الملف الشخصي" و"تغيير الدور" (Privilege boundary غير موجود).

### ٤. كيف أمنعها في Java Spring Boot
استخدم **DTO** يحتوي فقط على الحقول المسموحة، ولا تربط الـRequest بالكيان مباشرة:

```java
public record UpdateProfileRequest(
    @Email String email
    // لا يوجد role، ولا id، ولا أي حقل صلاحيات
) {}
```

```java
@PostMapping("/my-account/change-email")
public ResponseEntity<Void> updateProfile(
        @RequestBody @Valid UpdateProfileRequest request,
        Authentication auth) {

    AppUser user = users.findByUsername(auth.getName()).orElseThrow();
    user.setEmail(request.email()); // فقط الحقل المسموح
    users.save(user);
    return ResponseEntity.noContent().build();
}
```

- لا تستخدم `BeanUtils.copyProperties(request, user)` مع كائن قادم من العميل.
- اجعل تغيير الدور Endpoint منفصلًا ومحميًا:
  ```java
  @PreAuthorize("hasRole('ADMIN')")
  @PostMapping("/admin/users/{id}/role")
  public void changeRole(@PathVariable Long id, @RequestBody RoleChangeRequest req) { ... }
  ```
- **قاعدة مراجعة:** ابحث عن `@RequestBody User` أو `copyProperties` أو `ModelMapper` مباشرة إلى Entity.

### ٥. درس مستفاد
كل حقل في الـRequest هو سطح هجوم. حتى لو لم يظهر في الواجهة، يمكن إضافته يدويًا. الحل ليس إخفاء الحقل، بل عدم قبوله أصلًا.

### ٦. أسئلة للتفكير
1. كيف يختلف Mass Assignment عن IDOR؟
2. هل `@JsonIgnore` على حقل الدور كافٍ؟ ولماذا قد لا يكون؟
3. ما الحقول الأخرى عالية الخطورة في نظام حقيقي؟ (`isVerified`, `balance`, `tenantId`, `permissions`)

---

## 🇬🇧 English Version

### 1. Vulnerability
The profile update endpoint binds arbitrary JSON fields to the user object. Adding `"roleid": 2` promotes the current user to admin.

### 2. Exploitation
Log in as `wiener`, intercept `POST /my-account/change-email`, add `"roleid":2` to the JSON body, send it, then access `/admin` and delete `carlos`.

### 3. Root Cause
- Mass assignment: the request is bound directly to the `User` entity.
- The `roleid` field is writable through a profile endpoint.
- No separation between profile updates and role changes.

### 4. Prevention in Java Spring Boot
Use a DTO with only allowed fields, and never bind request bodies to entities.

```java
public record UpdateProfileRequest(@Email String email) {}

user.setEmail(request.email());
```

Expose role changes through a separate admin-only endpoint protected by `@PreAuthorize("hasRole('ADMIN')")`.

### 5. Lesson Learned
Every accepted field is attack surface. Do not rely on hiding fields in the UI — reject them on the server.

### 6. Reflection Questions
1. How does mass assignment differ from IDOR?
2. Is `@JsonIgnore` on the role field sufficient? Why or why not?
3. Which other fields are high risk in real systems? (`isVerified`, `balance`, `tenantId`, `permissions`)
