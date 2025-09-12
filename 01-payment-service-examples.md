# Ejemplos de Implementación - Payment Service

## Descripción

Este documento contiene **ejemplos de código real** para el Payment Service, basado en la experiencia con el cashier PSP actual.

## 1. Payment Controller

```java
// Payment Controller - Basado en tu experiencia con cashier PSP
@RestController
@RequestMapping("/api/v1/payments")
@Validated
@Slf4j
public class PaymentController {
    
    private final PaymentOrchestrator paymentOrchestrator;
    private final PSPAdapterResolver pspAdapterResolver;
    private final AuditLogger auditLogger;
    
    @PostMapping
    public ResponseEntity<PaymentResponse> processPayment(
            @Valid @RequestBody PaymentRequest request,
            @RequestHeader("X-Tenant-ID") String tenantId,
            @RequestHeader("X-Request-ID") String requestId) {
        
        try {
            // Validar tenant
            validateTenant(tenantId);
            
            // Log de auditoría
            auditLogger.logPaymentInitiation(requestId, tenantId, request);
            
            // Procesar pago
            PaymentResult result = paymentOrchestrator.processPayment(request, tenantId);
            
            // Log de resultado
            auditLogger.logPaymentResult(requestId, result);
            
            return ResponseEntity.ok(PaymentResponse.from(result));
            
        } catch (ValidationException e) {
            log.error("Validation error for request {}: {}", requestId, e.getMessage());
            return ResponseEntity.badRequest()
                .body(PaymentResponse.error("VALIDATION_ERROR", e.getMessage()));
                
        } catch (PaymentException e) {
            log.error("Payment error for request {}: {}", requestId, e.getMessage());
            return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY)
                .body(PaymentResponse.error("PAYMENT_ERROR", e.getMessage()));
                
        } catch (Exception e) {
            log.error("Unexpected error for request {}: {}", requestId, e.getMessage(), e);
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(PaymentResponse.error("INTERNAL_ERROR", "Internal server error"));
        }
    }
    
    @GetMapping("/{paymentId}")
    public ResponseEntity<PaymentStatus> getPaymentStatus(
            @PathVariable String paymentId,
            @RequestHeader("X-Tenant-ID") String tenantId) {
        
        try {
            PaymentStatus status = paymentOrchestrator.getPaymentStatus(paymentId, tenantId);
            return ResponseEntity.ok(status);
            
        } catch (PaymentNotFoundException e) {
            return ResponseEntity.notFound().build();
            
        } catch (Exception e) {
            log.error("Error getting payment status for {}: {}", paymentId, e.getMessage());
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build();
        }
    }
    
    @PostMapping("/{paymentId}/webhook")
    public ResponseEntity<String> handleWebhook(
            @PathVariable String paymentId,
            @RequestBody String payload,
            @RequestHeader("X-Signature") String signature,
            @RequestHeader("X-Tenant-ID") String tenantId) {
        
        try {
            // Verificar firma del webhook
            if (!webhookVerifier.verifySignature(payload, signature, tenantId)) {
                return ResponseEntity.status(HttpStatus.UNAUTHORIZED).body("Invalid signature");
            }
            
            // Procesar webhook
            webhookHandler.handleWebhook(paymentId, payload, tenantId);
            
            return ResponseEntity.ok("OK");
            
        } catch (Exception e) {
            log.error("Error processing webhook for {}: {}", paymentId, e.getMessage());
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body("Error");
        }
    }
    
    private void validateTenant(String tenantId) {
        if (StringUtils.isBlank(tenantId)) {
            throw new ValidationException("Tenant ID is required");
        }
        
        if (!tenantService.isValidTenant(tenantId)) {
            throw new ValidationException("Invalid tenant ID: " + tenantId);
        }
    }
}
```

## 2. Payment Orchestrator

