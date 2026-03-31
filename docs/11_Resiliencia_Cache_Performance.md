# Resiliência, Cache e Performance
**App View v2.0 - Estratégias de Retry, Cache e Dead Letter Queue**

**Versão:** 2.0.0  
**Data:** Janeiro de 2026  

---

## Visão Geral

Este documento detalha as estratégias de **resiliência**, **cache** e **performance** do App View v2.0, cobrindo:

1. **Retry + Exponential Backoff** para todas integrações externas
2. **Cache Strategy** para otimização de leitura
3. **Cache Invalidation** patterns
4. **Dead Letter Queue (DLQ)** para tratamento de falhas
5. **Circuit Breaker** policies detalhadas

---

## 1. Retry + Exponential Backoff

### Princípio Fundamental

**TODAS as chamadas para sistemas externos DEVEM ter retry com exponential backoff.**

### Por que Exponential Backoff?

```
Tentativa 1: falha → aguarda 1s
Tentativa 2: falha → aguarda 2s (2^1)
Tentativa 3: falha → aguarda 4s (2^2)
Tentativa 4: falha → aguarda 8s (2^3)
Tentativa 5: falha → aguarda 16s (2^4)
```

**Benefícios:**
- ✅ Evita sobrecarregar sistema já instável
- ✅ Dá tempo para o sistema externo se recuperar
- ✅ Reduz contenção de recursos
- ✅ Melhora taxa de sucesso geral

---

## 2. Estratégia de Retry por Sistema

### 2.1 Alfresco ECM (Crítico)

```yaml
resilience4j:
  retry:
    instances:
      alfresco:
        maxAttempts: 3
        waitDuration: 1s
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        retryExceptions:
          - org.apache.chemistry.opencmis.commons.exceptions.CmisConnectionException
          - org.apache.chemistry.opencmis.commons.exceptions.CmisRuntimeException
        ignoreExceptions:
          - org.apache.chemistry.opencmis.commons.exceptions.CmisPermissionDeniedException
          - org.apache.chemistry.opencmis.commons.exceptions.CmisConstraintException
```

**Lógica:**
- ✅ **Retry:** Erros de conexão, timeouts
- ❌ **Não retry:** Erros de permissão, validação (não vão resolver)
- **Fallback:** Enfileirar em `alfresco-retry-queue` (DLQ)

**Implementação:**

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class AlfrescoGateway {
    
    private final Session cmisSession;
    private final AzureServiceBusProducer serviceBusProducer;
    
    @CircuitBreaker(name = "alfresco", fallbackMethod = "uploadFallback")
    @Retry(name = "alfresco")
    public String uploadDocumento(DocumentoUpload upload) {
        log.info("Tentativa de upload para Alfresco: arquivo={}, tentativa={}", 
            upload.getNomeArquivo(),
            getCurrentRetryCount());
        
        try {
            // Upload logic
            Document doc = pasta.createDocument(...);
            
            log.info("Upload concluído com sucesso: documentId={}, tentativas={}", 
                doc.getId(),
                getCurrentRetryCount() + 1);
            
            return doc.getId();
            
        } catch (CmisConnectionException e) {
            log.warn("Erro de conexão com Alfresco (retry será executado): tentativa={}, error={}", 
                getCurrentRetryCount() + 1,
                e.getMessage());
            throw e; // Resilience4j vai fazer retry
        }
    }
    
    // Fallback: Enfileirar para processamento assíncrono
    private String uploadFallback(DocumentoUpload upload, Throwable t) {
        log.error("Todas as tentativas de upload falharam: arquivo={}, error={}", 
            upload.getNomeArquivo(), 
            t.getMessage());
        
        // Enviar para DLQ para retry posterior
        serviceBusProducer.sendToQueue("alfresco-retry-queue", upload);
        
        log.info("Documento enfileirado em DLQ para retry posterior: arquivo={}", 
            upload.getNomeArquivo());
        
        return null; // Upload será processado posteriormente
    }
    
    private int getCurrentRetryCount() {
        return RetryRegistry.ofDefaults()
            .retry("alfresco")
            .getMetrics()
            .getNumberOfFailedCallsWithRetryAttempt();
    }
}
```

---

### 2.2 Jarvis OCR (Alta Criticidade)

```yaml
resilience4j:
  retry:
    instances:
      jarvis:
        maxAttempts: 3
        waitDuration: 2s
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        retryExceptions:
          - java.net.ConnectException
          - java.net.SocketTimeoutException
          - org.springframework.web.client.ResourceAccessException
        ignoreExceptions:
          - org.springframework.web.client.HttpClientErrorException.BadRequest
          - org.springframework.web.client.HttpClientErrorException.Unauthorized
