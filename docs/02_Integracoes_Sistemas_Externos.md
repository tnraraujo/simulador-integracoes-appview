# Documentação de Integrações com Sistemas Externos
**App View v2.0 - Sistema de Gerenciamento de Documentos**

**Versão:** 2.0.0  
**Data:** Janeiro de 2026  

---

## 1. Visão Geral das Integrações

O App View integra com 7 sistemas externos principais e 1 sistema de armazenamento (ECM). Todas as integrações seguem padrões de resiliência com Circuit Breaker, Retry e Timeout.

### Mapa de Integrações

| Sistema | Tipo | Protocolo | Autenticação | SLA | Criticidade |
|---------|------|-----------|--------------|-----|-------------|
| **Alfresco ECM** | Síncrona | CMIS/HTTP | Basic Auth | 99.9% | Crítica |
| **Jarvis** | Assíncrona | REST/HTTPS | API Key | 99.5% | Alta |
| **Pega BPM** | Síncrona | REST/HTTPS | Basic Auth | 99.5% | Crítica |
| **CCM** | Síncrona | REST/HTTPS | Basic Auth | 99.0% | Alta |
| **Prestador** | Síncrona | REST/HTTPS | Basic Auth | 98.0% | Média |
| **Portal PJ** | Síncrona | CMIS/HTTP | CMIS Session | 99.0% | Alta |
| **BOD Seguros** | Síncrona | CMIS/HTTP | CMIS Session | 99.0% | Alta |
| **PostgreSQL** | Síncrona | JDBC | User/Password | 99.9% | Crítica |

---

## 2. Alfresco ECM (Enterprise Content Management)

### 2.1 Descrição

Sistema de gestão de conteúdo empresarial responsável por armazenar todos os documentos de sinistros, clientes e contratos.

### 2.2 Informações de Conexão

**Protocolo:** CMIS (Content Management Interoperability Services) 1.1  
**Binding:** AtomPub  

**Ambientes:**

| Ambiente | URL | Porta |
|----------|-----|-------|
| DEV | http://zs-alfresco-dev.azure.com | 8080 |
| UAT | http://zs-alfresco-uat.azure.com | 8080 |
| PRD | http://10.0.12.148 | 8080 |

**Endpoint CMIS:**
```
[HOST]:[PORT]/alfresco/api/-default-/public/cmis/versions/1.1/atom
```

### 2.3 Autenticação

**Tipo:** Basic Authentication

**Credenciais:** Armazenadas no Azure Key Vault
- Secret Name: `alfresco-username`
- Secret Name: `alfresco-password`

**Sessão CMIS:**
```java
SessionFactory factory = SessionFactoryImpl.newInstance();
Map<String, String> parameters = new HashMap<>();
parameters.put(SessionParameter.USER, username);
parameters.put(SessionParameter.PASSWORD, password);
parameters.put(SessionParameter.ATOMPUB_URL, cmisUrl);
parameters.put(SessionParameter.BINDING_TYPE, BindingType.ATOMPUB.value());

Session session = factory.createSession(parameters);
```

### 2.4 Operações Suportadas

#### 2.4.1 Upload de Documento

**Método CMIS:** `createDocument()`

**Parâmetros:**
- `folderId` - ID da pasta destino
- `name` - Nome do arquivo
- `contentStream` - Conteúdo do arquivo
- `properties` - Metadados customizados
- `versioningState` - MAJOR, MINOR ou NONE

**Estrutura de Pastas:**

**Sinistros:**
```
/Sites/prod-sinistros/documentLibrary/[FOLDER_ID]/YYYY/MM/{numero_sinistro}/
```

**Clientes:**
```
/Sites/prod-sinistros/documentLibrary/[FOLDER_ID]/YYYY/MM/{id_cliente}/
```

**BOD Seguros:**
```
/Sites/bod-seguros/documentLibrary/[FOLDER_ID]/YYYY/MM/{numero_contrato}/
```

**Exemplo de Código:**
```java
Folder parentFolder = (Folder) session.getObjectByPath("/Sites/prod-sinistros/documentLibrary/2026/01/SIN123456");

Map<String, Object> properties = new HashMap<>();
properties.put(PropertyIds.NAME, "documento_sinistro.pdf");
properties.put(PropertyIds.OBJECT_TYPE_ID, "zsSinistros:documentos_Sinistros");
properties.put("zsSinistros:sinistros_NumeroSinistro", "SIN123456");
properties.put("zsSinistros:sinistros_TipoDocumento", "LAUDO_MEDICO");

ContentStream contentStream = new ContentStreamImpl(
    "documento_sinistro.pdf",
    BigInteger.valueOf(fileBytes.length),
    "application/pdf",
    new ByteArrayInputStream(fileBytes)
);

Document doc = parentFolder.createDocument(properties, contentStream, VersioningState.MAJOR);
```

#### 2.4.2 Consulta de Documentos

