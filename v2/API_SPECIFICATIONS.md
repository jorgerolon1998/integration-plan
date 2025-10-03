# 📋 **ESPECIFICACIONES DETALLADAS DE APIs**

## 🎯 **USER SERVICE API**

### **Base URL**: `/api/v1/users`

```yaml
endpoints:
  # User Management
  POST /users:
    description: "Create new user"
    request:
      body:
        email: string
        password: string
        profile: object
        tenantId: string
    response:
      201: "User created successfully"
      400: "Invalid input data"
      409: "User already exists"
  
  GET /users/{userId}:
    description: "Get user by ID"
    parameters:
      userId: string (path)
    response:
      200: "User data"
      404: "User not found"
  
  PUT /users/{userId}:
    description: "Update user"
    parameters:
      userId: string (path)
    request:
      body:
        email?: string
        profile?: object
    response:
      200: "User updated"
      404: "User not found"
  
  DELETE /users/{userId}:
    description: "Delete user"
    parameters:
      userId: string (path)
    response:
      204: "User deleted"
      404: "User not found"
  
  # Authentication
  POST /auth/login:
    description: "User login"
    request:
      body:
        email: string
        password: string
        tenantId: string
    response:
      200: "Login successful + JWT token"
      401: "Invalid credentials"
  
  POST /auth/logout:
    description: "User logout"
    headers:
      Authorization: "Bearer {jwt_token}"
    response:
      200: "Logout successful"
  
  POST /auth/refresh:
    description: "Refresh JWT token"
    headers:
      Authorization: "Bearer {jwt_token}"
    response:
      200: "New JWT token"
      401: "Invalid token"
  
  # Profile Management
  GET /users/{userId}/profile:
    description: "Get user profile"
    response:
      200: "Profile data"
      404: "User not found"
  
  PUT /users/{userId}/profile:
    description: "Update user profile"
    request:
      body:
        firstName?: string
        lastName?: string
        phone?: string
        address?: object
    response:
      200: "Profile updated"
      404: "User not found"
```

---

## 💰 **ACCOUNT SERVICE API**

### **Base URL**: `/api/v1/accounts`

```yaml
endpoints:
  # Account Management
  POST /accounts:
    description: "Create new account"
    request:
      body:
        userId: string
        accountType: string
        currency: string
        tenantId: string
    response:
      201: "Account created"
      400: "Invalid input"
  
  GET /accounts/{accountId}:
    description: "Get account details"
    parameters:
      accountId: string (path)
    response:
      200: "Account data"
      404: "Account not found"
  
  GET /accounts/user/{userId}:
    description: "Get user accounts"
    parameters:
      userId: string (path)
    response:
      200: "List of accounts"
      404: "User not found"
  
  # Balance Operations
  GET /accounts/{accountId}/balance:
    description: "Get account balance"
    response:
      200: "Balance information"
      404: "Account not found"
  
  POST /accounts/{accountId}/balance:
    description: "Update balance (internal use)"
    request:
      body:
        amount: number
        operation: string # "debit" | "credit"
        reference: string
    response:
      200: "Balance updated"
      400: "Invalid operation"
  
  # Transaction History
  GET /accounts/{accountId}/transactions:
    description: "Get transaction history"
    parameters:
      accountId: string (path)
      fromDate?: string (query)
      toDate?: string (query)
      limit?: number (query)
      offset?: number (query)
    response:
      200: "Transaction list"
      404: "Account not found"
  
  GET /accounts/{accountId}/transactions/{transactionId}:
    description: "Get specific transaction"
    response:
      200: "Transaction details"
      404: "Transaction not found"
```

---

## 💳 **PAYMENT SERVICE API**

### **Base URL**: `/api/v1/payments`

