# Documento de Arquitetura de Solução (DAS)
**App View - Sistema de Gerenciamento de Documentos de Sinistros**

**Versão:** 2.0.0  
**Data:** Janeiro de 2026  
**Status:** Em Desenvolvimento  

---

## 1. Visão Geral da Solução

### 1.1 Contexto do Negócio

O App View é um hub de integrações crítico da Zurich Santander que atua como middleware entre o sistema de gestão de conteúdo empresarial (Alfresco ECM) e múltiplos sistemas de seguros. O sistema gerencia o ciclo de vida completo de documentos relacionados a sinistros, clientes e contratos corporativos.

### 1.2 Objetivos da Nova Versão

- **Modernização Tecnológica**: Migração de Spring MVC 4.x para Spring Boot 3.x
- **Arquitetura Sustentável**: Implementação de Clean Architecture com DDD e Event-Driven
- **Escalabilidade**: Suporte a crescimento de volume sem degradação de performance
- **Manutenibilidade**: Código testável, modular e de fácil evolução
- **Observabilidade**: Monitoramento completo de integrações e processos de negócio
- **Segurança**: Conformidade com padrões de segurança corporativos e OWASP

### 1.3 Escopo da Solução

**Inclui:**
- Processamento de documentos de sinistros e clientes
- Integração com Alfresco ECM (CMIS)
- Integração com sistemas terceiros (Jarvis, Pega, CCM, Prestador, Portal PJ, BOD Seguros)
- Processamento OCR via Jarvis
- Orquestração de workflows via Pega BPM
- Comunicações automatizadas via CCM
- Portal jurídico (Portal PJ)
- Gestão de contratos corporativos (BOD Seguros)
- Dashboards e relatórios operacionais
- Sistema de autenticação e autorização (RBAC)

**Não Inclui:**
- Modificação de sistemas terceiros (Jarvis, Pega, CCM, etc.)
- Migração de dados históricos do Alfresco
- Implementação de novos processos de negócio não documentados

---

## 2. Arquitetura Proposta

### 2.1 Visão Geral Arquitetural

A nova versão do App View adota uma **arquitetura monolítica modular** com os seguintes pilares:

- **Clean Architecture**: Separação clara entre domínio, aplicação e infraestrutura
- **Domain-Driven Design (DDD)**: Modelagem rica de domínio com Aggregates, Value Objects e Domain Events
- **Event-Driven Architecture**: Comunicação entre módulos via eventos de domínio
- **Monólito Modular**: Módulos independentes com acoplamento mínimo, facilitando evolução para microserviços se necessário

### 2.2 Padrões Arquiteturais

#### 2.2.1 Clean Architecture

A aplicação segue a estrutura em camadas da Clean Architecture:

```
┌─────────────────────────────────────────────┐
│           Infrastructure Layer              │
│  (Controllers, Repositories, Gateways,      │
│   External APIs, Database, Message Queue)   │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│           Application Layer                 │
│  (Use Cases, DTOs, Event Handlers,          │
│   Application Services, Factories)          │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│              Domain Layer                   │
│  (Entities, Aggregates, Value Objects,      │
│   Domain Events, Repository Interfaces)     │
└─────────────────────────────────────────────┘
```

**Regras de Dependência:**
- Infrastructure → Application → Domain
- Domain não depende de nada
- Application depende apenas do Domain
- Infrastructure depende de Application e Domain

#### 2.2.2 Domain-Driven Design (DDD)

**Bounded Contexts Identificados:**

1. **Sinistro Context** - Gestão de documentos e processos de sinistros
2. **Cliente Context** - Gestão de documentos e dados de clientes
3. **Contrato Context** - Gestão de contratos corporativos (BOD Seguros)
4. **Comunicacao Context** - Notificações e comunicações (CCM)
5. **Pericia Context** - Gestão de perícias e prestadores
6. **Portal Context** - Acesso jurídico e consulta de propostas
7. **Documento Context** - Operações genéricas de documentos no ECM
8. **IAM Context** - Autenticação, autorização e gestão de usuários

**Padrões Táticos DDD:**
- **Aggregates**: Raízes de agregação que garantem consistência (ex: Sinistro, Cliente, Contrato)
- **Value Objects**: Objetos imutáveis (ex: CPF, CNPJ, NumeroSinistro, Endereco, DataValidade)
- **Domain Events**: Eventos de negócio (ex: SinistroRecebido, DocumentoProcessado, OCRConcluido)
- **Repositories**: Abstração de persistência
- **Factories**: Criação complexa de entidades

#### 2.2.3 Event-Driven Architecture

**Comunicação entre Use Cases via Eventos:**

```
Use Case A → Domain Event → Event Bus → Event Handler → Use Case B
```