```

**Lógica especial para Jarvis:**
- ✅ **Retry:** Conexão, timeout, 500 Internal Server Error
- ⚠️ **Retry com delay maior:** 429 Too Many Requests (60s wait)
- ❌ **Não retry:** 400 Bad Request, 401 Unauthorized
- **Fallback:** Enfileirar em `jarvis-ocr-queue` com delay

**Implementação:**

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class JarvisGateway {
    
    private final RestTemplate restTemplate;
    private final AzureServiceBusProducer serviceBusProducer;
    
    @CircuitBreaker(name = "jarvis", fallbackMethod = "enviarParaOCRFallback")
    @Retry(name = "jarvis")
    @RateLimiter(name = "jarvis") // 50 req/min
    public void enviarParaOCR(SinistroId sinistroId, String documentCode) {
        log.info("Enviando documento para Jarvis OCR: sinistroId={}, documentCode={}", 
            sinistroId, documentCode);
        
        try {
            JarvisRequest request = new JarvisRequest(documentCode, ...);
            
            ResponseEntity<JarvisResponse> response = restTemplate.postForEntity(
                jarvisUrl + "/ocr/analyze",
                request,
                JarvisResponse.class
            );
            
            if (response.getStatusCode() == HttpStatus.TOO_MANY_REQUESTS) {
                // RN-014: Rate limit excedido
                log.warn("Jarvis rate limit excedido, aguardando 60s antes de retry");
                Thread.sleep(60000); // Aguardar 60s
                throw new JarvisRateLimitException("Rate limit excedido");
            }
            
            log.info("Documento enviado para Jarvis com sucesso: sinistroId={}, requestId={}", 
                sinistroId,
                response.getBody().getRequestId());
            
        } catch (HttpClientErrorException.TooManyRequests e) {
            log.warn("Jarvis retornou 429 Too Many Requests: sinistroId={}", sinistroId);
            throw new JarvisRateLimitException("Rate limit", e);
        }
    }
    
    private void enviarParaOCRFallback(SinistroId sinistroId, String documentCode, Throwable t) {
        log.error("Falha ao enviar para Jarvis após retries: sinistroId={}, error={}", 
            sinistroId, 
            t.getMessage());
        
        // Enfileirar com delay de 5 minutos
        serviceBusProducer.sendToQueueWithDelay(
            "jarvis-ocr-queue",
            new JarvisOCRMessage(sinistroId, documentCode),
            Duration.ofMinutes(5)
        );
        
        log.info("Documento enfileirado em DLQ com delay de 5min: sinistroId={}", sinistroId);
    }
}
```

---

### 2.3 Pega BPM (Alta Criticidade)

```yaml
resilience4j:
  retry:
    instances:
      pega:
        maxAttempts: 3
        waitDuration: 1s
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        retryExceptions:
          - java.net.ConnectException
          - java.net.SocketTimeoutException
        ignoreExceptions:
          - org.springframework.web.client.HttpClientErrorException.Conflict # 409 - caso já existe
```

**Lógica:**
- ✅ **Retry:** Conexão, timeout
- ✅ **Idempotência:** 409 Conflict não é erro (caso já criado)
- ❌ **Não retry:** 400 Bad Request
- **Fallback:** Enfileirar em `pega-routing-queue`

