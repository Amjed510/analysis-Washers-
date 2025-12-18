# 🔧 التحليل التقني للمطورين - نظام WashPOS

> [!IMPORTANT]
> **التركيز:** Implementation-Ready Analysis  
> **الافتراض:** الباك-إند و الـAPI جاهزين (منتجات، فواتير، سندات)  
> **الهدف:** دليل تقني عملي للتنفيذ السريع

---

## 1. المعمارية المقترحة (Architecture Overview)

### 1.1 مخطط المكونات الرئيسية

```
┌─────────────────────────────────────────────────────────────┐
│                    Mobile App Architecture                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────────────────────────────────────────────────┐  │
│   │                    UI Layer                          │  │
│   │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │  │
│   │  │ Service │ │ Car Size│ │ Payment │ │ Receipt │   │  │
│   │  │ Screen  │ │ Sheet   │ │ Screen  │ │ Screen  │   │  │
│   │  └─────────┘ └─────────┘ └─────────┘ └─────────┘   │  │
│   └─────────────────────────────────────────────────────┘  │
│                           │                                 │
│                           ▼                                 │
│   ┌─────────────────────────────────────────────────────┐  │
│   │                 Business Layer                       │  │
│   │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │  │
│   │  │ POS        │ │ Payment     │ │ Sync        │   │  │
│   │  │ Controller │ │ Controller  │ │ Controller  │   │  │
│   │  └─────────────┘ └─────────────┘ └─────────────┘   │  │
│   └─────────────────────────────────────────────────────┘  │
│                           │                                 │
│                           ▼                                 │
│   ┌─────────────────────────────────────────────────────┐  │
│   │                  Data Layer                          │  │
│   │  ┌───────────┐ ┌───────────┐ ┌───────────┐         │  │
│   │  │ Local DB  │ │ Offline   │ │ API       │         │  │
│   │  │ (SQLite/  │ │ Queue     │ │ Client    │         │  │
│   │  │  Isar)    │ │           │ │ (Dio)     │         │  │
│   │  └───────────┘ └───────────┘ └───────────┘         │  │
│   └─────────────────────────────────────────────────────┘  │
│                           │                                 │
│                           ▼                                 │
│   ┌─────────────────────────────────────────────────────┐  │
│   │                Hardware Layer                        │  │
│   │  ┌───────────────┐         ┌───────────────┐        │  │
│   │  │ Payment SDK   │         │ Print Service │        │  │
│   │  │ (Terminal)    │         │ (Thermal)     │        │  │
│   │  └───────────────┘         └───────────────┘        │  │
│   └─────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 هيكل المجلدات المقترح

```
lib/
├── main.dart
├── app/
│   ├── bindings/
│   │   └── initial_binding.dart
│   ├── routes/
│   │   ├── app_pages.dart
│   │   └── app_routes.dart
│   └── themes/
│       └── app_theme.dart
│
├── core/
│   ├── database/
│   │   ├── local_db.dart
│   │   └── schemas/
│   │       ├── local_transaction.dart
│   │       └── cached_product.dart
│   │
│   ├── network/
│   │   ├── api_client.dart
│   │   ├── api_endpoints.dart
│   │   └── connectivity_service.dart
│   │
│   └── services/
│       ├── payment_terminal_service.dart
│       ├── printer_service.dart
│       ├── sync_service.dart
│       └── offline_queue_service.dart
│
├── data/
│   ├── models/
│   │   ├── product_model.dart
│   │   ├── invoice_model.dart
│   │   ├── receipt_model.dart
│   │   └── payment_result_model.dart
│   │
│   └── repositories/
│       ├── products_repository.dart
│       ├── invoice_repository.dart
│       └── receipt_repository.dart
│
├── features/
│   ├── auth/
│   │   ├── controllers/
│   │   └── views/
│   │
│   ├── pos/
│   │   ├── controllers/
│   │   │   ├── pos_controller.dart
│   │   │   └── cart_controller.dart
│   │   ├── views/
│   │   │   ├── service_selection_screen.dart
│   │   │   ├── car_size_sheet.dart
│   │   │   └── widgets/
│   │   │       ├── service_card.dart
│   │   │       └── size_button.dart
│   │   └── bindings/
│   │       └── pos_binding.dart
│   │
│   └── payment/
│       ├── controllers/
│       │   └── payment_controller.dart
│       ├── views/
│       │   ├── payment_screen.dart
│       │   ├── processing_overlay.dart
│       │   └── success_screen.dart
│       └── bindings/
│           └── payment_binding.dart
│
└── widgets/
    ├── connectivity_indicator.dart
    ├── loading_overlay.dart
    └── error_dialog.dart