**Exemplo de Fluxo:**
1. `ReceberDocumentoSinistroUseCase` cria evento `DocumentoSinistroRecebido`
2. Event bus despacha para handlers registrados
3. `EnviarParaJarvisHandler` captura evento e executa `EnviarDocumentoParaJarvisUseCase`
4. `EnviarDocumentoParaJarvisUseCase` cria evento `DocumentoEnviadoParaJarvis`
5. Ciclo continua até conclusão do fluxo

**Vantagens:**
- Desacoplamento total entre use cases
- Facilita adição de novos comportamentos sem modificar código existente
- Rastreabilidade completa via event log
- Possibilita processamento assíncrono futuro

#### 2.2.4 Monólito Modular

A aplicação é organizada em módulos independentes:

```
src/main/java/com/zurich/appview/
├── modules/
│   ├── sinistro/          # Bounded Context: Sinistro
│   │   ├── domain/
│   │   ├── application/
│   │   └── infrastructure/
│   ├── cliente/           # Bounded Context: Cliente
│   │   ├── domain/
│   │   ├── application/
│   │   └── infrastructure/
│   ├── contrato/          # Bounded Context: Contrato (BOD)
│   ├── comunicacao/       # Bounded Context: Comunicação (CCM)
│   ├── pericia/           # Bounded Context: Perícia
│   ├── portal/            # Bounded Context: Portal PJ
│   ├── documento/         # Bounded Context: Documento ECM
│   └── iam/               # Bounded Context: Identity & Access
├── shared/                # Código compartilhado
│   ├── domain/            # Domain building blocks
│   ├── infrastructure/    # Infraestrutura comum
│   └── application/       # Application building blocks
└── config/                # Configurações globais
```

**Princípios:**
- Cada módulo possui seu próprio domínio, casos de uso e infraestrutura
- Comunicação entre módulos exclusivamente via eventos
- Nenhum acesso direto a entidades de outros módulos
- Possibilidade de extração para microserviço futuro sem refatoração massiva

---

## 3. Decisões Arquiteturais (ADRs)

### ADR-001: Manter Monólito ao invés de Microserviços

**Status:** Aceito

**Contexto:**
- Time de desenvolvimento pequeno (4-6 desenvolvedores)
- Complexidade operacional de microserviços não se justifica
- Deploy e debugging mais complexos em arquitetura distribuída
- Sistema crítico que requer alta disponibilidade

**Decisão:**
Adotar monólito modular com separação clara de bounded contexts, permitindo evolução para microserviços no futuro se necessário.

**Consequências:**
- ✅ Deploy simplificado
- ✅ Debugging mais fácil
- ✅ Transações ACID nativas
- ✅ Menor complexidade operacional
- ⚠️ Escalabilidade vertical limitada
- ⚠️ Requer disciplina para manter módulos desacoplados

### ADR-002: Event-Driven entre Use Cases

**Status:** Aceito

**Contexto:**
- Necessidade de desacoplamento entre fluxos de negócio
- Rastreabilidade de operações críticas
- Facilitar evolução e adição de novos comportamentos

**Decisão:**
Comunicação entre use cases via Domain Events ao invés de chamadas diretas a services.

**Consequências:**
- ✅ Baixo acoplamento entre casos de uso
- ✅ Rastreabilidade via event log
- ✅ Facilita testes unitários
- ✅ Open/Closed Principle
- ⚠️ Curva de aprendizado para desenvolvedores
- ⚠️ Debugging pode ser menos intuitivo inicialmente

### ADR-003: Circuit Breaker Implementado Internamente

**Status:** Aceito

**Contexto:**
- Sistemas terceiros (Jarvis, Pega, CCM) são legados e podem ter instabilidades
- Necessidade de proteção contra sobrecarga de sistemas externos
- Evitar cascata de falhas

**Decisão:**
Implementar Circuit Breaker pattern internamente usando Resilience4j ao invés de API Gateway externo.

**Consequências:**
- ✅ Proteção contra falhas em cascata
- ✅ Não sobrecarrega sistemas legados com retries
- ✅ Controle fino sobre políticas de retry e timeout
- ✅ Menor complexidade de infraestrutura
- ⚠️ Circuit breaker por instância (não distribuído)

### ADR-004: Uso de Filas Azure Service Bus

**Status:** Aceito

**Contexto:**
- Processamento OCR do Jarvis é demorado (5-30 segundos)
- Processamento de lotes pode bloquear requests HTTP
- Necessidade de notificações assíncronas

**Decisão:**
Utilizar Azure Service Bus para processamento assíncrono de tarefas de longa duração.

**Consequências:**
- ✅ Melhor user experience (resposta imediata)
- ✅ Processamento resiliente com retry automático
- ✅ Escalabilidade de processamento
- ✅ Integração nativa com Azure
- ⚠️ Eventual consistency
- ⚠️ Necessidade de sistema de notificações