---

### 2.4 CCM, Prestador, BOD (Média Criticidade)

```yaml
resilience4j:
  retry:
    instances:
      ccm:
        maxAttempts: 2  # Menos tentativas (não crítico)
        waitDuration: 2s
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
```

**Lógica:**
- Menos agressivo (2 tentativas vs 3)
- Fallback: Notificação manual ou re-envio via dashboard

---

## 3. Cache Strategy

### 3.1 Quando Cachear?

✅ **CACHEAR:**
- Consultas de sinistros (leitura frequente)
- Consultas de clientes (dados mudam pouco)
- Listas de configuração (raramente mudam)
- Propostas do Portal PJ (consulta intensiva)
- Dashboards e métricas (agregações custosas)

❌ **NÃO CACHEAR:**
- Operações de escrita (POST, PUT, DELETE)
- Dados em tempo real (status de OCR em andamento)
- Dados sensíveis (senhas, tokens)

### 3.2 Cache de Sinistros (Caso Principal)

**Problema:** Consultas de sinistro são frequentes e pesadas (joins, metadados Alfresco)

**Solução:**

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ConsultarSinistroUseCase {
    
    private final SinistroRepository repository;
    private final RedisTemplate<String, SinistroDTO> redisTemplate;
    
    private static final String CACHE_KEY_PREFIX = "sinistro:";
    private static final Duration CACHE_TTL = Duration.ofMinutes(5);
    
    @Transactional(readOnly = true)
    public SinistroDTO executar(SinistroId sinistroId) {
        String cacheKey = CACHE_KEY_PREFIX + sinistroId.toString();
        
        // 1. Tentar buscar no cache
        log.debug("Buscando sinistro no cache: key={}", cacheKey);
        SinistroDTO cached = redisTemplate.opsForValue().get(cacheKey);
        
        if (cached != null) {
            log.info("Cache hit: sinistroId={}", sinistroId);
            return cached;
        }
        
        // 2. Cache miss: buscar no banco
        log.info("Cache miss: sinistroId={}, buscando no banco", sinistroId);
        Sinistro sinistro = repository.findById(sinistroId)
            .orElseThrow(() -> new SinistroNotFoundException(sinistroId));
        
        SinistroDTO dto = SinistroMapper.toDTO(sinistro);
        
        // 3. Armazenar no cache
        log.debug("Armazenando no cache: key={}, ttl={}s", cacheKey, CACHE_TTL.getSeconds());
        redisTemplate.opsForValue().set(cacheKey, dto, CACHE_TTL);
        
        return dto;
    }
}
```

### 3.3 Cache de Listas (Portal PJ)

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ConsultarPropostasUseCase {
    
    private final PropostaRepository repository;
    
    @Cacheable(
        value = "propostas",
        key = "#filtro.hashCode()",
        unless = "#result.isEmpty()",
        condition = "#filtro.isPageSizeValid()"
    )
    public Page<PropostaDTO> executar(PropostaFiltro filtro, Pageable pageable) {
        log.info("Consultando propostas (cache miss ou inválido): filtro={}", filtro);
        
        Specification<Proposta> spec = PropostaSpecifications.comFiltros(filtro);
        
        return repository.findAll(spec, pageable)
            .map(PropostaMapper::toDTO);
    }
}
```

**Configuração:**

```yaml
spring:
  cache:
    type: redis
    redis:
      time-to-live: 300000  # 5 minutos
      cache-null-values: false
    cache-names:
      - sinistros
      - clientes
      - propostas
      - contratos
      - dashboards
```

---

## 4. Cache Invalidation (Crítico!)

### 4.1 Quando Invalidar?

**Princípio:** Cache deve ser invalidado SEMPRE que os dados mudarem.

### 4.2 Strategies

#### Strategy 1: Invalidação Automática (TTL)

