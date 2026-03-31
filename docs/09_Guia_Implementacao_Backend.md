# Guia de Implementação Backend
**App View v2.0 - Spring Boot + Clean Architecture + DDD**

**Versão:** 2.0.0  
**Data:** Janeiro de 2026  

---

## Visão Geral

Este guia fornece instruções práticas para implementação do backend seguindo **Clean Architecture**, **DDD** e **Event-Driven Architecture** com Spring Boot 3.3+ e Java 21.

---

## Estrutura de Diretórios

```
app-view-backend/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/zurich/santander/appview/
│   │   │       ├── AppViewApplication.java
│   │   │       ├── modules/
│   │   │       │   ├── sinistro/
│   │   │       │   │   ├── domain/
│   │   │       │   │   │   ├── model/
│   │   │       │   │   │   │   ├── Sinistro.java
│   │   │       │   │   │   │   ├── SinistroId.java
│   │   │       │   │   │   │   ├── StatusSinistro.java
│   │   │       │   │   │   │   └── LossInfo.java
│   │   │       │   │   │   ├── repository/
│   │   │       │   │   │   │   └── SinistroRepository.java
│   │   │       │   │   │   ├── event/
│   │   │       │   │   │   │   ├── SinistroRecebidoEvent.java
│   │   │       │   │   │   │   ├── SinistroArmazenadoEvent.java
│   │   │       │   │   │   │   └── OCRConcluidoEvent.java
│   │   │       │   │   │   └── exception/
│   │   │       │   │   │       ├── SinistroDuplicadoException.java
│   │   │       │   │   │       └── DataInvalidaException.java
│   │   │       │   │   ├── application/
│   │   │       │   │   │   ├── usecase/
│   │   │       │   │   │   │   ├── ReceberSinistroUseCase.java
│   │   │       │   │   │   │   ├── ArmazenarSinistroNoECMUseCase.java
│   │   │       │   │   │   │   ├── EnviarSinistroParaOCRUseCase.java
│   │   │       │   │   │   │   ├── ProcessarCallbackOCRUseCase.java
│   │   │       │   │   │   │   └── ConsultarSinistroUseCase.java
│   │   │       │   │   │   ├── command/
│   │   │       │   │   │   │   ├── ReceberSinistroCommand.java
│   │   │       │   │   │   │   └── ArmazenarSinistroCommand.java
│   │   │       │   │   │   ├── query/
│   │   │       │   │   │   │   └── SinistroFiltro.java
│   │   │       │   │   │   └── handler/
│   │   │       │   │   │       ├── SinistroRecebidoHandler.java
│   │   │       │   │   │       ├── SinistroArmazenadoHandler.java
│   │   │       │   │   │       └── OCRConcluidoHandler.java
│   │   │       │   │   └── infrastructure/
│   │   │       │   │       ├── persistence/
│   │   │       │   │       │   ├── SinistroRepositoryImpl.java
│   │   │       │   │       │   └── SinistroEntityMapper.java
│   │   │       │   │       ├── web/
│   │   │       │   │       │   ├── SinistroController.java
│   │   │       │   │       │   ├── JarvisCallbackController.java
│   │   │       │   │       │   ├── dto/
│   │   │       │   │       │   │   ├── SinistroRequest.java
│   │   │       │   │       │   │   ├── SinistroResponse.java
│   │   │       │   │       │   │   └── JarvisCallbackRequest.java
│   │   │       │   │       │   └── mapper/
│   │   │       │   │       │       └── SinistroDTOMapper.java
│   │   │       │   │       └── gateway/
│   │   │       │   │           ├── AlfrescoGateway.java
│   │   │       │   │           ├── JarvisGateway.java
│   │   │       │   │           └── PegaGateway.java
│   │   │       │   ├── cliente/
│   │   │       │   │   ├── domain/
│   │   │       │   │   ├── application/
│   │   │       │   │   └── infrastructure/
│   │   │       │   ├── contrato/
│   │   │       │   ├── pericia/
│   │   │       │   ├── portal/
│   │   │       │   └── iam/
│   │   │       └── shared/
│   │   │           ├── domain/
│   │   │           │   ├── AggregateRoot.java
│   │   │           │   ├── Entity.java
│   │   │           │   ├── ValueObject.java
│   │   │           │   └── DomainEvent.java
│   │   │           ├── infrastructure/
│   │   │           │   ├── config/
│   │   │           │   │   ├── SecurityConfig.java
│   │   │           │   │   ├── DatabaseConfig.java
│   │   │           │   │   ├── CacheConfig.java
│   │   │           │   │   ├── EventConfig.java
│   │   │           │   │   └── OpenApiConfig.java
│   │   │           │   ├── aspect/
│   │   │           │   │   ├── IdempotencyAspect.java
│   │   │           │   │   ├── AuditAspect.java
│   │   │           │   │   └── LoggingAspect.java
│   │   │           │   ├── exception/
│   │   │           │   │   ├── GlobalExceptionHandler.java
│   │   │           │   │   └── ErrorResponse.java
│   │   │           │   └── interceptor/
│   │   │           │       ├── RequestIdInterceptor.java
│   │   │           │       └── IdempotencyInterceptor.java
│   │   │           └── annotation/
│   │   │               ├── UseCase.java
│   │   │               ├── Gateway.java
│   │   │               └── RequiresPermission.java
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-dev.yml
│   │       ├── application-uat.yml
│   │       ├── application-prd.yml
│   │       ├── db/migration/
│   │       │   └── V1__initial_schema.sql
│   │       └── logback-spring.xml
│   └── test/
│       ├── java/
│       │   └── com/zurich/santander/appview/
│       │       ├── modules/
│       │       │   └── sinistro/
│       │       │       ├── domain/
│       │       │       │   └── SinistroTest.java
│       │       │       ├── application/
│       │       │       │   └── ReceberSinistroUseCaseTest.java
│       │       │       └── infrastructure/
│       │       │           └── SinistroControllerTest.java
│       │       └── architecture/
│       │           └── ArchitectureTest.java
│       └── resources/
│           └── application-test.yml
├── pom.xml
├── .gitignore
├── README.md
└── docker-compose.yml
```