### ADR-005: Idempotência Obrigatória em Todos os Endpoints

**Status:** Aceito

**Contexto:**
- Sistema crítico que gerencia documentos financeiros e jurídicos
- Duplicação de sinistros pode causar problemas graves em sistemas terceiros
- Retries e falhas de rede podem causar duplicação

**Decisão:**
Todos os endpoints de escrita devem ser idempotentes usando idempotency keys.

**Consequências:**
- ✅ Segurança contra duplicação acidental
- ✅ Retries seguros
- ✅ Conformidade com requisitos de negócio
- ⚠️ Necessidade de armazenamento de idempotency keys
- ⚠️ Validação adicional em cada request

### ADR-006: Spring Boot 3.x com Java 21

**Status:** Aceito

**Contexto:**
- Sistema atual usa Spring MVC 4.x com Java 8
- Necessidade de suporte de longo prazo
- Performance e recursos modernos de Java

**Decisão:**
Migrar para Spring Boot 3.3+ com Java 21 LTS.

**Consequências:**
- ✅ Suporte até setembro de 2029 (Java 21 LTS)
- ✅ Virtual Threads para melhor concorrência
- ✅ Pattern Matching e Records
- ✅ Spring Boot Actuator out-of-the-box
- ⚠️ Reescrita completa do código base
- ⚠️ Migração de dependências

### ADR-007: PostgreSQL como Banco de Dados Principal

**Status:** Aceito

**Contexto:**
- Sistema atual já usa PostgreSQL
- Necessidade de suporte a JSON nativo
- Transações ACID críticas

**Decisão:**
Manter PostgreSQL (versão 15+) como banco de dados principal.

**Consequências:**
- ✅ Continuidade operacional
- ✅ JSON/JSONB nativo para event sourcing
- ✅ Transações robustas
- ✅ Time já possui expertise

### ADR-008: Feature-First no Frontend

**Status:** Aceito

**Contexto:**
- Frontend atual em JSP é difícil de manter
- Necessidade de UI moderna e responsiva
- Times podem trabalhar em features independentes

**Decisão:**
Novo frontend em React com arquitetura Feature-First.

**Consequências:**
- ✅ Modularidade e escalabilidade
- ✅ Times independentes
- ✅ Reuso de componentes
- ✅ Melhor testabilidade
- ⚠️ Reescrita completa do frontend

---

## 4. Stack Tecnológico

### 4.1 Backend

#### Core Framework
- **Java 21 LTS** (Oracle JDK ou OpenJDK)
- **Spring Boot 3.3.x**
  - Spring Web (REST APIs)
  - Spring Data JPA
  - Spring Security 6.x
  - Spring Events
  - Spring Actuator
  - Spring Validation

#### Persistência
- **PostgreSQL 15+**
- **Hibernate 6.x** (via Spring Data JPA)
- **Flyway** (Database Migrations)
- **HikariCP** (Connection Pooling)

#### Integrações
- **Apache Chemistry OpenCMIS 1.1+** (Alfresco)
- **Spring Cloud OpenFeign** (HTTP Clients declarativos)
- **Resilience4j** (Circuit Breaker, Retry, Rate Limiter)
- **Azure SDK for Java** (Service Bus, Storage, Key Vault)

#### Event-Driven
- **Spring Events** (Eventos síncronos internos)
- **Azure Service Bus** (Filas assíncronas)
- **Jackson** (Serialização de eventos)

#### Observabilidade
- **Micrometer** (Métricas)
- **Azure Application Insights** (APM)
- **SLF4J + Logback** (Logging estruturado)
- **Spring Boot Actuator** (Health checks, metrics)

#### Testes
- **JUnit 5** (Jupiter)
- **Mockito** (Mocking)
- **TestContainers** (Testes de integração com PostgreSQL e Azure Service Bus)
- **REST Assured** (Testes de API)
- **ArchUnit** (Testes de arquitetura)

#### Documentação
- **Springdoc OpenAPI 3** (Swagger UI)
- **Asciidoctor** (Documentação técnica)

#### Utilitários
- **MapStruct** (Mapeamento de objetos)
- **Lombok** (Redução de boilerplate)
- **Vavr** (Functional programming utilities)
- **Apache Commons Lang3**
- **Guava** (Google Core Libraries)

### 4.2 Frontend

#### Core Framework
- **React 18.x** (TypeScript)
- **Vite** (Build tool)
- **React Router 6.x** (Roteamento)

#### State Management
- **TanStack Query (React Query)** (Server state)
- **Zustand** (Client state)

#### UI/UX
- **Tailwind CSS** (Estilização)
- **shadcn/ui** (Componentes base)
- **Radix UI** (Componentes headless)
- **Lucide React** (Ícones)

#### Forms & Validation
- **React Hook Form** (Gestão de formulários)
- **Zod** (Schema validation)