```yaml
# Deixar expirar naturalmente após TTL
spring:
  cache:
    redis:
      time-to-live: 300000  # 5 minutos
```

✅ **Pros:** Simples, zero código  
❌ **Cons:** Dados podem ficar stale por até 5 minutos

**Quando usar:** Dados que mudam raramente (configurações, listas estáticas)

---

#### Strategy 2: Invalidação Manual (Write-Through)

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class AtualizarSinistroUseCase {
    
    private final SinistroRepository repository;
    private final RedisTemplate<String, SinistroDTO> redisTemplate;
    
    @Transactional
    @CacheEvict(value = "sinistros", key = "#command.sinistroId")
    public void executar(AtualizarSinistroCommand command) {
        log.info("Atualizando sinistro: id={}", command.sinistroId());
        
        Sinistro sinistro = repository.findById(command.sinistroId())
            .orElseThrow();
        
        // Atualizar entidade
        sinistro.atualizar(command.toData());
        repository.save(sinistro);
        
        // Invalidar cache manualmente (redundante com @CacheEvict, mas explícito)
        String cacheKey = "sinistro:" + command.sinistroId();
        redisTemplate.delete(cacheKey);
        
        log.info("Cache invalidado: key={}", cacheKey);
    }
}
```

✅ **Pros:** Dados sempre consistentes  
✅ **Cons:** Nenhum (melhor approach)

**Quando usar:** Operações de escrita (SEMPRE)

---

#### Strategy 3: Invalidação por Evento (Event-Driven)

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class SinistroEventHandler {
    
    private final RedisTemplate<String, SinistroDTO> redisTemplate;
    
    @EventListener
    @Async
    public void onSinistroArmazenado(SinistroArmazenadoEvent event) {
        // Invalidar cache quando sinistro for armazenado no Alfresco
        String cacheKey = "sinistro:" + event.getSinistroId();
        
        log.info("Invalidando cache após SinistroArmazenado: key={}", cacheKey);
        redisTemplate.delete(cacheKey);
    }
    
    @EventListener
    @Async
    public void onOCRConcluido(OCRConcluidoEvent event) {
        // Invalidar cache quando OCR for concluído
        String cacheKey = "sinistro:" + event.getSinistroId();
        
        log.info("Invalidando cache após OCRConcluido: key={}", cacheKey);
        redisTemplate.delete(cacheKey);
    }
}
```

✅ **Pros:** Desacoplado, assíncrono  
✅ **Cons:** Nenhum (excelente para event-driven)

**Quando usar:** Quando mudanças vêm de eventos de domínio

---

#### Strategy 4: Invalidação por Pattern (Bulk)

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class CacheInvalidationService {
    
    private final RedisTemplate<String, Object> redisTemplate;
    
    /**
     * Invalida todos os caches de sinistros
     */
    public void invalidateAllSinistros() {
        log.info("Invalidando TODOS os caches de sinistros");
        
        Set<String> keys = redisTemplate.keys("sinistro:*");
        
        if (keys != null && !keys.isEmpty()) {
            redisTemplate.delete(keys);
            log.info("Invalidados {} caches de sinistros", keys.size());
        }
    }
    
    /**
     * Invalida caches de um cliente específico
     */
    public void invalidateCachesByCliente(String clienteId) {
        log.info("Invalidando caches relacionados ao cliente: id={}", clienteId);
        
        Set<String> keys = redisTemplate.keys("*:cliente:" + clienteId + ":*");
        
        if (keys != null && !keys.isEmpty()) {
            redisTemplate.delete(keys);
            log.info("Invalidados {} caches relacionados ao cliente", keys.size());
        }
    }
}
```

✅ **Pros:** Útil para invalidações em massa  
⚠️ **Cons:** `KEYS` é lento (usar com cuidado)

**Quando usar:** Manutenção, migrações, operações admin

---

### 4.3 Cache Invalidation Matrix

| Operação | Invalidação | Estratégia |
|----------|-------------|------------|
| Criar Sinistro | Não | Cache só para leitura |
| Atualizar Sinistro | SIM | @CacheEvict + manual |
| Armazenar no Alfresco | SIM | Event-driven |
| OCR Concluído | SIM | Event-driven |
| Rotear para Pega | SIM | Event-driven |
| Criar Cliente | Não | Cache só para leitura |
| Atualizar Cliente | SIM | @CacheEvict |
| Consulta (GET) | Não | Apenas leitura |

---

## 5. Dead Letter Queue (DLQ)

### 5.1 Arquitetura DLQ

```
[Operação Falha após todos os retries]
            ↓
    [Azure Service Bus]
            ↓