---

## Domain Layer

### 1. Aggregate Root

Base class para todos os agregados:

```java
package com.zurich.santander.appview.shared.domain;

import jakarta.persistence.*;
import lombok.Getter;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;

@MappedSuperclass
@Getter
public abstract class AggregateRoot<ID> {
    
    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;
    
    @Column(name = "updated_at", nullable = false)
    private Instant updatedAt;
    
    @Version
    @Column(name = "version")
    private Long version;
    
    @Transient
    private List<DomainEvent> domainEvents = new ArrayList<>();
    
    protected AggregateRoot() {
        this.createdAt = Instant.now();
        this.updatedAt = Instant.now();
    }
    
    protected void registerEvent(DomainEvent event) {
        this.domainEvents.add(event);
    }
    
    public List<DomainEvent> getDomainEvents() {
        return List.copyOf(domainEvents);
    }
    
    public void clearDomainEvents() {
        this.domainEvents.clear();
    }
    
    protected void touch() {
        this.updatedAt = Instant.now();
    }
    
    public abstract ID getId();
}
```

---

### 2. Entity - Sinistro

```java
package com.zurich.santander.appview.modules.sinistro.domain.model;

import com.zurich.santander.appview.shared.domain.AggregateRoot;
import com.zurich.santander.appview.modules.sinistro.domain.event.*;
import jakarta.persistence.*;
import lombok.Getter;
import lombok.NoArgsConstructor;

@Entity
@Table(name = "tb_sinistro", schema = "public")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Sinistro extends AggregateRoot<SinistroId> {
    
    @EmbeddedId
    private SinistroId id;
    
    @Column(name = "numero_sinistro", nullable = false, unique = true, length = 50)
    private String numeroSinistro;
    
    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 30)
    private StatusSinistro status;
    
    @Embedded
    private LossInfo lossInfo;
    
    @Column(length = 20)
    private String canal;
    
    @Column(name = "data_recebimento", nullable = false)
    private LocalDate dataRecebimento;
    
    @Column(length = 50)
    private String apolice;
    
    @Column(length = 50)
    private String ramo;
    
    @Column(name = "cliente_id", length = 20)
    private String clienteId;
    
    @Column(name = "alfresco_document_id", length = 100)
    private String alfrescoDocumentId;
    
    @Column(name = "pega_case_id", length = 50)
    private String pegaCaseId;
    
    @Column(name = "ocr_legibilidade")
    private Integer ocrLegibilidade;
    
    @Column(name = "ocr_acuracidade")
    private Integer ocrAcuracidade;
    
    @Column(name = "deleted_at")
    private Instant deletedAt;
    
    // Factory method
    public static Sinistro criar(
            SinistroData data,
            SinistroRepository repository) {
        
        // RN-001: Número de sinistro deve ser único
        if (repository.existsByNumeroSinistro(data.numeroSinistro())) {
            throw new SinistroDuplicadoException(data.numeroSinistro());
        }
        
        // RN-002: Data não pode ser futura
        if (data.dataRecebimento().isAfter(LocalDate.now())) {
            throw new DataInvalidaException("Data de recebimento não pode ser futura");
        }
        
        var sinistro = new Sinistro();
        sinistro.id = SinistroId.generate();
        sinistro.numeroSinistro = data.numeroSinistro();
        sinistro.status = StatusSinistro.RECEBIDO;
        sinistro.lossInfo = data.lossInfo();
        sinistro.canal = data.canal();
        sinistro.dataRecebimento = data.dataRecebimento();
        sinistro.apolice = data.apolice();
        sinistro.ramo = data.ramo();
        sinistro.clienteId = data.clienteId();
        
        sinistro.registerEvent(new SinistroRecebidoEvent(sinistro.id));
        
        return sinistro;
    }
    
    // Domain behaviors
    public void marcarComoArmazenado(String alfrescoDocumentId) {
        this.alfrescoDocumentId = alfrescoDocumentId;
        this.alterarStatus(StatusSinistro.ARMAZENADO);
        this.registerEvent(new SinistroArmazenadoEvent(this.id, alfrescoDocumentId));
    }
    
    public void atualizarScoresOCR(Integer legibilidade, Integer acuracidade) {
        // RN-015: Validar scores (0-100)
        if (legibilidade < 0 || legibilidade > 100) {
            throw new ScoreInvalidoException("Legibilidade", legibilidade);
        }
        if (acuracidade < 0 || acuracidade > 100) {
            throw new ScoreInvalidoException("Acuracidade", acuracidade);
        }
        
        this.ocrLegibilidade = legibilidade;
        this.ocrAcuracidade = acuracidade;
        this.alterarStatus(StatusSinistro.OCR_CONCLUIDO);
        this.touch();
        
        // RN-016, RN-019, RN-020: Regras de legibilidade
        if (legibilidade < 40) {
            this.marcarComoRejeitado("Legibilidade insuficiente");
            return;
        }
        
        if (legibilidade < 60) {
            this.marcarParaRevisaoManual();
            return;
        }
        
        this.registerEvent(new OCRConcluidoEvent(this.id, legibilidade, acuracidade));
    }
    
    public void rotearParaPega(String pegaCaseId) {
        this.pegaCaseId = pegaCaseId;
        this.alterarStatus(StatusSinistro.ROTEADO_PARA_PEGA);
        this.touch();
    }
    
    private void alterarStatus(StatusSinistro novoStatus) {
        // RN-076: Validar transição de status
        if (!this.status.podeTransicionarPara(novoStatus)) {
            throw new TransicaoInvalidaException(this.status, novoStatus);
        }
        this.status = novoStatus;
    }
    
    private void marcarParaRevisaoManual() {
        this.status = StatusSinistro.REVISAO_MANUAL;
        this.touch();
        this.registerEvent(new DocumentoRequerRevisaoEvent(this.id));
    }
    
    private void marcarComoRejeitado(String motivo) {
        this.status = StatusSinistro.REJEITADO;
        this.touch();
        this.registerEvent(new SinistroRejeitadoEvent(this.id, motivo));
    }
    
    @Override
    public SinistroId getId() {
        return id;
    }
}
```

