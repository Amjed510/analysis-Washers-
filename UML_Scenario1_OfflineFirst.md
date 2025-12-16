# 📐 مخططات UML - السيناريو الأول

## نظام WashPOS بهيكلية Offline-First

---

> [!NOTE]
> هذه المخططات خاصة بالسيناريو الأول (Offline-First) مع التركيز على:
>
> - قاعدة بيانات Isar المحلية
> - المزامنة الخلفية
> - العمل بدون إنترنت

---

## 1. مخطط التسلسل - الدفع النقدي (Offline)

```mermaid
sequenceDiagram
    autonumber
    participant K as 🧑‍💼 الكاشير
    participant UI as 📱 واجهة POS
    participant Cart as 🛒 متحكم السلة
    participant Pay as 💰 متحكم الدفع
    participant Isar as 💾 قاعدة Isar
    participant ZATCA as 📋 خدمة ZATCA
    participant Print as 🖨️ الطابعة
    participant Queue as 📤 قائمة المزامنة

    Note over K,Queue: ✅ لا يتطلب إنترنت

    K->>UI: اختيار الخدمات
    UI->>Cart: إضافة للسلة
    Cart-->>UI: تحديث الإجمالي

    K->>UI: الضغط على "دفع نقدي"
    UI->>Pay: معالجة الدفع النقدي

    Pay->>Pay: حساب الباقي
    Note right of Pay: المدفوع - الإجمالي = الباقي

    Pay->>Isar: حفظ الطلب محلياً
    Note right of Isar: syncStatus = pending
    Isar-->>Pay: تم الحفظ ✅

    Pay->>ZATCA: توليد رمز QR
    ZATCA-->>Pay: qrCodeData

    Pay->>Print: طباعة الفاتورة + QR
    Print-->>Pay: تمت الطباعة ✅

    Pay->>Queue: إضافة للمزامنة لاحقاً
    Queue-->>Pay: تمت الإضافة

    Pay-->>UI: نجاح + الباقي
    UI-->>K: ✅ تم - الباقي: X ريال
```

---

## 2. مخطط التسلسل - المزامنة الخلفية (Sync Worker)

```mermaid
sequenceDiagram
    autonumber
    participant Timer as ⏰ المؤقت
    participant Sync as 🔄 عامل المزامنة
    participant Conn as 📶 خدمة الاتصال
    participant Isar as 💾 قاعدة Isar
    participant API as 🌐 عميل API
    participant Server as ☁️ السيرفر

    Timer->>Sync: تنبيه (كل 5 دقائق)
    Sync->>Conn: هل يوجد اتصال؟

    alt غير متصل
        Conn-->>Sync: ❌ لا
        Sync-->>Timer: تخطي المزامنة
    else متصل
        Conn-->>Sync: ✅ نعم

        Sync->>Isar: جلب الطلبات المعلقة
        Note right of Isar: WHERE syncStatus = pending<br/>AND retryCount < 3
        Isar-->>Sync: قائمة الطلبات

        loop لكل طلب معلق
            Sync->>Isar: تحديث: جاري المزامنة

            Sync->>API: POST /orders
            Note right of API: + Idempotency-Key

            alt نجاح
                API->>Server: حفظ الطلب
                Server-->>API: orderId
                API-->>Sync: ✅ نجاح

                Sync->>Isar: تحديث: تمت المزامنة
                Note right of Isar: serverId = xxx<br/>syncStatus = synced

            else فشل
                API--xSync: ❌ خطأ

                Sync->>Sync: retryCount++

                alt المحاولات >= 3
                    Sync->>Isar: تحديث: فشل نهائي
                    Sync->>Sync: إشعار المدير
                else المحاولات < 3
                    Sync->>Isar: تحديث: معلق (إعادة لاحقاً)
                end
            end
        end
    end
```

---

## 3. مخطط التسلسل - الدفع بالبطاقة (يتطلب إنترنت)