```

---

## 2. التعامل مع المنتجات (Products Handling)

### 2.1 جلب الخدمات من الـ API

الخدمات تُجلب من `invProducts` ويتم فلترتها حسب:
- **ServiceType**: نوع الخدمة (غسيل، تلميع، إضافات)
- **CarSize**: حجم السيارة (صغير، متوسط، كبير)

```dart
class ProductsRepository {
  final ApiClient _api;
  
  /// جلب جميع الخدمات
  Future<List<Product>> getAllServices() async {
    final response = await _api.get('/invProducts');
    return (response.data as List)
        .map((json) => Product.fromJson(json))
        .toList();
  }
  
  /// فلترة حسب نوع الخدمة
  List<Product> filterByServiceType(
    List<Product> products, 
    ServiceType type
  ) {
    return products.where((p) => p.serviceType == type).toList();
  }
  
  /// فلترة حسب حجم السيارة
  List<Product> filterByCarSize(
    List<Product> products, 
    CarSize size
  ) {
    return products.where((p) => p.carSize == size).toList();
  }
  
  /// الحصول على السعر للخدمة والحجم المحددين
  double getPriceForService(String serviceName, CarSize size) {
    final product = _products.firstWhere(
      (p) => p.name == serviceName && p.carSize == size,
      orElse: () => throw ProductNotFoundException(),
    );
    return product.price;
  }
}
```

### 2.2 نموذج المنتج

```dart
@JsonSerializable()
class Product {
  final String id;
  final String name;
  final String nameAr;
  final double price;
  
  @JsonKey(name: 'service_type')
  final ServiceType serviceType;
  
  @JsonKey(name: 'car_size')
  final CarSize? carSize;
  
  final String? icon;
  final bool isActive;
  
  Product({
    required this.id,
    required this.name,
    required this.nameAr,
    required this.price,
    required this.serviceType,
    this.carSize,
    this.icon,
    this.isActive = true,
  });
  
  factory Product.fromJson(Map<String, dynamic> json) => 
      _$ProductFromJson(json);
}

enum ServiceType {
  @JsonValue('wash')
  wash,           // غسيل
  
  @JsonValue('steam')
  steam,          // بخار
  
  @JsonValue('express')
  express,        // اكسبرس
  
  @JsonValue('polish')
  polish,         // تلميع
  
  @JsonValue('addon')
  addon,          // إضافات
}

enum CarSize {
  @JsonValue('small')
  small,          // صغيرة
  
  @JsonValue('medium')
  medium,         // متوسطة
  
  @JsonValue('large')
  large,          // كبيرة
}
```

---

## 3. الأوفلاين (Offline-First Strategy)

> [!CAUTION]
> **نقطة مفصلية:** يجب أن يعمل التطبيق بدون إنترنت للمدفوعات النقدية

### 3.1 ما يُخزَّن محلياً

| البيانات | السبب | الحذف بعد |
|----------|-------|-----------|
| الفواتير | إنشاء فاتورة بدون إنترنت | بعد المزامنة الناجحة + 7 أيام |
| السندات | إثبات الدفع | بعد المزامنة الناجحة + 7 أيام |
| نتائج الدفع | للمرجعية | بعد المزامنة الناجحة + 30 يوم |
| حالة الطباعة | إعادة الطباعة | بعد الطباعة الناجحة |
| المنتجات | عرض الخدمات | تحديث يومي |

### 3.2 جدول المعاملات المحلية

```dart
@collection
class LocalTransaction {
  Id id = Isar.autoIncrement;
  