┌───────────────────────────┐
│  Main Queue               │
│  - jarvis-ocr-queue       │
│  - pega-routing-queue     │
│  - alfresco-retry-queue   │
└───────────────────────────┘
            ↓ (falha após 5 tentativas)
┌───────────────────────────┐
│  Dead Letter Queue (DLQ)  │
│  - Mensagens "envenenadas"│
│  - Max TTL: 14 dias       │
└───────────────────────────┘
            ↓
    [DLQ Processor Job]
    (analisa e reprocessa)
```

### 5.2 Configuração Service Bus

```yaml
# Azure Service Bus Configuration
azure:
  servicebus:
    connection-string: ${AZURE_SERVICEBUS_CONNECTION_STRING}
    queues:
      jarvis-ocr:
        name: jarvis-ocr-queue
        max-delivery-count: 5  # Após 5 falhas → DLQ
        lock-duration: PT5M     # Lock por 5 minutos
        default-message-ttl: P1D  # TTL 1 dia
        dead-letter-on-message-expiration: true
        
      pega-routing:
        name: pega-routing-queue
        max-delivery-count: 5
        lock-duration: PT5M
        default-message-ttl: P1D
        
      alfresco-retry:
        name: alfresco-retry-queue
        max-delivery-count: 5
        lock-duration: PT10M  # Lock maior (upload pode ser lento)
        default-message-ttl: P1D
```

### 5.3 DLQ Processor (Reprocessamento)

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class DeadLetterQueueProcessor {
    
    private final ServiceBusReceiverClient dlqReceiver;
    private final JarvisGateway jarvisGateway;
    private final AlfrescoGateway alfrescoGateway;
    
    /**
     * Job executado a cada 1 hora para processar mensagens da DLQ
     */
    @Scheduled(cron = "0 0 * * * *")  // A cada hora
    public void processDeadLetterQueue() {
        log.info("Iniciando processamento de Dead Letter Queue");
        
        try {
            // Buscar mensagens da DLQ (max 10 por vez)
            dlqReceiver.receiveMessages(10, Duration.ofSeconds(30))
                .forEach(this::processDeadLetterMessage);
                
        } catch (Exception e) {
            log.error("Erro ao processar DLQ", e);
        }
    }
    
    private void processDeadLetterMessage(ServiceBusReceivedMessage message) {
        String messageId = message.getMessageId();
        String queueName = message.getDeadLetterSource();
        
        log.info("Processando mensagem da DLQ: messageId={}, originalQueue={}, deliveryCount={}", 
            messageId,
            queueName,
            message.getDeliveryCount());
        
        try {
            // Determinar tipo de mensagem e reprocessar
            if (queueName.contains("jarvis-ocr")) {
                reprocessJarvisOCR(message);
            } else if (queueName.contains("alfresco-retry")) {
                reprocessAlfrescoUpload(message);
            } else if (queueName.contains("pega-routing")) {
                reprocessPegaRouting(message);
            }
            
            // Completar mensagem (remover da DLQ)
            dlqReceiver.complete(message);
            log.info("Mensagem reprocessada e removida da DLQ: messageId={}", messageId);
            
        } catch (Exception e) {
            log.error("Falha ao reprocessar mensagem da DLQ: messageId={}, error={}", 
                messageId, 
                e.getMessage());
            
            // Se falhar novamente, abandonar (volta para DLQ)
            dlqReceiver.abandon(message);
            
            // Alerta para time de operações
            sendAlertToOps(messageId, queueName, e);
        }
    }
    
    private void reprocessJarvisOCR(ServiceBusReceivedMessage message) {
        JarvisOCRMessage payload = deserialize(message.getBody(), JarvisOCRMessage.class);
        
        log.info("Reprocessando envio para Jarvis: sinistroId={}", payload.getSinistroId());
        
        // Tentar enviar novamente
        jarvisGateway.enviarParaOCR(payload.getSinistroId(), payload.getDocumentCode());
    }
    
    private void sendAlertToOps(String messageId, String queueName, Exception e) {
        // Enviar alerta para equipe de ops (email, Slack, PagerDuty, etc.)
        log.error("ALERTA: Mensagem na DLQ falhou após reprocessamento: messageId={}, queue={}", 
            messageId, 
            queueName);
        
        // TODO: Integrar com sistema de alertas
    }
}
```

