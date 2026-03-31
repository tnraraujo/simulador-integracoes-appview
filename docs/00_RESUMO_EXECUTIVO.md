# Resumo Executivo - App View v2.0
**Documentação Técnica Completa**

**Versão:** 2.0.0  
**Data:** Janeiro de 2026  
**Projeto:** App View v2.0 - Hub de Integração Zurich Santander  

---

## 📋 Visão Geral do Projeto

### Contexto de Negócio
O **App View** é um sistema crítico que atua como hub de integração para gerenciamento do ciclo de vida completo de documentos de sinistros e clientes da Zurich Santander. O sistema processa milhares de documentos mensalmente, orquestrando integrações com 8 sistemas externos.

### Objetivo da Versão 2.0
Modernizar a arquitetura legada (Spring MVC 4.x) para uma solução robusta, escalável e manutenível baseada em:
- **Clean Architecture + DDD**
- **Event-Driven Architecture**
- **Modular Monolith**
- **Spring Boot 3.3 + Java 21**
- **Azure Cloud-Native**

---

## 📊 Números do Projeto

| Métrica | Valor |
|---------|-------|
| **Total de Documentos** | 9 documentos técnicos |
| **Total de Linhas Escritas** | ~9,500 linhas |
| **Casos de Uso Especificados** | 37 |
| **Regras de Negócio Catalogadas** | 82 |
| **Diagramas de Sequência** | 14 |
| **Bounded Contexts (DDD)** | 8 módulos |
| **Sistemas Externos Integrados** | 8 |
| **Perfis RBAC** | 5 |
| **Permissões Granulares** | 47 |
| **Endpoints API** | 60+ |

---

## 📚 Índice de Documentação Entregue

### 1. Arquitetura e Design
- **01_DAS_Documento_Arquitetura_Solucao.md** (1,120 linhas)
  - Visão arquitetural completa
  - 8 ADRs (Architectural Decision Records)
  - Padrões e convenções
  - Estratégia de migração (Strangler Fig)
  - Requisitos não-funcionais

- **08_Diagramas_C4.md** (687 linhas)
  - Diagrama de Contexto (Nível 1)
  - Diagrama de Contêineres (Nível 2)
  - Diagrama de Componentes Backend (Nível 3)
  - Diagrama de Componentes Frontend (Nível 3)

### 2. Integrações e Fluxos
- **02_Integracoes_Sistemas_Externos.md** (1,297 linhas)
  - 8 sistemas: Alfresco, Jarvis, Pega, CCM, Prestador, Portal PJ, BOD, PostgreSQL
  - Detalhes de conexão e autenticação
  - Payloads completos (request/response)
  - Políticas de resiliência (Circuit Breaker, Retry, Timeout)
  - Estratégias de disaster recovery

- **03_Diagramas_Sequencia.md** (1,114 linhas)
  - 14 fluxos detalhados em Mermaid
  - Cobertura de cenários críticos: sinistro, cliente, OCR, Pega, perícia, autenticação
  - Tratamento de erros e retry

### 3. Modelo de Dados
- **04_Diagrama_Banco_Dados_ERD.md** (1,051 linhas)
  - ERD completo com todas as entidades
  - 4 schemas: public, audit, iam, integration
  - DDL completo com indexes, constraints, checks
  - Materialized views para performance
  - Estratégia de particionamento
  - Migrações Flyway

### 4. Funcionalidades e Regras
- **05_Casos_de_Uso.md** (967 linhas)
  - 37 casos de uso detalhados
  - Distribuídos em 9 módulos
  - Atores, pré/pós-condições, fluxos principal/alternativo/exceção
  - Payloads de request/response
  - Matriz de rastreabilidade

- **07_Catalogo_Regras_Negocio.md** (710 linhas)
  - 82 regras de negócio catalogadas
  - Organizadas em 7 categorias
  - Implementações em código
  - Criticidade (35 altas, 30 médias, 17 baixas)
  - Rastreabilidade com casos de uso

