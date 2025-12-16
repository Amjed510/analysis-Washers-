# 📐 مخططات UML لنظام WashPOS

## نظام نقاط البيع للمغاسل

---

> [!NOTE]
> المخططات التالية مرسومة باستخدام Mermaid وتغطي كلا السيناريوهين

---

## 1. مخطط حالات الاستخدام (Use Case Diagram)

```mermaid
flowchart TB
    subgraph Actors["الجهات الفاعلة"]
        Cashier["🧑‍💼 الكاشير"]
        Supervisor["👨‍💻 المشرف"]
        Admin["👔 المدير"]
        Customer["👤 العميل"]
    end

    subgraph POS["نظام نقاط البيع"]
        UC1["إنشاء طلب جديد"]
        UC2["إضافة خدمات للسلة"]
        UC3["الدفع النقدي"]
        UC4["الدفع بالبطاقة"]
        UC5["طباعة الفاتورة"]
        UC6["تسجيل عميل جديد"]
        UC7["تطبيق برنامج الولاء"]
    end

    subgraph Shift["إدارة الورديات"]
        UC8["فتح الوردية"]
        UC9["إغلاق الوردية"]
        UC10["مطابقة الصندوق"]
        UC11["طباعة X-Report"]
        UC12["طباعة Z-Report"]
    end

    subgraph Admin_Panel["لوحة الإدارة"]
        UC13["إدارة الخدمات"]
        UC14["إدارة الموظفين"]
        UC15["عرض التقارير"]
        UC16["إدارة الأسعار"]
    end

    Cashier --> UC1
    Cashier --> UC2
    Cashier --> UC3
    Cashier --> UC4
    Cashier --> UC5
    Cashier --> UC6
    Cashier --> UC7

    Supervisor --> UC8
    Supervisor --> UC9
    Supervisor --> UC10
    Supervisor --> UC11
    Supervisor --> UC12
    Supervisor --> UC3
    Supervisor --> UC4

    Admin --> UC13
    Admin --> UC14
    Admin --> UC15
    Admin --> UC16

    Customer -.-> UC3
    Customer -.-> UC4
```

---

## 2. مخطط الفئات (Class Diagram)

```mermaid
classDiagram
    class Order {
        +String id
        +String localId
        +String? serverId
        +DateTime createdAt
        +double subtotal
        +double taxAmount
        +double totalAmount
        +double paidAmount
        +double? changeAmount
        +PaymentType paymentType
        +OrderStatus status
        +SyncStatus syncStatus
        +String? customerPhone
        +List~OrderItem~ items
        +calculateTotal()
        +addItem(OrderItem)
        +removeItem(OrderItem)
    }

    class OrderItem {
        +String id
        +String serviceName
        +double price
        +int quantity
        +double getSubtotal()
    }

    class Service {
        +String id
        +String name
        +String nameAr
        +double price
        +String category
        +CarSize carSize
        +bool isActive
    }

    class Customer {
        +String id
        +String phone
        +String? name
        +int loyaltyPoints
        +int totalVisits
        +DateTime? lastVisit
        +addPoints(int)
        +redeemPoints(int)
    }

    class Shift {
        +String id
        +String cashierId
        +DateTime openedAt
        +DateTime? closedAt
        +double openingCash
        +double? closingCash
        +double expectedCash
        +double? difference
        +ShiftStatus status
        +open(double)
        +close(double)
        +calculateExpected()
    }

    class Payment {
        +String id
        +String orderId
        +PaymentType type
        +double amount
        +PaymentStatus status
        +String? transactionId
        +String? idempotencyKey
        +DateTime processedAt
    }

    class User {
        +String id
        +String name
        +String pin
        +UserRole role
        +bool isActive
        +validatePin(String)
    }

    Order "1" --> "*" OrderItem : contains
    Order "1" --> "1" Payment : has
    Order "*" --> "0..1" Customer : belongsTo
    Shift "1" --> "*" Order : contains
    User "1" --> "*" Shift : manages
    OrderItem "*" --> "1" Service : references

    class PaymentType {
        <<enumeration>>
        cash
        card
    }

    class OrderStatus {
        <<enumeration>>
        pending
        completed
        cancelled
        refunded
    }

    class SyncStatus {
        <<enumeration>>
        pending
        syncing
        synced
        failed
        expired
    }

    class ShiftStatus {
        <<enumeration>>
        open
        closed
    }

    class UserRole {
        <<enumeration>>
        cashier
        supervisor
        admin
    }

    class CarSize {
        <<enumeration>>
        small
        medium
        large
        suv
    }
```