  /// معرف محلي فريد (UUID)
  @Index(unique: true)
  String localId = const Uuid().v4();
  
  /// معرف السيرفر (يُملأ بعد المزامنة)
  String? serverId;
  
  /// بيانات الفاتورة (JSON)
  String invoiceData;
  
  /// بيانات السند (JSON)
  String? receiptData;
  
  /// نوع الدفع
  @enumerated
  PaymentType paymentType = PaymentType.cash;
  
  /// حالة الدفع
  @enumerated
  PaymentStatus paymentStatus = PaymentStatus.pending;
  
  /// حالة المزامنة
  @enumerated
  SyncStatus syncStatus = SyncStatus.pending;
  
  /// حالة الطباعة
  @enumerated
  PrintStatus printStatus = PrintStatus.pending;
  
  /// مرجع البنك (للدفع بالشبكة)
  String? bankReference;
  
  /// عدد محاولات المزامنة
  int syncRetryCount = 0;
  
  /// آخر خطأ في المزامنة
  String? lastSyncError;
  
  /// تاريخ الإنشاء
  DateTime createdAt = DateTime.now();
  
  /// تاريخ المزامنة
  DateTime? syncedAt;
  
  /// تاريخ الطباعة
  DateTime? printedAt;
}

enum PaymentType { cash, card }
enum PaymentStatus { pending, completed, failed, refunded }
enum SyncStatus { pending, syncing, synced, failed, expired }
enum PrintStatus { pending, printing, printed, failed }
```

### 3.3 خدمة الـ Offline Queue

```dart
class OfflineQueueService extends GetxService {
  final LocalDatabase _db;
  final ApiClient _api;
  final ConnectivityService _connectivity;
  
  /// إضافة معاملة للقائمة
  Future<void> enqueue(LocalTransaction transaction) async {
    await _db.transactions.put(transaction);
    
    // محاولة المزامنة الفورية إذا متصل
    if (_connectivity.isConnected.value) {
      await _processQueue();
    }
  }
  
  /// الحصول على المعاملات المعلقة
  Future<List<LocalTransaction>> getPendingTransactions() async {
    return await _db.transactions
        .filter()
        .syncStatusEqualTo(SyncStatus.pending)
        .findAll();
  }
  
  /// معالجة قائمة الانتظار
  Future<void> _processQueue() async {
    final pending = await getPendingTransactions();
    
    for (final tx in pending) {
      await _syncTransaction(tx);
    }
  }
  
  /// مزامنة معاملة واحدة
  Future<void> _syncTransaction(LocalTransaction tx) async {
    try {
      tx.syncStatus = SyncStatus.syncing;
      await _db.transactions.put(tx);
      
      // 1. إنشاء الفاتورة على السيرفر
      final invoiceResponse = await _api.post(
        '/invoices',
        data: jsonDecode(tx.invoiceData),
        options: Options(
          headers: {'Idempotency-Key': tx.localId},
        ),
      );
      
      tx.serverId = invoiceResponse.data['id'];
      
      // 2. إنشاء السند إذا موجود
      if (tx.receiptData != null) {
        await _api.post(
          '/receipts',
          data: jsonDecode(tx.receiptData!),
          options: Options(
            headers: {'Idempotency-Key': '${tx.localId}-receipt'},
          ),
        );
      }
      
      tx.syncStatus = SyncStatus.synced;
      tx.syncedAt = DateTime.now();
      
    } catch (e) {
      tx.syncRetryCount++;
      tx.lastSyncError = e.toString();
      
      if (tx.syncRetryCount >= 5) {
        tx.syncStatus = SyncStatus.failed;
      } else {
        tx.syncStatus = SyncStatus.pending;
      }
    }
    
    await _db.transactions.put(tx);
  }
}
```

---

## 4. آلية الـ Sync (Synchronization)

### 4.1 متى تتم المزامنة؟

| الحدث | الإجراء |
|-------|---------|
| توفر الإنترنت | مزامنة فورية |
| فتح التطبيق | مزامنة المعلق |
| كل 5 دقائق | مزامنة خلفية |
| طلب المستخدم | مزامنة يدوية |

### 4.2 حالات المزامنة

```
Pending ──► Syncing ──► Synced
              │
              └──► Failed (بعد 5 محاولات)