```java
// Payment Orchestrator - Lógica de negocio
@Service
@Slf4j
public class PaymentOrchestrator {
    
    private final PSPAdapterResolver pspAdapterResolver;
    private final RiskService riskService;
    private final FraudService fraudService;
    private final TransactionRepository transactionRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final CircuitBreakerRegistry circuitBreakerRegistry;
    
    @Transactional
    public PaymentResult processPayment(PaymentRequest request, String tenantId) {
        String paymentId = generatePaymentId();
        
        try {
            // 1. Validar pago
            validatePayment(request);
            
            // 2. Evaluar riesgo
            RiskAssessment risk = evaluateRisk(request, tenantId);
            if (risk.getScore() > 0.8) {
                throw new HighRiskTransactionException("Transaction rejected due to high risk");
            }
            
            // 3. Detectar fraude
            FraudAssessment fraud = detectFraud(request, tenantId);
            if (fraud.isFraudulent()) {
                throw new FraudDetectedException("Fraudulent transaction detected");
            }
            
            // 4. Resolver PSP
            PSPAdapter pspAdapter = pspAdapterResolver.resolveProvider(
                request.getPaymentMethod().getProvider(), tenantId);
            
            // 5. Procesar pago
            PaymentResult result = processPaymentWithPSP(pspAdapter, request, paymentId);
            
            // 6. Guardar transacción
            saveTransaction(result, request, tenantId);
            
            // 7. Publicar evento
            publishPaymentEvent(result, tenantId);
            
            return result;
            
        } catch (Exception e) {
            log.error("Error processing payment {}: {}", paymentId, e.getMessage(), e);
            
            // Guardar transacción fallida
            saveFailedTransaction(paymentId, request, tenantId, e);
            
            throw new PaymentException("Payment processing failed", e);
        }
    }
    
    private RiskAssessment evaluateRisk(PaymentRequest request, String tenantId) {
        CircuitBreaker riskCircuitBreaker = circuitBreakerRegistry.circuitBreaker("risk-service");
        
        return riskCircuitBreaker.executeSupplier(() -> {
            try {
                return riskService.evaluateRisk(request, tenantId);
            } catch (Exception e) {
                log.warn("Risk service unavailable, using default risk score");
                return RiskAssessment.builder()
                    .score(0.5) // Score neutral por defecto
                    .action(RiskAction.APPROVE)
                    .build();
            }
        });
    }
    
    private FraudAssessment detectFraud(PaymentRequest request, String tenantId) {
        CircuitBreaker fraudCircuitBreaker = circuitBreakerRegistry.circuitBreaker("fraud-service");
        
        return fraudCircuitBreaker.executeSupplier(() -> {
            try {
                return fraudService.detectFraud(request, tenantId);
            } catch (Exception e) {
                log.warn("Fraud service unavailable, using basic validation");
                return FraudAssessment.builder()
                    .fraudScore(0.1) // Score bajo por defecto
                    .isFraudulent(false)
                    .action(FraudAction.APPROVE)
                    .build();
            }
        });
    }
    
    private PaymentResult processPaymentWithPSP(PSPAdapter pspAdapter, 
                                               PaymentRequest request, 
                                               String paymentId) {
        try {
            return pspAdapter.processPayment(request, paymentId);
        } catch (PSPException e) {
            log.error("PSP processing failed: {}", e.getMessage());
            throw new PaymentException("PSP processing failed", e);
        }
    }
    
    private void saveTransaction(PaymentResult result, PaymentRequest request, String tenantId) {
        Transaction transaction = Transaction.builder()
            .paymentId(result.getPaymentId())
            .tenantId(tenantId)
            .amount(request.getAmount())
            .currency(request.getCurrency())
            .status(result.getStatus())
            .provider(request.getPaymentMethod().getProvider())
            .createdAt(Instant.now())
            .build();
            
        transactionRepository.save(transaction);
    }
    
    private void publishPaymentEvent(PaymentResult result, String tenantId) {
        PaymentEvent event = PaymentEvent.builder()
            .paymentId(result.getPaymentId())
            .tenantId(tenantId)
            .status(result.getStatus())
            .amount(result.getAmount())
            .timestamp(Instant.now())
            .build();
            
        kafkaTemplate.send("payment-events", event);
    }
}
```

## 3. PSP Adapter Resolver

