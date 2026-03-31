# 15. Guia de Boas Práticas - App View v2.0

**Versão:** 1.0  
**Última Atualização:** Janeiro 2025  
**Responsável:** Arquitetura & DevOps

---

## Sumário

1. [Commits](#commits)
2. [Branching Strategy](#branching-strategy)
3. [Pull Requests](#pull-requests)
4. [Code Review](#code-review)
5. [Convenções de Código](#convencoes-de-codigo)
6. [Segurança](#seguranca)
7. [Testes](#testes)
8. [Documentação](#documentacao)

---

## Commits

### Padrão de Commits

Todos os commits devem seguir o padrão:

```
:<emoji>: <mensagem no presente>
```

### Regras

1. **Tempo verbal:** Sempre no **presente**, não no passado
2. **Emojis:** Sempre iniciar com emoji descritivo
3. **Mensagem:** Clara, concisa e em português
4. **Tamanho:** Máximo 72 caracteres no título

### ✅ Exemplos Corretos

```bash
git commit -m ":sparkles: adiciona validação de CPF"
git commit -m ":bug: corrige cálculo de valor estimado"
git commit -m ":recycle: refatora lógica de retry"
git commit -m ":test_tube: adiciona testes para SinistroService"
git commit -m ":memo: atualiza documentação de API"
git commit -m ":rocket: configura deploy para UAT"
```

### ❌ Exemplos Incorretos

```bash
# ❌ Sem emoji
git commit -m "adiciona validação de CPF"

# ❌ No passado
git commit -m ":sparkles: adicionou validação de CPF"

# ❌ Muito vago
git commit -m ":bug: corrige bug"

# ❌ Muito longo
git commit -m ":sparkles: adiciona validação de CPF para verificar se o CPF digitado pelo usuário está no formato correto"
```

### Tabela de Emojis

| Emoji | Código | Quando Usar |
|-------|--------|-------------|
| ✨ | `:sparkles:` | Nova feature |
| 🐛 | `:bug:` | Correção de bug |
| ♻️ | `:recycle:` | Refatoração |
| 🧪 | `:test_tube:` | Adicionar/modificar testes |
| 📝 | `:memo:` | Documentação |
| 🚀 | `:rocket:` | Deploy/CI/CD |
| 🔒 | `:lock:` | Segurança |
| ⚡ | `:zap:` | Performance |
| 💄 | `:lipstick:` | UI/styling |
| 🔧 | `:wrench:` | Configuração |
| 🗃️ | `:card_file_box:` | Database/migrations |
| 🚨 | `:rotating_light:` | Fix de linter/warnings |
| ⬆️ | `:arrow_up:` | Upgrade de dependências |
| ⬇️ | `:arrow_down:` | Downgrade de dependências |
| 🔥 | `:fire:` | Remover código/arquivos |
| 🚧 | `:construction:` | Work in progress |
| ⏪ | `:rewind:` | Reverter mudanças |

### Commits Atômicos

Cada commit deve representar **uma unidade lógica de mudança**:

```bash
# ✅ Bom: commits separados por responsabilidade
git commit -m ":sparkles: adiciona endpoint de criação de sinistro"
git commit -m ":test_tube: adiciona testes para criação de sinistro"
git commit -m ":memo: documenta endpoint de criação de sinistro"

# ❌ Ruim: commit com múltiplas responsabilidades
git commit -m ":sparkles: adiciona endpoint, testes e documentação"
```

### Corpo do Commit (Opcional)

Para commits mais complexos, use o corpo para detalhar:

```bash
git commit -m ":sparkles: adiciona retry exponencial para Alfresco

Implementa estratégia de retry com backoff exponencial:
- 3 tentativas máximas
- Delays: 1s → 2s → 4s
- Retry apenas para erros de rede (5xx)
- Circuit breaker após 3 falhas consecutivas

Refs: #PROJ-123"
```

---

## Branching Strategy

### Estrutura de Branches

```
main (production)
  ↓
uat (pre-production)
  ↓
dev (development)
  ↓
feature/PROJ-123-validacao-cpf
feature/PROJ-456-integracao-alfresco
hotfix/PROJ-789-corrige-calculo
```

### Branches Principais

#### 1. **main** - Produção

- Código **100% estável** em produção
- **Protected:** Requer PR + 2 aprovações
- Merges apenas via PR após validação completa
- Deploy automático para PRD (após aprovação manual)

**Regras:**
- ❌ Commits diretos proibidos
- ✅ Merge apenas de `uat` após validação
- ✅ Sempre tagged com versão (v2.0.1)

#### 2. **uat** - Pré-Produção (UAT)

- Ambiente de homologação para validação final
- **Protected:** Requer PR + 1 aprovação
- Merges de `dev` ou `hotfix/*`
- Deploy automático para UAT

**Regras:**
- ❌ Commits diretos proibidos
- ✅ Merge de `dev` após features prontas
- ✅ Validação com times de negócio

#### 3. **dev** - Desenvolvimento

- Ambiente de desenvolvimento contínuo
- **Protected:** Requer PR
- Integração de features (`feature/*`)
- Deploy automático para DEV

**Regras:**
- ❌ Commits diretos evitados (usar PRs)
- ✅ Merge de `feature/*` branches
- ✅ Testes automatizados devem passar

### Branches Temporárias

#### Feature Branches

**Nomenclatura:**
```
feature/<JIRA-ID>-<descricao-curta>
```

**Exemplos:**
```
feature/PROJ-123-validacao-cpf
feature/PROJ-456-integracao-alfresco
feature/PROJ-789-relatorio-sinistros
```

**Workflow:**
```bash
# Criar feature branch a partir de dev
git checkout dev
git pull origin dev
git checkout -b feature/PROJ-123-validacao-cpf

# Desenvolver
git add .
git commit -m ":sparkles: adiciona validação de CPF"

# Push
git push origin feature/PROJ-123-validacao-cpf

# Criar PR para dev
# (via GitHub/GitLab UI)
```

**Ciclo de vida:**
- Criada a partir de `dev`
- Merge via PR para `dev`
- **Deletada após merge**

#### Hotfix Branches

**Nomenclatura:**
```
hotfix/<JIRA-ID>-<descricao-urgente>
```

**Exemplos:**
```
hotfix/PROJ-999-corrige-calculo-valor
hotfix/PROJ-888-fix-alfresco-timeout
```

**Workflow:**
```bash
# Criar hotfix a partir de main
git checkout main
git pull origin main
git checkout -b hotfix/PROJ-999-corrige-calculo-valor

# Fix
git add .
git commit -m ":bug: corrige cálculo de valor estimado"

# Push
git push origin hotfix/PROJ-999-corrige-calculo-valor

# Criar PR para main E uat E dev
# (merge para todas as branches)
```

**Ciclo de vida:**
- Criada a partir de `main` (produção)
- Merge via PR para `main`, `uat` e `dev`
- **Deletada após merge**

---

## Pull Requests

### Criando PRs

#### Nomenclatura

```
[TIPO] [JIRA-ID] Descrição curta
```

**Exemplos:**
```
[FEATURE] PROJ-123 Validação de CPF
[FIX] PROJ-456 Corrige timeout no Alfresco
[REFACTOR] PROJ-789 Refatora SinistroService
[DOCS] PROJ-999 Atualiza documentação de API
```

#### Template de PR

Todo PR deve seguir o template abaixo (configurado no `.github/PULL_REQUEST_TEMPLATE.md`):

```markdown
## Descrição
<!-- Descreva CLARAMENTE o que foi feito -->

**JIRA:** [PROJ-123](https://jira.company.com/browse/PROJ-123)

**Resumo:**
- Adiciona validação de CPF
- Valida formato (11 dígitos)
- Valida dígitos verificadores

## Tipo de Mudança
<!-- Marque com [x] -->

- [ ] 🐛 Bug fix (correção de bug)
- [ ] ✨ Nova feature (funcionalidade)
- [ ] ♻️ Refatoração (sem mudança de comportamento)
- [ ] 📝 Documentação
- [ ] 🧪 Testes
- [ ] 🔧 Configuração/CI

## Checklist Obrigatório
<!-- TODOS devem ser marcados [x] antes de aprovar -->

- [ ] ✅ App compila sem erros
- [ ] ✅ Linter passa (`mvn spotless:check`)
- [ ] ✅ Testes unitários passam (`mvn test`)
- [ ] ✅ Testes de integração passam (se aplicável)
- [ ] ✅ Testado localmente
- [ ] ✅ Documentação atualizada (se aplicável)
- [ ] ✅ Migrations criadas (se alterou DB)

## Screenshots (se aplicável)
<!-- Adicione prints se mudou UI -->

## Notas de Teste
<!-- Como testar essa mudança? -->

1. Acessar página de criação de sinistro
2. Digitar CPF inválido: "111.111.111-11"
3. Verificar mensagem de erro: "CPF inválido"

## Breaking Changes
<!-- Lista TODAS as breaking changes -->

- [ ] Nenhuma

## Dependências
<!-- Esta PR depende de outra PR? -->

- [ ] Nenhuma

---

**Revisores sugeridos:** @joao, @maria
```

### Configuração do Template

Criar arquivo `.github/PULL_REQUEST_TEMPLATE.md` no repositório:

```bash
mkdir -p .github
cat > .github/PULL_REQUEST_TEMPLATE.md << 'EOF'
## Descrição
<!-- Descreva CLARAMENTE o que foi feito -->

**JIRA:** [PROJ-XXX](https://jira.company.com/browse/PROJ-XXX)

**Resumo:**
- 
- 

## Tipo de Mudança
- [ ] 🐛 Bug fix
- [ ] ✨ Nova feature
- [ ] ♻️ Refatoração
- [ ] 📝 Documentação
- [ ] 🧪 Testes
- [ ] 🔧 Configuração/CI

## Checklist Obrigatório
- [ ] ✅ App compila sem erros
- [ ] ✅ Linter passa
- [ ] ✅ Testes passam
- [ ] ✅ Testado localmente

## Screenshots

## Notas de Teste

## Breaking Changes
- [ ] Nenhuma
EOF
```

### Tamanho de PRs

PRs devem ser **pequenas e focadas**:

| Tamanho | Linhas | Recomendação |
|---------|--------|--------------|
| **Ideal** | < 200 linhas | ✅ Perfeito para review |
| **Aceitável** | 200-500 linhas | ⚠️ OK, mas considere quebrar |
| **Grande** | 500-1000 linhas | ❌ Dificulta review |
| **Muito Grande** | > 1000 linhas | 🚫 Quebrar obrigatoriamente |

**Como medir:**
```bash
# Ver diff de linhas
git diff --stat dev..feature/PROJ-123
```

**Dica:** Use `git add -p` para fazer commits incrementais e quebrar PRs grandes.

### Aprovações Necessárias

| Branch Destino | Aprovações | Quem Pode Aprovar |
|----------------|------------|-------------------|
| **main** | 2 | Tech Lead + Senior |
| **uat** | 1 | Senior+ |
| **dev** | 1 | Qualquer dev |

---

## Code Review

### Responsabilidades do Autor

Antes de abrir PR:
- ✅ Executar linter: `mvn spotless:apply`
- ✅ Rodar testes: `mvn test`
- ✅ Testar manualmente
- ✅ Preencher template completo
- ✅ Adicionar screenshots (se UI)
- ✅ Linkar JIRA ticket

Durante review:
- ✅ Responder comentários em até 24h
- ✅ Fazer mudanças solicitadas
- ✅ Resolver conversas após implementar
- ✅ Re-request review após mudanças

### Responsabilidades do Reviewer

Durante review:
- ✅ Revisar em até 48h
- ✅ Ser construtivo nos comentários
- ✅ Testar localmente (casos complexos)
- ✅ Verificar checklist completo
- ✅ Aprovar OU solicitar mudanças (não deixar pendente)

### Checklist de Review

#### 1. **Lógica e Funcionalidade**

- [ ] Código faz o que diz que faz?
- [ ] Edge cases tratados?
- [ ] Validações adequadas?
- [ ] Não introduz bugs?

#### 2. **Qualidade de Código**

- [ ] Código legível e auto-explicativo?
- [ ] Nomes de variáveis/métodos claros?
- [ ] Duplicação evitada (DRY)?
- [ ] Complexidade razoável?
- [ ] Segue padrões do projeto?

#### 3. **Testes**

- [ ] Testes unitários adequados?
- [ ] Testes de integração (se necessário)?
- [ ] Coverage mantido/aumentado?
- [ ] Testes falham se código quebrar?

#### 4. **Performance**

- [ ] Queries otimizadas?
- [ ] N+1 queries evitadas?
- [ ] Uso adequado de índices?
- [ ] Caching apropriado?

#### 5. **Segurança**

- [ ] Input validado?
- [ ] SQL injection prevenido?
- [ ] Secrets não expostos?
- [ ] Autenticação/autorização OK?

#### 6. **Documentação**

- [ ] Javadocs em métodos públicos?
- [ ] README atualizado?
- [ ] API docs atualizados?
- [ ] Comments apenas onde necessário?

### Tipos de Comentários

Prefixar comentários com severidade:

#### 🚫 **BLOCKING:** Deve ser corrigido antes de merge

```
🚫 BLOCKING: Essa query tem N+1 problem. Use JOIN ou batch loading.
```

#### ⚠️ **IMPORTANTE:** Deve ser corrigido, mas pode ser em PR seguinte

```
⚠️ IMPORTANTE: Falta validação de CPF aqui. Pode criar ticket separado?
```

#### 💡 **SUGESTÃO:** Opcional, mas recomendado

```
💡 SUGESTÃO: Considere usar Optional<T> para evitar null checks.
```

#### 📝 **NITPICK:** Detalhes de estilo (não bloqueante)

```
📝 NITPICK: Renomear `data` para `sinistroData` para clareza.
```

#### ❓ **PERGUNTA:** Clarificação

```
❓ PERGUNTA: Por que usar List aqui ao invés de Set?
```

### Exemplo de Review Completo

```markdown
## Review - PR #123: Validação de CPF

### Geral
Código bem estruturado! Apenas alguns pontos abaixo.

### Comentários

**src/domain/valueobject/CPF.java**
```java
🚫 BLOCKING: Linha 23
A validação do dígito verificador está incorreta. Use o algoritmo oficial:
https://www.geradorcpf.com/algoritmo_do_cpf.htm
```

**src/application/usecase/CriarSinistroUseCase.java**
```java
⚠️ IMPORTANTE: Linha 45
Falta logar quando CPF é inválido. Adicione:
log.warn("CPF inválido: {}", cpf.getMasked());
```

**src/domain/valueobject/CPF.java**
```java
💡 SUGESTÃO: Linha 15
Considere cachear regex Pattern como constante static:
private static final Pattern CPF_PATTERN = Pattern.compile("\\d{11}");
```

### Testes
✅ Testes cobrem casos principais
⚠️ Falta teste para CPF com caracteres especiais (ex: "123.456.789-00")

### Checklist
- [x] Código funciona
- [x] Testes passam
- [ ] ⚠️ Falta teste de edge case
- [x] Documentação OK

**Decisão:** REQUEST CHANGES (corrigir blocking + adicionar teste)
```

---

## Convenções de Código

### Java

#### Nomenclatura

```java
// ✅ Classes: PascalCase
public class SinistroService {}

// ✅ Métodos: camelCase
public void criarSinistro() {}

// ✅ Variáveis: camelCase
String numeroSinistro = "2025000001";

// ✅ Constantes: UPPER_SNAKE_CASE
private static final int MAX_RETRIES = 3;

// ✅ Packages: lowercase
package br.com.appview.sinistro.domain;
```

#### Formatação

Use **Spotless** para formatação automática:

```bash
# Aplicar formatação
mvn spotless:apply

# Verificar formatação
mvn spotless:check
```

#### Ordem de Membros

```java
public class SinistroService {
    // 1. Constantes
    private static final Logger LOG = LoggerFactory.getLogger(SinistroService.class);
    private static final int MAX_RETRIES = 3;
    
    // 2. Atributos
    private final SinistroRepository repository;
    private final EventPublisher eventPublisher;
    
    // 3. Construtor
    public SinistroService(SinistroRepository repository, EventPublisher eventPublisher) {
        this.repository = repository;
        this.eventPublisher = eventPublisher;
    }
    
    // 4. Métodos públicos
    public Sinistro criarSinistro(CriarSinistroCommand command) {
        // ...
    }
    
    // 5. Métodos privados
    private void validarSinistro(Sinistro sinistro) {
        // ...
    }
}
```

### TypeScript/React

```typescript
// ✅ Componentes: PascalCase
export function SinistroCard() {}

// ✅ Hooks: camelCase com prefixo "use"
export function useSinistros() {}

// ✅ Interfaces: PascalCase
interface SinistroProps {}

// ✅ Enums: PascalCase
enum StatusSinistro {}

// ✅ Constantes: UPPER_SNAKE_CASE
const MAX_FILE_SIZE = 10 * 1024 * 1024;
```

---

## Segurança

### Secrets

❌ **NUNCA** comitar secrets no código:

```java
// ❌ NUNCA fazer isso
private static final String API_KEY = "abc123secret";
```

✅ **SEMPRE** usar variáveis de ambiente:

```java
// ✅ Fazer isso
@Value("${alfresco.api-key}")
private String apiKey;
```

### SQL Injection

❌ **NUNCA** concatenar SQL:

```java
// ❌ VULNERÁVEL a SQL injection
String sql = "SELECT * FROM sinistro WHERE cpf = '" + cpf + "'";
```

✅ **SEMPRE** usar prepared statements:

```java
// ✅ SEGURO
@Query("SELECT s FROM Sinistro s WHERE s.cpf = :cpf")
List<Sinistro> findByCpf(@Param("cpf") String cpf);
```

### Input Validation

✅ **SEMPRE** validar input do usuário:

```java
public void criarSinistro(CreateSinistroDTO dto) {
    // Validar CPF
    if (!CPF.isValid(dto.getCpf())) {
        throw new ValidationException("CPF inválido");
    }
    
    // Sanitizar descrição
    String descricao = HtmlUtils.htmlEscape(dto.getDescricao());
}
```

---

## Testes

### Coverage Mínimo

- **Domain (Entities, VOs):** 95%
- **Application (Use Cases):** 90%
- **Infrastructure:** 70%
- **Geral:** 80%

### Verificar Coverage

```bash
mvn test jacoco:report
open target/site/jacoco/index.html
```

### Executar Testes

```bash
# Todos os testes
mvn test

# Apenas unitários
mvn test -Dtest="**/*UnitTest"

# Apenas integração
mvn test -Dtest="**/*IntegrationTest"
```

---

## Documentação

### Javadoc

Documentar **todos os métodos públicos**:

```java
/**
 * Cria um novo sinistro no sistema.
 *
 * @param command dados para criação do sinistro
 * @return sinistro criado com ID gerado
 * @throws ClienteNaoEncontradoException se cliente não existir
 * @throws ContratoInvalidoException se contrato estiver cancelado
 */
public Sinistro criarSinistro(CriarSinistroCommand command) {
    // ...
}
```

### README

Manter README atualizado:

```markdown
# App View v2.0

## Stack
- Java 21
- Spring Boot 3.3
- PostgreSQL 15

## Setup
```bash
mvn clean install
mvn spring-boot:run
```

## Testes
```bash
mvn test
```
```

### Comentários

**Quando comentar:**
- ✅ Lógica complexa não óbvia
- ✅ Workarounds temporários (com TODO)
- ✅ Decisões arquiteturais importantes

**Quando NÃO comentar:**
- ❌ Código auto-explicativo
- ❌ O que o código faz (deve ser óbvio)

```java
// ❌ Ruim: comenta o óbvio
// Soma 1 ao contador
counter++;

// ✅ Bom: explica o "porquê"
// Retry com delay para evitar rate limiting do Jarvis (max 10 req/s)
Thread.sleep(100);
```

---

## Checklist Geral de Boas Práticas

### Antes de Comitar

- [ ] Código compila sem warnings
- [ ] Linter passou (`mvn spotless:check`)
- [ ] Testes passam (`mvn test`)
- [ ] Coverage mantido/aumentado
- [ ] Secrets não expostos
- [ ] Logs adicionados onde necessário
- [ ] Documentação atualizada

### Antes de Abrir PR

- [ ] Branch atualizada com base (`git rebase dev`)
- [ ] Commits limpos e atômicos
- [ ] Template de PR preenchido
- [ ] Screenshots adicionados (se UI)
- [ ] Testado localmente
- [ ] JIRA ticket linkado

### Antes de Mergear

- [ ] Todas as aprovações obtidas
- [ ] CI/CD passou (checks verdes)
- [ ] Conversas resolvidas
- [ ] Conflitos resolvidos
- [ ] Validado que não quebra nada

---

**Fim do Documento**
