# 13. Estratégia de Testes - App View v2.0

**Versão:** 1.0  
**Última Atualização:** Janeiro 2025  
**Responsável:** Arquitetura & QA

---

## Sumário

1. [Visão Geral](#visao-geral)
2. [Pirâmide de Testes](#piramide-de-testes)
3. [Testes Unitários (Unit Tests)](#testes-unitarios-unit-tests)
4. [Testes de Integração (Integration Tests)](#testes-de-integracao-integration-tests)
5. [Testes End-to-End (E2E Tests)](#testes-end-to-end-e2e-tests)
6. [Smoke Tests](#smoke-tests)
7. [Testes de Contrato (Contract Tests)](#testes-de-contrato-contract-tests)
8. [Testes de Performance](#testes-de-performance)
9. [Frameworks e Ferramentas](#frameworks-e-ferramentas)
10. [Convenções e Padrões](#convencoes-e-padroes)
11. [Coverage e Métricas](#coverage-e-metricas)
12. [Estratégia de Dados de Teste](#estrategia-de-dados-de-teste)
13. [Execução no CI/CD](#execucao-no-ci-cd)

---

## Visão Geral

### Objetivos

A estratégia de testes do App View v2.0 busca:

1. ✅ **Prevenir regressões** - Detectar bugs antes de chegarem em produção
2. ✅ **Documentar comportamento** - Testes como especificação viva do sistema
3. ✅ **Facilitar refatoração** - Confiança para mudar código
4. ✅ **Garantir qualidade** - 80%+ coverage em código crítico
5. ✅ **Fast feedback** - Testes rápidos para desenvolvimento ágil

### Princípios

1. **Fail Fast** - Testes devem falhar rapidamente quando algo quebra
2. **Testes Isolados** - Cada teste deve ser independente (sem ordem de execução)
3. **Testes Legíveis** - Testes são documentação; devem ser claros como prosa
4. **Testes Rápidos** - Unit tests < 1s, Integration tests < 30s
5. **Testes Confiáveis** - Zero flakiness; testes intermitentes devem ser corrigidos ou removidos

---

## Pirâmide de Testes

Nossa estratégia segue a pirâmide de testes clássica:

```
                    ▲
                   ╱ ╲
                  ╱   ╲
                 ╱ E2E ╲              ~5% dos testes
                ╱───────╲             Lentos, caros, frágeis
               ╱         ╲            Validam fluxos críticos
              ╱───────────╲
             ╱ Integration ╲          ~20% dos testes
            ╱───────────────╲         Médios, validam integração
           ╱                 ╲        com sistemas externos
          ╱───────────────────╲
         ╱       Unit          ╲      ~75% dos testes
        ╱─────────────────────────╲   Rápidos, baratos, confiáveis
       ╱                           ╲  Validam lógica de negócio
      ╱─────────────────────────────╲
```

### Distribuição de Testes

| Tipo | % do Total | Duração Típica | Escopo | Exemplo |
|------|------------|----------------|--------|---------|
| **Unit** | 75% | < 100ms | Classe/método isolado | Validar regra de negócio |
| **Integration** | 20% | < 5s | Componente + dependências reais | Repository com banco real |
| **E2E** | 5% | < 30s | Fluxo completo end-to-end | Criar sinistro + OCR + Alfresco |
| **Smoke** | ~10 testes | < 10s | Sanity check produção | Health checks, login |

---

## Testes Unitários (Unit Tests)

### Escopo

Testam **unidades isoladas** de código sem dependências externas:
- Entidades de domínio
- Value Objects
- Use Cases (com mocks)
- Regras de negócio

### Características

- ✅ **Rápidos:** < 100ms cada
- ✅ **Isolados:** Sem I/O (banco, rede, filesystem)
- ✅ **Determinísticos:** Sempre mesmo resultado
- ✅ **Sem dependências externas:** Mocks/stubs para tudo

### Exemplo 1: Testar Entidade de Domínio

```java
package br.com.appview.sinistro.domain;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.DisplayName;

import static org.assertj.core.api.Assertions.*;

@DisplayName("Sinistro - Testes Unitários")
class SinistroUnitTest {

    @Test
    @DisplayName("Deve criar sinistro com status ABERTO")
    void deveCriarSinistroComStatusAberto() {
        // Arrange
        var numero = new NumeroSinistro("2025000001");
        var cpf = new CPF("12345678900");
        var valor = Money.of(10000.00);

        // Act
        var sinistro = Sinistro.criar(numero, cpf, valor);

        // Assert
        assertThat(sinistro.getStatus()).isEqualTo(StatusSinistro.ABERTO);
        assertThat(sinistro.getNumero()).isEqualTo(numero);
        assertThat(sinistro.getValorEstimado()).isEqualTo(valor);
        assertThat(sinistro.getDataAbertura()).isToday();
    }

    @Test
    @DisplayName("Deve lançar exceção ao criar sinistro com CPF inválido")
    void deveLancarExcecaoAoCriarSinistroComCpfInvalido() {
        // Arrange
        var cpfInvalido = "111.111.111-11";

        // Act & Assert
        assertThatThrownBy(() -> new CPF(cpfInvalido))
            .isInstanceOf(DomainException.class)
            .hasMessageContaining("CPF inválido");
    }

    @Test
    @DisplayName("Deve transicionar de ABERTO para EM_ANALISE")
    void deveTransicionarDeAbertoParaEmAnalise() {
        // Arrange
        var sinistro = SinistroTestFixtures.umSinistroAberto();

        // Act
        sinistro.iniciarAnalise();

        // Assert
        assertThat(sinistro.getStatus()).isEqualTo(StatusSinistro.EM_ANALISE);
        assertThat(sinistro.getDomainEvents())
            .hasSize(1)
            .first()
            .isInstanceOf(SinistroEmAnaliseEvent.class);
    }

    @Test
    @DisplayName("Não deve permitir transição de FECHADO para EM_ANALISE")
    void naoDevePermitirTransicaoDeFechadoParaEmAnalise() {
        // Arrange
        var sinistro = SinistroTestFixtures.umSinistroFechado();

        // Act & Assert
        assertThatThrownBy(() -> sinistro.iniciarAnalise())
            .isInstanceOf(DomainException.class)
            .hasMessageContaining("Transição inválida");
    }
}
```

### Exemplo 2: Testar Use Case com Mocks

```java
package br.com.appview.sinistro.application.usecase;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
@DisplayName("CriarSinistroUseCase - Testes Unitários")
class CriarSinistroUseCaseUnitTest {

    @Mock
    private SinistroRepository sinistroRepository;

    @Mock
    private ClienteRepository clienteRepository;

    @Mock
    private ContratoRepository contratoRepository;

    @Mock
    private EventPublisher eventPublisher;

    @InjectMocks
    private CriarSinistroUseCase useCase;

    @Test
    @DisplayName("Deve criar sinistro com sucesso")
    void deveCriarSinistroComSucesso() {
        // Arrange
        var command = CriarSinistroCommand.builder()
            .cpf("12345678900")
            .numeroContrato("123456")
            .tipoSinistro(TipoSinistro.DANOS_MATERIAIS)
            .valorEstimado(10000.00)
            .descricao("Colisão traseira")
            .build();

        var cliente = ClienteTestFixtures.umCliente();
        var contrato = ContratoTestFixtures.umContratoAtivo();

        when(clienteRepository.findByCpf(any(CPF.class)))
            .thenReturn(Optional.of(cliente));
        when(contratoRepository.findByNumero(any(NumeroContrato.class)))
            .thenReturn(Optional.of(contrato));
        when(sinistroRepository.save(any(Sinistro.class)))
            .thenAnswer(invocation -> invocation.getArgument(0));

        // Act
        var resultado = useCase.execute(command);

        // Assert
        assertThat(resultado).isNotNull();
        assertThat(resultado.getNumeroSinistro()).isNotBlank();
        assertThat(resultado.getStatus()).isEqualTo("ABERTO");

        // Verify
        verify(sinistroRepository).save(any(Sinistro.class));
        verify(eventPublisher).publish(any(SinistroCriadoEvent.class));
    }

    @Test
    @DisplayName("Deve lançar exceção quando cliente não existe")
    void deveLancarExcecaoQuandoClienteNaoExiste() {
        // Arrange
        var command = CriarSinistroCommand.builder()
            .cpf("12345678900")
            .numeroContrato("123456")
            .build();

        when(clienteRepository.findByCpf(any(CPF.class)))
            .thenReturn(Optional.empty());

        // Act & Assert
        assertThatThrownBy(() -> useCase.execute(command))
            .isInstanceOf(ClienteNaoEncontradoException.class)
            .hasMessageContaining("Cliente não encontrado");

        verify(sinistroRepository, never()).save(any());
        verify(eventPublisher, never()).publish(any());
    }

    @Test
    @DisplayName("Deve lançar exceção quando contrato está cancelado")
    void deveLancarExcecaoQuandoContratoEstaCancelado() {
        // Arrange
        var command = CriarSinistroCommand.builder()
            .cpf("12345678900")
            .numeroContrato("123456")
            .build();

        var cliente = ClienteTestFixtures.umCliente();
        var contratoCancelado = ContratoTestFixtures.umContratoCancelado();

        when(clienteRepository.findByCpf(any(CPF.class)))
            .thenReturn(Optional.of(cliente));
        when(contratoRepository.findByNumero(any(NumeroContrato.class)))
            .thenReturn(Optional.of(contratoCancelado));

        // Act & Assert
        assertThatThrownBy(() -> useCase.execute(command))
            .isInstanceOf(ContratoInvalidoException.class)
            .hasMessageContaining("Contrato cancelado");
    }
}
```

### Convenção de Nomenclatura

```
<ClasseTestada><Scenario>UnitTest.java

Exemplos:
- SinistroUnitTest.java
- CPFUnitTest.java
- CriarSinistroUseCaseUnitTest.java
```

### Executar Unit Tests

```bash
# Todos os testes unitários
mvn test -Dtest="**/*UnitTest"

# Testes de um módulo específico
mvn test -Dtest="br.com.appview.sinistro.**/*UnitTest"

# Com coverage
mvn test jacoco:report
```

---

## Testes de Integração (Integration Tests)

### Escopo

Testam **integração entre componentes** com dependências REAIS:
- Repositories com banco de dados real (Testcontainers)
- Chamadas HTTP para sistemas externos (WireMock)
- Message queues (Azure Service Bus Testcontainers)
- Cache (Redis Testcontainers)

### Características

- ✅ **Médios:** < 5s cada
- ✅ **Dependências reais:** Banco, Redis, filas
- ✅ **Isolados por teste:** Rollback ou cleanup após cada teste
- ✅ **Reproduzíveis:** Containers Docker garantem consistência

### Exemplo 1: Integration Test com Repository

```java
package br.com.appview.sinistro.infrastructure.persistence;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.DisplayName;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import static org.assertj.core.api.Assertions.*;

@DataJpaTest
@Testcontainers
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@DisplayName("SinistroRepository - Testes de Integração")
class SinistroRepositoryIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("appview_test")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private SinistroRepository repository;

    @Test
    @DisplayName("Deve salvar e recuperar sinistro")
    void deveSalvarERecuperarSinistro() {
        // Arrange
        var sinistro = SinistroTestFixtures.umSinistroAberto();

        // Act
        var saved = repository.save(sinistro);
        var retrieved = repository.findById(saved.getId());

        // Assert
        assertThat(retrieved).isPresent();
        assertThat(retrieved.get().getNumero()).isEqualTo(sinistro.getNumero());
        assertThat(retrieved.get().getStatus()).isEqualTo(StatusSinistro.ABERTO);
    }

    @Test
    @DisplayName("Deve buscar sinistros por CPF")
    void deveBuscarSinistrosPorCpf() {
        // Arrange
        var cpf = new CPF("12345678900");
        var sinistro1 = SinistroTestFixtures.umSinistroAberto(cpf);
        var sinistro2 = SinistroTestFixtures.umSinistroEmAnalise(cpf);

        repository.save(sinistro1);
        repository.save(sinistro2);

        // Act
        var sinistros = repository.findByCpf(cpf);

        // Assert
        assertThat(sinistros).hasSize(2);
        assertThat(sinistros)
            .extracting(Sinistro::getNumero)
            .containsExactlyInAnyOrder(
                sinistro1.getNumero(),
                sinistro2.getNumero()
            );
    }

    @Test
    @DisplayName("Deve contar sinistros por status")
    void deveContarSinistrosPorStatus() {
        // Arrange
        repository.save(SinistroTestFixtures.umSinistroAberto());
        repository.save(SinistroTestFixtures.umSinistroAberto());
        repository.save(SinistroTestFixtures.umSinistroFechado());

        // Act
        var abertos = repository.countByStatus(StatusSinistro.ABERTO);
        var fechados = repository.countByStatus(StatusSinistro.FECHADO);

        // Assert
        assertThat(abertos).isEqualTo(2);
        assertThat(fechados).isEqualTo(1);
    }
}
```

### Exemplo 2: Integration Test com API Externa (WireMock)

```java
package br.com.appview.alfresco.infrastructure;

import com.github.tomakehurst.wiremock.WireMockServer;
import com.github.tomakehurst.wiremock.client.WireMock;
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.*;

@SpringBootTest
@DisplayName("AlfrescoClient - Testes de Integração")
class AlfrescoClientIntegrationTest {

    private static WireMockServer wireMockServer;

    @Autowired
    private AlfrescoClient alfrescoClient;

    @BeforeAll
    static void startWireMock() {
        wireMockServer = new WireMockServer(8089);
        wireMockServer.start();
        WireMock.configureFor("localhost", 8089);
    }

    @AfterAll
    static void stopWireMock() {
        wireMockServer.stop();
    }

    @AfterEach
    void resetWireMock() {
        wireMockServer.resetAll();
    }

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("alfresco.base-url", () -> "http://localhost:8089");
    }

    @Test
    @DisplayName("Deve fazer upload de documento com sucesso")
    void deveFazerUploadDeDocumentoComSucesso() {
        // Arrange
        stubFor(post(urlEqualTo("/alfresco/api/-default-/public/cmis/versions/1.1/atom/content"))
            .willReturn(aResponse()
                .withStatus(201)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {
                      "id": "abc-123",
                      "name": "sinistro-001.pdf",
                      "mimeType": "application/pdf"
                    }
                    """)));

        var documento = DocumentoTestFixtures.umPDF();

        // Act
        var nodeId = alfrescoClient.uploadDocument(documento);

        // Assert
        assertThat(nodeId).isEqualTo("abc-123");

        verify(postRequestedFor(urlEqualTo("/alfresco/api/-default-/public/cmis/versions/1.1/atom/content"))
            .withHeader("Content-Type", containing("multipart/form-data")));
    }

    @Test
    @DisplayName("Deve fazer retry em caso de falha temporária")
    void deveFazerRetryEmCasoDeFalhaTemporaria() {
        // Arrange
        stubFor(post(urlEqualTo("/alfresco/api/-default-/public/cmis/versions/1.1/atom/content"))
            .inScenario("Retry")
            .whenScenarioStateIs(STARTED)
            .willReturn(aResponse().withStatus(500))
            .willSetStateTo("Second attempt"));

        stubFor(post(urlEqualTo("/alfresco/api/-default-/public/cmis/versions/1.1/atom/content"))
            .inScenario("Retry")
            .whenScenarioStateIs("Second attempt")
            .willReturn(aResponse()
                .withStatus(201)
                .withBody("""
                    {
                      "id": "abc-456",
                      "name": "sinistro-002.pdf"
                    }
                    """)));

        var documento = DocumentoTestFixtures.umPDF();

        // Act
        var nodeId = alfrescoClient.uploadDocument(documento);

        // Assert
        assertThat(nodeId).isEqualTo("abc-456");

        verify(2, postRequestedFor(
            urlEqualTo("/alfresco/api/-default-/public/cmis/versions/1.1/atom/content")));
    }
}
```

### Exemplo 3: Integration Test com Message Queue

```java
package br.com.appview.messaging;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.DisplayName;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.*;
import static org.awaitility.Awaitility.*;

@SpringBootTest
@Testcontainers
@DisplayName("MessagePublisher - Testes de Integração")
class MessagePublisherIntegrationTest {

    @Container
    static GenericContainer<?> azurite = new GenericContainer<>("mcr.microsoft.com/azure-storage/azurite")
        .withExposedPorts(10000, 10001, 10002);

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("azure.servicebus.connection-string", 
            () -> "Endpoint=sb://localhost;SharedAccessKeyName=test;SharedAccessKey=test");
    }

    @Autowired
    private MessagePublisher messagePublisher;

    @Autowired
    private TestMessageListener testListener;

    @Test
    @DisplayName("Deve publicar e consumir mensagem com sucesso")
    void devePublicarEConsumirMensagemComSucesso() {
        // Arrange
        var event = new SinistroCriadoEvent(
            "2025000001",
            "12345678900",
            LocalDateTime.now()
        );

        // Act
        messagePublisher.publish(event);

        // Assert
        await()
            .atMost(5, TimeUnit.SECONDS)
            .untilAsserted(() -> {
                assertThat(testListener.getReceivedEvents())
                    .hasSize(1)
                    .first()
                    .hasFieldOrPropertyWithValue("numeroSinistro", "2025000001");
            });
    }
}
```

### Convenção de Nomenclatura

```
<ClasseTestada>IntegrationTest.java

Exemplos:
- SinistroRepositoryIntegrationTest.java
- AlfrescoClientIntegrationTest.java
- OCRProcessorIntegrationTest.java
```

### Executar Integration Tests

```bash
# Todos os testes de integração
mvn test -Dtest="**/*IntegrationTest"

# Com Testcontainers
mvn verify -Dspring.profiles.active=integration-test
```

---

## Testes End-to-End (E2E Tests)

### Escopo

Testam **fluxos completos** simulando usuário real:
- API REST completa (HTTP)
- Fluxos multi-step (criar → processar → consultar)
- Validação de integrações entre múltiplos componentes

### Características

- ✅ **Lentos:** < 30s cada
- ✅ **Frágeis:** Mais suscetíveis a quebrar
- ✅ **Poucos:** Apenas fluxos críticos de negócio
- ✅ **Black-box:** Testam pela API pública

### Exemplo: E2E Test - Fluxo Completo de Sinistro

```java
package br.com.appview.sinistro.e2e;

import io.restassured.RestAssured;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.DisplayName;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import static io.restassured.RestAssured.*;
import static org.hamcrest.Matchers.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@DisplayName("Sinistro - Testes E2E")
class SinistroE2ETest {

    @LocalServerPort
    private int port;

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @BeforeEach
    void setup() {
        RestAssured.port = port;
        RestAssured.basePath = "/api/v1";
    }

    @Test
    @DisplayName("Deve executar fluxo completo de criação de sinistro")
    void deveExecutarFluxoCompletoDecriacaoDeSinistro() {
        // Step 1: Criar sinistro
        var numeroSinistro = given()
            .contentType(ContentType.JSON)
            .header("Authorization", "Bearer " + obterToken())
            .body("""
                {
                  "cpf": "12345678900",
                  "numeroContrato": "CON-2025-001",
                  "tipoSinistro": "DANOS_MATERIAIS",
                  "valorEstimado": 15000.00,
                  "descricao": "Colisão traseira no estacionamento",
                  "dataOcorrencia": "2025-01-15"
                }
                """)
        .when()
            .post("/sinistros")
        .then()
            .statusCode(201)
            .body("numeroSinistro", notNullValue())
            .body("status", equalTo("ABERTO"))
            .extract()
            .path("numeroSinistro");

        // Step 2: Anexar documento
        given()
            .contentType(ContentType.MULTIPART)
            .header("Authorization", "Bearer " + obterToken())
            .multiPart("file", "boletim-ocorrencia.pdf", "PDF content".getBytes(), "application/pdf")
            .multiPart("tipoDocumento", "BOLETIM_OCORRENCIA")
        .when()
            .post("/sinistros/{numero}/documentos", numeroSinistro)
        .then()
            .statusCode(201)
            .body("documentoId", notNullValue())
            .body("status", equalTo("AGUARDANDO_OCR"));

        // Step 3: Aguardar processamento OCR (async)
        await()
            .atMost(10, TimeUnit.SECONDS)
            .pollInterval(1, TimeUnit.SECONDS)
            .untilAsserted(() -> {
                given()
                    .header("Authorization", "Bearer " + obterToken())
                .when()
                    .get("/sinistros/{numero}/documentos", numeroSinistro)
                .then()
                    .statusCode(200)
                    .body("[0].status", equalTo("OCR_CONCLUIDO"));
            });

        // Step 4: Consultar sinistro atualizado
        given()
            .header("Authorization", "Bearer " + obterToken())
        .when()
            .get("/sinistros/{numero}", numeroSinistro)
        .then()
            .statusCode(200)
            .body("numeroSinistro", equalTo(numeroSinistro))
            .body("status", equalTo("EM_ANALISE"))
            .body("documentos", hasSize(1))
            .body("documentos[0].ocrRealizado", equalTo(true));
    }

    @Test
    @DisplayName("Deve rejeitar criação de sinistro com dados inválidos")
    void deveRejeitarCriacaoDeSinistroComDadosInvalidos() {
        given()
            .contentType(ContentType.JSON)
            .header("Authorization", "Bearer " + obterToken())
            .body("""
                {
                  "cpf": "111.111.111-11",
                  "numeroContrato": "",
                  "tipoSinistro": "INVALIDO",
                  "valorEstimado": -100
                }
                """)
        .when()
            .post("/sinistros")
        .then()
            .statusCode(400)
            .body("errors", hasSize(greaterThan(0)))
            .body("errors[0].field", notNullValue())
            .body("errors[0].message", notNullValue());
    }

    private String obterToken() {
        return given()
            .contentType(ContentType.JSON)
            .body("""
                {
                  "username": "admin@appview.com",
                  "password": "admin123"
                }
                """)
        .when()
            .post("/auth/login")
        .then()
            .statusCode(200)
            .extract()
            .path("token");
    }
}
```

### Convenção de Nomenclatura

```
<Modulo>E2ETest.java

Exemplos:
- SinistroE2ETest.java
- PericiaE2ETest.java
- DocumentoE2ETest.java
```

### Executar E2E Tests

```bash
# Todos os testes E2E
mvn test -Dtest="**/*E2ETest"

# Com ambiente completo (banco + mocks externos)
mvn verify -Dspring.profiles.active=e2e
```

---

## Smoke Tests

### Escopo

Testes **mínimos e rápidos** para validar que:
- Aplicação está rodando
- Health checks funcionam
- Endpoints críticos estão acessíveis

### Características

- ✅ **Muito rápidos:** < 10s total
- ✅ **Production-safe:** Não modificam dados
- ✅ **Executados pós-deploy:** Validação em produção
- ✅ **Idempotentes:** Podem rodar múltiplas vezes

### Exemplo: Smoke Tests

```java
package br.com.appview.smoke;

import io.restassured.RestAssured;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.DisplayName;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;

import static io.restassured.RestAssured.*;
import static org.hamcrest.Matchers.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@DisplayName("Smoke Tests - Validações Críticas")
class SmokeTest {

    @LocalServerPort
    private int port;

    @BeforeEach
    void setup() {
        RestAssured.port = port;
    }

    @Test
    @DisplayName("Deve validar health check")
    void deveValidarHealthCheck() {
        given()
        .when()
            .get("/actuator/health")
        .then()
            .statusCode(200)
            .body("status", equalTo("UP"))
            .body("components.db.status", equalTo("UP"))
            .body("components.diskSpace.status", equalTo("UP"));
    }

    @Test
    @DisplayName("Deve validar info endpoint")
    void deveValidarInfoEndpoint() {
        given()
        .when()
            .get("/actuator/info")
        .then()
            .statusCode(200)
            .body("app.name", equalTo("App View v2.0"))
            .body("app.version", notNullValue());
    }

    @Test
    @DisplayName("Deve validar métricas do Prometheus")
    void deveValidarMetricasDoPrometheus() {
        given()
        .when()
            .get("/actuator/prometheus")
        .then()
            .statusCode(200)
            .contentType("text/plain;version=0.0.4;charset=utf-8")
            .body(containsString("jvm_memory_used_bytes"));
    }

    @Test
    @DisplayName("Deve validar endpoint de autenticação")
    void deveValidarEndpointDeAutenticacao() {
        given()
            .contentType("application/json")
        .when()
            .post("/api/v1/auth/health")
        .then()
            .statusCode(200)
            .body("status", equalTo("healthy"));
    }

    @Test
    @DisplayName("Deve validar conectividade com banco de dados")
    void deveValidarConectividadeComBancoDeDados() {
        given()
        .when()
            .get("/actuator/health/db")
        .then()
            .statusCode(200)
            .body("status", equalTo("UP"))
            .body("details.database", equalTo("PostgreSQL"));
    }

    @Test
    @DisplayName("Deve validar conectividade com Redis")
    void deveValidarConectividadeComRedis() {
        given()
        .when()
            .get("/actuator/health/redis")
        .then()
            .statusCode(200)
            .body("status", equalTo("UP"));
    }
}
```

### Convenção de Nomenclatura

```
SmokeTest.java

Nota: Apenas UM arquivo com todos os smoke tests
```

### Executar Smoke Tests

```bash
# Smoke tests
mvn test -Dtest="SmokeTest"

# Em produção (via GitHub Actions)
mvn test -Dtest="SmokeTest" -Dtest.base.url=https://appview-prod.azurewebsites.net
```

---

## Testes de Contrato (Contract Tests)

### Escopo

Validam **contratos de APIs externas**:
- Requests/Responses esperados de APIs (Alfresco, Jarvis, Pega)
- Previne quebras quando API externa muda
- Documentação viva dos contratos

### Exemplo: Contract Test com Pact

```java
package br.com.appview.alfresco.contract;

import au.com.dius.pact.consumer.dsl.PactDslWithProvider;
import au.com.dius.pact.consumer.junit5.PactConsumerTestExt;
import au.com.dius.pact.consumer.junit5.PactTestFor;
import au.com.dius.pact.core.model.RequestResponsePact;
import au.com.dius.pact.core.model.annotations.Pact;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import static org.assertj.core.api.Assertions.*;

@SpringBootTest
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "alfresco-api")
class AlfrescoApiContractTest {

    @Autowired
    private AlfrescoClient alfrescoClient;

    @Pact(consumer = "appview-backend")
    public RequestResponsePact uploadDocumentContract(PactDslWithProvider builder) {
        return builder
            .given("Alfresco está online")
            .uponReceiving("requisição de upload de documento")
                .path("/alfresco/api/-default-/public/cmis/versions/1.1/atom/content")
                .method("POST")
                .headers("Content-Type", "multipart/form-data")
            .willRespondWith()
                .status(201)
                .headers(Map.of("Content-Type", "application/json"))
                .body("""
                    {
                      "id": "node-123",
                      "name": "documento.pdf",
                      "mimeType": "application/pdf"
                    }
                    """)
            .toPact();
    }

    @Test
    @PactTestFor(pactMethod = "uploadDocumentContract")
    void deveValidarContratoDeUploadDeDocumento() {
        // Act
        var documento = DocumentoTestFixtures.umPDF();
        var nodeId = alfrescoClient.uploadDocument(documento);

        // Assert
        assertThat(nodeId).isEqualTo("node-123");
    }
}
```

---

## Testes de Performance

### Escopo

Validam **performance e escalabilidade**:
- Latência (P95, P99)
- Throughput (requests/segundo)
- Carga máxima suportada
- Identificação de gargalos

### Exemplo: JMeter Test Plan

```xml
<!-- sinistro-load-test.jmx -->
<jmeterTestPlan version="1.2">
  <hashTree>
    <TestPlan>
      <stringProp name="TestPlan.name">Sinistro Load Test</stringProp>
    </TestPlan>
    <hashTree>
      <!-- Thread Group: 100 usuários simultâneos -->
      <ThreadGroup>
        <stringProp name="ThreadGroup.num_threads">100</stringProp>
        <stringProp name="ThreadGroup.ramp_time">30</stringProp>
        <stringProp name="ThreadGroup.duration">300</stringProp>
      </ThreadGroup>
      <hashTree>
        <!-- HTTP Request: Criar Sinistro -->
        <HTTPSamplerProxy>
          <stringProp name="HTTPSampler.path">/api/v1/sinistros</stringProp>
          <stringProp name="HTTPSampler.method">POST</stringProp>
          <stringProp name="HTTPSampler.postBodyRaw">
            {
              "cpf": "${CPF}",
              "numeroContrato": "${CONTRATO}",
              "valorEstimado": 10000.00
            }
          </stringProp>
        </HTTPSamplerProxy>
        
        <!-- Assertions -->
        <ResponseAssertion>
          <stringProp name="Assertion.test_field">Assertion.response_code</stringProp>
          <stringProp name="Assertion.test_type">1</stringProp>
          <stringProp name="Assertion.test_string">201</stringProp>
        </ResponseAssertion>
        
        <DurationAssertion>
          <stringProp name="DurationAssertion.duration">2000</stringProp>
        </DurationAssertion>
      </hashTree>
    </hashTree>
  </hashTree>
</jmeterTestPlan>
```

### Executar Performance Tests

```bash
# Via JMeter CLI
jmeter -n -t sinistro-load-test.jmx -l results.jtl

# Via Gatling (alternativa)
mvn gatling:test -Dgatling.simulationClass=SinistroLoadSimulation
```

### Thresholds de Performance

| Métrica | Target | Limite |
|---------|--------|--------|
| **P95 Latency** | < 500ms | < 2s |
| **P99 Latency** | < 1s | < 5s |
| **Throughput** | > 100 req/s | > 50 req/s |
| **Error Rate** | < 0.1% | < 1% |
| **Concurrent Users** | 200 | 100 |

---

## Frameworks e Ferramentas

### Stack de Testes

| Tipo | Framework/Ferramenta | Versão | Propósito |
|------|---------------------|--------|-----------|
| **Unit Tests** | JUnit 5 | 5.10+ | Framework de testes |
| **Assertions** | AssertJ | 3.24+ | Assertions fluentes |
| **Mocking** | Mockito | 5.7+ | Mocks e stubs |
| **Integration** | Testcontainers | 1.19+ | Containers Docker para testes |
| **HTTP Client** | RestAssured | 5.4+ | Testes de API REST |
| **Mocking HTTP** | WireMock | 3.3+ | Mock de APIs externas |
| **Contract** | Pact JVM | 4.6+ | Contract testing |
| **Performance** | JMeter | 5.6+ | Load testing |
| **Coverage** | JaCoCo | 0.8.11+ | Code coverage |
| **Mutation** | Pitest | 1.15+ | Mutation testing |

### Dependências Maven

```xml
<dependencies>
    <!-- JUnit 5 -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- AssertJ -->
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <version>3.24.2</version>
        <scope>test</scope>
    </dependency>

    <!-- Mockito -->
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- Testcontainers -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>testcontainers</artifactId>
        <version>1.19.3</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>postgresql</artifactId>
        <version>1.19.3</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>1.19.3</version>
        <scope>test</scope>
    </dependency>

    <!-- RestAssured -->
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>rest-assured</artifactId>
        <version>5.4.0</version>
        <scope>test</scope>
    </dependency>

    <!-- WireMock -->
    <dependency>
        <groupId>org.wiremock</groupId>
        <artifactId>wiremock</artifactId>
        <version>3.3.1</version>
        <scope>test</scope>
    </dependency>

    <!-- Awaitility (async testing) -->
    <dependency>
        <groupId>org.awaitility</groupId>
        <artifactId>awaitility</artifactId>
        <version>4.2.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

## Convenções e Padrões

### Padrão AAA (Arrange-Act-Assert)

Todos os testes devem seguir o padrão **AAA**:

```java
@Test
void exemplo() {
    // Arrange - Preparar dados e mocks
    var input = ...;
    when(mock.method()).thenReturn(...);

    // Act - Executar ação testada
    var result = useCase.execute(input);

    // Assert - Verificar resultado
    assertThat(result).isEqualTo(...);
}
```

### Nomenclatura de Métodos de Teste

```java
// Padrão: deve<Acao><Contexto>()

@Test
void deveCriarSinistroComSucesso() { ... }

@Test
void deveLancarExcecaoQuandoCpfInvalido() { ... }

@Test
void naoDevePermitirTransicaoInvalida() { ... }
```

### Usar @DisplayName

```java
@Test
@DisplayName("Deve criar sinistro com status ABERTO quando dados válidos")
void test() { ... }
```

### Test Fixtures e Builders

**Evitar:**
```java
// ❌ Ruim: setup complexo inline
var sinistro = new Sinistro();
sinistro.setNumero("2025000001");
sinistro.setCpf(new CPF("12345678900"));
sinistro.setStatus(StatusSinistro.ABERTO);
// ... 20 linhas mais
```

**Preferir:**
```java
// ✅ Bom: usar fixtures
var sinistro = SinistroTestFixtures.umSinistroAberto();

// ✅ Bom: usar builder
var sinistro = SinistroTestBuilder.builder()
    .aberto()
    .comCpf("12345678900")
    .build();
```

### Exemplo de Test Fixture

```java
package br.com.appview.sinistro.fixtures;

public class SinistroTestFixtures {

    public static Sinistro umSinistroAberto() {
        return Sinistro.criar(
            new NumeroSinistro("2025000001"),
            new CPF("12345678900"),
            Money.of(10000.00)
        );
    }

    public static Sinistro umSinistroEmAnalise() {
        var sinistro = umSinistroAberto();
        sinistro.iniciarAnalise();
        return sinistro;
    }

    public static Sinistro umSinistroFechado() {
        var sinistro = umSinistroEmAnalise();
        sinistro.fechar();
        return sinistro;
    }

    public static Sinistro umSinistroAberto(CPF cpf) {
        return Sinistro.criar(
            new NumeroSinistro("2025000001"),
            cpf,
            Money.of(10000.00)
        );
    }
}
```

---

## Coverage e Métricas

### Targets de Coverage

| Camada | Target | Crítico |
|--------|--------|---------|
| **Domain (Entities, VOs)** | 95% | ✅ Obrigatório |
| **Application (Use Cases)** | 90% | ✅ Obrigatório |
| **Infrastructure** | 70% | Recomendado |
| **API (Controllers)** | 80% | Recomendado |
| **Geral** | 80% | ✅ Obrigatório |

### Configuração JaCoCo

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
        <execution>
            <id>jacoco-check</id>
            <goals>
                <goal>check</goal>
            </goals>
            <configuration>
                <rules>
                    <rule>
                        <element>PACKAGE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.80</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### Gerar Relatório de Coverage

```bash
# Gerar relatório
mvn clean test jacoco:report

# Verificar threshold
mvn jacoco:check

# Ver relatório
open target/site/jacoco/index.html
```

### Métricas de Qualidade

| Métrica | Target | Como Medir |
|---------|--------|------------|
| **Coverage** | 80% | JaCoCo |
| **Mutation Score** | 70% | Pitest |
| **Test Execution Time** | < 5min | Maven Surefire |
| **Flaky Tests** | 0 | Manual tracking |

---

## Estratégia de Dados de Teste

### Database Seeding para Testes

```java
@TestConfiguration
public class TestDataConfiguration {

    @Bean
    public DatabaseSeeder databaseSeeder() {
        return new DatabaseSeeder();
    }
}

public class DatabaseSeeder implements ApplicationListener<ContextRefreshedEvent> {

    @Autowired
    private ClienteRepository clienteRepository;

    @Autowired
    private ContratoRepository contratoRepository;

    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        if (Arrays.asList(environment.getActiveProfiles()).contains("test")) {
            seedDatabase();
        }
    }

    private void seedDatabase() {
        // Cliente
        var cliente = Cliente.builder()
            .cpf(new CPF("12345678900"))
            .nome("João da Silva")
            .email("joao@email.com")
            .build();
        clienteRepository.save(cliente);

        // Contrato
        var contrato = Contrato.builder()
            .numero(new NumeroContrato("CON-2025-001"))
            .clienteId(cliente.getId())
            .status(StatusContrato.ATIVO)
            .build();
        contratoRepository.save(contrato);
    }
}
```

### Flyway Test Migrations

```sql
-- src/test/resources/db/migration/V999__test_data.sql
INSERT INTO tb_cliente (id, cpf, nome, email)
VALUES 
    ('11111111-1111-1111-1111-111111111111', '12345678900', 'João da Silva', 'joao@test.com'),
    ('22222222-2222-2222-2222-222222222222', '98765432100', 'Maria Santos', 'maria@test.com');

INSERT INTO tb_contrato (id, numero, cliente_id, status)
VALUES
    ('aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa', 'CON-2025-001', '11111111-1111-1111-1111-111111111111', 'ATIVO');
```

---

## Execução no CI/CD

### Pipeline de Testes

```yaml
# .github/workflows/pr-validation.yml (resumo)

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - name: Run Unit Tests
        run: mvn test -Dtest="**/*UnitTest"
      - name: Upload Coverage
        run: mvn jacoco:report

  integration-tests:
    runs-on: ubuntu-latest
    steps:
      - name: Run Integration Tests
        run: mvn test -Dtest="**/*IntegrationTest"

  e2e-tests:
    runs-on: ubuntu-latest
    steps:
      - name: Run E2E Tests
        run: mvn test -Dtest="**/*E2ETest"
```

### Executar Todos os Testes

```bash
# Tudo de uma vez
mvn verify

# Por tipo
mvn test -Dtest="**/*UnitTest"
mvn test -Dtest="**/*IntegrationTest"
mvn test -Dtest="**/*E2ETest"
mvn test -Dtest="SmokeTest"

# Com perfis
mvn verify -Punit-tests
mvn verify -Pintegration-tests
mvn verify -Pe2e-tests
```

---

## Checklist de Implementação

### Fase 1: Setup

- [ ] Configurar JUnit 5 + AssertJ + Mockito
- [ ] Adicionar Testcontainers
- [ ] Configurar RestAssured
- [ ] Configurar WireMock
- [ ] Configurar JaCoCo
- [ ] Criar fixtures e builders básicos

### Fase 2: Testes Unitários

- [ ] Testes para entities (Domain)
- [ ] Testes para value objects
- [ ] Testes para use cases (Application)
- [ ] Coverage > 80% no domínio

### Fase 3: Testes de Integração

- [ ] Repositories com Testcontainers
- [ ] Clients HTTP com WireMock
- [ ] Message publishers

### Fase 4: Testes E2E & Smoke

- [ ] Fluxos críticos (E2E)
- [ ] Smoke tests para produção
- [ ] Integrar no CI/CD

---

**Fim do Documento**