**Método CMIS:** `query()`

**CMIS Query Language (Similar a SQL):**
```sql
SELECT * FROM zsSinistros:documentos_Sinistros 
WHERE zsSinistros:sinistros_NumeroSinistro = 'SIN123456'
ORDER BY cmis:creationDate DESC
```

**Exemplo de Código:**
```java
String query = "SELECT * FROM zsSinistros:documentos_Sinistros " +
               "WHERE zsSinistros:sinistros_NumeroSinistro = 'SIN123456'";

ItemIterable<QueryResult> results = session.query(query, false);

for (QueryResult result : results) {
    String docId = result.getPropertyValueById(PropertyIds.OBJECT_ID);
    String nome = result.getPropertyValueById(PropertyIds.NAME);
}
```

#### 2.4.3 Download de Documento

**Método CMIS:** `getContentStream()`

**Exemplo:**
```java
Document doc = (Document) session.getObject(documentId);
ContentStream stream = doc.getContentStream();

byte[] content = IOUtils.toByteArray(stream.getStream());
```

#### 2.4.4 Atualização de Propriedades

**Método CMIS:** `updateProperties()`

**Exemplo (Atualizar scores OCR):**
```java
Map<String, Object> properties = new HashMap<>();
properties.put("zsSinistros:sinistros_legibilidade", "85");
properties.put("zsSinistros:sinistros_acuracidade", "92");
properties.put("zsSinistros:sinistros_match_documento", "78");

Document doc = (Document) session.getObject(documentId);
doc.updateProperties(properties);
```

### 2.5 Tipos de Documentos Customizados

#### zsSinistros:documentos_Sinistros

**Propriedades:**
```
- sinistros_NumeroSinistro (String, obrigatório)
- sinistros_TipoDocumento (String, obrigatório)
- sinistros_Canal (String)
- sinistros_LossNumber (String)
- sinistros_LossSequence (String)
- sinistros_LossYear (Integer)
- sinistros_Branch (String)
- sinistros_sequencial (String)
- sinistros_DataRecebimento (Date)
- sinistros_Apolice (String)
- sinistros_Ramo (String)
- sinistros_Certificado (String)
- sinistros_idCliente (String)
- sinistros_DescricaoDocumento (String)
- sinistros_Publico (Boolean)
- sinistros_NomeArquivo (String)
- sinistros_ClaimID (String)
- sinistros_legibilidade (Integer) - Score OCR 0-100
- sinistros_acuracidade (Integer) - Score OCR 0-100
- sinistros_match_documento (Integer) - Score OCR 0-100
```

#### zsSinistros:documentos_Clientes

**Propriedades:**
```
- clientes_sequencial (String, obrigatório)
- clientes_NumeroDocumento (String, obrigatório)
- clientes_IdCliente (String, obrigatório)
- clientes_TipoDocumento (String, obrigatório)
- clientes_DataRecebimento (Date)
- clientes_DataExpedicao (Date)
- clientes_DataValidade (Date)
- clientes_ResponsavelValidacao (String)
- clientes_Canal (String)
- clientes_Validado (Boolean)
- clientes_bpo (String)
- clientes_legibilidade (Integer)
- clientes_acuracidade (Integer)
- clientes_match_documento (Integer)
- ClaimInfo_LossYear (Integer)
- ClaimInfo_Branch (String)
- ClaimInfo_LossNumber (String)
- ClaimInfo_LossSequence (String)
- ClaimInfo_ClaimID (String)
```

### 2.6 Políticas de Resiliência

**Circuit Breaker:**
```yaml
resilience4j:
  circuitbreaker:
    instances:
      alfresco:
        failure-rate-threshold: 50
        slow-call-rate-threshold: 70
        slow-call-duration-threshold: 10s
        wait-duration-in-open-state: 60s
        permitted-number-of-calls-in-half-open-state: 5
        sliding-window-size: 100
        minimum-number-of-calls: 10
```

**Retry:**
```yaml
resilience4j:
  retry:
    instances:
      alfresco:
        max-attempts: 3
        wait-duration: 1s
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2
```

**Timeout:** 30 segundos

### 2.7 Tratamento de Erros

| Erro CMIS | HTTP Status | Ação |
|-----------|-------------|------|
| `CmisObjectNotFoundException` | 404 | Retornar erro ao cliente |
| `CmisPermissionDeniedException` | 403 | Log de segurança, retornar 403 |
| `CmisStorageException` | 507 | Retry 3x, circuit breaker |
| `CmisConnectionException` | 503 | Retry 3x, circuit breaker |
| `CmisConstraintException` | 400 | Validação falhou, retornar erro |

