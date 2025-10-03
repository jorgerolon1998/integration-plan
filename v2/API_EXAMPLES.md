# 💻 **EJEMPLOS DE USO DE APIs**

## 🎯 **EJEMPLOS DE USER SERVICE API**

### **Crear Usuario**

```bash
# Request
curl -X POST https://api.bank.com/api/v1/users \
  -H "Content-Type: application/json" \
  -H "X-Tenant-ID: tenant-123" \
  -d '{
    "email": "usuario@ejemplo.com",
    "password": "SecurePass123!",
    "profile": {
      "firstName": "Juan",
      "lastName": "Pérez",
      "phone": "+5491123456789",
      "address": {
        "street": "Av. Corrientes 1234",
        "city": "Buenos Aires",
        "postalCode": "1043",
        "country": "AR"
      }
    },
    "tenantId": "tenant-123"
  }'

# Response (201 Created)
{
  "userId": "user-456",
  "email": "usuario@ejemplo.com",
  "profile": {
    "firstName": "Juan",
    "lastName": "Pérez",
    "phone": "+5491123456789",
    "address": {
      "street": "Av. Corrientes 1234",
      "city": "Buenos Aires",
      "postalCode": "1043",
      "country": "AR"
    }
  },
  "status": "active",
  "createdAt": "2024-01-15T10:30:00Z",
  "tenantId": "tenant-123"
}
```

### **Login de Usuario**

```bash
# Request
curl -X POST https://api.bank.com/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "usuario@ejemplo.com",
    "password": "SecurePass123!",
    "tenantId": "tenant-123"
  }'

# Response (200 OK)
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "tokenType": "Bearer",
  "expiresIn": 3600,
  "user": {
    "userId": "user-456",
    "email": "usuario@ejemplo.com",
    "profile": {
      "firstName": "Juan",
      "lastName": "Pérez"
    }
  }
}
```

---

## 💰 **EJEMPLOS DE ACCOUNT SERVICE API**

### **Crear Cuenta**

```bash
# Request
curl -X POST https://api.bank.com/api/v1/accounts \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "X-Tenant-ID: tenant-123" \
  -d '{
    "userId": "user-456",
    "accountType": "checking",
    "currency": "ARS",
    "tenantId": "tenant-123"
  }'

# Response (201 Created)
{
  "accountId": "acc-789",
  "userId": "user-456",
  "accountNumber": "1234567890",
  "accountType": "checking",
  "currency": "ARS",
  "balance": 0.00,
  "status": "active",
  "createdAt": "2024-01-15T10:35:00Z",
  "tenantId": "tenant-123"
}
```

### **Consultar Balance**

```bash
# Request
curl -X GET https://api.bank.com/api/v1/accounts/acc-789/balance \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "X-Tenant-ID: tenant-123"

# Response (200 OK)
{
  "accountId": "acc-789",
  "balance": 150000.50,
  "currency": "ARS",
  "availableBalance": 150000.50,
  "pendingTransactions": 0.00,
  "lastUpdated": "2024-01-15T14:22:30Z"
}
```

---

## 💳 **EJEMPLOS DE PAYMENT SERVICE API**

### **Procesar Pago**

```bash
# Request
curl -X POST https://api.bank.com/api/v1/payments \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "X-Tenant-ID: tenant-123" \
  -d '{
    "amount": 50000.00,
    "currency": "ARS",
    "paymentMethod": {
      "type": "credit_card",
      "provider": "mercadopago",
      "token": "card_token_123",
      "installments": 1
    },
    "recipientAccount": "acc-789",
    "description": "Pago de servicios",
    "tenantId": "tenant-123"
  }'

# Response (201 Created)
{
  "paymentId": "pay-101",
  "status": "processing",
  "amount": 50000.00,
  "currency": "ARS",
  "paymentMethod": {
    "type": "credit_card",
    "provider": "mercadopago",
    "lastFourDigits": "1234"
  },
  "recipientAccount": "acc-789",
  "description": "Pago de servicios",
  "createdAt": "2024-01-15T15:45:00Z",
  "estimatedCompletion": "2024-01-15T15:47:00Z",
  "tenantId": "tenant-123"
}
```

### **Consultar Estado de Pago**