```

### 4.3 خدمة المزامنة

```dart
class SyncService extends GetxService {
  final OfflineQueueService _queue;
  final ConnectivityService _connectivity;
  
  Timer? _periodicSync;
  
  @override
  void onInit() {
    super.onInit();
    _startPeriodicSync();
    _listenToConnectivity();
  }
  
  /// بدء المزامنة الدورية
  void _startPeriodicSync() {
    _periodicSync = Timer.periodic(
      const Duration(minutes: 5),
      (_) => sync(),
    );
  }
  
  /// الاستماع لتغير الاتصال
  void _listenToConnectivity() {
    _connectivity.isConnected.listen((connected) {
      if (connected) {
        sync();
      }
    });
  }
  
  /// تنفيذ المزامنة
  Future<SyncResult> sync() async {
    if (!_connectivity.isConnected.value) {
      return SyncResult.noConnection();
    }
    
    final pending = await _queue.getPendingTransactions();
    
    if (pending.isEmpty) {
      return SyncResult.nothingToSync();
    }
    
    int synced = 0;
    int failed = 0;
    
    for (final tx in pending) {
      try {
        await _queue._syncTransaction(tx);
        synced++;
      } catch (e) {
        failed++;
      }
    }
    
    return SyncResult(
      totalProcessed: pending.length,
      synced: synced,
      failed: failed,
    );
  }
  
  @override
  void onClose() {
    _periodicSync?.cancel();
    super.onClose();
  }
}
```

### 4.4 قواعد المزامنة الهامة

> [!WARNING]
> **قواعد حرجة:**
> - لا يتم إيقاف المستخدم أثناء الـSync
> - لا إعادة إنشاء فاتورة مكررة (Idempotency Key)
> - المعاملات القديمة (> 72 ساعة) تُميز كـ expired
> - إشعار المدير عند الفشل المتكرر

---

## 5. الربط مع جهاز الدفع (Payment Terminal)

### 5.1 أنواع الاتصال المدعومة

| الاتصال | الاستخدام |
|---------|----------|
| USB | SUNMI المدمج |
| Bluetooth | أجهزة خارجية |
| LAN | نفس الشبكة |

### 5.2 Payment Terminal Service

```dart
class PaymentTerminalService extends GetxService {
  /// تهيئة الجهاز
  Future<bool> initialize() async {
    try {
      // للأجهزة SUNMI
      final isReady = await SunmiPrinter.bindingPrinter();
      return isReady ?? false;
    } catch (e) {
      debugPrint('Terminal init error: $e');
      return false;
    }
  }
  
  /// معالجة الدفع بالبطاقة
  Future<TerminalResult> processCardPayment({
    required double amount,
    required String invoiceId,
  }) async {
    // إرسال طلب الدفع للجهاز
    final request = PaymentRequest(
      amount: (amount * 100).round(), // المبلغ بالهللات
      invoiceId: invoiceId,
      merchantId: _merchantId,
      terminalId: _terminalId,
    );
    
    try {
      // انتظار النتيجة من الجهاز
      final response = await _terminal.processPayment(
        request.toJson(),
        timeout: const Duration(seconds: 120),
      );
      
      if (response['status'] == 'SUCCESS') {
        return TerminalResult.success(
          refNo: response['refNo'],
          authCode: response['authCode'],
          cardType: response['cardType'],
        );
      } else {
        return TerminalResult.declined(
          reason: response['declineReason'],
        );
      }
    } on TimeoutException {
      return TerminalResult.timeout();
    } catch (e) {
      return TerminalResult.error(e.toString());
    }
  }
}