---

### 3. Value Object - SinistroId

```java
package com.zurich.santander.appview.modules.sinistro.domain.model;

import jakarta.persistence.Column;
import jakarta.persistence.Embeddable;
import lombok.EqualsAndHashCode;
import lombok.Getter;
import lombok.NoArgsConstructor;
import java.io.Serializable;
import java.util.UUID;

@Embeddable
@Getter
@EqualsAndHashCode
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class SinistroId implements Serializable {
    
    @Column(name = "id", nullable = false)
    private UUID value;
    
    private SinistroId(UUID value) {
        this.value = value;
    }
    
    public static SinistroId generate() {
        return new SinistroId(UUID.randomUUID());
    }
    
    public static SinistroId of(UUID value) {
        return new SinistroId(value);
    }
    
    public static SinistroId of(String value) {
        return new SinistroId(UUID.fromString(value));
    }
    
    @Override
    public String toString() {
        return value.toString();
    }
}
```

---

### 4. Repository Interface (Domain)

```java
package com.zurich.santander.appview.modules.sinistro.domain.repository;

import com.zurich.santander.appview.modules.sinistro.domain.model.Sinistro;
import com.zurich.santander.appview.modules.sinistro.domain.model.SinistroId;
import java.util.Optional;

public interface SinistroRepository {
    
    Sinistro save(Sinistro sinistro);
    
    Optional<Sinistro> findById(SinistroId id);
    
    boolean existsByNumeroSinistro(String numeroSinistro);
    
    void delete(Sinistro sinistro);
}
```