```java
// PSP Adapter Resolver - Basado en tu código actual
@Component
@Slf4j
public class PSPAdapterResolver {
    
    private final Map<PSPType, Map<String, PSPAdapter>> providerMap;
    private final TenantService tenantService;
    
    public PSPAdapterResolver(List<PSPAdapter> adapters, TenantService tenantService) {
        this.tenantService = tenantService;
        this.providerMap = initializeProviderMap(adapters);
    }
    
    public PSPAdapter resolveProvider(PSPType providerType, String tenantId) {
        log.debug("Resolving PSP adapter for provider: {} and tenant: {}", providerType, tenantId);
        
        Map<String, PSPAdapter> tenantProviders = providerMap.get(providerType);
        if (tenantProviders == null) {
            throw new PSPNotFoundException("Provider not found: " + providerType);
        }
        
        // Obtener configuración del tenant
        TenantConfig tenantConfig = tenantService.getTenantConfig(tenantId);
        String providerKey = tenantConfig.getProviderKey(providerType);
        
        PSPAdapter adapter = tenantProviders.get(providerKey);
        if (adapter == null) {
            throw new PSPNotFoundException("Provider not configured for tenant: " + tenantId);
        }
        
        log.debug("Resolved PSP adapter: {} for tenant: {}", adapter.getClass().getSimpleName(), tenantId);
        return adapter;
    }
    
    private Map<PSPType, Map<String, PSPAdapter>> initializeProviderMap(List<PSPAdapter> adapters) {
        Map<PSPType, Map<String, PSPAdapter>> map = new HashMap<>();
        
        for (PSPAdapter adapter : adapters) {
            PSPType providerType = adapter.getProviderType();
            String providerKey = adapter.getProviderKey();
            
            map.computeIfAbsent(providerType, k -> new HashMap<>())
               .put(providerKey, adapter);
        }
        
        return map;
    }
}
```

## 4. PSP Adapters

### 4.1 BTG Adapter

```java
// BTG Adapter - Brazilian Bank
@Component
@Slf4j
public class BTGAdapter implements PSPAdapter {
    
    private final BTGRestClient btgRestClient;
    private final BTGCache btgCache;
    private final ObjectMapper objectMapper;
    
    @Override
    public PaymentResult processPayment(PaymentRequest request, String paymentId) {
        try {
            // 1. Obtener token de acceso
            String accessToken = getAccessToken();
            
            // 2. Crear comando de pago
            BTGPaymentCommand command = createPaymentCommand(request, paymentId);
            
            // 3. Enviar pago a BTG
            BTGPaymentResponse response = btgRestClient.processPayment(command, accessToken);
            
            // 4. Procesar respuesta
            PaymentResult result = processBTGResponse(response);
            
            // 5. Cachear resultado
            btgCache.cachePaymentResult(paymentId, result);
            
            return result;
            
        } catch (BTGException e) {
            log.error("BTG payment failed: {}", e.getMessage());
            throw new PSPException("BTG payment processing failed", e);
        }
    }
    
    private String getAccessToken() {
        // Verificar cache primero
        String cachedToken = btgCache.getAccessToken();
        if (cachedToken != null && !isTokenExpired(cachedToken)) {
            return cachedToken;
        }
        
        // Obtener nuevo token
        BTGTokenResponse tokenResponse = btgRestClient.getAccessToken();
        btgCache.cacheAccessToken(tokenResponse.getAccessToken(), 
                                 tokenResponse.getExpiresIn());
        
        return tokenResponse.getAccessToken();
    }
    
    private BTGPaymentCommand createPaymentCommand(PaymentRequest request, String paymentId) {
        return BTGPaymentCommand.builder()
            .paymentId(paymentId)
            .amount(request.getAmount())
            .currency(request.getCurrency())
            .payer(createBTGPayer(request.getPayer()))
            .paymentMethod(createBTGPaymentMethod(request.getPaymentMethod()))
            .description(request.getDescription())
            .build();
    }
    
    private PaymentResult processBTGResponse(BTGPaymentResponse response) {
        return PaymentResult.builder()
            .paymentId(response.getPaymentId())
            .status(mapBTGStatus(response.getStatus()))
            .amount(response.getAmount())
            .currency(response.getCurrency())
            .transactionId(response.getTransactionId())
            .processedAt(Instant.now())
            .build();
    }
    
    @Override
    public PSPType getProviderType() {
        return PSPType.BTG;
    }
    
    @Override
    public String getProviderKey() {
        return "btg-default";
    }
}
```

### 4.2 TotalCoin Adapter