**Exemplo de Tratamento:**
```java
@CircuitBreaker(name = "alfresco", fallbackMethod = "alfrescoFallback")
@Retry(name = "alfresco")
public Document uploadDocument(UploadRequest request) {
    try {
        return alfrescoSession.createDocument(...);
    } catch (CmisConnectionException e) {
        log.error("Erro de conexão com Alfresco", e);
        throw new AlfrescoIndisponivelException("ECM temporariamente indisponível", e);
    }
}

private Document alfrescoFallback(UploadRequest request, Exception e) {
    log.error("Circuit breaker ativado para Alfresco", e);
    // Enfileirar para processamento posterior
    serviceBus.send("alfresco-retry-queue", request);
    throw new AlfrescoCircuitAberto("ECM indisponível, documento será processado em breve");
}
```

---

## 3. Jarvis (OCR e Análise Cognitiva)

### 3.1 Descrição

Serviço interno de OCR e análise cognitiva de documentos. Processa documentos de forma assíncrona e retorna scores de qualidade.

### 3.2 Informações de Conexão

**Protocolo:** REST/HTTPS  
**Método:** POST (envio), POST (callback)  

**Ambientes:**

| Ambiente | URL Base |
|----------|----------|
| DEV | https://zx-api.zsbrasil.com |
| UAT | https://zu-api.zsbrasil.com |
| PRD | https://zs-api.zsbrasil.com |

**Endpoint:**
```
POST /api/jarvis/cognitive-services/v1/documents-reading
```

### 3.3 Autenticação

**Tipo:** API Key via Header

**Header:**
```
Ocp-Apim-Subscription-Key: <API_KEY>
```

**Credenciais:** Azure Key Vault
- Secret Name: `jarvis-api-key-{env}` (ex: `jarvis-api-key-prd`)

### 3.4 Request - Envio de Documento

**Content-Type:** `application/json`

**Payload:**
```json
{
  "documentCode": "alfresco-document-id",
  "documentIndex": 1,
  "sequential": "001",
  "channel": "WPC",
  "channelId": "WPC-001",
  "holderName": "João da Silva",
  "origin": "ECM",
  "claimId": "SIN123456",
  "documentBase64": "JVBERi0xLjQKJeLjz9MKMyAwIG9iago8PC9UeXBlIC9QYWdlCi9QYXJlbnQgMSAwIFIKL1Jlc291cmNlcy..."
}
```

**Campos Obrigatórios:**
- `documentCode` - ID do documento no Alfresco
- `documentBase64` - Arquivo em base64
- `claimId` - Número do sinistro ou cliente

**Limites:**
- Tamanho máximo: 10MB (após base64)
- Formatos aceitos: PDF, PNG, JPG, TIFF
- Timeout: 60 segundos

### 3.5 Response - Aceite de Processamento

**Status:** 202 Accepted

**Payload:**
```json
{
  "requestId": "uuid-v4",
  "status": "PROCESSING",
  "estimatedCompletionTime": "2026-01-27T21:15:00Z"
}
```

### 3.6 Callback - Resultado da Análise

**Endpoint do App View:**
```
POST /api/jarvis/callback
```

**Payload Recebido:**
```json
{
  "documentCode": "alfresco-document-id",
  "claimId": "SIN123456",
  "status": "COMPLETED",
  "result": {
    "legibilidade": 85,
    "acuracidade": 92,
    "matchDocumento": 78,
    "extractedText": "Texto extraído do documento...",
    "confidence": 0.92,
    "documentType": "LAUDO_MEDICO",
    "fields": {
      "nome": "João da Silva",
      "cpf": "123.456.789-00",
      "data": "2026-01-15"
    }
  },
  "processedAt": "2026-01-27T21:14:35Z",
  "processingDurationMs": 12450
}
```

**Status Possíveis:**
- `COMPLETED` - Processamento concluído com sucesso
- `FAILED` - Falha no processamento
- `POOR_QUALITY` - Documento de baixa qualidade (legibilidade < 50)
- `TIMEOUT` - Timeout no processamento

### 3.7 Políticas de Resiliência

**Circuit Breaker:**
```yaml
resilience4j:
  circuitbreaker:
    instances:
      jarvis:
        failure-rate-threshold: 40
        slow-call-duration-threshold: 30s
        wait-duration-in-open-state: 120s
        sliding-window-size: 50
```

**Retry:**
- Apenas em caso de erro de rede (não em 4xx)
- Máximo: 2 tentativas
- Delay: 5 segundos (fixo)

**Timeout:** 60 segundos

### 3.8 Tratamento de Erros

| Status HTTP | Descrição | Ação do App View |
|-------------|-----------|------------------|
| 202 | Aceito para processamento | Aguardar callback |
| 400 | Payload inválido | Retornar erro ao cliente, não fazer retry |
| 401 | API Key inválida | Alert crítico, verificar Key Vault |
| 413 | Arquivo muito grande | Retornar erro ao cliente |
| 429 | Rate limit excedido | Retry após 60s |
| 500 | Erro interno Jarvis | Retry 2x, enfileirar se falhar |
| 503 | Jarvis indisponível | Circuit breaker, enfileirar |

