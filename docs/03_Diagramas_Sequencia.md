# Diagramas de Sequência - App View v2.0
**Fluxos Principais do Sistema**

**Versão:** 2.0.0  
**Data:** Janeiro de 2026  
**Formato:** Mermaid (UML)  

---

## Índice

1. [Fluxo 01: Recepção e Processamento de Sinistro](#fluxo-01-recepção-e-processamento-de-sinistro)
2. [Fluxo 02: Recepção e Processamento de Cliente](#fluxo-02-recepção-e-processamento-de-cliente)
3. [Fluxo 03: Callback de OCR do Jarvis](#fluxo-03-callback-de-ocr-do-jarvis)
4. [Fluxo 04: Roteamento para Pega BPM](#fluxo-04-roteamento-para-pega-bpm)
5. [Fluxo 05: Comunicação via CCM](#fluxo-05-comunicação-via-ccm)
6. [Fluxo 06: Agendamento de Perícia](#fluxo-06-agendamento-de-perícia)
7. [Fluxo 07: Portal PJ - Consulta de Propostas](#fluxo-07-portal-pj---consulta-de-propostas)
8. [Fluxo 08: BOD Seguros - Implantação de Contrato](#fluxo-08-bod-seguros---implantação-de-contrato)
9. [Fluxo 09: Autenticação OAuth 2.0](#fluxo-09-autenticação-oauth-20)
10. [Fluxo 10: Tratamento de Erro com Circuit Breaker](#fluxo-10-tratamento-de-erro-com-circuit-breaker)
11. [Fluxo 11: Processamento Assíncrono via Service Bus](#fluxo-11-processamento-assíncrono-via-service-bus)
12. [Fluxo 12: Geração de Relatórios e Dashboards](#fluxo-12-geração-de-relatórios-e-dashboards)

---

## Fluxo 01: Recepção e Processamento de Sinistro

**Descrição:** Fluxo completo desde a recepção de um documento de sinistro até o processamento OCR e roteamento para Pega.

```mermaid
sequenceDiagram
    participant WPC as Sistema WPC
    participant Controller as SinistroController
    participant UseCase1 as ReceberDocumentoUseCase
    participant Domain as Sinistro (Aggregate)
    participant EventBus as Spring EventBus
    participant Handler1 as SinistroRecebidoHandler
    participant UseCase2 as ArmazenarNoECMUseCase
    participant Alfresco as Alfresco ECM
    participant Handler2 as SinistroArmazenadoHandler
    participant Queue as Azure Service Bus
    participant DB as PostgreSQL

    WPC->>Controller: POST /api/sinistros<br/>{numeroSinistro, documento, metadata}
    activate Controller
    
    Controller->>Controller: Validar Idempotency-Key
    
    Controller->>UseCase1: executar(request)
    activate UseCase1
    
    UseCase1->>Domain: Sinistro.criar(numeroSinistro, tipo, canal)
    activate Domain
    Domain->>Domain: Validar invariantes
    Domain->>Domain: addEvent(SinistroRecebido)
    Domain-->>UseCase1: sinistro
    deactivate Domain
    
    UseCase1->>DB: repository.salvar(sinistro)
    activate DB
    DB-->>UseCase1: OK
    deactivate DB
    
    UseCase1->>EventBus: publishEvents(SinistroRecebido)
    activate EventBus
    deactivate UseCase1
    
    UseCase1-->>Controller: SinistroRecebidoResponse(id)
    Controller-->>WPC: 202 Accepted<br/>{sinistroId, status: RECEBIDO}
    deactivate Controller
    
    EventBus->>Handler1: onSinistroRecebido(event)
    deactivate EventBus
    activate Handler1
    
    Handler1->>UseCase2: executar(sinistroId)
    activate UseCase2
    
    UseCase2->>DB: repository.buscar(sinistroId)
    activate DB
    DB-->>UseCase2: sinistro
    deactivate DB
    
    UseCase2->>Alfresco: CMIS createDocument()<br/>tipo: documentos_Sinistros
    activate Alfresco
    Alfresco->>Alfresco: Criar estrutura de pastas<br/>/2026/01/SIN123456
    Alfresco-->>UseCase2: documentId
    deactivate Alfresco
    
    UseCase2->>Domain: sinistro.marcarComoArmazenado(documentId)
    activate Domain
    Domain->>Domain: addEvent(SinistroArmazenado)
    Domain-->>UseCase2: void
    deactivate Domain
    
    UseCase2->>DB: repository.atualizar(sinistro)
    activate DB
    DB-->>UseCase2: OK
    deactivate DB
    
    UseCase2->>EventBus: publishEvents(SinistroArmazenado)
    deactivate UseCase2
    deactivate Handler1
    
    EventBus->>Handler2: onSinistroArmazenado(event)
    activate Handler2
    
    Handler2->>Queue: sendMessage(jarvis-ocr-queue,<br/>{sinistroId, documentId})
    activate Queue
    Queue-->>Handler2: messageId
    deactivate Queue
    deactivate Handler2
    
    Note over Queue: Processamento assíncrono<br/>via Service Bus Listener
```

---

## Fluxo 02: Recepção e Processamento de Cliente

**Descrição:** Fluxo de recepção de documentos de cliente (RG, CPF, CNH) com validação OCR.

```mermaid
sequenceDiagram
    participant Pega as Sistema Pega
    participant Controller as ClienteController
    participant UseCase as ReceberDocumentoClienteUseCase
    participant Domain as Cliente (Aggregate)
    participant EventBus as Spring EventBus
    participant Handler as ClienteRecebidoHandler
    participant Alfresco as Alfresco ECM
    participant Queue as Azure Service Bus
    participant DB as PostgreSQL

    Pega->>Controller: POST /api/clientes<br/>{idCliente, tipoDoc, documento}
    activate Controller
    
    Controller->>Controller: Validar Idempotency-Key<br/>Validar Bean Validation
    
    Controller->>UseCase: executar(request)
    activate UseCase
    
    UseCase->>Domain: Cliente.criar(idCliente, numeroDoc, tipo)
    activate Domain
    Domain->>Domain: Validar CPF/CNPJ<br/>Validar tipo documento
    Domain->>Domain: addEvent(ClienteRecebido)
    Domain-->>UseCase: cliente
    deactivate Domain
    
    UseCase->>DB: repository.salvar(cliente)
    activate DB
    DB-->>UseCase: OK
    deactivate DB
    
    UseCase->>EventBus: publishEvents(ClienteRecebido)
    deactivate UseCase
    
    UseCase-->>Controller: ClienteRecebidoResponse
    Controller-->>Pega: 202 Accepted
    deactivate Controller
    
    EventBus->>Handler: onClienteRecebido(event)
    activate Handler
    
    Handler->>Alfresco: CMIS Upload<br/>tipo: documentos_Clientes
    activate Alfresco
    Alfresco-->>Handler: documentId
    deactivate Alfresco
    
    Handler->>Queue: sendMessage(jarvis-ocr-queue)
    activate Queue
    Queue-->>Handler: messageId
    deactivate Queue
    
    Handler->>DB: Atualizar cliente<br/>status: ENVIADO_PARA_OCR
    activate DB
    DB-->>Handler: OK
    deactivate DB
    deactivate Handler
```

---

## Fluxo 03: Callback de OCR do Jarvis

**Descrição:** Processamento do callback do Jarvis após conclusão da análise OCR.

```mermaid
sequenceDiagram
    participant Jarvis as Jarvis API
    participant Controller as JarvisCallbackController
    participant UseCase as ProcessarCallbackJarvisUseCase
    participant DB as PostgreSQL
    participant Domain as Sinistro
    participant Alfresco as Alfresco ECM
    participant EventBus as Spring EventBus
    participant Handler as OCRConcluidoHandler
    participant Pega as Pega Gateway

    Jarvis->>Controller: POST /api/jarvis/callback<br/>{documentCode, scores, status}
    activate Controller
    
    Controller->>Controller: Validar assinatura webhook
    
    Controller->>UseCase: executar(callback)
    activate UseCase
    
    UseCase->>DB: Buscar em tb_ecm_jarvis<br/>WHERE document_code = ?
    activate DB
    DB-->>UseCase: registro OCR
    deactivate DB
    
    UseCase->>DB: Buscar sinistro por documentId
    activate DB
    DB-->>UseCase: sinistro
    deactivate DB
    
    UseCase->>Domain: sinistro.atualizarScoresOCR(<br/>legibilidade, acuracidade, match)
    activate Domain
    Domain->>Domain: Validar scores (0-100)
    Domain->>Domain: addEvent(OCRConcluido)
    Domain-->>UseCase: void
    deactivate Domain
    
    UseCase->>Alfresco: updateProperties()<br/>sinistros_legibilidade = 85<br/>sinistros_acuracidade = 92
    activate Alfresco
    Alfresco-->>UseCase: OK
    deactivate Alfresco
    
    UseCase->>DB: UPDATE tb_ecm_jarvis<br/>SET status = 'COMPLETED'
    activate DB
    DB-->>UseCase: OK
    deactivate DB
    
    UseCase->>EventBus: publishEvents(OCRConcluido)
    deactivate UseCase
    
    UseCase-->>Controller: CallbackProcessadoResponse
    Controller-->>Jarvis: 200 OK
    deactivate Controller
    
    EventBus->>Handler: onOCRConcluido(event)
    activate Handler
    
    alt Scores aceitáveis (legibilidade > 60)
        Handler->>Pega: POST /api/pega/v1/cases/sinistro
        activate Pega
        Pega-->>Handler: caseId, status
        deactivate Pega
        
        Handler->>DB: INSERT tb_log_transacao<br/>servico: 'Pega Sinistro'
        activate DB
        DB-->>Handler: OK
        deactivate DB
    else Scores baixos
        Handler->>DB: Marcar para revisão manual
        activate DB
        DB-->>Handler: OK
        deactivate DB
    end
    deactivate Handler
```

---

## Fluxo 04: Roteamento para Pega BPM

**Descrição:** Criação de caso no Pega após validação OCR bem-sucedida.

```mermaid
sequenceDiagram
    participant Handler as OCRConcluidoHandler
    participant UseCase as RotearParaPegaUseCase
    participant DB as PostgreSQL
    participant Domain as Sinistro
    participant Gateway as PegaGateway
    participant CB as Circuit Breaker
    participant Pega as Pega API
    participant EventBus as Spring EventBus

    Handler->>UseCase: executar(sinistroId)
    activate UseCase
    
    UseCase->>DB: repository.buscar(sinistroId)
    activate DB
    DB-->>UseCase: sinistro
    deactivate DB
    
    UseCase->>UseCase: Montar payload Pega
    
    UseCase->>Gateway: criarCaso(request)
    activate Gateway
    
    Gateway->>CB: Verificar estado
    activate CB
    
    alt Circuit Breaker CLOSED
        CB-->>Gateway: OK para prosseguir
        deactivate CB
        
        Gateway->>Pega: POST /api/pega/v1/cases/sinistro
        activate Pega
        
        alt Sucesso
            Pega-->>Gateway: 201 Created<br/>{caseId, status}
            deactivate Pega
            
            Gateway-->>UseCase: PegaCaseResponse
            deactivate Gateway
            
            UseCase->>Domain: sinistro.marcarComoRoteado(pegaCaseId)
            activate Domain
            Domain->>Domain: addEvent(SinistroRoteadoParaPega)
            Domain-->>UseCase: void
            deactivate Domain
            
            UseCase->>DB: repository.atualizar(sinistro)
            activate DB
            DB-->>UseCase: OK
            deactivate DB
            
            UseCase->>DB: INSERT tb_log_transacao<br/>servico: 'Pega Sinistro'<br/>status: 1 (sucesso)
            activate DB
            DB-->>UseCase: OK
            deactivate DB
            
        else Erro 5xx
            Pega-->>Gateway: 500 Internal Error
            deactivate Pega
            Gateway->>Gateway: Registrar falha no CB
            Gateway-->>UseCase: PegaIndisponivelException
            deactivate Gateway
            
            UseCase->>DB: INSERT tb_log_transacao<br/>status: 2 (falha)
            activate DB
            DB-->>UseCase: OK
            deactivate DB
        end
        
    else Circuit Breaker OPEN
        CB-->>Gateway: Circuit Aberto
        deactivate CB
        Gateway->>Gateway: Fallback
        Gateway-->>UseCase: EnfileirarException
        deactivate Gateway
        
        UseCase->>Queue: sendMessage(pega-retry-queue)
        activate Queue
        Queue-->>UseCase: messageId
        deactivate Queue
    end
    
    UseCase->>EventBus: publishEvents(SinistroRoteadoParaPega)
    deactivate UseCase
```

---

## Fluxo 05: Comunicação via CCM

**Descrição:** Envio de comunicação (email/SMS) ao segurado via sistema CCM.

```mermaid
sequenceDiagram
    participant Handler as SinistroProcessadoHandler
    participant UseCase as NotificarSeguradoUseCase
    participant DB as PostgreSQL
    participant Gateway as CCMGateway
    participant CCM as CCM API
    participant EventBus as Spring EventBus

    Handler->>UseCase: executar(sinistroId, template)
    activate UseCase
    
    UseCase->>DB: Buscar dados do sinistro<br/>e segurado
    activate DB
    DB-->>UseCase: sinistro, segurado
    deactivate DB
    
    UseCase->>UseCase: Montar payload CCM<br/>com template e parâmetros
    
    UseCase->>Gateway: enviarComunicacao(request)
    activate Gateway
    
    Gateway->>CCM: POST /api/ccm/v1/comunicacoes
    activate CCM
    
    alt Sucesso
        CCM->>CCM: Agendar envio<br/>Validar destinatário
        CCM-->>Gateway: 201 Created<br/>{comunicacaoId, status}
        deactivate CCM
        
        Gateway-->>UseCase: ComunicacaoResponse
        deactivate Gateway
        
        UseCase->>DB: INSERT tb_log_transacao<br/>servico: 'CCM'<br/>status: 1
        activate DB
        DB-->>UseCase: OK
        deactivate DB
        
        UseCase->>EventBus: publishEvent(ComunicacaoAgendada)
        
    else Erro - Destinatário Inválido
        CCM-->>Gateway: 422 Unprocessable Entity
        deactivate CCM
        Gateway-->>UseCase: DestinatarioInvalidoException
        deactivate Gateway
        
        UseCase->>DB: Logar erro
        activate DB
        DB-->>UseCase: OK
        deactivate DB
    end
    
    deactivate UseCase
    
    Note over CCM: Processamento assíncrono<br/>Envio de email/SMS<br/>em background
```

---

## Fluxo 06: Agendamento de Perícia

**Descrição:** Agendamento de perícia com prestador após análise inicial do sinistro.

```mermaid
sequenceDiagram
    participant Analista as Usuário Analista
    participant Controller as PericiaController
    participant UseCase as AgendarPericiaUseCase
    participant Domain as Pericia (Aggregate)
    participant DB as PostgreSQL
    participant Gateway as PrestadorGateway
    participant Prestador as Sistema Prestador
    participant CCM as CCM Gateway

    Analista->>Controller: POST /api/pericias<br/>{sinistroId, prestadorId, data}
    activate Controller
    
    Controller->>UseCase: executar(request)
    activate UseCase
    
    UseCase->>DB: Buscar sinistro
    activate DB
    DB-->>UseCase: sinistro
    deactivate DB
    
    UseCase->>DB: Buscar prestador
    activate DB
    DB-->>UseCase: prestador
    deactivate DB
    
    UseCase->>Domain: Pericia.agendar(<br/>sinistro, prestador, data)
    activate Domain
    Domain->>Domain: Validar disponibilidade<br/>Validar prazo SLA
    Domain->>Domain: addEvent(PericiaAgendada)
    Domain-->>UseCase: pericia
    deactivate Domain
    
    UseCase->>DB: repository.salvar(pericia)
    activate DB
    DB-->>UseCase: OK
    deactivate DB
    
    UseCase->>Gateway: notificarPrestador(pericia)
    activate Gateway
    
    Gateway->>Prestador: POST /api/prestador/v1/pericias
    activate Prestador
    Prestador-->>Gateway: 201 Created
    deactivate Prestador
    
    Gateway-->>UseCase: confirmacao
    deactivate Gateway
    
    UseCase->>CCM: Notificar segurado<br/>template: AGENDAMENTO_PERICIA
    activate CCM
    CCM-->>UseCase: comunicacaoId
    deactivate CCM
    
    UseCase->>EventBus: publishEvents(PericiaAgendada)
    deactivate UseCase
    
    UseCase-->>Controller: AgendarPericiaResponse
    Controller-->>Analista: 201 Created<br/>{periciaId, prestador, data}
    deactivate Controller
```

---

## Fluxo 07: Portal PJ - Consulta de Propostas

**Descrição:** Acesso jurídico a propostas e documentação via Portal PJ.

```mermaid
sequenceDiagram
    participant Usuario as Usuário Jurídico
    participant Frontend as React Frontend
    participant Controller as PortalController
    participant UseCase as ConsultarPropostaUseCase
    participant Security as Spring Security
    participant DB as PostgreSQL
    participant Alfresco as Alfresco ECM

    Usuario->>Frontend: Acessar /portal/propostas
    activate Frontend
    
    Frontend->>Controller: GET /api/portal/propostas<br/>?produto=VIDA&cnpj=12345678000100
    activate Controller
    
    Controller->>Security: Validar JWT Token
    activate Security
    Security->>Security: Verificar role JURIDICO
    Security-->>Controller: Autorizado
    deactivate Security
    
    Controller->>UseCase: executar(filtros)
    activate UseCase
    
    UseCase->>DB: Query propostas<br/>WHERE cnpj = ?
    activate DB
    DB-->>UseCase: List<Proposta>
    deactivate DB
    
    loop Para cada proposta
        UseCase->>Alfresco: Query CMIS<br/>documentos da proposta
        activate Alfresco
        Alfresco-->>UseCase: List<Document>
        deactivate Alfresco
    end
    
    UseCase->>UseCase: Montar PropostaDTO<br/>com links de download
    
    UseCase-->>Controller: List<PropostaDTO>
    deactivate UseCase
    
    Controller-->>Frontend: 200 OK<br/>[{proposta, documentos[]}]
    deactivate Controller
    
    Frontend->>Frontend: Renderizar lista
    Frontend-->>Usuario: Exibir propostas
    deactivate Frontend
    
    Usuario->>Frontend: Clicar em "Baixar Contrato"
    activate Frontend
    
    Frontend->>Controller: GET /api/portal/documentos/{id}/download
    activate Controller
    
    Controller->>Alfresco: getContentStream(documentId)
    activate Alfresco
    Alfresco-->>Controller: InputStream
    deactivate Alfresco
    
    Controller-->>Frontend: 200 OK<br/>Content-Type: application/pdf
    deactivate Controller
    
    Frontend-->>Usuario: Download arquivo
    deactivate Frontend
```

---

## Fluxo 08: BOD Seguros - Implantação de Contrato

**Descrição:** Implantação de novo contrato corporativo no sistema BOD Seguros.

```mermaid
sequenceDiagram
    participant Sistema as Sistema Externo
    participant Controller as ContratoController
    participant UseCase as ImplantarContratoUseCase
    participant Domain as Contrato (Aggregate)
    participant DB as PostgreSQL
    participant Alfresco as Alfresco ECM
    participant EventBus as Spring EventBus

    Sistema->>Controller: POST /api/bod/implantacao<br/>{numeroContrato, arquivo, metadata}
    activate Controller
    
    Controller->>Controller: Validar Idempotency-Key
    
    Controller->>UseCase: executar(request)
    activate UseCase
    
    UseCase->>Domain: Contrato.implantar(<br/>numero, tipo, vigencia)
    activate Domain
    Domain->>Domain: Validar número único<br/>Validar datas de vigência
    Domain->>Domain: addEvent(ContratoImplantado)
    Domain-->>UseCase: contrato
    deactivate Domain
    
    UseCase->>DB: Verificar duplicação<br/>SELECT WHERE numeroContrato = ?
    activate DB
    DB-->>UseCase: null (não existe)
    deactivate DB
    
    UseCase->>Alfresco: CMIS Upload<br/>/bod-seguros/2026/01/{numero}
    activate Alfresco
    Alfresco->>Alfresco: Criar pasta do contrato
    Alfresco-->>UseCase: folderObjectId
    deactivate Alfresco
    
    UseCase->>DB: repository.salvar(contrato)
    activate DB
    DB-->>UseCase: OK
    deactivate DB
    
    UseCase->>DB: INSERT tb_log_transacao<br/>servico: 'BOD Implantacao'
    activate DB
    DB-->>UseCase: OK
    deactivate DB
    
    UseCase->>EventBus: publishEvents(ContratoImplantado)
    deactivate UseCase
    
    UseCase-->>Controller: ImplantacaoResponse
    Controller-->>Sistema: 201 Created<br/>{contratoId, status: ATIVO}
    deactivate Controller
```

---

## Fluxo 09: Autenticação OAuth 2.0

**Descrição:** Fluxo de autenticação via Azure AD com JWT.

```mermaid
sequenceDiagram
    participant Usuario as Usuário
    participant Frontend as React Frontend
    participant AppView as App View Backend
    participant AzureAD as Azure AD
    participant DB as PostgreSQL

    Usuario->>Frontend: Acessar aplicação
    activate Frontend
    
    Frontend->>Frontend: Verificar token localStorage
    
    alt Token não existe ou expirado
        Frontend->>AzureAD: Redirecionar para /oauth/authorize
        activate AzureAD
        
        AzureAD->>Usuario: Exibir tela de login
        Usuario->>AzureAD: Credenciais corporativas
        
        AzureAD->>AzureAD: Validar credenciais<br/>Gerar authorization code
        
        AzureAD-->>Frontend: Redirect com code
        deactivate AzureAD
        
        Frontend->>AppView: POST /api/auth/callback<br/>{code}
        activate AppView
        
        AppView->>AzureAD: POST /oauth/token<br/>{code, client_id, client_secret}
        activate AzureAD
        AzureAD-->>AppView: {access_token, refresh_token}
        deactivate AzureAD
        
        AppView->>AppView: Validar JWT<br/>Extrair claims (email, roles)
        
        AppView->>DB: Buscar ou criar usuário<br/>WHERE email = ?
        activate DB
        DB-->>AppView: usuario
        deactivate DB
        
        AppView->>DB: INSERT tb_sessao<br/>{userId, token, expiresAt}
        activate DB
        DB-->>AppView: OK
        deactivate DB
        
        AppView-->>Frontend: {accessToken, refreshToken, user}
        deactivate AppView
        
        Frontend->>Frontend: Armazenar em localStorage
    end
    
    Frontend->>AppView: GET /api/sinistros<br/>Authorization: Bearer {token}
    activate AppView
    
    AppView->>AppView: Validar JWT signature<br/>Verificar expiration
    
    AppView-->>Frontend: 200 OK + dados
    deactivate AppView
    
    Frontend-->>Usuario: Exibir dashboard
    deactivate Frontend
```

---

## Fluxo 10: Tratamento de Erro com Circuit Breaker

**Descrição:** Comportamento do circuit breaker em caso de falhas sucessivas.

```mermaid
sequenceDiagram
    participant UseCase as EnviarParaJarvisUseCase
    participant Gateway as JarvisGateway
    participant CB as Circuit Breaker
    participant Jarvis as Jarvis API
    participant Queue as Azure Service Bus
    participant Metrics as Micrometer Metrics

    Note over CB: Estado: CLOSED

    loop 10 chamadas bem-sucedidas
        UseCase->>Gateway: enviarDocumento(request)
        activate Gateway
        Gateway->>CB: check()
        activate CB
        CB-->>Gateway: OK (closed)
        deactivate CB
        Gateway->>Jarvis: POST /documents-reading
        activate Jarvis
        Jarvis-->>Gateway: 202 Accepted
        deactivate Jarvis
        Gateway-->>UseCase: Success
        deactivate Gateway
    end
    
    Note over Jarvis: Jarvis começa a falhar

    loop 6 chamadas com falha (>50%)
        UseCase->>Gateway: enviarDocumento(request)
        activate Gateway
        Gateway->>CB: check()
        activate CB
        CB-->>Gateway: OK (ainda closed)
        deactivate CB
        Gateway->>Jarvis: POST /documents-reading
        activate Jarvis
        Jarvis-->>Gateway: 503 Service Unavailable
        deactivate Jarvis
        Gateway->>CB: recordFailure()
        activate CB
        CB->>CB: failureRate = 60%
        CB->>CB: OPEN circuit!
        CB->>Metrics: circuit_breaker.state = 1
        CB-->>Gateway: void
        deactivate CB
        Gateway-->>UseCase: Exception + Fallback
        deactivate Gateway
        
        UseCase->>Queue: Enfileirar documento
        activate Queue
        Queue-->>UseCase: messageId
        deactivate Queue
    end
    
    Note over CB: Estado: OPEN<br/>Aguardando 2 minutos

    UseCase->>Gateway: enviarDocumento(request)
    activate Gateway
    Gateway->>CB: check()
    activate CB
    CB-->>Gateway: CircuitBreakerOpenException
    deactivate CB
    Gateway->>Gateway: Executar fallback
    Gateway-->>UseCase: Enfileirado
    deactivate Gateway
    
    Note over CB: Após 2 minutos

    UseCase->>Gateway: enviarDocumento(request)
    activate Gateway
    Gateway->>CB: check()
    activate CB
    CB->>CB: Transição para HALF_OPEN
    CB-->>Gateway: OK (half-open, permitir 5 chamadas)
    deactivate CB
    
    Gateway->>Jarvis: POST /documents-reading
    activate Jarvis
    Jarvis-->>Gateway: 202 Accepted
    deactivate Jarvis
    
    Gateway->>CB: recordSuccess()
    activate CB
    
    Note over CB: Após 5 sucessos consecutivos
    
    CB->>CB: CLOSED circuit
    CB->>Metrics: circuit_breaker.state = 0
    CB-->>Gateway: void
    deactivate CB
    
    Gateway-->>UseCase: Success
    deactivate Gateway
```

---

## Fluxo 11: Processamento Assíncrono via Service Bus

**Descrição:** Processamento de mensagens da fila para envio ao Jarvis.

```mermaid
sequenceDiagram
    participant Producer as Event Handler
    participant Queue as Azure Service Bus
    participant Listener as Service Bus Listener
    participant UseCase as ProcessarDocumentoJarvisUseCase
    participant DB as PostgreSQL
    participant Gateway as JarvisGateway
    participant Jarvis as Jarvis API

    Producer->>Queue: sendMessage(jarvis-ocr-queue,<br/>{sinistroId, documentId})
    activate Queue
    Queue-->>Producer: messageId
    
    Note over Queue: Mensagem enfileirada<br/>Aguardando processamento

    Queue->>Listener: onMessage(message)
    deactivate Queue
    activate Listener
    
    Listener->>Listener: Extrair payload<br/>Validar messageId (idempotência)
    
    Listener->>UseCase: executar(sinistroId, documentId)
    activate UseCase
    
    UseCase->>DB: Verificar se já processado<br/>WHERE messageId = ?
    activate DB
    DB-->>UseCase: null (não processado)
    deactivate DB
    
    UseCase->>DB: Buscar sinistro e documento
    activate DB
    DB-->>UseCase: sinistro, documento
    deactivate DB
    
    UseCase->>Gateway: enviarDocumento(documento)
    activate Gateway
    
    Gateway->>Jarvis: POST /documents-reading
    activate Jarvis
    
    alt Sucesso
        Jarvis-->>Gateway: 202 Accepted
        deactivate Jarvis
        Gateway-->>UseCase: requestId
        deactivate Gateway
        
        UseCase->>DB: INSERT tb_ecm_jarvis<br/>status: PROCESSING
        activate DB
        DB-->>UseCase: OK
        deactivate DB
        
        UseCase->>DB: Registrar messageId processado
        activate DB
        DB-->>UseCase: OK
        deactivate DB
        
        UseCase-->>Listener: ProcessamentoOK
        deactivate UseCase
        
        Listener->>Queue: complete(message)
        activate Queue
        Queue-->>Listener: OK
        deactivate Queue
        
    else Erro recuperável (5xx)
        Jarvis-->>Gateway: 503 Service Unavailable
        deactivate Jarvis
        Gateway-->>UseCase: JarvisIndisponivelException
        deactivate Gateway
        deactivate UseCase
        
        Listener->>Queue: abandon(message)<br/>incrementar deliveryCount
        activate Queue
        Queue-->>Listener: Re-enfileirado
        deactivate Queue
        
        Note over Queue: Retry automático<br/>após delay
        
    else Erro não recuperável (4xx)
        Jarvis-->>Gateway: 400 Bad Request
        deactivate Jarvis
        Gateway-->>UseCase: PayloadInvalidoException
        deactivate Gateway
        deactivate UseCase
        
        Listener->>Queue: deadLetter(message)
        activate Queue
        Queue-->>Listener: Movido para DLQ
        deactivate Queue
    end
    
    deactivate Listener
```

---

## Fluxo 12: Geração de Relatórios e Dashboards

**Descrição:** Consulta de métricas e geração de relatórios operacionais.

```mermaid
sequenceDiagram
    participant Usuario as Usuário Gestor
    participant Frontend as React Frontend
    participant Controller as RelatorioController
    participant UseCase as GerarDashboardUseCase
    participant DB as PostgreSQL
    participant Cache as Redis (futuro)

    Usuario->>Frontend: Acessar /dashboard
    activate Frontend
    
    Frontend->>Controller: GET /api/relatorios/dashboard<br/>?periodo=ULTIMO_MES
    activate Controller
    
    Controller->>UseCase: executar(periodo)
    activate UseCase
    
    alt Cache disponível
        UseCase->>Cache: GET dashboard:ultimo_mes
        activate Cache
        Cache-->>UseCase: dados (cached)
        deactivate Cache
    else Cache miss
        UseCase->>DB: Query tb_log_transacao<br/>Agregações por serviço
        activate DB
        DB-->>UseCase: estatísticas
        deactivate DB
        
        UseCase->>DB: Query tb_ecm_jarvis<br/>Médias de scores OCR
        activate DB
        DB-->>UseCase: scores médios
        deactivate DB
        
        UseCase->>DB: Query contagem por status
        activate DB
        DB-->>UseCase: distribuição
        deactivate DB
        
        UseCase->>UseCase: Calcular métricas:<br/>- Taxa de sucesso<br/>- Throughput<br/>- Latência média
        
        UseCase->>Cache: SET dashboard:ultimo_mes<br/>TTL: 5 minutos
        activate Cache
        Cache-->>UseCase: OK
        deactivate Cache
    end
    
    UseCase-->>Controller: DashboardDTO
    deactivate UseCase
    
    Controller-->>Frontend: 200 OK<br/>{metricas, graficos[]}
    deactivate Controller
    
    Frontend->>Frontend: Renderizar dashboards<br/>Gráficos com Recharts
    Frontend-->>Usuario: Exibir métricas
    deactivate Frontend
```

---

## Fluxo 13: Idempotência - Request Duplicado

**Descrição:** Tratamento de request duplicado usando Idempotency-Key.

```mermaid
sequenceDiagram
    participant Cliente as Sistema Cliente
    participant Controller as SinistroController
    participant Interceptor as IdempotencyInterceptor
    participant DB as PostgreSQL
    participant UseCase as ReceberDocumentoUseCase

    Cliente->>Controller: POST /api/sinistros<br/>Idempotency-Key: abc-123
    activate Controller
    
    Controller->>Interceptor: preHandle(request)
    activate Interceptor
    
    Interceptor->>DB: SELECT FROM tb_idempotency<br/>WHERE key = 'abc-123'<br/>AND endpoint = '/api/sinistros'
    activate DB
    DB-->>Interceptor: null (primeira vez)
    deactivate DB
    
    Interceptor-->>Controller: Prosseguir
    deactivate Interceptor
    
    Controller->>UseCase: executar(request)
    activate UseCase
    UseCase->>UseCase: Processar normalmente
    UseCase-->>Controller: SinistroResponse
    deactivate UseCase
    
    Controller->>Interceptor: afterCompletion(response)
    activate Interceptor
    
    Interceptor->>DB: INSERT tb_idempotency<br/>{key, endpoint, response, ttl}
    activate DB
    DB-->>Interceptor: OK
    deactivate DB
    deactivate Interceptor
    
    Controller-->>Cliente: 201 Created<br/>{sinistroId}
    deactivate Controller
    
    Note over Cliente: Cliente faz retry<br/>por timeout de rede

    Cliente->>Controller: POST /api/sinistros<br/>Idempotency-Key: abc-123<br/>(MESMO KEY)
    activate Controller
    
    Controller->>Interceptor: preHandle(request)
    activate Interceptor
    
    Interceptor->>DB: SELECT FROM tb_idempotency<br/>WHERE key = 'abc-123'
    activate DB
    DB-->>Interceptor: {response, createdAt}
    deactivate DB
    
    Interceptor->>Interceptor: Verificar TTL (< 24h)
    Interceptor-->>Controller: Response cached
    deactivate Interceptor
    
    Controller-->>Cliente: 200 OK<br/>{sinistroId} (mesmo response)
    deactivate Controller
    
    Note over Controller: Use case NÃO executado<br/>Sinistro não duplicado
```

---

## Fluxo 14: Retry com Exponential Backoff

**Descrição:** Política de retry em caso de falha temporária.

```mermaid
sequenceDiagram
    participant UseCase as RotearParaPegaUseCase
    participant Retry as Resilience4j Retry
    participant Gateway as PegaGateway
    participant Pega as Pega API
    participant Metrics as Micrometer

    UseCase->>Gateway: criarCaso(request)
    activate Gateway
    
    Gateway->>Retry: execute(() -> chamadaPega())
    activate Retry
    
    Note over Retry: Tentativa 1
    
    Retry->>Pega: POST /api/pega/v1/cases
    activate Pega
    Pega-->>Retry: 503 Service Unavailable
    deactivate Pega
    
    Retry->>Retry: Aguardar 1 segundo
    Retry->>Metrics: retry.count++
    
    Note over Retry: Tentativa 2
    
    Retry->>Pega: POST /api/pega/v1/cases
    activate Pega
    Pega-->>Retry: 503 Service Unavailable
    deactivate Pega
    
    Retry->>Retry: Aguardar 2 segundos<br/>(exponential backoff)
    Retry->>Metrics: retry.count++
    
    Note over Retry: Tentativa 3 (última)
    
    Retry->>Pega: POST /api/pega/v1/cases
    activate Pega
    Pega-->>Retry: 201 Created
    deactivate Pega
    
    Retry->>Metrics: retry.success++
    Retry-->>Gateway: Response
    deactivate Retry
    
    Gateway-->>UseCase: PegaResponse
    deactivate Gateway
```

---

## Observações

### Renderização dos Diagramas

Estes diagramas podem ser visualizados em:
- **GitHub/GitLab** - Suporte nativo a Mermaid
- **Azure DevOps** - Plugin Mermaid
- **VS Code** - Extensão "Markdown Preview Mermaid Support"
- **Online** - https://mermaid.live

### Exportação

Para exportar como imagem:
1. Abrir em https://mermaid.live
2. Clicar em "Actions" → "Export"
3. Escolher formato (PNG, SVG, PDF)

### Convenções

- **Linhas sólidas** (→): Chamadas síncronas
- **Linhas tracejadas** (--→): Respostas
- **Boxes** (activate/deactivate): Tempo de vida de execução
- **alt/else**: Fluxos condicionais
- **loop**: Iterações
- **Note**: Comentários explicativos

---

**Documento elaborado por:** Equipe de Arquitetura  
**Última atualização:** Janeiro de 2026