---

## 3. مخطط التسلسل - الدفع النقدي (Sequence Diagram - Cash Payment)

```mermaid
sequenceDiagram
    autonumber
    participant C as 🧑‍💼 الكاشير
    participant UI as 📱 واجهة POS
    participant Cart as 🛒 CartController
    participant Pay as 💰 PaymentController
    participant DB as 💾 Isar Database
    participant Print as 🖨️ PrinterService
    participant ZATCA as 📋 ZATCAService

    C->>UI: اختيار الخدمات
    UI->>Cart: addItem(service)
    Cart-->>UI: تحديث السلة

    C->>UI: الضغط على "دفع"
    UI->>Pay: processPayment(order, cash)

    Pay->>Pay: calculateChange(paid, total)
    Pay->>ZATCA: generateQRCode(order)
    ZATCA-->>Pay: qrCodeData

    Pay->>DB: saveOrder(order)
    DB-->>Pay: success

    Pay->>Print: printReceipt(order, qrCode)
    Print-->>Pay: printed

    Pay->>DB: addToSyncQueue(order)
    DB-->>Pay: queued

    Pay-->>UI: PaymentResult.success
    UI-->>C: ✅ تم الدفع + الباقي
```

---

## 4. مخطط التسلسل - الدفع بالبطاقة (Sequence Diagram - Card Payment)

```mermaid
sequenceDiagram
    autonumber
    participant C as 🧑‍💼 الكاشير
    participant UI as 📱 واجهة POS
    participant Pay as 💰 PaymentController
    participant Conn as 📶 ConnectivityService
    participant API as 🌐 API Client
    participant Bank as 🏦 بنك إنماء
    participant DB as 💾 Database
    participant Print as 🖨️ PrinterService

    C->>UI: اختيار "دفع بطاقة"
    UI->>Pay: processCardPayment(order)

    Pay->>Conn: isServerReachable()

    alt غير متصل
        Conn-->>Pay: false
        Pay-->>UI: ❌ لا يوجد اتصال
        UI-->>C: عرض خيار الدفع النقدي
    else متصل
        Conn-->>Pay: true

        Pay->>Pay: generateIdempotencyKey()
        Pay->>UI: عرض "جاري المعالجة..."

        Pay->>API: POST /payment/process
        API->>Bank: Authorization Request

        Note over Bank: معالجة البنك<br/>Timeout: 120s

        alt موافقة
            Bank-->>API: Approved
            API-->>Pay: success + transactionId

            Pay->>DB: saveOrder(order)
            Pay->>Print: printReceipt(order)
            Print-->>Pay: printed

            Pay-->>UI: ✅ تمت الموافقة
            UI-->>C: عرض نتيجة الدفع

        else رفض
            Bank-->>API: Declined
            API-->>Pay: declined + reason
            Pay-->>UI: ❌ مرفوض
            UI-->>C: عرض سبب الرفض + خيارات

        else Timeout
            Bank--xAPI: No Response
            API-->>Pay: TimeoutException
            Pay->>API: GET /payment/status?key=xxx

            alt المعاملة موجودة
                API-->>Pay: order exists
                Pay->>Print: printReceipt(order)
            else غير موجودة
                API-->>Pay: not found
                Pay-->>UI: عرض خيار إعادة المحاولة
            end
        end
    end
```

---

## 5. مخطط التسلسل - المزامنة (Sequence Diagram - Sync Worker)

```mermaid
sequenceDiagram
    autonumber
    participant Timer as ⏰ Timer
    participant Sync as 🔄 SyncWorker
    participant Conn as 📶 Connectivity
    participant DB as 💾 Isar
    participant API as 🌐 API
    participant Server as ☁️ Server

    Timer->>Sync: tick (every 5 min)
    Sync->>Conn: isConnected()

    alt غير متصل
        Conn-->>Sync: false
        Sync-->>Timer: skip sync
    else متصل
        Conn-->>Sync: true

        Sync->>DB: getPendingOrders()
        DB-->>Sync: List<Order>

        loop لكل طلب
            Sync->>Sync: order.syncStatus = syncing
            Sync->>DB: update(order)

            Sync->>API: POST /orders

            alt نجاح
                API->>Server: save order
                Server-->>API: orderId
                API-->>Sync: success

                Sync->>Sync: order.serverId = id
                Sync->>Sync: order.syncStatus = synced
                Sync->>DB: update(order)

            else فشل
                API-->>Sync: error
                Sync->>Sync: order.retryCount++

                alt retryCount >= 3
                    Sync->>Sync: order.syncStatus = failed
                    Sync->>Sync: notifyAdmin()
                else
                    Sync->>Sync: order.syncStatus = pending
                end

                Sync->>DB: update(order)
            end
        end
    end
```