```yaml
endpoints:
  # Payment Processing
  POST /payments:
    description: "Process payment"
    request:
      body:
        amount: number
        currency: string
        paymentMethod: object
        recipientAccount: string
        description?: string
        tenantId: string
    response:
      201: "Payment initiated"
      400: "Invalid payment data"
      402: "Payment failed"
  
  GET /payments/{paymentId}:
    description: "Get payment status"
    parameters:
      paymentId: string (path)
    response:
      200: "Payment details"
      404: "Payment not found"
  
  PUT /payments/{paymentId}/cancel:
    description: "Cancel payment"
    parameters:
      paymentId: string (path)
    response:
      200: "Payment cancelled"
      404: "Payment not found"
      400: "Cannot cancel payment"
  
  # PSP Integration
  POST /payments/{paymentId}/webhook:
    description: "PSP webhook endpoint"
    request:
      body:
        event: string
        data: object
        signature: string
    response:
      200: "Webhook processed"
      400: "Invalid webhook"
  
  # Payment Methods
  GET /payment-methods:
    description: "Get available payment methods"
    parameters:
      tenantId: string (query)
    response:
      200: "Available payment methods"
  
  POST /payment-methods:
    description: "Add payment method"
    request:
      body:
        type: string
        details: object
        tenantId: string
    response:
      201: "Payment method added"
      400: "Invalid payment method"
  
  # Multi-tenant PSP Resolution
  GET /psp-providers:
    description: "Get PSP providers by tenant"
    parameters:
      tenantId: string (query)
    response:
      200: "Available PSP providers"
  
  POST /psp-providers:
    description: "Configure PSP for tenant"
    request:
      body:
        tenantId: string
        pspType: string
        configuration: object
    response:
      201: "PSP configured"
      400: "Invalid configuration"
```

---

## ⚠️ **RISK SERVICE API**

### **Base URL**: `/api/v1/risk`

```yaml
endpoints:
  # Risk Assessment
  POST /risk/assess:
    description: "Assess transaction risk"
    request:
      body:
        transactionId: string
        userId: string
        amount: number
        paymentMethod: object
        context: object
    response:
      200: "Risk assessment result"
      400: "Invalid assessment data"
  
  GET /risk/assessments/{assessmentId}:
    description: "Get risk assessment"
    parameters:
      assessmentId: string (path)
    response:
      200: "Assessment details"
      404: "Assessment not found"
  
  # Risk Scoring
  POST /risk/score:
    description: "Calculate risk score"
    request:
      body:
        features: object
        model: string
    response:
      200: "Risk score"
      400: "Invalid features"
  
  # ML Model Management
  GET /risk/models:
    description: "Get available ML models"
    response:
      200: "List of models"
  
  POST /risk/models:
    description: "Deploy new ML model"
    request:
      body:
        modelName: string
        modelVersion: string
        endpointUrl: string
    response:
      201: "Model deployed"
      400: "Invalid model"
  
  # Risk Rules
  GET /risk/rules:
    description: "Get risk rules"
    response:
      200: "List of risk rules"
  
  POST /risk/rules:
    description: "Create risk rule"
    request:
      body:
        ruleName: string
        condition: object
        action: string
        priority: number
    response:
      201: "Rule created"
      400: "Invalid rule"
```

---

## 🛡️ **FRAUD SERVICE API**

### **Base URL**: `/api/v1/fraud`

```yaml
endpoints:
  # Fraud Detection
  POST /fraud/detect:
    description: "Detect fraud"
    request:
      body:
        transactionId: string
        userId: string
        deviceInfo: object
        behaviorData: object
        context: object
    response:
      200: "Fraud detection result"
      400: "Invalid detection data"
  
  GET /fraud/detections/{detectionId}:
    description: "Get fraud detection"
    parameters:
      detectionId: string (path)
    response:
      200: "Detection details"
      404: "Detection not found"
  
  # Behavioral Analysis
  POST /fraud/behavior/analyze:
    description: "Analyze user behavior"
    request:
      body:
        userId: string
        behaviorData: object
        timeWindow: string
    response:
      200: "Behavior analysis"
      400: "Invalid behavior data"
  
  # Device Fingerprinting
  POST /fraud/device/fingerprint:
    description: "Create device fingerprint"
    request:
      body:
        deviceInfo: object
        userId: string
    response:
      201: "Device fingerprint created"
      400: "Invalid device info"
  
  GET /fraud/device/{deviceId}:
    description: "Get device information"
    parameters:
      deviceId: string (path)
    response:
      200: "Device details"
      404: "Device not found"
  
  # Fraud Cases
  GET /fraud/cases:
    description: "Get fraud cases"
    parameters:
      status?: string (query)
      fromDate?: string (query)
      toDate?: string (query)
    response:
      200: "List of fraud cases"
  
  POST /fraud/cases:
    description: "Create fraud case"
    request:
      body:
        transactionId: string
        fraudType: string
        severity: string
        description: string
    response:
      201: "Fraud case created"
      400: "Invalid case data"
```

---

## 📧 **NOTIFICATION SERVICE API**

### **Base URL**: `/api/v1/notifications`