### 5. Segurança e Controle de Acesso
- **06_Matriz_RBAC.md** (563 linhas)
  - 5 perfis: ADMIN, OPERADOR, JURIDICO, GESTOR, CONSULTA
  - 47 permissões granulares
  - Mapeamento Azure AD → Perfis
  - Implementação Spring Security
  - Row-level security
  - Auditoria de acessos
  - Testes de autorização

### 6. Guias de Implementação
- **09_Guia_Implementacao_Backend.md** (962 linhas)
  - Estrutura de diretórios completa
  - Implementação Clean Architecture
  - Exemplos de código (Domain, Application, Infrastructure)
  - Aggregate Root, Entities, Value Objects
  - Use Cases, Event Handlers
  - Controllers, Gateways
  - Configurações (application.yml, Resilience4j)
  - Testes (domínio, use case, controller)

---

## 🏗️ Arquitetura Técnica

### Stack Tecnológica

#### Backend
- **Runtime:** Java 21 LTS
- **Framework:** Spring Boot 3.3+
- **Database:** PostgreSQL 15+
- **ORM:** Hibernate 6.x
- **Migrations:** Flyway
- **Messaging:** Azure Service Bus
- **Cache:** Redis
- **Resilience:** Resilience4j (Circuit Breaker, Retry, Rate Limiter)
- **Security:** Spring Security + OAuth 2.0 (Azure AD)
- **Testing:** JUnit 5, Mockito, TestContainers, REST Assured, ArchUnit

#### Frontend
- **Framework:** React 18 + TypeScript
- **Build Tool:** Vite
- **Routing:** React Router 6
- **State:** Zustand + TanStack Query
- **UI:** Tailwind CSS + shadcn/ui
- **Forms:** React Hook Form + Zod
- **Testing:** Vitest, Playwright

#### Infraestrutura (Azure)
- Azure App Service (Backend)
- Azure Static Web Apps (Frontend)
- Azure Database for PostgreSQL Flexible Server
- Azure Cache for Redis
- Azure Service Bus
- Azure Blob Storage
- Azure Key Vault
- Azure Application Insights

### Padrões Arquiteturais

**1. Clean Architecture**
```
Domain Layer (Entidades, Agregados, Value Objects)
   ↓ depende
Application Layer (Use Cases, Event Handlers)
   ↓ depende
Infrastructure Layer (Controllers, Gateways, Repositories)
```

**2. Domain-Driven Design (DDD)**
- 8 Bounded Contexts: Sinistro, Cliente, Contrato, Comunicação, Perícia, Portal, Documento, IAM
- Aggregates com invariantes de domínio
- Domain Events para comunicação entre contextos
- Repository pattern para persistência

**3. Event-Driven Architecture**
- Comunicação assíncrona via domain events
- Azure Service Bus para processamento long-running
- Event Handlers para orquestração de fluxos
- 3 filas principais: jarvis-ocr-queue, pega-routing-queue, notification-queue

**4. Modular Monolith**
- Módulos independentes e desacoplados
- Sem chamadas diretas entre módulos (apenas via eventos)
- Possibilidade futura de extração para microserviços se necessário

---

## 🔄 Fluxos Críticos

### Fluxo Principal: Processamento de Sinistro
```
[WPC] 
  → POST /api/sinistros (Idempotency-Key)
  → Validação + Persistência (RN-001, RN-002)
  → Evento: SinistroRecebido
  → Armazenamento Alfresco (CMIS, RN-006, RN-007)
  → Evento: SinistroArmazenado
  → Envio para Jarvis OCR (Azure Service Bus)
  → Callback OCR (scores: legibilidade, acuracidade)
  → Evento: OCRConcluido (se legibilidade >= 60)
  → Roteamento para Pega BPM
  → Criação de Case no Pega
  → Workflow iniciado
```

### Integrações Externas