---

## 6. مخطط النشاط - إنشاء طلب (Activity Diagram - Create Order)

```mermaid
flowchart TD
    Start([بداية]) --> SelectServices[اختيار الخدمات]
    SelectServices --> AddToCart{إضافة للسلة؟}

    AddToCart -->|نعم| UpdateCart[تحديث السلة]
    UpdateCart --> MoreServices{خدمات أخرى؟}
    MoreServices -->|نعم| SelectServices
    MoreServices -->|لا| ShowTotal[عرض الإجمالي]

    AddToCart -->|لا| ShowTotal

    ShowTotal --> AskLoyalty{لديه بطاقة ولاء؟}
    AskLoyalty -->|نعم| EnterPhone[إدخال رقم الجوال]
    EnterPhone --> CheckLoyalty[فحص نقاط الولاء]
    CheckLoyalty --> ApplyDiscount{تطبيق خصم؟}
    ApplyDiscount -->|نعم| UpdateTotal[تحديث الإجمالي]
    ApplyDiscount -->|لا| SelectPayment
    UpdateTotal --> SelectPayment

    AskLoyalty -->|لا| SelectPayment[اختيار طريقة الدفع]

    SelectPayment --> PaymentType{نوع الدفع}

    PaymentType -->|نقدي| CashPayment[معالجة نقدي]
    CashPayment --> CalculateChange[حساب الباقي]
    CalculateChange --> SaveLocal[حفظ محلياً]
    SaveLocal --> GenerateQR[توليد QR]
    GenerateQR --> PrintReceipt[طباعة الفاتورة]
    PrintReceipt --> AddToQueue[إضافة لقائمة المزامنة]
    AddToQueue --> End([نهاية])

    PaymentType -->|بطاقة| CheckConnection{متصل؟}
    CheckConnection -->|لا| ShowError[عرض خطأ]
    ShowError --> SuggestCash[اقتراح نقدي]
    SuggestCash --> SelectPayment

    CheckConnection -->|نعم| ProcessCard[معالجة البطاقة]
    ProcessCard --> WaitBank[انتظار البنك]
    WaitBank --> BankResponse{رد البنك}

    BankResponse -->|موافقة| SaveServer[حفظ على السيرفر]
    SaveServer --> GenerateQR

    BankResponse -->|رفض| ShowDecline[عرض سبب الرفض]
    ShowDecline --> RetryOption{إعادة؟}
    RetryOption -->|نعم| SelectPayment
    RetryOption -->|لا| CancelOrder[إلغاء الطلب]
    CancelOrder --> End

    BankResponse -->|Timeout| CheckStatus[التحقق من الحالة]
    CheckStatus --> StatusResult{النتيجة}
    StatusResult -->|موجود| SaveServer
    StatusResult -->|غير موجود| RetryOption
```

---

## 7. مخطط النشاط - إدارة الوردية (Activity Diagram - Shift Management)