```mermaid
sequenceDiagram
    autonumber
    participant K as 🧑‍💼 الكاشير
    participant UI as 📱 واجهة POS
    participant Pay as 💰 متحكم الدفع
    participant Conn as 📶 خدمة الاتصال
    participant Isar as 💾 قاعدة Isar
    participant API as 🌐 عميل API
    participant Bank as 🏦 بنك إنماء
    participant Print as 🖨️ الطابعة

    K->>UI: اختيار "دفع بالبطاقة"
    UI->>Pay: معالجة الدفع بالبطاقة

    Pay->>Conn: هل يوجد اتصال؟

    alt غير متصل
        Conn-->>Pay: ❌ لا
        Pay-->>UI: ⚠️ الدفع بالبطاقة يتطلب إنترنت
        UI-->>K: عرض خيار الدفع النقدي
    else متصل
        Conn-->>Pay: ✅ نعم

        Pay->>Pay: توليد Idempotency Key
        Pay->>UI: جاري المعالجة...

        Pay->>API: POST /payment/card
        API->>Bank: طلب الترخيص

        Note over Bank: معالجة البنك<br/>المهلة: 120 ثانية

        alt موافقة
            Bank-->>API: ✅ Approved
            API-->>Pay: transactionId

            Pay->>Isar: حفظ الطلب
            Note right of Isar: syncStatus = synced<br/>(تمت المزامنة مع البنك)

            Pay->>Print: طباعة الفاتورة
            Print-->>Pay: ✅

            Pay-->>UI: ✅ تمت الموافقة
            UI-->>K: عرض النتيجة

        else رفض
            Bank-->>API: ❌ Declined
            API-->>Pay: سبب الرفض
            Pay-->>UI: ❌ مرفوض
            UI-->>K: عرض السبب + خيارات

        else Timeout
            Bank--xAPI: لا استجابة
            API-->>Pay: TimeoutException

            Pay->>API: GET /payment/status?key=xxx

            alt المعاملة موجودة
                API-->>Pay: ✅ الطلب تم
                Pay->>Print: طباعة
            else غير موجودة
                API-->>Pay: 404
                Pay-->>UI: خيار إعادة المحاولة
            end
        end
    end
```

---

## 4. مخطط المكونات - بنية Offline-First

```mermaid
flowchart TB
    subgraph App["📱 تطبيق WashPOS (Offline-First)"]
        subgraph Presentation["طبقة العرض"]
            Views["الشاشات"]
            Widgets["الودجات"]
            Controllers["متحكمات GetX"]
        end

        subgraph Domain["طبقة المنطق"]
            UseCases["حالات الاستخدام"]
            Entities["الكيانات"]
            RepoInterfaces["واجهات المستودعات"]
        end

        subgraph Data["طبقة البيانات"]
            RepoImpl["تنفيذ المستودعات"]

            subgraph LocalFirst["🟢 المصدر الأساسي (محلي)"]
                Isar["💾 قاعدة Isar"]
                SyncQueue["📤 قائمة المزامنة"]
            end

            subgraph Remote["🔵 المصدر الثانوي (سحابي)"]
                DioClient["🌐 Dio HTTP"]
            end
        end

        subgraph Core["الخدمات الأساسية"]
            SyncWorker["🔄 عامل المزامنة"]
            PrinterSvc["🖨️ خدمة الطباعة"]
            ZATCASvc["📋 خدمة ZATCA"]
            ConnSvc["📶 مراقب الاتصال"]
        end
    end

    subgraph Hardware["🔧 العتاد"]
        SUNMI["جهاز SUNMI V2s"]
        Printer["الطابعة الحرارية"]
    end

    subgraph Cloud["☁️ السحابة (اختياري)"]
        API["واجهة REST API"]
        DB["قاعدة PostgreSQL"]
        PayGW["بوابة بنك إنماء"]
    end

    Views --> Controllers
    Controllers --> UseCases
    UseCases --> RepoInterfaces
    RepoImpl --> Isar
    RepoImpl --> SyncQueue
    RepoImpl -.-> DioClient

    SyncWorker --> SyncQueue
    SyncWorker --> Isar
    SyncWorker --> DioClient
    SyncWorker --> ConnSvc

    Controllers --> PrinterSvc
    Controllers --> ZATCASvc

    PrinterSvc --> Printer
    SUNMI --> Printer

    DioClient -.-> API
    API -.-> DB
    API -.-> PayGW

    style LocalFirst fill:#d4edda,stroke:#28a745
    style Remote fill:#cce5ff,stroke:#007bff
```