| Sistema | Protocolo | Resiliência | Criticidade |
|---------|-----------|-------------|-------------|
| **Alfresco ECM** | CMIS 1.1 | CB (50%, 120s wait) + Retry 3x | Crítica |
| **Jarvis API** | REST/JSON (Async) | CB (50%, 60s wait) + Rate Limit | Alta |
| **Pega BPM** | REST/JSON | CB (50%, 60s wait) + Retry 3x | Alta |
| **CCM** | REST/JSON | Retry 3x | Média |
| **Prestador** | REST/JSON | Retry 2x | Média |
| **Portal PJ** | CMIS 1.1 | CB + Cache 5min | Baixa |
| **BOD Seguros** | REST/JSON | Retry 2x | Média |
| **Azure AD** | OAuth 2.0 | - | Crítica |

---

## 🔐 Segurança

### Autenticação e Autorização
- **OAuth 2.0** via Azure AD
- **JWT tokens** (8h TTL para access, 30 dias para refresh)
- **Mapeamento automático** de grupos Azure AD → perfis App View
- **RBAC granular** com 47 permissões por recurso

### Perfis de Acesso

| Perfil | Permissões | Público-Alvo |
|--------|------------|--------------|
| **ADMIN** | 47 (todas) | TI, Suporte |
| **OPERADOR** | 28 | Operações, BackOffice |
| **JURIDICO** | 12 | Departamento Jurídico (Portal PJ) |
| **GESTOR** | 15 | Gerência, Liderança (relatórios) |
| **CONSULTA** | 8 | Auditoria, Compliance (readonly) |

### Controles de Segurança
- **Idempotência obrigatória** (RN-068): header `Idempotency-Key` em todas operações de escrita
- **Auditoria completa**: todos os acessos registrados em `tb_log_acesso` (imutável)
- **Mascaramento de dados sensíveis**: CPF parcial para perfil CONSULTA
- **Row-level security**: OPERADOR vê apenas últimos 6 meses
- **Secrets management**: Azure Key Vault para credenciais

---

## 📈 Requisitos Não-Funcionais

### Performance
- **Latência P95:** < 500ms para leitura, < 2s para escrita
- **Throughput:** 100 req/s (pico)
- **Cache:** Redis com TTL de 5 minutos para consultas
- **Paginação:** Máximo 100 itens por página

### Disponibilidade
- **SLA:** 99.5% (4.38 horas downtime/ano)
- **RTO:** 2 horas
- **RPO:** 15 minutos (backups)

### Escalabilidade
- **Horizontal:** Azure App Service com auto-scaling (2-10 instâncias)
- **Database:** Read replicas para consultas pesadas
- **Cache distribuído:** Redis com persistência

### Resiliência
- **Circuit Breaker:** Protege todos os sistemas externos
- **Retry com exponential backoff:** Máximo 3 tentativas
- **Dead Letter Queues:** Service Bus com retry automático (5 tentativas)
- **Timeout:** 15s para Pega, 30s para Alfresco

### Observabilidade
- **Logs estruturados:** JSON com correlation ID (`X-Request-ID`)
- **Métricas:** Prometheus + Azure Application Insights
- **Tracing:** Distributed tracing end-to-end
- **Alerting:** Proativo para circuit breaker open, dead letters, latência

---

## 🧪 Estratégia de Testes

### Cobertura
- **Mínimo:** 80% de cobertura (código + branch)
- **Domain:** 100% (regras de negócio críticas)
- **Use Cases:** 90%
- **Controllers:** 80%

### Pirâmide de Testes
```
E2E Tests (10%)
  ↑ Playwright (frontend), REST Assured (API)

Integration Tests (30%)
  ↑ TestContainers (PostgreSQL, Redis, Alfresco mock)

Unit Tests (60%)
  ↑ JUnit 5 + Mockito (domain, use cases)
```

### Testes de Arquitetura
- **ArchUnit:** Valida dependências entre camadas
- Garante que Domain não depende de Infrastructure
- Valida naming conventions

---

## 🚀 Estratégia de Deploy

### Ambientes