**Exemplo de Implementação:**
```java
@CircuitBreaker(name = "jarvis", fallbackMethod = "jarvisFallback")
@Retry(name = "jarvis", fallbackMethod = "jarvisRetryExhausted")
public JarvisResponse enviarDocumento(EnviarDocumentoRequest request) {
    HttpHeaders headers = new HttpHeaders();
    headers.set("Ocp-Apim-Subscription-Key", jarvisApiKey);
    headers.setContentType(MediaType.APPLICATION_JSON);
    
    JarvisPayload payload = mapToJarvisPayload(request);
    
    ResponseEntity<JarvisResponse> response = restTemplate.postForEntity(
        jarvisUrl + "/documents-reading",
        new HttpEntity<>(payload, headers),
        JarvisResponse.class
    );
    
    return response.getBody();
}

private JarvisResponse jarvisFallback(EnviarDocumentoRequest request, Exception e) {
    log.warn("Circuit breaker ativado para Jarvis, enfileirando documento", e);
    serviceBus.sendMessage("jarvis-retry-queue", request);
    return JarvisResponse.enfileirado();
}
```

### 3.9 Monitoramento

**Métricas:**
- `jarvis.requests.total` - Total de requisições
- `jarvis.requests.success` - Requisições bem-sucedidas (202)
- `jarvis.requests.failed` - Requisições falhadas
- `jarvis.callback.received` - Callbacks recebidos
- `jarvis.processing.duration` - Tempo de processamento
- `jarvis.ocr.quality.legibilidade` - Histogram de scores
- `jarvis.circuit_breaker.state` - Estado do circuit breaker

**Alertas:**
- Callback não recebido em 5 minutos → Warning
- Circuit breaker aberto → Critical
- Taxa de falha > 10% → Warning
- Score médio de legibilidade < 60 → Info

---

## 4. Pega BPM (Business Process Management)

### 4.1 Descrição

Sistema de orquestração de processos de negócio (BPM) que gerencia workflows de sinistros e clientes.

### 4.2 Informações de Conexão

**Protocolo:** REST/HTTPS  

**Ambientes:**

| Ambiente | URL Base |
|----------|----------|
| DEV | https://pega-dev.zsbrasil.com |
| UAT | https://pega-uat.zsbrasil.com |
| PRD | https://pega.zsbrasil.com |

### 4.3 Autenticação

**Tipo:** Basic Authentication ou OAuth 2.0

**Basic Auth:**
- Username/Password armazenados no Azure Key Vault
- Header: `Authorization: Basic base64(username:password)`

**OAuth 2.0 (Recomendado para PRD):**
```
POST /oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&client_id={client_id}&client_secret={client_secret}
```

### 4.4 Endpoints

#### 4.4.1 Criar Caso de Sinistro

**Endpoint:**
```
POST /api/pega/v1/cases/sinistro
```

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
  "documentoAlfresco": {
    "documentId": "workspace://SpacesStore/abc-123-def",
    "documentPath": "/2026/01/SIN123456/laudo.pdf"
  },
  "metadata": {
    "apolice": "POL-2026-001",
    "ramo": "AUTO",
    "certificado": "CERT-123"
  },
  "ocrScores": {
    "legibilidade": 85,
    "acuracidade": 92,
    "matchDocumento": 78
  }
}
```

**Response:**
```json
{
  "caseId": "PEGA-CASE-123456",
  "status": "CREATED",
  "workflowStarted": true,
  "nextStep": "ANALISE_DOCUMENTACAO",
  "createdAt": "2026-01-27T21:10:00Z"
}
```

#### 4.4.2 Atualizar Documento em Caso Existente

**Endpoint:**
```
PUT /api/pega/v1/cases/{caseId}/documents
```

**Request:**
```json
{
  "documentoAlfresco": {
    "documentId": "workspace://SpacesStore/xyz-789",
    "documentType": "COMPLEMENTO_MEDICO"
  },
  "ocrScores": {
    "legibilidade": 90,
    "acuracidade": 88
  }
}
```

**Response:**
```json
{
  "caseId": "PEGA-CASE-123456",
  "documentsCount": 3,
  "updatedAt": "2026-01-27T21:12:00Z"
}
```

#### 4.4.3 Consultar Status de Caso

**Endpoint:**
```
GET /api/pega/v1/cases/{caseId}
```

**Response:**
```json
{
  "caseId": "PEGA-CASE-123456",
  "numeroSinistro": "SIN123456",
  "status": "EM_ANALISE",
  "currentStep": "ANALISE_DOCUMENTACAO",
  "assignedTo": "analista.jose@zurich.com",
  "createdAt": "2026-01-27T21:10:00Z",
  "updatedAt": "2026-01-27T21:15:00Z",
  "sla": {
    "deadline": "2026-01-29T21:10:00Z",
    "remainingHours": 46
  }
}
```

### 4.5 Políticas de Resiliência

**Circuit Breaker:**
```yaml
resilience4j:
  circuitbreaker:
    instances:
      pega:
        failure-rate-threshold: 50
        wait-duration-in-open-state: 90s
        sliding-window-size: 100