/// طلب الدفع
@JsonSerializable()
class PaymentRequest {
  final int amount;          // المبلغ بالهللات
  final String invoiceId;    // معرف الفاتورة المحلي
  final String merchantId;   // معرف التاجر
  final String terminalId;   // معرف الجهاز
  
  Map<String, dynamic> toJson() => {
    'amount': amount,
    'invoiceId': invoiceId,
    'merchantId': merchantId,
    'terminalId': terminalId,
  };
}

/// نتيجة الدفع
class TerminalResult {
  final TerminalStatus status;
  final String? refNo;
  final String? authCode;
  final String? cardType;
  final String? reason;
  
  // Constructors...
}

enum TerminalStatus { 
  success, 
  declined, 
  timeout, 
  cancelled, 
  error 
}
```

---

## 6. منطق الدفع الكامل (Payment Flow Logic)

### 6.1 الدفع النقدي

```dart
class PaymentController extends GetxController {
  
  /// معالجة الدفع النقدي
  Future<PaymentResult> processCashPayment({
    required List<CartItem> items,
    required double totalAmount,
  }) async {
    try {
      // 1. إنشاء الفاتورة محلياً
      final invoice = _createInvoice(items, totalAmount);
      final invoiceJson = invoice.toJson();
      
      // 2. إنشاء سند القبض
      final receipt = _createCashReceipt(invoice.localId, totalAmount);
      final receiptJson = receipt.toJson();
      
      // 3. حفظ المعاملة محلياً
      final transaction = LocalTransaction()
        ..invoiceData = jsonEncode(invoiceJson)
        ..receiptData = jsonEncode(receiptJson)
        ..paymentType = PaymentType.cash
        ..paymentStatus = PaymentStatus.completed
        ..syncStatus = SyncStatus.pending;
      
      await _offlineQueue.enqueue(transaction);
      
      // 4. طباعة الفاتورة
      await _printReceipt(invoice, receipt);
      
      return PaymentResult.success(
        invoiceId: invoice.localId,
        message: 'تم الدفع بنجاح',
      );
      
    } catch (e) {
      return PaymentResult.error(e.toString());
    }
  }
}
```

### 6.2 الدفع بالشبكة (Card Payment)

```dart
class PaymentController extends GetxController {
  
