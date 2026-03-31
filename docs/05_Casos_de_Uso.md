# Especificação de Casos de Uso
**App View v2.0 - Documentação Funcional**

**Versão:** 2.0.0  
**Data:** Janeiro de 2026  

---

## Índice de Casos de Uso

### Módulo Sinistro
1. [UC-001: Receber Documento de Sinistro](#uc-001-receber-documento-de-sinistro)
2. [UC-002: Armazenar Sinistro no ECM](#uc-002-armazenar-sinistro-no-ecm)
3. [UC-003: Enviar Sinistro para OCR](#uc-003-enviar-sinistro-para-ocr)
4. [UC-004: Processar Callback de OCR](#uc-004-processar-callback-de-ocr)
5. [UC-005: Rotear Sinistro para Pega](#uc-005-rotear-sinistro-para-pega)
6. [UC-006: Consultar Sinistro](#uc-006-consultar-sinistro)
7. [UC-007: Atualizar Metadados de Sinistro](#uc-007-atualizar-metadados-de-sinistro)

### Módulo Cliente
8. [UC-008: Receber Documento de Cliente](#uc-008-receber-documento-de-cliente)
9. [UC-009: Validar Documento de Cliente](#uc-009-validar-documento-de-cliente)
10. [UC-010: Consultar Documentos de Cliente](#uc-010-consultar-documentos-de-cliente)
11. [UC-011: Atualizar Dados de Cliente](#uc-011-atualizar-dados-de-cliente)

### Módulo Contrato (BOD)
12. [UC-012: Implantar Contrato Corporativo](#uc-012-implantar-contrato-corporativo)
13. [UC-013: Renovar Contrato](#uc-013-renovar-contrato)
14. [UC-014: Registrar Endosso](#uc-014-registrar-endosso)
15. [UC-015: Cancelar Contrato](#uc-015-cancelar-contrato)
16. [UC-016: Consultar Contratos](#uc-016-consultar-contratos)

### Módulo Comunicação
17. [UC-017: Enviar Comunicação ao Segurado](#uc-017-enviar-comunicação-ao-segurado)
18. [UC-018: Consultar Status de Comunicação](#uc-018-consultar-status-de-comunicação)
19. [UC-019: Reenviar Comunicação](#uc-019-reenviar-comunicação)

### Módulo Perícia
20. [UC-020: Agendar Perícia](#uc-020-agendar-perícia)
21. [UC-021: Atualizar Status de Perícia](#uc-021-atualizar-status-de-perícia)
22. [UC-022: Registrar Resultado de Perícia](#uc-022-registrar-resultado-de-perícia)
23. [UC-023: Consultar Perícias por Prestador](#uc-023-consultar-perícias-por-prestador)

### Módulo Portal
24. [UC-024: Consultar Propostas no Portal PJ](#uc-024-consultar-propostas-no-portal-pj)
25. [UC-025: Baixar Documentos do Portal PJ](#uc-025-baixar-documentos-do-portal-pj)
26. [UC-026: Consultar Portabilidades](#uc-026-consultar-portabilidades)

### Módulo IAM
27. [UC-027: Autenticar Usuário](#uc-027-autenticar-usuário)
28. [UC-028: Autorizar Acesso a Recurso](#uc-028-autorizar-acesso-a-recurso)
29. [UC-029: Gerenciar Perfis](#uc-029-gerenciar-perfis)
30. [UC-030: Gerenciar Usuários](#uc-030-gerenciar-usuários)
31. [UC-031: Alterar Senha](#uc-031-alterar-senha)

### Módulo Documento
32. [UC-032: Upload de Documento Genérico](#uc-032-upload-de-documento-genérico)
33. [UC-033: Download de Documento](#uc-033-download-de-documento)
34. [UC-034: Gerar Link Temporário](#uc-034-gerar-link-temporário)

### Módulo Relatórios
35. [UC-035: Gerar Dashboard Operacional](#uc-035-gerar-dashboard-operacional)
36. [UC-036: Exportar Relatório](#uc-036-exportar-relatório)
37. [UC-037: Consultar Logs de Auditoria](#uc-037-consultar-logs-de-auditoria)

---

## UC-001: Receber Documento de Sinistro

### Descrição
Sistema recebe documento de sinistro de sistema externo (WPC), valida os dados e inicia o fluxo de processamento.

### Atores
- **Principal:** Sistema WPC (externo)
- **Secundário:** Sistema App View

### Pré-condições
- Sistema WPC possui credenciais válidas
- Idempotency-Key é fornecido no header

### Pós-condições (Sucesso)
- Sinistro registrado no banco de dados
- Evento `SinistroRecebido` publicado
- Response 202 Accepted retornado ao cliente

### Fluxo Principal

1. Sistema WPC envia requisição POST para `/api/sinistros`
2. Sistema valida header `Idempotency-Key`
3. Sistema valida campos obrigatórios via Bean Validation
4. Sistema cria entidade `Sinistro` via factory method
5. Sinistro valida invariantes de domínio
6. Sistema persiste sinistro no banco
7. Sistema publica evento `SinistroRecebido`
8. Sistema registra idempotency key
9. Sistema retorna 202 Accepted com ID do sinistro

### Fluxos Alternativos

**FA-1: Idempotency Key Duplicado**
- 3a. Sistema verifica que key já existe
- 3b. Sistema retorna response cacheado (200 OK)
- Fim do caso de uso (sem processamento duplicado)

**FA-2: Validação de Campos Falha**
- 4a. Bean Validation identifica campos inválidos
- 4b. Sistema retorna 400 Bad Request com lista de erros
- Fim do caso de uso

**FA-3: Número de Sinistro Duplicado**
- 5a. Entidade detecta número duplicado
- 5b. Sistema lança `SinistroDuplicadoException`
- 5c. Sistema retorna 409 Conflict
- Fim do caso de uso

### Fluxos de Exceção

**FE-1: Erro de Banco de Dados**
- 6a. PostgreSQL retorna erro (ex: conexão perdida)
- 6b. Sistema executa retry (3 tentativas)
- 6c. Se falhar: retorna 503 Service Unavailable
- 6d. Log de erro crítico é gerado

### Regras de Negócio

- **RN-001:** Número de sinistro deve ser único no sistema
- **RN-002:** Data de recebimento não pode ser futura
- **RN-003:** Campos obrigatórios: numeroSinistro, tipoDocumento, canal, dataRecebimento
- **RN-004:** Arquivo deve estar em base64 e ter no máximo 10MB
- **RN-005:** Tipos de documento válidos: LAUDO_MEDICO, BO_POLICIA, NOTA_FISCAL, ORCAMENTO, FOTO, OUTROS

### Validações

```java
@NotBlank(message = "Número do sinistro é obrigatório")
@Pattern(regexp = "^SIN[0-9]{6}$", message = "Formato inválido")
private String numeroSinistro;

@NotBlank(message = "Tipo de documento é obrigatório")
private String tipoDocumento;

@NotNull(message = "Data de recebimento é obrigatória")
@PastOrPresent(message = "Data não pode ser futura")
private LocalDate dataRecebimento;

@NotBlank(message = "Arquivo é obrigatório")
@Base64(message = "Arquivo deve estar em base64")
@Size(max = 14680064, message = "Arquivo muito grande (máx 10MB)")
private String fileData;
```

### Payloads

**Request:**
```json
{
  "numeroSinistro": "SIN123456",
  "tipoDocumento": "LAUDO_MEDICO",
  "canal": "WPC",
  "lossInfo": {
    "lossNumber": "LOSS123",
    "lossSequence": "01",
    "lossYear": 2026,
    "branch": "AUTO"
  },
  "dataRecebimento": "2026-01-27",
  "apolice": "POL-2026-001",
  "ramo": "AUTOMOVEL",
  "certificado": "CERT-123",
  "clienteId": "12345678900",
  "claimId": "CLAIM-456",
  "fileData": "JVBERi0xLjQK...",
  "nomeArquivo": "laudo_medico.pdf",
  "descricao": "Laudo médico do acidente",
  "publico": false
}
```

**Response (Sucesso):**
```json
{
  "sinistroId": "550e8400-e29b-41d4-a716-446655440000",
  "numeroSinistro": "SIN123456",
  "status": "RECEBIDO",
  "dataRecebimento": "2026-01-27T21:10:00Z"
}
```

**Response (Erro - Validação):**
```json
{
  "timestamp": "2026-01-27T21:10:00Z",
  "status": 400,
  "error": "Bad Request",
  "message": "Validation failed",
  "errors": [
    {
      "field": "numeroSinistro",
      "rejectedValue": "123",
      "message": "Formato inválido"
    },
    {
      "field": "dataRecebimento",
      "rejectedValue": "2027-01-01",
      "message": "Data não pode ser futura"
    }
  ]
}
```

---

## UC-002: Armazenar Sinistro no ECM

### Descrição
Após recepção do sinistro, o sistema armazena o documento no Alfresco ECM com metadados apropriados.

### Atores
- **Principal:** Sistema App View (Event Handler)
- **Secundário:** Alfresco ECM

### Pré-condições
- Sinistro foi recebido e persistido (UC-001)
- Evento `SinistroRecebido` foi publicado
- Arquivo está em formato válido

### Pós-condições (Sucesso)
- Documento armazenado no Alfresco
- Estrutura de pastas criada (se não existir)
- Metadados CMIS populados
- ID do documento Alfresco registrado no sinistro
- Evento `SinistroArmazenado` publicado

### Fluxo Principal

1. Handler captura evento `SinistroRecebido`
2. Sistema busca sinistro no banco de dados
3. Sistema extrai dados do sinistro
4. Sistema determina caminho no Alfresco: `/prod-sinistros/{ano}/{mes}/{numero_sinistro}/`
5. Sistema verifica se pasta existe, senão cria estrutura completa
6. Sistema prepara ContentStream com arquivo
7. Sistema monta propriedades CMIS customizadas
8. Sistema executa `createDocument()` no Alfresco
9. Sistema atualiza sinistro com `alfrescoDocumentId`
10. Sistema publica evento `SinistroArmazenado`

### Fluxos Alternativos

**FA-1: Pasta Já Existe**
- 5a. Pasta do sinistro já foi criada anteriormente
- 5b. Sistema verifica versionamento
- 5c. Se documento com mesmo nome: criar nova versão (major)
- Continua em passo 6

**FA-2: Alfresco Indisponível**
- 8a. CMIS lança `CmisConnectionException`
- 8b. Circuit Breaker detecta falha
- 8c. Sistema enfileira em `alfresco-retry-queue`
- 8d. Sistema marca sinistro como `PENDENTE_ARMAZENAMENTO`
- Fim do caso de uso (retry automático posteriormente)

### Regras de Negócio

- **RN-006:** Estrutura de pastas deve seguir padrão: `/{ano}/{mes}/{numero_sinistro}/`
- **RN-007:** Tipo CMIS deve ser `zsSinistros:documentos_Sinistros`
- **RN-008:** Versionamento: MAJOR para novos documentos
- **RN-009:** Propriedades customizadas obrigatórias devem ser preenchidas
- **RN-010:** Timeout de upload: 30 segundos

### Validações

- Arquivo não pode estar corrompido
- Tipo de conteúdo deve ser PDF, PNG, JPG ou TIFF
- Nome do arquivo não pode conter caracteres especiais
- Tamanho máximo: 10MB

### Tratamento de Erros

| Exceção CMIS | Ação |
|--------------|------|
| `CmisConnectionException` | Retry 3x, enfileirar se falhar |
| `CmisStorageException` | Alfresco sem espaço, alert crítico |
| `CmisPermissionDeniedException` | Credenciais inválidas, alert crítico |
| `CmisConstraintException` | Metadados inválidos, retornar erro |

---

## UC-003: Enviar Sinistro para OCR

### Descrição
Após armazenamento no Alfresco, documento é enviado para Jarvis para processamento OCR.

### Atores
- **Principal:** Sistema App View (Event Handler)
- **Secundário:** Jarvis API, Azure Service Bus

### Pré-condições
- Sinistro foi armazenado no Alfresco (UC-002)
- Evento `SinistroArmazenado` foi publicado

### Pós-condições (Sucesso)
- Mensagem enfileirada em `jarvis-ocr-queue`
- Sinistro marcado como `ENVIADO_PARA_OCR`
- Registro criado em `tb_ecm_jarvis`

### Fluxo Principal

1. Handler captura evento `SinistroArmazenado`
2. Sistema enfileira mensagem em Azure Service Bus (jarvis-ocr-queue)
3. Service Bus Listener processa mensagem
4. Sistema busca sinistro e documento do Alfresco
5. Sistema converte documento para base64
6. Sistema monta payload Jarvis
7. Sistema envia POST para Jarvis com circuit breaker
8. Jarvis retorna 202 Accepted
9. Sistema registra em `tb_ecm_jarvis` com status PROCESSING
10. Sistema completa mensagem do Service Bus

### Fluxos Alternativos

**FA-1: Jarvis Retorna Rate Limit (429)**
- 8a. Jarvis retorna 429 Too Many Requests
- 8b. Sistema abandona mensagem com delay de 60s
- 8c. Service Bus re-enfileira automaticamente
- Fim do caso de uso (retry posterior)

**FA-2: Circuit Breaker Aberto**
- 7a. Circuit breaker está em estado OPEN
- 7b. Sistema não faz chamada ao Jarvis
- 7c. Sistema abandona mensagem com delay
- Fim do caso de uso (retry quando circuit fechar)

### Regras de Negócio

- **RN-011:** Apenas documentos com status `ARMAZENADO` podem ser enviados para OCR
- **RN-012:** Máximo de 3 tentativas de envio ao Jarvis
- **RN-013:** Documentos com > 10MB não devem ser enviados ao Jarvis
- **RN-014:** Rate limit: 50 requisições/minuto para Jarvis

### Validações

- Document code (Alfresco ID) deve existir
- Arquivo deve estar acessível no Alfresco
- Formato deve ser PDF, PNG, JPG ou TIFF
- Base64 não pode exceder 14MB

---

## UC-004: Processar Callback de OCR

### Descrição
Sistema recebe callback do Jarvis com resultados da análise OCR e atualiza metadados do documento.

### Atores
- **Principal:** Jarvis API (externo)
- **Secundário:** Sistema App View

### Pré-condições
- Documento foi enviado para Jarvis (UC-003)
- Registro existe em `tb_ecm_jarvis`

### Pós-condições (Sucesso)
- Scores de OCR atualizados no sinistro
- Propriedades do Alfresco atualizadas
- Evento `OCRConcluido` publicado
- Registro em `tb_ecm_jarvis` atualizado

### Fluxo Principal

1. Jarvis envia POST para `/api/jarvis/callback`
2. Sistema valida assinatura do webhook (HMAC)
3. Sistema busca registro em `tb_ecm_jarvis` por `documentCode`
4. Sistema busca sinistro relacionado
5. Sistema valida scores (0-100)
6. Sinistro atualiza scores via método de domínio
7. Sistema atualiza propriedades no Alfresco (CMIS)
8. Sistema atualiza `tb_ecm_jarvis` com status COMPLETED
9. Sistema publica evento `OCRConcluido`
10. Sistema retorna 200 OK ao Jarvis

### Fluxos Alternativos

**FA-1: Document Code Não Encontrado**
- 3a. Registro não existe em `tb_ecm_jarvis`
- 3b. Sistema loga warning (possível callback duplicado ou tardio)
- 3c. Sistema retorna 404 Not Found
- Fim do caso de uso

**FA-2: Scores Indicam Baixa Qualidade**
- 6a. Legibilidade < 60
- 6b. Sistema marca sinistro como `REVISAO_MANUAL`
- 6c. Sistema publica evento `DocumentoRequerRevisao`
- 6d. Não roteia para Pega automaticamente
- Continua em passo 7

**FA-3: Status do Jarvis é FAILED**
- 3a. Callback indica falha no processamento
- 3b. Sistema marca em `tb_ecm_jarvis` como FAILED
- 3c. Sistema registra erro em `tb_log_transacao`
- 3d. Sistema publica evento `OCRFalhou`
- 3e. Sinistro pode ser reenviado manualmente
- Fim do caso de uso

### Regras de Negócio

- **RN-015:** Scores devem estar entre 0 e 100
- **RN-016:** Legibilidade mínima aceitável: 60
- **RN-017:** Callback deve ser recebido em até 5 minutos (warning se exceder)
- **RN-018:** Assinatura HMAC do webhook deve ser válida
- **RN-019:** Documentos com legibilidade < 40 são rejeitados automaticamente

### Validações

```java
@Min(value = 0, message = "Score mínimo é 0")
@Max(value = 100, message = "Score máximo é 100")
private Integer legibilidade;

@Min(value = 0, message = "Score mínimo é 0")
@Max(value = 100, message = "Score máximo é 100")
private Integer acuracidade;

@Min(value = 0, message = "Score mínimo é 0")
@Max(value = 100, message = "Score máximo é 100")
private Integer matchDocumento;
```

---

## UC-005: Rotear Sinistro para Pega

### Descrição
Após conclusão do OCR com scores aceitáveis, sinistro é roteado para Pega BPM para início do workflow.

### Atores
- **Principal:** Sistema App View (Event Handler)
- **Secundário:** Pega BPM

### Pré-condições
- OCR foi concluído com sucesso (UC-004)
- Scores de legibilidade >= 60
- Evento `OCRConcluido` foi publicado

### Pós-condições (Sucesso)
- Caso criado no Pega
- Workflow iniciado
- ID do caso Pega registrado no sinistro
- Log de auditoria criado em `tb_log_transacao`

### Fluxo Principal

1. Handler captura evento `OCRConcluido`
2. Sistema verifica score de legibilidade (>= 60)
3. Sistema busca sinistro completo
4. Sistema monta payload Pega com metadados + scores OCR
5. Sistema executa POST para Pega via gateway (com circuit breaker)
6. Pega cria caso e retorna 201 Created
7. Sistema atualiza sinistro com `pegaCaseId`
8. Sistema registra em `tb_log_transacao` (status: sucesso)
9. Sistema publica evento `SinistroRoteadoParaPega`

### Fluxos Alternativos

**FA-1: Pega Retorna 409 (Caso Já Existe)**
- 6a. Pega indica que caso já foi criado (idempotência do Pega)
- 6b. Sistema extrai caseId da resposta
- 6c. Sistema atualiza sinistro normalmente
- Continua em passo 7

**FA-2: Circuit Breaker Aberto**
- 5a. Gateway detecta circuit breaker em OPEN
- 5b. Sistema executa fallback: enfileirar em `pega-retry-queue`
- 5c. Sistema marca sinistro como `PENDENTE_ROTEAMENTO`
- 5d. Sistema registra em log (status: fallback)
- Fim do caso de uso

**FA-3: Timeout do Pega**
- 5a. Pega não responde em 15 segundos
- 5b. Sistema executa retry (3 tentativas com exponential backoff)
- 5c. Se todas falharem: enfileirar para retry posterior
- Fim do caso de uso

### Regras de Negócio

- **RN-020:** Apenas sinistros com legibilidade >= 60 são roteados automaticamente
- **RN-021:** Máximo de 3 retries para Pega
- **RN-022:** Timeout de 15 segundos para chamada ao Pega
- **RN-023:** Circuit breaker abre se taxa de falha > 50%

### Métricas Coletadas

- `pega.cases.created.total` - Total de casos criados
- `pega.integration.duration` - Latência da chamada
- `pega.integration.success.rate` - Taxa de sucesso

---

## UC-008: Receber Documento de Cliente

### Descrição
Sistema recebe documento de cliente (RG, CPF, CNH, etc.) de sistema externo para validação.

### Atores
- **Principal:** Sistema Pega (externo)
- **Secundário:** Sistema App View

### Pré-condições
- Sistema Pega possui credenciais válidas
- Cliente existe ou será criado automaticamente

### Pós-condições (Sucesso)
- Cliente registrado/atualizado
- Documento de cliente armazenado no Alfresco
- Enviado para validação OCR
- Response 202 Accepted retornado

### Fluxo Principal

1. Pega envia POST para `/api/clientes`
2. Sistema valida Idempotency-Key
3. Sistema valida campos obrigatórios
4. Sistema busca ou cria cliente por CPF/CNPJ
5. Sistema cria entidade `DocumentoCliente`
6. Sistema persiste no banco
7. Sistema publica evento `DocumentoClienteRecebido`
8. Sistema retorna 202 Accepted

### Fluxos Alternativos

**FA-1: Cliente Não Existe**
- 4a. CPF/CNPJ não encontrado
- 4b. Sistema cria novo cliente automaticamente
- 4c. Cliente criado com dados mínimos (nome, CPF/CNPJ, email)
- Continua em passo 5

**FA-2: CPF/CNPJ Inválido**
- 3a. Validação de CPF/CNPJ falha
- 3b. Sistema retorna 400 Bad Request
- Fim do caso de uso

### Regras de Negócio

- **RN-024:** CPF deve ser válido (dígitos verificadores)
- **RN-025:** CNPJ deve ser válido (dígitos verificadores)
- **RN-026:** Tipos de documento válidos: RG, CPF, CNH, CTPS, CNS, PASSAPORTE
- **RN-027:** Documentos RG/CNH devem ter data de validade
- **RN-028:** Data de expedição deve ser anterior à data de validade

### Validações

```java
@NotBlank(message = "ID do cliente é obrigatório")
@CPF_CNPJ(message = "CPF ou CNPJ inválido")
private String clienteId;

@NotBlank(message = "Tipo de documento é obrigatório")
@Pattern(regexp = "^(RG|CPF|CNH|CTPS|CNS|PASSAPORTE)$")
private String tipoDocumento;

@NotNull(message = "Data de expedição é obrigatória")
@Past(message = "Data de expedição deve ser passada")
private LocalDate dataExpedicao;

@Future(message = "Data de validade deve ser futura")
private LocalDate dataValidade;
```

---

## UC-012: Implantar Contrato Corporativo

### Descrição
Sistema registra implantação de novo contrato corporativo (BOD Seguros).

### Atores
- **Principal:** Sistema BOD Seguros (externo)
- **Secundário:** Sistema App View, Alfresco ECM

### Pré-condições
- Contrato foi aprovado no sistema BOD
- Cliente corporativo existe

### Pós-condições (Sucesso)
- Contrato registrado no banco
- Documentação armazenada no Alfresco
- Status: ATIVO
- Disponível para consulta no Portal

### Fluxo Principal

1. BOD envia POST para `/api/bod/implantacao`
2. Sistema valida Idempotency-Key
3. Sistema valida campos obrigatórios
4. Sistema cria entidade `Contrato`
5. Sistema valida regras de negócio (vigência, estipulante, etc.)
6. Sistema cria estrutura de pastas no Alfresco
7. Sistema faz upload da proposta
8. Sistema persiste contrato no banco
9. Sistema publica evento `ContratoImplantado`
10. Sistema retorna 201 Created

### Fluxos Alternativos

**FA-1: Número de Contrato Duplicado**
- 4a. Número de contrato já existe
- 4b. Sistema retorna 409 Conflict
- Fim do caso de uso

**FA-2: Data de Vigência Inválida**
- 5a. Fim de vigência anterior ao início
- 5b. Sistema retorna 400 Bad Request
- Fim do caso de uso

### Regras de Negócio

- **RN-029:** Número de contrato deve ser único
- **RN-030:** Data fim de vigência > data início de vigência
- **RN-031:** Vigência mínima: 30 dias
- **RN-032:** CNPJ do estipulante deve ser válido
- **RN-033:** Apolice é obrigatória para contratos VIDA e PREVIDENCIA

---

## UC-020: Agendar Perícia

### Descrição
Analista agenda perícia com prestador após análise inicial do sinistro.

### Atores
- **Principal:** Usuário Analista
- **Secundário:** Sistema Prestador, CCM

### Pré-condições
- Usuário possui perfil OPERADOR ou ADMIN
- Sinistro existe e está em análise
- Prestador está ativo

### Pós-condições (Sucesso)
- Perícia agendada
- Prestador notificado
- Segurado notificado via CCM
- Registro em `tb_pericia`

### Fluxo Principal

1. Analista acessa tela de sinistro
2. Analista clica em "Agendar Perícia"
3. Sistema exibe formulário
4. Analista seleciona prestador, tipo, data e local
5. Analista submete formulário
6. Sistema valida disponibilidade do prestador
7. Sistema valida prazo SLA (perícia deve ocorrer em até 7 dias)
8. Sistema cria entidade `Pericia`
9. Sistema notifica prestador via API
10. Sistema notifica segurado via CCM (template: AGENDAMENTO_PERICIA)
11. Sistema persiste perícia
12. Sistema retorna confirmação

### Fluxos Alternativos

**FA-1: Prestador Não Disponível na Data**
- 6a. Prestador já possui perícia agendada no mesmo horário
- 6b. Sistema retorna 409 Conflict
- 6c. Sistema sugere próximas datas disponíveis
- Fim do caso de uso

**FA-2: Data Fora do SLA**
- 7a. Data agendada é superior a 7 dias úteis
- 7b. Sistema exibe warning mas permite prosseguir
- 7c. Sistema marca como "FORA_SLA"
- Continua em passo 8

### Regras de Negócio

- **RN-034:** Perícia deve ser agendada em até 7 dias úteis (SLA)
- **RN-035:** Prestador deve estar ativo
- **RN-036:** Não permitir mais de 5 perícias para mesmo prestador no mesmo dia
- **RN-037:** Notificação ao segurado é obrigatória

---

## UC-024: Consultar Propostas no Portal PJ

### Descrição
Usuário do departamento jurídico consulta propostas de clientes corporativos.

### Atores
- **Principal:** Usuário Jurídico
- **Secundário:** Alfresco ECM

### Pré-condições
- Usuário autenticado
- Usuário possui perfil JURIDICO ou ADMIN
- Cliente corporativo possui propostas

### Pós-condições (Sucesso)
- Lista de propostas exibida
- Documentos disponíveis para download

### Fluxo Principal

1. Usuário acessa Portal PJ
2. Usuário seleciona filtros (produto, convênio, CNPJ)
3. Sistema valida permissões (role JURIDICO)
4. Sistema consulta propostas no banco
5. Para cada proposta, sistema busca documentos no Alfresco
6. Sistema monta DTOs com links de download
7. Sistema retorna lista paginada
8. Frontend exibe propostas em tabela

### Fluxos Alternativos

**FA-1: Nenhuma Proposta Encontrada**
- 4a. Query retorna vazio
- 4b. Sistema retorna 200 OK com array vazio
- 4c. Frontend exibe mensagem "Nenhuma proposta encontrada"
- Fim do caso de uso

**FA-2: Alfresco Indisponível**
- 5a. CMIS falha ao buscar documentos
- 5b. Sistema retorna propostas sem lista de documentos
- 5c. Frontend exibe warning "Documentos temporariamente indisponíveis"
- Continua em passo 7

### Regras de Negócio

- **RN-038:** Apenas usuários JURIDICO podem acessar Portal PJ
- **RN-039:** Resultados limitados a 100 por página
- **RN-040:** Cache de 5 minutos para consultas idênticas
- **RN-041:** Logs de acesso devem ser registrados (auditoria)

---

## UC-027: Autenticar Usuário

### Descrição
Usuário realiza login no sistema via Azure AD OAuth 2.0.

### Atores
- **Principal:** Usuário
- **Secundário:** Azure AD, Sistema App View

### Pré-condições
- Usuário possui conta corporativa Azure AD
- Usuário está cadastrado no sistema ou será criado automaticamente

### Pós-condições (Sucesso)
- Usuário autenticado
- JWT token gerado
- Sessão criada
- Redirecionado para dashboard

### Fluxo Principal

1. Usuário acessa aplicação
2. Sistema redireciona para Azure AD
3. Usuário insere credenciais corporativas
4. Azure AD valida credenciais
5. Azure AD redireciona com authorization code
6. Sistema troca code por access_token e refresh_token
7. Sistema valida JWT e extrai claims (email, nome, roles)
8. Sistema busca usuário por email
9. Se não existe, sistema cria usuário automaticamente
10. Sistema cria sessão em `tb_sessao`
11. Sistema retorna tokens para frontend
12. Frontend armazena tokens e redireciona para dashboard

### Fluxos Alternativos

**FA-1: Usuário Não Cadastrado**
- 8a. Email não encontrado em `tb_usuario`
- 8b. Sistema verifica claim `roles` do Azure AD
- 8c. Sistema mapeia roles para perfil (ex: grupo AD "Operadores" → perfil OPERADOR)
- 8d. Sistema cria usuário com perfil apropriado
- Continua em passo 10

**FA-2: Token Expirado**
- Sistema frontend detecta token expirado
- Frontend envia refresh_token para `/api/auth/refresh`
- Sistema valida refresh_token
- Sistema gera novo access_token
- Frontend atualiza localStorage

### Regras de Negócio

- **RN-042:** Apenas usuários com conta Azure AD podem acessar
- **RN-043:** Sessões expiram após 8 horas de inatividade
- **RN-044:** Refresh tokens expiram após 30 dias
- **RN-045:** Usuários inativos (90 dias sem login) são desabilitados automaticamente
- **RN-046:** Primeiro login força alteração de senha (se aplicável)

---

## UC-032: Upload de Documento Genérico

### Descrição
Upload manual de documento avulso no sistema (não vinculado a sinistro ou cliente específico).

### Atores
- **Principal:** Usuário Operador
- **Secundário:** Alfresco ECM

### Pré-condições
- Usuário autenticado
- Usuário possui permissão `documento:create`

### Pós-condições (Sucesso)
- Documento armazenado no Alfresco
- Metadata registrado no banco
- Link de download gerado

### Fluxo Principal

1. Usuário acessa tela de upload
2. Usuário seleciona arquivo e preenche metadata
3. Usuário clica em "Enviar"
4. Sistema valida tipo de arquivo
5. Sistema valida tamanho (máx 10MB)
6. Sistema cria estrutura de pastas no Alfresco
7. Sistema faz upload via CMIS
8. Sistema registra em `tb_documento`
9. Sistema retorna link de download
10. Frontend exibe confirmação

### Fluxos Alternativos

**FA-1: Arquivo Muito Grande**
- 5a. Arquivo excede 10MB
- 5b. Sistema retorna 413 Payload Too Large
- 5c. Frontend exibe erro "Arquivo muito grande"
- Fim do caso de uso

**FA-2: Tipo de Arquivo Não Suportado**
- 4a. Tipo não é PDF, PNG, JPG ou TIFF
- 4b. Sistema retorna 400 Bad Request
- Fim do caso de uso

### Regras de Negócio

- **RN-047:** Tipos permitidos: PDF, PNG, JPG, JPEG, TIFF
- **RN-048:** Tamanho máximo: 10MB
- **RN-049:** Nome do arquivo não pode conter: `/ \ : * ? " < > |`
- **RN-050:** Usuário CONSULTA não pode fazer upload

---

## UC-035: Gerar Dashboard Operacional

### Descrição
Sistema gera dashboard com métricas operacionais das integrações.

### Atores
- **Principal:** Usuário Gestor
- **Secundário:** PostgreSQL, Azure Application Insights

### Pré-condições
- Usuário autenticado
- Usuário possui permissão `relatorio:read`

### Pós-condições (Sucesso)
- Dashboard renderizado com métricas atualizadas
- Gráficos exibidos

### Fluxo Principal

1. Usuário acessa `/dashboard`
2. Frontend solicita GET `/api/relatorios/dashboard?periodo=ULTIMOS_7_DIAS`
3. Sistema verifica cache Redis (TTL 5 minutos)
4. Se cache miss: sistema executa queries agregadas
5. Sistema busca em `mv_dashboard_metricas` (materialized view)
6. Sistema calcula:
   - Total de transações
   - Taxa de sucesso por serviço
   - Latência P95
   - Throughput
7. Sistema armazena em cache
8. Sistema retorna JSON com métricas
9. Frontend renderiza gráficos (Recharts)

### Fluxos Alternativos

**FA-1: Cache Hit**
- 3a. Dados encontrados em cache Redis
- 3b. Sistema retorna imediatamente
- Pula passos 4-7

### Regras de Negócio

- **RN-051:** Dados são atualizados a cada 5 minutos
- **RN-052:** Materialized view é refreshed a cada 5 minutos
- **RN-053:** Máximo de 90 dias de histórico

---

## UC-037: Consultar Logs de Auditoria

### Descrição
Auditoria completa de operações realizadas no sistema.

### Atores
- **Principal:** Usuário Auditor
- **Secundário:** PostgreSQL

### Pré-condições
- Usuário autenticado
- Usuário possui perfil ADMIN

### Pós-condições (Sucesso)
- Logs exibidos com filtros aplicados
- Exportação disponível (Excel/CSV)

### Fluxo Principal

1. Auditor acessa `/auditoria`
2. Auditor aplica filtros (período, serviço, usuário, status)
3. Sistema valida permissões (role ADMIN)
4. Sistema consulta `tb_log_transacao` com filtros
5. Sistema retorna lista paginada (50 itens por página)
6. Frontend exibe logs em tabela
7. Auditor pode clicar em log para ver detalhes (JSON request/response)

### Regras de Negócio

- **RN-054:** Apenas ADMIN pode acessar logs de auditoria
- **RN-055:** Logs são imutáveis (não podem ser alterados ou deletados)
- **RN-056:** Dados sensíveis são mascarados na exibição (CPF, senhas, etc.)
- **RN-057:** Exportação limitada a 10.000 registros

---

## Resumo de Casos de Uso

| Módulo | Total CUs | Criticidade Alta | Criticidade Média |
|--------|-----------|------------------|-------------------|
| Sinistro | 7 | 5 | 2 |
| Cliente | 4 | 3 | 1 |
| Contrato | 5 | 4 | 1 |
| Comunicação | 3 | 2 | 1 |
| Perícia | 4 | 2 | 2 |
| Portal | 3 | 2 | 1 |
| IAM | 5 | 5 | 0 |
| Documento | 3 | 1 | 2 |
| Relatórios | 3 | 1 | 2 |
| **Total** | **37** | **25** | **12** |

---

## Matriz de Rastreabilidade

| Caso de Uso | Requisito Funcional | Regras de Negócio | Endpoints API |
|-------------|---------------------|-------------------|---------------|
| UC-001 | RF-001: Recepção de sinistros | RN-001 a RN-005 | POST /api/sinistros |
| UC-002 | RF-002: Armazenamento ECM | RN-006 a RN-010 | (Event-driven) |
| UC-003 | RF-003: Processamento OCR | RN-011 a RN-014 | (Event-driven) |
| UC-004 | RF-004: Callback OCR | RN-015 a RN-019 | POST /api/jarvis/callback |
| UC-005 | RF-005: Roteamento BPM | RN-020 a RN-023 | (Event-driven) |
| UC-008 | RF-008: Recepção de clientes | RN-024 a RN-028 | POST /api/clientes |
| UC-012 | RF-012: Implantação BOD | RN-029 a RN-033 | POST /api/bod/implantacao |
| UC-020 | RF-020: Agendamento perícia | RN-034 a RN-037 | POST /api/pericias |
| UC-024 | RF-024: Portal PJ | RN-038 a RN-041 | GET /api/portal/propostas |
| UC-027 | RF-027: Autenticação | RN-042 a RN-046 | POST /api/auth/login |
| UC-035 | RF-035: Dashboards | RN-051 a RN-053 | GET /api/relatorios/dashboard |

---

**Documento elaborado por:** Equipe de Produto e Arquitetura  
**Última atualização:** Janeiro de 2026
