# Secure Code Review — قائمة فحص عملية

> الهدف: تحويل كل ثغرة تعلّمتها إلى سؤال ثابت أراجعه في كود Java / Spring Boot.

---

## ١. نقاط الدخول (Controllers)

- [ ] هل كل Endpoint إداري محمي بـ`@PreAuthorize` أو قاعدة صريحة في `SecurityFilterChain`؟
- [ ] هل يوجد Endpoint يقرأ `id` أو `userId` من الطلب ثم يرجع كائنًا دون فحص ملكية؟
- [ ] هل توجد وظيفة تُغيّر الحالة عبر `GET`؟
- [ ] هل كل عملية متعددة الخطوات تفحص الصلاحية في **كل** خطوة؟
- [ ] هل توجد مسارات بديلة لنفس الوظيفة (v1/v2، trailing slash، case، extension)؟

## ٢. إعدادات الأمان (Security Config)

- [ ] هل القاعدة الافتراضية هي **Deny by default**؟
- [ ] هل `/admin/**` و`/actuator/**` و`/internal/**` محمية؟
- [ ] هل `X-Original-URL` و`X-Rewrite-URL` مرفوضة/محذوفة؟
- [ ] هل `HiddenHttpMethodFilter` معطّل إن لم يكن مطلوبًا؟
- [ ] هل CORS محدّد بدقة (وليس `*` مع credentials)؟
- [ ] هل CSRF مفعّل للمسارات المعتمدة على الجلسات؟

## ٣. منطق العمل (Services)

- [ ] هل هوية المستخدم تُؤخذ من `Authentication` وليس من Request Parameter؟
- [ ] هل تغيير الدور/الصلاحية يمر عبر Endpoint منفصل ومحمي؟
- [ ] هل العمليات الحساسة تُسجّل في Audit Log (من، ماذا، متى)؟
- [ ] هل State Machine للعمليات متعددة الخطوات محفوظة على السيرفر؟

## ٤. البيانات (Repositories / DTOs)

- [ ] هل الاستعلامات تربط الكائن بالمالك؟ (`findByIdAndOwnerId`)
- [ ] هل الـRequest DTO يحتوي فقط الحقول المسموحة (لا Mass Assignment)؟
- [ ] هل كلمة المرور/الـHash مستبعدة من كل Response DTO؟ (`@JsonIgnore`)
- [ ] هل `roleid` أو `permissions` أو `tenantId` قابلة للكتابة من العميل؟
- [ ] هل الملفات الخاصة تُقدّم عبر Controller يفحص الصلاحية (وليس Static Handler)؟

## ٥. أنماط بحث سريعة (Grep)

```text
@RequestBody.*User\b
copyProperties
roleid|roleId|isAdmin|permissions
findById\(
getParameter\("id"\)
X-Original-URL|X-Rewrite-URL
password
confirmed|step|stage
```

---

## سؤال المراجعة الذهبي

> **هل يستطيع مستخدم عادي، بجلسة صحيحة، الوصول إلى هذا الكائن أو تنفيذ هذه الوظيفة عبر تعديل قيمة يتحكم بها؟**

إذا كان الجواب "نعم" أو "ربما" — فهناك ثغرة تحكم وصول محتملة.