#### HTTP Client
- **Axios** (com interceptors para autenticação)

#### Testes
- **Vitest** (Unit tests)
- **Playwright** (E2E tests)
- **Testing Library** (Component tests)

#### Build & Quality
- **TypeScript 5.x**
- **ESLint** (Linting)
- **Prettier** (Code formatting)
- **Husky** (Git hooks)

### 4.3 DevOps & Infraestrutura

#### CI/CD
- **GitHub Actions** (Pipeline)
- **Azure DevOps** (Alternativa corporativa)

#### Containerização
- **Docker** (Containerização)
- **Azure Container Registry** (Registro de imagens)

#### Cloud (Azure)
- **Azure App Service** (Hosting backend)
- **Azure Static Web Apps** (Hosting frontend)
- **Azure Service Bus** (Message Queue)
- **Azure Blob Storage** (Arquivos temporários)
- **Azure Key Vault** (Secrets)
- **Azure Application Insights** (APM)
- **Azure Database for PostgreSQL** (Banco de dados)

#### Monitoramento
- **Azure Monitor**
- **Azure Application Insights**
- **Azure Log Analytics**

---

## 5. Estrutura de Módulos (Backend)

### 5.1 Módulo Sinistro

**Responsabilidades:**
- Receber documentos de sinistros
- Armazenar no Alfresco
- Enviar para processamento OCR (Jarvis)
- Rotear para workflow (Pega)
- Registrar auditoria

**Estrutura:**
```
sinistro/
├── domain/
│   ├── entities/
│   │   └── Sinistro.java              # Aggregate Root
│   ├── valueobjects/
│   │   ├── NumeroSinistro.java
│   │   ├── TipoDocumento.java
│   │   ├── Canal.java
│   │   └── LossInfo.java
│   ├── events/
│   │   ├── SinistroRecebido.java
│   │   ├── SinistroArmazenado.java
│   │   ├── SinistroEnviadoParaOCR.java
│   │   └── SinistroProcessado.java
│   ├── enums/
│   │   └── StatusSinistro.java
│   ├── repositories/
│   │   └── SinistroRepository.java    # Interface
│   └── exceptions/
│       ├── SinistroInvalidoException.java
│       └── SinistroDuplicadoException.java
├── application/
│   ├── usecases/
│   │   ├── ReceberDocumentoSinistroUseCase.java
│   │   ├── ArmazenarSinistroNoECMUseCase.java
│   │   ├── EnviarSinistroParaJarvisUseCase.java
│   │   ├── ProcessarCallbackJarvisUseCase.java
│   │   └── RotearSinistroParaPegaUseCase.java
│   ├── dtos/
│   │   ├── ReceberSinistroRequest.java
│   │   ├── ReceberSinistroResponse.java
│   │   └── SinistroDTO.java
│   ├── events/
│   │   ├── SinistroRecebidoHandler.java
│   │   ├── SinistroArmazenadoHandler.java
│   │   └── OCRConcluidoHandler.java
│   └── gateways/
│       ├── JarvisGateway.java         # Interface
│       └── PegaGateway.java           # Interface
└── infrastructure/
    ├── controllers/
    │   └── SinistroController.java
    ├── repositories/
    │   └── PostgreSinistroRepository.java
    ├── gateways/
    │   ├── JarvisHttpGateway.java
    │   └── PegaHttpGateway.java
    ├── persistence/
    │   └── SinistroJpaEntity.java
    └── config/
        └── SinistroModuleConfig.java
```

### 5.2 Módulo Cliente

**Responsabilidades:**
- Receber documentos de clientes (RG, CPF, CNH, etc.)
- Armazenar no Alfresco
- Enviar para validação OCR
- Atualizar status de validação

**Estrutura Similar ao Módulo Sinistro**

### 5.3 Módulo Documento (Shared)

**Responsabilidades:**
- Operações genéricas de documentos no ECM
- Upload, download, versionamento
- Geração de links temporários
- Gestão de metadados Alfresco

**Entidades:**
- `Documento` (Aggregate Root)
- `ConteudoDocumento` (Value Object)
- `MetadadosDocumento` (Value Object)

### 5.4 Módulo IAM (Identity & Access Management)

**Responsabilidades:**
- Autenticação de usuários
- Autorização (RBAC)
- Gestão de perfis e permissões
- Integração com LDAP/AD (futuro)

**Entidades:**
- `Usuario` (Aggregate Root)
- `Perfil` (Entity)
- `Permissao` (Value Object)

---

## 6. Padrões e Convenções

### 6.1 Camada de Domínio

**Entidades:**
- Sempre estendem `AggregateRoot` ou `Entity`
- Construtores privados, criação via factory methods
- Métodos de negócio que emitem eventos
- Validações de invariantes no construtor e métodos