  /// معالجة الدفع بالبطاقة
  Future<PaymentResult> processCardPayment({
    required List<CartItem> items,
    required double totalAmount,
  }) async {
    try {
      // 1. التحقق من الاتصال
      if (!_connectivity.isConnected.value) {
        return PaymentResult.error(
          'الدفع بالشبكة يتطلب اتصال بالإنترنت',
          suggestedAction: 'استخدم الدفع النقدي',
        );
      }
      
      // 2. إنشاء فاتورة مؤقتة محلية
      final invoice = _createInvoice(items, totalAmount);
      final tempTransaction = LocalTransaction()
        ..invoiceData = jsonEncode(invoice.toJson())
        ..paymentType = PaymentType.card
        ..paymentStatus = PaymentStatus.pending;
      
      await _db.transactions.put(tempTransaction);
      
      // 3. إرسال المبلغ للجهاز
      final terminalResult = await _terminal.processCardPayment(
        amount: totalAmount,
        invoiceId: invoice.localId,
      );
      
      if (terminalResult.status == TerminalStatus.success) {
        // 4. تثبيت الفاتورة
        final receipt = _createCardReceipt(
          invoiceId: invoice.localId,
          amount: totalAmount,
          refNo: terminalResult.refNo!,
          authCode: terminalResult.authCode!,
        );
        
        tempTransaction
          ..receiptData = jsonEncode(receipt.toJson())
          ..paymentStatus = PaymentStatus.completed
          ..bankReference = terminalResult.refNo;
        
        await _db.transactions.put(tempTransaction);
        await _offlineQueue.enqueue(tempTransaction);
        
        // 5. طباعة
        await _printReceipt(invoice, receipt);
        
        return PaymentResult.success(
          invoiceId: invoice.localId,
          reference: terminalResult.refNo,
        );
        
      } else {
        // فشل الدفع
        tempTransaction.paymentStatus = PaymentStatus.failed;
        await _db.transactions.put(tempTransaction);
        
        return PaymentResult.declined(
          reason: terminalResult.reason ?? 'رفض من البنك',
        );
      }
      
    } catch (e) {
      return PaymentResult.error(e.toString());
    }
  }
}
```

### 6.3 مخطط تسلسل الدفع بالشبكة

```
┌─────────────────────────────────────────────────────────────┐
│              Card Payment Sequence Diagram                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  User        App         LocalDB    Terminal    Bank        │
│   │           │             │           │         │         │
│   │──── 1. اضغط دفع ────►│             │           │         │
│   │           │             │           │         │         │
│   │           │──2.حفظ مؤقت─►│           │         │         │
│   │           │◄────────────│           │         │         │
│   │           │             │           │         │         │
│   │           │───3.إرسال المبلغ───────►│         │         │
│   │           │             │           │         │         │
│   │◄─ 4. أدخل البطاقة ─────│           │         │         │
│   │           │             │           │         │         │
│   │────► 5. إدخال PIN ─────────────────►│         │         │
│   │           │             │           │         │         │
│   │           │             │           │──6.تفويض─►│         │
│   │           │             │           │◄─7.موافق─│         │
│   │           │             │           │         │         │
│   │           │◄──────8.نتيجة──────────│         │         │
│   │           │             │           │         │         │
│   │           │──9.تثبيت───►│           │         │         │
│   │           │             │           │         │         │
│   │◄────10.طباعة + نجاح────│           │         │         │
│   │           │             │           │         │         │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. الطباعة (Printing)

### 7.1 المواصفات

| البند | القيمة |
|-------|--------|
| نوع الطابعة | Thermal Printer (SUNMI) |
| عرض الورق | 58mm أو 80mm |
| الاعتماد على الإنترنت | ❌ لا |
| إعادة المحاولة | 3 مرات تلقائياً |

### 7.2 Printer Service

```dart
class PrinterService extends GetxService {
  
  /// طباعة الفاتورة
  Future<PrintResult> printReceipt(
    Invoice invoice,
    Receipt receipt,
  ) async {
    try {
      // بناء محتوى الفاتورة
      final content = _buildReceiptContent(invoice, receipt);
      
      // إرسال للطابعة
      await SunmiPrinter.startTransactionPrint(true);
      
      // الهيدر
      await SunmiPrinter.printText(content.header);
      await SunmiPrinter.lineWrap(1);
      
      // تفاصيل الخدمات
      for (final item in content.items) {
        await SunmiPrinter.printText(item);
      }
      
      await SunmiPrinter.lineWrap(1);
      
      // الإجمالي
      await SunmiPrinter.printText(
        content.total,
        style: SunmiStyle(bold: true, fontSize: SunmiFontSize.LG),
      );
      
      // QR Code
      await SunmiPrinter.printQRCode(content.qrData);
      
      await SunmiPrinter.lineWrap(3);
      await SunmiPrinter.cut();
      
      await SunmiPrinter.submitTransactionPrint();
      
      return PrintResult.success();
      
    } catch (e) {
      return PrintResult.error(e.toString());
    }
  }
  
  /// إعادة المحاولة مع Backoff
  Future<PrintResult> printWithRetry(
    Invoice invoice,
    Receipt receipt,
  ) async {
    const maxRetries = 3;
    
    for (int i = 0; i < maxRetries; i++) {
      final result = await printReceipt(invoice, receipt);
      
      if (result.isSuccess) {
        return result;
      }
      
      // انتظار قبل المحاولة التالية
      await Future.delayed(Duration(seconds: (i + 1) * 2));
    }
    
    return PrintResult.error('فشلت جميع محاولات الطباعة');
  }
}
```