```bash
# Request
curl -X GET https://api.bank.com/api/v1/payments/pay-101 \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "X-Tenant-ID: tenant-123"

# Response (200 OK)
{
  "paymentId": "pay-101",
  "status": "completed",
  "amount": 50000.00,
  "currency": "ARS",
  "paymentMethod": {
    "type": "credit_card",
    "provider": "mercadopago",
    "lastFourDigits": "1234"
  },
  "recipientAccount": "acc-789",
  "description": "Pago de servicios",
  "createdAt": "2024-01-15T15:45:00Z",
  "completedAt": "2024-01-15T15:46:30Z",
  "transactionId": "txn-202",
  "tenantId": "tenant-123"
}
```

---

## ⚠️ **EJEMPLOS DE RISK SERVICE API**

### **Evaluar Riesgo de Transacción**

```bash
# Request
curl -X POST https://api.bank.com/api/v1/risk/assess \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "X-Tenant-ID: tenant-123" \
  -d '{
    "transactionId": "pay-101",
    "userId": "user-456",
    "amount": 50000.00,
    "paymentMethod": {
      "type": "credit_card",
      "provider": "mercadopago"
    },
    "context": {
      "deviceId": "device-789",
      "ipAddress": "192.168.1.100",
      "userAgent": "Mozilla/5.0...",
      "location": {
        "latitude": -34.6037,
        "longitude": -58.3816,
        "city": "Buenos Aires",
        "country": "AR"
      }
    }
  }'

# Response (200 OK)
{
  "assessmentId": "risk-303",
  "transactionId": "pay-101",
  "riskScore": 0.25,
  "riskLevel": "low",
  "action": "approve",
  "factors": [
    {
      "factor": "amount",
      "score": 0.1,
      "description": "Amount within normal range"
    },
    {
      "factor": "location",
      "score": 0.15,
      "description": "Location matches user profile"
    },
    {
      "factor": "device",
      "score": 0.0,
      "description": "Known device"
    }
  ],
  "mlModel": "risk-v2.1",
  "assessedAt": "2024-01-15T15:45:15Z"
}
```

---

## 🛡️ **EJEMPLOS DE FRAUD SERVICE API**

### **Detectar Fraude**

```bash
# Request
curl -X POST https://api.bank.com/api/v1/fraud/detect \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "X-Tenant-ID: tenant-123" \
  -d '{
    "transactionId": "pay-101",
    "userId": "user-456",
    "deviceInfo": {
      "deviceId": "device-789",
      "fingerprint": "fp_abc123",
      "os": "iOS 17.2",
      "browser": "Safari 17.2",
      "screenResolution": "1179x2556"
    },
    "behaviorData": {
      "typingSpeed": 45,
      "mouseMovements": 120,
      "timeOnPage": 30,
      "previousTransactions": 15
    },
    "context": {
      "ipAddress": "192.168.1.100",
      "userAgent": "Mozilla/5.0...",
      "timestamp": "2024-01-15T15:45:00Z"
    }
  }'

# Response (200 OK)
{
  "detectionId": "fraud-404",
  "transactionId": "pay-101",
  "fraudScore": 0.12,
  "isFraudulent": false,
  "action": "approve",
  "indicators": [
    {
      "indicator": "device_consistency",
      "score": 0.05,
      "description": "Device matches user history"
    },
    {
      "indicator": "behavior_pattern",
      "score": 0.07,
      "description": "Behavior consistent with user profile"
    }
  ],
  "riskFactors": [],
  "detectedAt": "2024-01-15T15:45:20Z"
}
```

---

## 📧 **EJEMPLOS DE NOTIFICATION SERVICE API**

### **Enviar Notificación**

```bash
# Request
curl -X POST https://api.bank.com/api/v1/notifications \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "X-Tenant-ID: tenant-123" \
  -d '{
    "userId": "user-456",
    "type": "payment_confirmation",
    "channel": "email",
    "template": "payment_success",
    "data": {
      "paymentId": "pay-101",
      "amount": 50000.00,
      "currency": "ARS",
      "recipient": "Servicios Públicos",
      "date": "2024-01-15T15:46:30Z"
    }
  }'

# Response (201 Created)
{
  "notificationId": "notif-505",
  "userId": "user-456",
  "type": "payment_confirmation",
  "channel": "email",
  "status": "sent",
  "template": "payment_success",
  "sentAt": "2024-01-15T15:47:00Z",
  "deliveryId": "delivery-606"
}
```