---

## Application Layer

### 1. Use Case

```java
package com.zurich.santander.appview.modules.sinistro.application.usecase;

import com.zurich.santander.appview.shared.annotation.UseCase;
import com.zurich.santander.appview.shared.infrastructure.idempotency.IdempotencyService;
import com.zurich.santander.appview.shared.infrastructure.event.DomainEventPublisher;
import com.zurich.santander.appview.modules.sinistro.domain.model.Sinistro;
import com.zurich.santander.appview.modules.sinistro.domain.model.SinistroId;
import com.zurich.santander.appview.modules.sinistro.domain.repository.SinistroRepository;
import com.zurich.santander.appview.modules.sinistro.application.command.ReceberSinistroCommand;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.transaction.annotation.Transactional;

@UseCase
@RequiredArgsConstructor
@Slf4j
public class ReceberSinistroUseCase {
    
    private final SinistroRepository repository;
    private final IdempotencyService idempotencyService;
    private final DomainEventPublisher eventPublisher;
    
    @Transactional
    public SinistroId executar(ReceberSinistroCommand command) {
        log.info("Recebendo sinistro: {}", command.numeroSinistro());
        
        // Verificar idempotência (RN-068)
        var cached = idempotencyService.buscar(command.idempotencyKey(), SinistroId.class);
        if (cached.isPresent()) {
            log.info("Sinistro já processado (idempotente): {}", cached.get());
            return cached.get();
        }
        
        // Criar sinistro (validações de domínio)
        var sinistro = Sinistro.criar(command.toData(), repository);
        
        // Persistir
        repository.save(sinistro);
        log.info("Sinistro persistido: {}", sinistro.getId());
        
        // Publicar eventos de domínio
        sinistro.getDomainEvents().forEach(eventPublisher::publish);
        sinistro.clearDomainEvents();
        
        // Cachear resultado para idempotência
        idempotencyService.cachear(command.idempotencyKey(), sinistro.getId());
        
        return sinistro.getId();
    }
}
```

---

### 2. Event Handler