```

**Retry:**
- Máximo: 3 tentativas
- Exponential backoff: 1s, 2s, 4s

**Timeout:** 15 segundos

### 4.6 Tratamento de Erros

| Status HTTP | Descrição | Ação |
|-------------|-----------|------|
| 201 | Caso criado | Sucesso |
| 200 | Caso atualizado | Sucesso |
| 400 | Dados inválidos | Retornar erro, não fazer retry |
| 404 | Caso não encontrado | Retornar erro |
| 409 | Caso duplicado | Idempotência, retornar caso existente |
| 500 | Erro interno Pega | Retry 3x, enfileirar se falhar |
| 503 | Pega indisponível | Circuit breaker, enfileirar |

---

## 5. CCM (Customer Communication Management)

### 5.1 Descrição

Sistema de gerenciamento de comunicações com clientes (emails, SMS, cartas).

### 5.2 Informações de Conexão

**Protocolo:** REST/HTTPS  

**Ambientes:**

| Ambiente | URL Base |
|----------|----------|
| DEV | https://ccm-dev.zsbrasil.com |
| UAT | https://ccm-uat.zsbrasil.com |
| PRD | https://ccm.zsbrasil.com |

### 5.3 Autenticação

**Tipo:** Basic Authentication

**Credenciais:** Azure Key Vault
- `ccm-username`
- `ccm-password`

### 5.4 Endpoints

#### 5.4.1 Criar Comunicação de Sinistro

**Endpoint:**
```
POST /api/ccm/v1/comunicacoes
```

**Request:**
```json
{
  "tipo": "SINISTRO",
  "numeroSinistro": "SIN123456",
  "destinatario": {
    "nome": "João da Silva",
    "cpfCnpj": "123.456.789-00",
    "email": "joao@example.com",
    "telefone": "+5511999998888"
  },
  "conteudo": {
    "template": "AGENDAMENTO_PERICIA",
    "parametros": {
      "nomeSegurado": "Maria da Silva",
      "numeroNotificacao": "NOT-12345",
      "nomeProduto": "Auto Standard",
      "etapaSinistro": "PERICIA_AGENDADA",
      "tipoAgendamento": "VISTORIA_VEICULAR",
      "dataAgendamento": "2026-02-10",
      "horaAgendamento": "14:00",
      "contatoVistoria": "Perito Paulo Santos",
      "foneVistoria": "+5511988887777",
      "urlPortal": "https://portal.zurichseguros.com.br/sinistro/SIN123456"
    }
  },
  "canais": ["EMAIL", "SMS"],
  "prioridade": "NORMAL",
  "agendarPara": "2026-01-28T09:00:00Z"
}
```

**Response:**
```json
{
  "comunicacaoId": "COM-789456",
  "status": "AGENDADA",
  "canaisAgendados": [
    {
      "canal": "EMAIL",
      "status": "AGENDADO",
      "envioEstimado": "2026-01-28T09:00:00Z"
    },
    {
      "canal": "SMS",
      "status": "AGENDADO",
      "envioEstimado": "2026-01-28T09:00:00Z"
    }
  ],
  "createdAt": "2026-01-27T21:10:00Z"
}
```

#### 5.4.2 Consultar Status de Comunicação

**Endpoint:**
```
GET /api/ccm/v1/comunicacoes/{comunicacaoId}
```

**Response:**
```json
{
  "comunicacaoId": "COM-789456",
  "status": "ENVIADA",
  "canais": [
    {
      "canal": "EMAIL",
      "status": "ENTREGUE",
      "enviadoEm": "2026-01-28T09:00:12Z",
      "entregueEm": "2026-01-28T09:00:45Z"
    },
    {
      "canal": "SMS",
      "status": "ENTREGUE",
      "enviadoEm": "2026-01-28T09:00:15Z",
      "entregueEm": "2026-01-28T09:00:18Z"
    }
  ]
}
```

### 5.5 Templates Disponíveis

| Template | Descrição | Canais |
|----------|-----------|--------|
| `AGENDAMENTO_PERICIA` | Notificação de perícia agendada | EMAIL, SMS |
| `DOCUMENTACAO_PENDENTE` | Solicitação de documentos | EMAIL, CARTA |
| `SINISTRO_APROVADO` | Aprovação de sinistro | EMAIL, SMS |
| `SINISTRO_RECUSADO` | Recusa de sinistro | EMAIL, CARTA |
| `PAGAMENTO_REALIZADO` | Confirmação de pagamento | EMAIL, SMS |

### 5.6 Políticas de Resiliência

**Circuit Breaker:**
```yaml
resilience4j:
  circuitbreaker:
    instances:
      ccm:
        failure-rate-threshold: 50
        wait-duration-in-open-state: 60s