| Ambiente | URL | Finalidade |
|----------|-----|------------|
| **DEV** | dev.appview.zurich.com | Desenvolvimento |
| **UAT** | uat.appview.zurich.com | Testes de aceitação |
| **PRD** | appview.zurich.com | Produção |

### Pipeline CI/CD (GitHub Actions)

```
[Commit] 
  → Build + Unit Tests
  → Integration Tests
  → Build Docker Image
  → Push to ACR
  → Deploy to DEV (automático)
  → Smoke Tests
  → Deploy to UAT (manual approval)
  → E2E Tests
  → Deploy to PRD (manual approval)
  → Blue/Green com canary (10% → 50% → 100%)
  → Rollback automático se erro rate > 5%
```

### Estratégia de Migração (Strangler Fig)

**Fase 1 (Meses 1-2):** Infraestrutura e Deploy Paralelo
- Setup Azure resources
- Deploy v2.0 em paralelo com v1.0
- Roteamento: 100% tráfego ainda em v1.0

**Fase 2 (Meses 3-5):** Migração Gradual por Bounded Context
- Mês 3: Portal PJ (baixo risco) → 100% tráfego para v2.0
- Mês 4: BOD, CCM, Perícia → 100% tráfego para v2.0
- Mês 5: Cliente, Sinistro (crítico) → Canary 10% → 50% → 100%

**Fase 3 (Mês 6):** Cutover Completo
- 100% tráfego em v2.0
- Desativação de v1.0
- Monitoramento intensivo (30 dias)

---

## 📊 Monitoramento e KPIs

### Métricas Técnicas
- **Latência:** P50, P95, P99 por endpoint
- **Taxa de sucesso:** % de requests 2xx
- **Circuit breaker:** % tempo em OPEN
- **Dead letters:** Total de mensagens na DLQ
- **Database:** Connection pool usage, slow queries (>2s)

### Métricas de Negócio
- **Volume de sinistros:** Total processados/dia
- **Taxa de OCR:** % com legibilidade >= 60
- **SLA de perícia:** % agendadas em até 7 dias
- **Taxa de reprocessamento:** % de documentos reenviados
- **Disponibilidade:** Uptime por sistema externo

### Alertas Críticos
- Circuit breaker OPEN por > 5 minutos
- Taxa de erro > 5% por 10 minutos
- Latência P95 > 2s por 5 minutos
- Dead letter queue > 50 mensagens
- Database connection pool exhaustion

---

## 🎯 Próximos Passos (Pós-Entrega)

### Documentação Adicional Recomendada
1. **OpenAPI 3.0 Specification** - Catálogo completo de APIs
2. **Frontend Implementation Guide** - Estrutura React + TypeScript
3. **Best Practices Guide** - Code review checklist, commits, branching
4. **Security Guide** - OWASP mitigations, rate limiting, security headers
5. **Performance Guide** - Cache strategies, lazy loading, compression
6. **Deployment Guide** - Procedimentos DEV/UAT/PRD, rollback
7. **Operations Runbook** - Troubleshooting, incident response

### Implementação
1. Setup de ambiente (Azure resources, GitHub repo)
2. Implementação incremental por bounded context
3. Testes contínuos (unit, integration, E2E)
4. Code review rigoroso
5. Deploy canary em PRD

---

## 📞 Contatos

**Equipe de Arquitetura**  
Email: arquitetura@zurich.com.br

**Equipe de Produto**  
Email: produto@zurich.com.br

**Suporte Técnico**  
Email: suporte.appview@zurich.com.br

---

## 📝 Histórico de Versões

| Versão | Data | Autor | Descrição |
|--------|------|-------|-----------|
| 2.0.0 | Janeiro 2026 | Equipe de Arquitetura | Documentação completa v2.0 |
| 1.0.0 | - | - | Sistema legado (Spring MVC 4.x) |

---

**Documento elaborado por:** Equipe de Arquitetura e Produto  
**Última atualização:** Janeiro de 2026  
**Status:** ✅ Completo e Aprovado
