# Observabilidade e Telemetria
**App View v2.0 - Logging, Tracing e Métricas**

**Versão:** 2.0.0  
**Data:** Janeiro de 2026  

---

## Visão Geral

Este documento define a estratégia completa de **observabilidade** do App View v2.0, cobrindo os três pilares fundamentais:

1. **Logs:** Narrativa estruturada do que aconteceu
2. **Traces:** Rastreamento end-to-end de requisições
3. **Metrics:** Medições quantitativas do sistema

**Princípio fundamental:** O sistema NÃO deve ser uma caixa preta. Devemos ser capazes de ler os logs e entender perfeitamente o que ocorreu.

---

## 1. Logging - Contando a História

### Filosofia de Logging

**Logs não são antipatern nem sujeira no código. Logs contam uma história.**

Cada operação crítica deve ter logs que permitam:
- ✅ Entender o fluxo completo de execução
- ✅ Identificar exatamente onde ocorreu um problema
- ✅ Rastrear o caminho de uma requisição através do sistema
- ✅ Debugar problemas em produção sem acesso ao debugger
- ✅ Auditoria completa de operações sensíveis

### Estrutura de Logs

Todos os logs devem ser **estruturados em JSON** com campos obrigatórios:

```json
{
  "timestamp": "2026-01-28T00:15:30.123Z",
  "level": "INFO",
  "service": "app-view-backend",
  "version": "2.0.0",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "spanId": "00f067aa0ba902b7",
  "logger": "com.zurich.santander.appview.modules.sinistro.application.ReceberSinistroUseCase",
  "thread": "http-nio-8080-exec-1",
  "message": "Recebendo sinistro",
  "context": {
    "numeroSinistro": "SIN123456",
    "canal": "WPC",
    "idempotencyKey": "550e8400-e29b-41d4-a716-446655440000"
  },
  "userId": "usuario@zurich.com.br",
  "environment": "production"
}
```

### Níveis de Log

| Nível | Uso | Exemplos |
|-------|-----|----------|
| **ERROR** | Erros que impedem a operação | Falha ao conectar no Alfresco, exception não tratada |
| **WARN** | Situações anormais mas recuperáveis | Circuit breaker aberto, retry sendo executado, data fora do SLA |
| **INFO** | Eventos importantes do negócio | Sinistro recebido, documento armazenado, OCR concluído |
| **DEBUG** | Informações detalhadas para debug | Valores de variáveis, decisões de lógica, queries SQL |
| **TRACE** | Informações extremamente detalhadas | Entry/exit de métodos, dados de payload completos |

### Configuração Logback (Spring Boot)

```xml
<!-- logback-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>
    
    <springProperty scope="context" name="APPLICATION_NAME" source="spring.application.name"/>
    <springProperty scope="context" name="APPLICATION_VERSION" source="info.app.version"/>
    
    <!-- Console Appender com JSON -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"service":"${APPLICATION_NAME}","version":"${APPLICATION_VERSION}"}</customFields>
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <includeMdcKeyName>spanId</includeMdcKeyName>
            <includeMdcKeyName>userId</includeMdcKeyName>
            <includeMdcKeyName>idempotencyKey</includeMdcKeyName>
            <includeMdcKeyName>numeroSinistro</includeMdcKeyName>
            <fieldNames>
                <timestamp>timestamp</timestamp>
                <version>version</version>
                <message>message</message>
                <logger>logger</logger>
                <thread>thread</thread>
                <level>level</level>
                <levelValue>levelValue</levelValue>
                <stackTrace>stackTrace</stackTrace>
            </fieldNames>
        </encoder>
    </appender>
    
    <!-- File Appender com Rotação -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/app-view.log</file>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/app-view.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>10GB</totalSizeCap>
        </rollingPolicy>
    </appender>
    
    <!-- Async Appender para Performance -->
    <appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
        <queueSize>512</queueSize>
        <discardingThreshold>0</discardingThreshold>
        <appender-ref ref="CONSOLE"/>
    </appender>
    
    <!-- Root Logger -->
    <root level="INFO">
        <appender-ref ref="ASYNC"/>
        <appender-ref ref="FILE"/>
    </root>
    
    <!-- Application Loggers -->
    <logger name="com.zurich.santander.appview" level="DEBUG"/>
    
    <!-- Framework Loggers -->
    <logger name="org.springframework.web" level="INFO"/>
    <logger name="org.hibernate.SQL" level="DEBUG"/>
    <logger name="org.hibernate.type.descriptor.sql.BasicBinder" level="TRACE"/>
    
    <!-- External Systems Loggers -->
    <logger name="org.apache.chemistry.opencmis" level="INFO"/>
</configuration>
```