```

**Retry:**
- Máximo: 3 tentativas
- Exponential backoff: 2s, 4s, 8s

**Timeout:** 10 segundos

### 5.7 Tratamento de Erros

| Status HTTP | Descrição | Ação |
|-------------|-----------|------|
| 201 | Comunicação criada | Sucesso |
| 400 | Template ou parâmetros inválidos | Retornar erro, logar |
| 404 | Template não encontrado | Retornar erro |
| 422 | Destinatário inválido (email/telefone) | Retornar erro |
| 500 | Erro interno CCM | Retry 3x, enfileirar |
| 503 | CCM indisponível | Circuit breaker, enfileirar |

---

## 6. Prestador (Sistema de Perícias)

### 6.1 Descrição

Sistema de gestão de prestadores de serviços e agendamento de perícias.

### 6.2 Informações de Conexão

**Protocolo:** REST/HTTPS  

**Ambientes:**

| Ambiente | URL Base |
|----------|----------|
| DEV | https://prestador-dev.zsbrasil.com |
| UAT | https://prestador-uat.zsbrasil.com |
| PRD | https://prestador.zsbrasil.com |

### 6.3 Autenticação

**Tipo:** Basic Authentication

### 6.4 Endpoints

#### 6.4.1 Criar Agendamento de Perícia

**Endpoint:**
```
POST /api/prestador/v1/pericias
```

**Request:**
```json
{
  "numeroSinistro": "SIN123456",
  "tipoPéricia": "VISTORIA_VEICULAR",
  "prestadorId": "PREST-001",
  "dataAgendamento": "2026-02-10T14:00:00Z",
  "local": {
    "endereco": "Rua das Flores, 123",
    "cidade": "São Paulo",
    "uf": "SP",
    "cep": "01234-567"
  },
  "contato": {
    "nome": "João da Silva",
    "telefone": "+5511999998888"
  },
  "observacoes": "Veículo disponível das 14h às 17h"
}
```

**Response:**
```json
{
  "periciaId": "PER-456789",
  "status": "AGENDADA",
  "prestador": {
    "nome": "Auto Vistoria SP",
    "telefone": "+5511988887777"
  },
  "dataAgendamento": "2026-02-10T14:00:00Z"
}
```

### 6.5 Políticas de Resiliência

**Timeout:** 10 segundos  
**Retry:** 2 tentativas com 3s de intervalo  

---

## 7. Portal PJ (Portal Pessoa Jurídica)

### 7.1 Descrição

Portal jurídico para acesso a documentos de propostas e portabilidades.

### 7.2 Operações

**Backend App View:**
- Consulta no Alfresco via CMIS
- Renderização de interface web
- Download de documentos

**Endpoints App View:**
```
GET /api/portal/propostas/{produto}/{convenio}/{cnpj}/{cpf}/{empresa}
GET /api/portal/portabilidades/{produto}/{convenio}/{cnpj}/{cpf}/{empresa}
GET /api/portal/documentos/{id}/download
```

**Integração:** Direta com Alfresco via CMIS (mesma sessão do App View)

---

## 8. BOD Seguros (Seguros Corporativos)

### 8.1 Descrição

Sistema de gestão de ciclo de vida de contratos corporativos (implantação, renovação, endosso, cancelamento, faturamento).

### 8.2 Operações

**Fluxos:**
- Implantação de contrato
- Renovação de contrato
- Endosso
- Cancelamento
- Faturamento
- Garantia de previdência

**Endpoints App View:**
```
POST /api/bod/implantacao
POST /api/bod/renovacao
POST /api/bod/endosso
POST /api/bod/cancelamento
POST /api/bod/faturamento
GET /api/bod/contratos/{numeroContrato}
```

**Armazenamento:**
- Documentos no Alfresco (CMIS)
- Metadados no PostgreSQL

---

## 9. PostgreSQL (Banco de Dados)

### 9.1 Informações de Conexão

**Versão:** PostgreSQL 15+

**Ambientes:**

| Ambiente | Host | Database |
|----------|------|----------|
| DEV | zs-postgresql-dev.postgres.database.azure.com | app_data_dev |
| UAT | zs-postgresql-uat.postgres.database.azure.com | app_data_uat |
| PRD | zs-postgresql-prd-ecmalfresco.postgres.database.azure.com | app_data |

**Porta:** 5432  
**SSL:** Obrigatório  

**Credenciais:** Azure Key Vault
- `postgresql-username-{env}`
- `postgresql-password-{env}`

### 9.2 Connection Pool (HikariCP)

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
```

### 9.3 Principais Tabelas

**Auditoria e Rastreamento:**
- `tb_log_transacao` - Log de todas as integrações
- `tb_ecm_jarvis` - Rastreamento de processamento OCR
- `tb_idempotency` - Controle de idempotência