---

## 5. مخطط الحالة - حالات الطلب والمزامنة

```mermaid
stateDiagram-v2
    [*] --> مسودة: إنشاء طلب

    state "حالات الطلب (محلي)" as OrderStates {
        مسودة --> قيد_الانتظار: إضافة خدمات
        قيد_الانتظار --> قيد_المعالجة: بدء الدفع

        قيد_المعالجة --> مكتمل_محلياً: دفع نقدي ✅
        قيد_المعالجة --> فشل: دفع مرفوض

        فشل --> قيد_المعالجة: إعادة المحاولة
        فشل --> ملغي: إلغاء

        مكتمل_محلياً --> مسترد: استرداد
    }

    state "حالات المزامنة" as SyncStates {
        [*] --> معلق: بعد اكتمال الطلب محلياً

        معلق --> جاري_المزامنة: توفر الإنترنت

        جاري_المزامنة --> تمت_المزامنة: ✅ نجاح
        جاري_المزامنة --> فشل_المزامنة: ❌ خطأ

        فشل_المزامنة --> معلق: انتظار (محاولات < 3)
        فشل_المزامنة --> منتهي_الصلاحية: محاولات >= 3

        منتهي_الصلاحية --> مراجعة_يدوية: تدخل المشرف
        مراجعة_يدوية --> جاري_المزامنة: إعادة
        مراجعة_يدوية --> أرشفة: حذف
    }

    note right of مكتمل_محلياً
        ✅ الطلب يعمل بالكامل
        حتى قبل المزامنة
    end note

    note right of معلق
        📤 في قائمة الانتظار
        للمزامنة عند توفر الإنترنت
    end note
```

---

## 6. مخطط النشاط - تدفق العمل الكامل

```mermaid
flowchart TD
    Start([🚀 البداية]) --> Login[تسجيل دخول بـ PIN]
    Login --> OpenShift{وردية مفتوحة؟}

    OpenShift -->|لا| EnterCash[إدخال المبلغ الافتتاحي]
    EnterCash --> CreateShift[فتح الوردية في Isar]
    CreateShift --> POSScreen

    OpenShift -->|نعم| POSScreen[شاشة نقاط البيع]

    POSScreen --> SelectServices[اختيار الخدمات]
    SelectServices --> AddToCart[إضافة للسلة]
    AddToCart --> More{خدمات أخرى؟}

    More -->|نعم| SelectServices
    More -->|لا| ShowTotal[عرض الإجمالي + الضريبة]

    ShowTotal --> PayMethod{طريقة الدفع}

    PayMethod -->|نقدي| CashFlow
    PayMethod -->|بطاقة| CardFlow

    subgraph CashFlow["💵 الدفع النقدي (Offline)"]
        EnterPaid[إدخال المبلغ المدفوع]
        CalcChange[حساب الباقي]
        SaveIsar[حفظ في Isar]
        GenQR[توليد QR]
        PrintReceipt[طباعة الفاتورة]
        AddQueue[إضافة لقائمة المزامنة]

        EnterPaid --> CalcChange
        CalcChange --> SaveIsar
        SaveIsar --> GenQR
        GenQR --> PrintReceipt
        PrintReceipt --> AddQueue
    end

    subgraph CardFlow["💳 الدفع بالبطاقة (Online)"]
        CheckConn{متصل؟}
        ShowError[⚠️ يتطلب إنترنت]
        ProcessCard[معالجة البطاقة]
        WaitBank[انتظار البنك]
        BankResult{النتيجة}
        SaveSynced[حفظ (تمت المزامنة)]
        ShowDecline[عرض سبب الرفض]

        CheckConn -->|لا| ShowError
        ShowError --> PayMethod
        CheckConn -->|نعم| ProcessCard
        ProcessCard --> WaitBank
        WaitBank --> BankResult
        BankResult -->|موافقة| SaveSynced
        BankResult -->|رفض| ShowDecline
        ShowDecline --> PayMethod
    end

    AddQueue --> Success[✅ تم الطلب]
    SaveSynced --> PrintReceipt2[طباعة الفاتورة]
    PrintReceipt2 --> Success

    Success --> NewOrder{طلب جديد؟}
    NewOrder -->|نعم| POSScreen
    NewOrder -->|لا| EndShift{إغلاق الوردية؟}

    EndShift -->|لا| POSScreen
    EndShift -->|نعم| CountCash[عد الصندوق]
    CountCash --> Reconcile[مطابقة]
    Reconcile --> PrintZReport[طباعة Z-Report]
    PrintZReport --> CloseShift[إغلاق الوردية]
    CloseShift --> End([🏁 النهاية])

    style CashFlow fill:#d4edda,stroke:#28a745
    style CardFlow fill:#fff3cd,stroke:#ffc107
```

