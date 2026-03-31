# Catálogo de Regras de Negócio
**App View v2.0 - Documentação Funcional**

**Versão:** 2.0.0  
**Data:** Janeiro de 2026  

---

## Visão Geral

Este documento consolida todas as **regras de negócio** que governam o comportamento do sistema App View. As regras estão organizadas por categoria e módulo para fácil localização e rastreabilidade.

**Total de Regras:** 82  
**Criticidade:** 35 Altas | 30 Médias | 17 Baixas

---

## Índice

1. [Regras de Validação](#regras-de-validação)
2. [Regras de Segurança](#regras-de-segurança)
3. [Regras de Workflow](#regras-de-workflow)
4. [Regras de Auditoria](#regras-de-auditoria)
5. [Regras de Notificação](#regras-de-notificação)
6. [Regras de Retry e Resiliência](#regras-de-retry-e-resiliência)
7. [Regras de Performance](#regras-de-performance)

---

## Regras de Validação

### Módulo Sinistro

| ID | Descrição | Criticidade | Enforcement |
|----|-----------|-------------|-------------|
| RN-001 | Número de sinistro deve ser único no sistema | Alta | Entity validation |
| RN-002 | Data de recebimento não pode ser futura | Média | Bean Validation |
| RN-003 | Campos obrigatórios: numeroSinistro, tipoDocumento, canal, dataRecebimento | Alta | Bean Validation |
| RN-004 | Arquivo deve estar em base64 e ter no máximo 10MB | Alta | Controller validation |
| RN-005 | Tipos de documento válidos: LAUDO_MEDICO, BO_POLICIA, NOTA_FISCAL, ORCAMENTO, FOTO, OUTROS | Média | Enum validation |

**Implementação RN-001:**
```java
@Entity
public class Sinistro extends AggregateRoot<SinistroId> {
    
    public static Sinistro criar(SinistroData data, SinistroRepository repository) {
        // RN-001: Número de sinistro deve ser único
        if (repository.existsByNumeroSinistro(data.numeroSinistro())) {
            throw new SinistroDuplicadoException(data.numeroSinistro());
        }
        
        // RN-002: Data não pode ser futura
        if (data.dataRecebimento().isAfter(LocalDate.now())) {
            throw new DataInvalidaException("Data de recebimento não pode ser futura");
        }
        
        return new Sinistro(data);
    }
}
```

---

### Módulo Cliente

| ID | Descrição | Criticidade | Enforcement |
|----|-----------|-------------|-------------|
| RN-024 | CPF deve ser válido (dígitos verificadores) | Alta | Custom Validator |
| RN-025 | CNPJ deve ser válido (dígitos verificadores) | Alta | Custom Validator |
| RN-026 | Tipos de documento válidos: RG, CPF, CNH, CTPS, CNS, PASSAPORTE | Média | Enum validation |
| RN-027 | Documentos RG/CNH devem ter data de validade | Média | Conditional validation |
| RN-028 | Data de expedição deve ser anterior à data de validade | Média | Cross-field validation |

**Implementação RN-024/RN-025:**
```java
@Constraint(validatedBy = CpfCnpjValidator.class)
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
public @interface CPF_CNPJ {
    String message() default "CPF ou CNPJ inválido";
}

public class CpfCnpjValidator implements ConstraintValidator<CPF_CNPJ, String> {
    
    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null || value.isBlank()) {
            return true;
        }
        
        String digitsOnly = value.replaceAll("\\D", "");
        
        return switch (digitsOnly.length()) {
            case 11 -> validarCPF(digitsOnly);
            case 14 -> validarCNPJ(digitsOnly);
            default -> false;
        };
    }
    
    private boolean validarCPF(String cpf) {
        // Algoritmo de validação CPF
        // ...
    }
    
    private boolean validarCNPJ(String cnpj) {
        // Algoritmo de validação CNPJ
        // ...
    }
}
```

---

### Módulo Contrato

| ID | Descrição | Criticidade | Enforcement |
|----|-----------|-------------|-------------|
| RN-029 | Número de contrato deve ser único | Alta | Entity validation |
| RN-030 | Data fim de vigência > data início de vigência | Alta | Entity validation |
| RN-031 | Vigência mínima: 30 dias | Média | Entity validation |
| RN-032 | CNPJ do estipulante deve ser válido | Alta | Bean Validation |
| RN-033 | Apolice é obrigatória para contratos VIDA e PREVIDENCIA | Média | Conditional validation |

**Implementação RN-030/RN-031:**
```java
public class Contrato extends AggregateRoot<ContratoId> {
    
    public static Contrato criar(ContratoData data) {
        // RN-030: Fim > Início
        if (data.dataFimVigencia().isBefore(data.dataInicioVigencia())) {
            throw new VigenciaInvalidaException(
                "Data fim deve ser posterior à data início");
        }
        
        // RN-031: Mínimo 30 dias
        long diasVigencia = ChronoUnit.DAYS.between(
            data.dataInicioVigencia(), 
            data.dataFimVigencia()
        );
        
        if (diasVigencia < 30) {
            throw new VigenciaInvalidaException(
                "Vigência mínima é de 30 dias");
        }
        
        // RN-033: Apolice obrigatória para VIDA/PREVIDENCIA
        if ((data.tipo() == TipoContrato.VIDA || 
             data.tipo() == TipoContrato.PREVIDENCIA) &&
             data.apolice() == null) {
            throw new ApoliceObrigatoriaException(data.tipo());
        }
        
        return new Contrato(data);
    }
}
```

---

## Regras de Segurança

### Autenticação e Autorização

| ID | Descrição | Criticidade | Enforcement |
|----|-----------|-------------|-------------|
| RN-042 | Apenas usuários com conta Azure AD podem acessar | Alta | OAuth 2.0 |
| RN-043 | Sessões expiram após 8 horas de inatividade | Alta | JWT expiration |
| RN-044 | Refresh tokens expiram após 30 dias | Alta | Token config |
| RN-045 | Usuários inativos (90 dias sem login) são desabilitados automaticamente | Média | Scheduled job |
| RN-046 | Primeiro login força alteração de senha (se aplicável) | Baixa | Login flow |

### RBAC

| ID | Descrição | Criticidade | Enforcement |
|----|-----------|-------------|-------------|
| RN-038 | Apenas usuários JURIDICO podem acessar Portal PJ | Alta | @PreAuthorize |
| RN-050 | Usuário CONSULTA não pode fazer upload | Alta | @PreAuthorize |
| RN-054 | Apenas ADMIN pode acessar logs de auditoria | Alta | @PreAuthorize |
| RN-055 | Logs são imutáveis (não podem ser alterados ou deletados) | Alta | Database constraints |

**Implementação RN-055:**
```sql
-- Tabela de auditoria é append-only
CREATE TABLE audit.tb_log_transacao (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- ... outros campos
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
) WITH (fillfactor=100);

-- Revoke DELETE e UPDATE
REVOKE DELETE, UPDATE ON audit.tb_log_transacao FROM appview_api_user;
GRANT INSERT, SELECT ON audit.tb_log_transacao TO appview_api_user;
```

### Dados Sensíveis

| ID | Descrição | Criticidade | Enforcement |
|----|-----------|-------------|-------------|
| RN-056 | Dados sensíveis são mascarados na exibição (CPF, senhas, etc.) | Alta | DTO Mapping |
| RN-058 | Senhas devem ter mínimo 12 caracteres, incluir maiúscula, minúscula, número e símbolo | Alta | Password policy |
| RN-059 | Dados de pagamento/cartão nunca são armazenados localmente | Alta | Design decision |

**Implementação RN-056:**
```java
@Component
public class ClienteMapper {
    
    public static ClienteDTO toDTO(Cliente cliente, Authentication auth) {
        String cpf = cliente.getCpf();
        
        // RN-056: Mascara CPF para perfis CONSULTA
        if (hasRole(auth, "CONSULTA")) {
            cpf = mascarCPF(cpf); // "123.456.789-00" → "***.***.789-**"
        }
        
        return new ClienteDTO(
            cliente.getId(),
            cliente.getNome(),
            cpf,
            // ...
        );
    }
    
    private static String mascarCPF(String cpf) {
        return cpf.replaceAll("(\\d{3})(\\d{3})(\\d{3})(\\d{2})", "***.***.***-**");
    }
}
```

---

## Regras de Workflow

### Fluxo de Sinistro

| ID | Descrição | Criticidade | Enforcement |
|----|-----------|-------------|-------------|
| RN-006 | Estrutura de pastas deve seguir padrão: `/{ano}/{mes}/{numero_sinistro}/` | Média | Service logic |
| RN-007 | Tipo CMIS deve ser `zsSinistros:documentos_Sinistros` | Alta | CMIS config |
| RN-008 | Versionamento: MAJOR para novos documentos | Média | CMIS operation |
| RN-011 | Apenas documentos com status `ARMAZENADO` podem ser enviados para OCR | Alta | State machine |
| RN-015 | Scores devem estar entre 0 e 100 | Média | Bean Validation |
| RN-016 | Legibilidade mínima aceitável: 60 | Alta | Business rule |
| RN-019 | Documentos com legibilidade < 40 são rejeitados automaticamente | Alta | Business rule |
| RN-020 | Apenas sinistros com legibilidade >= 60 são roteados automaticamente | Alta | Business rule |

**Implementação RN-016/RN-019/RN-020:**
```java
public class ProcessarCallbackOCRUseCase {
    
    public void executar(CallbackOCRData data) {
        Sinistro sinistro = repository.findById(data.sinistroId());
        
        sinistro.atualizarScoresOCR(
            data.legibilidade(),
            data.acuracidade(),
            data.matchDocumento()
        );
        
        // RN-019: Rejeição automática se < 40
        if (data.legibilidade() < 40) {
            sinistro.marcarComoRejeitado("Legibilidade insuficiente");
            eventPublisher.publish(new SinistroRejeitadoEvent(sinistro));
            return;
        }
        
        // RN-016: Revisão manual se < 60
        if (data.legibilidade() < 60) {
            sinistro.marcarParaRevisaoManual();
            eventPublisher.publish(new DocumentoRequerRevisaoEvent(sinistro));
            return;
        }
        
        // RN-020: Roteamento automático se >= 60
        eventPublisher.publish(new OCRConcluidoEvent(sinistro));
    }
}
```

### Fluxo de Perícia

| ID | Descrição | Criticidade | Enforcement |
|----|-----------|-------------|-------------|
| RN-034 | Perícia deve ser agendada em até 7 dias úteis (SLA) | Alta | Business rule |
| RN-035 | Prestador deve estar ativo | Alta | Database query |
| RN-036 | Não permitir mais de 5 perícias para mesmo prestador no mesmo dia | Média | Database constraint |
| RN-037 | Notificação ao segurado é obrigatória | Alta | Event handler |

**Implementação RN-034:**
```java
public class AgendarPericiaUseCase {
    
    public PericiaId executar(AgendarPericiaCommand cmd) {
        // RN-034: SLA de 7 dias úteis
        LocalDate prazoMaximo = calcularProximoDiaUtil(LocalDate.now(), 7);
        
        if (cmd.dataAgendamento().isAfter(prazoMaximo)) {
            // Warning mas permite prosseguir
            logger.warn("Perícia agendada fora do SLA: {} > {}", 
                cmd.dataAgendamento(), prazoMaximo);
        }
        
        // RN-035: Prestador ativo
        Prestador prestador = prestadorRepository.findById(cmd.prestadorId())
            .filter(Prestador::isAtivo)
            .orElseThrow(() -> new PrestadorInativoException(cmd.prestadorId()));
        
        // RN-036: Máximo 5 perícias/dia por prestador
        long countPericias = periciaRepository.countByPrestadorAndData(
            cmd.prestadorId(), 
            cmd.dataAgendamento()
        );
        
        if (countPericias >= 5) {
            throw new PrestadorIndisponivelException(
                "Prestador já possui 5 perícias agendadas nesta data");
        }
        
        Pericia pericia = Pericia.agendar(cmd);
        periciaRepository.save(pericia);
        
        // RN-037: Notificação obrigatória
        eventPublisher.publish(new PericiaAgendadaEvent(pericia));
        
        return pericia.getId();
    }
    
    private LocalDate calcularProximoDiaUtil(LocalDate inicio, int dias) {
        // Implementação de cálculo de dias úteis
        // ...
    }
}
```

---

## Regras de Auditoria

### Logs e Rastreabilidade

| ID | Descrição | Criticidade | Enforcement |
|----|-----------|-------------|-------------|
| RN-041 | Logs de acesso devem ser registrados (auditoria) | Alta | AOP Aspect |
| RN-057 | Exportação limitada a 10.000 registros | Média | Query limit |
| RN-060 | Toda operação crítica deve ter correlation ID | Alta | MDC |
| RN-061 | Logs devem ser estruturados em JSON | Média | Logback config |
| RN-062 | Retenção de logs: 90 dias em hot storage, 2 anos em cold storage | Baixa | Azure config |

**Implementação RN-060:**
```java
@Component
public class RequestIdInterceptor implements HandlerInterceptor {
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                            HttpServletResponse response, 
                            Object handler) {
        // RN-060: Correlation ID obrigatório
        String requestId = request.getHeader("X-Request-ID");
        
        if (requestId == null || requestId.isBlank()) {
            requestId = UUID.randomUUID().toString();
        }
        
        MDC.put("requestId", requestId);
        response.setHeader("X-Request-ID", requestId);
        
        return true;
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request,
                                HttpServletResponse response,
                                Object handler,
                                Exception ex) {
        MDC.remove("requestId");
    }
}
```

---

## Regras de Notificação

### Comunicações

| ID | Descrição | Criticidade | Enforcement |
|----|-----------|-------------|-------------|
| RN-063 | Notificações de perícia devem ser enviadas com 48h de antecedência | Alta | Scheduled job |
| RN-064 | Máximo de 3 tentativas de envio de comunicação | Média | Retry logic |
| RN-065 | Email e SMS devem ter templates pré-aprovados pelo Compliance | Alta | Template validation |
| RN-066 | Notificações devem respeitar opt-out do cliente | Alta | Preference check |

**Implementação RN-063:**
```java
@Component
public class NotificacaoPericiaScheduler {
    
    @Scheduled(cron = "0 0 9 * * *") // Todo dia às 9h
    public void notificarPericiasProximas() {
        LocalDate dataLimite = LocalDate.now().plusDays(2);
        
        // RN-063: Notificar perícias com 48h de antecedência
        List<Pericia> pericias = periciaRepository
            .findByDataAgendamentoBetween(
                LocalDate.now().plusDays(2),
                LocalDate.now().plusDays(2).plusDays(1)
            );
        
        pericias.forEach(pericia -> {
            eventPublisher.publish(
                new NotificarPericiaEvent(pericia, TipoNotificacao.LEMBRETE_48H)
            );
        });
    }
}
```

---

## Regras de Retry e Resiliência

### Circuit Breaker

| ID | Descrição | Criticidade | Enforcement |
|----|-----------|-------------|-------------|
| RN-010 | Timeout de upload Alfresco: 30 segundos | Alta | Circuit Breaker |
| RN-014 | Rate limit: 50 requisições/minuto para Jarvis | Alta | Rate Limiter |
| RN-021 | Máximo de 3 retries para Pega | Média | Retry policy |
| RN-022 | Timeout de 15 segundos para chamada ao Pega | Alta | Circuit Breaker |
| RN-023 | Circuit breaker abre se taxa de falha > 50% | Alta | Circuit Breaker |
| RN-067 | Mensagens na dead letter queue após 5 tentativas | Alta | Service Bus config |

**Implementação RN-023:**
```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      pega:
        registerHealthIndicator: true
        slidingWindowSize: 100
        minimumNumberOfCalls: 10
        failureRateThreshold: 50  # RN-023: 50%
        slowCallRateThreshold: 70
        slowCallDurationThreshold: 10s
        waitDurationInOpenState: 60s
        permittedNumberOfCallsInHalfOpenState: 5
        automaticTransitionFromOpenToHalfOpenEnabled: true
        
      alfresco:
        # RN-010: Timeout 30s
        slowCallDurationThreshold: 30s
        failureRateThreshold: 50
        waitDurationInOpenState: 120s
```

### Idempotência

| ID | Descrição | Criticidade | Enforcement |
|----|-----------|-------------|-------------|
| RN-068 | Idempotency-Key é obrigatório para todas operações de escrita | Alta | Interceptor |
| RN-069 | Idempotency keys têm TTL de 24 horas | Alta | Database TTL |
| RN-070 | Response idempotente deve retornar 200 OK (não 202) | Média | Controller logic |

**Implementação RN-068:**
```java
@Component
public class IdempotencyInterceptor implements HandlerInterceptor {
    
    @Override
    public boolean preHandle(HttpServletRequest request,
                            HttpServletResponse response,
                            Object handler) throws IOException {
        
        String method = request.getMethod();
        
        // RN-068: Idempotency-Key obrigatório para POST/PUT/PATCH
        if (List.of("POST", "PUT", "PATCH").contains(method)) {
            String idempotencyKey = request.getHeader("Idempotency-Key");
            
            if (idempotencyKey == null || idempotencyKey.isBlank()) {
                response.setStatus(400);
                response.setContentType("application/json");
                response.getWriter().write("""
                    {
                        "error": "Bad Request",
                        "message": "Header 'Idempotency-Key' é obrigatório"
                    }
                """);
                return false;
            }
            
            request.setAttribute("idempotencyKey", idempotencyKey);
        }
        
        return true;
    }
}
```

---

## Regras de Performance

### Cache e Otimização

| ID | Descrição | Criticidade | Enforcement |
|----|-----------|-------------|-------------|
| RN-039 | Resultados limitados a 100 por página | Média | Pageable default |
| RN-040 | Cache de 5 minutos para consultas idênticas | Baixa | @Cacheable |
| RN-051 | Dados de dashboard atualizados a cada 5 minutos | Média | Cache TTL |
| RN-052 | Materialized view é refreshed a cada 5 minutos | Média | Scheduled job |
| RN-053 | Máximo de 90 dias de histórico em dashboards | Baixa | Query filter |
| RN-071 | Queries lentas (>2s) devem ser logadas com WARN | Média | JPA logging |
| RN-072 | Máximo de 1000 registros em operações batch | Média | Batch size |

**Implementação RN-040:**
```java
@Service
public class PortalPJService {
    
    @Cacheable(
        value = "propostas", 
        key = "#filtro.hashCode()",
        unless = "#result.isEmpty()"
    )
    @CacheEvict(value = "propostas", allEntries = true, 
                condition = "#result.size() > 1000")
    public Page<PropostaDTO> consultarPropostas(
            PropostaFiltro filtro, 
            Pageable pageable) {
        
        // RN-040: Cache de 5 minutos
        return repository.findAll(
            PropostaSpecifications.comFiltros(filtro), 
            pageable
        ).map(PropostaMapper::toDTO);
    }
}
```

```yaml
spring:
  cache:
    type: redis
    redis:
      time-to-live: 300000  # RN-040: 5 minutos
```

### Arquivos e Upload

| ID | Descrição | Criticidade | Enforcement |
|----|-----------|-------------|-------------|
| RN-047 | Tipos permitidos: PDF, PNG, JPG, JPEG, TIFF | Alta | Content-Type validation |
| RN-048 | Tamanho máximo: 10MB | Alta | MultipartConfig |
| RN-049 | Nome do arquivo não pode conter: `/ \ : * ? " < > \|` | Média | Filename sanitization |
| RN-073 | Upload maior que 5MB deve usar chunking | Baixa | Upload strategy |

**Implementação RN-047/RN-048:**
```java
@Component
public class DocumentoValidator {
    
    private static final Set<String> TIPOS_PERMITIDOS = Set.of(
        "application/pdf",
        "image/png",
        "image/jpeg",
        "image/tiff"
    );
    
    private static final long MAX_SIZE = 10 * 1024 * 1024; // 10MB
    
    public void validar(MultipartFile file) {
        // RN-047: Tipos permitidos
        if (!TIPOS_PERMITIDOS.contains(file.getContentType())) {
            throw new TipoArquivoInvalidoException(file.getContentType());
        }
        
        // RN-048: Tamanho máximo
        if (file.getSize() > MAX_SIZE) {
            throw new ArquivoMuitoGrandeException(file.getSize(), MAX_SIZE);
        }
        
        // RN-049: Caracteres inválidos no nome
        String filename = file.getOriginalFilename();
        if (filename.matches(".*[/\\\\:*?\"<>|].*")) {
            throw new NomeArquivoInvalidoException(filename);
        }
    }
}
```

---

## Regras de Integridade de Dados

### Constraints e Validações

| ID | Descrição | Criticidade | Enforcement |
|----|-----------|-------------|-------------|
| RN-074 | Deleção de cliente só é permitida se não houver documentos associados | Alta | Foreign key |
| RN-075 | Alteração de número de sinistro não é permitida após criação | Alta | Entity immutability |
| RN-076 | Status de sinistro segue máquina de estados válida | Alta | State machine |
| RN-077 | Soft delete em vez de hard delete para entidades críticas | Alta | @SQLDelete |
| RN-078 | Versioning otimista para prevenir conflitos de atualização | Média | @Version |

**Implementação RN-076:**
```java
public enum StatusSinistro {
    RECEBIDO,
    ARMAZENADO,
    ENVIADO_PARA_OCR,
    OCR_CONCLUIDO,
    ROTEADO_PARA_PEGA,
    REVISAO_MANUAL,
    REJEITADO,
    CONCLUIDO;
    
    // RN-076: Transições válidas
    private static final Map<StatusSinistro, Set<StatusSinistro>> TRANSICOES = Map.of(
        RECEBIDO, Set.of(ARMAZENADO),
        ARMAZENADO, Set.of(ENVIADO_PARA_OCR),
        ENVIADO_PARA_OCR, Set.of(OCR_CONCLUIDO, REVISAO_MANUAL, REJEITADO),
        OCR_CONCLUIDO, Set.of(ROTEADO_PARA_PEGA),
        REVISAO_MANUAL, Set.of(ENVIADO_PARA_OCR, REJEITADO),
        ROTEADO_PARA_PEGA, Set.of(CONCLUIDO),
        REJEITADO, Set.of(),
        CONCLUIDO, Set.of()
    );
    
    public boolean podeTransicionarPara(StatusSinistro novo) {
        return TRANSICOES.get(this).contains(novo);
    }
}

@Entity
public class Sinistro {
    
    public void alterarStatus(StatusSinistro novoStatus) {
        // RN-076: Validar transição
        if (!this.status.podeTransicionarPara(novoStatus)) {
            throw new TransicaoInvalidaException(this.status, novoStatus);
        }
        
        this.status = novoStatus;
        this.updatedAt = Instant.now();
    }
}
```

**Implementação RN-077:**
```java
@Entity
@SQLDelete(sql = "UPDATE tb_sinistro SET deleted_at = NOW() WHERE id = ?")
@Where(clause = "deleted_at IS NULL")
public class Sinistro {
    
    @Column(name = "deleted_at")
    private Instant deletedAt;
    
    public boolean isDeleted() {
        return deletedAt != null;
    }
}
```

---

## Matriz de Rastreabilidade

| Regra | Use Case | Endpoint | Teste |
|-------|----------|----------|-------|
| RN-001 | UC-001 | POST /api/sinistros | SinistroValidationTest |
| RN-016 | UC-004, UC-005 | POST /api/jarvis/callback | OCRWorkflowTest |
| RN-023 | UC-005 | POST /api/pega/cases | PegaCircuitBreakerTest |
| RN-034 | UC-020 | POST /api/pericias | PericiaAgendamentoTest |
| RN-038 | UC-024 | GET /api/portal/propostas | PortalAuthorizationTest |
| RN-068 | Todos POST/PUT | * | IdempotencyTest |

---

## Resumo por Categoria

| Categoria | Total Regras | Alta | Média | Baixa |
|-----------|--------------|------|-------|-------|
| Validação | 22 | 12 | 8 | 2 |
| Segurança | 15 | 12 | 2 | 1 |
| Workflow | 18 | 10 | 6 | 2 |
| Auditoria | 8 | 5 | 2 | 1 |
| Notificação | 4 | 3 | 1 | 0 |
| Retry/Resiliência | 8 | 6 | 2 | 0 |
| Performance | 7 | 2 | 4 | 1 |
| **Total** | **82** | **35** | **30** | **17** |

---

## Boas Práticas de Implementação

1. **Regras no Domínio:** Sempre validar regras de negócio na camada de domínio (Entities/Aggregates)
2. **Fail Fast:** Validar regras críticas o mais cedo possível no fluxo
3. **Mensagens Claras:** Exceções devem indicar claramente qual regra foi violada
4. **Testes de Regra:** Cada regra deve ter pelo menos 1 teste automatizado
5. **Documentação:** Comentar código com ID da regra (ex: `// RN-001`)

---

**Documento elaborado por:** Equipe de Produto e Arquitetura  
**Última atualização:** Janeiro de 2026