**Domínio:**
- `tb_sinistro` - Sinistros
- `tb_cliente` - Clientes
- `tb_contrato` - Contratos BOD
- `tb_usuario` - Usuários
- `tb_perfil` - Perfis RBAC
- `tb_prestador` - Prestadores

---

## 10. Fluxos de Integração Detalhados

### 10.1 Fluxo Completo: Sinistro

```
1. WPC → App View (POST /api/sinistros)
   ├─ Validação de idempotência
   ├─ Validação de negócio
   └─ Evento: SinistroRecebido

2. Event Handler → ArmazenarSinistroNoECMUseCase
   ├─ App View → Alfresco (CMIS Upload)
   ├─ Criação de estrutura de pastas
   ├─ Upload com propriedades customizadas
   └─ Evento: SinistroArmazenado

3. Event Handler → EnviarSinistroParaJarvisUseCase
   ├─ App View → Azure Service Bus (jarvis-ocr-queue)
   └─ Retorno imediato ao WPC (202 Accepted)

4. Service Bus Listener → ProcessarDocumentoJarvisUseCase
   ├─ App View → Jarvis (POST /documents-reading)
   ├─ Jarvis processa assincronamente
   └─ Registro em tb_ecm_jarvis

5. Jarvis → App View (POST /api/jarvis/callback)
   ├─ Recebe scores OCR
   ├─ Evento: OCRConcluido
   └─ Event Handler → AtualizarPropriedadesAlfrescoUseCase

6. Event Handler → RotearParaPegaUseCase
   ├─ App View → Pega (POST /api/pega/v1/cases/sinistro)
   ├─ Workflow iniciado
   └─ Log em tb_log_transacao

7. (Opcional) Event Handler → NotificarCCMUseCase
   ├─ App View → CCM (POST /api/ccm/v1/comunicacoes)
   └─ Email/SMS enviado ao segurado
```

### 10.2 Tratamento de Erros em Cascata

**Cenário: Alfresco Indisponível**
```
1. Upload falha → CircuitBreaker abre
2. Request enfileirado em Azure Service Bus (alfresco-retry-queue)
3. Retorno ao cliente: 503 Service Unavailable
4. Service Bus: Retry automático após 1 minuto
5. Se Alfresco voltar: Processamento continua
6. Se falhar 3x: Dead Letter Queue → Dashboard de erros
```

**Cenário: Jarvis com Alta Latência**
```
1. Jarvis demora > 30s → Slow Call detectada
2. Circuit Breaker: Monitoramento de taxa de slow calls
3. Se > 70% slow calls: Circuit breaker abre
4. Requisições seguintes: Enfileiradas direto, sem chamada ao Jarvis
5. Após 2 minutos: Half-open state (testa 5 chamadas)
6. Se voltou ao normal: Circuit fecha
```

---

## 11. SLAs e Monitoramento

### 11.1 SLAs por Integração

| Sistema | Disponibilidade | Latência P95 | Retry Policy |
|---------|----------------|--------------|--------------|
| Alfresco | 99.9% | < 2s | 3x, exp backoff |
| Jarvis | 99.5% | < 30s (async) | 2x, fixed 5s |
| Pega | 99.5% | < 5s | 3x, exp backoff |
| CCM | 99.0% | < 3s | 3x, exp backoff |
| Prestador | 98.0% | < 5s | 2x, fixed 3s |

### 11.2 Métricas de Integração

**Para cada integração:**
- `{sistema}.requests.total` - Total de requisições
- `{sistema}.requests.success` - Requisições bem-sucedidas
- `{sistema}.requests.failed` - Requisições falhadas
- `{sistema}.duration` - Latência (histogram)
- `{sistema}.circuit_breaker.state` - Estado do circuit breaker (0=closed, 1=open, 0.5=half-open)
- `{sistema}.retry.count` - Número de retries

### 11.3 Alertas Configurados

**Crítico (PagerDuty):**
- Circuit breaker de Alfresco ou Pega aberto por > 5 minutos
- Taxa de falha de Jarvis > 20% em 15 minutos
- PostgreSQL indisponível

**Warning (Email):**
- Circuit breaker de CCM ou Prestador aberto
- Taxa de falha > 10% em qualquer integração
- Latência P95 > SLA por 10 minutos

**Info (Logs):**
- Circuit breaker em half-open state
- Retry executado

---

## 12. Segurança nas Integrações

### 12.1 Secrets Management

**Todos os secrets armazenados no Azure Key Vault:**
- API Keys (Jarvis)
- Credenciais Basic Auth (Pega, CCM, Prestador)
- Credenciais CMIS (Alfresco, Portal PJ)
- Connection string PostgreSQL

**Rotação:**
- Secrets rotacionados a cada 90 dias
- Notificação 15 dias antes da expiração
- Suporte a múltiplas versões durante rotação (zero downtime)

### 12.2 TLS/SSL