**Exemplo:**
```java
public class Sinistro extends AggregateRoot {
    private final SinistroId id;
    private NumeroSinistro numero;
    private TipoDocumento tipo;
    private StatusSinistro status;
    
    private Sinistro(SinistroId id, NumeroSinistro numero, TipoDocumento tipo) {
        this.id = id;
        this.numero = numero;
        this.tipo = tipo;
        this.status = StatusSinistro.RECEBIDO;
    }
    
    public static Sinistro criar(NumeroSinistro numero, TipoDocumento tipo) {
        Sinistro sinistro = new Sinistro(SinistroId.gerar(), numero, tipo);
        sinistro.addDomainEvent(new SinistroRecebido(sinistro.id, sinistro.numero));
        return sinistro;
    }
    
    public void marcarComoArmazenado(AlfrescoDocumentId documentId) {
        this.status = StatusSinistro.ARMAZENADO;
        this.addDomainEvent(new SinistroArmazenado(this.id, documentId));
    }
}
```

**Value Objects:**
- Imutáveis
- Validação no construtor
- Equality por valor

**Exemplo:**
```java
public record NumeroSinistro(String valor) {
    public NumeroSinistro {
        if (valor == null || valor.isBlank()) {
            throw new IllegalArgumentException("Número do sinistro não pode ser vazio");
        }
        if (!valor.matches("^[A-Z0-9]{6,20}$")) {
            throw new IllegalArgumentException("Formato inválido de número de sinistro");
        }
    }
}
```

### 6.2 Camada de Aplicação

**Use Cases:**
- Um use case = uma responsabilidade
- Input via DTO de request
- Output via DTO de response ou entidade
- Transacional (@Transactional)
- Emite eventos via ApplicationEventPublisher

**Exemplo:**
```java
@UseCase
@Transactional
public class ReceberDocumentoSinistroUseCase {
    
    private final SinistroRepository repository;
    private final ApplicationEventPublisher eventPublisher;
    
    public ReceberSinistroResponse executar(ReceberSinistroRequest request) {
        // 1. Validar idempotência
        validarIdempotencia(request.idempotencyKey());
        
        // 2. Criar entidade de domínio
        Sinistro sinistro = Sinistro.criar(
            new NumeroSinistro(request.numeroSinistro()),
            new TipoDocumento(request.tipoDocumento())
        );
        
        // 3. Salvar
        repository.salvar(sinistro);
        
        // 4. Publicar eventos
        sinistro.getDomainEvents().forEach(eventPublisher::publishEvent);
        
        return new ReceberSinistroResponse(sinistro.getId().valor());
    }
}
```

**Event Handlers:**
- Anotados com `@EventListener`
- Stateless
- Podem chamar outros use cases
- Tratamento de erros robusto

### 6.3 Camada de Infraestrutura

**Controllers:**
- Apenas mapeamento de HTTP para use cases
- Validação de entrada via Bean Validation
- Tratamento de exceções via @ControllerAdvice
- Documentação via @Operation (OpenAPI)

**Repositories:**
- Implementam interfaces do domínio
- Mapeamento entre entidades de domínio e JPA entities
- Uso de Specifications para queries complexas

**Gateways:**
- Implementam interfaces da aplicação
- Circuit breaker e retry via @CircuitBreaker
- Logging de requests/responses
- Conversão de exceções externas para domain exceptions

---

## 7. Estratégia de Migração

### 7.1 Abordagem: Strangler Fig Pattern

A migração do sistema legado será feita gradualmente usando o padrão Strangler Fig:

**Fase 1: Novo Sistema Paralelo (Meses 1-2)**
- Deploy do novo sistema em ambiente separado
- Roteamento de tráfego via proxy (Azure Application Gateway)
- 100% do tráfego ainda vai para sistema legado
- Testes E2E no novo sistema

**Fase 2: Migração Gradual de Módulos (Meses 3-5)**
- Migração por bounded context (um de cada vez)
- Ordem sugerida:
  1. Portal PJ (menor risco, menor volume)
  2. BOD Seguros (volume médio, fluxo simples)
  3. Comunicação/CCM (crítico mas isolado)
  4. Perícia/Prestador (médio volume)
  5. Cliente (alto volume)
  6. Sinistro (crítico, alto volume - por último)

**Fase 3: Cutover Completo (Mês 6)**
- Validação de paridade de funcionalidades
- 100% do tráfego no novo sistema
- Sistema legado em standby por 1 mês
- Descomissionamento do legado

### 7.2 Estratégia de Dados

**Banco de Dados:**
- Manter PostgreSQL atual
- Criar novos schemas para nova estrutura
- Views de compatibilidade para dashboards legados
- Flyway migrations versionadas

**Alfresco:**
- Sem migração necessária
- Nova aplicação usa mesma estrutura de pastas
- Compatibilidade total com CMIS