```mermaid
flowchart TD
    Start([بداية الدوام]) --> Login[تسجيل دخول بـ PIN]
    Login --> ValidatePIN{PIN صحيح؟}

    ValidatePIN -->|لا| ShowError[خطأ في PIN]
    ShowError --> Login

    ValidatePIN -->|نعم| CheckShift{وردية مفتوحة؟}

    CheckShift -->|نعم| ResumeShift[استئناف الوردية]
    ResumeShift --> POSScreen[شاشة POS]

    CheckShift -->|لا| OpenShift[فتح وردية جديدة]
    OpenShift --> EnterOpeningCash[إدخال المبلغ الافتتاحي]
    EnterOpeningCash --> ConfirmOpening[تأكيد الفتح]
    ConfirmOpening --> POSScreen

    POSScreen --> ProcessOrders[معالجة الطلبات]
    ProcessOrders --> EndDay{نهاية الدوام؟}

    EndDay -->|لا| ProcessOrders
    EndDay -->|نعم| CloseShift[طلب إغلاق الوردية]

    CloseShift --> CountCash[عد النقدية في الصندوق]
    CountCash --> EnterClosingCash[إدخال المبلغ الفعلي]
    EnterClosingCash --> CalculateDiff[حساب الفرق]

    CalculateDiff --> DiffCheck{هناك فرق؟}

    DiffCheck -->|لا| PrintZReport[طباعة Z-Report]

    DiffCheck -->|نعم| ShowDifference[عرض الفرق]
    ShowDifference --> SupervisorApproval{موافقة المشرف؟}

    SupervisorApproval -->|نعم| EnterNote[إدخال ملاحظة]
    EnterNote --> PrintZReport

    SupervisorApproval -->|لا| ReCount[إعادة العد]
    ReCount --> CountCash

    PrintZReport --> CloseSuccess[إغلاق الوردية]
    CloseSuccess --> Logout[تسجيل خروج]
    Logout --> End([نهاية])
```

---

## 8. مخطط المكونات (Component Diagram)

```mermaid
flowchart TB
    subgraph Mobile["📱 Flutter App (SUNMI V2s)"]
        subgraph Presentation["Presentation Layer"]
            Views["Views/Screens"]
            Widgets["Widgets"]
            Controllers["GetX Controllers"]
        end

        subgraph Domain["Domain Layer"]
            UseCases["Use Cases"]
            Entities["Entities"]
            Repos["Repository Interfaces"]
        end

        subgraph Data["Data Layer"]
            RepoImpl["Repository Impl"]
            LocalDS["Local DataSource"]
            RemoteDS["Remote DataSource"]
        end

        subgraph Core["Core Services"]
            IsarDB["Isar Database"]
            DioClient["Dio HTTP Client"]
            Printer["Printer Service"]
            ZATCA["ZATCA Service"]
            Connectivity["Connectivity Service"]
        end
    end

    subgraph Hardware["🔧 Hardware"]
        SUNMI["SUNMI V2s"]
        ThermalPrinter["Thermal Printer"]
        NFCReader["NFC Reader"]
    end

    subgraph Cloud["☁️ Cloud Services"]
        API["REST API"]
        Database["PostgreSQL"]
        PaymentGW["Alinma Gateway"]
    end

    Views --> Controllers
    Controllers --> UseCases
    UseCases --> Repos
    RepoImpl --> LocalDS
    RepoImpl --> RemoteDS

    LocalDS --> IsarDB
    RemoteDS --> DioClient

    Controllers --> Printer
    Controllers --> ZATCA
    Controllers --> Connectivity

    Printer --> ThermalPrinter
    SUNMI --> ThermalPrinter
    SUNMI --> NFCReader

    DioClient --> API
    API --> Database
    API --> PaymentGW

    PaymentGW --> Bank["🏦 Alinma Bank"]
```

---

## 9. مخطط الحالة - حالات الطلب (State Diagram - Order States)

```mermaid
stateDiagram-v2
    [*] --> Draft: إنشاء طلب

    Draft --> Pending: إضافة خدمات
    Pending --> Pending: تعديل السلة

    Pending --> Processing: بدء الدفع

    Processing --> Completed: دفع ناجح
    Processing --> Failed: دفع مرفوض
    Processing --> Pending: إلغاء الدفع

    Failed --> Processing: إعادة المحاولة
    Failed --> Cancelled: إلغاء نهائي

    Completed --> Refunded: استرداد

    Cancelled --> [*]
    Completed --> [*]
    Refunded --> [*]

    note right of Processing
        الدفع النقدي: فوري
        الدفع بالبطاقة: انتظار البنك
    end note
```

---

## 10. مخطط الحالة - حالات المزامنة (State Diagram - Sync States)

```mermaid
stateDiagram-v2
    [*] --> Pending: حفظ محلي

    Pending --> Syncing: بدء المزامنة

    Syncing --> Synced: نجاح
    Syncing --> Failed: فشل (retry < 3)
    Syncing --> Expired: فشل (retry >= 3)

    Failed --> Pending: إعادة المحاولة
    Failed --> Syncing: محاولة تلقائية

    Expired --> ManualReview: مراجعة يدوية
    ManualReview --> Syncing: إعادة المزامنة
    ManualReview --> Archived: أرشفة

    Synced --> [*]
    Archived --> [*]

    note right of Pending
        الحالة الافتراضية
        للطلبات الجديدة Offline
    end note

    note right of Synced
        تمت المزامنة
        مع السيرفر
    end note
```