### Dependências Maven

```xml
<dependencies>
    <!-- Logback with JSON encoding -->
    <dependency>
        <groupId>net.logstash.logback</groupId>
        <artifactId>logstash-logback-encoder</artifactId>
        <version>7.4</version>
    </dependency>
    
    <!-- SLF4J API -->
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
    </dependency>
</dependencies>
```

---

## 2. Tracing Distribuído com OpenTelemetry

### Arquitetura de Tracing

```
[Request]
  ↓ traceId: 4bf92f35... (gerado ou recebido)
[API Gateway]
  ↓ spanId: parent-span
[SinistroController] → span: http.server
  ↓ spanId: controller-span
[ReceberSinistroUseCase] → span: sinistro.receber
  ↓ spanId: usecase-span
[SinistroRepository] → span: db.query
  ↓ spanId: db-span
[PostgreSQL]
  ↓
[DomainEventPublisher] → span: event.publish
  ↓ spanId: event-span
[Azure Service Bus]
  ↓ (novo contexto com mesmo traceId)
[SinistroRecebidoHandler] → span: event.handler
  ↓ spanId: handler-span
[AlfrescoGateway] → span: http.client.alfresco
  ↓ spanId: alfresco-span
[Alfresco ECM]
```

**Todos os componentes compartilham o mesmo `traceId`**, permitindo rastreamento end-to-end.

### Configuração OpenTelemetry

#### Dependências Maven

```xml
<dependencies>
    <!-- OpenTelemetry BOM -->
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-bom</artifactId>
        <version>1.34.1</version>
        <type>pom</type>
        <scope>import</scope>
    </dependency>
    
    <!-- OpenTelemetry API -->
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-api</artifactId>
    </dependency>
    
    <!-- OpenTelemetry SDK -->
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-sdk</artifactId>
    </dependency>
    
    <!-- OpenTelemetry SDK Extension Autoconfigure -->
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-sdk-extension-autoconfigure</artifactId>
    </dependency>
    
    <!-- OpenTelemetry Instrumentation for Spring Boot -->
    <dependency>
        <groupId>io.opentelemetry.instrumentation</groupId>
        <artifactId>opentelemetry-spring-boot-starter</artifactId>
        <version>2.0.0</version>
    </dependency>
    
    <!-- OpenTelemetry Exporter para Azure Application Insights -->
    <dependency>
        <groupId>com.azure</groupId>
        <artifactId>azure-monitor-opentelemetry-exporter</artifactId>
        <version>1.0.0-beta.13</version>
    </dependency>
</dependencies>
```

#### Configuração Spring Boot

```yaml
# application.yml
spring:
  application:
    name: app-view-backend

management:
  otlp:
    tracing:
      endpoint: ${OTEL_EXPORTER_OTLP_ENDPOINT:http://localhost:4318/v1/traces}
  tracing:
    sampling:
      probability: 1.0  # 100% em DEV, 0.1 (10%) em PRD

# OpenTelemetry Configuration
otel:
  service:
    name: ${spring.application.name}
    version: ${info.app.version:2.0.0}
  traces:
    exporter: otlp
  metrics:
    exporter: otlp
  logs:
    exporter: otlp
  exporter:
    otlp:
      endpoint: ${AZURE_MONITOR_CONNECTION_STRING}
  resource:
    attributes:
      deployment.environment: ${ENVIRONMENT:development}
      service.namespace: zurich-santander
```

#### Configuração Java