### 7.3 Riscos e Mitigações

| Risco | Impacto | Probabilidade | Mitigação |
|-------|---------|---------------|-----------|
| Incompatibilidade de APIs terceiros | Alto | Médio | Testes de contrato, validação em UAT antes de PRD |
| Perda de dados na migração | Crítico | Baixo | Nenhuma migração de dados, apenas novo código |
| Downtime durante cutover | Alto | Médio | Blue/Green deployment, rollback instantâneo |
| Performance inferior ao legado | Médio | Baixo | Load tests antes do cutover |
| Bugs não detectados | Alto | Médio | Cobertura de testes >80%, E2E completos |

---

## 8. Requisitos Não-Funcionais

### 8.1 Performance

- **Latência de API**: p95 < 500ms (endpoints síncronos)
- **Throughput**: 100 requisições/segundo por instância
- **Processamento OCR**: Assíncrono, SLA de 30 segundos
- **Queries de Dashboard**: < 2 segundos

### 8.2 Disponibilidade

- **SLA**: 99.5% (excluindo janelas de manutenção)
- **RTO (Recovery Time Objective)**: 15 minutos
- **RPO (Recovery Point Objective)**: 5 minutos

### 8.3 Segurança

- **Autenticação**: OAuth 2.0 / OpenID Connect (Azure AD)
- **Autorização**: RBAC granular
- **Criptografia**: TLS 1.3 em trânsito, AES-256 em repouso
- **Secrets**: Azure Key Vault
- **Rate Limiting**: 100 req/min por usuário, 1000 req/min por IP
- **Auditoria**: Todas as operações de escrita logadas

### 8.4 Escalabilidade

- **Horizontal**: Suporte a múltiplas instâncias (stateless)
- **Vertical**: Otimizado para 4 vCPU / 8GB RAM por instância
- **Filas**: Processamento assíncrono via Azure Service Bus
- **Cache**: Redis para sessões e queries frequentes (futuro)

### 8.5 Manutenibilidade

- **Cobertura de Testes**: Mínimo 80%
  - Unitários: 85%+
  - Integração: 70%+
  - E2E: 50 cenários críticos
- **Documentação**: OpenAPI 3.0, ADRs, Runbooks
- **Code Quality**: SonarQube (A rating, 0 critical issues)

### 8.6 Observabilidade

- **Logs Estruturados**: JSON format, correlation IDs
- **Métricas**: Micrometer → Azure Application Insights
- **Tracing Distribuído**: Correlation IDs em todas as requisições
- **Health Checks**: Liveness e Readiness probes
- **Dashboards**: Grafana (ou Azure Monitor Workbooks)

---

## 9. Integrações - Visão Geral

### 9.1 Mapa de Integrações

```
┌──────────────────────────────────────────────────────────┐
│                      App View                            │
│                   (Hub de Integrações)                   │
└──────────────────────────────────────────────────────────┘
         │        │        │        │        │        │
         ▼        ▼        ▼        ▼        ▼        ▼
    ┌────────┬────────┬────────┬─────────┬────────┬────────┐
    │Alfresco│ Jarvis │  Pega  │   CCM   │Prestad.│Portal PJ│
    │  ECM   │  OCR   │  BPM   │ Comunic.│ Perícia│Jurídico│
    └────────┴────────┴────────┴─────────┴────────┴────────┘
```

### 9.2 Características por Integração

| Sistema     | Protocolo | Sync/Async       | Circuit Breaker | Retry           | Timeout |
| ----------- | --------- | ---------------- | --------------- | --------------- | ------- |
| Alfresco    | CMIS/HTTP | Sync             | Sim             | 3x, exp backoff | 30s     |
| Jarvis      | REST      | Async (callback) | Sim             | 2x, fixed delay | 60s     |
| Pega        | REST      | Sync             | Sim             | 3x, exp backoff | 15s     |
| CCM         | REST      | Sync             | Sim             | 3x, exp backoff | 10s     |
| Prestador   | REST      | Sync             | Sim             | 2x, fixed delay | 10s     |
| Portal PJ   | CMIS      | Sync             | Sim             | 3x, exp backoff | 20s     |
| BOD Seguros | CMIS      | Sync             | Sim             | 3x, exp backoff | 30s     |

**Políticas de Circuit Breaker (Resilience4j):**
- **Failure Rate Threshold**: 50%
- **Slow Call Threshold**: 70%
- **Wait Duration in Open State**: 60 segundos
- **Sliding Window Size**: 100 chamadas
- **Minimum Number of Calls**: 10

---

## 10. Segurança

### 10.1 Autenticação

**Primária:** OAuth 2.0 / OpenID Connect via Azure AD

**Fluxo:**
1. Usuário acessa aplicação
2. Redirecionado para Azure AD
3. Autenticação SSO
4. Recebe JWT token
5. Token validado em cada request