```java
// TotalCoin Adapter - Crypto Payments
@Component
@Slf4j
public class TotalCoinAdapter implements PSPAdapter {
    
    private final TotalCoinRestClient totalCoinRestClient;
    private final TotalCoinCache totalCoinCache;
    
    @Override
    public PaymentResult processPayment(PaymentRequest request, String paymentId) {
        try {
            // 1. Obtener token de TotalCoin
            String token = getTotalCoinToken();
            
            // 2. Crear request de pago
            TotalCoinPaymentRequest paymentRequest = createTotalCoinRequest(request, paymentId);
            
            // 3. Procesar pago
            TotalCoinPaymentResponse response = totalCoinRestClient.processPayment(paymentRequest, token);
            
            // 4. Mapear respuesta
            PaymentResult result = mapTotalCoinResponse(response);
            
            return result;
            
        } catch (TotalCoinException e) {
            log.error("TotalCoin payment failed: {}", e.getMessage());
            throw new PSPException("TotalCoin payment processing failed", e);
        }
    }
    
    private String getTotalCoinToken() {
        String cachedToken = totalCoinCache.getToken();
        if (cachedToken != null && !isTokenExpired(cachedToken)) {
            return cachedToken;
        }
        
        TotalCoinTokenResponse tokenResponse = totalCoinRestClient.getToken();
        totalCoinCache.cacheToken(tokenResponse.getToken(), tokenResponse.getExpiresIn());
        
        return tokenResponse.getToken();
    }
    
    private TotalCoinPaymentRequest createTotalCoinRequest(PaymentRequest request, String paymentId) {
        return TotalCoinPaymentRequest.builder()
            .paymentId(paymentId)
            .amount(request.getAmount())
            .currency(request.getCurrency())
            .paymentMethod(request.getPaymentMethod().getType())
            .description(request.getDescription())
            .build();
    }
    
    @Override
    public PSPType getProviderType() {
        return PSPType.TOTALCOIN;
    }
    
    @Override
    public String getProviderKey() {
        return "totalcoin-default";
    }
}
```

## 5. Webhook Handler

```java
// Webhook Handler - PSP Notifications
@Component
@Slf4j
public class WebhookHandler {
    
    private final WebhookVerifier webhookVerifier;
    private final TransactionRepository transactionRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final AuditLogger auditLogger;
    
    public void handleWebhook(String paymentId, String payload, String tenantId) {
        try {
            // 1. Parsear payload
            WebhookNotification notification = parseWebhookPayload(payload);
            
            // 2. Validar notificación
            validateWebhookNotification(notification, paymentId);
            
            // 3. Actualizar transacción
            updateTransaction(paymentId, notification, tenantId);
            
            // 4. Publicar evento
            publishWebhookEvent(paymentId, notification, tenantId);
            
            // 5. Log de auditoría
            auditLogger.logWebhookReceived(paymentId, notification, tenantId);
            
        } catch (Exception e) {
            log.error("Error processing webhook for payment {}: {}", paymentId, e.getMessage(), e);
            throw new WebhookProcessingException("Webhook processing failed", e);
        }
    }
    
    private WebhookNotification parseWebhookPayload(String payload) {
        try {
            return objectMapper.readValue(payload, WebhookNotification.class);
        } catch (JsonProcessingException e) {
            throw new WebhookParsingException("Failed to parse webhook payload", e);
        }
    }
    
    private void updateTransaction(String paymentId, WebhookNotification notification, String tenantId) {
        Transaction transaction = transactionRepository.findByPaymentIdAndTenantId(paymentId, tenantId)
            .orElseThrow(() -> new TransactionNotFoundException("Transaction not found: " + paymentId));
        
        transaction.setStatus(mapWebhookStatus(notification.getStatus()));
        transaction.setUpdatedAt(Instant.now());
        transaction.setWebhookData(notification.getData());
        
        transactionRepository.save(transaction);
    }
    
    private void publishWebhookEvent(String paymentId, WebhookNotification notification, String tenantId) {
        WebhookEvent event = WebhookEvent.builder()
            .paymentId(paymentId)
            .tenantId(tenantId)
            .status(notification.getStatus())
            .timestamp(Instant.now())
            .build();
            
        kafkaTemplate.send("webhook-events", event);
    }
}
```

## 6. DTOs y Models

### 6.1 Payment Request

```java
// Payment Request DTO
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class PaymentRequest {
    
    @NotNull
    @Positive
    private BigDecimal amount;
    
    @NotNull
    @Size(min = 3, max = 3)
    private String currency;
    
    @NotNull
    @Valid
    private PaymentMethod paymentMethod;
    
    @NotNull
    @Valid
    private Payer payer;
    
    @Size(max = 500)
    private String description;
    
    @Valid
    private Map<String, Object> metadata;
    
    @Valid
    private List<PaymentItem> items;
}
```

### 6.2 Payment Result

```java
// Payment Result DTO
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class PaymentResult {
    
    @NotNull
    private String paymentId;
    
    @NotNull
    private PaymentStatus status;
    
    @NotNull
    private BigDecimal amount;
    
    @NotNull
    private String currency;
    
    private String transactionId;
    
    private String providerTransactionId;
    
    private Instant processedAt;
    
    private String failureReason;
    
    private Map<String, Object> providerData;
}
```

