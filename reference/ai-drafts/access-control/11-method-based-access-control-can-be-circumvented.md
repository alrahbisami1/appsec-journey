# 11 — Method-based access control can be circumvented

| | |
|---|---|
| **المصدر / Source** | [PortSwigger Lab](https://portswigger.net/web-security/access-control/lab-method-based-access-control-can-be-circumvented) |
| **المستوى / Level** | Practitioner |
| **تاريخ التوثيق / Documented** | 2026-11-04 (توثيق بأثر رجعي) |
| **الحالة / Status** | ✅ Solved |

---

## 🇸🇦 النسخة العربية

### ١. وصف الثغرة
التطبيق يفرض الصلاحية الإدارية على طلبات `POST /admin-roles` فقط، بينما نفس الوظيفة متاحة عبر `GET` دون تحقق. تغيير طريقة HTTP من `POST` إلى `GET` مع نفس المعاملات يرقّي المستخدم العادي إلى مدير.

- **النوع:** Vertical Privilege Escalation — HTTP Method-based access control bypass.
- **بيانات الدخول:** `wiener:peter` (وللاستكشاف: `administrator:admin`).
- **الهدف:** ترقية `wiener` إلى مدير.

### ٢. كيف استغلّيتها
1. سجّلت الدخول كمدير (`administrator:admin`) لاستكشاف الوظيفة.
2. من لوحة الإدارة، رقّيت مستخدمًا ولاحظت الطلب:
   ```http
   POST /admin-roles HTTP/1.1
   Content-Type: application/x-www-form-urlencoded

   username=wiener&action=upgrade
   ```
3. سجّلت الخروج ودخلت بـ`wiener:peter`.
4. أعدت نفس الطلب لكن بطريقة `GET` ووضعت المعاملات في الـURL:
   ```http
   GET /admin-roles?username=wiener&action=upgrade HTTP/1.1
   Host: <lab-host>
   Cookie: session=<wiener-session>
   ```
5. أصبح `wiener` مديرًا.

**النتيجة:** تنفيذ وظيفة إدارية بواسطة مستخدم عادي عبر تغيير Method فقط.

### ٣. السبب الجذري (Root Cause)
- **التحكم في الوصول مربوط بطريقة HTTP** بدل الوظيفة نفسها.
- الـHandler الخاص بـ`GET` ينفّذ نفس منطق الترقية دون فحص صلاحية.
- لا يوجد تحقق مركزي على مستوى الـService.
- غالبًا توجد قاعدة مثل: `DENY: POST, /admin-roles` فقط.

### ٤. كيف أمنعها في Java Spring Boot
**١) افحص الصلاحية على الوظيفة، لا على الـMethod:**

```java
@RestController
@RequestMapping("/admin-roles")
@PreAuthorize("hasRole('ADMIN')")
public class AdminRolesController {

    @PostMapping
    public void changeRole(@RequestBody RoleChangeRequest req) {
        roleService.changeRole(req);
    }
}
```

**٢) افصل القراءة عن الكتابة، ولا تسمح بتغيير حالة عبر `GET`:**

```java
@GetMapping
@PreAuthorize("hasRole('ADMIN')")
public List<RoleDto> listRoles() { ... } // قراءة فقط

@PostMapping
@PreAuthorize("hasRole('ADMIN')")
public void changeRole(...) { ... }      // كتابة
```

**٣) أضف تحققًا في طبقة الـService كدفاع أخير:**

```java
@PreAuthorize("hasRole('ADMIN')")
public void changeRole(RoleChangeRequest req) { ... }
```

**٤) عطّل `HiddenHttpMethodFilter` إن لم تكن بحاجة إليه، ولا تعتمد على `_method` لتغيير الطريقة.**

- **قاعدة مراجعة:** لكل Endpoint إداري، اختبر كل الـMethods (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`) — وليس فقط الطريقة المتوقعة.
- **قاعدة تصميم:** `GET` يجب أن يكون آمنًا (Safe) ولا يغيّر الحالة (Idempotent read).

### ٥. درس مستفاد
المهاجم لا يحترم "الطريقة المتوقعة". إذا كانت الوظيفة موجودة، فسيجرّب كل الطرق للوصول إليها. الحماية يجب أن تكون على العملية نفسها.

### ٦. أسئلة للتفكير
1. كيف يختلف هذا عن CSRF؟ وما العلاقة بين تغيير الحالة عبر GET وCSRF؟
2. هل `@PreAuthorize` على الـController كافٍ لو كان هناك مسار آخر للـService؟
3. كيف أختبر كل الـMethods بشكل منهجي باستخدام Burp أو ffuf؟

---

## 🇬🇧 English Version

### 1. Vulnerability
Admin authorization is enforced only on `POST /admin-roles`. The same role-change action is reachable via `GET` without any check, so a normal user can promote themselves.

### 2. Exploitation
Observe the admin request as `administrator`:
```http
POST /admin-roles
username=wiener&action=upgrade
```
Then, as `wiener`, send:
```http
GET /admin-roles?username=wiener&action=upgrade
```
The user is promoted to admin.

### 3. Root Cause
- Access control is tied to the HTTP method instead of the function.
- The `GET` handler performs the same state change without authorization.
- No central check at the service layer.

### 4. Prevention in Java Spring Boot
Authorize the function, not the verb:

```java
@RestController
@RequestMapping("/admin-roles")
@PreAuthorize("hasRole('ADMIN')")
public class AdminRolesController {
    @PostMapping public void changeRole(...) { ... }
}
```
Keep `GET` read-only, enforce `@PreAuthorize` at the service layer too, and test every HTTP method against admin endpoints.

### 5. Lesson Learned
Attackers try every method. Protection must cover the operation itself, not the expected request shape.

### 6. Reflection Questions
1. How does this relate to CSRF and state-changing GET requests?
2. Is a controller-level `@PreAuthorize` enough if another path reaches the service?
3. How would you systematically test all HTTP methods?