```java
package com.zurich.santander.appview.shared.infrastructure.config;

import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.context.propagation.ContextPropagators;
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter;
import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.resources.Resource;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor;
import io.opentelemetry.semconv.resource.attributes.ResourceAttributes;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OpenTelemetryConfig {
    
    @Value("${spring.application.name}")
    private String serviceName;
    
    @Value("${info.app.version:2.0.0}")
    private String serviceVersion;
    
    @Value("${otel.exporter.otlp.endpoint}")
    private String otlpEndpoint;
    
    @Bean
    public OpenTelemetry openTelemetry() {
        Resource resource = Resource.getDefault()
            .merge(Resource.create(Attributes.of(
                ResourceAttributes.SERVICE_NAME, serviceName,
                ResourceAttributes.SERVICE_VERSION, serviceVersion,
                ResourceAttributes.SERVICE_NAMESPACE, "zurich-santander",
                ResourceAttributes.DEPLOYMENT_ENVIRONMENT, getEnvironment()
            )));
        
        SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
            .addSpanProcessor(BatchSpanProcessor.builder(
                OtlpGrpcSpanExporter.builder()
                    .setEndpoint(otlpEndpoint)
                    .build()
            ).build())
            .setResource(resource)
            .build();
        
        return OpenTelemetrySdk.builder()
            .setTracerProvider(tracerProvider)
            .setPropagators(ContextPropagators.create(
                io.opentelemetry.extension.trace.propagation.B3Propagator.injectingSingleHeader()
            ))
            .buildAndRegisterGlobal();
    }
    
    @Bean
    public Tracer tracer(OpenTelemetry openTelemetry) {
        return openTelemetry.getTracer(serviceName, serviceVersion);
    }
    
    private String getEnvironment() {
        String env = System.getenv("ENVIRONMENT");
        return env != null ? env : "development";
    }
}
```

---

## 3. Logging com Contexto de Tracing

### MDC (Mapped Diagnostic Context)

Usar MDC para propagar `traceId` e `spanId` através de todos os logs:

```java
package com.zurich.santander.appview.shared.infrastructure.interceptor;

import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.SpanContext;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.MDC;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;

@Component
public class TracingInterceptor implements HandlerInterceptor {
    
    @Override
    public boolean preHandle(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler) {
        
        // Obter span atual do OpenTelemetry
        Span currentSpan = Span.current();
        SpanContext spanContext = currentSpan.getSpanContext();
        
        // Propagar traceId e spanId para MDC (logs)
        MDC.put("traceId", spanContext.getTraceId());
        MDC.put("spanId", spanContext.getSpanId());
        
        // Adicionar ao response header para client-side tracing
        response.setHeader("X-Trace-Id", spanContext.getTraceId());
        
        return true;
    }
    
    @Override
    public void afterCompletion(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler,
            Exception ex) {
        
        MDC.clear();
    }
}
```

### Contexto Adicional no MDC

```java
package com.zurich.santander.appview.shared.infrastructure.aspect;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.slf4j.MDC;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class BusinessContextAspect {
    
    /**
     * Adiciona contexto de negócio ao MDC para enriquecer logs
     */
    @Around("@annotation(com.zurich.santander.appview.shared.annotation.UseCase)")
    public Object enrichMDC(ProceedingJoinPoint joinPoint) throws Throwable {
        try {
            // Extrair informações do command/query
            Object[] args = joinPoint.getArgs();
            if (args.length > 0) {
                Object firstArg = args[0];
                
                // Adicionar contexto específico baseado no tipo
                if (firstArg instanceof ReceberSinistroCommand cmd) {
                    MDC.put("numeroSinistro", cmd.numeroSinistro());
                    MDC.put("idempotencyKey", cmd.idempotencyKey());
                }
                // ... outros comandos
            }
            
            return joinPoint.proceed();
            
        } finally {
            // Limpar contexto específico (manter traceId/spanId)
            MDC.remove("numeroSinistro");
            MDC.remove("idempotencyKey");
        }
    }
}
```

---

## 4. Logging Prático - Exemplos Completos

### Use Case com Logging Completo