```java
package com.zurich.santander.appview.modules.sinistro.application.handler;

import com.zurich.santander.appview.modules.sinistro.domain.event.SinistroRecebidoEvent;
import com.zurich.santander.appview.modules.sinistro.application.usecase.ArmazenarSinistroNoECMUseCase;
import com.zurich.santander.appview.modules.sinistro.application.command.ArmazenarSinistroCommand;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.event.EventListener;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
@Slf4j
public class SinistroRecebidoHandler {
    
    private final ArmazenarSinistroNoECMUseCase armazenarUseCase;
    
    @EventListener
    @Async
    public void handle(SinistroRecebidoEvent event) {
        log.info("Processando evento SinistroRecebidoEvent: {}", event.getSinistroId());
        
        try {
            armazenarUseCase.executar(
                new ArmazenarSinistroCommand(event.getSinistroId())
            );
        } catch (Exception e) {
            log.error("Erro ao armazenar sinistro {}", event.getSinistroId(), e);
            // TODO: Enfileirar em retry queue
        }
    }
}
```

---

## Infrastructure Layer

### 1. Controller

```java
package com.zurich.santander.appview.modules.sinistro.infrastructure.web;

import com.zurich.santander.appview.shared.annotation.RequiresPermission;
import com.zurich.santander.appview.modules.sinistro.application.usecase.ReceberSinistroUseCase;
import com.zurich.santander.appview.modules.sinistro.infrastructure.web.dto.SinistroRequest;
import com.zurich.santander.appview.modules.sinistro.infrastructure.web.dto.SinistroResponse;
import com.zurich.santander.appview.modules.sinistro.infrastructure.web.mapper.SinistroDTOMapper;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/sinistros")
@RequiredArgsConstructor
@Slf4j
public class SinistroController {
    
    private final ReceberSinistroUseCase receberSinistroUseCase;
    private final SinistroDTOMapper mapper;
    
    @PostMapping
    @RequiresPermission("sinistro:create")
    public ResponseEntity<SinistroResponse> receber(
            @RequestBody @Valid SinistroRequest request,
            @RequestHeader("Idempotency-Key") String idempotencyKey) {
        
        log.info("Recebendo requisição de sinistro: {}", request.getNumeroSinistro());
        
        var command = mapper.toCommand(request, idempotencyKey);
        var sinistroId = receberSinistroUseCase.executar(command);
        var response = mapper.toResponse(sinistroId, request);
        
        return ResponseEntity.accepted().body(response);
    }
}
```

---

### 2. Gateway

