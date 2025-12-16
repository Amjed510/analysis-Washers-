# 📐 مخططات UML - السيناريو الثاني

## نظام WashPOS بهيكلية Cloud-Centric

---

> [!CAUTION]
> هذه المخططات خاصة بالسيناريو الثاني (Cloud-Centric) مع التركيز على:
>
> - الاتصال الإلزامي بالإنترنت
> - عدم وجود قاعدة بيانات محلية
> - المعالجة الفورية على السيرفر

---

## 1. مخطط التسلسل - الدفع (يتطلب إنترنت دائماً)

```mermaid
sequenceDiagram
    autonumber
    participant K as 🧑‍💼 الكاشير
    participant UI as 📱 واجهة POS
    participant Conn as 📶 مراقب الاتصال
    participant Pay as 💰 متحكم الدفع
    participant API as 🌐 عميل API
    participant Server as ☁️ السيرفر
    participant Bank as 🏦 بنك إنماء
    participant Print as 🖨️ الطابعة

    Note over K,Print: ⚠️ يتطلب اتصال بالإنترنت

    K->>UI: الضغط على "دفع"
    UI->>Conn: التحقق من الاتصال

    alt غير متصل
        Conn-->>UI: ❌ لا يوجد اتصال
        UI->>UI: عرض شاشة الحظر
        Note over UI: 📡 لا يوجد اتصال<br/>يرجى التحقق من الإنترنت<br/>[إعادة المحاولة]
        UI-->>K: انتظار عودة الاتصال
    else متصل
        Conn-->>UI: ✅ متصل
        UI->>Pay: معالجة الدفع

        Pay->>Pay: توليد Idempotency Key
        Pay->>UI: جاري المعالجة...

        Pay->>API: POST /orders
        Note right of API: Headers:<br/>Idempotency-Key: xxx<br/>Authorization: Bearer xxx

        API->>Server: إنشاء الطلب

        alt دفع نقدي
            Server->>Server: حفظ الطلب
            Server-->>API: orderId ✅
            API-->>Pay: success

        else دفع بطاقة
            Server->>Bank: طلب الترخيص
            Note over Bank: معالجة البنك<br/>المهلة: 120 ثانية
            Bank-->>Server: Approved/Declined
            Server-->>API: نتيجة الدفع
            API-->>Pay: success/failure
        end

        alt نجاح
            Pay->>Print: طباعة الفاتورة
            Note right of Print: ⚠️ الطباعة فقط<br/>بعد تأكيد السيرفر
            Print-->>Pay: ✅ تمت الطباعة
            Pay-->>UI: ✅ تم الطلب
            UI-->>K: عرض النتيجة

        else فشل
            Pay-->>UI: ❌ فشل
            UI->>UI: عرض خيارات
            Note over UI: [إعادة المحاولة] [إلغاء]
            UI-->>K: انتظار القرار
        end
    end
```

---

## 2. مخطط التسلسل - معالجة انقطاع الاتصال

```mermaid
sequenceDiagram
    autonumber
    participant K as 🧑‍💼 الكاشير
    participant UI as 📱 واجهة POS
    participant Conn as 📶 مراقب الاتصال
    participant Modal as 🚫 شاشة الحظر

    Note over K,Modal: مراقبة مستمرة للاتصال

    loop كل 30 ثانية
        Conn->>Conn: فحص الاتصال
        Conn->>Conn: ping /health
    end

    Conn->>Conn: ⚡ فقد الاتصال
    Conn->>Modal: عرض شاشة الحظر

    activate Modal
    Note over Modal: 📡 لا يوجد اتصال<br/><br/>يرجى التحقق من:<br/>• اتصال WiFi<br/>• شبكة 4G<br/><br/>[⟳ إعادة المحاولة]

    Modal-->>K: منع جميع العمليات

    K->>Modal: الضغط على "إعادة المحاولة"
    Modal->>Conn: فحص الاتصال

    alt لا يزال غير متصل
        Conn-->>Modal: ❌
        Modal-->>K: البقاء في شاشة الحظر
    else عاد الاتصال
        Conn-->>Modal: ✅
        Modal->>Modal: إغلاق
        deactivate Modal
        Conn-->>UI: استعادة الواجهة
        UI-->>K: ✅ يمكنك المتابعة
    end
```

---

## 3. مخطط التسلسل - معالجة Timeout

