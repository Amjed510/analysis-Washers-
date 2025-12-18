# 🔌 API Contract - نظام WashPOS

> [!IMPORTANT]
> عقد الـ API بين التطبيق والخادم  
> **الإصدار:** v1.0  
> **Base URL:** `https://api.washpos.sa/v1`

---

## 📋 جدول المحتويات

1. [المصادقة (Authentication)](#1-المصادقة-authentication)
2. [المنتجات والخدمات (Products)](#2-المنتجات-والخدمات-products)
3. [الفواتير (Invoices)](#3-الفواتير-invoices)
4. [سندات القبض (Receipts)](#4-سندات-القبض-receipts)
5. [الدفع (Payments)](#5-الدفع-payments)
6. [المزامنة (Sync)](#6-المزامنة-sync)
7. [الأخطاء (Errors)](#7-الأخطاء-errors)

---

## 🔐 Headers المشتركة

```http
Content-Type: application/json
Accept: application/json
Accept-Language: ar
Authorization: Bearer {access_token}
X-Terminal-ID: {terminal_id}
X-Idempotency-Key: {unique_key}
```

---

## 1. المصادقة (Authentication)

### 1.1 تسجيل الدخول

```http
POST /auth/login
```

**Request:**
```json
{
  "username": "cashier1",
  "password": "password123",
  "terminal_id": "TERM-001"
}
```

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "USR-001",
      "name": "أحمد محمد",
      "role": "cashier",
      "branch_id": "BR-001"
    },
    "access_token": "eyJhbGciOiJIUzI1NiIs...",
    "refresh_token": "eyJhbGciOiJIUzI1NiIs...",
    "expires_in": 3600,
    "shift": {
      "id": "SH-001",
      "started_at": "2024-12-18T08:00:00Z",
      "type": "morning"
    }
  }
}
```

**Response: 401 Unauthorized**
```json
{
  "success": false,
  "error": {
    "code": "INVALID_CREDENTIALS",
    "message": "اسم المستخدم أو كلمة المرور غير صحيحة"
  }
}
```

### 1.2 تجديد التوكن

```http
POST /auth/refresh
```

**Request:**
```json
{
  "refresh_token": "eyJhbGciOiJIUzI1NiIs..."
}
```

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIs...",
    "expires_in": 3600
  }
}
```

---

## 2. المنتجات والخدمات (Products)

### 2.1 جلب جميع الخدمات

```http
GET /products
```

**Query Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| category | string | No | wash, steam, express, polish, addon |
| car_size | string | No | small, medium, large |
| active | boolean | No | Filter by active status |

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "products": [
      {
        "id": "PRD-001",
        "name": "غسيل تركواز صغير",
        "name_en": "Turquoise Wash Small",
        "category": "wash",
        "car_size": "small",
        "price": 70.00,
        "tax_rate": 0.15,
        "icon": "turquoise",
        "is_active": true,
        "created_at": "2024-01-01T00:00:00Z",
        "updated_at": "2024-12-01T00:00:00Z"
      },
      {
        "id": "PRD-002",
        "name": "غسيل تركواز متوسط",
        "name_en": "Turquoise Wash Medium",
        "category": "wash",
        "car_size": "medium",
        "price": 75.00,
        "tax_rate": 0.15,
        "icon": "turquoise",
        "is_active": true,
        "created_at": "2024-01-01T00:00:00Z",
        "updated_at": "2024-12-01T00:00:00Z"
      }
    ],
    "meta": {
      "total": 35,
      "last_updated": "2024-12-18T10:00:00Z"
    }
  }
}
```

### 2.2 جلب منتج واحد

```http
GET /products/{product_id}
```

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "id": "PRD-001",
    "name": "غسيل تركواز صغير",
    "name_en": "Turquoise Wash Small",
    "category": "wash",
    "car_size": "small",
    "price": 70.00,
    "tax_rate": 0.15,
    "icon": "turquoise",
    "description": "غسيل شامل للسيارة مع تلميع خارجي",
    "is_active": true
  }
}
```

---

## 3. الفواتير (Invoices)

### 3.1 إنشاء فاتورة

```http
POST /invoices
```

**Headers (Required):**
```http
X-Idempotency-Key: ORD-{timestamp}-{uuid}
```

**Request:**
```json
{
  "local_id": "LOCAL-550e8400-e29b-41d4-a716-446655440000",
  "items": [
    {
      "product_id": "PRD-001",
      "quantity": 1,
      "unit_price": 70.00,
      "total": 70.00
    }
  ],
  "subtotal": 70.00,
  "tax_amount": 10.50,
  "total_amount": 80.50,
  "payment_method": "cash",
  "terminal_id": "TERM-001",
  "cashier_id": "USR-001",
  "created_at_local": "2024-12-18T14:30:00+03:00"
}
```

**Response: 201 Created**
```json
{
  "success": true,
  "data": {
    "id": "INV-2024-001234",
    "local_id": "LOCAL-550e8400-e29b-41d4-a716-446655440000",
    "invoice_number": "INV-2024-001234",
    "status": "completed",
    "items": [
      {
        "product_id": "PRD-001",
        "product_name": "غسيل تركواز صغير",
        "quantity": 1,
        "unit_price": 70.00,
        "total": 70.00
      }
    ],
    "subtotal": 70.00,
    "tax_rate": 0.15,
    "tax_amount": 10.50,
    "total_amount": 80.50,
    "payment_method": "cash",
    "qr_code": "AQZhbGVnYW4bD0FsLVNhbGFtIENhciBXYXNo...",
    "zatca_hash": "HjFb+/x8fJP...",
    "created_at": "2024-12-18T14:30:00Z"
  }
}
```

**Response: 409 Conflict (Idempotent Return)**
```json
{
  "success": true,
  "data": {
    "id": "INV-2024-001234",
    "message": "Invoice already exists",
    "idempotent": true
  }
}
```

### 3.2 جلب فاتورة

```http
GET /invoices/{invoice_id}
```

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "id": "INV-2024-001234",
    "invoice_number": "INV-2024-001234",
    "status": "completed",
    "items": [...],
    "subtotal": 70.00,
    "tax_amount": 10.50,
    "total_amount": 80.50,
    "payment": {
      "method": "cash",
      "status": "completed",
      "paid_amount": 100.00,
      "change_amount": 19.50
    },
    "receipt": {
      "id": "REC-2024-001234",
      "type": "cash"
    },
    "created_at": "2024-12-18T14:30:00Z"
  }
}
```

### 3.3 التحقق من حالة فاتورة بـ Idempotency Key

```http
GET /invoices/status?idempotency_key={key}
```

**Response: 200 OK (Found)**
```json
{
  "success": true,
  "data": {
    "exists": true,
    "invoice_id": "INV-2024-001234",
    "status": "completed"
  }
}
```

**Response: 200 OK (Not Found)**
```json
{
  "success": true,
  "data": {
    "exists": false
  }
}
```

---

## 4. سندات القبض (Receipts)

### 4.1 إنشاء سند قبض

```http
POST /receipts
```

**Request (Cash):**
```json
{
  "local_id": "REC-LOCAL-001",
  "invoice_id": "INV-2024-001234",
  "type": "cash",
  "amount": 80.50,
  "paid_amount": 100.00,
  "change_amount": 19.50,
  "terminal_id": "TERM-001",
  "cashier_id": "USR-001",
  "created_at_local": "2024-12-18T14:30:00+03:00"
}
```

**Request (Card):**
```json
{
  "local_id": "REC-LOCAL-002",
  "invoice_id": "INV-2024-001235",
  "type": "card",
  "amount": 80.50,
  "card_details": {
    "card_type": "mada",
    "last_four": "1234",
    "auth_code": "123456",
    "reference_no": "REF-001234",
    "terminal_ref": "TRXN-001"
  },
  "terminal_id": "TERM-001",
  "cashier_id": "USR-001",
  "created_at_local": "2024-12-18T14:30:00+03:00"
}
```

**Response: 201 Created**
```json
{
  "success": true,
  "data": {
    "id": "REC-2024-001234",
    "local_id": "REC-LOCAL-001",
    "invoice_id": "INV-2024-001234",
    "receipt_number": "REC-2024-001234",
    "type": "cash",
    "amount": 80.50,
    "status": "completed",
    "created_at": "2024-12-18T14:30:00Z"
  }
}
```

---

## 5. الدفع (Payments)

### 5.1 بدء عملية دفع بالبطاقة

```http
POST /payments/card/initiate
```

**Request:**
```json
{
  "invoice_id": "INV-2024-001234",
  "amount": 80.50,
  "terminal_id": "TERM-001",
  "idempotency_key": "PAY-001234"
}
```

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "payment_id": "PAY-2024-001234",
    "status": "pending",
    "terminal_command": {
      "action": "process_payment",
      "amount": 8050,
      "currency": "SAR",
      "reference": "PAY-2024-001234"
    }
  }
}
```