```java
package com.zurich.santander.appview.modules.sinistro.infrastructure.gateway;

import com.zurich.santander.appview.shared.annotation.Gateway;
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.retry.annotation.Retry;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.apache.chemistry.opencmis.client.api.*;
import org.apache.chemistry.opencmis.commons.PropertyIds;
import org.apache.chemistry.opencmis.commons.data.ContentStream;
import org.apache.chemistry.opencmis.commons.enums.VersioningState;
import org.springframework.stereotype.Component;
import java.io.ByteArrayInputStream;
import java.math.BigInteger;
import java.util.Map;

@Gateway
@Component
@RequiredArgsConstructor
@Slf4j
public class AlfrescoGateway {
    
    private final Session cmisSession;
    
    @CircuitBreaker(name = "alfresco", fallbackMethod = "uploadFallback")
    @Retry(name = "alfresco")
    public String uploadDocumento(DocumentoUpload upload) {
        log.info("Fazendo upload de documento: {}", upload.getNomeArquivo());
        
        try {
            // Criar estrutura de pastas
            Folder pasta = criarEstruturaPastas(upload.getCaminho());
            
            // Preparar content stream
            ContentStream contentStream = new ContentStreamImpl(
                upload.getNomeArquivo(),
                BigInteger.valueOf(upload.getTamanho()),
                upload.getMimeType(),
                new ByteArrayInputStream(upload.getConteudo())
            );
            
            // Propriedades CMIS customizadas
            Map<String, Object> properties = Map.of(
                PropertyIds.OBJECT_TYPE_ID, "zsSinistros:documentos_Sinistros",
                PropertyIds.NAME, upload.getNomeArquivo(),
                "zsSinistros:numeroSinistro", upload.getNumeroSinistro(),
                "zsSinistros:tipoDocumento", upload.getTipoDocumento(),
                "zsSinistros:apolice", upload.getApolice()
            );
            
            // Upload com versionamento MAJOR (RN-008)
            Document doc = pasta.createDocument(
                properties,
                contentStream,
                VersioningState.MAJOR
            );
            
            log.info("Documento criado com ID: {}", doc.getId());
            return doc.getId();
            
        } catch (CmisConnectionException e) {
            log.error("Erro de conexão com Alfresco", e);
            throw new AlfrescoIndisponivelException("Alfresco indisponível", e);
        } catch (CmisStorageException e) {
            log.error("Alfresco sem espaço", e);
            throw new AlfrescoSemEspacoException("Alfresco sem espaço disponível", e);
        }
    }
    
    private Folder criarEstruturaPastas(String caminho) {
        // RN-006: Estrutura /{ano}/{mes}/{numero_sinistro}/
        String[] parts = caminho.split("/");
        Folder current = (Folder) cmisSession.getObjectByPath("/prod-sinistros");
        
        for (String part : parts) {
            if (part.isBlank()) continue;
            current = getOrCreateFolder(current, part);
        }
        
        return current;
    }
    
    private Folder getOrCreateFolder(Folder parent, String folderName) {
        try {
            return (Folder) cmisSession.getObjectByPath(
                parent.getPath() + "/" + folderName
            );
        } catch (CmisObjectNotFoundException e) {
            Map<String, Object> props = Map.of(
                PropertyIds.OBJECT_TYPE_ID, "cmis:folder",
                PropertyIds.NAME, folderName
            );
            return parent.createFolder(props);
        }
    }
    
    private String uploadFallback(DocumentoUpload upload, Throwable t) {
        log.warn("Fallback ativado para upload: {}", upload.getNomeArquivo());
        // Enfileirar em retry queue
        return null;
    }
}
```

---

## Configurações

### 1. application.yml

```yaml
spring:
  application:
    name: app-view-backend
  
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:appview}
    username: ${DB_USER:appview}
    password: ${DB_PASSWORD:password}
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2
      connection-timeout: 30000
  
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        default_schema: public
        jdbc:
          batch_size: 20
        order_inserts: true
        order_updates: true
  
  flyway:
    enabled: true
    baseline-on-migrate: true
    locations: classpath:db/migration
  
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${AZURE_AD_ISSUER_URI}
          jwk-set-uri: ${AZURE_AD_JWK_SET_URI}
  
  cache:
    type: redis
    redis:
      time-to-live: 300000  # 5 minutos
  
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}

# Resilience4j
resilience4j:
  circuitbreaker:
    instances:
      alfresco:
        registerHealthIndicator: true
        slidingWindowSize: 100
        minimumNumberOfCalls: 10
        failureRateThreshold: 50
        slowCallRateThreshold: 70
        slowCallDurationThreshold: 30s
        waitDurationInOpenState: 120s
        permittedNumberOfCallsInHalfOpenState: 5
        automaticTransitionFromOpenToHalfOpenEnabled: true
      
      jarvis:
        slidingWindowSize: 100
        failureRateThreshold: 50
        waitDurationInOpenState: 60s
      
      pega:
        slidingWindowSize: 100
        failureRateThreshold: 50
        waitDurationInOpenState: 60s
  
  retry:
    instances:
      alfresco:
        maxAttempts: 3
        waitDuration: 1s
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2

# Logging
logging:
  level:
    root: INFO
    com.zurich.santander.appview: DEBUG
    org.hibernate.SQL: DEBUG
    org.springframework.security: DEBUG
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} [%X{requestId}] - %msg%n"

# Management
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  endpoint:
    health:
      show-details: always
  metrics:
    export:
      prometheus:
        enabled: true

# App Config
app:
  security:
    role-mapping:
      AppView-Admins: ADMIN
      AppView-Operadores: OPERADOR
      AppView-Juridico: JURIDICO
      AppView-Gestores: GESTOR
      AppView-Auditores: CONSULTA
  
  idempotency:
    ttl: 86400  # 24 horas
```