**Fallback:** Basic Auth para APIs de integração (compatibilidade com sistemas legados)

### 10.2 Autorização (RBAC)

**Perfis:**
- `ADMIN` - Acesso total
- `OPERADOR` - Operações de sinistro e cliente
- `PRESTADOR` - Acesso limitado a perícias
- `JURIDICO` - Acesso ao Portal PJ
- `CONSULTA` - Apenas leitura

**Permissões Granulares:**
- Por módulo (ex: `sinistro:create`, `cliente:read`)
- Validadas via `@PreAuthorize` nos controllers
- Auditadas em todas as operações

### 10.3 Proteções

- **Rate Limiting**: Bucket4j (100 req/min por usuário)
- **Input Validation**: Bean Validation em todos os DTOs
- **SQL Injection**: Hibernate/JPA parametrizado
- **XSS**: Sanitização de inputs, Content Security Policy
- **CSRF**: Tokens em formulários (desabilitado em APIs REST stateless)
- **CORS**: Whitelist de origens permitidas

### 10.4 Secrets Management

- **Azure Key Vault** para:
  - Credenciais de banco de dados
  - API Keys (Jarvis, Pega, CCM)
  - Chaves de criptografia
  - Certificados
- Rotação automática de secrets
- Sem secrets em código ou variáveis de ambiente

---

## 11. Processamento Assíncrono

### 11.1 Filas Azure Service Bus

**Filas:**

1. **jarvis-ocr-queue**
   - Processamento de documentos via Jarvis
   - Dead Letter Queue após 3 tentativas
   - TTL: 1 hora

2. **pega-routing-queue**
   - Roteamento de sinistros/clientes para Pega
   - Dead Letter Queue após 5 tentativas
   - TTL: 2 horas

3. **notification-queue**
   - Envio de notificações (email, SMS)
   - Dead Letter Queue após 3 tentativas
   - TTL: 24 horas

### 11.2 Processamento de Mensagens

- **Listeners**: Spring Cloud Azure Service Bus
- **Concorrência**: 5 threads por fila
- **Idempotência**: Message ID usado como idempotency key
- **Retry**: Exponential backoff (1s, 2s, 4s)
- **Dead Letter**: Processamento manual via dashboard

---

## 12. Idempotência

### 12.1 Estratégia

**Todos os endpoints de escrita aceitam header:**
```
Idempotency-Key: <UUID>
```

**Armazenamento:**
- Tabela `tb_idempotency` com TTL de 24 horas
- Chave: `idempotency_key` + `endpoint`
- Valor: Response original (JSON)

**Comportamento:**
- Se key já existe: retorna response cached (HTTP 200)
- Se key nova: processa normalmente e armazena response
- Cleanup automático via scheduled job

**Exemplo:**
```http
POST /api/sinistros
Idempotency-Key: 123e4567-e89b-12d3-a456-426614174000
Content-Type: application/json

{
  "numeroSinistro": "SIN123456",
  ...
}
```

---

## 13. Observabilidade

### 13.1 Logging

**Framework:** SLF4J + Logback

**Formato:** JSON estruturado

**Campos Padrão:**
```json
{
  "timestamp": "2026-01-27T21:00:00Z",
  "level": "INFO",
  "logger": "com.zurich.appview.sinistro",
  "message": "Sinistro recebido",
  "correlationId": "abc-123-def",
  "userId": "user@zurich.com",
  "sinistroId": "SIN123456",
  "trace": {
    "traceId": "xyz",
    "spanId": "span1"
  }
}
```

**Níveis de Log:**
- **ERROR**: Falhas que impedem operação
- **WARN**: Situações anormais mas recuperáveis
- **INFO**: Eventos de negócio importantes
- **DEBUG**: Debugging (desabilitado em PRD)

### 13.2 Métricas

**Via Micrometer:**
- Counters: Total de requisições por endpoint
- Gauges: Documentos em processamento
- Timers: Latência de use cases e integrações
- Distribution Summaries: Tamanho de documentos

**Métricas de Negócio:**
- `sinistros.recebidos.total`
- `sinistros.processados.total`
- `jarvis.ocr.duration`
- `pega.integration.success.rate`

### 13.3 Tracing

- **Correlation ID** em toda a stack
- Propagação via HTTP headers (`X-Correlation-ID`)
- Integração com Azure Application Insights
- Rastreamento de eventos de domínio

### 13.4 Health Checks

**Liveness:** `/actuator/health/liveness`
- Verifica se aplicação está rodando

**Readiness:** `/actuator/health/readiness`
- Verifica dependências:
  - PostgreSQL
  - Alfresco (CMIS)
  - Azure Service Bus
  - Jarvis API
  - Pega API

---

## 14. Diagramas C4

