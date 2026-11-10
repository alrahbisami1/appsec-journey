# [اسم اللاب]

| | |
|---|---|
| **المصدر / Source** | [PortSwigger Lab](https://portswigger.net/web-security/...) |
| **المستوى / Level** | Apprentice / Practitioner / Expert |
| **تاريخ التوثيق / Documented** | YYYY-MM-DD |
| **الحالة / Status** | ✅ Solved / 🔁 Needs review |

---

## 🇸🇦 النسخة العربية

### ١. وصف الثغرة
- ما نوع الخلل؟ (Vertical / Horizontal / Context-dependent / IDOR ...)
- أين تظهر في التطبيق؟
- ما الشرط الذي يجعلها قابلة للاستغلال؟

### ٢. كيف استغلّيتها
```http
# الطلب الفعلي أو الخطوات المختصرة
GET /example HTTP/1.1
Host: target
Cookie: session=...
```

**النتيجة:** ماذا حصلت عليه؟ (بيانات مستخدم آخر، صلاحية إدارية، ...)

### ٣. السبب الجذري (Root Cause)
- ما الافتراض الخاطئ في التصميم؟
- هل المشكلة في غياب التحقق، أم في التحقق في الطبقة الخاطئة؟
- هل الاعتماد على بيانات يتحكم بها المستخدم؟

### ٤. كيف أمنعها في Java Spring Boot
```java
// كود أو إعداد فعلي
```

- قاعدة إضافية على مستوى التصميم:
- قاعدة على مستوى المراجعة (Checklist):

### ٥. درس مستفاد
- ...

### ٦. أسئلة للتفكير
1. كيف أكتشفها في مشروع حقيقي بدون معرفة مسبقة باللاب؟
2. هل يوجد مسار بديل لنفس الوظيفة لا يمر عبر نفس الحماية؟
3. ما أرخص تغيير في الكود يمنعها جذريًا؟

---

## 🇬🇧 English Version

### 1. Vulnerability
- ...

### 2. Exploitation
```http
...
```

### 3. Root Cause
- ...

### 4. Prevention in Java Spring Boot
```java
...
```

### 5. Lesson Learned
- ...

### 6. Reflection Questions
1. ...
2. ...