### 6.3 Transaction Entity

```java
// Transaction Entity
@Entity
@Table(name = "transactions")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Transaction {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "payment_id", nullable = false, unique = true)
    private String paymentId;
    
    @Column(name = "tenant_id", nullable = false)
    private String tenantId;
    
    @Column(name = "amount", nullable = false, precision = 19, scale = 2)
    private BigDecimal amount;
    
    @Column(name = "currency", nullable = false, length = 3)
    private String currency;
    
    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private PaymentStatus status;
    
    @Enumerated(EnumType.STRING)
    @Column(name = "provider", nullable = false)
    private PSPType provider;
    
    @Column(name = "transaction_id")
    private String transactionId;
    
    @Column(name = "created_at", nullable = false)
    private Instant createdAt;
    
    @Column(name = "updated_at")
    private Instant updatedAt;
    
    @Convert(converter = JsonbConverter.class)
    @Column(name = "webhook_data", columnDefinition = "jsonb")
    private Map<String, Object> webhookData;
}
```

## 7. Configuration

### 7.1 Application Properties

```yaml
# application.yml
payment:
  service:
    timeout: 30s
    retry:
      max-attempts: 3
      delay: 1s
      multiplier: 2
    
  psp:
    btg:
      base-url: ${BTG_BASE_URL:https://api.btg.com}
      client-id: ${BTG_CLIENT_ID}
      client-secret: ${BTG_CLIENT_SECRET}
      timeout: 10s
      
    totalcoin:
      base-url: ${TOTALCOIN_BASE_URL:https://api.totalcoin.com}
      api-key: ${TOTALCOIN_API_KEY}
      timeout: 10s

kafka:
  bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
  topics:
    payment-events: payment-events
    webhook-events: webhook-events
  producer:
    acks: all
    retries: 3
    batch-size: 16384
    linger-ms: 5
```

### 7.2 Circuit Breaker Configuration

```java
// Circuit Breaker Configuration
@Configuration
public class CircuitBreakerConfig {
    
    @Bean
    public CircuitBreakerConfigCustomizer circuitBreakerConfigCustomizer() {
        return CircuitBreakerConfigCustomizer.of("risk-service", builder -> builder
            .slidingWindowSize(10)
            .minimumNumberOfCalls(5)
            .failureRateThreshold(50)
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .permittedNumberOfCallsInHalfOpenState(3)
            .slowCallRateThreshold(50)
            .slowCallDurationThreshold(Duration.ofSeconds(2))
        );
    }
}
```

## 8. Testing

### 8.1 Unit Tests

```java
// Payment Controller Test
@WebMvcTest(PaymentController.class)
class PaymentControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private PaymentOrchestrator paymentOrchestrator;
    
    @Test
    void shouldProcessPaymentSuccessfully() throws Exception {
        // Given
        PaymentRequest request = createValidPaymentRequest();
        PaymentResult result = createPaymentResult();
        
        when(paymentOrchestrator.processPayment(any(), any()))
            .thenReturn(result);
        
        // When & Then
        mockMvc.perform(post("/api/v1/payments")
                .header("X-Tenant-ID", "tenant-123")
                .header("X-Request-ID", "req-123")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.paymentId").value(result.getPaymentId()))
                .andExpect(jsonPath("$.status").value(result.getStatus().toString()));
    }
}
```

### 8.2 Integration Tests

```java
// Payment Service Integration Test
@SpringBootTest
@Testcontainers
class PaymentServiceIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:13")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");
    
    @Container
    static GenericContainer<?> kafka = new GenericContainer<>("confluentinc/cp-kafka:latest")
            .withExposedPorts(9092)
            .withEnv("KAFKA_ZOOKEEPER_CONNECT", "zookeeper:2181")
            .withEnv("KAFKA_ADVERTISED_LISTENERS", "PLAINTEXT://localhost:9092");
    
    @Test
    void shouldProcessPaymentEndToEnd() {
        // Given
        PaymentRequest request = createValidPaymentRequest();
        
        // When
        PaymentResult result = paymentService.processPayment(request, "tenant-123");
        
        // Then
        assertThat(result).isNotNull();
        assertThat(result.getPaymentId()).isNotBlank();
        assertThat(result.getStatus()).isIn(PaymentStatus.PROCESSING, PaymentStatus.COMPLETED);
    }
}
```

---

**Documento**: Ejemplos de Implementación - Payment Service  
**Versión**: 1.0  
**Fecha**: Diciembre 2024