```mermaid
sequenceDiagram
    autonumber
    participant K as 🧑‍💼 الكاشير
    participant UI as 📱 واجهة POS
    participant Pay as 💰 متحكم الدفع
    participant API as 🌐 عميل API
    participant Server as ☁️ السيرفر

    K->>UI: دفع 100 ريال
    UI->>Pay: processPayment()
    Pay->>Pay: idempotencyKey = "ORD-123"
    Pay->>UI: جاري المعالجة...

    Pay->>API: POST /orders
    Note right of API: Idempotency-Key: ORD-123

    API->>Server: إرسال الطلب

    Note over Server: ═══════════════<br/>⚡ انقطاع الاتصال<br/>═══════════════

    API--xPay: TimeoutException (30s)

    Pay->>UI: ⚠️ انتهت المهلة

    Note over UI: ╔════════════════════════╗<br/>║    فشل الاتصال       ║<br/>║                        ║<br/>║ هل وصل الطلب للسيرفر؟ ║<br/>║                        ║<br/>║ [التحقق] [إلغاء]       ║<br/>╚════════════════════════╝

    K->>UI: الضغط على "التحقق"
    UI->>Pay: checkOrderStatus()

    Pay->>API: GET /orders/status?key=ORD-123

    alt الطلب موجود
        API-->>Pay: 200 + orderId
        Pay->>UI: ✅ الطلب تم بنجاح
        UI->>Pay: طباعة الفاتورة
    else الطلب غير موجود
        API-->>Pay: 404
        Pay->>UI: الطلب لم يصل
        UI->>UI: عرض [إعادة المحاولة]

        K->>UI: إعادة المحاولة
        UI->>Pay: processPayment()
        Note right of Pay: نفس Idempotency Key<br/>= لا تكرار
    end
```

---

## 4. مخطط المكونات - بنية Cloud-Centric

```mermaid
flowchart TB
    subgraph App["📱 تطبيق WashPOS (Cloud-Centric)"]
        subgraph Presentation["طبقة العرض"]
            Views["الشاشات"]
            Widgets["الودجات"]
            Controllers["متحكمات GetX"]
            BlockingScreen["🚫 شاشة الحظر"]
        end

        subgraph Domain["طبقة المنطق"]
            UseCases["حالات الاستخدام"]
            Entities["الكيانات (Models)"]
            RepoInterfaces["واجهات المستودعات"]
        end

        subgraph Data["طبقة البيانات"]
            RepoImpl["تنفيذ المستودعات"]

            subgraph Remote["🔵 المصدر الوحيد (سحابي)"]
                DioClient["🌐 Dio HTTP Client"]
                Interceptors["Interceptors"]
            end
        end

        subgraph Core["الخدمات الأساسية"]
            ConnService["📶 خدمة الاتصال<br/>(CRITICAL)"]
            PrinterSvc["🖨️ خدمة الطباعة"]
            ZATCASvc["📋 خدمة ZATCA"]
        end
    end

    subgraph Hardware["🔧 العتاد"]
        SUNMI["جهاز SUNMI V2s"]
        Printer["الطابعة الحرارية"]
    end

    subgraph Cloud["☁️ السحابة (إلزامي)"]
        LB["Load Balancer"]
        API["واجهة REST API"]
        DB["قاعدة PostgreSQL"]
        PayGW["بوابة بنك إنماء"]
    end

    Views --> Controllers
    Controllers --> UseCases
    UseCases --> RepoInterfaces
    RepoImpl --> DioClient
    DioClient --> Interceptors

    ConnService --> BlockingScreen
    ConnService --> DioClient

    Controllers --> PrinterSvc
    Controllers --> ZATCASvc
    Controllers --> ConnService

    PrinterSvc --> Printer
    SUNMI --> Printer

    DioClient --> LB
    LB --> API
    API --> DB
    API --> PayGW

    style Remote fill:#cce5ff,stroke:#007bff
    style ConnService fill:#fff3cd,stroke:#ffc107
    style BlockingScreen fill:#f8d7da,stroke:#dc3545
```

---

## 5. مخطط الحالة - حالات الطلب