```java
package com.zurich.santander.appview.modules.sinistro.application.usecase;

import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.Tracer;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.slf4j.MDC;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
@Slf4j
public class ReceberSinistroUseCase {
    
    private final SinistroRepository repository;
    private final IdempotencyService idempotencyService;
    private final DomainEventPublisher eventPublisher;
    private final Tracer tracer;
    
    @Transactional
    public SinistroId executar(ReceberSinistroCommand command) {
        // Criar span personalizado
        Span span = tracer.spanBuilder("sinistro.receber")
            .setAttribute("sinistro.numero", command.numeroSinistro())
            .setAttribute("sinistro.canal", command.canal())
            .startSpan();
        
        try (var scope = span.makeCurrent()) {
            
            // LOG 1: Início da operação
            log.info("Iniciando recepção de sinistro: numero={}, canal={}, idempotencyKey={}", 
                command.numeroSinistro(), 
                command.canal(),
                command.idempotencyKey());
            
            // LOG 2: Verificação de idempotência
            log.debug("Verificando idempotência para key={}", command.idempotencyKey());
            var cached = idempotencyService.buscar(command.idempotencyKey(), SinistroId.class);
            
            if (cached.isPresent()) {
                // LOG 3: Request idempotente (não é erro!)
                log.info("Sinistro já processado anteriormente (idempotente): sinistroId={}, idempotencyKey={}", 
                    cached.get(), 
                    command.idempotencyKey());
                span.setAttribute("idempotent", true);
                return cached.get();
            }
            
            // LOG 4: Criação do agregado
            log.debug("Criando agregado Sinistro com dados: {}", command);
            var sinistro = Sinistro.criar(command.toData(), repository);
            
            // LOG 5: Persistência
            log.debug("Persistindo sinistro: id={}", sinistro.getId());
            repository.save(sinistro);
            log.info("Sinistro persistido com sucesso: id={}, numero={}", 
                sinistro.getId(), 
                sinistro.getNumeroSinistro());
            
            // LOG 6: Eventos de domínio
            var events = sinistro.getDomainEvents();
            log.debug("Publicando {} eventos de domínio", events.size());
            events.forEach(event -> {
                log.debug("Publicando evento: type={}, sinistroId={}", 
                    event.getClass().getSimpleName(), 
                    sinistro.getId());
                eventPublisher.publish(event);
            });
            sinistro.clearDomainEvents();
            
            // LOG 7: Cache de idempotência
            log.debug("Cacheando resultado para idempotencyKey={}", command.idempotencyKey());
            idempotencyService.cachear(command.idempotencyKey(), sinistro.getId());
            
            // LOG 8: Sucesso final
            log.info("Sinistro recebido com sucesso: id={}, numero={}, status={}", 
                sinistro.getId(), 
                sinistro.getNumeroSinistro(),
                sinistro.getStatus());
            
            span.setAttribute("sinistro.id", sinistro.getId().toString());
            span.setAttribute("success", true);
            
            return sinistro.getId();
            
        } catch (SinistroDuplicadoException e) {
            // LOG 9: Erro de negócio (RN-001)
            log.warn("Tentativa de criar sinistro duplicado: numero={}, message={}", 
                command.numeroSinistro(), 
                e.getMessage());
            span.recordException(e);
            span.setAttribute("error.type", "business");
            throw e;
            
        } catch (DataInvalidaException e) {
            // LOG 10: Erro de validação (RN-002)
            log.warn("Dados inválidos no sinistro: numero={}, message={}", 
                command.numeroSinistro(), 
                e.getMessage());
            span.recordException(e);
            span.setAttribute("error.type", "validation");
            throw e;
            
        } catch (Exception e) {
            // LOG 11: Erro técnico inesperado
            log.error("Erro inesperado ao receber sinistro: numero={}", 
                command.numeroSinistro(), 
                e);
            span.recordException(e);
            span.setAttribute("error.type", "technical");
            throw new SinistroProcessingException("Erro ao processar sinistro", e);
            
        } finally {
            span.end();
        }
    }
}
```

### Gateway com Logging de Integração