### 5.2 تأكيد نتيجة الدفع

```http
POST /payments/card/confirm
```

**Request:**
```json
{
  "payment_id": "PAY-2024-001234",
  "terminal_response": {
    "status": "SUCCESS",
    "auth_code": "123456",
    "reference_no": "REF-001234",
    "card_type": "mada",
    "card_last_four": "1234",
    "merchant_id": "MID-001"
  }
}
```

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "payment_id": "PAY-2024-001234",
    "status": "completed",
    "receipt_id": "REC-2024-001235",
    "invoice": {
      "id": "INV-2024-001234",
      "qr_code": "..."
    }
  }
}
```

### 5.3 التحقق من حالة الدفع

```http
GET /payments/{payment_id}/status
```

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "payment_id": "PAY-2024-001234",
    "status": "completed",
    "amount": 80.50,
    "method": "card",
    "completed_at": "2024-12-18T14:31:00Z"
  }
}
```

---

## 6. المزامنة (Sync)

### 6.1 مزامنة دفعة من المعاملات

```http
POST /sync/batch
```

**Request:**
```json
{
  "transactions": [
    {
      "local_id": "LOCAL-001",
      "type": "invoice",
      "data": {
        "items": [...],
        "total_amount": 80.50,
        "payment_method": "cash"
      },
      "created_at_local": "2024-12-18T14:30:00+03:00",
      "idempotency_key": "ORD-001"
    },
    {
      "local_id": "LOCAL-002",
      "type": "receipt",
      "data": {
        "invoice_local_id": "LOCAL-001",
        "amount": 80.50,
        "type": "cash"
      },
      "created_at_local": "2024-12-18T14:30:00+03:00",
      "idempotency_key": "REC-001"
    }
  ]
}
```

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "processed": 2,
    "results": [
      {
        "local_id": "LOCAL-001",
        "server_id": "INV-2024-001234",
        "status": "synced"
      },
      {
        "local_id": "LOCAL-002",
        "server_id": "REC-2024-001234",
        "status": "synced"
      }
    ],
    "failed": []
  }
}
```

**Response: 207 Multi-Status (Partial Success)**
```json
{
  "success": true,
  "data": {
    "processed": 2,
    "results": [
      {
        "local_id": "LOCAL-001",
        "server_id": "INV-2024-001234",
        "status": "synced"
      }
    ],
    "failed": [
      {
        "local_id": "LOCAL-002",
        "error": {
          "code": "INVALID_INVOICE",
          "message": "Invoice reference not found"
        }
      }
    ]
  }
}
```

### 6.2 جلب آخر التحديثات

```http
GET /sync/updates?since={timestamp}
```

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "products": {
      "updated": [...],
      "deleted_ids": []
    },
    "settings": {
      "tax_rate": 0.15,
      "offline_limit": 100.00
    },
    "last_sync": "2024-12-18T14:35:00Z"
  }
}
```