---

## 7. مخطط ERD - قاعدة بيانات Isar المحلية

```mermaid
erDiagram
    ORDER ||--o{ ORDER_ITEM : يحتوي
    ORDER ||--o| PAYMENT : لديه
    ORDER }o--|| SHIFT : ينتمي
    ORDER }o--o| CUSTOMER : مرتبط
    ORDER_ITEM }o--|| SERVICE : يشير
    SHIFT }o--|| USER : يديرها

    ORDER {
        int id PK "معرف تلقائي"
        string localId UK "UUID محلي"
        string serverId "معرف السيرفر (بعد المزامنة)"
        datetime createdAt "تاريخ الإنشاء"
        double subtotal "المجموع الفرعي"
        double taxAmount "الضريبة"
        double totalAmount "الإجمالي"
        double paidAmount "المدفوع"
        double changeAmount "الباقي"
        enum paymentType "نقدي/بطاقة"
        enum status "الحالة"
        enum syncStatus "حالة المزامنة"
        int syncRetryCount "عدد المحاولات"
        string lastSyncError "آخر خطأ"
        string idempotencyKey "مفتاح التكرار"
    }

    ORDER_ITEM {
        int id PK
        string serviceName "اسم الخدمة"
        double price "السعر"
        int quantity "الكمية"
    }

    SERVICE {
        int id PK
        string name "الاسم"
        string nameAr "الاسم بالعربي"
        double price "السعر"
        string category "التصنيف"
        enum carSize "حجم السيارة"
        bool isActive "نشط"
    }

    CUSTOMER {
        int id PK
        string phone UK "رقم الجوال"
        string name "الاسم"
        int loyaltyPoints "نقاط الولاء"
        int totalVisits "عدد الزيارات"
    }

    SHIFT {
        int id PK
        datetime openedAt "وقت الفتح"
        datetime closedAt "وقت الإغلاق"
        double openingCash "المبلغ الافتتاحي"
        double closingCash "المبلغ الختامي"
        double expectedCash "المتوقع"
        double difference "الفرق"
        enum status "الحالة"
        enum syncStatus "حالة المزامنة"
    }

    PAYMENT {
        int id PK
        enum type "النوع"
        double amount "المبلغ"
        enum status "الحالة"
        string transactionId "رقم المعاملة"
        string idempotencyKey UK "مفتاح التكرار"
    }

    USER {
        int id PK
        string name "الاسم"
        string pinHash "PIN مشفر"
        enum role "الدور"
        bool isActive "نشط"
    }
```

---

## ملخص السيناريو الأول

| الميزة             | الوصف              |
| ------------------ | ------------------ |
| **قاعدة البيانات** | Isar (محلية أولاً) |
| **الدفع النقدي**   | ✅ يعمل Offline    |
| **الدفع بالبطاقة** | ⚠️ يتطلب إنترنت    |
| **المزامنة**       | خلفية (كل 5 دقائق) |
| **حل التعارضات**   | Last Write Wins    |
| **الموثوقية**      | عالية جداً         |