```java
package com.zurich.santander.appview.modules.sinistro.infrastructure.gateway;

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.Tracer;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
@Slf4j
public class AlfrescoGateway {
    
    private final Session cmisSession;
    private final Tracer tracer;
    
    @CircuitBreaker(name = "alfresco", fallbackMethod = "uploadFallback")
    public String uploadDocumento(DocumentoUpload upload) {
        Span span = tracer.spanBuilder("http.client.alfresco.upload")
            .setAttribute("alfresco.operation", "createDocument")
            .setAttribute("documento.nome", upload.getNomeArquivo())
            .setAttribute("documento.tamanho", upload.getTamanho())
            .startSpan();
        
        try (var scope = span.makeCurrent()) {
            
            // LOG 1: Início da integração
            log.info("Iniciando upload para Alfresco: arquivo={}, tamanho={}bytes, caminho={}", 
                upload.getNomeArquivo(), 
                upload.getTamanho(),
                upload.getCaminho());
            
            long startTime = System.currentTimeMillis();
            
            // LOG 2: Criar estrutura de pastas
            log.debug("Criando estrutura de pastas: caminho={}", upload.getCaminho());
            Folder pasta = criarEstruturaPastas(upload.getCaminho());
            log.debug("Estrutura de pastas criada: folderId={}", pasta.getId());
            
            // LOG 3: Preparar content stream
            log.debug("Preparando content stream: mimeType={}", upload.getMimeType());
            ContentStream contentStream = new ContentStreamImpl(
                upload.getNomeArquivo(),
                BigInteger.valueOf(upload.getTamanho()),
                upload.getMimeType(),
                new ByteArrayInputStream(upload.getConteudo())
            );
            
            // LOG 4: Propriedades CMIS
            Map<String, Object> properties = Map.of(
                PropertyIds.OBJECT_TYPE_ID, "zsSinistros:documentos_Sinistros",
                PropertyIds.NAME, upload.getNomeArquivo(),
                "zsSinistros:numeroSinistro", upload.getNumeroSinistro(),
                "zsSinistros:tipoDocumento", upload.getTipoDocumento()
            );
            log.debug("Propriedades CMIS configuradas: type={}, numeroSinistro={}", 
                properties.get(PropertyIds.OBJECT_TYPE_ID),
                properties.get("zsSinistros:numeroSinistro"));
            
            // LOG 5: Executar upload
            log.debug("Executando createDocument no Alfresco");
            Document doc = pasta.createDocument(
                properties,
                contentStream,
                VersioningState.MAJOR
            );
            
            long duration = System.currentTimeMillis() - startTime;
            
            // LOG 6: Sucesso
            log.info("Upload concluído com sucesso: documentId={}, arquivo={}, duration={}ms", 
                doc.getId(), 
                upload.getNomeArquivo(),
                duration);
            
            span.setAttribute("alfresco.document.id", doc.getId());
            span.setAttribute("duration.ms", duration);
            span.setAttribute("success", true);
            
            return doc.getId();
            
        } catch (CmisConnectionException e) {
            // LOG 7: Erro de conexão
            log.error("Erro de conexão com Alfresco: arquivo={}, error={}", 
                upload.getNomeArquivo(), 
                e.getMessage());
            span.recordException(e);
            span.setAttribute("error.type", "connection");
            throw new AlfrescoIndisponivelException("Alfresco indisponível", e);
            
        } catch (CmisStorageException e) {
            // LOG 8: Erro de armazenamento
            log.error("Alfresco sem espaço disponível: arquivo={}, error={}", 
                upload.getNomeArquivo(), 
                e.getMessage());
            span.recordException(e);
            span.setAttribute("error.type", "storage");
            throw new AlfrescoSemEspacoException("Alfresco sem espaço", e);
            
        } catch (Exception e) {
            // LOG 9: Erro genérico
            log.error("Erro inesperado ao fazer upload para Alfresco: arquivo={}", 
                upload.getNomeArquivo(), 
                e);
            span.recordException(e);
            span.setAttribute("error.type", "unknown");
            throw e;
            
        } finally {
            span.end();
        }
    }
    
    private String uploadFallback(DocumentoUpload upload, Throwable t) {
        // LOG 10: Fallback ativado
        log.warn("Fallback ativado para upload Alfresco: arquivo={}, reason={}, errorType={}", 
            upload.getNomeArquivo(), 
            t.getMessage(),
            t.getClass().getSimpleName());
        
        // Enfileirar para retry assíncrono
        log.info("Enfileirando documento para retry assíncrono: arquivo={}", 
            upload.getNomeArquivo());
        
        return null;
    }
}
```

---

## 5. Métricas com OpenTelemetry

### Configuração de Métricas