### 5.4 Dashboard de Monitoramento DLQ

```java
@RestController
@RequestMapping("/api/admin/dlq")
@RequiredArgsConstructor
public class DLQMonitoringController {
    
    private final ServiceBusAdministrationClient serviceBusAdmin;
    
    @GetMapping("/stats")
    @RequiresPermission("admin:dlq:read")
    public ResponseEntity<DLQStats> getDLQStats() {
        List<QueueStats> stats = new ArrayList<>();
        
        // Obter estatísticas de cada DLQ
        for (String queueName : List.of("jarvis-ocr-queue", "pega-routing-queue", "alfresco-retry-queue")) {
            QueueRuntimeProperties props = serviceBusAdmin.getQueueRuntimeProperties(queueName);
            
            stats.add(new QueueStats(
                queueName,
                props.getActiveMessageCount(),
                props.getDeadLetterMessageCount(),
                props.getScheduledMessageCount()
            ));
        }
        
        return ResponseEntity.ok(new DLQStats(stats));
    }
    
    @PostMapping("/reprocess/{queueName}")
    @RequiresPermission("admin:dlq:write")
    public ResponseEntity<Void> reprocessQueue(@PathVariable String queueName) {
        log.info("Reprocessamento manual da DLQ solicitado: queue={}", queueName);
        
        // Trigger manual de reprocessamento
        // TODO: implementar
        
        return ResponseEntity.accepted().build();
    }
}
```

---

## 6. Circuit Breaker Detalhado

### 6.1 Estados do Circuit Breaker

```
┌─────────┐
│ CLOSED  │  ← Estado normal (requisições passam)
└────┬────┘
     │ (falhas > threshold)
     ↓
┌─────────┐
│  OPEN   │  ← Circuit aberto (todas requisições falham imediatamente)
└────┬────┘
     │ (após waitDuration)
     ↓
┌─────────┐
│ HALF_   │  ← Estado de teste (algumas requisições passam)
│ OPEN    │
└────┬────┘
     │ (sucesso → CLOSED | falha → OPEN)
```

### 6.2 Configuração Completa

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        registerHealthIndicator: true
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 100
        minimumNumberOfCalls: 10
        failureRateThreshold: 50
        slowCallRateThreshold: 70
        slowCallDurationThreshold: 5s
        permittedNumberOfCallsInHalfOpenState: 5
        automaticTransitionFromOpenToHalfOpenEnabled: true
        waitDurationInOpenState: 60s
        recordExceptions:
          - java.net.ConnectException
          - java.net.SocketTimeoutException
          - org.springframework.web.client.ResourceAccessException
        ignoreExceptions:
          - com.zurich.santander.appview.shared.exception.BusinessException
    
    instances:
      alfresco:
        baseConfig: default
        waitDurationInOpenState: 120s  # 2 minutos (sistema legado)
        slowCallDurationThreshold: 30s  # Uploads são lentos
        
      jarvis:
        baseConfig: default
        waitDurationInOpenState: 60s
        
      pega:
        baseConfig: default
        waitDurationInOpenState: 60s
        
      ccm:
        baseConfig: default
        failureRateThreshold: 60  # Menos crítico, aceita mais falhas