---

## Testes

### 1. Teste de Domínio

```java
package com.zurich.santander.appview.modules.sinistro.domain;

import com.zurich.santander.appview.modules.sinistro.domain.model.*;
import com.zurich.santander.appview.modules.sinistro.domain.repository.SinistroRepository;
import com.zurich.santander.appview.modules.sinistro.domain.exception.*;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import java.time.LocalDate;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class SinistroTest {
    
    @Mock
    private SinistroRepository repository;
    
    @Test
    void deveCriarSinistroComSucesso() {
        // Given
        var data = new SinistroData(
            "SIN123456",
            "LAUDO_MEDICO",
            "WPC",
            LocalDate.now(),
            null,
            null,
            null,
            null,
            null
        );
        
        when(repository.existsByNumeroSinistro("SIN123456")).thenReturn(false);
        
        // When
        var sinistro = Sinistro.criar(data, repository);
        
        // Then
        assertThat(sinistro).isNotNull();
        assertThat(sinistro.getNumeroSinistro()).isEqualTo("SIN123456");
        assertThat(sinistro.getStatus()).isEqualTo(StatusSinistro.RECEBIDO);
        assertThat(sinistro.getDomainEvents()).hasSize(1);
    }
    
    @Test
    void deveLancarExcecaoSeNumeroSinistroDuplicado() {
        // Given: RN-001
        when(repository.existsByNumeroSinistro("SIN123456")).thenReturn(true);
        
        var data = new SinistroData(
            "SIN123456",
            "LAUDO_MEDICO",
            "WPC",
            LocalDate.now(),
            null,
            null,
            null,
            null,
            null
        );
        
        // When/Then
        assertThatThrownBy(() -> Sinistro.criar(data, repository))
            .isInstanceOf(SinistroDuplicadoException.class)
            .hasMessageContaining("SIN123456");
    }
    
    @Test
    void deveLancarExcecaoSeDataFutura() {
        // Given: RN-002
        var data = new SinistroData(
            "SIN123456",
            "LAUDO_MEDICO",
            "WPC",
            LocalDate.now().plusDays(1),  // Data futura
            null,
            null,
            null,
            null,
            null
        );
        
        when(repository.existsByNumeroSinistro("SIN123456")).thenReturn(false);
        
        // When/Then
        assertThatThrownBy(() -> Sinistro.criar(data, repository))
            .isInstanceOf(DataInvalidaException.class);
    }
}
```

---

### 2. Teste de Use Case

```java
package com.zurich.santander.appview.modules.sinistro.application;

import com.zurich.santander.appview.modules.sinistro.application.usecase.ReceberSinistroUseCase;
import com.zurich.santander.appview.modules.sinistro.domain.model.Sinistro;
import com.zurich.santander.appview.modules.sinistro.domain.repository.SinistroRepository;
import com.zurich.santander.appview.shared.infrastructure.idempotency.IdempotencyService;
import com.zurich.santander.appview.shared.infrastructure.event.DomainEventPublisher;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class ReceberSinistroUseCaseTest {
    
    @Mock
    private SinistroRepository repository;
    
    @Mock
    private IdempotencyService idempotencyService;
    
    @Mock
    private DomainEventPublisher eventPublisher;
    
    @InjectMocks
    private ReceberSinistroUseCase useCase;
    
    @Test
    void deveReceberSinistroComSucesso() {
        // Given
        var command = mock(ReceberSinistroCommand.class);
        when(idempotencyService.buscar(any(), any())).thenReturn(Optional.empty());
        when(repository.save(any(Sinistro.class))).thenAnswer(i -> i.getArgument(0));
        
        // When
        useCase.executar(command);
        
        // Then
        verify(repository).save(any(Sinistro.class));
        verify(eventPublisher, atLeastOnce()).publish(any());
        verify(idempotencyService).cachear(any(), any());
    }
}
```

---

**Documento elaborado por:** Equipe de Arquitetura  
**Última atualização:** Janeiro de 2026