---

## 🔌 **EJEMPLOS DE OPEN FINANCE API**

### **Obtener Datos de Cuentas (con Consentimiento)**

```bash
# Request
curl -X GET "https://api.bank.com/api/v1/openfinance/accounts?consentId=consent-707" \
  -H "Authorization: Bearer oauth_token_123" \
  -H "X-Third-Party-ID: fintech-abc"

# Response (200 OK)
{
  "consentId": "consent-707",
  "accounts": [
    {
      "accountId": "acc-789",
      "accountNumber": "1234567890",
      "accountType": "checking",
      "currency": "ARS",
      "balance": 150000.50,
      "status": "active"
    }
  ],
  "retrievedAt": "2024-01-15T16:00:00Z",
  "thirdPartyId": "fintech-abc"
}
```

### **Crear Consentimiento**

```bash
# Request
curl -X POST https://api.bank.com/api/v1/openfinance/consent \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -d '{
    "userId": "user-456",
    "thirdPartyId": "fintech-abc",
    "scope": ["accounts", "transactions"],
    "duration": "30d",
    "description": "Acceso a datos para análisis financiero"
  }'

# Response (201 Created)
{
  "consentId": "consent-707",
  "userId": "user-456",
  "thirdPartyId": "fintech-abc",
  "scope": ["accounts", "transactions"],
  "status": "active",
  "expiresAt": "2024-02-14T16:00:00Z",
  "createdAt": "2024-01-15T16:00:00Z"
}
```

---

## 📊 **EJEMPLOS DE HEALTH CHECK API**

### **Health Check General**

```bash
# Request
curl -X GET https://api.bank.com/health

# Response (200 OK)
{
  "status": "healthy",
  "timestamp": "2024-01-15T16:30:00Z",
  "version": "1.2.3",
  "uptime": "72h15m30s",
  "dependencies": {
    "database": "healthy",
    "cache": "healthy",
    "external_apis": "healthy"
  }
}
```

### **Health Check de Dependencias**

```bash
# Request
curl -X GET https://api.bank.com/health/dependencies

# Response (200 OK)
{
  "status": "healthy",
  "dependencies": [
    {
      "name": "postgresql",
      "status": "healthy",
      "responseTime": "15ms",
      "lastChecked": "2024-01-15T16:30:00Z"
    },
    {
      "name": "redis",
      "status": "healthy",
      "responseTime": "2ms",
      "lastChecked": "2024-01-15T16:30:00Z"
    },
    {
      "name": "mercadopago_api",
      "status": "healthy",
      "responseTime": "120ms",
      "lastChecked": "2024-01-15T16:30:00Z"
    }
  ]
}
```

---

## 🔧 **EJEMPLOS DE CONFIGURACIÓN**

### **Variables de Entorno**

```bash
# Configuración para desarrollo
export API_BASE_URL=https://api-dev.bank.com
export TENANT_ID=tenant-123
export JWT_SECRET=your-jwt-secret
export DB_HOST=localhost
export DB_PORT=5432
export DB_NAME=banking_db
export REDIS_HOST=localhost
export REDIS_PORT=6379
```

### **Headers Comunes**

```bash
# Headers para todas las requests
-H "Content-Type: application/json"
-H "Authorization: Bearer {jwt_token}"
-H "X-Tenant-ID: {tenant_id}"
-H "X-Request-ID: {unique_request_id}"
-H "User-Agent: MyApp/1.0.0"
```

---

## 📝 **NOTAS IMPORTANTES**

### **Rate Limiting**
- **Límite por usuario**: 1000 requests/hora
- **Límite por tenant**: 10000 requests/hora
- **Límite por IP**: 5000 requests/hora

### **Códigos de Error Comunes**
- **400**: Bad Request - Datos inválidos
- **401**: Unauthorized - Token inválido o expirado
- **403**: Forbidden - Sin permisos para la operación
- **404**: Not Found - Recurso no encontrado
- **429**: Too Many Requests - Rate limit excedido
- **500**: Internal Server Error - Error interno del servidor

### **Paginación**
```bash
# Ejemplo de paginación
GET /api/v1/accounts/user/123/transactions?page=1&limit=20&sort=date&order=desc
```

---

**Estos ejemplos proporcionan una guía completa para interactuar con todas las APIs del sistema.**