- **Obrigatório** para todas as integrações externas
- Mínimo: TLS 1.2
- Recomendado: TLS 1.3
- Certificate pinning para APIs críticas (Alfresco, Pega)

### 12.3 Rate Limiting

**Por Sistema Externo:**
- Alfresco: 200 req/min por instância
- Jarvis: 50 req/min (limite da API)
- Pega: 100 req/min
- CCM: 150 req/min

**Implementação:** Resilience4j RateLimiter

```java
@RateLimiter(name = "jarvis")
public JarvisResponse enviarDocumento(EnviarDocumentoRequest request) {
    // ...
}
```

### 12.4 Auditoria

**Todas as integrações logam:**
- Request payload (sanitizado, sem dados sensíveis)
- Response payload
- Headers (exceto Authorization)
- Latência
- Status code
- Correlation ID
- User ID (se aplicável)

**Armazenamento:**
- Logs estruturados em Azure Log Analytics
- Retenção: 90 dias (logs), 1 ano (auditoria)

---

## 13. Disaster Recovery

### 13.1 Estratégia por Sistema

**Alfresco:**
- Backup diário automático (Azure Backup)
- Recovery Point: 24 horas
- Recovery Time: 4 horas
- Plano B: Enfileirar uploads até recuperação

**Jarvis:**
- Processamento assíncrono via fila
- Retry automático quando voltar
- Sem perda de dados (documentos no Alfresco)

**Pega:**
- Enfileiramento de casos em Service Bus
- Processamento quando voltar
- SLA de processamento pode ser afetado

**PostgreSQL:**
- Azure Database for PostgreSQL (HA nativo)
- Geo-replication ativa
- Automatic failover < 2 minutos
- Point-in-time restore (7 dias)

### 13.2 Procedimentos de Fallback

**Se Alfresco indisponível:**
1. Aceitar upload do cliente
2. Armazenar temporariamente em Azure Blob Storage
3. Enfileirar em `alfresco-pending-queue`
4. Retornar 202 Accepted
5. Processar quando Alfresco voltar

**Se Jarvis indisponível:**
1. Armazenar documento no Alfresco
2. Enfileirar em `jarvis-ocr-queue` com delay
3. Retornar 202 Accepted
4. Notificar usuário quando processamento concluir

---

## 14. Testes de Integração

### 14.1 Testes de Contrato

**Pact/Spring Cloud Contract** para garantir compatibilidade:

```java
@AutoConfigureStubRunner(
    ids = "com.zurich:pega-api:+:stubs:8080",
    stubsMode = StubRunnerProperties.StubsMode.LOCAL
)
class PegaGatewayTest {
    
    @Test
    void deveCriarCasoSinistro() {
        // Given
        CriarCasoRequest request = new CriarCasoRequest(...);
        
        // When
        CriarCasoResponse response = pegaGateway.criarCaso(request);
        
        // Then
        assertThat(response.getCaseId()).isNotNull();
        assertThat(response.getStatus()).isEqualTo("CREATED");
    }
}
```

### 14.2 Testes com TestContainers

**PostgreSQL:**
```java
@Testcontainers
@SpringBootTest
class SinistroRepositoryTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine");
    
    @Test
    void deveSalvarSinistro() {
        // ...
    }
}
```

**Azure Service Bus (Emulador):**
```java
@Testcontainers
class ServiceBusIntegrationTest {
    
    @Container
    static GenericContainer<?> azurite = new GenericContainer<>("mcr.microsoft.com/azure-storage/azurite")
        .withExposedPorts(10000, 10001);
    
    @Test
    void deveEnviarMensagemParaFila() {
        // ...
    }
}
```

### 14.3 Testes E2E de Integração

**Cenário: Fluxo Completo de Sinistro**
```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@AutoConfigureWireMock // Mock de Jarvis, Pega, CCM
class FluxoSinistroE2ETest {
    
    @Test
    void deveProcessarSinistroCompleto() {
        // 1. Enviar sinistro
        // 2. Verificar upload Alfresco (mock)
        // 3. Simular callback Jarvis
        // 4. Verificar envio para Pega (mock)
        // 5. Validar log de auditoria
    }
}
```

---

## 15. Documentação de Referência

### 15.1 Links Úteis

- **Alfresco CMIS Spec**: https://docs.alfresco.com/content-services/latest/develop/api-reference/
- **OpenCMIS Documentation**: https://chemistry.apache.org/java/developing/guide.html
- **Resilience4j Guide**: https://resilience4j.readme.io/
- **Azure Service Bus**: https://learn.microsoft.com/azure/service-bus-messaging/

### 15.2 Diagramas de Sequência

Consulte documento separado: `03_Diagramas_Sequencia.md`

### 15.3 OpenAPI Specification

Consulte documento separado: `04_Catalogo_APIs.md`

---

**Documento elaborado por:** Equipe de Arquitetura  
**Última atualização:** Janeiro de 2026