```java
package com.zurich.santander.appview.shared.infrastructure.metrics;

import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.metrics.Meter;
import io.opentelemetry.api.metrics.LongCounter;
import io.opentelemetry.api.metrics.DoubleHistogram;
import org.springframework.stereotype.Component;

@Component
public class AppViewMetrics {
    
    private final Meter meter;
    
    // Counters
    private final LongCounter sinistrosRecebidos;
    private final LongCounter sinistrosProcessados;
    private final LongCounter sinistrosRejeitados;
    private final LongCounter alfrescoUploads;
    private final LongCounter alfrescoErros;
    private final LongCounter pegaCasesCreated;
    
    // Histograms
    private final DoubleHistogram requestDuration;
    private final DoubleHistogram alfrescoUploadDuration;
    private final DoubleHistogram databaseQueryDuration;
    
    public AppViewMetrics(OpenTelemetry openTelemetry) {
        this.meter = openTelemetry.getMeter("app-view-backend");
        
        // Inicializar counters
        this.sinistrosRecebidos = meter
            .counterBuilder("sinistros.recebidos.total")
            .setDescription("Total de sinistros recebidos")
            .setUnit("sinistros")
            .build();
        
        this.sinistrosProcessados = meter
            .counterBuilder("sinistros.processados.total")
            .setDescription("Total de sinistros processados com sucesso")
            .setUnit("sinistros")
            .build();
        
        this.sinistrosRejeitados = meter
            .counterBuilder("sinistros.rejeitados.total")
            .setDescription("Total de sinistros rejeitados")
            .setUnit("sinistros")
            .build();
        
        this.alfrescoUploads = meter
            .counterBuilder("alfresco.uploads.total")
            .setDescription("Total de uploads para Alfresco")
            .setUnit("uploads")
            .build();
        
        this.alfrescoErros = meter
            .counterBuilder("alfresco.erros.total")
            .setDescription("Total de erros do Alfresco")
            .setUnit("erros")
            .build();
        
        // Inicializar histograms
        this.requestDuration = meter
            .histogramBuilder("http.server.duration")
            .setDescription("Duração de requisições HTTP")
            .setUnit("ms")
            .build();
        
        this.alfrescoUploadDuration = meter
            .histogramBuilder("alfresco.upload.duration")
            .setDescription("Duração de uploads para Alfresco")
            .setUnit("ms")
            .build();
    }
    
    // Métodos para incrementar métricas
    public void incrementSinistrosRecebidos(String canal) {
        sinistrosRecebidos.add(1, 
            io.opentelemetry.api.common.Attributes.of(
                io.opentelemetry.api.common.AttributeKey.stringKey("canal"), canal
            ));
    }
    
    public void recordAlfrescoUploadDuration(long durationMs, boolean success) {
        alfrescoUploadDuration.record(durationMs,
            io.opentelemetry.api.common.Attributes.of(
                io.opentelemetry.api.common.AttributeKey.booleanKey("success"), success
            ));
    }
}
```

### Uso de Métricas

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ReceberSinistroUseCase {
    
    private final AppViewMetrics metrics;
    
    @Transactional
    public SinistroId executar(ReceberSinistroCommand command) {
        long startTime = System.currentTimeMillis();
        
        try {
            // Incrementar métrica de sinistros recebidos
            metrics.incrementSinistrosRecebidos(command.canal());
            
            var sinistro = Sinistro.criar(command.toData(), repository);
            repository.save(sinistro);
            
            // Incrementar métrica de sucesso
            metrics.incrementSinistrosProcessados(command.canal());
            
            return sinistro.getId();
            
        } catch (Exception e) {
            // Incrementar métrica de erro
            metrics.incrementSinistrosRejeitados(command.canal());
            throw e;
            
        } finally {
            // Registrar duração
            long duration = System.currentTimeMillis() - startTime;
            metrics.recordRequestDuration(duration);
        }
    }
}
```

---

## 6. Queries para Análise de Logs

### Azure Application Insights (KQL)

```kusto
// 1. Rastrear uma requisição específica por traceId
traces
| where customDimensions.traceId == "4bf92f3577b34da6a3ce929d0e0e4736"
| project timestamp, message, severityLevel, customDimensions
| order by timestamp asc

// 2. Todas as operações de um sinistro específico
traces
| where customDimensions.numeroSinistro == "SIN123456"
| project timestamp, message, severityLevel, logger = customDimensions.logger
| order by timestamp asc

// 3. Erros nas últimas 24 horas
traces
| where timestamp > ago(24h)
| where severityLevel >= 3  // ERROR
| summarize count() by bin(timestamp, 1h), logger = tostring(customDimensions.logger)
| render timechart

// 4. Performance de integrações
traces
| where message has "Upload concluído com sucesso"
| extend duration = toint(customDimensions["duration.ms"])
| summarize 
    count(),
    avg(duration),
    percentile(duration, 95),
    percentile(duration, 99)
  by bin(timestamp, 5m)
| render timechart