---

## 8. الأمان (Security)

### 8.1 قواعد الأمان

| القاعدة | التطبيق |
|---------|---------|
| **منع تعديل السعر** | السعر يُحسب من المنتج فقط |
| **توقيع محلي** | UUID فريد لكل عملية |
| **Idempotency** | منع التكرار |
| **تشفير** | البيانات الحساسة مشفرة |

### 8.2 التحقق من السعر

```dart
class PriceValidator {
  final ProductsRepository _products;
  
  /// التحقق من صحة السعر
  bool validatePrice(CartItem item) {
    final product = _products.getById(item.productId);
    
    if (product == null) {
      throw InvalidProductException();
    }
    
    // السعر يجب أن يتطابق مع المنتج
    if (item.price != product.price) {
      throw PriceMismatchException(
        expected: product.price,
        received: item.price,
      );
    }
    
    return true;
  }
  
  /// حساب الإجمالي الصحيح
  double calculateTotal(List<CartItem> items) {
    double total = 0;
    
    for (final item in items) {
      validatePrice(item);
      total += item.price * item.quantity;
    }
    
    return total;
  }
}
```

### 8.3 توقيع المعاملات

```dart
class TransactionSigner {
  
  /// توليد توقيع للمعاملة
  String signTransaction(LocalTransaction tx) {
    final payload = '${tx.localId}|${tx.invoiceData}|${tx.createdAt}';
    final bytes = utf8.encode(payload);
    final digest = sha256.convert(bytes);
    return digest.toString();
  }
  
  /// التحقق من التوقيع
  bool verifySignature(LocalTransaction tx, String signature) {
    final expected = signTransaction(tx);
    return expected == signature;
  }
}
```

---

## 9. Checklist للمطورين

### 9.1 قبل البدء

- [ ] إعداد Flutter 3.19+
- [ ] تثبيت المكتبات المطلوبة
- [ ] إعداد بيئة تطوير SUNMI

### 9.2 Core Features

- [ ] Local Database (Isar/SQLite)
- [ ] Offline Queue
- [ ] Sync Service
- [ ] Connectivity Monitoring

### 9.3 Payment Features

- [ ] Cash Payment
- [ ] Card Payment Integration
- [ ] Payment Terminal SDK
- [ ] Receipt Printing

### 9.4 UX Features

- [ ] Service Selection Grid
- [ ] Car Size Bottom Sheet
- [ ] Payment Screen
- [ ] Loading Overlays
- [ ] Error Dialogs
- [ ] Success Animations

### 9.5 Testing

- [ ] Unit Tests للـ Business Logic
- [ ] Integration Tests للـ Payment Flow
- [ ] E2E Tests على جهاز SUNMI

---

## 10. المكتبات المطلوبة

```yaml
dependencies:
  flutter:
    sdk: flutter
  
  # State Management
  get: ^4.6.6
  
  # Database
  isar: ^3.1.0+1
  isar_flutter_libs: ^3.1.0+1
  
  # Network
  dio: ^5.4.0
  connectivity_plus: ^5.0.2
  
  # Hardware
  sunmi_printer_plus: ^1.3.5
  
  # ZATCA
  zatca_fatoora_flutter: ^1.0.0
  
  # Utils
  uuid: ^4.2.2
  intl: ^0.18.1
  json_annotation: ^4.8.1
  crypto: ^3.0.3
  
dev_dependencies:
  build_runner: ^2.4.7
  isar_generator: ^3.1.0+1
  json_serializable: ^6.7.1
```

---

> [!TIP]
> **للبدء السريع:**
> 1. Clone المشروع
> 2. `flutter pub get`
> 3. `flutter pub run build_runner build`
> 4. اختبر على SUNMI V2s أو المحاكي