```mermaid
stateDiagram-v2
    [*] --> مسودة: إنشاء طلب

    state "حالات الطلب" as OrderStates {
        مسودة --> قيد_الانتظار: إضافة خدمات
        قيد_الانتظار --> قيد_الانتظار: تعديل السلة

        قيد_الانتظار --> فحص_الاتصال: الضغط على دفع

        فحص_الاتصال --> محظور: ❌ غير متصل
        فحص_الاتصال --> قيد_المعالجة: ✅ متصل

        محظور --> فحص_الاتصال: إعادة المحاولة

        قيد_المعالجة --> انتظار_السيرفر: إرسال للسيرفر

        انتظار_السيرفر --> مكتمل: ✅ نجاح
        انتظار_السيرفر --> فشل: ❌ خطأ
        انتظار_السيرفر --> Timeout: ⏰ انتهاء المهلة

        Timeout --> التحقق: فحص الحالة
        التحقق --> مكتمل: الطلب موجود
        التحقق --> فحص_الاتصال: الطلب غير موجود

        فشل --> فحص_الاتصال: إعادة المحاولة
        فشل --> ملغي: إلغاء

        مكتمل --> مسترد: استرداد
    }

    مكتمل --> [*]
    ملغي --> [*]

    note right of محظور
        🚫 شاشة الحظر
        لا يمكن المتابعة
    end note

    note right of انتظار_السيرفر
        ⏳ جاري المعالجة...
        لا تغلق التطبيق
    end note

    note right of مكتمل
        ✅ تم الحفظ على السيرفر
        يمكن الطباعة الآن
    end note
```

---

## 6. مخطط الحالة - حالات الاتصال

```mermaid
stateDiagram-v2
    [*] --> جاري_الفحص: بدء التطبيق

    state "حالات الاتصال" as ConnStates {
        جاري_الفحص --> متصل: ✅ ping نجح
        جاري_الفحص --> غير_متصل: ❌ ping فشل

        متصل --> غير_متصل: فقد الاتصال
        غير_متصل --> جاري_الفحص: إعادة المحاولة

        متصل --> متصل: ping كل 30 ثانية
    end

    state "تأثير على الواجهة" as UIEffect {
        واجهة_عادية
        شاشة_حظر
    }

    متصل --> واجهة_عادية
    غير_متصل --> شاشة_حظر

    note right of متصل
        🟢 جميع العمليات متاحة
    end note

    note right of غير_متصل
        🔴 جميع العمليات محظورة
        حتى الدفع النقدي!
    end note
```

---

## 7. مخطط النشاط - تدفق العمل الكامل

```mermaid
flowchart TD
    Start([🚀 البداية]) --> CheckConn{فحص الاتصال}

    CheckConn -->|❌ غير متصل| BlockScreen[🚫 شاشة الحظر]
    BlockScreen --> RetryConn{إعادة الفحص}
    RetryConn -->|❌| BlockScreen
    RetryConn -->|✅| Login

    CheckConn -->|✅ متصل| Login[تسجيل دخول بـ PIN]

    Login --> AuthAPI{التحقق من السيرفر}
    AuthAPI -->|❌ فشل| BlockScreen
    AuthAPI -->|✅ نجح| LoadShift[تحميل بيانات الوردية]

    LoadShift --> ShiftOpen{وردية مفتوحة؟}
    ShiftOpen -->|لا| OpenShiftAPI[POST /shifts/open]
    OpenShiftAPI --> POSScreen
    ShiftOpen -->|نعم| POSScreen[شاشة نقاط البيع]

    POSScreen --> SelectServices[اختيار الخدمات]
    SelectServices --> AddToCart[إضافة للسلة]
    AddToCart --> More{خدمات أخرى؟}

    More -->|نعم| SelectServices
    More -->|لا| ShowTotal[عرض الإجمالي]

    ShowTotal --> PayMethod{طريقة الدفع}

    PayMethod --> PreCheck{فحص الاتصال}
    PreCheck -->|❌| BlockScreen
    PreCheck -->|✅| ProcessPayment

    subgraph ProcessPayment["☁️ معالجة الدفع (على السيرفر)"]
        SendAPI[POST /orders]
        WaitServer[انتظار السيرفر]
        ServerResult{النتيجة}

        SendAPI --> WaitServer
        WaitServer --> ServerResult
    end

    ServerResult -->|✅ نجاح| PrintReceipt[طباعة الفاتورة]
    ServerResult -->|❌ فشل| ShowError[عرض الخطأ]
    ServerResult -->|⏰ Timeout| CheckStatus[التحقق من الحالة]

    CheckStatus --> StatusResult{موجود؟}
    StatusResult -->|نعم| PrintReceipt
    StatusResult -->|لا| ShowRetry[خيار إعادة المحاولة]

    ShowError --> RetryOrCancel{القرار}
    ShowRetry --> RetryOrCancel
    RetryOrCancel -->|إعادة| PreCheck
    RetryOrCancel -->|إلغاء| POSScreen

    PrintReceipt --> Success[✅ تم الطلب]

    Success --> NewOrder{طلب جديد؟}
    NewOrder -->|نعم| POSScreen
    NewOrder -->|لا| EndShift{إغلاق الوردية؟}

    EndShift -->|لا| POSScreen
    EndShift -->|نعم| CloseAPI[POST /shifts/close]
    CloseAPI --> PrintZReport[طباعة Z-Report]
    PrintZReport --> End([🏁 النهاية])

    style BlockScreen fill:#f8d7da,stroke:#dc3545
    style ProcessPayment fill:#cce5ff,stroke:#007bff
```