// 5. Taxa de idempotência
traces
| where message has "já processado anteriormente"
| summarize idempotent_count = count() by bin(timestamp, 1h)

// 6. Circuit breaker aberto
traces
| where message has "Fallback ativado"
| project timestamp, message, customDimensions.arquivo
| order by timestamp desc
```

---

## 7. Dashboard de Observabilidade

### Grafana Dashboard (JSON)

```json
{
  "dashboard": {
    "title": "App View - Observabilidade",
    "panels": [
      {
        "title": "Taxa de Requisições",
        "targets": [
          {
            "query": "rate(http_server_requests_total[5m])"
          }
        ]
      },
      {
        "title": "Latência P95",
        "targets": [
          {
            "query": "histogram_quantile(0.95, rate(http_server_duration_bucket[5m]))"
          }
        ]
      },
      {
        "title": "Taxa de Erro",
        "targets": [
          {
            "query": "rate(http_server_requests_total{status=~\"5..\"}[5m]) / rate(http_server_requests_total[5m])"
          }
        ]
      },
      {
        "title": "Alfresco - Duração de Upload",
        "targets": [
          {
            "query": "histogram_quantile(0.95, rate(alfresco_upload_duration_bucket[5m]))"
          }
        ]
      },
      {
        "title": "Circuit Breaker - Estados",
        "targets": [
          {
            "query": "resilience4j_circuitbreaker_state"
          }
        ]
      }
    ]
  }
}
```

---

## 8. Alertas Críticos

### Configuração de Alertas (Azure Monitor)

```yaml
alerts:
  - name: "High Error Rate"
    condition: "rate(http_server_requests_total{status=~\"5..\"}[5m]) > 0.05"
    severity: "critical"
    notification: ["ops-team@zurich.com.br"]
    
  - name: "Circuit Breaker Open"
    condition: "resilience4j_circuitbreaker_state{state=\"open\"} > 0"
    duration: "5m"
    severity: "high"
    notification: ["ops-team@zurich.com.br"]
    
  - name: "High Latency"
    condition: "histogram_quantile(0.95, rate(http_server_duration_bucket[5m])) > 2000"
    duration: "10m"
    severity: "medium"
    notification: ["dev-team@zurich.com.br"]
    
  - name: "Alfresco Integration Failure"
    condition: "rate(alfresco_erros_total[5m]) > 10"
    severity: "high"
    notification: ["ops-team@zurich.com.br", "integration-team@zurich.com.br"]
```

---

## 9. Boas Práticas

### ✅ FAZER

1. **Logar sempre com contexto:** Incluir IDs relevantes (sinistroId, clienteId, etc.)
2. **Usar níveis apropriados:** ERROR para erros, INFO para eventos de negócio
3. **Logar entrada e saída de operações críticas**
4. **Incluir duração de operações longas**
5. **Logar exceções com stack trace completo**
6. **Usar mensagens descritivas e consistentes**
7. **Incluir traceId em todos os logs**
8. **Criar spans customizados para operações importantes**
9. **Adicionar atributos significativos aos spans**
10. **Usar MDC para contexto thread-local**

### ❌ NÃO FAZER

1. **Não logar dados sensíveis** (senhas, tokens, CPF completo)
2. **Não usar System.out.println()** (sempre usar logger)
3. **Não logar em loops intensivos** sem rate limiting
4. **Não ignorar exceções silenciosamente**
5. **Não usar string concatenation** em logs (usar placeholders)
6. **Não logar objetos complexos** sem toString apropriado
7. **Não criar spans desnecessários** (overhead)

---

## 10. Checklist de Implementação

- [ ] Configurar Logback com JSON encoder
- [ ] Adicionar dependências OpenTelemetry
- [ ] Configurar OpenTelemetry SDK
- [ ] Implementar TracingInterceptor
- [ ] Criar AppViewMetrics
- [ ] Adicionar logs em todos os Use Cases
- [ ] Adicionar logs em todos os Gateways
- [ ] Configurar exportação para Azure Application Insights
- [ ] Criar dashboards no Grafana
- [ ] Configurar alertas críticos
- [ ] Documentar queries úteis de análise
- [ ] Treinar equipe em análise de logs/traces

---

**Documento elaborado por:** Equipe de Arquitetura e SRE  
**Última atualização:** Janeiro de 2026  
**Status:** ✅ Completo e Aprovado
