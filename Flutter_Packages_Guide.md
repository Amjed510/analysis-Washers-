# 📦 دليل المكتبات (Flutter Packages) - نظام WashPOS

> [!IMPORTANT]
> شرح تفصيلي لجميع المكتبات المطلوبة لتطوير التطبيق  
> **الإصدارات:** محدثة لـ Flutter 3.19+

---

## 📋 جدول المحتويات

1. [State Management](#1-state-management)
2. [Database & Storage](#2-database--storage)
3. [Network & API](#3-network--api)
4. [Hardware Integration](#4-hardware-integration)
5. [ZATCA & Invoicing](#5-zatca--invoicing)
6. [Utilities](#6-utilities)
7. [ملخص pubspec.yaml](#7-ملخص-pubspecyaml)

---

## 1. State Management

### 1.1 GetX (`get`)

```yaml
get: ^4.6.6
```

**الوصف:**
مكتبة شاملة لإدارة الحالة (State Management) والـ Navigation والـ Dependency Injection.

**لماذا نستخدمها؟**
- أخف وزناً من Provider و Bloc
- تجمع 3 وظائف في مكتبة واحدة
- سهلة التعلم والاستخدام
- أداء ممتاز

**الاستخدامات في المشروع:**

```dart
// 1. State Management
class PaymentController extends GetxController {
  var isProcessing = false.obs;
  var totalAmount = 0.0.obs;
  
  void startPayment() {
    isProcessing.value = true;
  }
}

// 2. Navigation
Get.to(() => PaymentScreen());
Get.back();
Get.offAll(() => HomeScreen());

// 3. Dependency Injection
Get.put(PaymentController());
Get.find<PaymentController>();

// 4. Snackbars & Dialogs
Get.snackbar('نجاح', 'تمت العملية بنجاح');
Get.dialog(AlertDialog(...));
```

**المميزات:**
| الميزة | الوصف |
|--------|-------|
| `.obs` | جعل أي متغير قابل للمراقبة |
| `Obx()` | Widget يُعاد بناؤه تلقائياً |
| `GetBuilder` | للتحكم اليدوي في إعادة البناء |
| `GetConnect` | HTTP client مدمج |

---

## 2. Database & Storage

### 2.1 Isar Database (`isar` + `isar_flutter_libs`)

```yaml
isar: ^3.1.0+1
isar_flutter_libs: ^3.1.0+1
```

**الوصف:**
قاعدة بيانات NoSQL سريعة جداً مصممة لـ Flutter، بديل لـ Hive و SQLite.

**لماذا نستخدمها؟**
- أسرع 10x من SQLite
- تدعم الفهارس والاستعلامات المعقدة
- تعمل Offline بالكامل
- حجم صغير جداً

**الاستخدامات في المشروع:**

```dart
// تعريف Schema
@collection
class LocalTransaction {
  Id id = Isar.autoIncrement;
  
  @Index(unique: true)
  String localId = '';
  
  String? serverId;
  String invoiceData = '';
  
  @enumerated
  SyncStatus syncStatus = SyncStatus.pending;
  
  DateTime createdAt = DateTime.now();
}

// فتح قاعدة البيانات
final isar = await Isar.open([
  LocalTransactionSchema,
  CachedProductSchema,
]);

// إضافة بيانات
await isar.writeTxn(() async {
  await isar.localTransactions.put(transaction);
});

// الاستعلام
final pending = await isar.localTransactions
    .filter()
    .syncStatusEqualTo(SyncStatus.pending)
    .findAll();
```

**المميزات:**
| الميزة | الوصف |
|--------|-------|
| ACID | معاملات آمنة |
| Indexes | فهارس للبحث السريع |
| Links | علاقات بين الجداول |
| Watch | مراقبة التغييرات |

### 2.2 Isar Generator (`isar_generator`)

```yaml
dev_dependencies:
  isar_generator: ^3.1.0+1
```

**الوصف:**
مولد الكود لإنشاء ملفات الـ Schema تلقائياً.

**الاستخدام:**
```bash
flutter pub run build_runner build
```

---

## 3. Network & API

### 3.1 Dio (`dio`)

```yaml
dio: ^5.4.0
```

**الوصف:**
HTTP client قوي لـ Dart، أفضل من http package.

**لماذا نستخدمها؟**
- تدعم Interceptors
- تدعم FormData و Multipart
- تدعم Cancel tokens
- تدعم Timeout مخصص

**الاستخدامات في المشروع:**

```dart
// إنشاء Instance
final dio = Dio(BaseOptions(
  baseUrl: 'https://api.washpos.sa/v1',
  connectTimeout: Duration(seconds: 30),
  receiveTimeout: Duration(seconds: 30),
  headers: {
    'Content-Type': 'application/json',
    'Accept-Language': 'ar',
  },
));

// Interceptor للتوكن
dio.interceptors.add(InterceptorsWrapper(
  onRequest: (options, handler) {
    options.headers['Authorization'] = 'Bearer $token';
    return handler.next(options);
  },
  onError: (error, handler) {
    if (error.response?.statusCode == 401) {
      // Refresh token
    }
    return handler.next(error);
  },
));

// إرسال طلب
final response = await dio.post(
  '/invoices',
  data: invoice.toJson(),
  options: Options(
    headers: {'X-Idempotency-Key': idempotencyKey},
  ),
);
```

**المميزات:**
| الميزة | الوصف |
|--------|-------|
| Interceptors | معالجة الطلبات قبل/بعد الإرسال |
| CancelToken | إلغاء الطلبات |
| Retry | إعادة المحاولة تلقائياً |
| FormData | رفع الملفات |

### 3.2 Connectivity Plus (`connectivity_plus`)

```yaml
connectivity_plus: ^5.0.2
```

**الوصف:**
مكتبة لمراقبة حالة اتصال الإنترنت (WiFi, Mobile, None).

**لماذا نستخدمها؟**
- معرفة نوع الاتصال
- إشعار عند تغير الحالة
- ضرورية لـ Offline-First

**الاستخدامات في المشروع:**

```dart
class ConnectivityService extends GetxService {
  final Connectivity _connectivity = Connectivity();
  final isConnected = false.obs;
  
  @override
  void onInit() {
    super.onInit();
    
    // مراقبة التغييرات
    _connectivity.onConnectivityChanged.listen((result) {
      isConnected.value = result != ConnectivityResult.none;
      
      if (isConnected.value) {
        // بدء المزامنة
        Get.find<SyncService>().syncNow();
      }
    });
  }
  
  Future<bool> checkConnection() async {
    final result = await _connectivity.checkConnectivity();
    return result != ConnectivityResult.none;
  }
}
```

**أنواع الاتصال:**
| القيمة | الوصف |
|--------|-------|
| `ConnectivityResult.wifi` | واي فاي |
| `ConnectivityResult.mobile` | بيانات الجوال |
| `ConnectivityResult.ethernet` | كيبل |
| `ConnectivityResult.none` | لا يوجد اتصال |

---

## 4. Hardware Integration

### 4.1 SUNMI Printer Plus (`sunmi_printer_plus`)

```yaml
sunmi_printer_plus: ^1.3.5
```

**الوصف:**
مكتبة للتحكم في طابعات SUNMI الحرارية المدمجة في أجهزة SUNMI V2s.

**لماذا نستخدمها؟**
- الطريقة الرسمية للطباعة على SUNMI
- تدعم النصوص، الصور، QR Codes
- تدعم الأنماط المختلفة (Bold, Size)

**الاستخدامات في المشروع:**

```dart
class PrinterService extends GetxService {
  
  /// تهيئة الطابعة
  Future<void> initialize() async {
    await SunmiPrinter.bindingPrinter();
  }
  
  /// طباعة الفاتورة
  Future<void> printReceipt(Invoice invoice) async {
    await SunmiPrinter.startTransactionPrint(true);
    
    // الهيدر
    await SunmiPrinter.setAlignment(SunmiPrintAlign.CENTER);
    await SunmiPrinter.printText(
      'مغسلة الأمان',
      style: SunmiStyle(fontSize: SunmiFontSize.LG, bold: true),
    );
    await SunmiPrinter.lineWrap(1);
    
    // رقم الفاتورة
    await SunmiPrinter.setAlignment(SunmiPrintAlign.RIGHT);
    await SunmiPrinter.printText('رقم الفاتورة: ${invoice.number}');
    await SunmiPrinter.printText('التاريخ: ${invoice.date}');
    await SunmiPrinter.lineWrap(1);
    
    // الخدمات
    await SunmiPrinter.printText('─' * 32);
    for (final item in invoice.items) {
      await SunmiPrinter.printRow(cols: [
        ColumnMaker(text: item.name, width: 20, align: SunmiPrintAlign.RIGHT),
        ColumnMaker(text: '${item.price} ر.س', width: 12, align: SunmiPrintAlign.LEFT),
      ]);
    }
    await SunmiPrinter.printText('─' * 32);
    
    // الإجمالي
    await SunmiPrinter.printText(
      'الإجمالي: ${invoice.total} ر.س',
      style: SunmiStyle(fontSize: SunmiFontSize.LG, bold: true),
    );
    await SunmiPrinter.lineWrap(1);
    
    // QR Code للضريبة
    await SunmiPrinter.setAlignment(SunmiPrintAlign.CENTER);
    await SunmiPrinter.printQRCode(invoice.zatcaQR);
    await SunmiPrinter.lineWrap(3);
    
    // قص الورق
    await SunmiPrinter.cut();
    
    await SunmiPrinter.submitTransactionPrint();
  }
  
  /// فحص حالة الطابعة
  Future<PrinterStatus> getStatus() async {
    final status = await SunmiPrinter.getPrinterStatus();
    return PrinterStatus.fromCode(status);
  }
}

enum PrinterStatus {
  normal,    // 0 - طبيعي
  noPaper,   // 1 - لا يوجد ورق
  overHeat,  // 2 - حرارة عالية
  error,     // 3 - خطأ
}
```

**الدوال الأساسية:**

| الدالة | الوصف |
|--------|-------|
| `bindingPrinter()` | تهيئة الطابعة |
| `printText()` | طباعة نص |
| `printRow()` | طباعة صف (أعمدة) |
| `printQRCode()` | طباعة QR Code |
| `printImage()` | طباعة صورة |
| `lineWrap()` | أسطر فارغة |
| `cut()` | قص الورق |
| `setAlignment()` | المحاذاة |
| `getPrinterStatus()` | حالة الطابعة |

### 4.2 NFC Payment - الدفع عبر NFC

> [!IMPORTANT]
> الدفع عبر NFC على SUNMI V2s يتطلب **SUNMI PAY SDK V2** وهو SDK أصلي لـ Android (ليس مكتبة Flutter)  
> يتم التواصل معه عبر **Platform Channels**

#### 4.2.1 الطريقة الصحيحة للتكامل

```
┌─────────────────────────────────────────────────────────────────┐
│                  Flutter + SUNMI Payment Architecture            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────┐                                       │
│   │    Flutter App      │                                       │
│   │    (Dart Code)      │                                       │
│   └──────────┬──────────┘                                       │
│              │                                                   │
│              │ Platform Channel                                  │
│              │ (MethodChannel)                                   │
│              ▼                                                   │
│   ┌─────────────────────┐                                       │
│   │   Native Android    │                                       │
│   │   (Kotlin/Java)     │                                       │
│   └──────────┬──────────┘                                       │
│              │                                                   │
│              ▼                                                   │
│   ┌─────────────────────┐    ┌─────────────────────┐           │
│   │  SUNMI PAY SDK V2   │───▶│   Payment Gateway   │           │
│   │  (Native Android)   │    │  (Geidea/PayTabs)   │           │
│   └─────────────────────┘    └─────────────────────┘           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 4.2.2 SUNMI PAY SDK V2 (Native Android)

> [!NOTE]
> **مصدر الـ SDK:**
> - يُحصل عليه من [SUNMI Developer Portal](https://developer.sunmi.com)
> - أو من مزود بوابة الدفع (Payment Gateway) مثل Geidea

**الملفات المطلوبة:**
```
android/
├── app/
│   ├── libs/
│   │   ├── sunmi-payment-sdk.aar      ← SDK من SUNMI
│   │   └── sunmi-basic-sdk.aar        ← SDK أساسي
│   └── build.gradle                   ← إضافة الـ dependencies
```

**إضافة للـ build.gradle:**
```gradle
dependencies {
    implementation files('libs/sunmi-payment-sdk.aar')
    implementation files('libs/sunmi-basic-sdk.aar')
}
```

#### 4.2.3 Platform Channel (جسر Flutter-Android)

**الجانب Flutter (Dart):**

```dart
import 'package:flutter/services.dart';

class SunmiPaymentChannel {
  static const MethodChannel _channel = MethodChannel('com.washpos/payment');
  
  /// بدء عملية الدفع عبر NFC
  static Future<Map<String, dynamic>> processNFCPayment({
    required double amount,
    required String invoiceId,
  }) async {
    try {
      final result = await _channel.invokeMethod('processNFCPayment', {
        'amount': (amount * 100).round(), // بالهللات
        'invoiceId': invoiceId,
        'currencyCode': 'SAR',
      });
      
      return Map<String, dynamic>.from(result);
    } on PlatformException catch (e) {
      return {
        'success': false,
        'error': e.message,
        'code': e.code,
      };
    }
  }
  
  /// إلغاء العملية الجارية
  static Future<void> cancelPayment() async {
    await _channel.invokeMethod('cancelPayment');
  }
  
  /// فحص جاهزية الجهاز
  static Future<bool> isReady() async {
    final result = await _channel.invokeMethod('isDeviceReady');
    return result == true;
  }
}
```

**الجانب Android (Kotlin):**

```kotlin
// android/app/src/main/kotlin/.../MainActivity.kt

class MainActivity : FlutterActivity() {
    private val CHANNEL = "com.washpos/payment"
    
    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)
        
        MethodChannel(flutterEngine.dartExecutor.binaryMessenger, CHANNEL)
            .setMethodCallHandler { call, result ->
                when (call.method) {
                    "processNFCPayment" -> {
                        val amount = call.argument<Int>("amount") ?: 0
                        val invoiceId = call.argument<String>("invoiceId") ?: ""
                        processPayment(amount, invoiceId, result)
                    }
                    "cancelPayment" -> {
                        cancelCurrentPayment()
                        result.success(null)
                    }
                    "isDeviceReady" -> {
                        result.success(isSunmiDeviceReady())
                    }
                    else -> result.notImplemented()
                }
            }
    }
    
    private fun processPayment(
        amount: Int,
        invoiceId: String,
        result: MethodChannel.Result
    ) {
        // استخدام SUNMI PAY SDK
        val paymentRequest = PaymentRequest.Builder()
            .setAmount(amount)
            .setTransactionType(TransactionType.PURCHASE)
            .setReference(invoiceId)
            .build()
        
        SunmiPaymentApi.startPayment(this, paymentRequest, object : PaymentCallback {
            override fun onSuccess(response: PaymentResponse) {
                result.success(mapOf(
                    "success" to true,
                    "referenceNumber" to response.referenceNumber,
                    "authCode" to response.authorizationCode,
                    "cardType" to response.cardScheme,
                    "maskedPAN" to response.maskedPAN
                ))
            }
            
            override fun onFailure(error: PaymentError) {
                result.success(mapOf(
                    "success" to false,
                    "error" to error.message,
                    "code" to error.code
                ))
            }
        })
    }
}
```

#### 4.2.4 بوابات الدفع المدعومة في السعودية (mada)

| البوابة | الموقع | ملاحظات |
|---------|--------|---------|
| **Geidea** | geidea.net | تدعم SUNMI مباشرة |
| **PayTabs** | paytabs.com | SDK جاهز |
| **Tap Payments** | tap.company | شائعة في السعودية |
| **HyperPay** | hyperpay.com | دعم mada |
| **Moyasar** | moyasar.com | للمشاريع الصغيرة |

#### 4.2.5 استخدام الـ Service في Flutter

```dart
class NFCPaymentService extends GetxService {
  final isProcessing = false.obs;
  final statusMessage = ''.obs;
  
  /// معالجة الدفع عبر NFC
  Future<PaymentResult> processPayment({
    required double amount,
    required String invoiceId,
  }) async {
    isProcessing.value = true;
    statusMessage.value = 'قرّب البطاقة من الجهاز...';
    
    try {
      // استدعاء Native SDK عبر Platform Channel
      final response = await SunmiPaymentChannel.processNFCPayment(
        amount: amount,
        invoiceId: invoiceId,
      );
      
      if (response['success'] == true) {
        statusMessage.value = 'تمت العملية بنجاح ✅';
        return PaymentResult(
          success: true,
          referenceNumber: response['referenceNumber'],
          authCode: response['authCode'],
          cardType: response['cardType'],
        );
      } else {
        statusMessage.value = 'فشلت العملية: ${response['error']}';
        return PaymentResult(
          success: false,
          error: response['error'],
        );
      }
    } finally {
      isProcessing.value = false;
    }
  }
}
```

**الاستخدامات في المشروع:**

```dart
class NFCPaymentService extends GetxService {
  
  final isReading = false.obs;
  final paymentStatus = ''.obs;
  
  /// تهيئة قارئ البطاقات
  Future<void> initialize() async {
    await SunmiCardReader.init();
    print('✅ NFC Card Reader initialized');
  }
  
  /// معالجة الدفع عبر NFC
  Future<PaymentResult> processNFCPayment({
    required double amount,
    required String invoiceId,
  }) async {
    isReading.value = true;
    paymentStatus.value = 'في انتظار البطاقة...';
    
    try {
      // 1. تحويل المبلغ للهللات
      final amountInHalalas = (amount * 100).round();
      
      // 2. بدء عملية الدفع
      paymentStatus.value = 'تقريب البطاقة من الجهاز...';
      
      final result = await SunmiCardReader.startContactlessPayment(
        amount: amountInHalalas,
        currencyCode: '682',  // SAR
        transactionType: TransactionType.purchase,
        reference: invoiceId,
        timeout: 60,  // ثانية
      );
      
      // 3. معالجة النتيجة
      if (result.status == PaymentStatus.approved) {
        paymentStatus.value = 'تمت الموافقة ✅';
        
        return PaymentResult(
          success: true,
          referenceNumber: result.referenceNumber,
          authCode: result.authorizationCode,
          cardType: result.cardScheme,  // mada, Visa, MC
          maskedPan: result.maskedPAN,  // ****1234
          receiptData: result.merchantReceipt,
        );
      } else {
        paymentStatus.value = 'رُفضت العملية ❌';
        
        return PaymentResult(
          success: false,
          error: result.declineReason,
          errorCode: result.responseCode,
        );
      }
      
    } on TimeoutException {
      paymentStatus.value = 'انتهت المهلة';
      return PaymentResult(success: false, error: 'لم يتم تقريب البطاقة');
      
    } on CardReadException catch (e) {
      paymentStatus.value = 'خطأ في قراءة البطاقة';
      return PaymentResult(success: false, error: e.message);
      
    } finally {
      isReading.value = false;
      await SunmiCardReader.cancelTransaction();
    }
  }
  
  /// إلغاء العملية الجارية
  Future<void> cancelPayment() async {
    await SunmiCardReader.cancelTransaction();
    isReading.value = false;
    paymentStatus.value = 'تم الإلغاء';
  }
}
```

**تدفق الدفع عبر NFC:**

```
┌─────────────────────────────────────────────────────────────────┐
│                      NFC Payment Flow                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐     │
│  │  Start  │───▶│ Waiting │───▶│ Reading │───▶│ Process │     │
│  │ Payment │    │   NFC   │    │  Card   │    │   Bank  │     │
│  └─────────┘    └─────────┘    └─────────┘    └────┬────┘     │
│                                                    │            │
│                                        ┌───────────┴──────┐    │
│                                        │                  │    │
│                                        ▼                  ▼    │
│                                   ┌─────────┐       ┌─────────┐│
│                                   │ Approved│       │ Declined││
│                                   │   ✅    │       │   ❌    ││
│                                   └────┬────┘       └────┬────┘│
│                                        │                  │    │
│                                        ▼                  ▼    │
│                                   ┌─────────┐       ┌─────────┐│
│                                   │  Print  │       │  Retry/ ││
│                                   │ Receipt │       │  Cancel ││
│                                   └─────────┘       └─────────┘│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**حالات البطاقة:**

| الحالة | الوصف | الإجراء |
|--------|-------|---------|
| `approved` | تمت الموافقة | طباعة الإيصال |
| `declined` | رُفضت | عرض السبب + بديل |
| `timeout` | انتهت المهلة | إعادة المحاولة |
| `card_error` | خطأ في البطاقة | تقريب مرة أخرى |
| `cancelled` | ألغى المستخدم | العودة للدفع |

**أكواد الرد الشائعة:**

| الكود | الوصف |
|-------|-------|
| `00` | تمت الموافقة |
| `51` | رصيد غير كافي |
| `54` | بطاقة منتهية |
| `55` | PIN خاطئ |
| `61` | تجاوز الحد |
| `91` | البنك غير متاح |

**UI للدفع NFC:**

```dart
class NFCPaymentScreen extends StatelessWidget {
  final NFCPaymentService _nfcService = Get.find();
  final double amount;
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: Obx(() {
          if (_nfcService.isReading.value) {
            return Column(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                // أيقونة NFC متحركة
                Lottie.asset('assets/nfc_animation.json', width: 200),
                
                SizedBox(height: 24),
                
                Text(
                  'المبلغ: ${amount.toStringAsFixed(2)} ر.س',
                  style: TextStyle(fontSize: 28, fontWeight: FontWeight.bold),
                ),
                
                SizedBox(height: 16),
                
                Text(
                  _nfcService.paymentStatus.value,
                  style: TextStyle(fontSize: 18, color: Colors.grey[600]),
                ),
                
                SizedBox(height: 32),
                
                // زر الإلغاء
                TextButton(
                  onPressed: () => _nfcService.cancelPayment(),
                  child: Text('إلغاء'),
                ),
              ],
            );
          }
          return SizedBox();
        }),
      ),
    );
  }
}
```

---

## 5. ZATCA & Invoicing

### 5.1 ZATCA Fatoora (`zatca_fatoora_flutter`)

```yaml
zatca_fatoora_flutter: ^1.0.0
```

**الوصف:**
مكتبة لتوليد QR Code متوافق مع متطلبات هيئة الزكاة والضريبة والجمارك (ZATCA).

**لماذا نستخدمها؟**
- مطلوبة قانونياً في السعودية
- تنسيق TLV القياسي
- Base64 encoding

**الاستخدامات في المشروع:**

```dart
import 'package:zatca_fatoora_flutter/zatca_fatoora_flutter.dart';

class ZatcaService {
  
  /// توليد QR Code للفاتورة
  String generateQRCode({
    required String sellerName,
    required String vatNumber,
    required DateTime invoiceDate,
    required double totalWithVat,
    required double vatAmount,
  }) {
    final fatoora = Fatoora(
      sellerName: sellerName,           // اسم البائع
      taxNumber: vatNumber,              // الرقم الضريبي
      invoiceDate: invoiceDate,          // تاريخ الفاتورة
      totalAmount: totalWithVat,         // المبلغ شامل الضريبة
      vatAmount: vatAmount,              // مبلغ الضريبة
    );
    
    return fatoora.toBase64();
  }
  
  /// مثال على الاستخدام
  void example() {
    final qrData = generateQRCode(
      sellerName: 'مغسلة الأمان',
      vatNumber: '300000000000003',
      invoiceDate: DateTime.now(),
      totalWithVat: 80.50,
      vatAmount: 10.50,
    );
    
    // qrData يُستخدم مع:
    // SunmiPrinter.printQRCode(qrData);
  }
}
```

**حقول QR Code (TLV Format):**

| Tag | الحقل | الوصف |
|-----|-------|-------|
| 1 | Seller Name | اسم البائع |
| 2 | VAT Number | الرقم الضريبي |
| 3 | Invoice Date | تاريخ ووقت الفاتورة |
| 4 | Total with VAT | المبلغ شامل الضريبة |
| 5 | VAT Amount | مبلغ الضريبة |

---

## 6. Utilities

### 6.1 UUID (`uuid`)

```yaml
uuid: ^4.2.2
```

**الوصف:**
مكتبة لتوليد معرفات فريدة عالمياً (UUID v4).

**لماذا نستخدمها؟**
- توليد Idempotency Keys
- معرفات محلية للمعاملات
- منع التكرار

**الاستخدامات في المشروع:**

```dart
import 'package:uuid/uuid.dart';

const uuid = Uuid();

// توليد معرف فريد
String localId = uuid.v4();
// مثال: "550e8400-e29b-41d4-a716-446655440000"

// Idempotency Key للـ API
String idempotencyKey = 'ORD-${DateTime.now().millisecondsSinceEpoch}-${uuid.v4()}';
// مثال: "ORD-1702905600000-550e8400-e29b-41d4-a716-446655440000"
```

### 6.2 Intl (`intl`)

```yaml
intl: ^0.18.1
```

**الوصف:**
مكتبة للتعامل مع التواريخ والأرقام والعملات حسب اللغة/المنطقة.

**لماذا نستخدمها؟**
- تنسيق التواريخ بالعربي
- تنسيق العملات (ر.س)
- تنسيق الأرقام

**الاستخدامات في المشروع:**

```dart
import 'package:intl/intl.dart';

class Formatters {
  
  /// تنسيق التاريخ
  static String formatDate(DateTime date) {
    return DateFormat('yyyy/MM/dd - HH:mm', 'ar').format(date);
    // مثال: "2024/12/21 - 14:30"
  }
  
  /// تنسيق العملة
  static String formatCurrency(double amount) {
    final formatter = NumberFormat.currency(
      locale: 'ar_SA',
      symbol: 'ر.س ',
      decimalDigits: 2,
    );
    return formatter.format(amount);
    // مثال: "ر.س 80.50"
  }
  
  /// تنسيق رقم
  static String formatNumber(int number) {
    return NumberFormat('#,###', 'ar').format(number);
    // مثال: "1,250"
  }
}
```

### 6.3 JSON Annotation (`json_annotation` + `json_serializable`)

```yaml
json_annotation: ^4.8.1

dev_dependencies:
  json_serializable: ^6.7.1
  build_runner: ^2.4.7
```

**الوصف:**
مكتبات لتحويل الـ JSON إلى Dart objects والعكس تلقائياً.

**لماذا نستخدمها؟**
- Type safety
- أقل كود يدوي
- أقل أخطاء

**الاستخدامات في المشروع:**

```dart
import 'package:json_annotation/json_annotation.dart';

part 'product.g.dart';

@JsonSerializable()
class Product {
  final String id;
  final String name;
  
  @JsonKey(name: 'name_ar')
  final String nameAr;
  
  final double price;
  
  @JsonKey(name: 'car_size')
  final String? carSize;
  
  Product({
    required this.id,
    required this.name,
    required this.nameAr,
    required this.price,
    this.carSize,
  });
  
  factory Product.fromJson(Map<String, dynamic> json) => 
      _$ProductFromJson(json);
  
  Map<String, dynamic> toJson() => _$ProductToJson(this);
}
```

**توليد الكود:**
```bash
flutter pub run build_runner build
```

### 6.4 Crypto (`crypto`)

```yaml
crypto: ^3.0.3
```

**الوصف:**
مكتبة للتشفير (SHA256, MD5, HMAC).

**لماذا نستخدمها؟**
- توقيع المعاملات
- التحقق من السلامة
- تشفير البيانات الحساسة

**الاستخدامات في المشروع:**

```dart
import 'dart:convert';
import 'package:crypto/crypto.dart';

class SecurityUtils {
  
  /// توقيع المعاملة
  static String signTransaction(String data, String secretKey) {
    final key = utf8.encode(secretKey);
    final bytes = utf8.encode(data);
    
    final hmac = Hmac(sha256, key);
    final digest = hmac.convert(bytes);
    
    return digest.toString();
  }
  
  /// تجزئة SHA256
  static String hashData(String data) {
    final bytes = utf8.encode(data);
    final digest = sha256.convert(bytes);
    return digest.toString();
  }
  
  /// التحقق من التوقيع
  static bool verifySignature(String data, String signature, String secretKey) {
    final expected = signTransaction(data, secretKey);
    return expected == signature;
  }
}
```

---

## 7. ملخص pubspec.yaml

```yaml
name: washpos
description: نظام نقاط البيع للمغاسل
version: 1.0.0+1

environment:
  sdk: '>=3.0.0 <4.0.0'

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
  
  # Hardware - Printer
  sunmi_printer_plus: ^1.3.5
  # ملاحظة: الدفع NFC يستخدم SUNMI PAY SDK V2 (Native Android) عبر Platform Channels
  lottie: ^3.0.0  # للأيقونات المتحركة
  
  # ZATCA
  zatca_fatoora_flutter: ^1.0.0
  
  # Utilities
  uuid: ^4.2.2
  intl: ^0.18.1
  json_annotation: ^4.8.1
  crypto: ^3.0.3
  
  # UI
  flutter_svg: ^2.0.9
  cached_network_image: ^3.3.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  
  # Code Generation
  build_runner: ^2.4.7
  isar_generator: ^3.1.0+1
  json_serializable: ^6.7.1
  
  flutter_lints: ^3.0.1

flutter:
  uses-material-design: true
  
  assets:
    - assets/images/
    - assets/icons/
```

---

## 📊 ملخص المكتبات

| المكتبة | الفئة | الوظيفة الأساسية |
|---------|-------|------------------|
| `get` | State | إدارة الحالة والتنقل |
| `isar` | Database | قاعدة بيانات محلية |
| `dio` | Network | HTTP requests |
| `connectivity_plus` | Network | مراقبة الاتصال |
| `sunmi_printer_plus` | Hardware | الطباعة الحرارية |
| `zatca_fatoora_flutter` | Business | QR Code الضريبي |
| `uuid` | Utility | معرفات فريدة |
| `intl` | Utility | تنسيق التواريخ والأرقام |
| `json_annotation` | Utility | تحويل JSON |
| `crypto` | Security | التشفير والتوقيع |
| `lottie` | UI | الأيقونات المتحركة |

> [!NOTE]
> **الدفع عبر NFC:** يستخدم **SUNMI PAY SDK V2** (Native Android) عبر Platform Channels - وليس مكتبة Flutter

---

> [!TIP]
> **للتثبيت:**
> ```bash
> flutter pub get
> flutter pub run build_runner build
> ```