---

## 8. مخطط Dio Interceptors

```mermaid
flowchart LR
    subgraph Request["📤 الطلب الصادر"]
        R1[Request]
        R2[AuthInterceptor]
        R3[IdempotencyInterceptor]
        R4[TimeoutInterceptor]
        R5[LogInterceptor]

        R1 --> R2 --> R3 --> R4 --> R5
    end

    R5 --> Server["☁️ السيرفر"]

    subgraph Response["📥 الرد الوارد"]
        S1[Response]
        S2[ErrorInterceptor]
        S3[ConnectivityInterceptor]
        S4[RetryInterceptor]
        S5[LogInterceptor]

        Server --> S5 --> S4 --> S3 --> S2 --> S1
    end

    subgraph AuthInterceptor["🔐 AuthInterceptor"]
        A1["إضافة Authorization Header"]
        A2["Bearer Token"]
    end

    subgraph IdempotencyInterceptor["🔑 IdempotencyInterceptor"]
        I1["إضافة Idempotency-Key"]
        I2["لطلبات /payment و /orders"]
    end

    subgraph ErrorInterceptor["⚠️ ErrorInterceptor"]
        E1["معالجة 401: تسجيل خروج"]
        E2["معالجة 429: انتظار"]
        E3["معالجة 5xx: إعادة المحاولة"]
    end

    subgraph ConnectivityInterceptor["📶 ConnectivityInterceptor"]
        C1["عند Timeout: فحص الاتصال"]
        C2["عرض شاشة الحظر إذا لزم"]
    end
```

---

## 9. مخطط تدفق البيانات

```mermaid
flowchart TB
    subgraph Client["📱 العميل"]
        UI["واجهة المستخدم"]
        Controller["المتحكم"]
        Dio["Dio Client"]
    end

    subgraph Server["☁️ السيرفر"]
        API["REST API"]
        Auth["خدمة المصادقة"]
        Orders["خدمة الطلبات"]
        Payments["خدمة المدفوعات"]
        DB[("PostgreSQL")]
    end

    subgraph External["🌐 خارجي"]
        Alinma["بنك إنماء"]
    end

    UI -->|1. إنشاء طلب| Controller
    Controller -->|2. POST /orders| Dio
    Dio -->|3. HTTP Request| API
    API -->|4. التحقق| Auth
    Auth -->|5. ✅| API
    API -->|6. حفظ| Orders
    Orders -->|7. INSERT| DB

    Controller -->|8. POST /payment| Dio
    Dio -->|9. HTTP Request| API
    API -->|10. معالجة| Payments
    Payments -->|11. Authorization| Alinma
    Alinma -->|12. Approved| Payments
    Payments -->|13. UPDATE| DB
    Payments -->|14. Response| API
    API -->|15. JSON| Dio
    Dio -->|16. OrderModel| Controller
    Controller -->|17. عرض النتيجة| UI
```

---

## ملخص السيناريو الثاني

| الميزة             | الوصف                 |
| ------------------ | --------------------- |
| **قاعدة البيانات** | لا يوجد (السيرفر فقط) |
| **الدفع النقدي**   | ⚠️ يتطلب إنترنت       |
| **الدفع بالبطاقة** | ⚠️ يتطلب إنترنت       |
| **المزامنة**       | فورية (Real-time)     |
| **شاشة الحظر**     | ✅ عند فقد الاتصال    |
| **الموثوقية**      | تعتمد على الشبكة      |

---

## ⚠️ تحذيرات هامة

> [!CAUTION] > **هذا السيناريو لا يناسب:**
>
> - المغاسل في مواقف سفلية (Basements)
> - المناطق ذات الشبكة الضعيفة
> - المناطق الريفية
> - أي موقع قد ينقطع فيه الإنترنت بشكل متكرر