### 14.1 Nível 1 - Contexto do Sistema

```
┌─────────────────────────────────────────────────────────┐
│                    Sistemas Externos                     │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │   WPC    │  │  Portal  │  │ Sistemas │             │
│  │(Sistema  │  │    PJ    │  │ Internos │             │
│  │Originador│  │(Jurídico)│  │          │             │
│  └─────┬────┘  └────┬─────┘  └────┬─────┘             │
│        │            │              │                    │
└────────┼────────────┼──────────────┼────────────────────┘
         │            │              │
         ▼            ▼              ▼
┌─────────────────────────────────────────────────────────┐
│                      APP VIEW                            │
│        (Hub de Integrações de Documentos)                │
│                                                          │
│  • Recepção de documentos                               │
│  • Processamento OCR                                    │
│  • Orquestração de workflows                           │
│  • Gestão de conteúdo empresarial                      │
└─────────────────────────────────────────────────────────┘
         │            │              │
         ▼            ▼              ▼
┌─────────────────────────────────────────────────────────┐
│               Sistemas de Backend                        │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │ Alfresco │  │  Jarvis  │  │   Pega   │             │
│  │   ECM    │  │   OCR    │  │   BPM    │             │
│  └──────────┘  └──────────┘  └──────────┘             │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │   CCM    │  │Prestador │  │   BOD    │             │
│  │ Comunic. │  │  Perícia │  │ Seguros  │             │
│  └──────────┘  └──────────┘  └──────────┘             │
└─────────────────────────────────────────────────────────┘
```

### 14.2 Nível 2 - Container

```
┌───────────────────────────────────────────────────────────────┐
│                         APP VIEW                              │
│                                                               │
│  ┌──────────────────────┐      ┌──────────────────────┐     │
│  │   React Frontend     │      │   Spring Boot API    │     │
│  │   (Static Web App)   │◄────►│   (App Service)      │     │
│  │                      │      │                      │     │
│  │  • Feature modules   │      │  • REST Controllers  │     │
│  │  • React Query       │      │  • Use Cases         │     │
│  │  • Zustand           │      │  • Domain Model      │     │
│  └──────────────────────┘      │  • Repositories      │     │
│                                │  • Event Bus         │     │
│                                └──────────┬───────────┘     │
│                                           │                  │
│  ┌──────────────────────┐                │                  │
│  │  Azure Service Bus   │◄───────────────┘                  │
│  │                      │                                   │
│  │  • jarvis-ocr-queue  │                                   │
│  │  • pega-routing-queue│                                   │
│  │  • notification-queue│                                   │
│  └──────────────────────┘                                   │
└───────────────────────────────────────────────────────────────┘
                    │                │
                    ▼                ▼
┌────────────────────────────────────────────────────────┐
│              External Systems & Storage                 │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐    │
│  │PostgreSQL│  │ Alfresco │  │  Azure Key Vault │    │
│  │ Database │  │   ECM    │  │    (Secrets)     │    │
│  └──────────┘  └──────────┘  └──────────────────┘    │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│  │  Jarvis  │  │   Pega   │  │   CCM    │            │
│  │   API    │  │   API    │  │   API    │            │
│  └──────────┘  └──────────┘  └──────────┘            │
└────────────────────────────────────────────────────────┘
```

---

## 15. Considerações Finais

### 15.1 Pontos de Atenção

1. **Idempotência é Crítica**: Sistema lida com operações financeiras e jurídicas
2. **Circuit Breaker Obrigatório**: Sistemas terceiros são legados e instáveis
3. **Event Sourcing**: Considerar para auditoria completa (futuro)
4. **Cache Estratégico**: Dashboards podem se beneficiar de cache Redis
5. **Backup e DR**: Estratégia de disaster recovery deve ser documentada

### 15.2 Roadmap Técnico

**Q1/2026:**
- Desenvolvimento do novo sistema
- Testes E2E completos
- Deploy em DEV e UAT

**Q2/2026:**
- Migração gradual (Strangler Fig)
- Validação em produção com tráfego parcial

**Q3/2026:**
- Cutover completo
- Descomissionamento do sistema legado

**Q4/2026:**
- Otimizações de performance
- Implementação de features avançadas (cache, event sourcing)

### 15.3 Próximos Passos

1. ✅ Aprovação deste DAS
2. ⬜ Refinamento de casos de uso (30+)
3. ⬜ Modelagem detalhada de domínio
4. ⬜ Especificação de APIs (OpenAPI 3.0)
5. ⬜ Setup de ambiente de desenvolvimento
6. ⬜ Implementação de módulos (ordem: Documento → Sinistro → Cliente → ...)

---

**Documento elaborado por:** Equipe de Arquitetura  
**Aprovado por:** [Pendente]  
**Data de Aprovação:** [Pendente]