---

## 7. الأخطاء (Errors)

### 7.1 شكل الخطأ

```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "رسالة الخطأ بالعربية",
    "message_en": "Error message in English",
    "details": {
      "field": "additional_info"
    }
  }
}
```

### 7.2 أكواد الأخطاء

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `INVALID_CREDENTIALS` | 401 | اسم المستخدم أو كلمة المرور غير صحيحة |
| `TOKEN_EXPIRED` | 401 | انتهت صلاحية التوكن |
| `UNAUTHORIZED` | 403 | غير مصرح بالوصول |
| `NOT_FOUND` | 404 | المورد غير موجود |
| `VALIDATION_ERROR` | 422 | خطأ في البيانات المدخلة |
| `DUPLICATE_TRANSACTION` | 409 | المعاملة موجودة مسبقاً |
| `PAYMENT_FAILED` | 400 | فشل في معالجة الدفع |
| `SERVER_ERROR` | 500 | خطأ في السيرفر |

### 7.3 Validation Error Example

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "البيانات المدخلة غير صحيحة",
    "details": {
      "items": "يجب أن تحتوي الفاتورة على عنصر واحد على الأقل",
      "total_amount": "المبلغ يجب أن يكون أكبر من صفر"
    }
  }
}
```

---

## 📊 Rate Limiting

| Endpoint | Rate Limit |
|----------|------------|
| `/auth/*` | 10 req/min |
| `/products/*` | 60 req/min |
| `/invoices/*` | 120 req/min |
| `/payments/*` | 60 req/min |
| `/sync/*` | 30 req/min |

**Response: 429 Too Many Requests**
```json
{
  "success": false,
  "error": {
    "code": "RATE_LIMITED",
    "message": "تم تجاوز الحد المسموح من الطلبات",
    "retry_after": 60
  }
}
```

---

## 🔒 Security Notes

> [!CAUTION]
> **قواعد أمان:**
> - جميع الاتصالات عبر HTTPS
> - التوكن صالح لمدة ساعة واحدة
> - Idempotency Key إلزامي للعمليات المالية
> - تشفير AES-256 للبيانات الحساسة

---

## 📝 Changelog

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2024-12-18 | Initial API Contract |

---

> [!TIP]
> استخدم Postman Collection المرفق لاختبار الـ APIs