```yaml
endpoints:
  # Notification Management
  POST /notifications:
    description: "Send notification"
    request:
      body:
        userId: string
        type: string
        channel: string
        template: string
        data: object
    response:
      201: "Notification sent"
      400: "Invalid notification"
  
  GET /notifications/{notificationId}:
    description: "Get notification status"
    parameters:
      notificationId: string (path)
    response:
      200: "Notification details"
      404: "Notification not found"
  
  GET /notifications/user/{userId}:
    description: "Get user notifications"
    parameters:
      userId: string (path)
      status?: string (query)
      limit?: number (query)
    response:
      200: "List of notifications"
  
  # Template Management
  GET /templates:
    description: "Get notification templates"
    response:
      200: "List of templates"
  
  POST /templates:
    description: "Create notification template"
    request:
      body:
        name: string
        type: string
        channel: string
        content: string
        variables: array
    response:
      201: "Template created"
      400: "Invalid template"
  
  # Delivery Tracking
  GET /notifications/{notificationId}/delivery:
    description: "Get delivery status"
    parameters:
      notificationId: string (path)
    response:
      200: "Delivery details"
      404: "Notification not found"
  
  # Multi-channel Support
  POST /notifications/bulk:
    description: "Send bulk notifications"
    request:
      body:
        recipients: array
        type: string
        template: string
        data: object
    response:
      201: "Bulk notification sent"
      400: "Invalid bulk notification"
```

---

## 🔌 **OPEN FINANCE API**

### **Base URL**: `/api/v1/openfinance`

```yaml
endpoints:
  # Data Access
  GET /accounts:
    description: "Get user accounts (with consent)"
    headers:
      Authorization: "Bearer {oauth_token}"
    parameters:
      consentId: string (query)
    response:
      200: "Account data"
      403: "Consent required"
  
  GET /transactions:
    description: "Get transaction data (with consent)"
    headers:
      Authorization: "Bearer {oauth_token}"
    parameters:
      consentId: string (query)
      accountId?: string (query)
      fromDate?: string (query)
      toDate?: string (query)
    response:
      200: "Transaction data"
      403: "Consent required"
  
  # Consent Management
  POST /consent:
    description: "Create data consent"
    request:
      body:
        userId: string
        thirdPartyId: string
        scope: array
        duration: string
    response:
      201: "Consent created"
      400: "Invalid consent"
  
  GET /consent/{consentId}:
    description: "Get consent status"
    parameters:
      consentId: string (path)
    response:
      200: "Consent details"
      404: "Consent not found"
  
  PUT /consent/{consentId}/revoke:
    description: "Revoke consent"
    parameters:
      consentId: string (path)
    response:
      200: "Consent revoked"
      404: "Consent not found"
  
  # Third-party Registration
  POST /third-parties:
    description: "Register third-party application"
    request:
      body:
        name: string
        description: string
        redirectUri: string
        scope: array
    response:
      201: "Third-party registered"
      400: "Invalid registration"
```

---

## 📊 **HEALTH CHECK API**

### **Base URL**: `/health`

```yaml
endpoints:
  GET /health:
    description: "Service health check"
    response:
      200: "Service healthy"
      503: "Service unhealthy"
  
  GET /health/ready:
    description: "Readiness probe"
    response:
      200: "Service ready"
      503: "Service not ready"
  
  GET /health/live:
    description: "Liveness probe"
    response:
      200: "Service alive"
      503: "Service not alive"
  
  GET /health/dependencies:
    description: "Check external dependencies"
    response:
      200: "Dependencies healthy"
      503: "Dependencies unhealthy"
```

---

## 📈 **MÉTRICAS Y MONITOREO**

### **Métricas por API**

```yaml
metrics:
  request_count: "Número total de requests"
  request_duration: "Duración promedio de requests"
  error_rate: "Tasa de errores por endpoint"
  response_size: "Tamaño promedio de respuestas"
  concurrent_requests: "Requests concurrentes"
  cache_hit_rate: "Tasa de acierto de cache"
```

### **Alertas Configuradas**

```yaml
alerts:
  high_error_rate: "Error rate > 5%"
  high_latency: "Response time > 500ms"
  low_availability: "Availability < 99.9%"
  high_memory_usage: "Memory usage > 80%"
  disk_space_low: "Disk space < 20%"
```

---

**Estas especificaciones detalladas proporcionan la base para la implementación y testing de todas las APIs internas del sistema.**