---

## 11. مخطط التوزيع (Deployment Diagram)

```mermaid
flowchart TB
    subgraph Store["🏪 المغسلة"]
        subgraph Device1["SUNMI V2s #1"]
            App1["WashPOS App"]
            Isar1["Isar DB"]
            Printer1["Thermal Printer"]
        end

        subgraph Device2["SUNMI V2s #2"]
            App2["WashPOS App"]
            Isar2["Isar DB"]
            Printer2["Thermal Printer"]
        end

        Router["WiFi Router"]
    end

    subgraph Cloud["☁️ السحابة"]
        subgraph AWS["AWS / GCP"]
            LB["Load Balancer"]
            API1["API Server 1"]
            API2["API Server 2"]
            DB["PostgreSQL"]
            Redis["Redis Cache"]
        end

        subgraph PaymentInfra["بنية الدفع"]
            AlinmaGW["Alinma Gateway"]
            MadaNet["Mada Network"]
        end
    end

    subgraph Admin["🖥️ الإدارة"]
        WebApp["لوحة التحكم Web"]
        Reports["نظام التقارير"]
    end

    Device1 --> Router
    Device2 --> Router
    Router --> LB

    LB --> API1
    LB --> API2

    API1 --> DB
    API2 --> DB
    API1 --> Redis
    API2 --> Redis

    API1 --> AlinmaGW
    AlinmaGW --> MadaNet

    WebApp --> LB
    Reports --> DB
```

---

## 12. مخطط ERD (Entity Relationship Diagram)

```mermaid
erDiagram
    USER ||--o{ SHIFT : manages
    USER {
        uuid id PK
        string name
        string pin_hash
        enum role
        boolean is_active
        timestamp created_at
    }

    SHIFT ||--o{ ORDER : contains
    SHIFT {
        uuid id PK
        uuid user_id FK
        timestamp opened_at
        timestamp closed_at
        decimal opening_cash
        decimal closing_cash
        decimal expected_cash
        decimal difference
        enum status
    }

    ORDER ||--o{ ORDER_ITEM : has
    ORDER ||--o| PAYMENT : has
    ORDER }o--|| CUSTOMER : belongs_to
    ORDER {
        uuid id PK
        uuid local_id UK
        uuid server_id
        uuid shift_id FK
        uuid customer_id FK
        decimal subtotal
        decimal tax_amount
        decimal total_amount
        decimal paid_amount
        decimal change_amount
        enum payment_type
        enum status
        enum sync_status
        timestamp created_at
    }

    ORDER_ITEM }o--|| SERVICE : references
    ORDER_ITEM {
        uuid id PK
        uuid order_id FK
        uuid service_id FK
        string service_name
        decimal price
        int quantity
    }

    SERVICE {
        uuid id PK
        string name
        string name_ar
        decimal price
        string category
        enum car_size
        boolean is_active
    }

    CUSTOMER {
        uuid id PK
        string phone UK
        string name
        int loyalty_points
        int total_visits
        timestamp last_visit
    }

    PAYMENT {
        uuid id PK
        uuid order_id FK
        enum type
        decimal amount
        enum status
        string transaction_id
        string idempotency_key UK
        timestamp processed_at
    }
```

---

## ملخص المخططات

| #   | المخطط           | الوصف                           |
| --- | ---------------- | ------------------------------- |
| 1   | Use Case         | الجهات الفاعلة وحالات الاستخدام |
| 2   | Class            | الفئات والعلاقات                |
| 3   | Sequence (Cash)  | تسلسل الدفع النقدي              |
| 4   | Sequence (Card)  | تسلسل الدفع بالبطاقة            |
| 5   | Sequence (Sync)  | تسلسل المزامنة                  |
| 6   | Activity (Order) | نشاط إنشاء الطلب                |
| 7   | Activity (Shift) | نشاط إدارة الوردية              |
| 8   | Component        | مكونات النظام                   |
| 9   | State (Order)    | حالات الطلب                     |
| 10  | State (Sync)     | حالات المزامنة                  |
| 11  | Deployment       | توزيع النظام                    |
| 12  | ERD              | قاعدة البيانات                  |