```

---

## 7. Métricas e Monitoramento

### 7.1 Métricas de Retry

```java
@Component
public class RetryMetrics {
    
    private final MeterRegistry meterRegistry;
    
    public RetryMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        
        // Registrar métricas de retry para cada sistema
        RetryRegistry.ofDefaults()
            .getAllRetries()
            .forEach(retry -> {
                retry.getEventPublisher()
                    .onRetry(event -> {
                        meterRegistry.counter("retry.attempts",
                            "name", retry.getName(),
                            "result", "retry"
                        ).increment();
                    })
                    .onSuccess(event -> {
                        meterRegistry.counter("retry.attempts",
                            "name", retry.getName(),
                            "result", "success"
                        ).increment();
                    })
                    .onError(event -> {
                        meterRegistry.counter("retry.attempts",
                            "name", retry.getName(),
                            "result", "failure"
                        ).increment();
                    });
            });
    }
}
```

### 7.2 Alertas Críticos

```yaml
alerts:
  - name: "DLQ Messages Accumulating"
    condition: "azure_servicebus_deadletter_message_count > 50"
    severity: "high"
    notification: ["ops-team@zurich.com.br"]
    
  - name: "Circuit Breaker Open"
    condition: "resilience4j_circuitbreaker_state{state='open'} > 0"
    duration: "5m"
    severity: "critical"
    notification: ["ops-team@zurich.com.br", "dev-team@zurich.com.br"]
    
  - name: "High Retry Rate"
    condition: "rate(retry_attempts_total{result='retry'}[5m]) > 10"
    severity: "medium"
    notification: ["dev-team@zurich.com.br"]
    
  - name: "Cache Miss Rate High"
    condition: "cache_miss_rate > 0.8"
    duration: "10m"
    severity: "low"
    notification: ["dev-team@zurich.com.br"]
```

---

## 8. Boas Práticas

### ✅ FAZER

1. **Sempre usar retry com exponential backoff** para chamadas externas
2. **Cachear leituras frequentes** (sinistros, clientes, propostas)
3. **Invalidar cache em todas as escritas**
4. **Monitorar DLQ** diariamente
5. **Configurar circuit breaker** para TODOS os sistemas externos
6. **Logar tentativas de retry** para análise
7. **Alertar quando DLQ > 50 mensagens**
8. **Usar TTL apropriado** (5min para dados dinâmicos, 1h para estáticos)

### ❌ NÃO FAZER

1. **Não fazer retry infinito** (máximo 3-5 tentativas)
2. **Não cachear operações de escrita**
3. **Não ignorar mensagens na DLQ** (sempre processar)
4. **Não usar cache sem invalidação**
5. **Não fazer retry em erros 4xx** (BadRequest, Unauthorized)
6. **Não logar payloads completos** em retry (sensibilidade de dados)

---

## 9. Checklist de Implementação

- [ ] Configurar Resilience4j (retry + circuit breaker)
- [ ] Implementar fallbacks em todos os Gateways
- [ ] Configurar Redis cache
- [ ] Implementar invalidação de cache em escritas
- [ ] Configurar Azure Service Bus com DLQ
- [ ] Implementar DLQ Processor job
- [ ] Criar dashboard de monitoramento DLQ
- [ ] Configurar alertas para DLQ
- [ ] Adicionar métricas de retry
- [ ] Documentar estratégia de cache por entidade
- [ ] Treinar equipe em análise de falhas

---

**Documento elaborado por:** Equipe de Arquitetura e SRE  
**Última atualização:** Janeiro de 2026  
**Status:** ✅ Completo e Aprovado
