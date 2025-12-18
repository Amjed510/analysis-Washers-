# ✅ Developer Checklist - نظام WashPOS

> [!IMPORTANT]
> قائمة تحقق شاملة للمطورين قبل وأثناء وبعد التطوير  
> **الهدف:** ضمان جودة التطبيق وسلامة العمليات المالية

---

## 📋 جدول المحتويات

1. [قبل البدء (Pre-Development)](#1-قبل-البدء-pre-development)
2. [إعداد البيئة (Environment Setup)](#2-إعداد-البيئة-environment-setup)
3. [Core Features](#3-core-features)
4. [Payment Integration](#4-payment-integration)
5. [Offline & Sync](#5-offline--sync)
6. [UI/UX Requirements](#6-uiux-requirements)
7. [Security](#7-security)
8. [Testing](#8-testing)
9. [Pre-Production](#9-pre-production)
10. [Deployment](#10-deployment)

---

## 1. قبل البدء (Pre-Development)

### 1.1 المتطلبات الأساسية

| # | المتطلب | ✓ |
|---|---------|---|
| 1.1.1 | Flutter SDK 3.19+ مثبت | ☐ |
| 1.1.2 | Android Studio / VS Code مع Flutter plugins | ☐ |
| 1.1.3 | جهاز أندرويد للاختبار (API 24+) | ☐ |
| 1.1.4 | حساب مطور على Google Play (إذا لزم) | ☐ |
| 1.1.5 | الوصول إلى Sandbox Server | ☐ |

### 1.2 الأجهزة الخاصة

| # | الجهاز | ✓ |
|---|--------|---|
| 1.2.1 | SUNMI V2s متاح للاختبار | ☐ |
| 1.2.2 | جهاز دفع Alinma Bank (sandbox) | ☐ |
| 1.2.3 | ورق طابعة حرارية | ☐ |
| 1.2.4 | SIM Card للإنترنت (إذا لزم) | ☐ |

### 1.3 الوثائق المطلوبة

| # | الوثيقة | ✓ |
|---|---------|---|
| 1.3.1 | API Documentation | ☐ |
| 1.3.2 | SUNMI SDK Documentation | ☐ |
| 1.3.3 | Alinma Payment SDK Docs | ☐ |
| 1.3.4 | ZATCA QR Code Specifications | ☐ |

---

## 2. إعداد البيئة (Environment Setup)

### 2.1 Project Setup

| # | الخطوة | الأمر / الملاحظة | ✓ |
|---|--------|------------------|---|
| 2.1.1 | إنشاء المشروع | `flutter create washpos` | ☐ |
| 2.1.2 | تثبيت المكتبات | `flutter pub get` | ☐ |
| 2.1.3 | تشغيل build_runner | `flutter pub run build_runner build` | ☐ |
| 2.1.4 | إعداد Flavors (dev/staging/prod) | Android Gradle | ☐ |

### 2.2 Dependencies (pubspec.yaml)

```yaml
# تحقق من وجود المكتبات التالية:
```

| # | المكتبة | الإصدار | ✓ |
|---|---------|---------|---|
| 2.2.1 | get | ^4.6.6 | ☐ |
| 2.2.2 | isar | ^3.1.0 | ☐ |
| 2.2.3 | isar_flutter_libs | ^3.1.0 | ☐ |
| 2.2.4 | dio | ^5.4.0 | ☐ |
| 2.2.5 | connectivity_plus | ^5.0.2 | ☐ |
| 2.2.6 | sunmi_printer_plus | ^1.3.5 | ☐ |
| 2.2.7 | uuid | ^4.2.2 | ☐ |
| 2.2.8 | intl | ^0.18.1 | ☐ |
| 2.2.9 | json_annotation | ^4.8.1 | ☐ |
| 2.2.10 | crypto | ^3.0.3 | ☐ |

### 2.3 Environment Variables

| # | المتغير | الوصف | ✓ |
|---|---------|-------|---|
| 2.3.1 | API_BASE_URL | عنوان السيرفر | ☐ |
| 2.3.2 | MERCHANT_ID | معرف التاجر | ☐ |
| 2.3.3 | TERMINAL_ID | معرف الجهاز | ☐ |
| 2.3.4 | ENCRYPTION_KEY | مفتاح التشفير | ☐ |

---

## 3. Core Features

### 3.1 Database (Isar)

| # | الميزة | الوصف | ✓ |
|---|--------|-------|---|
| 3.1.1 | LocalTransaction schema | جدول المعاملات المحلية | ☐ |
| 3.1.2 | CachedProduct schema | جدول المنتجات المخزنة | ☐ |
| 3.1.3 | User schema | جدول المستخدمين | ☐ |
| 3.1.4 | Database initialization | فتح قاعدة البيانات | ☐ |
| 3.1.5 | Indexes | فهارس للبحث السريع | ☐ |

### 3.2 Products

| # | الميزة | الوصف | ✓ |
|---|--------|-------|---|
| 3.2.1 | Fetch products from API | جلب المنتجات | ☐ |
| 3.2.2 | Cache products locally | تخزين محلي | ☐ |
| 3.2.3 | Filter by category | فلترة حسب النوع | ☐ |
| 3.2.4 | Filter by car size | فلترة حسب الحجم | ☐ |
| 3.2.5 | Price validation | التحقق من السعر | ☐ |

### 3.3 Cart / Order

| # | الميزة | الوصف | ✓ |
|---|--------|-------|---|
| 3.3.1 | Add item to cart | إضافة عنصر | ☐ |
| 3.3.2 | Remove item from cart | حذف عنصر | ☐ |
| 3.3.3 | Calculate subtotal | حساب المجموع الفرعي | ☐ |
| 3.3.4 | Calculate tax (15% VAT) | حساب الضريبة | ☐ |
| 3.3.5 | Calculate total | حساب الإجمالي | ☐ |
| 3.3.6 | Clear cart | إفراغ السلة | ☐ |

---

## 4. Payment Integration

### 4.1 Cash Payment

| # | الميزة | الوصف | ✓ |
|---|--------|-------|---|
| 4.1.1 | Cash button | زر الدفع نقداً | ☐ |
| 4.1.2 | Create local invoice | إنشاء فاتورة محلية | ☐ |
| 4.1.3 | Create local receipt | إنشاء سند محلي | ☐ |
| 4.1.4 | Add to sync queue | إضافة لقائمة المزامنة | ☐ |
| 4.1.5 | Print receipt | طباعة الإيصال | ☐ |
| 4.1.6 | Success feedback | تأكيد النجاح | ☐ |

### 4.2 Card Payment

| # | الميزة | الوصف | ✓ |
|---|--------|-------|---|
| 4.2.1 | Check internet connection | فحص الاتصال | ☐ |
| 4.2.2 | Card button (disabled if offline) | زر البطاقة | ☐ |
| 4.2.3 | Initialize terminal SDK | تهيئة SDK | ☐ |
| 4.2.4 | Send payment request | إرسال طلب الدفع | ☐ |
| 4.2.5 | Handle terminal response | معالجة الرد | ☐ |
| 4.2.6 | Create invoice on success | إنشاء الفاتورة | ☐ |
| 4.2.7 | Print receipt | طباعة الإيصال | ☐ |
| 4.2.8 | Handle decline | معالجة الرفض | ☐ |
| 4.2.9 | Handle timeout | معالجة انتهاء المهلة | ☐ |

### 4.3 Payment Terminal

| # | الميزة | الوصف | ✓ |
|---|--------|-------|---|
| 4.3.1 | Terminal initialization | تهيئة الجهاز | ☐ |
| 4.3.2 | Connection check | فحص الاتصال بالجهاز | ☐ |
| 4.3.3 | Amount formatting (in Halalas) | تنسيق المبلغ | ☐ |
| 4.3.4 | Transaction timeout (120s) | مهلة المعاملة | ☐ |
| 4.3.5 | Capture auth code | التقاط رمز التفويض | ☐ |
| 4.3.6 | Capture reference number | التقاط رقم المرجع | ☐ |

---

## 5. Offline & Sync

### 5.1 Offline Queue

| # | الميزة | الوصف | ✓ |
|---|--------|-------|---|
| 5.1.1 | Enqueue transaction | إضافة للقائمة | ☐ |
| 5.1.2 | Get pending transactions | جلب المعلق | ☐ |
| 5.1.3 | Update transaction status | تحديث الحالة | ☐ |
| 5.1.4 | Priority ordering | ترتيب الأولوية | ☐ |
| 5.1.5 | Queue size limit | حد حجم القائمة | ☐ |

### 5.2 Sync Service

| # | الميزة | الوصف | ✓ |
|---|--------|-------|---|
| 5.2.1 | Trigger on internet connect | المحفز عند الاتصال | ☐ |
| 5.2.2 | Periodic sync (5 min) | مزامنة دورية | ☐ |
| 5.2.3 | Sync on app foreground | عند فتح التطبيق | ☐ |
| 5.2.4 | Manual sync button | زر مزامنة يدوي | ☐ |
| 5.2.5 | Idempotency key | منع التكرار | ☐ |
| 5.2.6 | Retry with exponential backoff | إعادة المحاولة | ☐ |
| 5.2.7 | Max retry limit (5) | حد المحاولات | ☐ |
| 5.2.8 | Failed transaction notification | إشعار الفشل | ☐ |

### 5.3 Connectivity

| # | الميزة | الوصف | ✓ |
|---|--------|-------|---|
| 5.3.1 | Listen to connectivity changes | مراقبة الاتصال | ☐ |
| 5.3.2 | Server reachability check | فحص الوصول للسيرفر | ☐ |
| 5.3.3 | Connection status indicator | مؤشر الحالة | ☐ |
| 5.3.4 | Offline mode detection | اكتشاف وضع عدم الاتصال | ☐ |

---

## 6. UI/UX Requirements

### 6.1 شاشة الخدمات

| # | المتطلب | المعيار | ✓ |
|---|---------|---------|---|
| 6.1.1 | Service cards grid | 2 أعمدة | ☐ |
| 6.1.2 | Card touch target | 48dp minimum | ☐ |
| 6.1.3 | Press animation | scale + highlight | ☐ |
| 6.1.4 | Icon size | واضحة ومقروءة | ☐ |
| 6.1.5 | Card labels | عربي + واضح | ☐ |

### 6.2 Bottom Sheet (حجم السيارة)

| # | المتطلب | المعيار | ✓ |
|---|---------|---------|---|
| 6.2.1 | Bottom sheet animation | smooth slide | ☐ |
| 6.2.2 | Size options | 3 خيارات واضحة | ☐ |
| 6.2.3 | Price display | مع كل خيار | ☐ |
| 6.2.4 | Direct navigation | للدفع مباشرة | ☐ |

### 6.3 شاشة الدفع

| # | المتطلب | المعيار | ✓ |
|---|---------|---------|---|
| 6.3.1 | Order summary | Service + Size + Price | ☐ |
| 6.3.2 | Price font size | 48sp bold | ☐ |
| 6.3.3 | Payment method selection | Radio buttons | ☐ |
| 6.3.4 | Pay button | 56dp, green | ☐ |
| 6.3.5 | Debounce (prevent double tap) | 500ms | ☐ |

### 6.4 Loading & Errors

| # | المتطلب | المعيار | ✓ |
|---|---------|---------|---|
| 6.4.1 | Processing overlay | Block UI | ☐ |
| 6.4.2 | Prevent back button | During processing | ☐ |
| 6.4.3 | Clear loading indicator | Spinner | ☐ |
| 6.4.4 | Progress for card payment | If available | ☐ |
| 6.4.5 | Error messages | Arabic, clear | ☐ |
| 6.4.6 | Retry button | On error | ☐ |

### 6.5 شاشة النجاح

| # | المتطلب | المعيار | ✓ |
|---|---------|---------|---|
| 6.5.1 | Success animation | Bounce checkmark | ☐ |
| 6.5.2 | Invoice number | Displayed | ☐ |
| 6.5.3 | New order button | Primary | ☐ |
| 6.5.4 | Reprint button | Secondary | ☐ |
| 6.5.5 | Success sound | Optional beep | ☐ |

---

## 7. Security

### 7.1 Data Security

| # | المتطلب | الوصف | ✓ |
|---|---------|-------|---|
| 7.1.1 | No hardcoded credentials | لا مفاتيح في الكود | ☐ |
| 7.1.2 | Environment variables | متغيرات البيئة | ☐ |
| 7.1.3 | Secure storage for tokens | FlutterSecureStorage | ☐ |
| 7.1.4 | Certificate pinning | للاتصالات الحساسة | ☐ |
| 7.1.5 | HTTPS only | جميع الاتصالات | ☐ |

### 7.2 Payment Security

| # | المتطلب | الوصف | ✓ |
|---|---------|-------|---|
| 7.2.1 | Price from product only | السعر من المنتج فقط | ☐ |
| 7.2.2 | No client-side price editing | منع تعديل السعر | ☐ |
| 7.2.3 | Transaction signing | توقيع المعاملات | ☐ |
| 7.2.4 | Idempotency keys | منع التكرار | ☐ |
| 7.2.5 | PCI DSS compliance | متطلبات البطاقات | ☐ |

### 7.3 App Security

| # | المتطلب | الوصف | ✓ |
|---|---------|-------|---|
| 7.3.1 | Biometric login option | البصمة | ☐ |
| 7.3.2 | Session timeout | انتهاء الجلسة | ☐ |
| 7.3.3 | Code obfuscation | تشفير الكود | ☐ |
| 7.3.4 | Disable screenshots on sensitive screens | منع التصوير | ☐ |

---

## 8. Testing

### 8.1 Unit Tests

| # | الاختبار | التغطية | ✓ |
|---|----------|---------|---|
| 8.1.1 | Price calculation | Subtotal, Tax, Total | ☐ |
| 8.1.2 | Idempotency key generation | Format, Uniqueness | ☐ |
| 8.1.3 | Transaction validation | Required fields | ☐ |
| 8.1.4 | Retry logic | Backoff timing | ☐ |

### 8.2 Integration Tests

| # | الاختبار | السيناريو | ✓ |
|---|----------|----------|---|
| 8.2.1 | Cash payment flow | Complete flow | ☐ |
| 8.2.2 | Card payment flow | Success + Decline | ☐ |
| 8.2.3 | Offline queue | Add + Sync | ☐ |
| 8.2.4 | Print receipt | Full receipt | ☐ |

### 8.3 Device Tests

| # | الجهاز | السيناريو | ✓ |
|---|--------|----------|---|
| 8.3.1 | SUNMI V2s | Full app flow | ☐ |
| 8.3.2 | Payment terminal | Card payment | ☐ |
| 8.3.3 | Thermal printer | Print quality | ☐ |
| 8.3.4 | Low battery | Behavior | ☐ |
| 8.3.5 | Network switch | WiFi to Mobile | ☐ |

---

## 9. Pre-Production

### 9.1 Performance

| # | المتطلب | المعيار | ✓ |
|---|---------|---------|---|
| 9.1.1 | App launch time | < 3 seconds | ☐ |
| 9.1.2 | Screen transition | < 300ms | ☐ |
| 9.1.3 | Button response | < 100ms | ☐ |
| 9.1.4 | Payment to print | < 5 seconds | ☐ |
| 9.1.5 | Memory usage | < 150MB | ☐ |

### 9.2 Error Handling

| # | السيناريو | السلوك المتوقع | ✓ |
|---|----------|----------------|---|
| 9.2.1 | Network disconnection | Show indicator, save locally | ☐ |
| 9.2.2 | API timeout | Show retry option | ☐ |
| 9.2.3 | Card declined | Show reason + alternatives | ☐ |
| 9.2.4 | Printer paper out | Show warning | ☐ |
| 9.2.5 | App crash recovery | Restore pending transaction | ☐ |

### 9.3 Localization

| # | المتطلب | الوصف | ✓ |
|---|---------|-------|---|
| 9.3.1 | Arabic language | الواجهة الكاملة | ☐ |
| 9.3.2 | RTL layout | اتجاه من اليمين | ☐ |
| 9.3.3 | Arabic numbers | الأرقام العربية | ☐ |
| 9.3.4 | Date format | تنسيق التاريخ | ☐ |
| 9.3.5 | Currency format | ر.س | ☐ |

---

## 10. Deployment

### 10.1 Pre-Deployment

| # | الخطوة | الوصف | ✓ |
|---|--------|-------|---|
| 10.1.1 | Code review | مراجعة الكود | ☐ |
| 10.1.2 | All tests passing | جميع الاختبارات | ☐ |
| 10.1.3 | Version bump | تحديث الإصدار | ☐ |
| 10.1.4 | Changelog update | سجل التغييرات | ☐ |
| 10.1.5 | Production API URL | عنوان الإنتاج | ☐ |

### 10.2 Build

| # | الخطوة | الأمر | ✓ |
|---|--------|-------|---|
| 10.2.1 | Clean build | `flutter clean` | ☐ |
| 10.2.2 | Get dependencies | `flutter pub get` | ☐ |
| 10.2.3 | Build APK | `flutter build apk --release` | ☐ |
| 10.2.4 | Sign APK | With production key | ☐ |
| 10.2.5 | Test release build | On device | ☐ |

### 10.3 Post-Deployment

| # | الخطوة | الوصف | ✓ |
|---|--------|-------|---|
| 10.3.1 | Install on devices | جميع الأجهزة | ☐ |
| 10.3.2 | Verify connectivity | فحص الاتصال | ☐ |
| 10.3.3 | Test payment flow | اختبار الدفع | ☐ |
| 10.3.4 | Monitor first transactions | مراقبة أول المعاملات | ☐ |
| 10.3.5 | Backup plan ready | خطة طوارئ | ☐ |

---

## 📊 Progress Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Progress Tracker                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Section               Total    Done    Progress                          │
│   ─────────────────────────────────────────────────────                    │
│   1. Pre-Development      12      ☐       ░░░░░░░░░░  0%                  │
│   2. Environment          22      ☐       ░░░░░░░░░░  0%                  │
│   3. Core Features        17      ☐       ░░░░░░░░░░  0%                  │
│   4. Payment              19      ☐       ░░░░░░░░░░  0%                  │
│   5. Offline & Sync       16      ☐       ░░░░░░░░░░  0%                  │
│   6. UI/UX                21      ☐       ░░░░░░░░░░  0%                  │
│   7. Security             12      ☐       ░░░░░░░░░░  0%                  │
│   8. Testing              14      ☐       ░░░░░░░░░░  0%                  │
│   9. Pre-Production       15      ☐       ░░░░░░░░░░  0%                  │
│   10. Deployment          13      ☐       ░░░░░░░░░░  0%                  │
│   ─────────────────────────────────────────────────────                    │
│   TOTAL                  161      0       ░░░░░░░░░░  0%                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

> [!CAUTION]
> **هام:** لا تبدأ الإنتاج قبل إكمال جميع العناصر الحرجة (4.x و 7.x)

---

> [!TIP]
> استخدم هذه القائمة كمرجع يومي واجمع التحديثات في Meeting أسبوعي
